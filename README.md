# pihole-peek

A small shell tool that asks the **Pi-hole v6 REST API** which domains a client resolved, and
groups the answer by domain. It tells you what a device on your network talks to — the smart TV
that phones home, the phone that runs an ad SDK, the IoT box that never stops — and it exports the
result as a table, a plain domain list, CSV or JSON.

```
$ pihole-peek --client 192.0.2.70 --status blocked
DOMAIN                                                  HITS  LAST SEEN
nrdp25.logs.netflix.com                                  807  2026-09-11 23:35
unagi-eu.amazon.com                                      765  2026-09-12 14:14
eic.service.lgtvcommon.com                               469  2026-09-11 23:41
eu-acr92.alphonso.tv                                     399  2026-09-11 21:46
prov-lg.alphonso.tv                                      214  2026-09-11 23:35

192.0.2.70 · blocked · 5 domains · 2654 queries · last 24 h
```

Pi-hole v6 only. Version 5 used `admin/api.php` with an auth token, which this tool does not speak.

## Requirements

`bash` 4.2 or later, `curl`, `jq`, and the GNU `date` command.

## Install

```sh
git clone https://github.com/nightwatch75/pihole-peek.git
cd pihole-peek
ln -s "$PWD/pihole-peek" ~/.local/bin/pihole-peek     # any directory in your PATH
```

The script follows its own symlink, so the config file next to the real script is always found.

## Configuration

Three sources, in this order — **the command line wins over the environment, and the environment
wins over the config file**:

| Source | Where |
|---|---|
| command line | `--url`, `--client`, … |
| environment | `PIHOLE_URL`, `PIHOLE_CLIENT`, `PIHOLE_PASSWORD` |
| config file | a file named `config` next to the script, or the path in `PIHOLE_PEEK_CONFIG` |

Copy the example and edit it:

```sh
cp config.example config
chmod 600 config          # it can hold a password
```

```sh
PIHOLE_URL="http://pihole.example.lan"
#PIHOLE_CLIENT="192.0.2.70"
#PIHOLE_PASSWORD="change-me"
#PIHOLE_STATUS="blocked"
#PIHOLE_HOURS="24"
#PIHOLE_FORMAT="count"
#PIHOLE_INSECURE="0"
```

`config` is in `.gitignore`: your address and your password stay on your machine.
If the config file sets a default client, `--all-clients` puts the report back on every client.

## Usage

```sh
pihole-peek --list-clients                       # which clients does the Pi-hole see?

pihole-peek -c 192.0.2.70                        # blocked domains of one client, last 24 h
pihole-peek -c 192.0.2.70 -s allowed             # what it resolved instead
pihole-peek -c 192.0.2.70 -s all -t 6            # everything, last 6 hours
pihole-peek -c tv.lan -f list > tv-domains.txt   # one domain per line

pihole-peek                                      # every client, blocked, with a CLIENT column
pihole-peek -n 20 -f csv > report.csv            # top 20 rows as CSV
pihole-peek -d 'doubleclick|googleads' -s all    # only the domains that match a regex

pihole-peek --since '2026-09-12 00:00' --until '2026-09-12 08:00'
pihole-peek -u https://pihole.example.lan:8443 -k    # HTTPS with a self-signed certificate
pihole-peek -f raw | jq '.queries[] | .upstream'     # raw API answer, your own jq
```

### Options

| Flag | Environment | Default | What it does |
|---|---|---|---|
| `-u`, `--url URL` | `PIHOLE_URL` | — | Pi-hole base URL, e.g. `http://pihole.lan` or `https://pihole.lan:8443`. The API is at `URL/api`. A missing scheme becomes `http://`, a trailing `/admin` is removed. |
| `-c`, `--client ADDR` | `PIHOLE_CLIENT` | every client | client IP or hostname |
| `-a`, `--all-clients` | — | — | report every client, even when a default client is configured |
| `-s`, `--status SET` | `PIHOLE_STATUS` | `blocked` | which queries to export — see the table below |
| `-t`, `--hours N` | `PIHOLE_HOURS` | `24` | time window, hours back from now |
| `--since TS` | — | — | absolute start, in any format GNU `date -d` reads |
| `--until TS` | — | now | absolute end |
| `-d`, `--domain REGEX` | — | — | keep only the domains that match the regex (case insensitive) |
| `-n`, `--top N` | — | every row | keep only the first N rows |
| `-f`, `--format FMT` | `PIHOLE_FORMAT` | `count` | `count`, `list`, `csv`, `json` or `raw` |
| `--list-clients` | — | — | list the clients the Pi-hole knows, with their query count, and exit |
| `-k`, `--insecure` | `PIHOLE_INSECURE` | off | accept a self-signed TLS certificate |
| `--totp CODE` | — | — | two-factor code, when the Pi-hole asks for one |
| — | `PIHOLE_PASSWORD` | — | web or app password. There is no flag for it, so it never lands in your shell history. |
| `-V`, `--version` / `-h`, `--help` | — | — | |

