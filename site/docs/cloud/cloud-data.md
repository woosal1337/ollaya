---
title: Ollaya Cloud data handling
nav: Cloud data
description: What Ollaya Cloud keeps about your requests, for how long, and where: no request content, only metadata.
order: 25
---

# Data handling

**Ollaya Cloud never stores or logs what you send or what it answers.** A state and its answers exist only in memory while Ollaya Cloud answers the request. No database, log, trace, or metric holds them.

## What is kept

For each request, Ollaya Cloud keeps metadata only, which is what the **Usage** and **Requests** pages show:

- the time, request ID, key, endpoint, and region
- the model asked for and the model that answered
- the status and error code, and the number of questions
- input tokens, the price, and the cost
- the total, model, and load times
- your tags (`X-Ollaya-Tags`) and the SDK name and version it sent

## For how long

| Where | What | How long |
|---|---|---|
| Request log | The metadata above, one row per request | 90 days after the request, then deleted |
| Usage charts | Counts of requests and errors, input tokens, cost, and latency percentiles per key, model, endpoint, and region | Per minute for 6 hours, per hour for 32 days, per day for as long as the request log keeps the requests |
| Usage stream | The metadata above, on its way from the gateway to billing and to the request log | The newest 10 million events of the region. An event that billing cannot post stays in a separate stream until the Ollaya Cloud team removes it. |
| Traces | The timing of each decision and the log lines written during it, never request content | 30 days |
| Service logs | Metadata such as request, organization, and key IDs, never request content | In the output of each service on its host. The code sets no time limit yet. |
| Ledger | Credits, and debits as sums per organization, never per request | No deletion |
| Spend totals | Spend per organization and per key, by month and in total, and the days of unusual spend with the keys and models that spent most | No deletion |

Account data (your name, email address, passkeys, organizations, members, and the masked forms of keys) is kept while the account exists. To delete an account, write to the Ollaya Cloud team. **Account** → **Download my data (JSON)** exports your profile, sessions, passkeys, memberships, and what you did in each organization. The audit log keeps who changed what in each organization, and nobody can edit or delete an entry.

## Keys

Ollaya Cloud stores only a keyed hash of each API key, never the key itself, so nobody can read a key back, Ollaya included. See [Keys](/docs/cloud-keys).
