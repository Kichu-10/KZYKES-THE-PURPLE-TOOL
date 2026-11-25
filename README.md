You are building a dual-purpose security toolkit called **(KZYKES) — The Purple Tool** — a modular system auditing framework with both offensive (red-team) and defensive (blue-team) capabilities. It must be production-grade, bug-free, and self-contained.

# DELIVERABLE STRUCTURE

Produce a complete, runnable repository with this exact layout:
TODAY ALSO NO WORK

```
kzykes/
├── kzykes.py                    # unified CLI entrypoint
├── dashboard.py                 # Flask web dashboard entrypoint
├── requirements.txt
├── README.md
├── setup.py                     # pip-installable
├── pyproject.toml
├── core/
│   ├── __init__.py
│   ├── config.py                # config + profile loader (yaml)
│   ├── logger.py                # structured logging (json + human)
│   ├── engine.py                # module orchestration + rate limit + retry
│   ├── registry.py              # module auto-discovery
│   ├── reporter.py              # JSON / CSV / Markdown / HTML / SARIF
│   ├── cache.py                 # sqlite result cache
│   └── safety.py                # scope allowlist + dry-run guard
├── offensive/
│   ├── __init__.py
│   ├── portscan.py              # TCP/UDP connect + SYN-lite scan
│   ├── banner.py                # service fingerprint
│   ├── subdomain.py             # crt.sh + brute + DNS resolve
│   ├── dirbust.py               # directory/file enumeration
│   ├── webrecon.py              # headers, robots, sitemap, tech detect
│   ├── xss.py                   # reflected + stored + DOM detection
│   ├── sqli.py                  # error + boolean + time-based
│   ├── idor.py                  # id walking + bypass matrix
│   ├── ssrf.py                  # internal probe
│   ├── csrf.py                  # token absence check
│   ├── lfi.py                   # traversal detection
│   ├── openredirect.py          # redirect param fuzz
│   ├── auth.py                  # login brute + lockout detect
│   ├── jwt.py                   # alg confusion + weak secret
│   └── secrets.py               # regex secrets in responses
├── defensive/
│   ├── __init__.py
│   ├── logaudit.py              # parse auth/system logs for indicators
│   ├── filemon.py               # hash baseline + drift detect
│   ├── persaudit.py             # cron/systemd/rc/startup audit
│   ├── useraudit.py             # sudoers + shadow hygiene
│   ├── netaudit.py              # listening ports + firewall check
│   ├── hardening.py             # sysctl + ssh + mount recommendations
│   ├── yara_lite.py             # regex indicator matcher
│   ├── honey.py                 # honeypot port listener + alert
│   └── report_card.py           # score system against CIS subset
├── dashboard/
│   ├── app.py                   # flask app
│   ├── templates/
│   │   ├── base.html
│   │   ├── index.html
│   │   ├── results.html
│   │   ├── module.html
│   │   └── jobs.html
│   └── static/
│       ├── css/style.css
│       └── js/app.js
├── profiles/
│   ├── quick.yaml               # fast recon
│   ├── full.yaml                # everything
│   ├── web.yaml                 # web-only offensive
│   └── defense.yaml             # defensive audit
└── tests/
    ├── test_engine.py
    ├── test_reporter.py
    ├── test_safety.py
    └── test_modules.py
```

# HARD REQUIREMENTS

## 1. CLI (`kzykes.py`)

Subcommands:

- `scan <target> [-m modules] [-p profile] [-o out] [--dry-run]` — run offensive or defensive modules
- `audit` — defensive suite only, no args, audits the local host
- `list` — list all modules with side (`offensive`/`defensive`), description, accepted targets
- `show <module>` — print module manifest (params, side, safety level)
- `dashboard [--port 5000]` — launch web dashboard
- `version`
- `init-config` — write default `kzykes.yaml`

Every subcommand must:
- accept `--config <path>` (default `./kzykes.yaml`)
- honor `--quiet` (suppress stdout, keep logs)
- accept `--log-level {debug,info,warn,error}`
- accept `--json` (machine-readable output)
- exit non-zero on any module error (unless `--allow-fail`)

CLI must be built with **argparse** (no external CLI lib) and must print colored output via `rich` if installed, plain text otherwise.

## 2. Dashboard (`dashboard.py`)

Flask app with these routes:

