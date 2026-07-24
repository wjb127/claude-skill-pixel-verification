# claude-skill-pixel-verification

> Part of [**claude-skills**](https://github.com/wjb127/claude-skills) — install everything with `/plugin marketplace add wjb127/claude-skills`

A [Claude Code](https://claude.com/claude-code) skill that verifies analytics and ad pixels are **actually firing** on a deployed site — captures screenshot evidence, generates a client-facing HTML report, and optionally **auto-fixes** broken installations in a loop.

> When a client says "the pixel isn't installed", you don't argue. You attach the report.

## What it verifies

The skill detects and validates the following platforms via real browser network capture (not just "is the script tag present"):

- **Google Analytics 4** — `gtag/js?id=G-XXXX`, `g/collect`, custom events
- **Google Ads** — `googleadservices.com/pagead/conversion/`
- **Google Tag Manager** — `gtm.js?id=GTM-XXXX`, dataLayer events
- **Meta Pixel** — `connect.facebook.net/.../fbevents.js`, `tr?id=` PageView/Lead/Purchase
- **TikTok Pixel** — `analytics.tiktok.com/api/v2/pixel/`
- **Naver Wcs** — `wcs.naver.com/wcslog.js`, `wcs_do`
- **Kakao Pixel** — `t1.daumcdn.net/kas/`
- **Bing UET** — `bat.bing.com/bat.js`
- **Hotjar** — `static.hotjar.com/c/hotjar-`
- **Microsoft Clarity** — `clarity.ms/tag/`

Beyond presence, it checks for:

- **Consent Mode V2** — all 4 signals (`ad_storage`, `analytics_storage`, `ad_user_data`, `ad_personalization`)
- **CAPI deduplication** — `event_id` parity between Meta browser-side and Conversions API server-side
- **Dual-installation issues** — `gtag` + GTM firing the same PageView twice
- **Trigger placement** — Lead/Purchase firing on the right element, not on every page load
- **GTM trigger timing** — fired before consent grant means dropped events

## Why this exists

AI-generated tracking installations have a few recurring failure modes:

- Placeholder IDs left in the code (`G-XXXXXX`, `GTM-XXXXXX`)
- UA tags (`UA-`) instead of GA4 (`G-`) — silent for months
- Both `gtag` and GTM installed, doubling all event counts
- Consent Mode V2 missing → Google Ads remarketing audience drops to zero
- Lead event triggers on page load instead of form submit
- `event_id` mismatch between browser and CAPI → duplicate conversions

This skill catches all of the above before the client does.

## Workflow (the client-proof part)

1. Skill scans the codebase + crawls the deployed URL with Playwright.
2. Captures every outbound tracking request (`page.on('request')`).
3. Triggers key events: PageView, Lead form submit, button clicks.
4. Saves screenshots of the Network panel state + a structured JSON of every captured hit.
5. Generates an HTML report you can hand to the client: "Here's the proof, every pixel fired with the correct parameters at the correct moment."

## Auto-Fix Loop

Pass `--loop` and the skill will:

```
detect issue → propose fix → apply → build → deploy → re-verify → repeat
```

with safety rails:

- **Max 5 iterations** (configurable via `--max-loops N`)
- **30-minute timeout** total
- **STUCK detection** — if the same diagnosis appears 3 times in a row, the loop exits
- **`git stash` before every loop** — instant revert if a build breaks
- **Escape conditions** — build failure, deploy failure, user input required (e.g., new API key)

### Auto-fixable (9 categories)
Placeholder IDs, UA → GA4 migration, gtag/GTM duplication, missing page tags, Consent Mode V2 signals, duplicate PageView, Lead trigger placement, GTM trigger timing, missing `data-testid` for trigger targets.

### Requires human (8 categories)
New ad account credentials, CAPI access token, GTM container access, server-side tagging endpoints, custom dimension setup in GA4 UI, conversion API tokens, audience definitions, anything that needs login at facebook.com / analytics.google.com.

## Installation

```bash
git clone https://github.com/wjb127/claude-skill-pixel-verification.git
mkdir -p ~/.claude/skills
cp -r claude-skill-pixel-verification ~/.claude/skills/verify-pixels
```

Or symlink:

```bash
ln -s "$(pwd)/claude-skill-pixel-verification" ~/.claude/skills/verify-pixels
```

Restart Claude Code, then:

```
/verify-pixels
/verify-pixels --loop
/verify-pixels --loop --max-loops 3 --no-deploy
/verify-pixels --interactive
/verify-pixels --dry-run
```

## Options

| Flag | Effect |
|---|---|
| `--loop` | Enable auto-fix loop (default off) |
| `--max-loops N` | Cap iterations (default 5) |
| `--no-deploy` | Apply fixes locally only, skip Vercel/host deploy |
| `--interactive` | Prompt before applying each fix |
| `--dry-run` | Diagnose only, no edits |

## Output

After a run you get:

```
report-pixels-YYYYMMDD.html      # Client-facing evidence
report-pixels-YYYYMMDD.json      # Raw captured hits for diffing
screenshots/<platform>-<ts>.png  # Per-pixel proof shots
loop-history.json                # Each loop's diagnosis + fix + result (when --loop)
```

## Companion skill

For Unit / Integration / E2E test generation, see **[claude-skill-web-testing](https://github.com/wjb127/claude-skill-web-testing)**.

## License

MIT
