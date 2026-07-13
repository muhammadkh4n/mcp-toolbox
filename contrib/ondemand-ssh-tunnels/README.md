# On-demand SSH tunnels for database sources (systemd user units)

Per-database SSH tunnels that exist only while a connection is open — the
natural pairing for `TOOLBOX_LAZY_SOURCES=1` (this fork's lazy source init):
starting Toolbox opens no tunnels, and the first tool run against a source
dials its tunnel on demand.

## How it works

Each database gets a pair of **systemd user units**:

- `db-tunnel-PORT.socket` — listens on `127.0.0.1:PORT` with `Accept=yes`,
  so systemd spawns one service instance **per incoming connection**.
- `db-tunnel-PORT@.service` — the instance: an `ssh -W DB_HOST:DB_PORT`
  stdio-forward through your bastion, wired to the accepted socket via
  `StandardInput=socket`/`StandardOutput=socket`.

Nothing runs while idle. Any local MySQL client (Toolbox, an app's dev
server, a one-off script) just connects to `127.0.0.1:PORT`.

## Install

1. Copy the two template files to `~/.config/systemd/user/`, once per
   database, replacing in both filenames and contents:
   - `PORT` — the local listen port (e.g. `3306`, `3307`, …)
   - `DB_HOST:DB_PORT` — the remote database endpoint as seen from the bastion
   - `SSH_USER@BASTION_HOST` — your bastion login
   - `SSH_KEY` — the private key filename under `~/.ssh/`
2. Reload and enable each socket:

   ```sh
   systemctl --user daemon-reload
   systemctl --user enable --now db-tunnel-3306.socket
   ```

3. Point your Toolbox `sources:` (and anything else) at `127.0.0.1:PORT`.

## Pitfall: do NOT add SSH connection multiplexing to the service

It is tempting to speed up per-connection dials with
`-o ControlMaster=auto -o ControlPath=... -o ControlPersist=15s`. **Under
`Accept=yes` socket activation this breaks in a maddening way:** the
persisted control master is forked *inside* one connection's service
instance, and systemd's per-instance cleanup tears it down while the *next*
connection is multiplexing through it. The symptom is every second
connection dying mid-handshake with `Connection lost: The server closed the
connection` (MySQL clients report `PROTOCOL_CONNECTION_LOST`) in a strict
OK/FAIL alternation — while the same ssh command run by hand works every
time, which makes it look like anything except the tunnel.

The templates therefore use one independent ssh per connection. Cost: a
full SSH handshake (~0.5s) per **new** database connection — connection
pools make this a non-issue. If you need faster dials, run a persistent
master in its own long-lived unit instead of letting per-connection
instances own it.

Also deliberate: no `-q`, so ssh failures land in the journal
(`journalctl --user -u "db-tunnel-PORT@*"`) instead of vanishing.
