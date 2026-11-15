# ADR-0010: Compute a request's state once, and fork it into each question

## Status

Accepted for `kev`, `decider`, `decision` and the Qwen3.5 tags of `decision2`, in their ONNX graphs and on MLX. The
other families and llama.cpp are follow-ups (see Consequences).

## Date

2026-11-10

## Context

The Qwen3.5 decoder families answer one question per row, and every row starts with the request's state. A
five-question request computes that state five times, plus the padding of each row to a multiple of 64. `decider`
also gives each level of an isolated score question its own row, and every `nimble` row carries the whole request
(only the requested field at its end differs).

Flide ([OmniJev/Flide](https://github.com/OmniJev/Flide), Apache-2.0) serves the same kind of models from one
computation of the state, forked into one branch per question, and calls it exact state forking: each branch
continues from the Gated DeltaNet state, the conv window and the attention keys at the end of the shared prefix,
so it computes the same function as the question asked alone. Flide is Python, PyTorch and Triton on CUDA, so
Ollaya takes the idea and none of the code.

## Decision

1. **A fork form of the graph.** `Qwen35Trunk.fork` (`convert/ollaya_convert/families/llm_common/qwen35.py`) takes
   `prefix_ids [J, P]`, `branch_ids [B, S]` and `branch_job [B]`. Row `b` is `prefix_ids[branch_job[b]] ++
   branch_ids[b]`; read positions count from the start of the row, so the heads read what they read before. Every
   prefix runs once, and each branch starts from its prefix's DeltaNet state and conv window and attends to its
   prefix's keys.
2. **Chunk-aligned prefixes.** P and S are multiples of 64, the gated delta rule's chunk. The prefix's chunks are then
   the row's chunks, and so are the branch's. The function is the same, not an approximation: in eager fp32 the
   hidden states are bit-identical to the row form's on kev-0.8b and decider-0.8b. The goldens stay as they are, and
   the gate is unchanged.
3. **The split lives in the runner.** `crates/ollaya-runner/src/fork.rs` cuts a request's token rows, with no family
   code:
   - rows that share 64 tokens or more fork at their longest common prefix, rounded down to 64 and short of every
     row's end (one job);
   - otherwise each row is its own job, cut one chunk before the end of the longest row, so the run costs what the row
     form costs;
   - a run carries the prefix once and as many branches as the token budget leaves room for, at least half of it, so
     a prefix over the budget still runs once per run.

   `qwen35.fork_rows` is the same split in Python, for the exports and `ort_rows.run_rows`.
4. **Both forms run.** A family's runner takes the fork path when its graph has a `prefix_ids` input and the row path
   otherwise, so installed models keep working and each family moves when its fork graph passes its gate.
   `decision.json` names the form (`contract.form`).

## Results

kev-0.8b, decider-0.8b, decision-eos-0.8b and decision2-eos-0.8b in fp32 on ONNX Runtime, CPU, against goldens made
from the authors' code (the edge cases and 40 typed-decisions requests, and decision2's 13 cases of its own):

| Model | Questions | Decisions the same | Max logit difference |
|---|---:|---:|---:|
| kev:0.8b | 580 | 100% | 4.5e-5 (as the row graph) |
| decider:0.8b | 579 | 100% | 3.8e-5 |
| decision:eos | 566 | 100% | 2.8e-5 (as the row graph) |
| decision2:eos | 563 | 100% | 2.0e-5 (as the row graph) |

On the 40 typed-decisions requests, kev's fork graph computes 35,968 tokens instead of 67,200. A decision row repeats
less of its request: the shared start is about 210 of its 363 tokens, and the fork graph computes 51,840 tokens
instead of 80,000. decision2's rows give the same counts. Latency of the same requests on one development machine
(an Apple M5, CPU, `bench --ids td/`), not an edition machine:

| Model | p50, row → fork | p95, row → fork | Memory, row → fork |
|---|---|---|---|
| kev:0.8b | 1,920 → 1,007 ms | 3,470 → 1,386 ms | 6.6 → 4.2 GiB |
| decider:0.8b | 4,543 → 1,878 ms | 7,637 → 2,707 ms | 9.4 → 6.1 GiB |
| decision:eos | 2,798 → 1,712 ms | 14,233 → 2,649 ms | 7.5 → 5.8 GiB |

The decision runs used `--warmup 3 --full-only` and took the memory from `/usr/bin/time -l` (peak memory footprint).
Another job loaded the machine during them (load average about 16), so the row graph's p95 is noisy. decision2:eos
has no CPU latency run: other jobs loaded the machine at the time, so its CPU times would mean nothing.

The gain grows with the state: short one-line states save little, long ones approach one state per request instead
of one per question.

The MLX fork (`Qwen35::fork`) on Metal, on an Apple M5 with 24 GB: parity against the same goldens, and latency of
the same 40 typed-decisions requests (`bench --ids td/ --warmup 3 --full-only`). The row path is the same build with
the fork turned off. Memory is the peak memory footprint of the process (`/usr/bin/time -l`). The resident set size
on macOS does not count the buffers of MLX.

| Model | Decisions the same | Max logit difference | p50, row → fork | p95, row → fork | Memory, row → fork |
|---|---:|---:|---|---|---|
| kev:0.8b | 580 of 580 | 3.6e-5 | 1,155 → 585 ms | 1,863 → 789 ms | 5.6 → 4.7 GiB |
| decider:0.8b | 579 of 579 | 4.2e-5 | 2,744 → 1,123 ms | 4,251 → 1,741 ms | 6.0 → 5.4 GiB |
| decision:eos | 566 of 566 | 1.8e-5 | 1,531 → 1,135 ms | 2,200 → 1,916 ms | 5.7 → 4.8 GiB |
| decision2:eos | 563 of 563 | 2.6e-5 | 1,368 → 739 ms | 2,143 → 1,020 ms | 6.1 → 5.5 GiB |

decision2:eos had one run of each path. Other jobs loaded the machine during them (load average 14 to 19).

## Consequences

- A family moves to the fork form with its export and its runner, and ships after its gate on CUDA and the CPU, as
  any graph change does (`docs/model-lifecycle.md`). kev's and decider's larger tags, the `-fp16` tags and CUDA are
  not gated yet, and neither are decision and decision2 on CUDA. The decision2 export also writes Sol, Nox and Lux in
  the fork form, but they have no gate with it yet.
- Still in the row form: `nimble` and `jeeves` (converted on a branch, waiting for a 9B gate), `decision2:kai` (a
  Qwen3 trunk), `d3`, `strands`, `arbiter` and `decider-vision`. Attention-only trunks (Qwen3, Gemma) need only the
  keys and values.
- The MLX engine computes from the weights, not the graph, so it has its own fork (`Qwen35::fork` in
  `mlx/qwen35.rs`). On MLX, `kev`, `decider`, `decision` and `decision2` fork whenever the backbone is Qwen3.5
  (`Decoder::forks`), whatever the form of the installed graph. kev:0.8b, decider:0.8b, decision:eos and
  decision2:eos pass their gate on Metal with the fork (Results). Their larger tags have no Metal run with the fork
  yet, and decision2 has no arch layer in the library, so installed decision2 models do not run on MLX.
- llama.cpp can fork a recurrent cache (`llama_memory_seq_cp` copies the state on write), but that changes the
  model's batch split, which moves its numbers away from the goldens of the authors' client. That needs its own
  decision.
- The contract can hold several requests in one run (`branch_job`), which batching across requests and a cache of
  agents' states would build on. Neither is part of this decision.
