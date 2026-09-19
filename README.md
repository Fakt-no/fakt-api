# fakt API

Public API for Norwegian labour-market data — job ads, employers, salary, recruitment patterns and market insights.

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

> **`YOUR_API_KEY_HERE` is a placeholder.** Replace it with a real key from your dashboard. Every request needs a key — there is no anonymous access.

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

1. **Get a key** (required — free, no payment details): create one from [fakt.no/dashboard/innstillinger](https://fakt.no/dashboard/innstillinger).
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

Bulk exports (`/export/*`) are charged **by volume, not per request**: every 25 rows returned
cost one call of quota (rounded up, minimum 1). A full-corpus export of ~13,600 jobs is
therefore ~546 calls, not 1. Exports are also capped in concurrency — a second export on the
same key, or a third server-wide, gets `429` with `Retry-After`.

| Plan | Per minute | Per day | Per month | Endpoints |
| --- | --- | --- | --- | --- |
| **Free** | 10 | 50 | 1,500 | Core |
| **Pro** | 60 | 1,000 | 30,000 | Core + Market |
| **Business** | 300 | 10,000 | 300,000 | Core + Market + Exports |
| **Enterprise** | 1,000 | 100,000 | 3,000,000 | Core + Market + Exports |

**Feature groups**

- **Core** — `/jobs`, `/employers`, `/events` (available to every plan, starting with Free)
- **Market** — salary and market-intelligence endpoints (`/market/*`, `/recruitment`)
- **Exports** — bulk NDJSON export (`/export/*`)

When a daily or monthly quota, the per-minute burst limit or the export concurrency limit is
reached the API returns `429`, with the relevant `X-*` headers plus `Retry-After` for exports
(see [Response headers](#response-headers--errors)).

---

## Endpoints

| Method | Path | Description | Access |
| --- | --- | --- | --- |
| GET | `/jobs` | Search active jobs (FTS, county, category, employment type, salary, postal + radius) | Core |
| GET | `/jobs/{id}` | Full enriched job: description, tags, salary + expected salary (SSB), history | Core |
| GET | `/jobs/{id}/similar` | Recommended jobs based on occupation/category/location/skills | Core |
| GET | `/employers` | Employer list with aggregates (open, salary share, avg quality) | Core |
| GET | `/employers/{name}` | Employer profile: open jobs, monthly timeline, Brønnøysund registry + recruitment pattern | Core |
| GET | `/events` | Change log (append-only event stream), keyset pagination | Core |
| GET | `/stream` | SSE live stream of the change log | Core |
| GET | `/market` | Market KPIs: active jobs, new today (Oslo time), salary share, repeats | Market |
| GET | `/market/salary` | Salary breakdown by category/occupation/county: median, p25/p75, min/max, distribution | Market |
| GET | `/market/history` | Point-in-time market history: active + new jobs per day | Market |
| GET | `/market/timetofill` | Time-to-fill from measured intervals, by category and county | Market |
| GET | `/recruitment` | Employer recruitment patterns (six explainable signals → persistence score) | Market |
| GET | `/recruitment/audit` | Algorithm validation: drift check + signal distributions | Market |
| GET/POST | `/watchlists` | List / create saved searches | Core |
| GET/DELETE | `/watchlists/{id}` | New jobs since last check / delete a saved search | Core |
| GET | `/usage` | Your quota & usage status | Core |
| GET | `/export/jobs` | Bulk NDJSON export of jobs | Exports |
| GET | `/export/employers` | Bulk NDJSON export of employers (incl. registry + patterns) | Exports |
| GET | `/export/events` | Bulk NDJSON export of the change log | Exports |

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
| `400` | Bad request |
| `401` | Missing `X-API-Key` (`code: api_key_required`) or invalid/revoked key |
| `403` | Key expired or endpoint not on your plan |
| `404` | Resource not found |
| `429` | Quota exhausted (monthly, daily or per-minute) |

Response headers: `X-Plan`, `X-Quota-Limit`, `X-Quota-Remaining`, `X-Daily-Limit`,
`X-Daily-Remaining` (keyed calls), `X-Key-Required: true` (the keyless 401) and
`X-Export-Rows` / `X-Export-Units` (bulk exports).

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
