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

> **`YOUR_API_KEY_HERE` is a placeholder.** Replace it with a real key from your dashboard, or omit the header entirely to use the open **demo** tier.

---

## Authentication

The API uses a single API key sent in the `X-API-Key` header.

| Mode | How | Limit |
| --- | --- | --- |
| **Demo (no key)** | Call without a header | Read-only, per-IP **50 requests/day** (see [Rate limits](#rate-limits)) |
| **Keyed** | Send `X-API-Key: <key>` | Plan-based monthly + daily quota |

Create a key from [fakt.no/dashboard](https://fakt.no/dashboard) → **API keys**. Keys are shown once at issue time (stored hashed) — keep it secret.

```bash
# Demo (no key)
curl "https://fakt.no/api/v1/jobs?q=sykepleier&limit=5"

# Keyed
curl -H "X-API-Key: YOUR_API_KEY_HERE" \
  "https://fakt.no/api/v1/jobs?q=sykepleier&limit=5"
```

---

## Getting started

1. **Get a key** (optional but recommended): create one from [fakt.no/dashboard](https://fakt.no/dashboard).
2. **Call an endpoint** with `curl` (or any HTTP client) against `https://fakt.no/api/v1`.
3. **Check your quota** at any time with [`GET /usage`](#endpoints).
4. **Generate a client** from [`openapi.yaml`](./openapi.yaml) if you want typed SDKs.

### Python

```python
import requests

BASE = "https://fakt.no/api/v1"
HEADERS = {"X-API-Key": "YOUR_API_KEY_HERE"}   # omit the header for demo mode

r = requests.get(f"{BASE}/jobs", headers=HEADERS, params={"q": "elektriker", "county": "Rogaland"})
data = r.json()
for job in data.get("items", []):
    print(job["title"], "—", job.get("location"))
```

---

## Rate limits

Limits are enforced per key (per IP for demo). Plan values match the [fakt.no pricing](https://fakt.no) tiers.

| Plan | Per minute | Per day | Per month | Endpoints |
| --- | --- | --- | --- | --- |
| **Demo** (no key) | 10 | 50 (per IP) | ~1,500 | Core (read-only) |
| **Free** | 10 | 50 | 1,500 | Core |
| **Pro** | 60 | 1,000 | 30,000 | Core + Market |
| **Business** | 300 | 10,000 | 300,000 | Core + Market + Exports |
| **Enterprise** | 1,000 | 100,000 | 3,000,000 | Core + Market + Exports |

**Feature groups**

- **Core** — `/jobs`, `/employers`, `/events` (available to every plan, including demo)
- **Market** — salary and market-intelligence endpoints (`/market/*`, `/recruitment`)
- **Exports** — bulk NDJSON export (`/export/*`)

When a daily or monthly quota is exhausted the API returns `429`, with the relevant `X-*` headers (see [Response headers](#response-headers--errors)).

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
| `401` | Invalid / expired `X-API-Key` |
| `403` | Key expired or endpoint not on your plan |
| `404` | Resource not found |
| `429` | Quota exhausted (monthly, daily or per-minute) |

Response headers: `X-Plan`, `X-Quota-Limit`, `X-Quota-Remaining`, `X-Daily-Limit`, `X-Daily-Remaining`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`.

Errors return JSON, e.g.:

```json
{ "error": "Demo limit reached (50 req/day per IP). Get an API key for full access" }
```

---

## Data & methodology

- **Sources:** NAV Arbeidsplassen, SSB, Brønnøysundregistrene.
- **Methodology & data sources:** [fakt.no/metodikk](https://fakt.no/metodikk)
- **About fakt:** [fakt.no/about](https://fakt.no/about)

## License

MIT — see [`LICENSE`](./LICENSE).
