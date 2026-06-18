<div align="center">

# Traffic Dashboard

</div>

<p align="center">
  <img src="docs/badges/html.svg" alt="HTML">
  <img src="docs/badges/actions.svg" alt="GitHub Actions">
  <img src="docs/badges/license.svg" alt="MIT">
</p>

<p align="center">
  <a href="https://b67687.github.io/Traffic-Dashboard/">
    <img src="docs/badges/dashboard-preview.svg" alt="Traffic Dashboard preview" width="760">
  </a>
</p>

<p align="center">
  <sub>Built with AI assistance — see <a href="./CREDITS.md">CREDITS.md</a></sub>
  <br>
  <a href="./CREDITS.md"><img src="docs/badges/deepseek.svg" alt="DeepSeek V4 Flash"></a>
</p>

A reusable GitHub Actions workflow + HTML dashboard for tracking clone, download, and view statistics across repositories. Originally embedded in every repo individually — now consolidated here as a shared resource.

## Usage

### 1. Add the workflow

Copy [`.github/workflows/traffic-badges.yml`](.github/workflows/traffic-badges.yml) into your repo's `.github/workflows/` directory.

### 2. Configure secrets and variables

| Setting              | Type                | Description                                                                             |
| -------------------- | ------------------- | --------------------------------------------------------------------------------------- |
| `TRAFFIC_GIST_ID`         | Repository variable | Gist ID for storing badge state (auto-created on first run if empty)                    |
| `TRAFFIC_ARCHIVE_GIST_ID` | Repository variable | Gist ID for monthly archive (separate gist, created manually)                           |
| `TRAFFIC_GIST_TOKEN`      | Repository secret   | Fine-grained PAT with **Gists (R/W)**, **Traffic (R)**, and **Actions (R)** permissions |

### 3. (Optional) Deploy the dashboard

The [`index.html`](index.html) file is an HTML dashboard page. Serve it via GitHub Pages from the root for a visual overview.

## Architecture

Two independent parts that share a Gist as the bridge:

- **Collector** — `.github/workflows/traffic-badges.yml` (GitHub Actions)
- **Viewer** — `index.html` (static, served via GitHub Pages)

### Data Flow

```
GitHub Traffic API
       │
       ▼
traffic-badges.yml (runs daily 3AM UTC + on push)
       │
       ├─ Fetches: release downloads, clones (14d), views (14d),
       │            referrers, popular paths, repo metadata (stars/forks)
       │
       ├─ Accumulates deltas (never re-downloads what it already saw)
       │
       ├─ Counts CI checkouts by inspecting every workflow run's job steps
       │   → organicClones = rawClones − ciCheckouts
       │
       ├─ Writes state.json + badge JSONs → Gist
       │
       └─ On 1st of each month → archive snapshot → archive Gist
               │
               ▼
index.html (fetches state.json from Gist)
       │
       ├─ Auto-detects REPO_NAME from URL path
       ├─ Auto-detects REPO_OWNER from hostname
       ├─ Looks up REPO_CONFIG → gets gist IDs + repo creation date
       ├─ Fetches state.json & optional archive
       └─ Renders 5 tabs with Chart.js
```

### Key Design Decisions

**Delta accumulation** — The Traffic API only returns 14 days of data. The workflow tracks the last-seen count per date. On each run it only adds the *increase* since the last check, preventing double-counting and handling the rolling API window.

**CI isolation** — Raw clones include CI/CD automation. The workflow inspects every Actions job's steps looking for `actions/checkout`, and counts Pages builds as hidden clones (each Pages deploy produces 1 clone). This produces `organicClones` — actual human activity.

**31-day rolling window** — `dailyHistory` keeps the last 31 days. Monthly archives persist full snapshots to a separate archive Gist for long-term retention.

**Fully static frontend** — Zero build step, zero server. Pure HTML + JS that fetches JSON from a raw Gist URL. Chart.js loaded from CDN with SRI hash. Deployed via GitHub Pages from `/` on `main`.

**Org-agnostic auto-detection** — `REPO_OWNER` is derived from the Pages hostname and `REPO_NAME` from the URL path, so the same `index.html` works under any GitHub Pages domain without hardcoded configuration.

## What the Dashboard Shows

| Tab | Data | Source |
|---|---|---|
| **Overview** | Combined daily activity chart with toggles for views, clones, downloads | `state.json` dailyHistory |
| **Installs** | Downloads, clones, organic clones, activity windows (24h/7d/30d) | `state.json` dailyHistory |
| **Views** | Page views, unique visitors, top referrers, popular pages (14-day snapshot) | `state.json` views/referrers/paths |
| **Community** | Stars, forks, open issues, star history chart | `state.json` metadata + live GitHub API |
| **Dev** | CI breakdown, raw vs organic clones, commit activity, code frequency, participation, punch card, contributors | `state.json` + live GitHub API |

## Files

| File                                                               | Purpose                                               |
| ------------------------------------------------------------------ | ----------------------------------------------------- |
| `.github/workflows/traffic-badges.yml`                             | Main workflow — collects stats, generates badges      |
| [`index.html`](https://b67687.github.io/Traffic-Dashboard/) | HTML dashboard for visual overview (served via Pages) |