- `GET /` — overview: recent jobs, module counts, system health card (defensive score)
- `GET /results` — list every saved result file (json) with target, modules, timestamp
- `GET /results/<id>` — inspect one result, tabbed by module
- `GET /modules` — visual grid of all modules with side badge (red/blue), click → detail
- `GET /jobs` — live job queue (in-memory) with progress bar
- `POST /api/scan` — trigger a scan from UI (target, module list, profile)
- `POST /api/upload` — drag-drop a result json
- `GET /api/results` — json list
- `GET /api/results/<id>` — json blob
- `GET /api/health` — returns `{status, uptime, modules_loaded}`

UI requirements:
- **purple theme** — `#0d1117` bg, `#a371f7` primary accent, `#7c3aed` secondary, `#3fb950` success, `#f85149` danger — The Purple Tool should *look* purple
- responsive, no external CSS framework (vanilla CSS)
- live job polling via `fetch` every 2s
- no authentication for local, but a `--auth` flag adds basic HTTP auth
- runs on `127.0.0.1:5000` by default

## 3. Module contract

Every module is a `.py` file in `offensive/` or `defensive/` with this exact shape:

```python
NAME = "xss"
SIDE = "offensive"          # offensive | defensive
DESCRIPTION = "Detect reflected, stored, and DOM-based XSS"
ACCEPTS = ["url"]           # url, domain, ip, email, username, host, file
SAFETY = "active"           # passive | active | intrusive
PARAMS = {
    "timeout": {"type": "int", "default": 10},
    "payloads": {"type": "str", "default": "default"},
}

async def run(ctx) -> dict:
    """
    ctx = {
        target, config, session (aiohttp), cache, logger,
        params (dict merged from CLI + module PARAMS defaults),
        dry_run (bool),
    }
    Returns a dict that becomes the module's result.
    """
```

Modules must:
- be idempotent (safe to run twice)
- honor `ctx["dry_run"]` — print what *would* be sent, send nothing
- honor `ctx["params"]` — never hardcode a value that's in PARAMS
- return a dict, not a string, not a list
- raise nothing — catch internally, return `{"error": str(e)}`
- never print to stdout directly — use `ctx["logger"]`

## 4. Safety layer (`core/safety.py`)

Mandatory guardrails:

- **scope allowlist** — config `scope.allow` is a list of CIDRs / domains. Every target must match; otherwise refuse with a clear error.
- **dry-run** — `--dry-run` disables every module from sending network traffic that mutates state (POST/PUT/DELETE) or runs an intrusive payload (SQLi timing, brute force). It must still run passive checks.
- **rate limit** — per-module token bucket, configurable in yaml (`rate.default: 10`, `rate.per_module: {portscan: 500}`)
- **never** run destructive modules against anything not in `scope.allow`

## 5. Engine (`core/engine.py`)

- async orchestration with `asyncio.Semaphore` for concurrency
- per-module retry (max 2, exponential backoff) on network errors only
- per-module timeout wrapper — kill and record `{"error": "timeout"}` if it exceeds `config.timeout * 3`
- progress callback → dashboard job queue
- result aggregation + deduplication
- on `KeyboardInterrupt`, cancel cleanly and still write partial results

## 6. Reporter (`core/reporter.py`)

Five output formats:

- **json** — full nested dict, one file per scan
- **csv** — flattened `module,key,value` rows
- **markdown** — human report with per-module sections, summary card at top
- **html** — self-contained single-file report (inline CSS, purple theme, no external deps), with collapsible sections
- **sarif** — SARIF 2.1.0 for CI integration (offensive modules map to security findings; defensive to configuration violations)

## 7. Offensive modules — behavior spec

- **portscan** — TCP connect scan on top 1000 ports, `--full` for 65535; UDP on top 50; banner on open; service guess table
- **banner** — probes open ports with protocol-specific payloads (HTTP, SSH, SMTP, FTP, MySQL), returns raw banners + parsed product/version
- **subdomain** — crt.sh + optional wordlist brute (`wordlists/subdomains.txt` — ship 500 entries), DNS A/AAAA/CNAME resolve, wildcard detection
- **dirbust** — common paths (`admin`, `api`, `backup`, `.git`, `.env`, `robots.txt`) with response-code filtering; wordlist optional
- **webrecon** — HTTP headers, tech fingerprint (server, framework, cdn, waf), robots.txt, sitemap.xml, security header gaps
- **xss** — reflected (URL/body echo), stored (canary re-fetch), DOM (source→sink regex on inline scripts); respects `payloads` param
- **sqli** — error-based (`'`, `"`, `)`), boolean (`' AND 1=1`, `' AND 1=2`), time-based (`' OR SLEEP(5)`) — skips time-based in dry-run
- **idor** — id walking (1..N) with baseline diff, then bypass matrix (header/path/method/cookie/auth)
- **ssrf** — probes URL params with `127.0.0.1`, `169.254.169.254`, `file:///etc/passwd`; flags internal responses
- **csrf** — parses forms, flags ones without tokens on state-changing endpoints (POST/PUT/DELETE)
- **lfi** — `../../etc/passwd`, `php://filter`, `....//` variants; detects by response diff
- **openredirect** — tries `?next=`, `?redirect=`, `?url=`, `?return=` with external domain
- **auth** — login endpoint brute with a small wordlist (`admin/admin`, `admin/password`, etc.), detects lockout after N failures
- **jwt** — decodes JWT in cookies/headers, tries alg=none + weak secret dict (10 entries) against signature
- **secrets** — scans response bodies for API keys, AWS, GitHub tokens, private keys, JWT secrets

