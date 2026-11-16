# PlutosCookies

A PyQt6 desktop tool for checking cookie/session dumps against 39 sites,
capturing account intel from live sessions, scraping credential dumps for
cards/tokens/IPs, and validating proxy lists — all local, all in one window.

## Requirements

- Python 3.10+
- `pip install PyQt6 curl_cffi playwright` (Playwright browsers only needed
  for the Claude fallback path: `playwright install chromium`)

Run: `python plutoscookies.py`

## Tabs

### Home
Dashboard with session stats and quick navigation.

### Checkers
Single-file / pasted-cookie checker.

- **Input**: pick a cookie dump file (or paste raw cookies) and choose sites
- **Check**: validates each site's session live and shows `VALID` / `INVALID`
  / `ERROR` per site with the account identity when valid
- **Hits**: valid results are written immediately to `hits/tester/<site>/`
  as Netscape cookie exports with captured account fields in the header —
  nothing waits for the end of the run
- **Export**: dump all results to `.txt`
- Every table column is drag-resizable, hover shows the full cell text, and
  double-click opens a read-only viewer for long values

### Sessions
Batch scanning of whole folders of dumps.

- Create named scan sessions pointing at a folder; each session tracks
  progress (files done / total), hits count, and live per-file log
- One proxy sticks to a whole file's checks (round-robin across files) so a
  session's multi-request check never hops IPs mid-flight
- Valid hits save per-site under `hits/<session>/<site>/` as they're found
- Start / Stop / View per session; sessions persist in `scan_sessions.json`
  across restarts

### Dump Scraper
Regex extraction over dump folders — passwords, credit cards, Discord
tokens, FTP creds, SSH/RDP/proxy IPs.

- **Credit cards** only captured with the full `number MM/YY CVV` tail
  (expiry + 3-4 digit CVV required; Luhn-checked)
- **Discord tokens** are validated live via `/users/@me` — output shows
  `VALID · account` / `INVALID` / `ERROR · reason`
- Results stream to disk **while scanning**: master log
  `scraped_<timestamp>.txt` plus per-type files
- Output lands in `hits/<folder name>/dumpscrape/`:
  `Passwords.txt`, `DiscordTokens.txt`, `Cards.txt`, `FTP.txt`, `SSH.txt`,
  `RDP.txt`, `Proxies.txt`, `IPPort.txt`, `IPs.txt`
- Actions: Start / **Stop** / Save / Dedupe / Clear; progress + item count
  live; table capped at 20k rows (log file holds everything)
- Engine: chunked regex scanning (~1MB chunks) so the GUI stays responsive
  on multi-GB folders; configurable thread count in Settings

### Settings
- **Performance**: checker threads, headed threads, scrape threads,
  request timeout, max file size, Playwright wait
- **Scan**: cookies-only mode, save hit files, priority scan, live log
- **Proxy** (see below)
- **Notifications**: Telegram/Discord webhook fields (coming soon, disabled)
- **Debug mode** toggle exposes the event console

### Debug
Event console — gated behind Settings → Debug Mode.

## Proxy system

Settings → Proxy Settings:

- **Browse** accepts any proxy format: `host:port`,
  `host:port:user:pass`, `user:pass@host:port`, `http(s)://`,
  `socks4/5://`, `socks5h://`. Uploads **stack** into `proxies.txt`
  (deduped, order kept) — the unchecked database
- **Proxy Check Threads**: concurrent tests, 1–100000, default 50
- **▶ Start** tests the unchecked queue manually (no auto-test). Testing
  hits `api.ipify.org` through each proxy via curl_cffi:
  - `200` + IP body → valid
  - `200` + junk → invalid (captive portal)
  - any 4xx/5xx incl. **429 → valid** (rate-limit ≠ dead proxy)
  - timeout/conn failure → one retry → invalid
- On a **completed** run: passers merge into `proxies.valid.txt` (the valid
  database), the unchecked file is wiped, invalids are deleted. **Stop**
  leaves the queue untouched
- Both files persist across restarts; Reset to Defaults clears them
- Scans prefer `proxies.valid.txt` automatically when **Use Proxy** is on;
  Discord token checks in the scraper also rotate through the valid list
  (each proxy = separate Discord rate bucket, ~8 req/s vs ~3/s direct)

## Site checkers

Custom dedicated checkers (requests-first, Playwright fallback where noted):

| Site | Method |
|---|---|
| ChatGPT | session API + JWT backend-api, full billing/payment capture |
| Reddit | `/api/me.json` — karma/age/premium/mod capture |
| Claude AI / Claude Platform | requests-first, browser on ambiguous |
| Roblox | `.ROBLOSECURITY` token refresh chain |
| LinkedIn | voyager/api/me (needs `JSESSIONID` + `li_at`) |
| Kleinanzeigen | authenticated-page redirect + Astro island props |
| Xbox/Microsoft | ms_capture port |

The remaining ~31 sites (Instagram, TikTok, Twitter/X, Facebook, Google,
Steam, Netflix-style generics, etc.) use the shared generic checker —
expect false-negatives on Cloudflare/login-walled sites. See
`checker_status.txt` for the full module table and known gaps.

## Capture

On a VALID result the checker hits the site's account/profile endpoints
(`CAPTURE_SPECS` in `sites.py`) and pulls identity + account intel —
gamertag, email, plan/subscription, payment flags, rewards — stored in the
hit file header and shown in results.

## Output layout

```
hits/
  tester/<site>/           single-check hits
  <session>/<site>/        batch scan hits
  <folder>/dumpscrape/     scraper output (master log + per-type files)
proxies.txt                unchecked proxy database
proxies.valid.txt          validated proxy database
scraped_<ts>.txt           scraper master log (app dir fallback)
scan_sessions.json         saved scan sessions
settings.json              app settings
checker_status.txt         checker module status / TODO list
```

Hit files are Netscape cookie exports (or raw token for token-based hits)
with `#`-prefixed capture fields, prefixed `[Plan] Site__name.ext` when
plan info is captured.

## Notes

- All network I/O uses `curl_cffi` with Chrome TLS impersonation — plain
  `requests` gets fingerprinted/blocked by Cloudflare-fronted sites
- Discord token checks are globally throttled (~3 req/s direct, ~8 req/s
  proxied) with `Retry-After` honoring — big token dumps take minutes by
  design
- IP-sensitive sessions (TikTok `sid_guard`, Roblox) can false-negative
  through proxies in the wrong region
