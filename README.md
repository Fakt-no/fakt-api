# fakt API

Offentlig API for norsk arbeidsmarkedsdata — stillinger, arbeidsgivere, lønn, rekrutteringsmønstre og markedsinnsikt.

> fakt observerer den offisielle **NAV Arbeidsplassen**-feeden for stillingsannonser kontinuerlig og bygger historikk over tid. Data kombineres med **SSB** (lønn/inntekt) og **Brønnøysundregistrene** (arbeidsgiverinfo). Alle tall er **observert**, ikke selvrapportert.

- **OpenAPI 3.0:** [`openapi.yaml`](./openapi.yaml)
- **Base URL:** `https://fakt.no/api/v1`
- **Siste stillingsdata:** [fakt.no](https://fakt.no) · [Innsikt](https://fakt.no/insights)

---

## Kom i gang

### 1. Få en API-nøkkel (valgfritt)

API-en kjører i to moduser:

| Modus | Autentisering | Begrensning |
| --- | --- | --- |
| **Demo (open)** | Ingen nøkkel | Read-only, per-IP døgnkvote (standard 50 kall/dag) |
| **Pro (nøkkel)** | `X-API-Key` | Planbasert månedskvote, døgnkvote og funksjonstilgang |

Opprett en nøkkel fra [fakt.no/dashboard](https://fakt.no/dashboard) (API-nøkler). Uten nøkkel kan du fortsatt bruke alle lese-endepunktene med demokvoten.

### 2. Kall en endepunkt

```bash
# Demo (ingen nøkkel)
curl "https://fakt.no/api/v1/jobs?q=sykepleier&limit=5"

# Med API-nøkkel
curl -H "X-API-Key: din_nøkkel" \
  "https://fakt.no/api/v1/jobs?q=sykepleier&limit=5"
```

### 3. Python

```python
import requests

BASE = "https://fakt.no/api/v1"
HEADERS = {"X-API-Key": "din_nøkkel"}          # utelat for demo-modus

r = requests.get(f"{BASE}/jobs", headers=HEADERS, params={"q": "elektriker", "county": "Rogaland"})
data = r.json()
for job in data.get("jobs", []):
    print(job["title"], "—", job.get("location"))
```

---

## Endpunkter

| Metode | Sti | Beskrivelse |
| --- | --- | --- |
| GET | `/jobs` | Søk i aktive stillinger (fulltekst, fylke, kategori, postnummer, radius) |
| GET | `/jobs/{id}` | Enkeltannonse med målt historikk |
| GET | `/jobs/{id}/similar` | Lignende stillinger med likhetsscore |
| GET | `/employers` | Arbeidsgiverliste med åpne tellinger |
| GET | `/employers/{name}` | Arbeidsgiverprofil med register- og rekrutteringsmønster |
| GET | `/recruitment` | Rekrutteringsmønstre (persistensscore + bevis) |
| GET | `/recruitment/audit` | Algoritmevalidering (drift + signalfordelinger) |
| GET | `/events` | Endringslogg (keyset-paginering) |
| GET | `/stream` | SSE-livestream av endringsloggen |
| GET/POST | `/watchlists` | Lagrede søk |
| GET/DELETE | `/watchlists/{id}` | Nye stillinger siden siste sjekk / slett søk |
| GET | `/market` | Markedsoversikt (nye, fjernede, total, toppkategori/-fylke) |
| GET | `/market/history` | Tidsserie for markedet |
| GET | `/market/timetofill` | Tid-til-fylt for stillinger |
| GET | `/usage` | Egen kvote- og bruksstatus |
| GET | `/export/jobs` | Bulk-eksport av stillinger |
| GET | `/export/employers` | Bulk-eksport av arbeidsgivere |
| GET | `/export/events` | Bulk-eksport av endringslogg |

Full parameter- og responsspesifikasjon finnes i [`openapi.yaml`](./openapi.yaml) eller på [fakt.no/api-docs](https://fakt.no/api-docs).

---

## Vanlige spørringer

**Siste stillinger innen en kategori:**
```bash
curl "https://fakt.no/api/v1/jobs?category=Forsker&sort=newest&limit=10"
```

**Markedsoversikt og historikk:**
```bash
curl "https://fakt.no/api/v1/market"
curl "https://fakt.no/api/v1/market/history?limit=30"
```

Lønnsstatistikk per yrke og arbeidsgiver finner du på [fakt.no/lonn](https://fakt.no/lonn) og i [fakt.no/rapport](https://fakt.no/rapport).

**Arbeidsgivere som ansetter mest:**
```bash
curl "https://fakt.no/api/v1/employers?sort=count&limit=10"
```

---

## Svarhoder og feil

API-en bruker standard HTTP-statuskoder. Ved feil returneres JSON med `error` (og noen ganger `expiresAt`/`used`/`quota`).

| Status | Betydning |
| --- | --- |
| `200` | OK |
| `400` | Ugyldig forespørsel |
| `401` | Ugyldig/utløpt `X-API-Key` |
| `403` | Nøkkelen er utløpt |
| `404` | Fant ikke ressursen |
| `429` | Kvote brukt opp (månedlig eller daglig) |

Relevante svarnagler: `X-Plan`, `X-Quota-Remaining`, `X-Quota-Limit`, `X-Daily-Remaining`, `X-Daily-Limit`.

---

## Data & metodikk

- **Kilder:** NAV Arbeidsplassen, SSB, Brønnøysundregistrene.
- **Metodikk og datakilder:** [fakt.no/metodikk](https://fakt.no/metodikk)
- **Om fakt:** [fakt.no/about](https://fakt.no/about)

## Lisens

MIT — se [LICENSE](./LICENSE).
