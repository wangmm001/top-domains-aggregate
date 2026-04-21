# top-domains-aggregate

Daily cross-list aggregation of domain popularity signals, computed by [`wangmm001/feedcache`](https://github.com/wangmm001/feedcache).

This repo is **derived data** — unlike the six sibling repos (`umbrella-top1m-cache`, `tranco-top1m-cache`, `majestic-million-cache`, `cloudflare-radar-rankings-cache`, `crux-top-lists-mirror`, `common-crawl-ranks-cache`), no upstream serves this file. It's the daily full-outer-join of those six lists by domain name.

## Layout

```
data/
  YYYY-MM-DD.csv.gz     # daily aggregate, gzipped
  current.csv.gz        # pointer to latest
```

## Format

CSV with exactly 4 columns:

```
domain,count,score,lists
google.com,6,0.066518,cloudflare-radar|common-crawl|crux|majestic|tranco|umbrella
youtube.com,5,0.049075,cloudflare-radar|common-crawl|crux|tranco|umbrella
...
cc-only.example,1,0.000943,common-crawl
only-in-umbrella.test,1,0.000001,umbrella
```

- `domain` — lowercase, ASCII, **reduced to registrable form (eTLD+1)** via the Mozilla [Public Suffix List](https://publicsuffix.org/). So `en.wikipedia.org` → `wikipedia.org`, `https://www.google.com` → `google.com`. Private PSL entries are honored: `blog.github.io` stays as-is because `github.io` is itself a public suffix (user-content hosting), so `blog` is the registered label.
- `count` — integer 1–6 = number of distinct input lists the domain appears in.
- `score` — **rank-fusion score**, `sum(1/rank_in_list_i)` for each list the domain is in. 6 decimal places. Larger = more broadly popular across lists.
- `lists` — alphabetically sorted, `|`-separated source names.

Rows sorted by `(count DESC, score DESC, domain ASC)`. Full output — includes domains with count=1.

### How the `score` is computed

Each list contributes `1/rank` to a domain's score, where `rank` depends on the list's nature:

| List | Rank used | Why |
|---|---|---|
| umbrella | true ordinal 1–1,000,000 | ordered list |
| tranco | true ordinal 1–1,000,000 | ordered list |
| majestic | `GlobalRank` column | ordered list |
| cloudflare-radar | `1,000,000` (worst-case) | Cloudflare top-1M bucket is **unordered** |
| crux | the `rank` value (1000 / 10000 / 100000 / 1000000) | CrUX publishes **magnitude buckets**, not ordinal ranks |
| common-crawl | `rank` column (harmonicc_pos) | ordered list — CC's webgraph Harmonic Centrality is a true ordinal 1–1,000,000 |

So `google.com` at Umbrella rank 1, Tranco rank 1, Majestic rank 1, Cloudflare membership, CrUX bucket 1000, CC rank 1:
`score = 1 + 1 + 1 + 1e-6 + 1e-3 + 1 ≈ 3.001001 + 1 = 4.001001`

A bottom-of-list-everywhere domain:
`score ≈ 1e-6 * 6 = 6e-6`, so it ranks last within its count tier.

This is a "reciprocal rank fusion" (RRF) style score — top-1 in a list dominates; membership in an unordered list adds a tiny nudge; CrUX buckets weight proportionally to how deep the bucket is.

## Input lists

Fetched from these `current` files at aggregation time:

| List | Source | URL |
|---|---|---|
| `umbrella` | Cisco Umbrella Top 1M | `raw.githubusercontent.com/wangmm001/umbrella-top1m-cache/main/data/current.csv.gz` |
| `tranco` | Tranco Top 1M | `…/tranco-top1m-cache/main/data/current.csv.gz` |
| `majestic` | Majestic Million | `…/majestic-million-cache/main/data/current.csv.gz` |
| `cloudflare-radar` | Cloudflare Radar top-1M bucket | `…/cloudflare-radar-rankings-cache/main/data/current/top-1000000.csv.gz` |
| `crux` | Chrome UX Report Top 1M (via zakird mirror) | `…/crux-top-lists-mirror/main/data/global/current.csv.gz` |
| `common-crawl` | Common Crawl webgraph (host/domain rank cache) | `raw.githubusercontent.com/wangmm001/common-crawl-ranks-cache/main/data/domain/current.csv.gz` |

The first 4 are direct 1M-domain lists. CrUX's 1M global list uses `rank` as a **magnitude bucket** (1000/10000/100000/1000000), not an ordinal — all ~1M rows are treated as "present in CrUX" regardless of bucket. Common Crawl's list uses a true ordinal `harmonicc_pos` rank from the CC webgraph Harmonic Centrality computation.

## Known simplifications

- **PSL is the "public suffix" snapshot of the day we ran**: aggregator fetches `raw.githubusercontent.com/wangmm001/public-suffix-list-cache/main/data/current.dat.gz` at the start of each run. If that repo's cron is behind by a day, we use a slightly stale PSL. Trivial in practice.
- **IDN**: ASCII (punycode) vs Unicode inconsistencies across sources are not reconciled. Rare for top domains.
- **CC's domain granularity vs. PSL:** `common-crawl-ranks-cache` exports a domain-level file where CC's own reverse-domain-merge has already produced registrable-form domains. The aggregator still passes these through PSL `privatesuffix` so all 6 inputs go through the same normalization. In practice the two agree on ~all cases; the main divergence is around public-suffix edge cases (e.g. `github.io`).

## Consume

```bash
# Top 50 hottest today (count=6 + highest score)
curl -L https://raw.githubusercontent.com/wangmm001/top-domains-aggregate/main/data/current.csv.gz | zcat | head -51

# Everything in all 6 lists (the universally-popular tier), sorted by score
curl -L https://raw.githubusercontent.com/wangmm001/top-domains-aggregate/main/data/current.csv.gz | zcat | awk -F, 'NR==1 || $2==6'

# Count distribution (needs python, shell cut misparses CSV-quoted commas)
curl -L https://raw.githubusercontent.com/wangmm001/top-domains-aggregate/main/data/current.csv.gz | zcat | python3 -c "
import csv, sys
from collections import Counter
c = Counter(r['count'] for r in csv.DictReader(sys.stdin))
for k in sorted(c, key=int, reverse=True): print(f'{k}: {c[k]}')"
```

## License

Scaffolding: MIT. Derived data is a join of six upstream lists — each retains its upstream license. When redistributing, honor all six (Umbrella ToS, Tranco CC-BY, Majestic CC-BY, Cloudflare Radar terms, CrUX terms, Common Crawl terms).

## How it works

Daily GitHub Actions cron (05:30 UTC) calls `wangmm001/feedcache`'s reusable workflow with `source: aggregate-top-domains`. That step `pip install`s feedcache, runs `feedcache aggregate-top-domains data/`, which fetches the 6 raw URLs above, computes the aggregate in memory, and writes `data/YYYY-MM-DD.csv.gz` + `data/current.csv.gz`. Atomic: any fetch failure aborts before writing. Deterministic gzip + `git diff --cached --quiet` means no commit on days where upstream content hasn't changed.

Cron time 05:30 UTC is chosen to run **after** all 6 input repos have completed their daily updates (03:30 / 03:45 / 04:00 / 04:15 / 05:00 / 05:15 UTC respectively — the crux mirror at 05:00 is the latest of the original five).
