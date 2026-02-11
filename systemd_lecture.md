# Systemd Lecture: How Services Start at Boot (Even Before Any User Login)

## 1) Core Concepts: Unit, Service, and Install

`systemd` is the init system and service manager used by most modern Linux distributions. It is responsible for bringing the machine from **power-on** to a fully usable system state.

### What is a *unit*?
A **unit** is any object managed by systemd. Unit files are plain-text configuration files and usually live in:
- `/usr/lib/systemd/system/` (or `/lib/systemd/system/`) for package-provided units
- `/etc/systemd/system/` for administrator overrides/custom units

Common unit types:
- `.service` — long-running processes (daemons), one-shot jobs
- `.target` — synchronization/grouping points (similar to runlevels)
- `.socket` — socket activation
- `.timer` — scheduled activation (like cron)
- `.mount`, `.automount`, `.path`, `.device`, etc.

A `.service` file is still a **unit**; “service” is just one specific unit type.

---

### What are `[Unit]`, `[Service]`, and `[Install]` sections?

#### `[Unit]`
Defines metadata and dependency/order relationships:
- `Description=` human-readable name
- `Wants=` soft dependency (attempt to start listed unit)
- `Requires=` hard dependency (if required unit fails, this unit fails)
- `After=` ordering only (start this unit *after* listed units)
- `Before=` ordering only (start this unit *before* listed units)

> Important teaching point: **dependency and ordering are different**.
> - `Wants=/Requires=` answer *what should also be activated*
> - `After=/Before=` answer *in what order*

#### `[Service]`
Defines how the service process is run and supervised:
- `ExecStart=` command to run
- `Type=` startup behavior contract (`simple`, `notify`, `forking`, etc.)
- `Restart=` restart policy
- `User=` / `Group=` runtime identity
- Hardening knobs (`ProtectSystem=`, `NoNewPrivileges=`, …)

#### `[Install]`
Defines how `systemctl enable` creates persistent startup links:
- `WantedBy=multi-user.target` means: when enabled, this service is pulled in by `multi-user.target` at boot

So: **`[Install]` does not start a service by itself**. It defines how enabling wires the unit into the boot target graph.

---

## 2) For a System-Level Unit: When Exactly Does It Start After Boot?

For system services (`/etc/systemd/system/*.service`), startup usually follows this sequence:

1. Kernel initializes hardware and mounts early root filesystems.
2. PID 1 (`systemd`) starts and begins activating default target (`default.target`, often linked to `multi-user.target` or `graphical.target`).
3. systemd resolves the unit dependency graph.
4. Units enabled into boot targets (for example via `WantedBy=multi-user.target`) are pulled in.
5. systemd starts units as soon as dependencies and ordering constraints allow, in parallel where possible.

### Practical timing model
A service starts when all of these are true:
- It is **requested** (enabled target, dependency, or manual start)
- Its ordering constraints (`After=`) are satisfied
- Its required dependencies (if any) are active/successful enough for startup logic

If a service is enabled by `multi-user.target`, it can start during boot **before any interactive login prompt is used**, because login is not a prerequisite for system units.

### Commands to inspect startup timing
- `systemctl is-enabled <unit>` — enabled/disabled state
- `systemctl status <unit>` — current state and recent logs
- `systemd-analyze blame` — time spent starting units
- `systemd-analyze critical-chain <unit>` — boot ordering chain
- `journalctl -u <unit> -b` — logs for this boot

---

## 3) Example A: Why `sshd` Is Running After Boot Without Any Login

Given unit:

```ini
[Unit]
After=network.target sshdgenkeys.service
Before=ssh-access.target
Description=OpenSSH Daemon
Documentation=man:sshd(8) man:sshd_config(5)
Wants=sshdgenkeys.service ssh-access.target

[Service]
Type=notify-reload
ExecStart=/usr/bin/sshd -D
KillMode=process
Restart=always

[Install]
WantedBy=multi-user.target
```

### Why it starts automatically
- `WantedBy=multi-user.target` means when you run `systemctl enable sshd`, systemd creates a symlink so `multi-user.target` pulls in `sshd.service`.
- `multi-user.target` is part of normal non-graphical/server boot, so sshd starts during boot.
- No user login is required; this is a **system daemon**, not a user session process.

