---
name: urnetwork
description: Route traffic through URnetwork — private HTTPS, SOCKS5, or WireGuard egress from a specific country, region, or city. Use when a task needs requests to originate from another location, needs many distinct egress IPs (distributed scraping), needs a private/anonymous network path, or when code must handle an HTTP 402 from api.bringyour.com and pay for its own plan upgrade over x402.
license: MPL2.0, see LICENSE
---

# URnetwork

URnetwork is a decentralized privacy network. Humans use the apps (Android, iOS, Chrome); agents use the API.

- API host: `https://api.bringyour.com`
- API spec: https://ur.io/openapi.yml (mirrored from github.com/urnetwork/connect `api/bringyour.yml`)

Every call is `Content-Type: application/json` and, except for `/auth/code-login`, requires `Authorization: Bearer <JWT>`.

## MCP server (use it if your runtime speaks MCP)

If your harness supports the Model Context Protocol, the MCP server is the
happy path and this skill's proxy recipes are the fallback:

```
https://mcp.bringyour.com
```

- Protocol: MCPv2 — the stateless 2026-07-28 revision (streamable HTTP, JSON
  responses). Older revisions negotiate down automatically.
- Auth: OAuth — the first connect opens a browser sign-in and issues a scoped
  token; no auth code to paste. Discovery metadata:
  `https://mcp.bringyour.com/.well-known/oauth-protected-resource`
- Tools: `providerLocations` (find countries/regions/cities with provider
  counts; scope `mcp:read`) and `fetch` (load a URL as if browsing from a
  chosen place, threading `signed_proxy_id`/`cookies`/`continuation` back
  through the caller; scope `mcp:fetch`).
- Add-server recipes for 20+ harnesses: https://ur.io/agents.md

Everything below is the API path for runtimes WITHOUT MCP — auth-code
sign-in, raw proxies, and paying for an upgrade over x402.

## Quickstart

Three calls get you a working proxy:

```bash
# 1. auth code -> JWT (ask the human for the auth code)
JWT=$(curl -s -X POST https://api.bringyour.com/auth/code-login \
  -H 'Content-Type: application/json' \
  -d '{"auth_code": "<AUTH CODE>"}' | jq -r '.by_jwt')

# 2. find the location you want, and take its location_id
curl -s -X POST https://api.bringyour.com/network/find-provider-locations \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"query": "Japan"}' | jq '.locations[] | select(.location_type=="country")'

# 3. create a proxy pinned to that location
curl -s -X POST https://api.bringyour.com/network/auth-client \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"proxy_config": {"initial_device_state": {"location": {"connect_location_id": {"location_id": "<LOCATION ID>"}}}}}' \
  | jq '.proxy_config_result'
```

