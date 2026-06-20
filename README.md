# mt5-server

Headless Wine-based MetaTrader 5 bridge for `eth-trader-bot`.

Built on a minimal `debian:bookworm-slim` base to keep the image small (the
KasmVNC desktop variant produced a ~4–5 GB image that Zeabur couldn't pull →
`ImagePullBackOff`). This variant has **no GUI/VNC** — MT5 runs under a virtual
display (Xvfb) and logs in automatically from environment variables.

## Architecture

- **`debian:bookworm-slim` + winehq-stable** runs the Windows MT5 terminal and a
  full Windows Python 3.9.13.
- **Xvfb** provides a headless virtual display (MT5 is a GUI app and needs one),
  but there is **no VNC or web desktop**.
- **mt5linux RPyC bridge** is started from the Linux side via
  `python3 -m mt5linux ... -w wine python.exe` and exposes the MetaTrader5 API
  on port 8001. The bridge runs in the foreground and keeps the container alive.
- No server-side `mt5.initialize()` readiness loop — `initialize()` is called by
  the **client** (eth-trader-bot).

## Auto-login (required for headless)

Since there's no GUI, MT5 must log in from credentials. Set **either**:

- `MT5_CMD_OPTIONS=/login:12345 /password:yourpass /server:Broker-Demo`, **or**
- the individual vars `MT5_LOGIN`, `MT5_PASSWORD`, `MT5_SERVER` (the script builds
  the command-line options from them).

The broker named in `/server:` must be reachable from MetaQuotes' server
directory for a first-time headless login to succeed.

## Ports

| Port | Purpose |
|------|---------|
| 8001 | mt5linux RPyC bridge (the Python API endpoint) |

## Client usage

```python
from mt5linux import MetaTrader5
mt5 = MetaTrader5(host="<zeabur-host>", port=8001)
mt5.initialize()
```

## Environment Overrides

- `BRIDGE_PORT` — RPyC bridge port (default `8001`)
- `MT5_CMD_OPTIONS` — full terminal command-line flags (auto-login)
- `MT5_LOGIN` / `MT5_PASSWORD` / `MT5_SERVER` — used if `MT5_CMD_OPTIONS` is unset

## Deploy Notes (Zeabur)

- Expose **port 8001**.
- The `/config` volume persists Wine, MT5, and Python across reboots — first boot
  installs everything (a few minutes), later boots reuse it.
- If a previous broken prefix is reused, delete the service volume once and
  redeploy so `/config/.wine` is recreated cleanly.