### Why remote SSH login is possible
- `ExecStart=/usr/bin/sshd -D` starts the SSH daemon in foreground mode for systemd supervision.
- Once running and listening on port 22 (assuming firewall/network permit), remote clients can connect.
- Authentication happens *after* connection establishment; therefore the daemon must run *before* user login.

### Dependency/ordering interpretation
- `After=network.target sshdgenkeys.service`
  - sshd starts after base network stack target and after host keys generation service ordering-wise.
- `Wants=sshdgenkeys.service ssh-access.target`
  - systemd also tries to activate these units.
- `Before=ssh-access.target`
  - sshd should come up before that synchronization target.

### Teaching nuance: `network.target` vs `network-online.target`
- `network.target` generally indicates network management stack started, not necessarily fully configured connectivity.
- For servers that just need to bind local sockets, this is often enough.

---

## 4) Example B: Ensuring `frpc` Is Available After Boot Without Login

Given unit:

```ini
[Unit]
Description=FRP Client (frpc)
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=frpc
Group=frpc

# Pick ONE of these, depending on your config format:
# ExecStart=/usr/bin/frpc -c /etc/frp/frpc.ini
ExecStart=/usr/bin/frpc -c /etc/frp/frpc.toml

WorkingDirectory=/var/lib/frpc

Restart=on-failure
RestartSec=3s

# Security hardening (safe baseline; loosen only if frpc needs more)
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=true
ProtectControlGroups=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
SystemCallArchitectures=native

# Allow reading config + writing state in WorkingDirectory
ReadWritePaths=/var/lib/frpc
# (Config is read-only under ProtectSystem=strict; that's fine.)

# If you need DNS resolving reliability:
# After=nss-lookup.target
# Wants=nss-lookup.target

# Logging goes to journald by default
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Why this can be available after boot without login
Exactly the same boot principle as sshd:
- Put the unit at system scope (`/etc/systemd/system/frpc.service`)
- `enable` it so it is wanted by `multi-user.target`
- It starts during boot, not at user login

### Why `network-online.target` is used here
- FRP client usually needs actual outbound connectivity early.
- `Wants=network-online.target` + `After=network-online.target` expresses intent to wait for network to be considered online before starting.

### Production checklist for guaranteed auto-start behavior
1. Create unit file: `/etc/systemd/system/frpc.service`
2. Reload manager config:
   ```bash
   sudo systemctl daemon-reload
   ```
3. Enable for boot:
   ```bash
   sudo systemctl enable frpc.service
   ```
4. Start now (optional but common):
   ```bash
   sudo systemctl start frpc.service
   ```
5. Verify:
   ```bash
   systemctl is-enabled frpc.service
   systemctl status frpc.service
   journalctl -u frpc.service -b
   ```

If `is-enabled` shows `enabled`, it will be pulled in on next boot even if nobody logs in locally.

---

## 5) “No-Login Boot” Mental Model (Important for Admins)

A frequent misconception is:
> “Services only run after someone logs in.”

For **system units**, this is false.
- Boot-time services are managed by PID 1 (`systemd`) as part of boot target activation.
- Login managers/getty are themselves just other units that start in parallel according to dependencies.
- Remote accessibility (SSH, FRP tunnel, web server, etc.) depends on service enablement + networking/firewall, not on local interactive login.

---

## 6) Minimal Troubleshooting Flow

If a service is not available after reboot:

1. Check enablement:
   ```bash
   systemctl is-enabled <name>.service
   ```
2. Check boot logs:
   ```bash
   journalctl -u <name>.service -b
   ```
3. Check dependency chain:
   ```bash
   systemd-analyze critical-chain <name>.service
   ```
4. Confirm network assumptions:
   - Should it use `network.target` or `network-online.target`?
5. Check security hardening side effects:
   - `ProtectSystem=strict`, `ProtectHome=true`, and similar options can block file access if paths are not whitelisted.

---

## 7) Key Takeaways

- A **unit** is a systemd-managed object; a **service** is one unit type.
- `[Install] WantedBy=multi-user.target` + `systemctl enable` is the classic way to auto-start system daemons at boot.
- Boot-time startup does **not** require local login.
- `After=` controls order; `Wants=/Requires=` control pull-in/necessity.
- `sshd` and `frpc` both become available pre-login when enabled as system services with appropriate network dependencies.