**Always check the response for an `error` object before using it.** See [Handling errors](#handling-errors) — a plan limit returns HTTP 200 with an error body, not an HTTP error.

## Authentication

Ask the human for an auth code, then exchange it for a JWT via `/auth/code-login`. The JWT can be stored and reused. To refresh it, ask for a new auth code and repeat.

## Choose a proxy protocol

| Use case | Protocol | Plan | How to use it |
| --- | --- | --- | --- |
| Scraping, web browsing (TCP/HTTP) | **HTTPS** | Any | Use `proxy_config_result.https_proxy_url`. No username or password needed. |
| Low-level sockets, UDP | **SOCKS5** | **Pro** | Use `proxy_config_result.socks_proxy_url`, or `proxy_host` + `socks_proxy_port`. Username is `proxy_config_result.auth_token`; password is empty. Remote DNS is supported (socks5h). |
| System-wide / all IP packets | **WireGuard** | **Pro** | Set `proxy_config.enable_wg: true` in the request. Use `proxy_config_result.wg_config.config` as a complete WireGuard config file. |

Prefer HTTPS unless you specifically need UDP/raw sockets (SOCKS) or full system routing (WireGuard). Only fall back to the HTTP proxy if the target library genuinely cannot do HTTPS — switching to a library that supports HTTPS is the better fix.

**SOCKS and WireGuard require UR Pro.** On a free plan they are simply not issued: `socks_proxy_url` comes back empty and `wg_config` comes back null, with no error. Do not retry — that is a plan limit, not a transient failure. See [Handling errors](#handling-errors).

## Select a location

Locations are always selected with a **`connect_location_id`**. There is no `country_code` field on `initial_device_state` — passing one is silently ignored and you will get a proxy in the wrong place.

`connect_location_id` takes exactly one of:

| Field | Selects |
| --- | --- |
| `location_id` | A country, region, or city (from `/network/find-provider-locations`). |
| `client_id` | One specific provider (one specific egress IP). |
| `location_group_id` | A location group. |
| `best_available: true` | Anywhere; let the network choose. |

Search with `/network/find-provider-locations`, then **filter the `locations` array by `location_type`** (`country`, `region`, or `city`) so the `location_id` matches what the user actually asked for. A query for "Japan" returns cities and regions too.

| `location_type` | Covers |
| --- | --- |
| `country` | Countries. |
| `region` | States, provinces, administrative regions, metro areas. |
| `city` | Cities. |

`location_id` values are stable — save them in code instead of re-searching.

## Recipe: many distinct egress IPs (distributed scraping)

Enumerate the individual providers in a location and create one proxy per provider.

```bash
# 1. find the location, take its location_id (as above)

# 2. list providers (egress IPs) IN that location. Note: specs takes the
#    location_id -- `count` is how many distinct providers you want back.
curl -s -X POST https://api.bringyour.com/network/find-providers2 \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"specs": [{"location_id": "<LOCATION ID>"}], "count": 10}' | jq '.providers'

# 3. for each provider client_id, create a proxy pinned to that ONE egress IP
curl -s -X POST https://api.bringyour.com/network/auth-client \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"proxy_config": {"initial_device_state": {"location": {"connect_location_id": {"client_id": "<CLIENT ID>"}}}}}'
```

`client_id` values are stable and can be saved. Each proxy you create is a concurrent client — see the limits below.

## Handling errors

`/network/auth-client` reports plan limits as an HTTP 200 with an `error` object. If you ask for x402 terms (`X-UR-X402: 1`, see below), a limit that UR Pro lifts answers HTTP 402 instead. Handle both.

### 1. HTTP 200 with an `error` object

```json
{ "error": { "client_limit_exceeded": true, "upgrade_required": true, "message": "..." } }
```

| Response | Meaning | What to do |
| --- | --- | --- |
| `upgrade_required: true` | A **plan** limit that UR Pro lifts: the free plan's 5 active proxies or 5 new proxies a minute (UR Pro: 1000 and 20 a minute), its concurrent client limit, or a Pro-only feature. The `message` says which. | Upgrade. Either tell the human, or pay for it yourself — see [Paying with x402](#paying-with-x402). Do **not** retry in a loop: the proxy cap holds until a proxy is removed, and the rate until the next minute. |
| `client_limit_exceeded: true` **without** `upgrade_required` | A **hard cap** on provisioned clients. Upgrading does not lift it. | Release clients you are no longer using, then retry. |
| `socks_proxy_url` empty / `wg_config` null, no error | SOCKS and WireGuard are **Pro-only** and were not issued. | Upgrade, or use the HTTPS proxy instead. Do not retry. |

Every proxy you create counts as a concurrent client, so a scraping fan-out is the usual way to hit the limit. Release proxies you have finished with.

### 2. HTTP 402 Payment Required

Only when you asked for it: the request carried `X-UR-X402: 1`. The server is quoting you a price, in both x402 protocol versions at once:

- **x402 v2:** the `PAYMENT-REQUIRED` response header, base64 of this JSON. Networks are CAIP-2 ids (`eip155:8453` is Base, `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` is Solana).

```json
{
  "x402Version": 2,
  "error": "<why>. UR Pro, 1 month costs $5.00 in USDC. x402 clients refuse a payment over $1 by default: raise your client's per-payment cap to $5.00 or more to pay (see https://ur.io/agents.md).",
  "resource": { "url": "https://api.bringyour.com/network/auth-client",
                "description": "UR Pro, 1 month", "mimeType": "application/json" },
  "accepts": [
    { "scheme": "exact", "network": "eip155:8453", "amount": "5000000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "payTo": "<merchant>",
      "maxTimeoutSeconds": 300, "extra": { "name": "USD Coin", "version": "2" } },
    { "scheme": "exact", "network": "solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp", "amount": "5000000", "...": "..." }
  ]
}
```

- **x402 v1:** the body, the same terms with networks by name (`base`, `solana`), `maxAmountRequired` instead of `amount`, and each term's own `resource`.

An x402 client library reads whichever version it speaks. Pay it and retry — see below.

## Paying with x402

x402 lets an agent buy an upgrade inline, with no human and no checkout page. It is **chain-neutral**: one entry in `accepts[]` per supported chain, and you pick whichever you hold funds on.

**The flow is: request with `X-UR-X402: 1` → 402 → pay → retry the SAME request with the signed payment in a `PAYMENT-SIGNATURE` header (x402 v2) or an `X-PAYMENT` header (x402 v1).** Send one payment header, not both.

x402 has no header a client sends before it is quoted, so the server quotes only a request that asks with `X-UR-X402: 1`. Without it a plan limit is the HTTP 200 with `upgrade_required` above, never a 402, and you get the terms from `/x402/purchase` instead (below).

```bash
# ask for terms: a plan limit answers 402 with them
curl -s -X POST https://api.bringyour.com/network/auth-client \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -H 'X-UR-X402: 1' \
  -d '{"proxy_config": {...}}'

# the same call that returned 402, now with a signed x402 v2 payment
curl -s -X POST https://api.bringyour.com/network/auth-client \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -H 'X-UR-X402: 1' \
  -H "PAYMENT-SIGNATURE: <base64 signed x402 v2 payload>" \
  -d '{"proxy_config": {...}}'
```

On success you get `200` plus the settlement receipt in a `PAYMENT-RESPONSE` header (`X-PAYMENT-RESPONSE` for an x402 v1 payment). The upgrade takes effect immediately, so the retried request simply succeeds. A payment that is refused comes back as another 402 with the terms, the reason in their `error`; nothing was charged for it. A malformed payment is a 400.

Amounts are integer atomic units of the asset: USDC has 6 decimals, so `"5000000"` is **$5.00**.

### Raise your client's per-payment cap first

x402 clients refuse a payment over $1 by default, and every sku but `data_256gib` costs more (UR Pro is $5 a month), so a default client refuses those terms before it signs anything. Set the cap to at least the price of what you buy:

```
// TypeScript (@x402/core, @x402/fetch, @x402/axios, @x402/mcp)
spendControls: { maxAmountPerPayment: "$5" }
// Go (github.com/x402-foundation/x402/go/v2)
client.SetSpendControls(x402.SpendControls{MaxAmountPerPayment: "$5"})
# Python (x402)
spend_controls={"max_amount_per_payment": "$5"}
// Coinbase AgentKit (or X402_MAX_PAYMENT_USDC=5); it pays only registered services
x402ActionProvider({ maxPaymentUsdc: 5, registeredServices: ["https://api.bringyour.com"] })
// v1 x402-fetch: maxValue in atomic USDC (the default is $0.10)
wrapFetchWithPayment(fetch, walletClient, BigInt(5_000_000))
```

Keep the cap no higher than `URNETWORK_X402_MAX` (below). The 402's `error` and `/x402/skus` (`price_usd` and the atomic `amount` of each sku) say the price.

**A client with a $1 cap it cannot raise** (Cloudflare Agents' `withX402Client` builds its x402 client without spend controls, so it refuses anything over $1) can buy `data_256gib`: 256 GiB of data for $0.75, atomic amount `750000`. It is the pack for clients with a $1 per-payment cap. A default x402 SDK client or Coinbase AgentKit pays it as it is; a client that defaults to $0.10 (the v1 `x402-fetch`'s `maxValue`, `withX402Client`'s `maxPaymentValue`) needs its cap set to at least `750000`. It is data only: it does not lift a plan limit, so an `upgrade_required` refusal still needs `pro_1month`.

### What you can buy

```bash
curl -s https://api.bringyour.com/x402/skus   # public; no JWT needed
```

| `sku_id` | What it grants |
| --- | --- |
| `pro_1month` | UR Pro for one month: raises the proxy cap and create rate and the concurrent client limit, and enables SOCKS + WireGuard. |
| `data_1tib` | 1 TiB of data. **Data only — this does not grant Pro.** |
| `data_10tib` | 10 TiB of data. **Data only — this does not grant Pro.** |
| `data_256gib` | 256 GiB of data for $0.75: the pack for clients with a $1 per-payment cap. Sold only over x402 and with Pay with Gamma. **Data only — this does not grant Pro.** |

Buying data does **not** lift a concurrent-client limit and does **not** enable SOCKS or WireGuard. If you got `upgrade_required`, you need `pro_1month`.

To buy something directly rather than waiting to be quoted (for example after an `upgrade_required` you did not ask to be quoted for; then retry the refused request):

```bash
# no payment -> answers 402 with the terms for this sku
curl -s -X POST https://api.bringyour.com/x402/purchase \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"sku_id": "pro_1month", "email": "receipts@example.com"}'

# with a signed payment (PAYMENT-SIGNATURE, or X-PAYMENT in x402 v1) -> settles and grants
curl -s -X POST https://api.bringyour.com/x402/purchase \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -H "PAYMENT-SIGNATURE: <base64 signed x402 v2 payload>" \
  -d '{"sku_id": "pro_1month", "email": "receipts@example.com"}'
```

`email` is optional; supply it to get a receipt.

### Spend safely

- **Honor a hard cap.** If `URNETWORK_X402_MAX` is set (a USD ceiling), never pay above it. Stop and ask the human instead.
- **Never pay twice for the same 402.** Settlement is not idempotent from your side — retry the request once with the payment, and if it still fails, stop and report rather than paying again. If the refusal's reason is `settlement_pending`, your payment was broadcast and may still confirm: do not sign another one.
- **Pay only terms the server quoted.** Use an entry from `accepts[]` verbatim; never construct your own `payTo` or amount.
- If `/x402/skus` returns 404, x402 is not enabled on this deployment. Fall back to telling the human to upgrade.
