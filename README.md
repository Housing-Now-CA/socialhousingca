# Social Housing California

The website for [Social Housing California](https://socialhousingca.org), a statewide campaign for permanently affordable, community-governed housing.

Built and maintained for Housing Now! California by [The Adriel Hampton Group](https://adrielhampton.com).

## Architecture

Static site hosted on GitHub Pages. Content lives in a Google Sheet and is rendered into `index.html` by a Python build script that runs on GitHub Actions.

```
Google Sheet              GitHub Actions             GitHub Pages
─────────────────         ──────────────────         ──────────────────
publish_token=v17    ──►  cron / manual fire   ──►  socialhousingca.org
content tabs              ↓
                          fetch CSV (gviz)
                          ↓
                          rewrite BUILD: sections
                          in index.html
                          ↓
                          commit + push to main
```

There is no database, no CMS, no build step the user runs locally. The Sheet is the source of truth for content. The HTML, CSS, and JS in the repo are the source of truth for everything else.

## Repo layout

```
.
├── .github/workflows/build.yml    # GitHub Actions workflow
├── images/                        # All hosted images
├── scripts/build.py               # Build script (Python, no dependencies beyond requests)
├── .build-state.json              # Tracks last-built publish_token; managed by build
├── CNAME                          # Tells GitHub Pages the custom domain
├── README.md                      # This file
├── favicon.svg                    # Browser tab icon
├── index.html                     # Source of truth AND deployed file
├── privacy.html                   # Static privacy page
├── robots.txt                     # Allow all + sitemap reference
└── sitemap.xml                    # For Google Search Console
```

## How the build works

The build script reads from six tabs in the Google Sheet:

| Tab | Purpose |
|---|---|
| `page_copy` | Key/value text for the site, plus three reserved control rows at the top. Drives all editable copy: headings, body text, button labels, video IDs, nav links, footer, etc. |
| `map_locations` | One row per map pin |
| `partners` | One row per coalition partner |
| `resources` | One row per Library PDF / brief |
| `stories` | One row per News, Story, or Campaign card |
| `model_cards` | One row per card in "Social Housing Models" or "Social Housing Principles" grids |

There's also a `map key` tab that's reference-only documentation — the build does not read it.

### The control rows in `page_copy`

The top three rows of `page_copy` gate the build:

| key | value | purpose |
|---|---|---|
| `publish` | `yes` or `no` | master switch |
| `publish_token` | any string | bumping this triggers a rebuild |
| `note` | optional | logged on each publish |

The build runs only when `publish == yes` AND `publish_token != stored_token`. Otherwise it exits after a single HTTP request, keeping idle cron cycles free.

These three keys are filtered out before page copy is rendered, so they can't accidentally show up in the site.

### Section markers

Inside `index.html`, content regions managed by the build are wrapped in marker comments:

```html
<!-- BUILD:stories:START -->
...generated content...
<!-- BUILD:stories:END -->
```

The build only rewrites content between markers. Everything outside — CSS, navigation, scripts, hand-edits — is preserved across builds. The map data marker uses JS-comment style instead (`/* BUILD:map_data:START */`) because it lives inside a `<script>` tag.

To add a new section: add markers to `index.html`, add a tab to the Sheet, write a `render_*` function in `build.py`, and add it to the `sections_for_index` dict in `main()`.

### Cron schedule

The workflow runs every 30 minutes during ~9am-6pm Pacific, Monday-Friday. GitHub Actions cron is UTC-only; Pacific Time shifts by 1 hour twice a year due to DST. The schedule is intentionally about an hour wider than 9-to-6 to absorb DST without missing publishes near the edges. Idle ticks cost nothing — the build exits after one HTTP request when `publish_token` is unchanged.

To publish outside business hours, the editor manually fires the workflow from the Actions tab → Run workflow.

## Publishing flow

1. Editor changes content in the Sheet
2. Editor bumps `publish_token` (e.g., `v17` → `v18` or any string change)
3. **Editor tabs out of the cell to commit it** — Google Sheets shows the new value on screen but doesn't save until focus leaves the cell
4. Cron fires (or editor runs the workflow manually) within 30 min during business hours
5. Build sees the new token, fetches all tabs, rewrites marker regions in `index.html`, commits if anything changed
6. GitHub Pages redeploys automatically (~30-60s)

## Sheet schemas

### `page_copy`
| Column | Type | Notes |
|---|---|---|
| key | string | Lowercase, snake_case |
| value | string | Free text, used by `{{page_copy.key}}` placeholders |

Reserved keys: `publish`, `publish_token`, `note` (filtered out before render).

The `page_copy` tab holds essentially all editable text on the homepage — section eyebrows, headings, body paragraphs, button labels, navigation, footer, meta tags, video IDs, and more. Rows whose `key` starts with `---` (e.g., `---HERO---`) are visual section separators in the Sheet and don't render anywhere on the site.

The list of keys evolves with the site. The current full inventory lives in the editor's guide and the `page_copy.csv` import file in the repo.

### `map_locations`
| Column | Type | Notes |
|---|---|---|
| id | string | Unique identifier per row |
| name | string | Org or project name |
| project | string | Optional subtitle (e.g., "23rd Avenue Community Building") |
| tags | string | Comma-separated; `clt`, `coop`, `nonprofit`, or `mobilehome`. Empty defaults to `clt`. |
| city | string | |
| region | string | "Bay Area", "Central Valley", or "Southern California" |
| lat, lng | float | Decimal coordinates |
| desc | string | Drawer body text |
| youtube | string | Optional YouTube URL — if present, marker click opens video modal |
| img | string | Optional image URL — if present, fills drawer media slot |
| url | string | Link button destination |
| urlLabel | string | Optional custom button label (default: "Learn More") |
| active | bool | `TRUE` to render, `FALSE` to hide |

### `partners`
| Column | Type | Notes |
|---|---|---|
| name | string | Display name |
| url | string | Link destination |
| logo_filename | string | Optional; relative path like `images/partners/acme.png` |
| active | bool | `TRUE` / `FALSE` |

### `resources`
| Column | Type | Notes |
|---|---|---|
| title | string | |
| description | string | Excerpt body |
| link | string | Required — usually a Drive URL |
| category | string | Display label, e.g., "Policy Brief" |
| date | string | YYYY-MM-DD; sorts newest-first; displays as "Mon YYYY" |
| active | bool | `TRUE` / `FALSE` |

### `stories`
| Column | Type | Notes |
|---|---|---|
| title | string | |
| organization | string | Org or publication credited |
| category | string | `news`, `story`, or `campaign` (defaults to `story`) |
| date | string | YYYY-MM-DD; sorts newest-first |
| media_url | string | YouTube URL (auto-embeds) or image URL |
| body | string | Optional excerpt |
| link_url | string | Optional "Read More" destination |
| active | bool | `TRUE` / `FALSE` |

### `model_cards`
| Column | Type | Notes |
|---|---|---|
| id | string | Unique identifier per row (slug-style: `clt`, `pha`, `permaff`, etc.) |
| type | string | `model` (appears under "Social Housing Models") or `principle` (appears under "Social Housing Principles") |
| title | string | Card title (the h3 heading) |
| body | string | Card body text |
| active | bool | `TRUE` / `FALSE` |

Row order in the Sheet determines display order on the page. To reorder cards, drag rows up or down. The build sorts cards by `type`, then preserves Sheet row order within each type.

## Known gotchas

**The publish_token cell-commit trap.** If the editor changes `publish_token` and clicks Run Workflow without first tabbing out of the cell, Google Sheets still has the old value saved. The build sees `unchanged` and exits. Always tab out, hit enter, or click another cell before triggering a build.

**The `&headers=1` gviz parameter.** The Sheet fetch uses `gviz/tq?tqx=out:csv&sheet=NAME&headers=1`. Without `&headers=1`, gviz auto-detects multi-row headers on text-heavy tabs and silently drops data rows. Don't remove this from the URL pattern in `build.py`.

**Cron is UTC, not Pacific.** The schedule in `build.yml` uses two cron entries (`16-23` and `0-1` UTC, weekdays) to span the UTC day boundary that falls in the middle of the Pacific business day. The window is intentionally about an hour wider than 9-to-6 to absorb DST without redoing the schedule twice a year.

**GitHub Pages caches aggressively.** After a successful build commit, the live site usually updates within 30-60 seconds, but the GitHub raw CDN can serve stale content for a few minutes longer. If verifying a new build, use incognito and hard-refresh (Cmd+Shift+R).

**Browser cache on the favicon.** Browsers cache favicons for a long time. After updating `favicon.svg`, users won't see the new icon until they hard-refresh or revisit the site days later. Acceptable for non-launch updates.

## Troubleshooting

**Build runs but nothing changes on the site.** Check that `publish_token` is actually different from the value in `.build-state.json`. If they match, the build exits without rendering. Bump the token.

**Build fails with `ERROR: SHEET_ID env var not set`.** The repo's Actions secret was renamed or deleted. Settings → Secrets and variables → Actions → re-add `SHEET_ID` with the long string from the Sheet's URL.

**Build runs but markers say `WARNING: marker BUILD:X not found`.** Someone hand-edited `index.html` and accidentally removed marker comments. Restore them by copying from a previous commit or from this repo's git history.

**Map pins don't appear after a publish.** Most likely cause: a row in `map_locations` has malformed `tags` or invalid lat/lng. Check the build log for parse warnings. Fall back: revert the most recent commit on `index.html`.

**GitHub Pages shows configuration error.** DNS issue. Verify the four A records at the registrar still point to GitHub's IPs (`185.199.108.153`, `.109.153`, `.110.153`, `.111.153`) and the CNAME for `www` points to `housing-now-ca.github.io`.

## DNS

| Record | Name | Value |
|---|---|---|
| A | @ | 185.199.108.153 |
| A | @ | 185.199.109.153 |
| A | @ | 185.199.110.153 |
| A | @ | 185.199.111.153 |
| CNAME | www | housing-now-ca.github.io |

Domain registrar: GoDaddy (Adriel Hampton has delegate access for the Housing Now! account as of cutover).

## Analytics

GoatCounter at `housingnowca.goatcounter.com`. The tracking script in `index.html` near `</body>` reports page hits. Privacy-respecting (no cookies, no individual user tracking). Dashboard requires the credentials Nathan registered with.

## Local development

There is no local dev environment. All edits happen via the GitHub web UI or directly in the Sheet. To test changes to `build.py`:

```bash
git clone https://github.com/Housing-Now-CA/socialhousingca.git
cd socialhousingca
pip install requests
SHEET_ID=15q5LP9tNGWSaxbTXQBr2-X_iLyqwOehxa1Q4K05Hxvk python scripts/build.py
```

This will rewrite `index.html` locally. Don't commit unless you intend to publish.

## Cutover playbook

Documented for future migrations (e.g., if the repo ever needs to move again).

1. **Old repo Pages settings** → release the custom domain. Skipping this causes GitHub to refuse domain verification on the new repo.
2. **New repo (empty)** → upload all files. Watch for hidden files (`.github/`, `.build-state.json`).
3. **New repo Settings → Secrets** → add `SHEET_ID`.
4. **New repo Settings → Pages** → enable, source `main`, add custom domain.
5. **DNS** → update A records and CNAME at registrar.
6. **Wait** 10-30 min for DNS propagation and GitHub HTTPS cert provisioning.
7. **Enable HTTPS** in Pages settings once the checkbox becomes clickable.
8. **Verify** site loads at the custom domain over HTTPS.
9. **Bump publish_token** and run workflow once to confirm the new pipeline works end-to-end.
10. **Update Search Console** to point to the new domain if applicable.

## Support

Originally built by The Adriel Hampton Group. For build-system questions or breakages, contact Adriel Hampton at adriel@adrielhampton.com.
