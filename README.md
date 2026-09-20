# fakt API

Public API for Norwegian labour-market data, job ads, employers, salary, recruitment patterns and market insights.

> fakt continuously observes the official **NAV Arbeidsplassen** feed for job postings and builds history over time. Data is combined with **SSB** (salary / income) and the **Brønnøysundregistrene** (company registry). All figures are **observed**, not self-reported.

- **Base URL:** `https://fakt.no/api/v1`
- **OpenAPI 3.0:** [`openapi.yaml`](./openapi.yaml) — the authoritative, machine-readable full reference
- **Latest data:** [fakt.no](https://fakt.no) · [Innsikt](https://fakt.no/insights)

---

## Quick start

```bash
curl -H "X-API-Key: YOUR_API_KEY_HERE" \
  "https://fakt.no/api/v1/jobs?q=sykepleier&county=Oslo&limit=2"
```

<details>
<summary>Example response</summary>

```json
{
  "count": 2,
  "total": 1477,
  "offset": 0,
  "limit": 2,
  "sort": "newest",
  "items": [
    {
      "id": "cmsx8av5o1hmc4lzdinoyw55w",
      "title": "Sykehjemslege",
      "employer": "Bjølsenhjemmet",
      "employerId": "cmssolric07ebb8gfqd9ie0oo",
      "location": "OSLO",
      "category": "Allmennpraktiserende leger",
      "occupation": "Allmennpraktiserende leger",
      "employmentType": "Vikariat",
      "publishedAt": "2026-09-05T05:05:28.473Z",
      "salaryDisclosed": false,
      "qualityScore": 55,
      "tone": "mid"
    }
  ]
}
```
</details>

> **`YOUR_API_KEY_HERE` is a placeholder.** Replace it with a real key from your dashboard. Every request needs a key, there is no anonymous access.

---

## Authentication

The API uses a single API key sent in the `X-API-Key` header. **A key is required for
every request — there is no anonymous access.** A request without a key (or with an invalid
or expired one) gets `401` with code `api_key_required` and instructions for creating one.
Only `GET /api/v1` (the catalog) and `GET /api/v1/openapi` are public metadata, so a new
caller can discover how to get a key.

| How | Limit |
| --- | --- |
| `X-API-Key: <key>` (required) | Plan-based monthly + daily quota and a per-plan per-minute burst limit (see [Rate limits](#rate-limits)) |

The **Free** plan costs nothing and requires **no payment details**: create an account at
[fakt.no/signup](https://fakt.no/signup), then create a key from
[fakt.no/dashboard/innstillinger](https://fakt.no/dashboard/innstillinger) → **API keys**.
Keys are shown once at issue time (stored hashed) — keep it secret.

```bash
# Keyed (the only way to call the API)
curl -H "X-API-Key: YOUR_API_KEY_HERE" \
  "https://fakt.no/api/v1/jobs?q=sykepleier&limit=5"

# Without a key -> 401 {"error":"API key required (X-API-Key). This API has no anonymous access.", "code":"api_key_required", ...}
curl -i "https://fakt.no/api/v1/jobs?q=sykepleier&limit=5"
```

---

## Getting started

1. **Get a key** (required, free, no payment details): create one from [fakt.no/dashboard/innstillinger](https://fakt.no/dashboard/innstillinger).
2. **Call an endpoint** with `curl` (or any HTTP client) against `https://fakt.no/api/v1`.
3. **Check your quota** at any time with [`GET /usage`](#endpoints).
4. **Generate a client** from [`openapi.yaml`](./openapi.yaml) if you want typed SDKs.

### Python

```python
import requests

BASE = "https://fakt.no/api/v1"
HEADERS = {"X-API-Key": "YOUR_API_KEY_HERE"}   # required; free key from the dashboard

r = requests.get(f"{BASE}/jobs", headers=HEADERS, params={"q": "elektriker", "county": "Rogaland"})
data = r.json()
for job in data.get("items", []):
    print(job["title"], "—", job.get("location"))
```

---

## Rate limits

Limits are enforced per key. Plan values match the [fakt.no pricing](https://fakt.no) tiers.

The per-minute burst limit is counted **per API key**, so several keys behind one office/NAT
address each get their own bucket. A separate, deliberately high per-IP wall (1,200 requests
per minute across all keys by default) additionally guards against one address rotating through
many keys, and answers `429 {"code":"rate_limited_per_ip"}`.

Bulk exports (`/export/*`) are charged **by volume, not per request**: every 25 rows returned
cost one call of quota (rounded up, minimum 1). A full-corpus export of ~13,600 jobs is
therefore ~550 calls, not 1. Exports are also capped in concurrency — a second export on the
same key, or a third server-wide, gets `429` with `Retry-After`. Since 1.5.0 a self-service
export is also **bounded**: see [Record-exposure limits](#record-exposure-limits).

| Plan | Per minute | Per day | Per month | Endpoints |
| --- | --- | --- | --- | --- |
| **Free** | 10 | 50 | 1,500 | Core |
| **Pro** | 60 | 1,000 | 30,000 | Core + Market |
| **Business** | 300 | 10,000 | 300,000 | Core + Market + Recruitment + Bounded exports |
| **Enterprise** | 1,000 | 100,000 | 3,000,000 | Core + Market + Recruitment + Exports (data licence: full dataset) |

### Record-exposure limits

Request quotas are not the protection against rebuilding the database — this is. Every key is
metered on **unique records delivered per customer account per calendar month**, counted
separately for jobs, employers, event rows and ad texts.

* **Shared by the account.** All API keys belonging to one customer share one budget. Creating
  more keys does **not** add allowance.
* **Already-delivered records are free.** Re-requesting a record the account has already
  received consumes nothing, the budget measures how much of the corpus the account has seen,
  not how many requests it made.
* **Aggregates are not metered.** Market, salary, recruitment, time-to-fill and every other
  derived/intelligence endpoint has no record budget.

| Plan | Unique jobs | Unique employers | Event rows | Ad texts |
| --- | --- | --- | --- | --- |
| **Free** | 750 | 500 | 1,000 | 100 |
| **Pro** | 4,000 | 2,500 | 20,000 | 500 |
| **Business** | 12,000 | 8,000 | 100,000 | 5,000 |
| **Enterprise** | 60,000 | 40,000 | 300,000 | 25,000 |
| **Enterprise + data licence** | not metered | not metered | not metered | not metered |

For reference, the corpus is ~13,600 live ads (~56,800 including removed ones), ~31,250
employers and ~828,000 change-log rows, so no self-service plan can cover it.

Every metered response carries `X-Exposure-<Kind>-Used` / `-Limit` / `-Remaining`
(`Kind` = `Job`, `Employer`, `Event`, `Description`). `GET /usage` reports the whole budget of
the account. Exceeding one returns:

```json
{ "error": "Record-exposure budget exhausted for this customer account ...",
  "code": "record_exposure_limit", "kind": "job", "used": 750, "limit": 750, "resets": "2026-10-01" }
```

**Honest limitation.** If Fakt returns a raw record to a customer, Fakt cannot prevent the
customer from storing it. This protection therefore limits **cumulative raw-record exposure**;
it does not make copying impossible.

### Page size, filters and cursors

* **A page is at most 100 records.** An oversized `?limit=` (e.g. `limit=100000`) is
  `400 {"code":"limit_too_large","max":100,"requested":100000}` — never a silently truncated
  page. `?offset=` above 1000 is `400 offset_too_deep`.
* **Large record endpoints require a meaningful filter** on a fresh call: `/jobs` accepts `q`,
  `occupation`, `category`, `county`, `postal` (+`radius_km`), `employerId`, `employmentType`,
  `publishedAfter`; `/employers` accepts `q`, `orgNumber`, `county`; `/events` accepts `event`,
  `since`. Without one: `400 {"code":"filters_required","accepted":[...]}`.
* **Cursors are opaque, signed, account-bound and expire after 24 h.** `?after=<nextCursor>`
  continues a walk (no filter needed again). A cursor cannot be minted for an arbitrary
  position, used from another account (`400 cursor_wrong_account`), used on another endpoint, or
  resumed after it expires.

### Harvesting detection

Walking the whole dataset in a systematic way, very long cursor walks, sweeping one filter
dimension, repeating one broad query, many overlapping queries or unusually high budget
pressure, is detected. Several independent signals must co-occur before the account is
throttled for 15 minutes with
`429 {"code":"enumeration_suspected","signals":[...],"retryAfterSeconds":900}`. A single signal
never affects a normal customer, this is never a ban, and every event is recorded.

**Feature groups**

- **Core** — `/jobs`, `/employers`, `/events`, `/watchlists`, `/status`, `/changelog`
  (available to every plan, starting with Free)
- **Market**, salary and market analytics (`/market`, `/market/salary`, `/market/history`,
  `/market/timetofill`) and Fakt's model salary estimate (`salaryEstimatedMin/Median/Max`,
  `salaryBenchmark`) on `/jobs/{id}`. The *advertised* salary of an individual ad is Core: it is
  returned by `/jobs` and `/jobs/{id}` on every plan.
- **Recruitment**, employer recruitment analytics (`/recruitment`, `/recruitment/audit`, and
  the `recruitment` block of `/employers/{name}`)
- **Exports**, bounded bulk export (`/export/*`), NDJSON or CSV. Self-service: max 1000 rows per
  request, `?updatedSince=` required for `/export/jobs`, `include=description` requires a data
  licence. Unrestricted full-dataset export is the separately licensed Enterprise product.

The minimum plan for each operation is also machine-readable: `x-plan` in
[`openapi.yaml`](./openapi.yaml) and `plan` in the catalog at `GET /api/v1`.

When a daily/monthly quota, the per-minute burst limit, a record-exposure budget, the export
concurrency limit or the anti-harvesting detector is hit the API returns `429` with the relevant
`X-*` headers plus `Retry-After` (see [Response headers](#response-headers--errors)).

---

## Endpoints

| Method | Path | Description | Access |
| --- | --- | --- | --- |
| GET | `/jobs` | Search active jobs (FTS, county, category, occupation, employment type, employerId, publishedAfter, postal + radius). **A filter is required** (`400 filters_required`). Page max 100; signed `nextCursor` (24 h). Response carries the advertised salary band (`salaryMin`/`salaryMax`/`salaryDisclosed`). Metered on unique jobs | Core |
| GET | `/jobs/{id}` | Full enriched job: description, tags, history, advertised salary (`salaryDisclosed`, `salaryText`, `salaryActualText`, `salaryMin`/`salaryMax` — every plan, identical to `/jobs`). Fakt's model estimate (`salaryEstimatedMin/Median/Max`, `salaryBenchmark`) requires Pro; `salaryEstimatesAvailable` says whether it is present. Metered on the job and, separately, the ad text | Core / **Pro** for the estimate |
| GET | `/jobs/{id}/similar` | Recommended jobs based on occupation/category/location/skills. Metered on the job ids returned | Core |
| GET | `/employers` | Employer search: id, name, orgNumber, number of open ads. **A filter is required** (`q`, `orgNumber`, `county`). Page max 100. Metered on unique employers | Core |
| GET | `/employers/{name}` | Employer profile: open jobs, monthly timeline, salary share, Brønnøysund registry. `recruitment` (score + evidence) requires Business (`recruitmentAvailable` says whether it is present) | Core / **Business** |
| GET | `/events` | Change log (append-only event stream), signed keyset pagination. **`event=` or `since=` is required**; unknown event types are `400 invalid_event`. The classifier context is not returned. Metered on event rows and job ids | Core |
| GET | `/stream` | Disabled — returns `410 Gone`. Use `/events` | — |
| GET | `/market` | Market KPIs: active jobs, new today (Oslo time), salary share, repeats | Market |
| GET | `/market/salary` | Salary breakdown by category/occupation/county: median, p25/p75, min/max, distribution | Market |
| GET | `/market/history` | Point-in-time market history: active + new jobs per day | Market |
| GET | `/market/timetofill` | Time-to-fill from measured intervals, by category and county | Market |
| GET | `/recruitment` | Employer recruitment patterns (six explainable signals → persistence score, with per-employer evidence). The model's own weights are not returned. Metered on unique employers | **Business** |
| GET | `/recruitment/audit` | Algorithm validation: drift check + signal distributions | **Business** |
| GET/POST | `/watchlists` | List / create saved searches (scoped to your key) | Core |
| GET/DELETE | `/watchlists/{id}` | New jobs since last check / delete a saved search. Another key's watchlist returns `404` | Core |
| GET | `/usage` | Your quota, usage and the **record-exposure budget of your account** | Core |
| GET | `/status` | API and pipeline health (no key required) | Public |
| GET | `/changelog` | What changed in the API, and when (no key required) | Public |
| GET | `/export/jobs` | Bounded job export: NDJSON (default) or CSV, max 1000 rows, `?updatedSince=<ISO>` required, `?include=salaryEstimated,salaryActualText,aiMentions` (`description` needs a data licence) | Exports |
| GET | `/export/employers` | Bounded NDJSON export of employers (registry fields), max 1000 rows, no cursor | Exports |
| GET | `/export/events` | Bounded NDJSON export of the change log, max 1000 rows, `?since=<ISO>` window | Exports |

Full parameter and response schemas are in [`openapi.yaml`](./openapi.yaml).

---

## Common queries

**Newest jobs in a category:**

```bash
curl "https://fakt.no/api/v1/jobs?category=Forsker&sort=newest&limit=10"
```

**Jobs near you:**

```bash
curl "https://fakt.no/api/v1/jobs?q=sykepleier&postal=0150&radius_km=10&sort=distance&limit=10"
```

**Market overview + history:**

```bash
curl "https://fakt.no/api/v1/market"
curl "https://fakt.no/api/v1/market/history?days=30"
```

**Employer lookup** (returns employers A–Z by name — `/employers` has no `sort` parameter):

```bash
curl "https://fakt.no/api/v1/employers?q=sykehus&limit=10"
```

---

## Response headers & errors

The API uses standard HTTP status codes. Simple read endpoints return JSON arrays/objects; bulk exports return NDJSON (`application/x-ndjson`); the stream returns `text/event-stream`.

| Status | Meaning |
| --- | --- |
| `200` | OK |
| `400` | Bad request — `limit_too_large`, `invalid_limit`, `filters_required`, `offset_too_deep`, `cursor_invalid` / `cursor_expired` / `cursor_wrong_account` / `cursor_wrong_kind`, `invalid_event`, `invalid_since`, `export_window_required`, `invalid_include`, `invalid_format`, `invalid_updated_since` |
| `401` | Missing `X-API-Key` (`code: api_key_required`) or invalid/revoked key |
| `403` | Key expired, endpoint not on your plan, or `field_requires_data_license` |
| `404` | Resource not found |
| `429` | Quota exhausted (monthly, daily, per-minute), `record_exposure_limit`, `enumeration_suspected`, export concurrency |

Response headers: `X-Plan`, `X-Quota-Limit`, `X-Quota-Remaining`, `X-Daily-Limit`,
`X-Daily-Remaining` (keyed calls), `X-Exposure-<Kind>-Used` / `-Limit` / `-Remaining`
(metered record endpoints), `X-Key-Required: true` (the keyless 401),
`X-Export-Rows` / `X-Export-Units` / `X-Export-Format` / `X-Export-Max-Rows` /
`X-Export-Data-License` (exports), `X-Export-Concurrency-Limit` and `Retry-After` (export
concurrency, exposure budget and harvesting throttle) and `X-Required-Feature` (a `403` for a
plan the key does not have).

`GET /`, `/openapi`, `/market` and `/changelog` send a weak `ETag` and honour
`If-None-Match` with `304 Not Modified`. `GET /status` also sends an `ETag`, but its body
contains `servedAt` and a live database latency and therefore changes on every call, so it is
not 304-cacheable. `Cache-Control` on v1 responses is `private`, so no
shared cache can mix two keys' quota headers.

**Browser use (CORS).** Cross-origin access is off unless the operator has allow-listed your
origin (`V1_CORS_ORIGINS` on the server). When allowed, responses carry
`Access-Control-Allow-Origin` and the `OPTIONS` preflight is answered with `204`; the exposed
headers are `ETag`, `X-Plan`, `X-Quota-*`, `X-Daily-*`, `X-Export-*` and `Retry-After`.

---

## Metric definitions

- **Disclosed salary** — an ad discloses salary when the salary parser accepted its NAV salary
  field (`salaryActualText`) or a normalized monthly-FTE figure exists (`salaryActualMin` /
  `salaryActualMax`, inside 15 000–150 000 NOK/month). `salaryDisclosed`,
  `salaryDisclosedCount` and `salaryDisclosedPct` all use this definition.
  `Job.salaryDisclosed` / `Job.salaryMin` / `Job.salaryMax` are legacy columns that are no
  longer written and are used by no metric.
- **`salaryMin` / `salaryMax`** on a job are the **advertised** monthly band in NOK/month; the
  two ends fall back to each other for a single-sided ad.
- **`salaryEstimatedMin` / `Median` / `Max`, `salaryBenchmark`, `salaryConfidence`** are Fakt's
  model estimate (historical NAV comparables, SSB benchmarks, sector/region adjustment). They
  are not the employer's disclosure.
- **Time-to-fill** — observed ad lifetime in days (published → now for open ads,
  published → last seen for closed ads). It measures how long ads stay listed, not an
  employer-reported fill date.
- **Recruitment `persistenceScore`** — 0–100 employer score from six weighted signals:
  repost .25, duration .20, similarity .15, reactivation .15, frequency .15, duplicate .10
  (`recruit-1.1.6`).
- **`aiMentions` / the KI metric** — counts ads whose **own text** (title or description)
  mentions AI/KI terms (KI, AI, kunstig intelligens, maskinlæring, GPT, LLM, Claude, Copilot,
  generativ, prompt, chatbot, språkmodell, nevralt nettverk). It is a **detector of term
  mentions in the ad text**. It is **not** an AI-authorship signal, and must not be used or
  described as one.
- **History depth** — collection started 2026-08-11. Snapshots, the event log and daily
  statistics begin there; `publishedAt` on collected ads goes back to 2019. `/market/history`
  serves up to 120 days.

Errors return JSON, e.g.:

```json
{
  "error": "API key required (X-API-Key). This API has no anonymous access.",
  "code": "api_key_required",
  "howTo": "Create a free account at https://fakt.no/signup (no payment required), create an API key on your dashboard at https://fakt.no/dashboard/innstillinger, then send it as the X-API-Key header.",
  "signup": "https://fakt.no/signup",
  "dashboard": "https://fakt.no/dashboard/innstillinger",
  "catalog": "https://fakt.no/api/v1"
}
```

---

## Data & methodology

- **Sources:** NAV Arbeidsplassen, SSB, Brønnøysundregistrene.
- **Methodology & data sources:** [fakt.no/metodikk](https://fakt.no/metodikk)
- **About fakt:** [fakt.no/about](https://fakt.no/about)

## License

MIT — see [`LICENSE`](./LICENSE).