## 8. Defensive modules — behavior spec

- **logaudit** — parses `/var/log/auth.log`, `/var/log/secure`, `/var/log/syslog` for failed-login spikes, sudo anomalies, root logins off-hours
- **filemon** — creates hash baseline (`~/.kzykes/baseline.json`) on first run, on subsequent runs reports added/removed/modified files under configured paths
- **persaudit** — enumerates cron (`/etc/cron*`, `crontab -l`), systemd units, rc.local, `.bashrc`, `.profile`, autostart — flags anything not in allowlist
- **useraudit** — user list, sudoers parse, shadow permissions, empty passwords, UID 0 accounts besides root
- **netaudit** — `ss -tulpn` equivalent, compares against expected listen ports; checks firewall status (ufw/iptables/firewalld)
- **hardening** — reads `/etc/sysctl.conf`, `/etc/ssh/sshd_config`, `/etc/fstab`, checks for: ASLR, ptrace scope, ssh root login, ssh password auth, mount nosuid/nodev/noexec on /tmp — each check returns a pass/fail with a fix command
- **yara_lite** — regex indicator matching against files in a directory (ship `indicators/default.yml` with 20 rules)
- **honey** — spawns listener on ports 2222, 3389, 5900 (configurable); any connection logs the source IP with timestamp; alerts to dashboard
- **report_card** — aggregates every defensive module's findings into a 0–100 CIS-subset score with per-category breakdown

## 9. Tests

`pytest` suite covering:

- engine: module discovery, timeout, retry, cancellation
- reporter: every format roundtrips valid output
- safety: scope allowlist rejects out-of-scope target; dry-run blocks outbound POST
- modules: at least one smoke test per module using a local Flask test server (`127.0.0.1` only, no external)

Coverage target: ≥ 70% on `core/`.

## 10. README

Must include:

- install (pip + venv + docker)
- quickstart (one command per subcommand)
- module table with side + safety + accepted target types
- scope/safety configuration walkthrough
- dashboard screenshots (described in text, purple theme)
- SARIF + CI example (GitHub Actions snippet)
- contribution guide (module contract reference)
- license section

## 11. Requirements

```
aiohttp>=3.9
dnspython>=2.4
flask>=3.0
flask-socketio>=5.3
pyyaml>=6.0
rich>=13.0
requests>=2.31
beautifulsoup4>=4.12
Pillow>=10.0
pytest>=8.0
eventlet>=0.35
```

## 12. Quality bar

- every module must run without raising on: empty response, 500 status, malformed JSON, connection refused, DNS failure
- no `print()` outside `core/logger.py`
- no hardcoded credentials, secrets, or API keys
- all file writes go to `~/.kzykes/` or `./results/`
- all network calls honor `ctx["config"]["timeout"]`
- code passes `ruff check` and `black --check` with no errors (ship `pyproject.toml` with both configured)
- every public function has a docstring
- every module returns `{"error": ...}` instead of raising
- no external binary dependencies (no `nmap`, no `nikto`, no `ffuf`) — everything pure Python

## 13. What "done" looks like

Run these and get clean output:

```bash
kzykes.py list
kzykes.py init-config
kzykes.py scan 127.0.0.1 -m portscan,banner,webrecon -o out.json
kzykes.py audit
kzykes.py dashboard --port 5000 &
curl http://127.0.0.1:5000/api/health
pytest tests/ -v
```

You must **generate every file** in the layout above with complete, working code — no placeholders, no `# TODO`, no `pass` except in exception handlers. If any module is too large to fully render in one reply, split across multiple replies and confirm each file is complete before moving on.

Begin with: `requirements.txt`, `setup.py`, `pyproject.toml`, and `core/config.py`. Then move through `core/` → `offensive/` → `defensive/` → `dashboard/` → `tests/` → `README.md`. Ask nothing. Produce.
