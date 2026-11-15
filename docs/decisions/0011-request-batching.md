# ADR-0011: Batch concurrent requests in the runner, one family at a time, behind a load gate

## Status

Proposed (OLY-214), for review by Mert.

## Date

2026-11-15

## Context

**One request per pass.** The daemon starts one runner process per loaded model
(`crates/ollaya-server/src/scheduler.rs:1-6`, `:238-242`). It forwards each request to that runner over HTTP as soon
as the request has a lease (`service.rs:172-183`, `scheduler.rs:155-170`). Only `OLLAYA_MAX_QUEUE` (default 512)
limits the requests in flight (`http.rs:602-615`). The runner turns each request into one job on its single model
thread, and the thread runs the jobs in arrival order (`crates/ollaya-runner/src/server.rs:510-526`, `:611-642`). The
thread must stay the same thread for the life of the runner, because ONNX Runtime pools CUDA streams per calling
thread (`server.rs:498-509`, issue #69).

**Rows already share a pass inside one request.** Each ONNX family turns a request into rows, pads them to the longest
row and runs them in passes of at most a token budget (`engine.rs:65-107`). Laya, and so `coba`, gives one row per
question, with an attention mask and marker slots (`onnx.rs:288-353`). The pplx family, and so `sura`, puts all the
questions of a request in one sequence (`pplx.rs:4-9`). A typical `sura` request is thus a pass of one row, while
the pass budget holds 8,192 padded tokens and up to 4,096 rows (`pplx.rs:27-31`).

**Why it matters.** The pricing note puts `pplx-decider:flash-fp16`, of the same family as `sura`, at $0.174 per 1M
input tokens at 40% use, against $0.010 on the market. Encoders already make money. The production risk table lists
"one forward pass at a time in the engine" and names batching as the answer.

**The fixed plan.** The llama.cpp engine evaluates every question "with one fixed plan, so its numbers never depend
on what ran before it or on the other questions of its request" (`llama/mod.rs:10-17`). It decodes on sequence 0
only, one question after the other (`llama/mod.rs:1608-1660`, `:1710-1750`), and `n_seq_max` defaults to 1
(`llama/mod.rs:76-77`, `:94-96`). `docs/families/llm-logits.md` ("Determinism") measured that a different batch split
moves probabilities by up to 0.19 (Qwen3-4B) and 0.51 (Gemma 4 E2B) on CUDA, with argmax flips. In the ONNX
families, the passes of a request depend only on that request. Batching across requests breaks this: the numbers of
a request then depend on the other requests in its pass, and so on the load.

**Evidence on one CPU.** A scratch program (not in the repository) ran 32 golden requests of `kev:0.8b` (row graph,
fp32, 160 rows, 10,261 tokens) on ONNX Runtime 1.28, CPU, an Apple M5, on 2026-11-15. It ran each request alone, as
the runner does now, then the same requests merged into rounds of 2, 8 and 32, through the unchanged row path
(`KevModel::scores`: rows sorted by length, padded to 64, split by the token budget):

| Round size | Rows bit-identical to alone | Max logit gap to alone | Max probability gap | Decisions changed | Max logit gap to goldens | Decisions unlike goldens |
|---|---:|---:|---:|---:|---:|---:|
| 1 (alone) | 160 / 160 | 0 | 0 | 0 | 4.4e-5 | 0 |
| 2 | 160 / 160 | 0 | 0 | 0 | 4.4e-5 | 0 |
| 8 | 155 / 160 | 1.1e-5 | 4.6e-7 | 0 | 4.4e-5 | 0 |
| 32 | 155 / 160 | 1.8e-5 | 1.4e-6 | 0 | 4.4e-5 | 0 |

A second run gave the same rows and the same gaps, so a given batch gives the same numbers each time. The numbers move
with the batch on the CPU, but far inside the 1e-3 gate (`parity_kev.rs:34`). The program does not show the cause:
the padded length of a pass and its row count both change. The merged rounds were not faster on this CPU (0.38 to
1.02 times the speed of the alone runs, over two runs, with a load average of 16 to 24 on 10 cores from other work).
These times say nothing about a GPU. The program did not run on CUDA, on an encoder or on an fp16 graph.

**Pull requests in review.** #158 checks a client deadline (`X-Ollaya-Timeout-Ms`) on the model thread just before
the pass (`ModelThread::run_before`). #159 adds `/metrics`, where `ollaya_requests_queued` counts in-flight requests
minus the one pass that a runner computes. #169 (ADR 0010) adds the fork form of the Qwen3.5 graphs, whose
`prefix_ids [J, P]`, `branch_ids [B, S]` and `branch_job [B]` already let one run hold J prefixes.

## Decision

1. **The batch forms in the runner's model thread.**
   - The queue of the model thread carries typed decide jobs beside the closures it carries now. The thread takes the
     oldest job and encodes it, as today. Then it adds the next decide jobs, in order, and stops at the first of:
     - the batch holds `OLLAYA_BATCH_MAX` requests
     - the rows of the next request do not fit in the same pass (the family's token budget and row limit)
     - the wait window ends
     - the next job is not a decide job (a load or a warm-up always runs alone, in order)
   - `OLLAYA_BATCH_MAX` is a count from 1 to 64, default 1. `OLLAYA_BATCH_WAIT` is a duration such as `2ms`, default
     `0ms`, at most `50ms`, counted from the arrival of the oldest request in the batch. The daemon validates both in
     `config.rs`, as it does `OLLAYA_THREADS` (`config.rs:171`), and passes them to each runner as `--batch-max` and
     `--batch-wait`, as it passes `--threads` (`scheduler.rs:541-542`). The shared duration parser reads a bare
     number as seconds (`config.rs:133-138`, `KeepAlive::parse`), so the cap of 50 ms stops a value of `2` at
     startup.
   - With the default of 1, the thread takes one job and runs it at once, on today's code path. A user who sets
     nothing sees no change in numbers or in latency.
   - With a wait of `0ms`, a batch holds only requests that are already in the queue. It adds no wait: it merges the
     requests that queued during the previous pass. A wait above zero helps only when a pass is short compared with
     the gap between requests. OLY-219 shows whether 2 ms pays.
   - A family that batches implements two steps: encode one request, and run the passes of several encoded requests,
     with one result for each request. A family that does not batch keeps `Engine::run`, and two of its requests
     never share a pass. A family batches only on the devices where it passed the gate in decision 3.

2. **ONNX encoders: the rows of several requests in one `session.run`.**
   - The thread encodes each request alone: tokenizer, layout, limits. An encode error (`INVALID_REQUEST`,
     `TOO_MANY_OPTIONS`) goes back to its own caller, and that request never joins the pass.
   - The rows of the batch go through the pass code that each family has now. It pads each row with the pad id to the
     longest row and sets attention mask 0 on the padding. It gives each row as many marker slots as the widest row,
     with `marker_mask` false on the extra slots (`onnx.rs:310-333`). Each row keeps its request and question index,
     and its outputs go back by that index, as `pplx.rs:252-257` does now inside one request. `input_tokens`,
     `state_tokens` and `state_truncated` stay per request.
   - The token budget does not change: 32,768 padded tokens for the encoders (`engine.rs:67`). One request can fill
     it today, so a batch cannot raise the peak activation memory. A request that needs more than one pass on its own
     never joins a batch. It runs alone, in its own passes, as now (`engine.rs:89-107`).
   - Error isolation: when a merged pass returns an error or panics, the thread runs each request of that pass alone
     and gives each request its own result. This costs time only after a failure.
   - Decima has its own pass shape, with state rows and option rows (`decima.rs:86-112`, `:219-246`). In a batch,
     its `option_state` index moves by the row offset of each request. NLI and GLiClass use the shared row path
     (`nli.rs:147-190`, `gliclass.rs:154-170`).

3. **Answers can move with the batch, so each family passes a load gate before it batches.**
   - Known: a request's passes depend only on the request today (`engine.rs:89-107`). On llama.cpp, a different split
     moves the numbers far outside the gate (Context). On ONNX Runtime on one CPU, `kev:0.8b` moved by at most 1.8e-5
     in a logit, with no decision changed (Context). TF32 is off on CUDA (`onnx.rs:148-152`).
   - Unknown until measured: CUDA, where cuBLAS and ONNX Runtime pick kernels by shape. The encoders. The fp16 graphs.
     Larger round sizes. Whether a gap ever moves a decision on the goldens.
   - The check: a new example, `batch_check`, loads one model, runs every golden request alone, then the same requests
     in rounds of 2, 8 and 32, merged in file order. It runs on the CPU and on CUDA. For each round size it reports
     the rows bit-identical to alone, the largest logit gap and the largest calibrated probability gap to alone, the
     decisions changed against alone, and the family's gate numbers against the goldens.
   - The gate: in every round size, the family's existing fp32 gate against the goldens holds, that is logits within
     1e-3 or the family's tighter tolerance, and every decision the same as the reference
     (`docs/model-lifecycle.md:102-103`). An `-fp16` tag holds its own gate (ADR 0008) in every round size. The check
     reports bit equality but does not require it.
   - A family or a device that fails the gate does not batch. Nobody loosens a tolerance to pass it.
   - K1 runs `batch_check --smoke` for each family whose batching code changes. K2 runs the full check for every
     family that batches, on every device it batches on. Its records go to `results/runs/`, as parity records do.
   - With batching on, a runner no longer promises bit-identical answers for the same request. The answers stay
     inside the gate, and the docs of `OLLAYA_BATCH_MAX` say so. With the default, the promise holds as today.

4. **Decoders.**
   - pplx (`sura`): one request is one sequence, and so one row (`pplx.rs:200-260`). The rows of several requests share
     a pass, padded to a multiple of 64, with each real length in `lengths`. The linear-attention layers are causal and
     the full-attention layers mask the padded keys (`docs/families/pplx.md`, "Padding"), so no padding reaches a real
     token. This family gains the most, because each of its passes holds one row today.
   - Row families in the fork form (`kev`, `decider`, after #169): each request is one job, and one run holds J jobs.
     The jobs of one run share one prefix length P, because `prefix_ids` is [J, P]. So the run cuts every job at the
     smallest fork point among them, a multiple of 64, and the rest of each row goes into its branches. ADR 0010 shows
     that a cut on a 64-token boundary computes the same function. The thread groups requests with close fork points,
     so the shorter cut costs little. The graph contract does not change. Row-form graphs batch as in decision 2.
   - llama.cpp: several sequences in one `llama_decode`, each with its own sequence id, make a new batch split.
     `n_seq_max` must then rise above 1, and without `kv_unified` each sequence gets only `n_ctx / n_seq_max` tokens
     of cache (`llama/mod.rs:745-750`). The goldens' reference runs the single-sequence plan on llama-server
     (`llm_common/plan.py`). A batch changes the split with each load, so no single set of goldens describes it, and
     the measured split gaps (0.19 to 0.51) are far outside the gate. OLY-216 therefore starts with the measurement:
     the determinism check, extended to several sequences per decode, against the current goldens, on the CPU and on
     CUDA. A family that passes batches. A family that fails does not batch, and its family doc records the gap.
     Goldens made from a multi-sequence reference would be new goldens for a new plan, which needs the maintainer's
     consent.

5. **Deadlines, fairness and limits.**
   - Deadlines (#158): the thread checks the deadline of each request when it adds the request to a batch. An expired
     request gets `504 DEADLINE_EXCEEDED` and does not join. The wait window ends at the earliest deadline in the
     batch if that comes first. Once a pass starts, it completes, as in #158.
   - Fairness: the thread takes requests in arrival order and never skips one to fill a batch with a later one. A
     request that does not fit closes the batch and opens the next. So no request waits for more than one pass behind
     a later request.
   - Added latency: a batched request waits at most `OLLAYA_BATCH_WAIT` for its batch, then for one pass of at most
     the token budget, instead of its own pass. OLY-219 measures the time of a full pass per model, and the cloud sets
     the two variables from it.
   - `/metrics` (#159): the runner adds `batch` (`requests`, `rows`, `wait_ms`) to its `/decide` answer. The daemon
     records two histograms: `ollaya_batch_requests` (buckets 1, 2, 4, 8, 16, 32, 64) and
     `ollaya_batch_wait_seconds` (buckets from 0.5 ms to 50 ms, because #159's lowest bucket is 5 ms).
     `ollaya_requests_queued` must subtract the requests of the pass in progress, not one. The runner's `/health`
     gives `queued` and `in_pass`, and the daemon reads them when Prometheus scrapes it.
   - Memory on CUDA: the budget bounds a pass as today, but a busy runner now fills the budget on most passes. The
     arena grows to the size of a full pass and keeps it (`onnx.rs:102-116`). OLY-219 records the GPU memory at each
     batch size. That number, not the idle size, is the size of the model for placement.

6. **The measurement plan (OLY-219, then OLY-215).**
   - One machine: hope's RTX 5090, through `scripts/remote-job.sh`. One build for the comparison: the commit of
     OLY-215, with batching off by default. Each row of the matrix then runs the same binary, and only the
     environment changes.
   - Models: the `coba` tag the cloud sells, `laya:en`, and `sura:2b` (fp32, and fp16 once it exists).
   - Load: the five-question triage preset and the typed-decisions requests, through the throughput tool of OLY-218,
     closed loop at concurrency 1, 2, 4, 8, 16, 32 and 64.
   - Rows of the matrix: 1, 2, 4 and 8 daemons per GPU with batching off. Then 1 daemon with `OLLAYA_BATCH_MAX` 8 and
     32, each at `OLLAYA_BATCH_WAIT` 0 and 2 ms. Then 2 daemons with the best batch setting.
   - For each row: requests per second, p50 and p99 latency, `utilization.gpu` and memory from `nvidia-smi`, the
     batch histograms, and the cost per 1M input tokens at 40% use, as JSON in the shape of `results/runs`.
   - OLY-219 can run the batching-off rows now, on `main`, for the pricing note. OLY-215 then runs the whole matrix
     again on its own commit, so that the rows it compares come from one machine and one build.

7. **Order of work.**

   | Order | Task | Scope | Gate before merge |
   |---|---|---|---|
   | 1 | OLY-219 | Daemons per GPU, batching off | The matrix rows with batching off, recorded |
   | 2 | OLY-215 | Model-thread queue, the two variables, `batch` fields, `batch_check`, laya (`coba`, `laya`) | `batch_check` passes for each laya tag in every round size on CPU and CUDA. Unit tests: the default runs one request per pass, one bad request leaves the others answered, an expired request never runs. K1. The whole matrix on the RTX 5090 |
   | 3 | OLY-215, follow-ups | The other ONNX encoders, one family at a time | `batch_check` per family and device |
   | 4 | New task | pplx (`sura`) as rows, then `kev` and `decider` in the fork form with J jobs, after #169 | `batch_check` on CPU and CUDA, fp16 tags on their own gate, cost per 1M tokens against the market price recorded |
   | 5 | OLY-216 | llama.cpp: the multi-sequence measurement first, code only for families that pass | `batch_check` against the current goldens, or the gap recorded |

## Alternatives considered

- **The daemon forms the batch.** The daemon does not tokenize, so it cannot apply a family's token budget or row
  limit. It would need a new wire format for several states in one call, and it would split errors across the
  process boundary. The model thread already orders every pass. Rejected.
- **Only more processes per GPU.** It needs no code, and OLY-219 measures it. But each process holds its own weights
  and its own CUDA arena, and it never merges passes, so a one-row `sura` pass stays one row. It stays the fallback
  for a family that fails the gate.
- **Fixed-shape passes.** Pad each pass to a fixed row count and a length bucket, so that the shape of a pass depends
  only on the row. That can give bit-identical numbers under load without new kernels. But a lone request then pays
  for a full pass of padding, and the new shapes change today's numbers with batching off. Rejected for now.
  Reconsider it if the cloud must give bit-identical answers under load, or if a family fails the gate.
- **Batch-invariant kernels**, as Flide writes them in Triton. ONNX Runtime has no such kernels, and Ollaya ships no
  kernels of its own for ONNX Runtime. Rejected.
- **A wait window by default.** It adds latency to every request on an idle server and gives nothing back. Rejected:
  the default wait is 0.
- **llama-server's slot batching.** It changes the split with the load of its slots, which moves the answers
  (decision 4). Rejected.

## Consequences

- With the defaults, nothing changes: the same code path, the same numbers and the same latency.
- Each family moves when its load gate passes on a device. K2 gains one `batch_check` run per batching family and
  device.
- `docs/api.md` and the configuration docs get the two variables and the note on numbers under load.
- The cloud can run gated families with batching on. Its answers can then differ in the last digits from the same
  request alone, inside the gate.
- The load check is the one that OLL-11 asks for before opt-in batching ships.
- The task notes name `0010-dynamic-batching.md`. ADR 0010 is state forking (#169), so this record is 0011.

## Open questions for the maintainer

1. Can the cloud accept answers that move inside the gate with load? If not, the fixed-shape passes come back, and
   their cost must be measured.
2. Should the gate also require that no decision changes against the same request alone, in every round size? The
   proposed gate compares each round with the goldens only.
3. Are a default wait of 0, a cap of 50 ms and a cap of 64 requests the right values?
4. Should the `batch` fields stay between runner and daemon, or also reach `/api/decide`, for example to split
   `eval_duration` into wait and compute?
5. For llama.cpp: if a family fails, do we accept that it never batches, or do we make goldens for a multi-sequence
   plan?
6. Should pplx (`sura`) get its own task, before the other encoders, because it carries the price gap?
