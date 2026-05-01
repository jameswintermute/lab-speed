# lab-speed
A rapid lab setup utility for short‑lived training labs: distribute files to lab hosts, open SSH sessions, broadcast commands across all hosts at once, and verify everything is healthy — quickly.
> Designed for **speed**: get connected and ready in minutes, then keep working without losing time to repetitive plumbing.

---

## What's new in v1.5.0
- **NEW — Broadcast (option 8)**: opens a tmux session in a new terminal with one pane per host, `synchronize-panes` ON by default. Type a command once, hit every host. Per-pane borders show which host is which. Requires `tmux` (optional dependency — only needed for this option).
- **NEW — Health check (option 9)**: parallel SSH to every host; reports hostname, uptime, free disk on `/`, free memory, Splunk service status (up/down) and Splunk version. Useful immediately after **GO!** to catch silent failures (a host that rsynced fine but has e.g. a wedged Splunk service) before they cost you an hour.
- Dependency check now flags `tmux` as an optional dependency.

## What's new in v1.4.0
- **FIX — line 310 crash on GO! / RSYNC again**: under `set -u`, the cleanup `RETURN` trap on a `local` variable expanded after the variable had gone out of scope, aborting the script with `tmpdir: unbound variable` after rsync had already succeeded. Fixed by expanding the path at trap-definition time.
- **NEW — chmod 777 after rsync**: files copied to remote hosts are now `chmod -R 777`. Convenient for lab work where you may need to read/edit/execute everything without fighting permissions. Announced once per run.
- **NEW — URLs (option 6)**: lists the `host-url` column from `local/hosts.csv`, numbered alongside the function name, so lab web UIs are easy to find and copy.

## What's new in v1.3.0
- Parallel provisioning, GUI terminal support (gnome-terminal/xterm/konsole/alacritty/wezterm), per-host title in tabs, log rotation, robust SSH options (ConnectTimeout + ServerAlive), parallel SSH key install (option 4).

---

## Fast Start

### 1) Get the repo
```bash
git clone https://github.com/jameswintermute/lab-speed
cd lab-speed
chmod +x lab-speed.bin
```

### 2) Fill in `local/credentials.txt`
Run the tool once — it will prompt and store with `chmod 600`:
```bash
./lab-speed.bin
```
Or edit manually:

`local/credentials.txt`
```bash
# lab-speed credentials (chmod 600)
username="YOUR_LAB_USERNAME"
password="YOUR_LAB_PASSWORD"
SSHPASS="YOUR_LAB_PASSWORD"   # must match password
```

> **Note:** `SSHPASS` is passed via environment variable (not the command line) so it does not appear in the process list.

### 3) Fill in `local/hosts.csv`
Required columns (column order does not matter — they're matched by header name):
- `host-url` — the lab web URL for option 6 (URLs)
- `external-ip` — the IP lab-speed connects to
- `function` — friendly label (e.g. `CM`, `IDX1`, `SH1`)

Example:
```csv
host-url,external-ip,internal-ip,function
https://example-cm.lab,1.2.3.4,10.0.0.1,CM
https://example-idx1.lab,2.3.5.157,10.0.0.2,IDX1
```

### 4) Put files you want copied into `files-to-copy/`
Everything in `files-to-copy/` is copied to each lab host. Remote target: `/tmp/lab-speed/`. Permissions are set to `777` after copy so you can read/edit/execute without fighting access.

### 5) Run **GO!** (menu option 1)
- **1) GO! (RSYNC files to all hosts)**

This will:
1. Validate inputs (`hosts.csv`, `credentials.txt`, `files-to-copy/`).
2. Provision **all hosts in parallel** — progress is shown live.
3. Run `chmod -R 777` on the remote target on each host.
4. Print a clear end summary and recommend next steps.

### 6) (Recommended) Run **Health check** (option 9)
Right after GO!, run the health check to confirm every host actually came up the way you expect — same status for everything is the green light to start work.

---

## Menu overview

| # | Action | Notes |
|---|--------|-------|
| 1 | GO! | Provision all hosts in parallel (rsync + chmod 777) |
| 2 | SSH | Function-first host list; opens a password-free session (new terminal when available) |
| 3 | RSYNC again | Re-push `files-to-copy/` to all hosts; skips dependency check for speed |
| 4 | Install SSH keys | Parallel `ssh-copy-id` to every host (subsequent SSH no longer needs sshpass) |
| 5 | Dependency check | Verifies required & optional tools are installed |
| 6 | URLs | Lists the `host-url` column from `hosts.csv` (handy for browser tabs) |
| 7 | Clean Up | Removes `local/credentials.txt` |
| 8 | Broadcast | Opens tmux in a new terminal — one pane per host, type once, hit every host |
| 9 | Health check | Parallel SSH; reports uptime, disk, memory, Splunk status + version |

---

## Broadcast (option 8) — the big lab time-saver

