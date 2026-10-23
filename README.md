<p align="center"><a href="https://apify.com/DevilScrapes/builtin-jobs-scraper"><img src="assets/banner.svg" alt="BuiltIn Jobs Scraper — Python example" width="100%"></a></p>

# BuiltIn Jobs Scraper in Python: tech job listings and salary bands for pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/builtin-jobs-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/builtin-jobs-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/builtin-jobs-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Built In is a job site focused on technology and startup roles. Unlike a single employer's careers page, it aggregates postings from thousands of companies and annotates each card with a salary band, a seniority level, a work mode, category tags and a few top skills. That annotation is what makes it useful for analysis. This repository is a compact Python client for the **[BuiltIn Jobs Scraper](https://apify.com/DevilScrapes/builtin-jobs-scraper)** on Apify: you describe the search, the Actor pages through builtin.com in the cloud, and you get a dataset where pay is already split into integer minimum and maximum columns. Save it as JSON or CSV and go straight to the analysis.

> **Actor:** [BuiltIn Jobs Scraper](https://apify.com/DevilScrapes/builtin-jobs-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [builtin-jobs-scraper-javascript](https://github.com/devilscrapes/builtin-jobs-scraper-javascript)

## 🔥 What you get

- **Three filters that compose**: a keyword, a city and a category can be combined in one run. Free text such as `Austin, TX` or `Data Science` is turned into the slug form builtin.com uses, so you do not have to look it up.
- **Salary as numbers, not just text**: a band like `142K-204K Annually` becomes `salary_min` 142000 and `salary_max` 204000, while `salary_raw` keeps the original wording including the pay period.
- **Multi-city cards unfolded**: when a card only says `3 Locations`, the `locations` array lists each city from the card's tooltip instead of the count.
- **A timestamp you can sort**: relative labels such as `5 Hours Ago` or `Reposted Yesterday` are converted to an ISO-8601 estimate at run time, with the raw label kept beside it.
- **Work mode in a fixed vocabulary**: Remote, Hybrid, Onsite, or `Not specified` when the card is ambiguous, so filtering never depends on how the site phrased it.
- **Clean stopping and no duplicates**: paging ends at your limit, on an empty page, or when a page repeats the previous one, and a job seen twice in the run is kept once.
- **A visible outcome**: a search that matches nothing finishes with a status message saying so, and pages that keep failing after retries make the run fail instead of returning a quietly short dataset.

## 🎯 Use cases

### Benchmark pay by seniority and work mode

Search one role family, keep rows whose `salary_raw` ends in `Annually`, and pivot the midpoint of `salary_min` and `salary_max` by `seniority` and `work_mode`. It answers the question compensation teams keep asking: does a remote senior engineer on Built In earn more or less than a hybrid one?

### Find the skills a market is asking for

`top_skills` is a list per row. Explode it in pandas and count, and you have a ranking of the tools employers attach to, say, data science roles in Chicago this month. Repeat the run monthly and the ranking becomes a trend line.

### Compare cities for a hiring or relocation decision

Run the same keyword for several cities, add a `city` column from the input you used, and compare volume, median band and remote share side by side. Because multi-location postings are unfolded into `locations`, a role open in three cities is not silently credited to one.

### Track posting velocity over time

Schedule the script daily with a small `maxResults` per search and store each output in a dated folder. `posted_at_estimate` lets you bucket new listings by day, which shows whether hiring in a category is speeding up or cooling.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/builtin-jobs-scraper-python.git
cd builtin-jobs-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [BuiltIn Jobs Scraper](https://apify.com/DevilScrapes/builtin-jobs-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Underneath the command-line wrapper, everything comes down to the few lines below. `ApifyClient` launches the BuiltIn Actor, blocks until its run reaches a terminal state, and hands back listings through a lazy iterator.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/builtin-jobs-scraper").call(run_input={'search': 'engineer', 'maxResults': 10, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

The included `input.json` searches for `engineer` across all of builtin.com and stops after ten listings, a quick and inexpensive first run. Add `city` or `category` to narrow it; both accept plain wording. When `search` and `city` are set together, the city is sent as a query parameter rather than a path segment, which mirrors how the site itself builds such URLs.

```json
{
  "search": "engineer",
  "maxResults": 10,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `search` | string | no | `"engineer"` | — |
| `city` | string | no | — | — |
| `category` | string | no | — | — |
| `maxResults` | integer | no | `100` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

You get one row per job card. Fields that a card does not show are `null` or an empty list rather than missing. The numeric salary columns are only filled when the band is written with a `K` or `M` suffix; hourly figures without one leave them `null` while `salary_raw` still holds the text.

| Field | Type | Always present |
|---|---|---|
| `job_id` | string | yes |
| `title` | string | yes |
| `url` | string | yes |
| `company_name` | string | yes |
| `company_url` | string or null | no |
| `company_logo_url` | string or null | no |
| `location` | string | yes |
| `locations` | array | yes |
| `work_mode` | string | yes |
| `posted_text` | string | yes |
| `posted_at_estimate` | string or null | no |
| `salary_raw` | string or null | no |
| `salary_min` | integer or null | no |
| `salary_max` | integer or null | no |
| `seniority` | string or null | no |
| `categories` | array | yes |
| `top_skills` | array | yes |
| `description_snippet` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| result | $0.0015 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/builtin-jobs-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

This pandas snippet (`pip install pandas`) keeps annual salary bands, computes a midpoint per listing and prints a median table of seniority against work mode. It then counts the fifteen most frequent skills across every row.

```python
import json

import pandas as pd

jobs = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))

annual = jobs[jobs["salary_raw"].fillna("").str.contains("Annually")].copy()
annual["midpoint"] = (annual["salary_min"] + annual["salary_max"]) / 2
bands = annual.pivot_table(
    index=annual["seniority"].fillna("Unstated"),
    columns="work_mode",
    values="midpoint",
    aggfunc="median",
)
print(bands.round(-3).to_string())

skills = jobs.explode("top_skills")["top_skills"].dropna().value_counts().head(15)
print(skills.to_string())
```

## ❓ FAQ

### Is Built In a single company's job board?

No, and that is the main difference from ATS-based scrapers. Built In lists roles from many employers on one site, so a single keyword search returns a cross-company sample. Use `company_name` and `company_url` to group by employer.

### Why are some salary columns empty?

Plenty of listings show no pay at all, and some show a band the parser cannot turn into thousands, such as an hourly figure with no `K`. In both cases `salary_min` and `salary_max` are `null`; check `salary_raw` to see which case you are in.

### How accurate is `posted_at_estimate`?

It is computed from Built In's relative label at the moment the run reads the page, so its precision follows the label: minutes for fresh posts, whole days for older ones. Labels in an unrecognised format leave it `null`.

### Does it read the full job description?

It returns the short description excerpt shown on the search card. Detail pages and company profile pages are not visited, so long-form requirements and benefits are out of scope.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/builtin-jobs-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [BuiltIn Jobs Scraper on Apify Store](https://apify.com/DevilScrapes/builtin-jobs-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/builtin-jobs-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Dice Tech Jobs Scraper](https://apify.com/DevilScrapes/dice-tech-jobs-scraper) — another tech-focused job board, heavier on contract and enterprise roles.
- [Wellfound Startup Jobs Scraper](https://apify.com/DevilScrapes/wellfound-jobs-scraper) — startup roles with funding-stage context.
- [RemoteOK Jobs Scraper](https://apify.com/DevilScrapes/remoteok-jobs-scraper) — remote-only tech postings from a different source.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/builtin-jobs-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Built In (builtin.com).
