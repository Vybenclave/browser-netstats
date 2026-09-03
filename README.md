# Browser Network Toolkit

A single-page, no-build web app that measures round-trip latency and packet
loss to regional endpoints straight from the browser. Dark Tokyo Night theme,
cyan/magenta graph on a 25% grey grid.

**Live page:** https://vybenclave.github.io/browser-netstats/

## What it does

- Pings a target every **500 ms** and draws one vertical bar per ping; bar
  height is the RTT in milliseconds.
- Cyan bars are successful pings, magenta bars are lost packets, and the
  magenta line is an 8-sample rolling average.
- Live metrics: last RTT, min / avg / max, jitter, **total pings**, and
  **packet loss %**.
- Presets for the four AWS US regions, plus a box for any custom host or IP.

| Preset | Endpoint |
| ------ | -------- |
| US East - N. Virginia | `s3.us-east-1.amazonaws.com` |
| US East - Ohio | `s3.us-east-2.amazonaws.com` |
| US West - N. California | `s3.us-west-1.amazonaws.com` |
| US West - Oregon | `s3.us-west-2.amazonaws.com` |

## How the "ping" works

A browser can't send ICMP, so this times an HTTPS request instead:

```js
fetch("https://<host>/?_bnt=<now>", { mode: "no-cors", cache: "no-store" })
```

`no-cors` means the response is opaque - we never read it, we only time it -
so the target doesn't need any CORS headers. An `AbortController` cancels the
request after 2 s; an abort or a network error is recorded as a lost packet.

Caveats:

- Values include DNS, TCP, and TLS setup, so they read higher than a real
  `ping`. The first request to a host is the slowest (fresh TLS handshake);
  later ones reuse the connection.
- A custom target must be reachable over **HTTPS** on port 443.
- Corporate proxies, caching, and rate limits can distort or inflate results.

## Run it

Just open `index.html` - there's no build step and no dependencies.

```sh
git clone https://github.com/Vybenclave/browser-netstats
xdg-open browser-netstats/index.html      # or drag it into a browser
```

To serve it locally instead:

```sh
python -m http.server -d browser-netstats 8080
```

## Configuration

Edit the constants at the top of the `<script>` block in `index.html`:

```js
const INTERVAL = 500;    // ms between pings
const TIMEOUT  = 2000;   // ms before a ping counts as lost
const MAXBARS  = 240;    // history kept on screen (~2 min)
const AVG_WIN  = 8;      // samples in the rolling-average line
```

Add or change presets in the `TARGETS` array in the same block.

## License

MIT - see [LICENSE](LICENSE).
