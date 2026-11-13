# SnipeScout

A self-hosted second-hand deal sniper. It watches **2dehands, Marktplaats, Vinted and Facebook Marketplace** around the clock and pushes new listings to **Telegram** the moment they appear, with photo, price, bargain score and a direct link.

Everything runs on your own machine. No account on any of the platforms is required except for Facebook, no data leaves your computer apart from the searches themselves, and the dashboard is bound to `127.0.0.1`.

![Snipers tab](docs/dashboard-snipers.png)

---

## Why

Good deals on second-hand marketplaces are gone within minutes. Refreshing search pages by hand does not scale, and the official apps only offer coarse e-mail alerts. SnipeScout polls the platforms' own search endpoints on a short interval, applies filters that the sites themselves do not offer (drawn map zones, exclude-words, a real "posted in the last X days" window), and notifies you instantly.

## Features

- **Four platforms in one place** — 2dehands.be, Marktplaats.nl, Vinted and Facebook Marketplace.
- **Vinted across borders** — searches `be, nl, fr, de, es, it, pt, lu, ie` in one pass. Vinted runs a single cross-border catalogue, so the same item shows up on several domains; items are deduplicated by id and alerted once.
- **Instant Telegram alerts** with photo, price, location, bargain score and an *Open listing* button. Snipers can also be created and toggled from Telegram.
- **Geofencing that the platforms do not offer** — draw include and exclude polygons on an OpenStreetMap canvas (Leaflet.draw), for example "60 km around Ghent but not Brussels". The platforms only accept a circle, so SnipeScout queries the smallest enclosing circle and filters the results against your polygon locally, using a GeoNames postcode table.
- **Per-sniper radius** around any postcode or town, independent of your home location.
- **"Posted in the last X days"** — backfills recent listings on the first run and ignores older ones afterwards, so promoted ads that resurface do not alert twice.
- **Deterministic matching.** Whether a listing matches is decided by your own rules, never by an LLM.
- **Optional Gemini** for two things only: generating search synonyms, and scoring how good a price is (1–10).
- **A Results tab that shows its work** — every listing each platform returned in the last round, with the exact reason it was or was not alerted.
- **Polite by default** — randomised intervals, per-platform minimums, exponential backoff on 403/429, and automatic 30-minute pausing of a Vinted country that keeps failing.

![Results tab](docs/dashboard-results.png)

## How a listing is judged

In this order. The first rule that fails stops the listing, and the reason is recorded in the Results tab.

| # | Check | Notes |
|---|-------|-------|
| 1 | Already seen? | Deduplicated per sniper via SQLite |
| 2 | Age | Only when "posted in the last X days" is set. No date from the platform counts as recent |
| 3 | Search words | Every word of at least one search term must appear in the title or description (optional) |
| 4 | Price | Min/max, with "make an offer" listings optional |
| 5 | Exclude words | Accent- and case-insensitive substring match |
| 6 | Location | Radius or polygon, resolved through the GeoNames postcode table. Skipped for Vinted (items are shipped) |
| 7 | Score | Rule-based from price vs. median and max; Gemini may refine the number, but can never reject the listing |

Keeping the LLM out of steps 1–6 is deliberate: an API outage, a rate limit or a bad answer can slow the score down, but it can never make you miss a deal.

## Requirements

- Python 3.10+
- ~400 MB of disk for the Chromium build that Playwright downloads (only needed for Facebook)
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- Optional: a Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey)

## Quick start (Windows)

```
git clone https://github.com/<you>/snipescout.git
cd snipescout
start.bat
```

`start.bat` creates a virtual environment, installs the dependencies, downloads Chromium, copies `.env.example` to `.env` and opens it in Notepad, then starts the app and your browser at <http://localhost:8000>.

Link Telegram once by sending `/start <code>` to your bot, using the code shown at the top of the dashboard.

For Facebook, run `fb_login.bat` once and log in. The session is stored in `data/fb_profile`. Use a throwaway account.

## Quick start (Linux / macOS)

