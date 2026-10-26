<p align="center"><a href="https://apify.com/DevilScrapes/france-sirene-companies-scraper"><img src="assets/banner.svg" alt="France SIRENE Company Registry Scraper — Python example" width="100%"></a></p>

# SIRENE API Scraper in Python: French company register data into pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/france-sirene-companies-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/sirene-api-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/sirene-api-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository wraps the **[France SIRENE Company Registry Scraper](https://apify.com/DevilScrapes/france-sirene-companies-scraper)** on Apify in a short, tested Python script. Describe a slice of the French economy the way INSEE classifies it, for example computer programming firms under NAF `62.01Z` in the Rhône department with 20 to 49 staff, and you get back one flat record per company: SIREN, names, legal form code, size class, sector, headcount band, status, dates and head-office address. The requests go to the state's open registry API from Apify's infrastructure, so there is no INSEE account to create and nothing nested to unpack. Output is written to `results.json` and `results.csv`, a format any notebook or BI tool already understands.

> **Actor:** [France SIRENE Company Registry Scraper](https://apify.com/DevilScrapes/france-sirene-companies-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [sirene-api-scraper-javascript](https://github.com/devilscrapes/sirene-api-scraper-javascript)

## 🔥 What you get

- **One row per legal unit**: each company is keyed by its nine-digit `siren`, with the head-office SIRET, address, postal code, commune and department lifted out of the nested registry block onto the row itself.
- **The registry's own filters**: free text that matches a name, SIREN or SIRET, a NAF/APE code in `NN.NNL` form, an INSEE department or commune code, a workforce band and active or ceased status, all combined server-side.
- **Codes with readable labels**: the one-letter NACE section arrives with its English name, and the INSEE size class (`GE`, `ETI`, `PME`, `TPE`) comes with a decoded label.
- **Compliance-friendly columns**: creation and closure dates, the year behind the headcount band, total and still-open establishment counts, the first registered VAT number and a link to the company's public directory page.
- **Mistakes caught before the meter starts**: a NAF code in the wrong format, an out-of-range `maxResults` or a form with every filter cleared is refused without the start fee.
- **Straight answers from the API**: rate limits and server errors are retried with exponential backoff; a filter value the registry rejects ends the run with the registry's own error text instead of five pointless retries.
- **Empty is not broken**: filters that match no company still end in a `SUCCEEDED` run, whose status line lists what was searched, and a malformed entry is skipped and counted rather than ending the page.

## 🎯 Use cases

### Size a market before you enter it

Run one query per department for the NAF codes you sell into, concatenate the files and pivot company counts by `department_code` and `company_size_category`. In an afternoon you know how many mid-caps versus micro-enterprises sit in each territory, which is the first slide of every French market-entry plan.

### Clean a supplier master file

Loop over the SIRENs in your vendor table, pass each one as `nameQuery`, and merge the answers back on `siren`. Mismatched legal names, missing VAT numbers and moved head offices fall out of a single pandas `merge` with an indicator column.

### Business creation cohorts

Group `creation_date` by year within a sector and region to see when a cluster formed and how many of its firms still have open sites. Pair an `A` run with a `C` run for the same filters and you have survivors and closures side by side.

### A dated, reproducible registry extract

Schedule the script from cron, keep each CSV under a dated filename and record the input next to it. When an analysis is questioned months later, you can show exactly what the register returned and when, with `last_updated` on every row.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/sirene-api-scraper-python.git
cd sirene-api-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [France SIRENE Company Registry Scraper](https://apify.com/DevilScrapes/france-sirene-companies-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

If you only want the integration itself, it is these few lines. `apify-client` launches the run, waits for its end and yields every dataset row.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/france-sirene-companies-scraper").call(run_input={'nafCode': '62.01Z', 'departmentCode': '69', 'workforceRange': '12', 'administrativeStatus': 'A', 'maxResults': 10, 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

The script hands `input.json` to the Actor verbatim. The bundled filter asks for active companies with NAF code `62.01Z` (computer programming) in department `69`, headcount band `12` (20 to 49 employees), and stops at ten, which costs a few cents. Swap the department or code, add `communeCode` for a single town, or put a SIREN in `nameQuery` to fetch one known company.

```json
{
  "nafCode": "62.01Z",
  "departmentCode": "69",
  "workforceRange": "12",
  "administrativeStatus": "A",
  "maxResults": 10,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `nameQuery` | string | no | — | — |
| `nafCode` | string | no | — | — |
| `departmentCode` | string | no | — | — |
| `communeCode` | string | no | — | — |
| `workforceRange` | string | no | — | — |
| `administrativeStatus` | string | no | `"A"` | `A`, `C` |
| `maxResults` | integer | no | `100` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

Every row is one legal unit with its head office flattened onto it, not one row per establishment. Keep `siren` and `siret_headquarters` as strings, because leading zeros matter and pandas will otherwise read them back as integers. Missing values are `None`: `acronym`, `closure_date` and `vat_number` are often empty, and that reflects the register, not a parsing failure.

| Field | Type | Always present |
|---|---|---|
| `siren` | string or null | no |
| `siret_headquarters` | string or null | no |
| `company_name` | string or null | no |
| `legal_name` | string or null | no |
| `acronym` | string or null | no |
| `legal_form_code` | string or null | no |
| `company_size_category` | string or null | no |
| `company_size_label` | string or null | no |
| `naf_code` | string or null | no |
| `naf_section_code` | string or null | no |
| `naf_section_label` | string or null | no |
| `workforce_range_code` | string or null | no |
| `workforce_range_year` | string or null | no |
| `administrative_status` | string or null | no |
| `is_active` | boolean or null | no |
| `creation_date` | string or null | no |
| `closure_date` | string or null | no |
| `last_updated` | string or null | no |
| `street_address` | string or null | no |
| `postal_code` | string or null | no |
| `city` | string or null | no |
| `department_code` | string or null | no |
| `establishment_count` | integer or null | no |
| `open_establishment_count` | integer or null | no |
| `vat_number` | string or null | no |
| `url` | string or null | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.004 |

Prices as configured on 2026-08-26. The [Actor page](https://apify.com/DevilScrapes/france-sirene-companies-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

The pandas script below (`pip install pandas`) protects SIRENs as text, computes company age and the share of still-open sites, and writes three tables: companies per city, a NAF-by-size matrix and creations per year.

```python
import json

import pandas as pd

rows = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
rows["siren"] = rows["siren"].astype(str).str.zfill(9)  # SIREN is an identifier, not a number
rows["created"] = pd.to_datetime(rows["creation_date"], errors="coerce")
rows["age_years"] = (pd.Timestamp.now() - rows["created"]).dt.days / 365.25
rows["open_site_share"] = rows["open_establishment_count"] / rows["establishment_count"]

by_city = (
    rows.groupby("city")
    .agg(
        companies=("siren", "nunique"),
        median_age_years=("age_years", "median"),
        open_sites=("open_establishment_count", "sum"),
        with_vat=("vat_number", "count"),
    )
    .sort_values("companies", ascending=False)
)
by_city.to_csv("companies_by_city.csv")

size_mix = pd.crosstab(rows["naf_code"], rows["company_size_category"].fillna("unknown"))
size_mix.to_csv("naf_by_size.csv")

cohorts = rows.dropna(subset=["created"]).groupby(rows["created"].dt.year)["siren"].count()
cohorts.to_csv("creations_by_year.csv")

# Reading a CSV back later? Keep the leading zeros: pd.read_csv(path, dtype={"siren": str})
print(by_city.head(10).round(1).to_string())
```

## ❓ FAQ

### Do I need an INSEE API key or a data.gouv.fr account?

No. The Actor reads the French state's open company search API, which is keyless. Your Apify token is the one credential this repository uses.

### Why can I not pull a whole department in one run?

A run stops at `maxResults`, which allows at most 1,000 companies, and the registry pages 25 at a time. For a full department or sector, split it by commune, NAF code or workforce band and run the slices, then deduplicate on `siren` in pandas.

### I searched a SIREN and got nothing. Why?

`administrativeStatus` defaults to `A`, so a company that has ceased trading is filtered out. Run the same query with `C` to check whether it closed, and use `closure_date` to see when.

### Does it return directors, shareholders or financial statements?

No. It returns identity, classification, status, headcount band, dates and the head-office address. Officers, beneficial owners, accounts and the full list of secondary establishments are outside the output.

### Is my NAF code format right?

Use two digits, a dot, two digits and a capital letter, such as `47.11F`. A code without the dot is rejected before the run is charged, because the registry refuses that form too.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/france-sirene-companies-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [France SIRENE Company Registry Scraper on Apify Store](https://apify.com/DevilScrapes/france-sirene-companies-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/sirene-api-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Companies House UK Scraper](https://apify.com/DevilScrapes/companies-house-uk-scraper) — the UK register, searchable by SIC code, town and incorporation date.
- [KVK Company Search Scraper](https://apify.com/DevilScrapes/kvk-company-search-scraper) — Dutch Chamber of Commerce records with KVK and branch numbers.
- [GLEIF LEI Scraper — Legal Entity Identifier Records](https://apify.com/DevilScrapes/gleif-lei-records-scraper) — LEI codes for French and foreign entities that trade in financial markets.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/france-sirene-companies-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with INSEE, the French administration or data.gouv.fr.
