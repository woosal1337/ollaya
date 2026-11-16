---
title: Ollaya Cloud limits, retries, and headers
nav: Cloud limits
description: Rate and concurrency limits per tier, the headers that report them, retries, deadlines, and usage tags on Ollaya Cloud.
order: 23
---

# Limits, retries, and headers

## Limits per tier

Each organization has a request rate (with a burst on top) and a number of requests it may have running at once. The dashboard's **Limits** page shows your tier and its numbers.

| Tier | Requests per second | Burst | Concurrent requests |
|---|---|---|---|
| Free | 5 | 10 | 2 |
| Tier 1 | 50 | 100 | 10 |
| Tier 2 | 200 | 400 | 40 |
| Tier 3 | 1,000 | 2,000 | 150 |
| Enterprise | by contract | by contract | by contract |

A key can have lower limits than its organization (see [Keys](/docs/cloud-keys)). Over the rate, a request gets `429 RATE_LIMITED`. Over the concurrency, it gets `429 CONCURRENCY_LIMITED`. Both carry `Retry-After` in seconds.

## Limits of the management API

The management API has its own request limits. They do not depend on the tier.

| Caller | Requests per second | Burst |
|---|---|---|
| A management key | 10 | 20 |
| A person in the dashboard | 20 | 60 |
| A request with no valid credential, for each client address | 10 | 30 |

Over a limit, a request gets `429 RATE_LIMITED` with `Retry-After` in seconds. A usage query that reaches its time limit of 30 seconds or its memory limit gets `422 INVALID_REQUEST`: use a shorter range.

Dashboard pages count against the limit of a person. Over it, a page says "Too many requests. Wait a few seconds, then reload." **Export line items as CSV** on the **Usage** page and **Download my data (JSON)** answer `429` with `Retry-After`.

## Members and invitations

An organization can have 100 pending invitations. An organization and a person can each send 50 invitation emails in 24 hours, resends included.

- At 100 pending invitations, the **Members** page refuses a new invitation. Cancel an invitation, then try again.
- At 50 invitation emails, the **Members** page says which limit refused the invitation and when to try again.
- At 100 members, the invitation page refuses to accept an invitation. Ask an Owner or Admin to remove a member, then accept again.

## Rate-limit headers

Answers to decisions carry the IETF rate-limit fields (`draft-ietf-httpapi-ratelimit-headers`, revision 11):

```http
RateLimit-Policy: "rate";q=10;w=2, "concurrent";q=2;qu="concurrent-requests"
RateLimit: "rate";r=9;t=1
```

`RateLimit-Policy` describes your limits: the burst `q` and the seconds `w` to refill it, and the concurrent requests. `RateLimit` says what is left: `r` requests, and `t` seconds until the burst is full again. An Enterprise organization without agreed limits has no limits, so its answers carry neither field.

## Overload

When a model's replicas are busy enough that a request would wait longer than your tier's limit, or longer than your deadline (below), Ollaya Cloud answers `503 QUEUE_FULL` with `Retry-After` at once instead of queueing it. The limit is two seconds of expected wait for Tier 3 and Enterprise, and a quarter, half, and three quarters of that for Free, Tier 1, and Tier 2, so under load Free traffic is refused first and paid traffic keeps its latency. A request for a model that is not loaded anywhere gets `503 QUEUE_FULL` with `Retry-After` set to the model's load time, while Ollaya Cloud loads it.

## Retries

The TypeSafe SDK retries `429` and `503` after `Retry-After`, and `5xx` answers, so most clients need nothing more. Decisions are idempotent, so a retry is always safe. Ollaya Cloud bills a `2xx` answer when it sends it, even if the answer does not reach you. Ollaya Cloud itself retries an attempt that failed in the worker once on another replica, within a retry budget, before you see an error.

## Deadlines

Send `X-Ollaya-Timeout-Ms: <milliseconds>` with a decision to say how long you will wait. Ollaya Cloud refuses at once (`503 QUEUE_FULL`) a request it could not answer in time, and cuts each attempt to that many milliseconds, instead of answering after you have given up. When the time runs out before a worker answers, for example while the request waits in a worker's queue, the answer is also `503 QUEUE_FULL`, with `Retry-After: 1`.

## Tags

Send up to 10 tags in `X-Ollaya-Tags` to split usage by your own dimensions, such as a customer or a feature:

```http
X-Ollaya-Tags: team=support, feature=refunds
```

Tags are `key=value` pairs separated by commas: keys of 1 to 32 characters from `a-z 0-9 _ . -`, values of 1 to 64 characters from `A-Z a-z 0-9 _ . : / -`. The management API's usage report groups by them (`group_by=tag:<key>`). A bad tag never fails the request: it is left out, and the answer carries `X-Ollaya-Tags-Dropped: 1`.
