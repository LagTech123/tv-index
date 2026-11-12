# tv-index

A machine-readable index of Android TV apps on Google Play, used by the `tvstore` client
so it can filter Play search results down to things that actually run on a TV.

Google Play has **no** TV filter for search, charts, or category browse. Its TV form
factor only changes the two home streams (apps and games). Everything else is the phone
catalogue, so the client needs its own list of TV-capable packages. This repo is that list.

## Files

| File | Size | Purpose |
|---|---|---|
| `index.json` | ~3 MB | Full metadata, one object per app. Use for details/offline fallback. |
| `index.min.json` | ~430 KB | Trimmed to the fields a browse/search grid needs. Use at startup. |

Both are regenerated together. Fetch from `raw.githubusercontent.com` and cache locally;
there is no per-request API.

## Schema

```jsonc
{
  "schemaVersion": 1,
  "generatedAt": "2026-11-12T20:21:14Z",
  "generator": "tvstore-spike/IndexBuilder",
  "playLocale": "en_US",
  "stats": {
    "apps": 879,
    "requests": 205,
    "searchQueries": 40,
    "durationSeconds": 201,
    "bySource": { "tv-apps": 74, "tv-games": 79, "search:tv": 58 /* ... */ },
    "bySignal": { "pkg-dot-tv": 605, "storefront:tv-apps": 74 /* ... */ }
  },
  "apps": [
    {
      "packageName": "com.google.android.youtube.tv",
      "title": "YouTube for Android TV",
      "developer": "Google LLC",
      "category": "ENTERTAINMENT",
      "iconUrl": "https://play-lh.googleusercontent.com/...",
      "coverUrl": "https://play-lh.googleusercontent.com/...",
      "videoUrl": "https://play-lh.googleusercontent.com/...",
      "shortDescription": "...",
      "installs": 604042572,
      "versionCode": 123456,
      "versionName": "1.2.3",
      "rating": 4.1,
      "ratingCount": 123456,
      "price": "0",
      "isFree": true,
      "containsAds": false,
      "inAppPurchases": "$0.99 - $9.99 per item",
      "updatedOn": "Oct 13, 2026",
      "releasedOn": "May 1, 2014",
      "shareUrl": "https://play.google.com/store/apps/details?id=...",
      "targetSdk": 34,
      "contentRating": "Everyone",
      "screenshots": ["https://play-lh.googleusercontent.com/..."],
      "sources": ["storefront:tv-apps"],
      "signals": ["storefront:tv-apps"]
    }
  ]
}
```

Required per entry: `packageName`, `title`, `isFree`, `sources`, `signals`. Everything
else is omitted when Play did not report it. `category` is Play's browse code
(`ENTERTAINMENT`, `TOOLS`, `GAME_ACTION`, …). `installs` is Play's bucketed install count.

`index.min.json` keeps only `packageName`, `title`, `category`, `iconUrl`, `coverUrl`,
`installs`, `rating`, `isFree`, plus `schemaVersion`/`generatedAt` at the top level.

## How an app gets in

Two independent sources, recorded in `sources`:

1. **TV storefront** — the app appeared in Play's TV app or TV games home stream, including
   pagination. Play itself considers these TV-compatible. Signalled as
   `storefront:tv-apps` / `storefront:tv-games`.
2. **Filtered search** — the app appeared in one of 40 Play searches and its package name
   matched a TV heuristic. Signalled by the rule that matched:
   - `pkg-dot-tv` — contains `.tv` (e.g. `com.google.android.youtube.tv`)
   - `pkg-tv-prefix` — starts with `tv.` (e.g. `tv.pluto.android`)
   - `pkg-androidtv` — contains `androidtv`
   - `pkg-leanback` — contains `.leanback`
   - `pkg-livingroom` — contains `.livingroom` (e.g. `com.amazon.amazonvideo.livingroom`)
   - `pkg-ninja` — ends with `.ninja` (e.g. `com.netflix.ninja`)

Storefront membership is authoritative. Package signals are a cheap bootstrap and can
produce false positives (phone-only TV-remote or casting apps, games published under a
`tv.*` package). The client should prefer storefront-sourced entries and may later add a
quality bar or manifest verification without changing this schema.

## Regenerate

The crawler lives in the app repo, not here (this repo is data only):

```powershell
cd D:\tvstore\spike
$env:JAVA_HOME="C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot"
.\gradlew.bat --no-daemon buildIndex
```

It rewrites `D:\tvstore\tv-index\index.json` and `index.min.json`. A full run is ~200
requests and ~3–4 minutes against the anonymous `play.google.com` `batchexecute` lane.

Refresh cadence: weekly is plenty. Play's TV home streams rotate slowly; the package
heuristics do not expire.

## Live URLs

This repo: <https://github.com/LagTech123/tv-index>

```
https://raw.githubusercontent.com/LagTech123/tv-index/main/index.min.json   # ~430 KB
https://raw.githubusercontent.com/LagTech123/tv-index/main/index.json       # ~3 MB
```

Raw GitHub responses are cached for 5 minutes (`cache-control: max-age=300`), so the client
should still keep its own on-disk copy and only re-fetch when `generatedAt` is older than
its refresh window.

## Caveats

- Play search itself is not TV-filtered, so search-discovered entries are only as good as
  the package heuristic. An app that runs on TV but has a phone-style package name
  (`com.disney.disneyplus`) is found only via the TV storefront crawl.
- The index is locale-independent (built with `en_US`), but Play availability is not.
  Packages may not resolve in every country.
- Icons/banners are Google-hosted `play-lh.googleusercontent.com` URLs. Cache them; do not
  hotlink from the TV UI.
