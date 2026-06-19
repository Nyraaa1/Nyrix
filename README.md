# Nyrix

A self-hosted **game server host**: like Aternos or Pebble, but yours. Create, run, and control every kind of Java and Bedrock Minecraft server from a single clean web dashboard, with **multi-user accounts**, **custom domains**, and a **remote node agent** so friends can pool their machines into one fleet.

Supports **Vanilla, Paper, Folia, Purpur, Fabric, Velocity, Waterfall, and Bedrock**. Pick the software and version in the browser, hit deploy, and Nyrix downloads the right server, sets it up, and gives you a live console.

---

## Features

- **Multi-user accounts**: invite-only account system (scrypt-hashed passwords, signed-cookie sessions). The first run creates an **admin**; admins invite **users**, who each see and manage only their own servers within a per-user **quota** (max servers + max RAM). Admin-only controls: users, the node fleet, and portal-wide settings (like FTP).
- **Custom domains**: add your own domain to a server, prove you own it with a one-line DNS **TXT record**, then optionally **lock the server to verified domains**: the proxy hides the server and refuses any connection that doesn't come in through your domain (direct-IP joins get a "use the domain" message).
- **Remote node agent**: a second program (`npm run agent`) that turns any Linux box into a worker in your fleet. It **auto-installs Docker + Java 21/25**, pairs to the portal with a connect token over an **outbound** connection (works behind home NAT, no port-forwarding on the worker), and reports its specs to the portal's live **node fleet** view. Let a few friends run it and you've got a little hosting network.
- **Every server type**: Java game servers, proxies, and native Bedrock, each downloaded from its official source.
- **One-click deploy**: choose software + version + RAM + port; Nyrix fetches and configures it.
- **Isolated & resource-limited**: each server runs in its own Docker container with hard memory and CPU caps, the matching Java version baked in, and its own network namespace, effectively a mini-VPS per server. [Aikar's GC flags](https://docs.papermc.io/paper/aikars-flags) are applied automatically for fast, low-pause performance.
- **Live console**: real-time output over Server-Sent Events, with a command input box.
- **Start / stop / restart / kill**: graceful shutdown (sends the correct stop command per software) plus a one-click force-stop for stuck servers.
- **File manager**: browse and edit any text file in place, **upload files from the browser** (button or drag & drop), create folders, and delete files/folders. Binary files are detected and left alone.
- **Built-in FTP server**: enable it in Settings to manage files remotely with any FTP client (root = the data directory, one folder per server). The panel's settings file is hidden from FTP.
- **Mod manager (Modrinth)**: search and one-click install **mods, plugins, and datapacks** per server. Results and versions are auto-filtered to the server's loader + game version (Fabric → mods, Paper/Purpur → plugins, Folia → Folia-supported plugins, Velocity/Waterfall → proxy plugins, Vanilla → datapacks), client-only mods are rejected, and required dependencies (like Fabric API) install automatically.
- **Built-in proxy (per server)**: keep players on one public port (e.g. `25565`) while the real server runs on a private internal port. The proxy is Minecraft-protocol-aware and serves:
  - a **custom offline MOTD** on the server list and a **custom offline join message** when the server is stopped or restarting,
  - **UDP forwarding** so Simple Voice Chat keeps working through the proxy,
  - **real player-IP forwarding** (optional, **PROXY protocol v2**) so the backend sees each player's real IP instead of `127.0.0.1`, which is what server-side IP bans, geo and anti-cheat need,
  - **server-list status caching**: pings are answered from a ~1.5s snapshot of the server's real status, so even a heavy ping flood never opens a single connection to the game server,
  - **VPN / proxy / datacenter IP blocking** (optional, via ip-api.com, cached + fail-open),
  - a **bot shield** that leaves real players alone: only sustained abuse (connection/ping floods, malformed packets, in-play packet spam) gets pushed into a slow, throttled lane, and it only ever slows the one bad connection, never a whole shared/CGNAT IP,
  - an **anti-DDoS connection shield**: per-IP concurrency + connection-rate + ping-rate limits, handshake timeouts, packet sanity checks, and temporary auto-bans, plus manual allow/block lists, with **live stats** (players now, peak, bandwidth, cache hits, blocks/bans) in the UI. (Application-layer only: it drops junk before it reaches your server, but can't absorb a volumetric uplink flood, see the security note.)
- **Persistent**: servers survive panel restarts; metadata is stored in `servers.json`, each server in its own folder.
- **Light footprint**: `express`, `adm-zip`, `ftp-srv`. No database, no socket.io.

---

## Requirements

- **Node.js 18+** (uses the built-in `fetch`)
- **Docker** (recommended), Nyrix runs each server in its own container, so you do **not** need Java installed on the host; the right JRE comes from the image. Install via [get.docker.com](https://get.docker.com), and make sure the user running Nyrix can talk to Docker (`sudo usermod -aG docker $USER`, then re-login).
- A **Linux** host (Bedrock ships a Linux-only native binary).

If Docker isn't present, Nyrix automatically falls back to **process mode**: running servers as plain child processes with Aikar's flags but no container isolation or hard limits. In that mode you *do* need Java on the host (e.g. `apt install openjdk-21-jre-headless`). The active engine is printed on startup and shown in the dashboard header.

---

## Install & run

```bash
# from the project folder
npm install
npm start
```

Then open `http://YOUR_SERVER_IP:25765`.

### Options

Configure via flags or environment variables (flags win):

```bash
node src/index.js --port 9000 --data /srv/minecraft
# or
NYRIX_PORT=9000 NYRIX_DATA_DIR=/srv/minecraft npm start
```

| Setting | Flag | Env | Default |
| --- | --- | --- | --- |
| Web UI port | `--port` | `NYRIX_PORT` | `25765` |
| Data directory | `--data` | `NYRIX_DATA_DIR` | `./servers` |

> The legacy `MC_PORT` / `MC_DATA_DIR` variables are still honored as a fallback.

### Run it as a global command (optional)

```bash
npm install -g .
nyrix --port 25765 --data /srv/minecraft
```

### Keep it running (systemd)

```ini
# /etc/systemd/system/nyrix.service
[Unit]
Description=Nyrix game server panel
After=network.target

[Service]
ExecStart=/usr/bin/node /opt/nyrix/src/index.js --data /srv/minecraft
WorkingDirectory=/opt/nyrix
Restart=on-failure
User=minecraft

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now nyrix
```

---

## Security note

Nyrix now has **account-based authentication**: every `/api` route requires a signed-cookie session, and the dashboard is gated behind a login. Passwords are scrypt-hashed; account creation is invite-only.

**Login hardening (built in):**

- **Two-factor auth (TOTP), mandatory.** Every account must set up 2FA: right after first login the panel is locked to a full-screen enrollment gate until you add Nyrix to an authenticator app (Google Authenticator, Aegis, 1Password…). Enrollment hands you **10 single-use backup codes**, and login then requires the 6-digit code (or a backup code). 2FA can't be turned off, only re-enrolled to a new device via account menu → *Security & 2FA* → *Change authenticator*. (To make it optional instead, flip `FORCE_2FA` in `src/index.js`.)
- **Verified email, mandatory.** Once email is configured (Settings → Email), every account must add and verify an email address before using the panel, a full-screen gate after the 2FA step, same pattern. Invited users are auto-verified (they accepted via an emailed link). The gate is **fail-open**: if email isn't configured server-side it's skipped, so a mail misconfiguration can't lock everyone out. (Flag: `REQUIRE_EMAIL_VERIFIED` in `src/index.js`.)
- **Brute-force throttling.** Failed logins are rate-limited per-IP **and** per-account with an escalating lockout, plus a coarse per-IP cap on the auth endpoints, so password guessing is slow and a login flood can't pin the server.
- **Non-blocking password hashing.** scrypt runs on the worker-thread pool with a concurrency cap, so even a flood of simultaneous logins (e.g. a distributed attack) can't freeze the event loop or exhaust memory.
- **Revocable sessions.** Tokens carry a per-account epoch: changing your password, enabling 2FA, or **Sign out everywhere** instantly invalidates every existing session.
- The session cookie is `HttpOnly`, `SameSite=Lax`, and gains the **`Secure`** flag automatically when the portal is served over HTTPS (directly or via `X-Forwarded-Proto`).
- **Behind a reverse proxy / tunnel** (Cloudflare, Caddy, nginx…), set **`NYRIX_TRUST_PROXY=1`** so per-IP rate limits and login throttling see each visitor's real IP (from `CF-Connecting-IP` / `X-Forwarded-For`) instead of lumping everyone under the proxy's `127.0.0.1`. Leave it **off** when the panel is directly exposed, so the header can't be spoofed.

When exposing it to the internet, it must be reached over **HTTPS**. You have two options:

**A. TLS terminated upstream (recommended), Cloudflare Tunnel, Caddy, or nginx.** The panel stays plain HTTP on localhost; the proxy/tunnel does HTTPS. This is correct and expected, *the panel does not need to be HTTPS itself.* The flow is `visitor -HTTPS→ proxy/edge -HTTP→ panel (localhost)`. Set `NYRIX_TRUST_PROXY=1` (see above) and the panel detects HTTPS from `X-Forwarded-Proto` / Cloudflare's `CF-Visitor` header and sets the `Secure` cookie flag. A Cloudflare Tunnel (see [setup-tunnel.py](setup-tunnel.py)) gives free HTTPS with no port-forwarding; or a quick Caddy config:

```caddyfile
panel.example.com {
    reverse_proxy localhost:25765
}
```

**B. Native HTTPS (no proxy).** Point the panel at a cert + key and it serves HTTPS directly:

```bash
NYRIX_TLS_CERT=/path/fullchain.pem NYRIX_TLS_KEY=/path/privkey.pem ./start.sh
# optional: NYRIX_TLS_CA=/path/chain.pem
```

It logs `https://…` on boot and the `Secure` cookie turns on automatically. Use a real cert (Let's Encrypt) for browsers to trust it.

Also open your servers' game ports in the firewall (e.g. `25565/tcp`, `19132/udp` for Bedrock), these are separate from the panel port.

More security notes:

- **Node connect tokens** are bearer secrets, anyone with one can pair a worker and receives the agent's trust. They're shown once in the panel; treat them like passwords. Remove a node to revoke it.

- **FTP is plaintext** (credentials and data are unencrypted on the wire). Use it on your LAN, or tunnel it; for internet access prefer an SSH tunnel or VPN. The FTP user gets full read/write over every server's files.
- **The built-in proxy is L7 (application-layer) protection.** It stops connection floods, slow-loris, ping spam, malformed packets and (optionally) VPN/datacenter IPs from ever reaching your server process, and auto-bans abusive IPs. The UDP voice relay is bounded too, a per-IP cap on voice mappings plus a per-IP packet-rate guard stop one source from exhausting the relay's sockets. It **cannot** absorb a volumetric L3/L4 flood that saturates your network uplink, that requires upstream/network-level protection (a real TCPShield/NeoProtect, Cloudflare Spectrum, your host's scrubbing, etc.). With the proxy on, the server binds an internal port on localhost (process mode) or `127.0.0.1` host-side (Docker), so players can't bypass the proxy by hitting the internal port directly.

---

## How it works

```
src/
  providers.js   resolves versions + download URLs (PaperMC, Purpur, Fabric, Mojang, Bedrock APIs)
  manager.js     Server + ServerManager: spawn/stop processes, stream console, crash detection, files
  modrinth.js    mod manager: Modrinth search/install/remove with compatibility + dependency resolution
  proxy.js       built-in Minecraft proxy: offline MOTD/kick, TCP+UDP forward, rate-limit/anti-DDoS, VPN check, domain-lock
  auth.js        accounts: scrypt passwords, signed-cookie sessions, roles, quotas
  domains.js     custom domains: TXT-record ownership verification (DNS)
  nodes.js       node fleet registry: pairing tokens + agent heartbeats
  agent/         the remote node agent (a 2nd runnable program) + its Docker/Java installer
  ai.js          AI agent (currently disabled, kept dormant for future use)
  ftp.js         built-in FTP server (ftp-srv) rooted at the data directory
  settings.js    panel settings store (DATA_DIR/nyrix.json): FTP config
  index.js       Express server: REST API + SSE console + serves the dashboard
public/
  index.html     dashboard markup
  style.css      dashboard styling
  app.js         dashboard logic (talks to the REST API)
```

Each server lives in `DATA_DIR/<slug>/`, bind-mounted into its container at `/data`. Launch commands:

- Java game server → `java <Aikar's flags> -Xms=Xmx=<memory> -jar <jar> nogui`
- Proxy (Velocity/Waterfall) → light heap + G1GC, no `nogui`
- Bedrock → native `./bedrock_server` with `LD_LIBRARY_PATH=.`

In Docker mode that command runs inside `eclipse-temurin:<N>-jre` (Java auto-selected from the MC version) or `ubuntu:22.04` for Bedrock, with the container attached over stdin/stdout so the live console and command input work exactly as they would for a local process.

---

## Performance & isolation

When Docker is available, every server is its own container:

- **Hard memory cap**: the container limit is your chosen heap + ~25% headroom (so the JVM's non-heap memory can't push the box into swap or get OOM-killed). Swap is disabled inside the container for predictable tick times, and a **soft `--memory-reservation`** keeps the heap resident under host memory pressure so the GC isn't fighting page reclaim.
- **CPU cap**: set **CPU cores** on the deploy form (`0` = all cores). Maps to Docker's `--cpus`. For a busy box you can also **pin** every server to a fixed CPU set with `NYRIX_CPUSET` (e.g. `NYRIX_CPUSET=2-5`): dedicating cores cuts scheduler migration and cache misses, which shows up as steadier tick times.
- **Sandboxed**: each container drops the `NET_RAW` capability and runs with `no-new-privileges` and a `--pids-limit`, so a compromised or runaway server can't open raw sockets, escalate, or fork-bomb the host.
- **Right Java, automatically**: Nyrix picks the JRE from the Minecraft version: Java 21 for 1.20.5+, 17 for 1.17-1.20.4, 8 for older. No manual JDK juggling.
- **Aikar's flags**: the community-standard G1GC tuning is applied to game servers by default, with large-heap tweaks above 12 GB.
- **Host networking by default**: game containers run with `--network host`, so packets skip Docker's userland `docker-proxy` and NAT table. That's noticeably lower latency for a game server than published-port bridge networking. When the built-in proxy is on, the backend binds `127.0.0.1` only, so it's still reachable *only* through the proxy. Set `NYRIX_DOCKER_NETWORK=bridge` to revert to classic published-port mapping. A raised file-descriptor ceiling (`nofile=65536`) keeps it stable under many connections.
- **Isolation**: each server has its own filesystem view, process space, and (in bridge mode) network namespace. Files in `/data` stay owned by your host user (the container runs as your UID).

**Overrides (env vars):**

| Variable | Purpose |
| --- | --- |
| `NYRIX_ENGINE` | Force `docker` or `process` (default: auto-detect) |
| `NYRIX_DOCKER_NETWORK` | `host` (default, lowest latency) or `bridge` (classic published ports) |
| `NYRIX_BEDROCK_IMAGE` | Base image for Bedrock servers (default `ubuntu:22.04`) |

> The legacy `COBALT_ENGINE` / `COBALT_BEDROCK_IMAGE` variables are still honored as a fallback.

> **Keep the panel running.** Containers are attached to Nyrix over stdin for the console, so if the Nyrix process stops, its running servers stop with it. Run it under systemd (above) so it stays up and restarts on failure.

---

## Deploying as a host (accounts, domains, nodes)

### Accounts

On first launch the panel asks you to **create an admin account** (and adopts any servers that already existed). After that, every `/api/*` call requires a session, and the dashboard is gated behind a login screen.

- **Admin**: manages users, the node fleet, and portal-wide settings (like FTP), and can see/manage every server.
- **User** (invite-only, admins create them from the **Users** dialog), sees only their own servers, bounded by a quota (max servers + max RAM).

Sessions are scrypt-hashed and carried in a signed, httpOnly cookie. **Run the portal behind HTTPS** (a reverse proxy like Caddy/nginx) when exposing it to the internet, see the Security note.

### Email & verification

Hook the panel up to your own domain so it can send transactional email. Set it up in **Settings → Email**; once configured it powers four flows:

- **Email verification**: users confirm their address from *Security & 2FA → Email*.
- **Password reset**: a *Forgot password?* link on the sign-in screen emails an expiring, single-use reset link.
- **Email invites**: in the **Users** dialog, *Invite by email* creates a pending account and emails an accept-link; the invitee sets their own password (and is then walked through mandatory 2FA).
- **Security alerts**: an email on a new sign-in or a password change.

**What it takes:**

1. **An SMTP sender.** Any provider works (it's plain SMTP). Recommended: **[Resend](https://resend.com)**: free tier, simplest domain setup. Or Postmark, Amazon SES, Mailgun, or your domain's own mailbox. In Settings → Email enter the host (e.g. `smtp.resend.com`), port `587`, your username + API key, and a from address like `noreply@yourdomain.com`. Set **Panel URL** to your public address (e.g. `https://panel.yourdomain.com`), that's what email links point at. Hit **Save**, then **Send test**.
2. **DNS records on your domain** (this is what keeps mail out of spam). Your provider gives you exact values:
   - **SPF**: a TXT record authorizing the sender (e.g. `v=spf1 include:_spf.resend.com ~all`).
   - **DKIM**: a CNAME/TXT record (the provider generates the key) that cryptographically signs your mail.
   - **DMARC**: a TXT record on `_dmarc.yourdomain.com`, e.g. `v=DMARC1; p=none; rua=mailto:you@yourdomain.com`.
   You only need **MX** records if you also want to *receive* mail at the domain, not required for sending.
3. Nothing else, the panel uses Nodemailer over SMTP; tokens for verify/reset/invite are HMAC-signed, expiring, and single-use (a reset link dies the moment it's used or the password changes).

### Custom domains

In a server's **Proxy** tab, add a domain (e.g. `play.you.com`). Nyrix shows a TXT record to create:

```
TXT  _nyrix-verify.play.you.com  →  nyrix-verify=<token>
```

Add it at your DNS host, point an `A` record (or a `_minecraft._tcp` SRV record) at the server's IP/port, and click **Verify**. Once verified you can tick **Lock to verified domains**: the proxy then refuses (and hides the server from) any connection that doesn't use a verified domain. Requires the built-in proxy to be on.

### Node fleet (remote workers)

The portal is the control plane; **nodes** are machines that run servers. The machine the portal runs on is always the built-in **local** node. To add a friend's machine:

1. In the panel: **Nodes → Add a node**, give it a name, and copy the printed connect command.
2. On the worker machine (Linux), clone this repo and:

   ```bash
   npm install
   npm run agent:install            # installs Docker + Java 21 & 25 (idempotent; needs sudo)
   node src/agent/agent.js connect --portal https://your-panel.com --token <connect-token>
   ```

The agent pairs over an **outbound** connection and pushes heartbeats, so it works from behind home NAT with no inbound ports opened on the worker. It shows up **online** in the fleet with its CPU/RAM/Java/Docker specs.

**Deploy & manage servers on a node**: in the panel, click an online node → **Manage**. From there you can:

- **Deploy** a new server onto it (name, type, version, memory), it downloads and sets up on the node.
- **Start / stop / restart / delete** its servers and watch a **live console** + send commands.

It's all NAT-friendly: the portal *queues* the action and the node's agent **pulls it on its heartbeat**, runs it with the same hosting core as the portal, and reports back, so the portal never needs to reach into the worker's network. Commands land within ~1.5s while you're actively managing, and the fleet view mirrors each node's server list every few seconds.

---

## Built-in proxy

Open a Java game server, go to the **Proxy** tab, and toggle it on. Nyrix assigns a random internal port (30000-39999), rewrites the server's `server.properties` to bind it (on localhost), and stands the proxy up on your chosen **public port**: so players keep connecting on the same address (e.g. `:25565`) no matter what the real server is doing.

**Restart the server after enabling/disabling** so it picks up the new bind port. If you enable the proxy while the server is still running on the public port, the proxy can't grab that port yet, the tab tells you to restart; once you do, it binds automatically.

What it does, per connection:

| Situation | Behavior |
| --- | --- |
| Server **online** | Transparently forwards TCP (game) and UDP (voice) to the internal port, with TCP_NODELAY on both hops for the lowest latency. |
| Server **offline** | Serves your custom offline MOTD on the server list, and your custom kick message to anyone who tries to join. |
| **Server-list ping** | Answered from a ~1.5s cached snapshot of the server's real status (when caching is on), so ping floods never touch the game server. |
| Connecting IP is a **VPN/proxy** (optional) | Refused at login with your VPN message (datacenter/hosting IPs too, if enabled). |
| Connection **flood / junk** | Dropped before reaching the server; repeat offenders are temp-banned. Live counters are in the tab. |
| **Sustained** in-play packet spam | The one offending connection is throttled; only a repeat offender on that connection earns an IP penalty. A real player's brief burst is never punished. |

Colour codes use `&` (e.g. `&c`, `&a`) and `\n` makes a second MOTD line.

**Real player IPs (PROXY protocol v2).** By default the backend sees every player as `127.0.0.1` (the proxy). Tick **Forward real player IPs** in the Proxy tab and Nyrix prepends a HAProxy PROXY-protocol v2 header to the backend stream carrying each player's true IP, so server-side IP bans, geo-IP and anti-cheat work. **You must enable the matching setting on the server first**, or it will reject every connection: Paper/Purpur → `settings.proxy-protocol: true` in `paper-global.yml`; Fabric/Forge → a PROXY-protocol mod. Then turn the toggle on.

**Voice chat (Simple Voice Chat):** handled automatically. The proxy forwards UDP on the public port, and when the proxy is enabled Nyrix sets `voice_host=:<publicPort>` in `config/voicechat/voicechat-server.properties` so voice clients connect to the public port (which the proxy forwards to the server's internal voice port). Leave voicechat's `port` at the default `-1`. **Restart the server** after enabling the proxy so voicechat reloads the setting. (If you'd set `voice_host` manually before, Nyrix now manages it for you.)

---

## Caveats

- **Java versions**: in Docker mode the correct JRE is chosen for you, per server. In process mode Nyrix uses whatever `java` is on `PATH`, so a mixed-version setup needs the right host JDK (or just use Docker).
- **Bedrock libraries**: the Bedrock binary occasionally needs shared libs not present in a bare `ubuntu:22.04`. If it won't boot, set `NYRIX_BEDROCK_IMAGE` to an image that includes `libcurl4`/`libssl`.
- **Proxy port presetting**: Velocity gets a `velocity.toml` written with your chosen bind port. Waterfall generates its own `config.yml` on first run (default `25577`); change it afterward via the **Files** tab.
- **Bedrock download**: pulled from Mojang's official link API. If Mojang changes that endpoint, update `bedrockDownload()` in `providers.js`.
- **EULA**: required for Java game servers; the deploy form won't let you create one without accepting it.

---

## License

MIT, do whatever you want with it.
