> [!WARNING]
> DISCLAIMER: The scripts and links provided on this GitHub page are for informational and educational purposes only. I do not claim responsibility for any issues, damages, or losses that may arise from the use of these scripts or links. Users are advised to use them at their own risk and discretion. Always review and test any code or links before implementing them in your projects.

All streams come from a free, publicly available source. No DRM content and no login required. For more content (such as on-demand movies), visit [Xumo](https://play.xumo.com) or download the app.

# ⭐ Xumo M3U Playlist and EPG Generator

Automatically generates an M3U playlist and a gzipped XMLTV EPG for Xumo's free US channels. A GitHub Action rebuilds both files every 6 hours.

## Links

| File | URL |
|---|---|
| M3U playlist | `https://raw.githubusercontent.com/jayeAch/xumo-playlist-generator/main/playlists/xumo_playlist.m3u` |
| EPG (XMLTV, gzipped) | `https://raw.githubusercontent.com/jayeAch/xumo-playlist-generator/main/playlists/xumo_epg.xml.gz` |

The M3U header already points at the EPG through `url-tvg`, so most players pick up the guide automatically. If yours doesn't, add the EPG URL manually.

Channel `tvg-id` values match the channel IDs in the EPG.

## How it works

1. Pulls the US channel list (list ID `10006`), keeping live, non-DRM channels.
2. Looks up each channel's current program and takes its stream URL.
3. Pulls the guide for the dates and time slices configured below.
4. Resolves program titles and descriptions, caching them in `playlists/asset_titles.json`.
5. Writes `xumo_playlist.m3u` and `xumo_epg.xml.gz`, and the workflow commits them.

## Improvements over the original

**EPG coverage**
- The guide endpoint (`/epg/<list>/<date>/<n>.json`) returns one 6-hour slice per call. The old script only ever requested slice `0` (about 00:00–06:00 UTC), which left roughly 17–18 hour gaps and no current program for most of the day. The script now fetches every valid slice for each date. Slices are discovered at runtime by probing; currently `0–3` are valid.
- Already-ended programs are dropped before processing, so the file only contains what's airing now or later.

**Titles and descriptions**
- The EPG response doesn't include titles, so each asset is looked up separately. Lookups are now run soonest-airing first, with 16 parallel workers and a retry, so the current and next programs are always resolved first.
- Lookups stop after a time budget (`ASSET_LOOKUP_BUDGET_SECONDS`, default 420s). Anything left over is filled in on later runs via the cache.
- Programs whose title can't be resolved are skipped instead of being published with a raw asset ID like `XM0WYASIG6RGO3` as the title.

**Playlist**
- The current program for each channel is now picked correctly. `broadcast.json` gives times as unix seconds under `timestamps`, which the old code didn't read, so it fell back to the first asset in the hour (often a program that had already ended).

**Workflow**
- Output files are only committed when they changed, with push retries and `concurrency` protection against overlapping runs.

## Configuration

Settings are at the top of `generate_xumo.py`.

| Setting | Default | Purpose |
|---|---|---|
| `EPG_FETCH_DAYS` | `3` | The script fetches yesterday plus this many days. `2` (yesterday, today, tomorrow) is enough for a live guide and is faster. |
| `EPG_PROBE_RANGE` | `range(0, 6)` | Slice values to test. `0–3` are valid, `4+` return HTTP 400. |
| `ASSET_LOOKUP_BUDGET_SECONDS` | `420` | Maximum time spent resolving titles per run. |
| `ASSET_INFO_WORKERS` | `16` | Parallel title lookups. |
| `GITHUB_USER` / `GITHUB_REPO` / `GITHUB_BRANCH` | `jayeAch` / `xumo-playlist-generator` / `main` | Used to build the EPG URL written into the M3U. Change these if you fork. |

## Running

**GitHub Actions:** the workflow in `.github/workflows/generate_xumo.yml` runs every 6 hours (`0 */6 * * *`) and can be started manually from the Actions tab. It needs `contents: write` permission to commit the output files, which the workflow file already sets.

**Locally:**

```bash
pip install requests
python generate_xumo.py
```

Output is written to `playlists/`.

## Notes

- **Don't delete `playlists/asset_titles.json`.** It's the title cache. With it, runs only need to look up new programs. Without it, the next run has to resolve thousands of titles and can hit the time budget, which leaves some programs without titles until later runs.
- The first run after clearing the cache is slow. Normal runs are much shorter. Most of the remaining time is the per-channel stream URL lookup (about 3–4 minutes).
- A small percentage of programs may be missing titles right after a cold cache. They fill in over the next few runs.
- If the guide goes stale in your player, force an EPG refresh or clear its guide data. Most players cache it.

## Credits

Based on the original [xumo-playlist-generator](https://github.com/BuddyChewChew/xumo-playlist-generator) by BuddyChewChew.

EPG coverage, title resolution and current-program selection fixes developed with [Claude](https://claude.ai) (Anthropic).