Opens tmux in a new terminal window with **one pane per host** and **`synchronize-panes` turned on by default**. Whatever you type goes to every host at once — `sudo systemctl restart splunkd`, `tail /opt/splunk/var/log/splunk/splunkd.log`, `df -h`, anything.

### Tmux keybindings while inside the broadcast session
- `Ctrl-b :setw synchronize-panes` — toggle sync on/off (use this when you need to run something on just one host)
- `Ctrl-b o` — cycle to next pane
- `Ctrl-b z` — zoom current pane (full screen, press again to un-zoom)
- `Ctrl-b d` — detach (leaves the session running; reattach with `tmux attach`)
- `Ctrl-b : kill-session` — close everything

### Common workflow
1. **Sync ON** — install packages, edit configs, restart services in lockstep across all hosts.
2. **Sync OFF + Ctrl-b o** — when one indexer needs different config from the others, toggle sync off, cycle to the right pane, do your one-off, toggle sync back on.
3. Pane borders show the function name (CM / IDX1 / SH1 / …) so you always know where you are.

### Requires
`tmux`. Install on Ubuntu/Debian:
```bash
sudo apt-get install -y tmux
```

If tmux isn't installed, option 8 prints an install hint and returns to the menu.

---

## Health check (option 9) — catch silent failures

Runs in parallel against every host and renders a single table:

```
  FUNC     IP              HOSTNAME     UPTIME          DISK/   MEM     SPLUNK    VER
  ----     --              --------     ------          -----   ---     ------    ---
  CM       18.118.167.232  cm1          2 hours         42G     6.1Gi   up        9.4.1
  MC       3.15.234.123    mc1          2 hours         44G     6.3Gi   up        9.4.1
  IDX1     18.118.111.63   idx1         2 hours         38G     5.8Gi   up        9.4.1
  IDX2     3.14.148.9      UNREACHABLE
  SH1      52.15.252.138   sh1          2 hours         41G     6.0Gi   up        9.4.1
```

Splunk status is colored: green for `up`, yellow for `down`. Unreachable hosts are flagged in red. Hosts without Splunk installed simply show `-` for the version — the tool isn't Splunk-specific; it just surfaces it when present.

The remote probe is small and POSIX-y (`hostname`, `uptime`, `df`, `free`, `pgrep`) so it works across most lab images without setup.

---

## Dependency check

### Required
- `bash`
- `ssh` (package: `openssh-client`)
- `rsync`
- `sshpass`
- `awk`, `sed`

### Optional
- `ssh-copy-id` — needed for option 4 (Install SSH keys)
- `tmux` — needed for option 8 (Broadcast)

### Install everything (Ubuntu/Debian)
```bash
sudo apt-get update && sudo apt-get install -y \
  openssh-client rsync gawk sed sshpass tmux
```

If you don't have admin rights, ask your lab instructor or admin to install: `openssh-client`, `rsync`, `gawk`, `sed`, `sshpass`, and (optionally) `tmux`.

---

## Troubleshooting

### "hostname contains invalid characters"
Hidden `\r` characters in `hosts.csv` (Windows line endings). Fix:
```bash
sed -i 's/\r$//' local/hosts.csv
```

### SSH session opens then immediately closes
Some terminal profiles close the window on exit. Either adjust your terminal profile, or run SSH from the main lab-speed window (no GUI terminal launch).

### rsync fails with "unknown option --mkpath"
lab-speed uses `rsync --mkpath` (rsync ≥ 3.2.3) for a single-connection copy. On older rsync, it falls back to a two-step mkdir + rsync automatically — no action needed. Check your version:
```bash
rsync --version | head -1
```

### A host shows as failed but SSH works manually
Check that the IP in `hosts.csv` is the **external** IP (not internal). Verify username and password in `local/credentials.txt`. Per-host failures are also logged to `logs/lab-speed-<timestamp>.log`.

### Broadcast: tmux opens but panes are tiny / overlapping
On 7+ hosts, narrow terminal windows can crowd panes. Either:
- Maximise the terminal window before launching option 8, or
- Inside tmux use `Ctrl-b z` to zoom the current pane to full screen (`Ctrl-b z` again to un-zoom).

### Health check: a host reports `?` for every field
Means the SSH succeeded but the remote shell failed to run the probe — usually a non-bash login shell (e.g. some images default to `dash` or have a hardened `rbash`). The tool counts it as "reachable" because SSH worked, but the values aren't usable. Investigate the lab image's default shell.

### Health check: Splunk shows `down` but it's actually running
The probe uses `pgrep -x splunkd` and checks `/opt/splunk/bin/splunk version`. If Splunk is installed somewhere else (`/opt/splunkforwarder`, `/Applications/Splunk`, etc.), the version field will say `-` and status may say `down` even when the service is up. The probe is intentionally simple to keep it fast; for non-standard installs use option 2 to ssh in and check directly.

---

## License
GPLv3 — see `LICENSE`.
