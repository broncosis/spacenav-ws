# Websockets exposer for the spacenav driver (spacenav‑ws)

![PyPI version](https://img.shields.io/pypi/v/spacenav-ws)
![Build Status](https://github.com/rmstorm/spacenav-ws/workflows/Test/badge.svg)
![License](https://img.shields.io/github/license/rmstorm/spacenav-ws)

## Table of Contents

- [About](#about)  
- [Prerequisites](#prerequisites)  
- [Usage](#usage)  
- [xDesign (Dassault 3DEXPERIENCE) support](#xdesign-dassault-3dexperience-support)  
- [Running as a systemd service](#running-as-a-systemd-service)  
- [Development](#development)  

## About

**spacenav‑ws** is a tiny Python CLI that exposes your 3Dconnexion SpaceMouse over a secure WebSocket, so Onshape on Linux can finally consume it. Under the hood it reverse‑engineers the same traffic Onshape’s Windows client uses and proxies it into your browser.

This lets you use [FreeSpacenav/spacenavd](https://github.com/FreeSpacenav/spacenavd) on Linux with Onshape, and with SOLIDWORKS xDesign on Dassault's 3DEXPERIENCE platform (see [below](#xdesign-dassault-3dexperience-support)).

## Prerequisites

- [uv/uvx](https://docs.astral.sh/uv/getting-started/installation/) or another Python env manager.
- A running instance of [spacenavd](https://github.com/FreeSpacenav/spacenavd)  
- A modern browser (Chrome/Firefox) with a userscript manager (Tampermonkey/Greasemonkey)  

## Usage

1. **Validate spacenavd**
```bash
uvx spacenav-ws@latest read-mouse
# → should print spacemouse events
```

2. **Run the server and trust the cert**
```bash
uvx spacenav-ws@latest serve
```
Now open: [https://127.51.68.120:8181](https://127.51.68.120:8181). When prompted, add a browser exception for the self‑signed cert.

3. **Install Tampermonkey and add the userscript**

Install [Tampermonkey](https://addons.mozilla.org/en-US/firefox/addon/tampermonkey/?utm_source=addons.mozilla.org&utm_medium=referral&utm_content=search). After installing, click this [link](https://greasyfork.org/en/scripts/533516-onshape-3d-mouse-on-linux-in-page-patch) for one‑click install of the script.

4. **Open an Onshape document and test your mouse!**

## xDesign (Dassault 3DEXPERIENCE) support

This fork adds support for **SOLIDWORKS xDesign** running on Dassault's 3DEXPERIENCE platform (`*.3dexperience.3ds.com`), in addition to Onshape. This required two changes on top of upstream:

- The CORS allowlist in `src/spacenav_ws/main.py` now also matches any `*.3dexperience.3ds.com` origin via `allow_origin_regex`, since xDesign is served from a different subdomain per tenant rather than one fixed origin.
- `Controller.start_mouse_event_stream` in `src/spacenav_ws/controller.py` now recognizes the `"SOLIDWORKS xDesign"` client name, so motion events get forwarded instead of being silently dropped as "Unknown client".

The userscript at `additional/3d-mouse-linux.user.js` covers both Onshape and xDesign with two `@match` patterns, so you only need to install one script:

```javascript
// ==UserScript==
// @name         3D‑Mouse on Linux (in‑page patch) — Onshape & xDesign
// @description  Fake the platform property on 'navigator' to convince Onshape/xDesign it's running under Windows. This causes it to ask for information on https://127.51.68.120:8181/3dconnexion/nlproxy so that a 3d mouse can be connected.
// @match        https://cad.onshape.com/documents/*
// @match        https://*.3dexperience.3ds.com/*
// @run-at       document-start
// @grant        none
// @version      0.0.2
// @license      MIT
// @namespace    https://greasyfork.org/users/1460506
// ==/UserScript==

Object.defineProperty(Navigator.prototype, 'platform', { get: () => 'Win32' });
console.log('[3D-Mouse patch] navigator.platform →', navigator.platform);
```

Install [Tampermonkey](https://www.tampermonkey.net/), create a new script, and paste the above in. This has been merged upstream as [RmStorm/spacenav-ws#7](https://github.com/RmStorm/spacenav-ws/pull/7) — once that lands, this fork and its README section are no longer necessary, but are kept here in the meantime.

## Running as a systemd service

Rather than running `uv run spacenav-ws serve` manually in a terminal every time, you can run it as a persistent `systemd --user` service that starts automatically on login/boot and restarts on crash.

1. **Clone this repo somewhere permanent**, e.g. `~/spacenav-ws`:
```bash
git clone https://github.com/broncosis/spacenav-ws.git ~/spacenav-ws
```

2. **Create the unit file** at `~/.config/systemd/user/spacenav-ws.service`:
```ini
[Unit]
Description=SpaceNav WebSocket Bridge (3D mouse driver proxy)
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=%h/spacenav-ws
ExecStart=%h/.local/bin/uv run spacenav-ws serve
Restart=always
RestartSec=5
TimeoutStopSec=30
TimeoutStartSec=30
KillMode=control-group
Environment=HOME=%h
Environment=PATH=/usr/bin:%h/.local/bin:/usr/local/bin:/bin

[Install]
WantedBy=default.target
```
Adjust the `uv` path in `ExecStart`/`PATH` if `which uv` points elsewhere for you.

3. **Enable and start it**:
```bash
systemctl --user daemon-reload
systemctl --user enable --now spacenav-ws.service
```

4. **(Optional) Enable lingering** so the service starts at boot even before you log in interactively:
```bash
loginctl enable-linger "$USER"
```

5. **Useful commands**:
```bash
systemctl --user status spacenav-ws    # check it's running
journalctl --user -u spacenav-ws -f    # follow live logs
systemctl --user restart spacenav-ws   # after pulling updates
```

## Developing

```bash
git clone https://github.com/you/spacenav-ws.git
cd spacenav-ws
uv run spacenav-ws serve --hot-reload
```

This starts the server with Uvicorn's `code watching / hot reload` feature enabled. When making changes the server restarts and any websocket state is nuked, however, Onshape should immediately reconnect automatically! This makes for a very smooth and fast iteration workflow.
