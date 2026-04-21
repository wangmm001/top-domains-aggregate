# top-domains-aggregate

Daily cross-list aggregation of domain popularity signals, computed by [`wangmm001/feedcache`](https://github.com/wangmm001/feedcache).

This repo is **derived data** — unlike the five sibling repos (`umbrella-top1m-cache`, `tranco-top1m-cache`, `majestic-million-cache`, `cloudflare-radar-rankings-cache`, `crux-top-lists-mirror`), no upstream serves this file. It's the daily full-outer-join of those five lists by domain name.

## Layout

```
data/
  YYYY-MM-DD.csv.gz     # daily aggregate, gzipped
  current.csv.gz        # pointer to latest
```

## Format

CSV with exactly 3 columns:

```
domain,count,lists
google.com,5,cloudflare-radar|crux|majestic|tranco|umbrella
youtube.com,5,cloudflare-radar|crux|majestic|tranco|umbrella
github.io,4,cloudflare-radar|majestic|tranco|umbrella
...
only-in-umbrella.test,1,umbrella
```

- `domain` — lowercase, ASCII. For CrUX origins (`https://www.example.com`), `www.` is stripped; sub-domains (e.g. `en.wikipedia.org`) are kept as-is.
- `count` — integer 1–5 = number of distinct input lists the domain appears in.
- `lists` — alphabetically sorted, `|`-separated list of source names.

Rows sorted by `(count DESC, domain ASC)`. Full output — includes domains with count=1.

## Input lists

Fetched from these `current` files at aggregation time:

| List | Source | URL |
|---|---|---|
| `umbrella` | Cisco Umbrella Top 1M | `raw.githubusercontent.com/wangmm001/umbrella-top1m-cache/main/data/current.csv.gz` |
| `tranco` | Tranco Top 1M | `…/tranco-top1m-cache/main/data/current.csv.gz` |
| `majestic` | Majestic Million | `…/majestic-million-cache/main/data/current.csv.gz` |
| `cloudflare-radar` | Cloudflare Radar top-1M bucket | `…/cloudflare-radar-rankings-cache/main/data/current/top-1000000.csv.gz` |
| `crux` | Chrome UX Report Top 1M (via zakird mirror) | `…/crux-top-lists-mirror/main/data/global/current.csv.gz` |

The first 4 are direct 1M-domain lists. CrUX's 1M global list uses `rank` as a **magnitude bucket** (1000/10000/100000/1000000), not an ordinal — all ~1M rows are treated as "present in CrUX" regardless of bucket.

## Known simplifications (v1)

- **No PSL normalization yet**: we don't reduce `en.wikipedia.org` → `wikipedia.org` using the Public Suffix List. Most well-known brands (`google.com`, `facebook.com`, …) already appear in the same shape across all 5 sources, so the count=5 tier is clean. Sub-domain-heavy CrUX entries may under-merge with the registrable-domain-centric other lists. A v2 could pull in `wangmm001/public-suffix-list-cache` to do full normalization.
- **IDN**: ASCII (punycode) vs Unicode inconsistencies across sources are not reconciled. Rare for top domains.

## Consume

```bash
# Top 50 today
curl -L https://raw.githubusercontent.com/wangmm001/top-domains-aggregate/main/data/current.csv.gz | zcat | head -50

# Count distribution
curl -L https://raw.githubusercontent.com/wangmm001/top-domains-aggregate/main/data/current.csv.gz | zcat | cut -d, -f2 | tail -n +2 | sort | uniq -c | sort -rn

# Everything in all 5 lists
curl -L https://raw.githubusercontent.com/wangmm001/top-domains-aggregate/main/data/current.csv.gz | zcat | awk -F, '$2==5'
```

## License

Scaffolding: MIT. Derived data is a join of five upstream lists — each retains its upstream license. When redistributing, honor all five (Umbrella ToS, Tranco CC-BY, Majestic CC-BY, Cloudflare Radar terms, CrUX terms).

## How it works

Daily GitHub Actions cron (05:30 UTC) calls `wangmm001/feedcache`'s reusable workflow with `source: aggregate-top-domains`. That step `pip install`s feedcache, runs `feedcache aggregate-top-domains data/`, which fetches the 5 raw URLs above, computes the aggregate in memory, and writes `data/YYYY-MM-DD.csv.gz` + `data/current.csv.gz`. Atomic: any fetch failure aborts before writing. Deterministic gzip + `git diff --cached --quiet` means no commit on days where upstream content hasn't changed.

Cron time 05:30 UTC is chosen to run **after** all 5 input repos have completed their daily updates (03:30 / 03:45 / 04:00 / 04:15 / 05:00 UTC respectively — the crux mirror at 05:00 is the latest).