Nothing in the Python code is Windows-specific; only the two `.bat` launchers are.

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
playwright install chromium        # only needed for Facebook
cp .env.example .env               # then fill in your tokens
python -m snipescout
```

## Configuration

Secrets and machine-level settings live in `.env`, which is gitignored. Everything else is configured in the dashboard and stored in SQLite.

| Variable | Default | Meaning |
|----------|---------|---------|
| `TELEGRAM_BOT_TOKEN` | – | Bot token from @BotFather. Without it, alerts are disabled |
| `GEMINI_API_KEY` | – | Optional. Without it, synonyms and AI scoring are disabled |
| `GEMINI_MODEL` | `gemini-flash-latest` | Any Gemini model id |
| `GEMINI_THINKING` | `low` | `minimal`, `low`, `medium` or `high`. Higher is slower |
| `GEMINI_MAX_PER_MINUTE` | `10` | Client-side rate limit, to stay inside the free tier |
| `BACKFILL_ALERT_LIMIT` | `25` | Max alerts per search term on a sniper's first round |
| `DASHBOARD_PORT` | `8000` | Dashboard port, bound to 127.0.0.1 only |
| `FB_HEADLESS` | `true` | Set to `false` to watch the Facebook browser work |

## Telegram commands

```
/new iPhone 16 Pro max 450 days 3 not defect, scherm kapot
/new scooter min 200 max 900
/list            all snipers
/on 3  /off 3  /delete 3
/pause  /resume  /status
```

Only the chat that completed the pairing handshake can issue commands.

## Architecture

```
snipescout/
├── __main__.py       entry point: logging, geo load, scheduler, uvicorn
├── config.py         .env parsing, platform table, Vinted domains, defaults
├── db.py             SQLite schema, migrations, snipers/zones/seen/matches
├── geo.py            GeoNames postcodes, haversine, point-in-polygon, enclosing circle
├── net.py            curl_cffi session (Chrome TLS fingerprint), async wrapper
├── scheduler.py      one asyncio loop per platform, jitter + exponential backoff
├── matcher.py        the filter chain, dedup, scoring, verdict recording
├── ai.py             Gemini: synonyms + price score, rate limit, cooldown
├── telegram.py       long-polling bot: alerts and commands
├── web.py            Starlette JSON API + dashboard, localhost-only guard
├── platforms/
│   ├── adevinta.py   2dehands + Marktplaats (shared backend)
│   ├── vinted.py     multi-country catalogue API with per-domain tokens
│   └── facebook.py   Playwright on a persistent, logged-in profile
└── static/           vanilla JS dashboard, no build step
```

Design notes worth knowing if you want to hack on it:

- **One asyncio loop per platform.** A slow or blocked platform never holds up the others. Each loop has its own interval, jitter and backoff multiplier.
- **`curl_cffi` instead of `requests`.** It impersonates Chrome's TLS and HTTP/2 fingerprint, which is what gets you past most of the basic bot filtering.
- **Vinted auth.** The `www` host sets an anonymous `access_token_web` cookie; that bearer token is then used against `api.vinted.<tld>/svc-catalogue/items`. Tokens are refreshed every 30 minutes and on any 401/403. Each country keeps its own session.
- **Dates are messy.** Adevinta returns localised strings such as `Vandaag`, `Gisteren` or `31 okt 26`; Vinted usually sends no date at all; Facebook only filters coarsely by 1, 7 or 30 days. `parse_date` normalises what exists, and an unknown date is treated as recent rather than dropped.
- **The dashboard is hardened for a local app**: `Host` must be loopback (blocks DNS rebinding), writes must be same-origin JSON (blocks CSRF from any page you have open), and all user input is validated server-side.
- **No build step.** The frontend is plain HTML/CSS/JS; Leaflet is the only CDN dependency.

## Limitations

- None of these APIs are public or documented. They change without notice, and when they do, that platform breaks until the adapter is updated. Vinted already moved its catalogue endpoint once.
- Facebook is the most fragile by far: it needs a logged-in Playwright profile, its DOM changes often, and automating it can get the account restricted.
- Locations resolve at town level, because listings do not expose exact addresses.
- Vinted ignores zones and radius, since items are shipped rather than collected.
- The machine has to stay awake. There is no cloud component.
- Running from a datacenter IP (VPS) gets blocked considerably faster than a residential connection.

## Disclaimer

This is a personal project, built for personal use, and is not affiliated with 2dehands, Marktplaats, Vinted, Meta or Google.

Automated access may conflict with those platforms' terms of service. You are responsible for how you use it. Keep the intervals sane, do not redistribute scraped data, and expect that an account used for automation may be restricted. The defaults are intentionally conservative for this reason.

## License

MIT. See [LICENSE](LICENSE).
