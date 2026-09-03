# Browser Network Toolkit

A single-page, no-build kit of network tools that run in the browser. Dark
Tokyo Night theme, cyan/magenta ping graph on a 25% grey grid.

**Live page:** https://vybenclave.github.io/browser-netstats/

Three tools, laid out as a card grid:

| Tool | Where it runs | Exports |
| ---- | ------------- | ------- |
| **Ping** | timed in your browser (HTTPS round-trip) | CSV, last 60 s |
| **Traceroute** | on a [Globalping](https://globalping.io) probe | CSV of the trace |
| **DNS lookup** | on a Globalping probe (pick the resolver) | - |

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

**Export CSV (60 s)** downloads every ping from the last 60 seconds:
`unix_ms, iso_time, target, rtt_ms, status` (`status` = `ok` | `lost`).

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
**Export CSV** gives `hop, hostname, ip, rtt1_ms, rtt2_ms, rtt3_ms`.

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

Ping works from `file://`; traceroute and DNS need network access to
`api.globalping.io`.

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
