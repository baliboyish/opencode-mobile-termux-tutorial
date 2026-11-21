# OpenCode on Your Phone — The Complete Termux + OpenCode Mobile Tutorial (For Dummies)

**Control your OpenCode coding sessions from your phone with a clean, elegant app — while the server runs on the phone itself (Termux), or on your laptop/desktop.**

This tutorial was written from a real setup that was built and tested end to end — including an actual phone-reboot test. Follow the chapters in order and by the end, the **OpenCode Mobile** app on your phone will be connected to an OpenCode server, and you'll be able to watch, resume, and steer coding sessions from anywhere. The couch counts.

> **Too dumb to read?** Share this GitHub link to your favorite AI agent and ask it to guide you 😬

> **Versions used in this tutorial**
> - App: **OpenCode Mobile v0.17.0** (unofficial client by beppe2880) — API spec **2.0.27**
> - Phone server: **OpenCode 2.0.27** inside proot Ubuntu (Termux)
> - Bonus desktop server: OpenCode Desktop **2.3.2** (Chapter 8)
>
> These version numbers matter. Half of this tutorial is about *matching versions* — read Chapter 2 before jumping anywhere.

*🇮🇩 Versi Bahasa Indonesia: [README.id.md](README.id.md)*

---

## Table of Contents

- [Chapter 0 — The big picture](#chapter-0--the-big-picture)
- [Chapter 1 — What you need](#chapter-1--what-you-need)
- [Chapter 2 — Why the app says "Not found on the server"](#chapter-2--why-the-app-says-not-found-on-the-server)
- [Chapter 3 — The easy way: let an agent build it for you](#chapter-3--the-easy-way-let-an-agent-build-it-for-you)
- [Chapter 4 — The manual way: proot Ubuntu + OpenCode 2.0.27](#chapter-4--the-manual-way-proot-ubuntu--opencode-2027)
- [Chapter 5 — Pairing the app](#chapter-5--pairing-the-app)
- [Chapter 6 — Making it survive: auto-start, battery, start/stop](#chapter-6--making-it-survive-auto-start-battery-startstop)
- [Chapter 7 — Troubleshooting](#chapter-7--troubleshooting)
- [Chapter 8 — Bonus: a second server from your laptop/desktop via Tailscale](#chapter-8--bonus-a-second-server-from-your-laptopdesktop-via-tailscale)
- [Chapter 9 — Security (do not skip)](#chapter-9--security-do-not-skip)
- [Closing](#closing)

---

## Chapter 0 — The big picture

The OpenCode Mobile app is just a **remote control**. It needs an engine: an OpenCode server running somewhere. This tutorial installs that engine in two places:

```
┌──────────────────────────────────────────────────────────────┐
│ Your phone                                                   │
│                                                              │
│  OpenCode Mobile app ──► 127.0.0.1:4096                      │
│                              │                               │
│                     ┌────────▼─────────────┐                 │
│                     │ proot Ubuntu (Termux)│                 │
│                     │ OpenCode 2.0.27      │                 │
│                     └──────────────────────┘                 │
│                                                              │
│  (Bonus, Chapter 8) the same app can also point to:          │
│  Laptop/Desktop ──► http://<laptop-tailscale-ip>:4096        │
│                     OpenCode Desktop 2.3.2                   │
└──────────────────────────────────────────────────────────────┘
```

Why does the server need proot Ubuntu instead of plain Termux OpenCode? That is the core problem — see Chapter 2. But first, gather your ingredients.

---

## Chapter 1 — What you need

1. **An Android phone** (this tutorial was tested on Android 14, no root).
2. **Termux** — install from [F-Droid](https://f-droid.org/packages/com.termux/) or [GitHub Releases](https://github.com/termux/termux-app/releases).
   ⚠️ **Important:** every Termux add-on (Termux:Boot, Termux:API, etc.) must come from the **same source** as your Termux, because their digital signatures must match. Termux from F-Droid → add-ons from F-Droid. If Android refuses to install an add-on with a *signature/conflict* error, this is why.
3. **OpenCode Mobile** — download from the official repo:
   - Repo: https://github.com/beppe2880/opencode-mobile
   - v0.17.0 release: https://github.com/beppe2880/opencode-mobile/releases/tag/v0.17.0
4. **OpenCode inside Termux** (the standard install, e.g. the community guysoft build) — this is the fuel for Chapter 3. If you already have OpenCode running in Termux, you already hold the main ingredient of this tutorial.
5. **Termux:Boot** — for Chapter 6, from the same source as your Termux (e.g. [F-Droid](https://f-droid.org/packages/com.termux.boot/)).
6. Optional but highly recommended: **Tailscale** on your phone and laptop (for Chapter 8).

---

## Chapter 2 — Why the app says "Not found on the server"

This is the first trap almost everyone hits, and the cause is **not** a wrong password.

OpenCode Mobile v0.17.0 checks the server's identity through this endpoint:

```
GET /api/info
```

That endpoint belongs to API spec **2.0.27**. The problem: the common native Termux build of OpenCode (the guysoft build) is version **2.0.0** — it only has the old `/api/server` endpoint, and `/api/info` **does not exist on it (404)**. The app translates that 404 into this message:

> **"Not found on the server"**

Check your old server yourself:

```bash
opencode service status
# → http://127.0.0.1:28570  (version 2.0.0-android-termux.2)

curl -s http://127.0.0.1:28570/api/server   # → 200, the server is alive
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:28570/api/info
# → 404  ← this is the culprit
```

The hard conclusions:

- **Downgrading the app doesn't help** — every app release uses `/api/info`.
- **Upgrading the native Termux build isn't available** — the newest native build is still 2.0.0.
- **The spec must match.** App v0.17.0 = 118 API paths. The right server must provide those 118 paths (exactly, or a superset like in Chapter 8).

So the fix: run **OpenCode 2.0.27** — and since there is no native build of it, we run it inside **proot Ubuntu** in Termux. Sounds like a lot, and it kind of is — which is why Chapter 3 exists.

---

## Chapter 3 — The easy way: let an agent build it for you

Already have OpenCode running in Termux? Then you have an *agent* that can build all of this for you. This is the method the author of this tutorial used, and it worked.

1. Open Termux and run:

   ```bash
   opencode
   ```

2. Paste the app repo link, then say exactly this:

   ```
   hi installed this app on my phone - and I want to login - provide me with correct url and password to login
   ```

3. That's it. The agent will diagnose the version mismatch, install proot Ubuntu, install OpenCode 2.0.27, migrate your session database, create a password, and hand you the correct URL + username + password to log in with in the app.

Your only job is to **supervise and read its report** — especially:
- Write down the URL, username, and password it gives you.
- Make sure the server binds to `127.0.0.1` only (see Chapter 9).
- Ask it to write a handover document, so you (and any other AI helping you) know exactly what got installed on your phone.

If you're the type who wants to understand every step — or your agent gives up halfway — the manual route is Chapter 4.

---

## Chapter 4 — The manual way: proot Ubuntu + OpenCode 2.0.27

> The short version: install Ubuntu in Termux with proot-distro → install OpenCode **2.0.27** inside it → (optional) migrate your session database from the old server → create start/stop scripts with a fixed password → run the server on `127.0.0.1:4096`.

### 4.1 Install Ubuntu in Termux

```bash
pkg update && pkg install proot-distro
proot-distro install ubuntu
```

The Ubuntu rootfs is about 1.7 GB. Be patient — this is the biggest download in the tutorial.

### 4.2 Install OpenCode 2.0.27 inside Ubuntu

Enter Ubuntu, then install with the version **pinned**:

```bash
proot-distro login ubuntu
curl -fsSL https://opencode.ai/v2/install | bash -s -- --version 2.0.27 --no-modify-path
export PATH="$HOME/.opencode/bin:$PATH"
opencode --version   # must say: 2.0.27
exit
```

⚠️ Don't just install the latest version. App v0.17.0's partner is 2.0.27. (If the app moves to a new spec later, adjust this number — see Chapter 7.)

### 4.3 (Optional) Migrate your sessions from the old server

If you have sessions on the old native server and want to bring them along, back up its database **while the old server is still running** (online-safe):

```bash
# in Termux (not inside proot)
sqlite3 ~/.local/share/opencode/opencode.db ".backup ~/.local/share/opencode/ocm-migrate.db"
```

Then place that snapshot as the new server's database inside the rootfs:

```bash
proot-distro login ubuntu -- env -i HOME=/root PATH=/usr/bin:/bin bash -lc '
  D=/root/.local/share/opencode
  mkdir -p "$D"
  rm -f "$D"/opencode.db "$D"/opencode.db-shm "$D"/opencode.db-wal
  cp /data/data/com.termux/files/home/.local/share/opencode/ocm-migrate.db "$D/opencode.db"'
```

The database will be migrated automatically by the 2.0.27 server on first run.

### 4.4 Create the start / stop / shell scripts

These three small scripts in `~/bin/` are how you'll operate your server from now on. **Replace `YOUR_PASSWORD` with a random password of your own** (a good format: 24 hex characters — generate one with `openssl rand -hex 12` in Termux).

`~/bin/oc-app-server` — starts the server:

```bash
#!/data/data/com.termux/files/usr/bin/bash
export OPENCODE_SERVER_PASSWORD="YOUR_PASSWORD"
export OPENCODE_SERVER_USERNAME="opencode"
proot-distro login ubuntu -- env \
  OPENCODE_SERVER_PASSWORD="$OPENCODE_SERVER_PASSWORD" \
  OPENCODE_SERVER_USERNAME="$OPENCODE_SERVER_USERNAME" \
  HOME=/root PATH=/root/.opencode/bin:/usr/bin:/bin \
  bash -lc 'cd /data/data/com.termux/files/home && opencode serve --port 4096 --hostname 127.0.0.1' \
  >> ~/.local/share/opencode/app-server.log 2>&1
```

`~/bin/oc-app-stop` — stops the server:

```bash
#!/data/data/com.termux/files/usr/bin/bash
pkill -f "opencode serve --port 4096" 2>/dev/null
echo "server stopped"
```

`~/bin/oc-app-shell` — enter Ubuntu with the right environment (for maintenance / `opencode auth login`):

```bash
#!/data/data/com.termux/files/usr/bin/bash
proot-distro login ubuntu -- env HOME=/root PATH=/root/.opencode/bin:/usr/bin:/bin bash "$@"
```

Make them executable, and guard the start script — your password lives in it:

```bash
chmod 700 ~/bin/oc-app-server ~/bin/oc-app-stop ~/bin/oc-app-shell
mkdir -p ~/.local/share/opencode
```

### 4.5 Start it and verify

```bash
nohup ~/bin/oc-app-server >/dev/null 2>&1 &
sleep 10

# without a password → must be 401
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:4096/api/info

# with the password → must show version 2.0.27
curl -s -u opencode:YOUR_PASSWORD http://127.0.0.1:4096/api/info
# → {"version":"2.0.27", ...}
```

If both checks pass, your server is ready to pair with the app.

---

## Chapter 5 — Pairing the app

1. Open **OpenCode Mobile**.
2. Choose to add a server manually, and fill in:
   - **URL:** `http://127.0.0.1:4096`
   - **Username:** `opencode`
   - **Password:** the password you created in Chapter 4.4
3. Save. The status must turn **Connected**, and your sessions (including migrated ones) appear in the list.

Congratulations — the remote control is now talking to the engine. Now make it last.

---

## Chapter 6 — Making it survive: auto-start, battery, start/stop

This server lives inside Termux. Which means: **if the Termux process dies, the server dies with it** — and the app cannot start it by itself. This chapter is the difference between a setup that "connected once" and a setup you can rely on daily. Every point here passed a **real phone-reboot test**.

### 6.1 The daily commands worth memorizing

```bash
# start the server (this one line is the key)
nohup ~/bin/oc-app-server >/dev/null 2>&1 &

# stop the server
~/bin/oc-app-stop

# check whether the server is alive — without opening the app
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:4096/api/info
# 401 = ALIVE (it's asking for a password — that's good) · 000 = DEAD
```

### 6.2 What kills the server (avoid these)

- **Swipe-closing Termux from recents** → the Termux process dies → the server dies.
- **Force-stopping Termux** (from Settings, or the EXIT button in Termux's notification) → same, total death.
- Closing the OpenCode TUI view (`Ctrl+C`) is **safe** — only the view dies, the server keeps running.

If the server dies from any of the above: open Termux, run the `nohup` line from 6.1, wait ±10 seconds, and the app reconnects by itself.

### 6.3 Auto-start after reboot: Termux:Boot

1. Install **Termux:Boot** from the same source as your Termux (Chapter 1), then **open the app once** — that one open is what activates it.
2. Create the hook file `~/.termux/boot/oc-app-server`:

   ```bash
   #!/data/data/com.termux/files/usr/bin/sh
   termux-wake-lock
   nohup ~/bin/oc-app-server >/dev/null 2>&1 &
   ```

3. Make it executable: `chmod +x ~/.termux/boot/oc-app-server`

### 6.4 Tame the battery saver

In your phone's Settings:

- **Apps → Termux → Battery → Unrestricted** (on OxygenOS: turn ON **Allow background activity**; leave "Allow foreground activity" on too).
- Lock Termux in recents (the padlock icon) so the system doesn't casually swipe it away.

### 6.5 The reboot test (do it once, so you trust it)

Restart your phone. After it boots, **open Termux once** — on many ROMs (OxygenOS included), the Termux process fully "wakes up" when you open the app, and the boot-hook server needs that. Wait ±1–5 minutes (a proot start right after boot is genuinely slow), then open OpenCode Mobile: it should be **Connected on its own, without you running a single command**.

The final formula is simple: **after a reboot → open Termux once → the app connects.**

---

## Chapter 7 — Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| App: *"Not found on the server"* | You're pointing at the 2.0.0 server (port 28570), or another version without `/api/info` | Make sure the URL is `:4096`; `curl .../api/info` must answer with a version that matches the app |
| App: *Cannot reach the server* / Connection refused | The 4096 server is dead | Run the `nohup` line from Chapter 6.1 |
| 401 Unauthorized | Wrong password | Check `OPENCODE_SERVER_PASSWORD` in `~/bin/oc-app-server`; match it with the app |
| No connection after a reboot | Termux:Boot missing / hook didn't run / Termux not opened yet | Chapter 6.3 + open Termux once (Chapter 6.5) |
| Server dies by itself mid-day | Killed by the battery saver / swiped away | Chapter 6.4; lock Termux in recents |
| App connects but chat errors | No provider credential yet | `~/bin/oc-app-shell` → `opencode auth login` (use your own API key), or use the built-in free `opencode` models |
| Old sessions don't show up | Database not migrated | Chapter 4.3 |
| App updated, suddenly incompatible again | The app moved to a new spec (e.g. 2.3.2) | Upgrade the server to the app's spec version: back up the DB first, then inside proot: `curl -fsSL https://opencode.ai/v2/install \| bash -s -- --version <target-version> --no-modify-path`, restart the server |
| Want to check spec compatibility | — | Compare the app repo's `api/openapi.json` (at its version tag) with your server's `/openapi.json`: every app path must exist on the server |

---

## Chapter 8 — Bonus: a second server from your laptop/desktop via Tailscale

The app supports **multiple saved servers**. If you run OpenCode Desktop on a laptop, you can add it as a second server and steer desktop work from your phone.

The principle is the same — **the spec must match** — but the news is good: OpenCode Desktop 2.3.2 provides **142 paths**, a full superset of the 118 paths app v0.17.0 needs (verified: 0 missing). So even the older app connects fine to the newer desktop.

Short version (Windows):

1. Take the CLI binary from your desktop install, e.g. `%APPDATA%\ai.opencode.desktop\cli\2.3.2-x64\opencode-cli.exe`. (Don't use an old npm CLI if you have one — the 1.x line's database is incompatible with the 2.x line.)
2. Prepare a dedicated folder with a **separate data home**, so this server's session database doesn't collide with the desktop app's database:

   ```bat
   set OPENCODE_SERVER_PASSWORD=YOUR_NEW_PASSWORD
   set OPENCODE_SERVER_USERNAME=opencode
   set XDG_DATA_HOME=C:\path\to\server-folder\data
   set XDG_CONFIG_HOME=C:\path\to\server-folder\config
   opencode-cli.exe serve --port 4096 --hostname <LAPTOP-TAILSCALE-IP>
   ```

3. Make it stick: wrap the above in a `.cmd` script with an auto-restart loop, and register it as a **Scheduled Task** that runs at logon (turn off "stop if on batteries" if your laptop is often unplugged).
4. In the app, add a new server:
   - **URL:** `http://<LAPTOP-TAILSCALE-IP>:4096`
   - **Username:** `opencode`
   - **Password:** your new password from step 2

Why bind to the Tailscale IP instead of `0.0.0.0`? Because then your server is reachable only through your own Tailscale network (encrypted, from anywhere) — not exposed to your local Wi-Fi, let alone the internet. The database separation has one consequence: your old desktop-app sessions won't show up on the phone, but new sessions you start from the phone **really run on the laptop**.

---

## Chapter 9 — Security (do not skip)

- **The phone server binds to `127.0.0.1` only.** Never change it to `0.0.0.0`, and never forward port 4096 to the internet. Anyone who can reach that port faces exactly one password.
- For cross-device access, **use Tailscale/WireGuard** (like Chapter 8), not port exposure.
- The server password is stored in plaintext in the start script (`chmod 700`). Treat that file — and **any screenshot that shows your password** — as a secret. Don't upload a handover document that still contains your real password anywhere public. (Yes, this line is written from experience.)
- To change the password: edit `OPENCODE_SERVER_PASSWORD` in the script, restart the server, and update the app too.
- proot is a compatibility layer, **not** a security sandbox.

---

## Closing

Recap of your journey, if you made it here with the app showing **Connected**:

1. You know why "Not found on the server" appears (spec mismatch, not a wrong password).
2. You have an OpenCode 2.0.27 server in proot Ubuntu, with your old sessions migrated.
3. You can start, stop, and check it with three short commands.
4. It auto-starts after a reboot (just open Termux once), and the battery saver no longer kills it.
5. Bonus: your laptop/desktop can be steered from the same phone, too.

**Important links**
- OpenCode Mobile (the app): https://github.com/beppe2880/opencode-mobile
- Download v0.17.0: https://github.com/beppe2880/opencode-mobile/releases/tag/v0.17.0
- Termux: https://f-droid.org/packages/com.termux/
- Termux:Boot: https://f-droid.org/packages/com.termux.boot/

*This tutorial is based on a real setup on one Android phone + one Windows laptop, built and verified step by step on 21 November 2026 — including the parts that failed along the way, so you don't have to repeat the same failures.*
