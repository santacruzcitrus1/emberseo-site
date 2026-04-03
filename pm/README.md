# Paul's Rental Market Dashboard — /pm/

## Live URL
**https://emberseo.ai/pm/** (unlisted — direct link only, not linked from main site)

## What It Is
A searchable rental market database for Paul (Judd's step-father) covering:
- **ZIP 95076** — Watsonville, CA
- **ZIP 95019** — Aptos/Freedom/Santa Cruz, CA

Pulls from Zillow + Realtor.com. Every listing is clickable — address links directly to the Zillow or Realtor listing page. Updates daily at 7 AM PST.

## Files
| File | Purpose |
|------|---------|
| `index.html` | Single-page searchable dashboard (filter by zip, beds, price, search) |
| `data.json` | Rental listing data — updated daily by cron |
| `README.md` | This file |

## Stack
- **Frontend:** Pure HTML/CSS/JS (no build step) — lives in emberseo-site GitHub repo at `/pm/`
- **Hosting:** GitHub Pages (emberseo.ai domain via CNAME)
- **Data:** JSON file fetched by the page at load time
- **Scraper:** `/root/.openclaw/workspace/scripts/rental-scraper.py`
- **Push script:** `/root/.openclaw/workspace/scripts/rental-push.sh`

## How It Updates (Daily Cron)
```
cron (7 AM PST) 
  → rental-scraper.py 
    → Firecrawl scrapes Zillow (with scroll actions to load all listings)
    → HomeHarvest scrapes Realtor.com
    → Merges + dedupes by address
    → Saves to emberseo/pm/data.json
  → rental-push.sh
    → git pull /tmp/emberseo-push
    → copies data.json to pm/
    → git commit + push to santacruzcitrus1/emberseo-site master
    → GitHub Pages serves updated data
```

## Cron Line
```
0 15 * * * cd /root/.openclaw/workspace && python3 scripts/rental-scraper.py >> /root/.openclaw/workspace/logs/rental-scraper.log 2>&1 && bash scripts/rental-push.sh >> /root/.openclaw/workspace/logs/rental-scraper.log 2>&1
```

## Scraper Decision Tree
When the scraper runs, here's what happens:

```
rental-scraper.py
├── For each ZIP (95076, 95019):
│   ├── Realtor.com via HomeHarvest (Python library)
│   │   └── Returns listings with full URLs ✅
│   └── Zillow via Firecrawl (stealth browser API)
│       ├── POST to https://api.firecrawl.dev/v2/scrape
│       ├── URL: https://www.zillow.com/watsonville-ca-{zip}/rentals/
│       ├── Uses scroll actions (5x scroll down) to load all lazy-loaded listings
│       ├── Returns markdown — parse with regex for price, address, URL, beds, baths, sqft
│       └── Zillow blocks direct browser CDP (PerimeterX bot detection)
│           └── Firecrawl's stealth browser bypasses it ✅
├── Merge: Zillow as base, add Realtor-only listings not already in Zillow
├── Compute stats: median, avg, min, max rent, breakdown by beds
└── Save to emberseo/pm/data.json

rental-push.sh
├── cd /tmp/emberseo-push (local clone of emberseo-site repo)
├── git pull master
├── cp data.json pm/data.json
├── git commit (skip if no changes)
└── git push to santacruzcitrus1/emberseo-site master
    └── GitHub Pages auto-deploys in ~60 seconds
```

## Key Tech Notes
- **Zillow anti-bot:** PerimeterX blocks direct HTTP and CDP browser. Firecrawl's headless stealth browser gets through.
- **Lazy loading:** Zillow only renders ~7 listings without scrolling. Must use Firecrawl scroll actions to get all 35+.
- **URL extraction:** Parsed from markdown with regex: `\[([0-9][^\]]+CA[^\]]*)\]\((https://www\.zillow\.com/homedetails/[^\)]+)\)`
- **Dedup key:** `re.sub(r'[^a-z0-9]', '', address.lower())` — strips punctuation for fuzzy match
- **GitHub PAT:** Stored at `/root/.config/github/pat` — needed for push. Expires ~1 year. Regenerate at github.com/settings/tokens (classic, repo scope).

## Repo
- **GitHub:** https://github.com/santacruzcitrus1/emberseo-site
- **Branch:** master
- **Path in repo:** `/pm/index.html` and `/pm/data.json`

## First Built
March 13, 2026 — rebuilt/restored March 25, 2026 (files were deleted in cleanup, restored from git history commit c66846ab)

## Firecrawl API
- **Key location:** `/root/.config/firecrawl/api-key`
- **Account:** santacruzcitrus1@gmail.com
- **Free tier:** 500 credits — each scrape costs ~1 credit
