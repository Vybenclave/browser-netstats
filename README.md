# Browser Network Toolkit

A single-page, no-build kit of network tools that run in the browser. Dark
Tokyo Night theme, cyan/magenta ping graph on a 25% grey grid.

**Live page:** https://vybenclave.github.io/browser-netstats/

Five tools, laid out as a card grid:

| Tool | Where it runs |
| ---- | ------------- |
| **Ping** | timed in your browser (HTTPS round-trip) |
| **Traceroute** | on a [Globalping](https://globalping.io) probe |
| **DNS lookup** | on a Globalping probe (pick the resolver) |
| **Web probe** | on a Globalping probe |
| **TLS certificate** | local file (default) or a Globalping probe |

Every card has a **Share result** button. It opens a small menu:

- **Share…** - only when the browser supports it (phones, some desktops):
  hands a `.csv` (or `.txt`) file to the OS share sheet, so the result can go
  to Files, Drive, Messages, Mail, AirDrop, etc.
- **Save .txt** - a human-readable report.
- **Save .csv** - the tabular data (ping samples, hops, DNS answers, port
  grid, cert fields), where it makes sense.
- **Copy text** - the report to the clipboard.

The button stays usable on the probe-backed cards even after Globalping goes
offline, so a result you already have can still be shared.

## Ping

Fires a request every **500 ms** and draws one vertical bar per ping; bar
height is the RTT in milliseconds. Cyan bars are successful pings, magenta
bars are lost packets, the magenta line is an 8-sample rolling average, and
the y-axis auto-scales to the 95th percentile so one slow sample doesn't
flatten the rest. Live metrics: last, min / avg / max, jitter, total pings,
packet loss %.

Presets are one endpoint per provider and coast:

| | East | West |
| --- | --- | --- |
| **AWS** | `dynamodb.us-east-1.amazonaws.com` | `dynamodb.us-west-2.amazonaws.com` |
| **Azure** | `eastus.api.cognitive.microsoft.com` | `westus.api.cognitive.microsoft.com` |
| **Google** | `us-east1-aiplatform.googleapis.com` | `us-west1-aiplatform.googleapis.com` |

Or type any host / IP into the custom box.

**How the "ping" works** - a browser can't send ICMP, so this times an HTTPS
request instead:

```js
fetch("https://<host>/?_bnt=<now>", { mode: "no-cors", cache: "no-store" })
```

`no-cors` means the response is opaque - we never read it, only time it - so
the target needs no CORS headers. An `AbortController` cancels after 2 s; an
abort or network error is recorded as a lost packet. Values include DNS / TCP
/ TLS setup, so they read higher than a real `ping`, and the first request to
a host is the slowest.

**Share result** carries the last 60 seconds: a summary plus a
`unix_ms, iso_time, target, rtt_ms, status` CSV (`status` = `ok` | `lost`).

## Traceroute

A browser genuinely cannot run traceroute - no raw sockets, no TTL control -
so this submits the job to the **Globalping** API, which runs it from a probe
in the network of your choice and returns the hops. Free and keyless, roughly
250 runs/hour per IP.

- **target** - host or IP.
- **max hops** - trims the result to N rows (1-20; Globalping's probes cap at
  20).
- **probe location** - a Globalping "magic" location: `US`, `New York`,
  `us-east-1`, `AWS`, `Comcast`, etc.
- **protocol** - ICMP (default), TCP, or UDP.

Output is a hop table (`# / host / ip / rtt`) plus the raw probe output.
**Share result** carries the raw traceroute as `.txt` and a
`hop, hostname, ip, rtt1_ms, rtt2_ms, rtt3_ms` CSV.

## DNS lookup

Also runs on a Globalping probe, which is what makes the **resolver** a choice:
**Default** (the probe's own resolver), **Cloudflare** `1.1.1.1`, **Google**
`8.8.8.8`, **Comcast** `75.75.75.75`, or **Custom** (any resolver IP or FQDN).
`75.75.75.75` only answers Comcast subscribers, so picking Comcast also pins
the probe onto Comcast's network (AS7922).

Types: A, AAAA, CNAME, NS, MX, TXT, SOA, SRV, PTR, plus two shortcuts:

- **SPF** - queries TXT at the name and highlights the `v=spf1` record.
- **DMARC** - queries TXT at `_dmarc.<name>` and highlights the `v=DMARC1`
  record.

**Reverse lookup** - pick PTR, or just enter an IP with any type (a bare IP,
or a `.in-addr.arpa` / `.ip6.arpa` name, is turned into a PTR query
automatically).

Output shows the resolver that answered, the response code, query time, and an
answer table (`name / type / ttl / data`).

## Web probe

Sends a `HEAD` from a Globalping probe to `host:port` for each port you list
and reports the HTTP status, `Server` / redirect headers, and a one-line cert
summary (CN + expiry). Full certificate detail lives in the TLS card. Public
hosts only. `scheme` does **HTTPS**, plain **HTTP**, or **HTTP/2**.

Common web-UI ports: **443**, **8443** (UniFi, pfSense), **8006** (Proxmox),
**5001** (Synology DSM), **10000** (Webmin), **9090** (Cockpit), **2087**
(WHM), 4443 / 7443 / 9443 (varies).

**Scan a range** (the fold-out) sends the same `HEAD` to every address in a
`/28`-`/32` CIDR or `a.b.c.d-e` range (16 max) and shows a grid: each port
marked up (HTTP code) or down, each host **live** or **offline**. Public
ranges only; it stops itself on a rate-limit.

## TLS certificate

Switch at the top of the card - **Local file** (default) or **Public host**.

**Local file** - pick or drag in a `.pem` / `.crt` / `.cer` / `.der`, or paste
one or more `-----BEGIN CERTIFICATE-----` blocks. An X.509 parser in the page
(~180 lines, no dependency) decodes it and shows version, subject and issuer
DN, serial, `not before` / `not after` and days remaining, SANs, public key
(RSA / EC / Ed25519 + bits), signature algorithm, basic constraints, key usage
and extended key usage, subject / authority key IDs, the SHA-1 and SHA-256
fingerprints, and the SPKI pin (`sha256/<base64>`, HPKP-style). A bundle /
`fullchain` renders every certificate in it. Nothing is uploaded, nothing is
validated, and it **works offline** - the way to inspect a cert you exported
from a LAN device. Fingerprint and SPKI pin were checked byte-for-byte against
`openssl`.

**Public host** - fetches the live certificate from an internet host:port via a
Globalping probe (handy for a quick check after rotating a cert on a public
site). Same layout, minus SHA-1 and the SPKI pin, which the probe doesn't
return.

### What neither card can do

- **Read a live LAN device's certificate.** Page JavaScript has no API for a
  connection's certificate, `fetch` to a private IP is blocked by Private
  Network Access, and Globalping rejects RFC1918 addresses. Export the cert
  from the device (`openssl s_client -connect host:443 | openssl x509`, or the
  device's own UI) and use **Local file**.
- **Non-web services** - SSH, SMTP, IMAP, RDP, SMB, VNC, database ports, SNMP,
  etc. Neither a browser (the Fetch "bad ports" blocklist, no raw sockets) nor
  Globalping (only ping / traceroute / mtr / dns / http measurements) can open
  a raw TCP socket to grab a banner. HTTP(S) on any port, DNS, ICMP, and
  traceroute are the whole menu - which is what these five tools cover.

## Run it

No build step, no dependencies - just open `index.html`.

```sh
git clone https://github.com/Vybenclave/browser-netstats
xdg-open browser-netstats/index.html
```

To serve it locally instead:

```sh
python -m http.server -d browser-netstats 8080
```

Ping and the TLS card's **Local file** mode work from `file://` with no
internet. Traceroute, DNS, the web probe, and the TLS card's **Public host**
mode need network access to `api.globalping.io`.

A status chip under the header checks `api.globalping.io` on load and every
5 minutes (and immediately on the browser's `online`/`offline` events). When it
can't be reached - offline, opened from disk with no internet, a captive portal
- the probe-backed cards grey out with a note, while ping and the local cert
parser stay fully usable.

## Configuration

Constants at the top of the `<script>` block in `index.html`:

```js
const INTERVAL   = 500;    // ms between pings
const TIMEOUT    = 2000;   // ms before a ping counts as lost
const MAXBARS    = 120;    // bars on screen (~1 min)
const CSV_WINDOW = 60000;  // ms of ping history kept for export
const AVG_WIN    = 8;      // samples in the rolling-average line
```

Edit the `TARGETS` map for ping presets. The DNS resolver presets are just
`<option value>` IPs in the markup, and `DNS_VIRTUAL` maps the SPF / DMARC
shortcuts onto TXT.

## License

MIT - see [LICENSE](LICENSE).
