# Android Phone as a Backend Server

### A practical Termux guide for running APIs, databases, and background services on Android

> **Purpose:** Turn an Android phone into a small self-hosted server for
> backend applications, databases, and background services.
>
> **Example stack:** Termux + Node.js/Express + PostgreSQL +
> `termux-services` (runit) + Termux:Boot + optional Cloudflare Tunnel.

This is a reusable, project-independent guide. It uses a small
Node.js/Express API and PostgreSQL database as a worked example, but the
same approach can be adapted for Python, Go, Java, and other runtimes.

## Contents

1.  [What you are building](#1-what-you-are-building)
2.  [Choose your configuration](#2-choose-your-configuration)
3.  [Phone server vs VPS](#3-phone-server-vs-vps)
4.  [Requirements and installation
    sources](#4-requirements-and-installation-sources)
5.  [Prepare Termux](#5-prepare-termux)
6.  [Install server packages](#6-install-server-packages)
7.  [Set up PostgreSQL](#7-set-up-postgresql)
8.  [Build a sample Express + PostgreSQL
    backend](#8-build-a-sample-express--postgresql-backend)
9.  [Run services with runit](#9-run-services-with-runit)
10. [Expose an API with Cloudflare
    Tunnel](#10-expose-an-api-with-cloudflare-tunnel)
11. [Start services after Android
    reboot](#11-start-services-after-android-reboot)
12. [Android background settings](#12-android-background-settings)
13. [Remote administration over SSH](#13-remote-administration-over-ssh)
14. [Backups and recovery](#14-backups-and-recovery)
15. [Security checklist](#15-security-checklist)
16. [Useful commands](#16-useful-commands)
17. [Troubleshooting](#17-troubleshooting)
18. [Deploy another backend](#18-deploy-another-backend)
19. [Adapting this guide to your project](#19-adapting-this-guide-to-your-project)
20. [Official references](#20-official-references)

------------------------------------------------------------------------

## 1. What you are building

Termux provides a Linux-like user-space environment on Android. Backend
applications and databases run as ordinary Termux processes. **runit**,
provided by `termux-services`, supervises long-running processes.

``` text
                    Internet
                       |
                Cloudflare edge
                       |
                Cloudflare Tunnel
                       |
              127.0.0.1:3000
                       |
                 Node.js API
                       |
              127.0.0.1:5432
                       |
                  PostgreSQL
```

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Android                             Hardware, power, Wi-Fi/mobile data,
                                      OS process policies

  Termux                              Linux-like command-line environment

  Node.js / Python / Go               Runs the backend

  PostgreSQL / SQLite / Redis         Stores application data

  `termux-services` + runit           Starts, supervises, and restarts
                                      services

  Termux:Boot                         Starts the Termux service
                                      supervisor after Android boots

  Cloudflare Tunnel (optional)        Provides an HTTPS route to a local
                                      API

  SSH (optional)                      Remote administration from a laptop
  -----------------------------------------------------------------------

**Recommended network rule:** Keep PostgreSQL bound to `127.0.0.1`. Let
the backend access the database. If remote access is required, expose
the API---not the database port.

## 2. Choose your configuration

Before running commands, decide where your project, API, and database
will live. The values below are examples, not fixed requirements.

| Setting | Example | What to change |
|---|---|---|
| Project directory | `~/projects/my-api` | Use your own project name |
| API bind address | `127.0.0.1` | Keep local unless you have a specific networking need |
| API port | `3000` | Choose an unused port |
| PostgreSQL port | `5432` | Keep the default unless it conflicts |
| Database name | `app_db` | Use a name for your application |
| Database role | `app_user` | Create a dedicated, non-superuser role |
| Public hostname | `api.example.com` | Optional; use a domain you control |

Check the device architecture and installed software instead of copying
version numbers from another setup:

```bash
uname -m
getprop ro.build.version.release
node --version
npm --version
psql --version
```

To view the phone's current Wi-Fi address:

```bash
ip -4 addr show wlan0
```

The address may change when the phone reconnects. If another device on
your local network needs to reach the phone directly, use a DHCP
reservation on your router rather than relying on a temporary IP.

## 3. Phone server vs VPS

An Android phone can run APIs and databases for development, personal
projects, demos, low-traffic services, and home-lab use. It is **not
equivalent to a production VPS**.

Android may stop or restrict background processes because of battery
optimization, Doze mode, memory pressure, thermal protection, Wi-Fi
changes, battery depletion, OS updates, or unexpected reboots.

`termux-services` supervises processes while its supervisor is running.
Termux:Boot can start that supervisor after a device reboot. Neither
guarantees uninterrupted uptime under every Android condition.

**Use a VPS or managed hosting for critical services that require
contractual uptime, high availability, or reliable inbound networking.**
Treat a phone as a best-effort home server.

## 4. Requirements and installation sources

### Required

-   Android phone with sufficient storage and RAM.
-   Termux.
-   Working Wi-Fi or mobile data.
-   Stable power for long-running use.
-   Local access for initial setup.

### Optional

-   Termux:Boot --- starts services after Android reboots.
-   Cloudflare account and domain --- for an HTTPS hostname.
-   OpenSSH --- remote administration from a laptop.
-   Termux:API --- optional Android integrations; not required by this
    example.

### Keep Termux add-ons compatible

Install Termux and its add-ons from compatible signing sources. For
example, use Termux and Termux:Boot from F-Droid together, or compatible
builds from the same official source. Do not casually mix F-Droid and
GitHub APKs because signing keys can differ.

References: - [Termux project](https://termux.dev/) - [Termux on
F-Droid](https://f-droid.org/packages/com.termux/) - [Termux:Boot on
F-Droid](https://f-droid.org/packages/com.termux.boot/)

## 5. Prepare Termux

Open Termux and update packages:

``` bash
pkg update
pkg upgrade
```

Create a workspace in Termux's private home directory:

``` bash
mkdir -p ~/projects ~/backups ~/logs
echo "$HOME"
echo "$PREFIX"
```

Typical paths:

``` text
/data/data/com.termux/files/home
/data/data/com.termux/files/usr
```

### Where should projects and databases live?

Prefer Termux's private home and `$PREFIX` directories for active
applications, databases, executable files, and service scripts.

Avoid running PostgreSQL directly from shared storage such as `/sdcard`
or `/storage/emulated/0`. Shared storage has filesystem and permission
behavior that can break executable scripts, Unix permissions, symlinks,
and database expectations.

Use shared storage for transferring backups if needed, but keep the live
database in Termux's private app storage.

## 6. Install server packages

Install the tools used in this guide:

``` bash
pkg install nodejs-lts postgresql termux-services openssh git curl nano
```

If you need a different Node.js release, inspect available packages:

``` bash
pkg search nodejs
```

Cloudflare Tunnel is optional:

``` bash
pkg install cloudflared
```

Verify:

``` bash
node --version
npm --version
psql --version
git --version
```

### Start the Termux service supervisor

After installing `termux-services`, restart the Termux shell. The
simplest method is to close the Termux session and open Termux again.

Check the supervisor:

``` bash
pgrep -af runsvdir
```

Custom runit services are registered under:

``` bash
$PREFIX/var/service
```

## 7. Set up PostgreSQL

This section creates a local PostgreSQL database for the sample Express
API.

### 7.1 Initialize the database directory

Check whether a database cluster already exists:

``` bash
ls -la "$PREFIX/var/lib/postgresql"
```

For a new installation only, initialize it:

``` bash
initdb "$PREFIX/var/lib/postgresql"
```

**Do not run `initdb` over an existing database.** It initializes a new
cluster; it is not a repair or reset command.

### 7.2 Configure PostgreSQL to listen locally

``` bash
nano "$PREFIX/var/lib/postgresql/postgresql.conf"
```

Set or update these values, avoiding duplicate active settings:

``` conf
listen_addresses = '127.0.0.1'
port = 5432
max_connections = 20
shared_buffers = 64MB
password_encryption = 'scram-sha-256'
```

Binding to `127.0.0.1` prevents direct connections from the public
internet or other LAN devices.

### 7.3 Configure password authentication

Open:

``` bash
nano "$PREFIX/var/lib/postgresql/pg_hba.conf"
```

Ensure a matching local TCP rule exists:

``` conf
host    all    all    127.0.0.1/32    scram-sha-256
```

For IPv6 loopback, if used:

``` conf
host    all    all    ::1/128         scram-sha-256
```

PostgreSQL uses the first matching `pg_hba.conf` rule. Review existing
rules and adjust conflicting permissive rules; do not blindly append
rules without checking their order.

### 7.4 Start PostgreSQL for initial setup

``` bash
pg_ctl -D "$PREFIX/var/lib/postgresql" \
  -l "$PREFIX/var/lib/postgresql/logfile" start
```

Check readiness:

``` bash
pg_isready -h 127.0.0.1 -p 5432
```

Expected:

``` text
127.0.0.1:5432 - accepting connections
```

### 7.5 Create an application role and database

Open the local administrative shell:

``` bash
psql -d postgres
```

Create a dedicated role and database. Replace the example password
locally with a strong, unique password. Never paste real passwords into
chat, screenshots, Git, or documentation.

``` sql
CREATE ROLE sample_app LOGIN PASSWORD 'REPLACE_WITH_A_LONG_RANDOM_PASSWORD';
CREATE DATABASE sample_db OWNER sample_app;
\q
```

Test the application credentials over TCP:

``` bash
psql -h 127.0.0.1 -p 5432 -U sample_app -d sample_db
```

Then:

``` sql
SELECT current_database(), current_user;
\q
```

Use a separate least-privilege role for each application. Do not run
application queries as the PostgreSQL superuser.

### 7.6 Enable the PostgreSQL service

Termux packages may provide a PostgreSQL service definition. Check:

``` bash
ls -la "$PREFIX/var/service/postgres"
```

If PostgreSQL was started manually above, stop it cleanly before
enabling the supervised service. Never start a second process against
the same data directory.

``` bash
pg_ctl -D "$PREFIX/var/lib/postgresql" stop -m fast
sv-enable postgres
sv up postgres
```

Verify:

``` bash
sv status postgres
pg_isready -h 127.0.0.1 -p 5432
```

Expected service status begins with `run:`.

If PostgreSQL is already running under runit, do not run `pg_ctl start`
again. Use `sv status`, `sv restart`, and service logs to manage it.

## 8. Build a sample Express + PostgreSQL backend

This small Notes API demonstrates a real backend connected to
PostgreSQL.

### 8.1 Create the project

``` bash
mkdir -p ~/projects/sample-api
cd ~/projects/sample-api
npm init -y
npm install express pg dotenv helmet
```

Project structure:

``` text
sample-api/
├── .env
├── .gitignore
├── package.json
└── server.js
```

### 8.2 Create the environment file

``` bash
nano .env
```

Add:

``` env
NODE_ENV=production
HOST=127.0.0.1
PORT=3000

DB_HOST=127.0.0.1
DB_PORT=5432
DB_NAME=sample_db
DB_USER=sample_app
DB_PASSWORD=REPLACE_WITH_YOUR_DATABASE_PASSWORD
```

Protect it:

``` bash
chmod 600 .env
```

Never commit `.env` to Git.

### 8.3 Create `.gitignore`

``` bash
nano .gitignore
```

Add:

``` gitignore
node_modules/
.env
*.log
.DS_Store
```

### 8.4 Create the backend

``` bash
nano server.js
```

Add:

``` javascript
require("dotenv").config();

const express = require("express");
const helmet = require("helmet");
const { Pool } = require("pg");

const app = express();

app.use(helmet());
app.use(express.json({ limit: "1mb" }));

const pool = new Pool({
  host: process.env.DB_HOST || "127.0.0.1",
  port: Number(process.env.DB_PORT || 5432),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 5,
  connectionTimeoutMillis: 5000,
  idleTimeoutMillis: 10000,
});

app.get("/health", async (_req, res) => {
  try {
    await pool.query("SELECT 1");
    res.status(200).json({
      status: "ok",
      database: "connected",
      service: "sample-api",
    });
  } catch (_error) {
    res.status(503).json({
      status: "error",
      database: "unavailable",
    });
  }
});

app.get("/api/notes", async (_req, res) => {
  try {
    const result = await pool.query(
      "SELECT id, title, created_at FROM notes ORDER BY id DESC"
    );
    res.json({ success: true, data: result.rows });
  } catch (_error) {
    res.status(500).json({ success: false, error: "Could not fetch notes" });
  }
});

app.post("/api/notes", async (req, res) => {
  const title = typeof req.body.title === "string" ? req.body.title.trim() : "";

  if (!title) {
    return res.status(400).json({
      success: false,
      error: "A non-empty title is required",
    });
  }

  try {
    const result = await pool.query(
      "INSERT INTO notes (title) VALUES ($1) RETURNING id, title, created_at",
      [title]
    );
    res.status(201).json({ success: true, data: result.rows[0] });
  } catch (_error) {
    res.status(500).json({ success: false, error: "Could not create note" });
  }
});

const host = process.env.HOST || "127.0.0.1";
const port = Number(process.env.PORT || 3000);

const server = app.listen(port, host, () => {
  console.log(`Sample API listening on http://${host}:${port}`);
});

async function shutdown(signal) {
  console.log(`${signal} received; shutting down...`);

  server.close(async () => {
    await pool.end();
    process.exit(0);
  });

  setTimeout(() => process.exit(1), 10000).unref();
}

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

The example uses parameterized SQL (`$1`) instead of concatenating user
input into SQL.

### 8.5 Create the database table

Connect:

``` bash
psql -h 127.0.0.1 -p 5432 -U sample_app -d sample_db
```

Run:

``` sql
CREATE TABLE IF NOT EXISTS notes (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
\q
```

### 8.6 Test manually

Start the backend:

``` bash
cd ~/projects/sample-api
node server.js
```

In another Termux session, test health:

``` bash
curl -i http://127.0.0.1:3000/health
```

Create a note:

``` bash
curl -i -X POST http://127.0.0.1:3000/api/notes \
  -H 'Content-Type: application/json' \
  -d '{"title":"My first phone-server note"}'
```

List notes:

``` bash
curl http://127.0.0.1:3000/api/notes
```

Stop the manual server with `Ctrl+C` before creating its runit service.
Do not run the manual and supervised server on the same port
simultaneously.

## 9. Run services with runit

A command launched in an interactive terminal is tied to that session
unless deliberately detached. For long-running applications, use the
service supervisor instead of leaving `node server.js` in a terminal.

### 9.1 How runit works

A runit service is a directory containing an executable `run` script.
The application must stay in the foreground so runit can supervise it.

Termux service directories live here:

``` bash
$PREFIX/var/service
```

### 9.2 Create a service for the sample API

``` bash
mkdir -p "$PREFIX/var/service/sample-api"
nano "$PREFIX/var/service/sample-api/run"
```

Add:

``` sh
#!/data/data/com.termux/files/usr/bin/sh

cd /data/data/com.termux/files/home/projects/sample-api || exit 111

# Wait until PostgreSQL accepts local connections.
until pg_isready -h 127.0.0.1 -p 5432 >/dev/null 2>&1; do
  sleep 2
done

export NODE_ENV=production

exec /data/data/com.termux/files/usr/bin/node server.js
```

Make it executable:

``` bash
chmod 700 "$PREFIX/var/service/sample-api/run"
```

`exec` replaces the shell with Node.js, allowing runit to supervise the
application process.

### 9.3 Optional: configure persistent service logs

Create directories:

``` bash
mkdir -p "$PREFIX/var/log/sv/sample-api"
mkdir -p "$PREFIX/var/service/sample-api/log"
nano "$PREFIX/var/service/sample-api/log/run"
```

Add:

``` sh
#!/data/data/com.termux/files/usr/bin/sh

mkdir -p "$PREFIX/var/log/sv/sample-api"
exec svlogd -tt "$PREFIX/var/log/sv/sample-api"
```

Make the logger executable:

``` bash
chmod 700 "$PREFIX/var/service/sample-api/log/run"
```

### 9.4 Enable and start the API

``` bash
sv-enable sample-api
sv up sample-api
sv status sample-api
```

Test:

``` bash
curl -i http://127.0.0.1:3000/health
```

If the logger is configured:

``` bash
tail -n 100 "$PREFIX/var/log/sv/sample-api/current"
```

### 9.5 Manage services

``` bash
sv status sample-api
sv restart sample-api
sv down sample-api
sv up sample-api
sv-disable sample-api
sv-enable sample-api
```

-   `sv up` starts a service now.
-   `sv down` stops it now.
-   `sv-enable` removes the disabled marker so it can start under the
    supervisor.
-   `sv-disable` marks it disabled.
-   `sv restart` restarts it.

For PostgreSQL, use the package-provided service rather than creating a
second service pointing at the same data directory.

## 10. Expose an API with Cloudflare Tunnel

This section is optional. Use it when an app or another system must
reach the phone's API from outside the home network.

### 10.1 Install and protect the tunnel token

``` bash
pkg install cloudflared
mkdir -p ~/.cloudflared
chmod 700 ~/.cloudflared
```

Create or obtain a tunnel token in the Cloudflare dashboard and save it
on the phone, for example:

``` text
~/.cloudflared/sample-tunnel.token
```

Protect it:

``` bash
chmod 600 ~/.cloudflared/sample-tunnel.token
```

Never paste the token into a public chat, screenshot, Git repository, or
service file.

### 10.2 Configure the public hostname

In the Cloudflare Zero Trust dashboard, configure a public hostname such
as:

``` text
api.example.com  →  http://127.0.0.1:3000
```

The hostname is an example; use one you control. The service target
points to the local API port.

**Do not create a public hostname for PostgreSQL port `5432`.**

### 10.3 Create a supervised tunnel service

``` bash
mkdir -p "$PREFIX/var/service/cloudflared-sample"
nano "$PREFIX/var/service/cloudflared-sample/run"
```

Add:

``` sh
#!/data/data/com.termux/files/usr/bin/sh

exec /data/data/com.termux/files/usr/bin/cloudflared \
  --no-autoupdate \
  tunnel \
  --metrics 127.0.0.1:49312 \
  run \
  --token-file /data/data/com.termux/files/home/.cloudflared/sample-tunnel.token
```

Make executable and start:

``` bash
chmod 700 "$PREFIX/var/service/cloudflared-sample/run"
sv-enable cloudflared-sample
sv up cloudflared-sample
sv status cloudflared-sample
```

The metrics listener is bound to loopback. Keep it private.

### 10.4 Test the public route

From the phone:

``` bash
curl -i https://api.example.com/health
```

Also test from another network, such as mobile data.

A successful HTTP response proves that the hostname reaches the API. It
does **not** replace application authentication, authorization, rate
limiting, or input validation.

### Public tunnel vs private service traffic

Cloudflare Tunnel adds request headers such as `CF-Ray` and
`CF-Connecting-IP`. If the backend deliberately rejects public-tunnel
requests for internal worker or service-to-service endpoints, do not
work around that protection by weakening middleware.

Use a private path---such as a private network/WARP design or a
carefully secured SSH tunnel---for trusted service-to-service traffic.
Keep public app routes and internal automation routes separate.

## 11. Start services after Android reboot

`termux-services` supervises processes while its supervisor is running.
To start the supervisor after a phone reboot, install **Termux:Boot**.

### 11.1 Install Termux:Boot

1.  Install Termux:Boot from the same compatible source/signing family
    as Termux.
2.  Open Termux:Boot once from its launcher.
3.  Return to Termux.

Official instructions: [Termux:Boot
README](https://github.com/termux/termux-boot).

### 11.2 Create the boot script

``` bash
mkdir -p ~/.termux/boot
chmod 700 ~/.termux/boot
nano ~/.termux/boot/01-start-services
```

Add:

``` sh
#!/data/data/com.termux/files/usr/bin/sh

# Keep the CPU awake while server services are expected to run.
termux-wake-lock

# Start the Termux runit service supervisor.
. /data/data/com.termux/files/usr/etc/profile.d/start-services.sh

echo "Termux services startup requested: $(date)" >> "$HOME/boot.log"
```

Make it executable:

``` bash
chmod 700 ~/.termux/boot/01-start-services
```

Termux:Boot runs executable scripts in `~/.termux/boot/` when Android
starts. If there are multiple scripts, they are processed in sorted
order; numeric prefixes help control order.

### 11.3 Reboot test

Before relying on this:

1.  Confirm PostgreSQL, API, and tunnel work manually.
2.  Confirm each required service is enabled.
3.  Confirm Termux:Boot has been opened at least once.
4.  Reboot the phone.
5.  Wait for Android and networking to finish starting.
6.  Open Termux and check:

``` bash
pgrep -af runsvdir
sv status postgres
sv status sample-api
sv status cloudflared-sample
```

Then:

``` bash
pg_isready -h 127.0.0.1 -p 5432
curl -i http://127.0.0.1:3000/health
```

Test the public hostname too, if configured. The API waits for
PostgreSQL in its run script; the tunnel may need additional time to
connect after Android networking starts.

## 12. Android background settings

Menu names differ between Android versions and custom ROMs. Look for
equivalent settings.

For **Termux** and **Termux:Boot**:

-   Set battery usage to **Unrestricted** or the least restrictive
    available option.
-   Allow background activity.
-   Enable AutoStart if the ROM provides it.
-   Do not force-stop Termux.
-   Allow required notifications if the OS uses them for ongoing
    background activity.
-   Keep Wi-Fi active during sleep if the device exposes that option.
-   Exclude Termux from aggressive battery cleanup or memory-cleaner
    features.

### Wake lock

The boot script uses:

``` bash
termux-wake-lock
```

Release it manually with:

``` bash
termux-wake-unlock
```

A wake lock can reduce the chance of CPU sleep, but consumes power and
does not guarantee network availability or prevent every Android process
restriction.

### Heat and battery

-   Keep the phone ventilated.
-   Avoid placing it under fabric or in direct sunlight.
-   Monitor temperature during sustained load.
-   Use a good charger and cable.
-   Avoid unnecessary CPU-heavy jobs.
-   Consider battery health if the phone remains plugged in for long
    periods.

## 13. Remote administration over SSH

SSH is optional but useful when managing the phone from a laptop.

Install OpenSSH:

``` bash
pkg install openssh
```

Set a Termux password:

``` bash
passwd
```

Start SSH:

``` bash
sshd
```

Termux OpenSSH commonly listens on port `8022`. Find the current phone
Wi-Fi IP:

``` bash
ip -4 addr show wlan0
whoami
```

From a laptop on the same trusted Wi-Fi network:

``` bash
ssh -p 8022 YOUR_TERMUX_USERNAME@PHONE_LAN_IP
```

For better security, configure SSH key authentication and disable
password login only after verifying that key-based access works.

**Do not expose SSH directly to the public internet without a deliberate
security design.** Prefer LAN access or a private VPN.

## 14. Backups and recovery

A phone can be lost, damaged, reset, or have its storage corrupted. A
server without tested backups is not reliable.

### 14.1 Back up a PostgreSQL database

``` bash
mkdir -p ~/backups/postgres
```

Create a custom-format dump:

``` bash
pg_dump -h 127.0.0.1 -p 5432 \
  -U sample_app -d sample_db \
  -Fc \
  -f "$HOME/backups/postgres/sample_db_$(date +%Y%m%d_%H%M%S).dump"
```

Back up database roles separately when required:

``` bash
pg_dumpall -h 127.0.0.1 -p 5432 \
  -U YOUR_ADMIN_ROLE --globals-only \
  > "$HOME/backups/postgres/globals_$(date +%Y%m%d_%H%M%S).sql"
```

Copy backups off the phone. A backup stored only on the same device does
not protect against device loss or storage failure.

### 14.2 Restore a database

Create an empty target database first, then restore:

``` bash
createdb -h 127.0.0.1 -p 5432 -U YOUR_ADMIN_ROLE sample_db_restore
pg_restore -h 127.0.0.1 -p 5432 \
  -U YOUR_ADMIN_ROLE \
  -d sample_db_restore \
  /path/to/your/database.dump
```

Test the restored database before replacing a live database.

### 14.3 Back up application files

Preserve source code, lockfiles, configuration templates, migrations,
and service definitions. Keep secrets in a secure password manager or
protected backup---not in a public repository.

Useful items to preserve:

-   Application source code and lockfiles.
-   Database dumps.
-   Database role/global definitions.
-   Migration files.
-   Service `run` scripts.
-   Termux:Boot scripts.
-   Cloudflare tunnel recovery information, stored securely.
-   A record of ports, hostnames, and service names.

## 15. Security checklist

Before exposing a backend publicly, verify:

-   [ ] PostgreSQL listens only on `127.0.0.1`.
-   [ ] PostgreSQL uses password authentication for local TCP
    connections.
-   [ ] Each application has its own database role.
-   [ ] Application roles are not PostgreSQL superusers.
-   [ ] `.env` files have restrictive permissions (`chmod 600`).
-   [ ] `.env`, service tokens, private keys, and credentials are
    excluded from Git.
-   [ ] The API binds to loopback when Cloudflare Tunnel is the intended
    ingress.
-   [ ] Public API routes enforce real authentication and authorization.
-   [ ] Internal worker routes use dedicated service authentication.
-   [ ] Rate limits, request validation, and safe error handling are
    enabled.
-   [ ] Database port `5432` is not exposed through a public tunnel.
-   [ ] Cloudflare tunnel tokens are stored in protected files.
-   [ ] Backups are copied off-device and restoration is tested.
-   [ ] Logs do not contain passwords, bearer tokens, or API secrets.
-   [ ] Android battery, boot, and thermal behavior are understood.

### Never do this

Do not solve an authentication error by enabling development
authentication in production, removing Firebase token verification,
making PostgreSQL public, disabling worker-route protections, placing
permanent API secrets in frontend/mobile code, or sharing `.env`
screenshots containing credentials.

## 16. Useful commands

### System and environment

``` bash
uname -m
getprop ro.build.version.release
df -h
free -h
top
```

### Service supervisor

``` bash
pgrep -af runsvdir
ls -la "$PREFIX/var/service"
```

### Service control

``` bash
sv status postgres
sv status sample-api
sv status cloudflared-sample

sv restart sample-api
sv down sample-api
sv up sample-api
```

### PostgreSQL

``` bash
pg_isready -h 127.0.0.1 -p 5432
psql -h 127.0.0.1 -p 5432 -U sample_app -d sample_db
```

### Ports and processes

``` bash
ss -ltn
pgrep -af node
pgrep -af postgres
pgrep -af cloudflared
```

### API checks

``` bash
curl -i http://127.0.0.1:3000/health
curl -i https://api.example.com/health
```

### Logs

``` bash
tail -n 100 "$PREFIX/var/log/sv/sample-api/current"
tail -n 100 "$PREFIX/var/log/sv/postgres/current"
```

A log file exists only when that service has a logger configured. Check
the service directory if the file is absent.

## 17. Troubleshooting

  -------------------------------------------------------------------------------------
  Symptom                 Likely cause            What to check
  ----------------------- ----------------------- -------------------------------------
  `sv status` says `down` Service exited, is      Check `run` script, permissions,
                          disabled, or cannot     logs, and `sv-enable`
                          start                   

  API exits immediately   Wrong working           Run the app manually once to see the
                          directory, missing      actual error
                          `.env`, dependency      
                          error, or port conflict 

  `EADDRINUSE`            Another process owns    `ss -ltnp` and `pgrep -af node`; stop
                          the port                the duplicate before starting runit

  API cannot connect to   Database stopped, wrong `sv status postgres`, `pg_isready`,
  PostgreSQL              credentials, wrong      and test `psql`
                          host/port, or database  
                          not ready               

  PostgreSQL already      A second PostgreSQL     Stop duplicate startup attempts; use
  running                 process was started     only the runit-managed service
                          against the same data   
                          directory               

  Public API returns      Tunnel connector is     `sv status cloudflared-sample`,
  Cloudflare 530 / 1033   offline or disconnected inspect tunnel logs and token path

  Public API returns 502  Tunnel reaches a local  Test
                          service that is not     `curl http://127.0.0.1:3000/health`
                          listening or uses the   on the phone
                          wrong port              

  API works locally but   Tunnel hostname or      Check the Cloudflare hostname target
  not publicly            route configuration is  and connector
                          wrong                   

  Works until phone       Termux:Boot not         Check `~/.termux/boot/`, permissions,
  reboot                  installed, not opened,  and Android battery settings
                          or boot script not      
                          executable              

  Works until screen      Android background      Check battery settings, wake lock,
  turns off               restrictions or         Wi-Fi sleep, and OEM app management
                          CPU/network sleep       

  Port is free but        Application startup     Run the command manually in the
  service fails           error                   project directory and read the full
                                                  error

  Log file is missing     No logger configured    Add a `log/run` script or inspect
                          for that custom service configured service output
  -------------------------------------------------------------------------------------

### Safe diagnostic sequence for a failed Node service

``` bash
sv down sample-api
pgrep -af node
ss -ltnp | grep ':3000'
cd ~/projects/sample-api
NODE_ENV=production node server.js
```

Read the error. Stop the manual test with `Ctrl+C`, fix the cause, then:

``` bash
sv up sample-api
sv status sample-api
```

Do not run the manual server and runit-managed server simultaneously.

## 18. Deploy another backend

The same pattern works for another project:

1.  Create the project under Termux's private home directory.
2.  Install its runtime and dependencies.
3.  Create its `.env` file and protect it.
4.  Create a database and dedicated database role if needed.
5.  Test the backend manually on `127.0.0.1`.
6.  Create a runit service under `$PREFIX/var/service`.
7.  Write an executable foreground `run` script.
8.  Add a logger if persistent logs are needed.
9.  Enable and start the service.
10. Add a Cloudflare hostname only if external access is required.
11. Add a Termux:Boot script once the service setup is stable.
12. Test after a full phone reboot.

### Example: Python FastAPI service

Install Python and FastAPI:

``` bash
pkg install python
mkdir -p ~/projects/python-api
cd ~/projects/python-api
python -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn
```

Create `main.py`:

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok", "service": "python-api"}
```

Test locally:

``` bash
uvicorn main:app --host 127.0.0.1 --port 8000
```

Stop the manual server with `Ctrl+C`, then create its runit service:

``` bash
mkdir -p "$PREFIX/var/service/python-api"
nano "$PREFIX/var/service/python-api/run"
```

Use:

``` sh
#!/data/data/com.termux/files/usr/bin/sh

cd /data/data/com.termux/files/home/projects/python-api || exit 111

exec /data/data/com.termux/files/home/projects/python-api/.venv/bin/uvicorn \
  main:app \
  --host 127.0.0.1 \
  --port 8000
```

Then:

``` bash
chmod 700 "$PREFIX/var/service/python-api/run"
sv-enable python-api
sv up python-api
sv status python-api
```

The same supervision pattern works for other foreground processes. Adapt
the runtime command, environment, health check, and dependencies to the
application.

## 19. Adapting this guide to your project

The sample API and database are only a starting point. For a different
application, change the project-specific configuration while keeping
the same deployment principles.

### Values to customize

- Project directory and service name.
- Runtime and start command (Node.js, Python, Go, Java, etc.).
- API port and health-check endpoint.
- Environment variables and secret-management approach.
- Database engine, database name, role, and backup strategy.
- Public hostname and tunnel configuration, if external access is needed.
- Startup dependencies and service restart behavior.

### Recommended deployment sequence

1. Transfer or clone the application into Termux's private home
   directory.
2. Install the required runtime and dependencies.
3. Create a production environment file. Do not commit it to Git.
4. Create a dedicated database and database user, if the application
   needs a database.
5. Test the database connection independently.
6. Run the backend manually and verify its health endpoint.
7. Stop the manual process before enabling the supervised service.
8. Create a runit service that runs the application in the foreground.
9. Verify service status, logs, API responses, and database access.
10. Configure a tunnel only if the API must be reachable from outside
    your local network.
11. Configure Termux:Boot and Android background settings.
12. Reboot the phone and repeat the health checks.
13. Test backup restoration before relying on the deployment.

### Example project layout

```text
~/projects/my-api/
├── src/
├── config/
├── .env                 # private; never commit
├── .env.example         # safe template; no real secrets
├── .gitignore
├── package.json
└── server.js
```

A suitable `.gitignore` should include at least:

```gitignore
.env
.env.*
!.env.example
node_modules/
logs/
*.log
backups/
```

### Multiple applications on one phone

Each application should have its own project directory, service name,
environment file, and database role. Give each API a different local
port. For example:

| Application | Local API port | Service name |
|---|---:|---|
| Project A | `3000` | `project-a` |
| Project B | `3001` | `project-b` |
| Project C | `8000` | `project-c` |

A single Cloudflare Tunnel can route multiple hostnames to different
local ports. Keep the database ports private and expose only the
application endpoints that need external access.

## 20. Official references

-   [Termux project](https://termux.dev/)
-   [Termux application source and
    installation](https://github.com/termux/termux-app)
-   [Termux:Boot official
    README](https://github.com/termux/termux-boot/blob/master/README.md)
-   [Termux:Boot F-Droid
    listing](https://f-droid.org/packages/com.termux.boot/)
-   [Termux:Services official
    README](https://github.com/termux/termux-services/blob/master/README.md)
-   [runit FAQ --- service supervision](https://smarden.org/runit/faq)
-   [PostgreSQL documentation](https://www.postgresql.org/docs/)
-   [Cloudflare Tunnel
    documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)

------------------------------------------------------------------------

## Final checklist

A phone-hosted backend is ready for a basic always-on test when:

-   [ ] The database starts under runit.
-   [ ] The backend starts under runit.
-   [ ] The backend can query the database.
-   [ ] The API is reachable locally.
-   [ ] Public access, if enabled, goes through the intended tunnel.
-   [ ] Termux:Boot starts the service supervisor after reboot.
-   [ ] Android background and battery settings are configured.
-   [ ] Logs and backups are available.
-   [ ] Authentication and secrets are handled safely.

**Remember:** A phone can be a surprisingly capable little server, but
it remains a mobile device. Use it for learning, personal
infrastructure, and controlled workloads; use a VPS or managed platform
when dependable production uptime matters.