## What to export: the status sets

`--status` takes a **group name** or a **comma separated list of FTL statuses**.

| Value | Statuses it covers | Use it for |
|---|---|---|
| `blocked` *(default)* | `GRAVITY`, `GRAVITY_CNAME`, `DENYLIST`, `DENYLIST_CNAME`, `REGEX`, `REGEX_CNAME`, `EXTERNAL_BLOCKED_IP`, `EXTERNAL_BLOCKED_NULL`, `EXTERNAL_BLOCKED_NXRA`, `EXTERNAL_BLOCKED_EDE15`, `SPECIAL_DOMAIN` | what the Pi-hole stopped |
| `allowed` | `FORWARDED`, `RETRIED`, `RETRIED_DNSSEC` | what really went out to the upstream resolver |
| `cached` | `CACHE`, `CACHE_STALE` | what was answered from the cache |
| `all` | every status | the full picture, blocked and allowed together |
| a list | e.g. `GRAVITY,DENYLIST` or `FORWARDED` | one exact status, or your own mix |

`GRAVITY` means a blocklist stopped it, `DENYLIST` an exact rule of yours, `REGEX` one of your
regular expressions. The `*_CNAME` variants are deep CNAME inspection: the domain itself is clean,
but it points at a blocked one.

> The API accepts one `status` value per request. `pihole-peek` therefore asks for the whole window
> and filters the group locally, so `--status blocked` is still a single HTTP request.

## Output formats

**`count`** (default) — a table sorted by hits, timestamps in your local time, and a summary line.
A `CLIENT` column appears when no client is selected.

```
DOMAIN                                               CLIENT              HITS  LAST SEEN
ms.applovin.com                                      192.0.2.128          947  2026-09-12 14:00
pubads.g.doubleclick.net                             192.0.2.128          332  2026-09-12 14:00

all clients · blocked · 2 domains · 1279 queries · last 24 h
```

**`list`** — the unique domains, one per line, sorted. Ready for a blocklist, a `diff` or `xargs`.

```
eic.service.lgtvcommon.com
it.info.lgsmartad.com
```

**`csv`** — `domain[,client],hits,status,last_seen`, sorted by hits, timestamps in UTC ISO-8601.

```csv
domain,hits,status,last_seen
"nrdp25.logs.netflix.com",807,"GRAVITY","2026-09-11T21:35:28Z"
```

**`json`** — the same records as an array, for a script downstream.

```json
[{ "domain": "nrdp25.logs.netflix.com", "hits": 807, "status": "GRAVITY", "last_seen": "2026-09-11T21:35:28Z" }]
```

**`raw`** — the API answer with no processing, for your own `jq`. Every field is there: query type,
upstream, reply time, DNSSEC state, CNAME chain.

With `--top N` the summary counts the rows that are shown, not the whole window.

## Authentication

`pihole-peek` asks `GET /api/auth` first and adapts:

* **no password** — nothing to do, the API answers `"no password set"`.
* **password** — put it in `PIHOLE_PASSWORD`, in the environment or in the config file. An **app
  password** (*Settings → Web interface / API*) is the better choice: it is revocable and it does
  not unlock the web interface.
* **two-factor** — add `--totp 123456`.
* **HTTPS with a self-signed certificate** — add `--insecure`.

The session is opened for the single run and closed with `DELETE /api/auth` when the script ends,
so it does not eat a session slot.

## The 24-hour limit

By default the API refuses to look further back than 24 hours, whatever `--since` says, because of
`webserver.api.maxHistory`. The long-term database usually holds much more. Raise the limit — one
week in this example:

```sh
curl -X PATCH http://pihole.example.lan/api/config \
  -H 'content-type: application/json' \
  -d '{"config":{"webserver":{"api":{"maxHistory":604800}}}}'
```

Or set `maxHistory` in `/etc/pihole/pihole.toml` and restart FTL. How far the data really goes back
is the `earliest_timestamp_disk` field of a `--format raw` answer.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | success |
| `1` | error — bad option, no address, no connection, authentication refused |
| `2` | no query matches the filters |

So a cron job can tell "nothing was blocked" from "the Pi-hole is down".

## License

MIT — see [LICENSE](LICENSE).
