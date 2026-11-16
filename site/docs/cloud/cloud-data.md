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

It also keeps the /24 (IPv4) or /48 (IPv6) prefix of the caller's address, never the full address. The pages do not show the prefix.

## For how long

| Where | What | How long |
|---|---|---|
| Request log | The metadata above, one row per request | 90 days after the request on every tier, then deleted |
| Usage charts | Counts of requests and errors, input tokens, cost, and latency percentiles per key, model, endpoint, and region | Per minute for 6 hours, per hour for 32 days, per day for 90 days |
| Usage stream | The metadata above, on its way from the gateway to billing and to the request log | 48 hours, or longer until billing and the request log read the event. At most the newest 1 million events of the region. An event that billing cannot post stays in a separate stream until the Ollaya Cloud team removes it. |
| Traces | The timing of each decision and the log lines written during it, never request content | 30 days |
| Service logs | Metadata such as request, organization, and key IDs, never request content | 30 days. The host of each service also keeps its newest lines, up to 50 MB for each service. |
| Ledger | Credits, and debits as sums per organization, never per request | No deletion |
| Spend totals | Spend per organization and per key, by month and in total, and the days of unusual spend with the keys and models that spent most | No deletion |

The usage charts and the request log thus go back 90 days. The **Usage** page and the management API accept a range of up to 400 days. They show no data for a day older than 90 days.

Account data (your name, email address, passkeys, organizations, members, and the masked forms of keys) is kept while the account exists. To delete it, see [Delete your account](#delete-your-account). **Account** → **Download my data (JSON)** exports your profile, sessions, passkeys, memberships, and what you did in each organization. The file holds at most 100 sessions, 100 passkeys, and your newest 10,000 actions. Its `lists` field says for each list whether it is complete.

The audit log keeps who changed what in each organization. Nobody can delete an entry. Only an account delete changes entries: it removes the person's email address, IP prefix, and user agent.

## Delete your account

**Account** → **Delete account** deletes your account. You cannot undo it. Before you confirm, the page lists each of your organizations and the keys that stop in it.

- Each organization where you are the only member closes. Every key of the organization stops, and so do its API and playground requests.
- You leave every other organization. The keys that you created there stop. The other members keep the organization and their own keys.
- The delete removes your sign-in, sessions, passkeys, two-step verification, and memberships. It also removes the invitations that you sent and the invitations to your address.
- A single sign-on connection that you set up for an organization that stays moves to another Owner of that organization.

Two things stop a delete. The page names each organization concerned and says what to do:

- You are the only Owner of an organization with other members. Make another member an Owner first.
- An organization that would close has credit other than the welcome credit. To close it, write to the Ollaya Cloud team.

The delete needs a sign-in of the last ten minutes. If your sign-in is older, the dialog asks you to sign in again. To confirm, type your email address. Ollaya Cloud then signs you out.

These stay after the delete:

- The financial record of each organization that closes: its ledger, credit grants, settings, and revoked keys. It names the organization, not you.
- The audit events of each organization. They keep your user ID as the actor, but not your email address, IP prefix, or user agent. Your name can stay in text that people typed, such as a key name.
- A keyed digest of your mailbox, which is not your address. Ollaya Cloud keeps it to refuse a second welcome credit to the same mailbox.

To use Ollaya Cloud again, sign in with any address. This starts a new account.

## Keys

Ollaya Cloud stores only a keyed hash of each API key, never the key itself, so nobody can read a key back, Ollaya included. See [Keys](/docs/cloud-keys).
