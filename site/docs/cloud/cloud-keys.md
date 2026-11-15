---
title: Ollaya Cloud API keys
nav: Cloud keys
description: Authenticate to Ollaya Cloud with API keys, and limit what each key may do, spend, and call.
order: 21
---

# API keys

Every request to `https://api.{{SITE_HOST}}` carries a key in `Authorization: Bearer <key>`. Keys belong to an organization. Anyone in it with the Owner, Admin, or Developer role can create them on the dashboard's **API keys** page or through the management API.

A key is `oly_live_` (or `oly_test_` for a test key) followed by 46 characters. The last six characters are a checksum, so Ollaya Cloud refuses a mistyped key before it looks it up. Ollaya Cloud stores only a keyed hash of each key and a masked form (`oly_live_…abcd`). So you see a key once, when you create it, and never again. If you lose it, create a new one.

## What a key may do

| Setting | Effect |
|---|---|
| Scope | `inference` calls models (`/v1/*`, `/api/decide`). `management` calls the management API: it manages keys and reads usage, requests, credits, and limits, with the role of the member who created it. Only Owners and Admins create management keys. A `read` scope is reserved for read-only access and opens no route yet. |
| Allowed models | Only these models. Any other model answers `403 MODEL_NOT_ALLOWED` |
| Spend cap | A budget per month or in total. Once it is used up, the key answers `402 SPEND_CAP_REACHED` |
| Rate and concurrency | Lower limits than the organization's, for one noisy client |
| IP allowlist | Up to 32 addresses or CIDR ranges. Calls from elsewhere answer `403 IP_NOT_ALLOWED` |
| Expiry | 30, 90, or 365 days, or none |

The organization's own settings apply on top: its model access list, its budget, and its residency region.

## Keep keys on the server

A key belongs in your server's environment or secret store, never in a web page or a mobile app. Ollaya Cloud refuses any request with an `Origin` header (`403 BROWSER_NOT_ALLOWED`), so a key pasted into front-end code fails at once instead of leaking quietly.

## Revoke a key

Revoke a key on the **API keys** page. Every gateway refuses it at once in normal operation. If the gateways cannot get the message, they refuse the key at most 60 seconds later. The audit log (**Settings** → **Audit log**, for Owners and Admins) records who created, changed, and revoked each key.

## Remove a member

When an Owner or Admin removes a member on the **Members** page, Ollaya Cloud first revokes the active keys that the member created. To keep these keys working, select **Keep *name*'s keys working** before you remove the member. A kept key bills the organization as before. A kept management key stops working, because it acts with the role of its creator.
