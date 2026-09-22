# Chrome relay sidecar

The optional Chrome relay lets the `agent_browser` tool drive the user's real,
headed Chrome — with all of its signed-in sessions, extensions, and cookies —
instead of a Playwright-launched browser. No `--remote-debugging-port` is
involved (Chrome 136+ refuses that flag on the default profile anyway); the
browser side is a small MV3 extension that proxies `chrome.debugger` over one
websocket to a local relay server, which impersonates Chrome's CDP discovery
endpoint so `connect` needs no special-casing.

The relay is vendored from [oh-my-pi](https://github.com/Can1357/oh-my-pi)
(MIT) with a Node server port and a configurable-host extension delta; see
[`chrome-relay/VENDOR.md`](../chrome-relay/VENDOR.md) for the exact deltas.

## Quick start

1. Build the relay server (once per checkout or after relay changes):

   ```sh
   npm run build
   ```

2. Start the sidecar in the foreground and generate a shared token:

   ```sh
   npx pi-agent-browser-chrome-relay start --token-gen
   # chrome relay listening on ws://127.0.0.1:9224/cdp
   ```

3. Load the extension in the real Chrome: open `chrome://extensions`, enable
   Developer mode, **Load unpacked**, and pick the directory printed by
   `npx pi-agent-browser-chrome-relay extension-path`. Chrome shows the
   "started debugging this browser" infobar while the relay drives it.

4. If you generated a token, open the extension's options page and paste it
   (plus a different host/port if you changed them). The extension reconnects
   automatically.

5. Connect from the wrapper like any CDP browser:

   ```json
   { "args": ["connect", "ws://127.0.0.1:9224/cdp"], "sessionMode": "fresh" }
   ```

`status` reports whether the server is up and the extension has completed its
handshake; `stop` terminates the recorded `start`. State lives in the OS temp
directory, so `status`/`stop` work from other shells.

## Commands

| Command | Purpose |
|---|---|
| `start [--port N] [--token SECRET \| --token-gen]` | Run the relay in the foreground (default port 9224); Ctrl-C to stop |
| `status` | `{running, pid, port, extensionConnected, extensionSeen}` as JSON |
| `stop` | SIGTERM the recorded relay and clear state |
| `token` | Print a fresh random shared token |
| `extension-path` | Print the unpacked-extension directory for `chrome://extensions` |

## Windows Chrome + WSL2

In WSL2 NAT mode (the default), Chrome on Windows cannot reach `127.0.0.1`
inside the WSL VM. Two fixes; prefer the first:

1. **Point the extension at the VM address.** In the extension options, set
   *Relay host* to the WSL IP (`ip -4 addr show eth0` inside WSL, e.g.
   `172.27.91.244`). The host field accepts a bare hostname/IP only. The IP
   changes on reboot — re-check it when the relay stops connecting.
2. **Port-forward from Windows.** In an administrator PowerShell:

   ```powershell
   netsh interface portproxy add v4tov4 listenaddress=127.0.0.1 listenport=9224 connectaddress=<WSL_IP> connectport=9224
   ```

   and keep the extension host at `127.0.0.1`. Same caveat: `<WSL_IP>` changes
   on reboot (`netsh interface portproxy show all` to inspect, `delete` to
   remove).

## Security model

- The server binds `127.0.0.1` only, and rejects websocket upgrades bearing an
  `Origin` header on `/cdp`, so a web page cannot drive the relay. `/ext` only
  accepts `chrome-extension://` origins and, when configured, requires the
  shared token as `?token=`.
- Anything that *can* reach the port can drive the logged-in browser — read
  page content, click, exfiltrate cookies through pages. Start the relay only
  while using it, and prefer `--token-gen` whenever anything else runs on the
  machine.
- Inherited from upstream: `Browser.close` is acknowledged but never forwarded
  (the user's browser never dies), `chrome://` pages are ineligible targets,
  and payloads cap at 256 MiB.
- Chrome shows its debugging infobar while the extension holds a tab, and only
  one debugger may attach per tab; the extension reports detach/replace as
  events rather than failing silently.
