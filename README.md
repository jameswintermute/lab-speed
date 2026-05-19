# lab-speed
A rapid lab setup utility for short‑lived training labs: distribute files to lab hosts, open SSH sessions, broadcast commands across all hosts at once, and verify everything is healthy — quickly.
> Designed for **speed**: get connected and ready in minutes, then keep working without losing time to repetitive plumbing.

---

## What's new in v1.11.1
- **FIX — fresh-clone friendliness.** On a fresh clone, picking any option that needs lab context (e.g. `p` Show password, `1` GO!, `2` SSH) used to drop you into a manual `Lab username:` prompt — confusing if you reached for an option expecting an already-loaded session. Now the script detects the "no session yet" state and offers to jump straight to Import instead. Decline and you return to the menu cleanly.

## What's new in v1.11.0
- **NEW — `p` Show password** in the Keys section. Displays the lab password for ~10 seconds, then auto-clears. Press Enter during the countdown to clear immediately. If `xclip`, `wl-copy`, or `pbcopy` is installed, the password is silently copied to the OS clipboard at the same time.
- **NEW — `c` Refresh.** Explicit menu redraw (and clock refresh). Any key works for refresh because the menu loop redraws on every iteration, but `c` is discoverable.
- **FIX — Import column-swap bug.** The Import parser could swap the SSH user and the lab password when the password happened to be alphanumeric (e.g. `7rxg9krr` would land as `username` and `sccStudent` as `SSHPASS`). The parser now uses positional anchors — the literal `admin` and `Ready` cells in the portal table — to lock columns in place. Falls back to the old heuristic only when the anchors are missing (non-standard paste).
- **Clock display** dropped seconds (now `Lab time: 00h 04m`, not `00h 04m 23s`) since it only refreshes on menu redraw — second-level accuracy was misleading. The caveat line was updated: "(elapsed since servers listed — press 'c' to refresh)".
- **Clean up (option 9)** now performs a full lab-session reset: removes credentials, hosts list, AND the lab-clock anchor (`.lab-start`). Confirmation prompt lists exactly what will be removed. Import backups (`.bak-<timestamp>`) and retrieved files (`local/retrieved/`) are explicitly preserved.

## What's new in v1.10.0
- **Menu options renumbered to match visual reading order.** Numbers now count 1→9 top-to-bottom inside the section groupings introduced in v1.8. `GO!` (1) and `SSH` (2) are unchanged; the options that had drifted out of sequence have moved:

  | Action | Old | New |
  |---|---|---|
  | Broadcast | 8 | **3** |
  | URLs | 6 | **4** |
  | RSYNC again | 3 | **5** |
  | Install SSH keys | 4 | **6** |
  | Dependency check | 5 | **7** |
  | Health check | 9 | **8** |
  | Clean up | 7 | **9** |

  Letters are unchanged: `i`, `r`, `m`, `q`.

## What's new in v1.9.0
- **NEW — Lab time clock.** A live "Lab time: HHh MMm SSs" counter appears below the version banner once you've run Import. Goes yellow at 3h30m, red at 4h — handy reminder that the cloud lab is about to time out. Hidden if no import has been run, or if the timestamp is stale (>72h). Override the thresholds in `local/credentials.txt`:
  ```
  LAB_WARN_SEC=12600    # yellow at 3h30m  (default)
  LAB_LIMIT_SEC=14400   # red    at 4h     (default)
  ```
- **NEW — Repo URL footer.** Dim grey URL at the bottom of every menu — useful if a screenshot gets shared.

## What's new in v1.8.0
- **Menu grouped into sections** (setup / connect / files / keys / checks). Same options, same keystrokes — just easier to scan.

## What's new in v1.7.0
- **NEW — Import hosts from paste (option `i`)**: paste the lab credentials table straight from the lab portal webpage. lab-speed parses the tab- or whitespace-separated rows, derives the function name from each URL (cm1 → CM, idx1 → IDX1, shcd → SHC-Dep, …), and writes both `local/hosts.csv` and `local/credentials.txt` in one step. Existing files are backed up to `.bak-<timestamp>` first.
- **NEW — Retrieve files (option `r`)**: pull files from `/tmp/lab-speed/` on selected hosts back to `local/retrieved/<timestamp>/<func>/`. Supports `a` (all), comma-separated singles (`3,7,8`), ranges (`3-5`), or combinations (`1,3-5,7`).
- **Internal IPs now visible** in option 2 (SSH menu), option 8 (Health check), and option `r` (Retrieve). The `hosts.csv` format is unchanged.

## What's new in v1.6.0
- **NEW — Mesh SSH keys (option `m`)**: sets up *host-to-host* passwordless SSH (option 6 only does laptop→host). Two modes — hub-and-spoke (recommended) or full mesh. Private keys never leave the lab.

## What's new in v1.5.0
- **NEW — Broadcast (option 3)**: tmux all-hosts session, `synchronize-panes` on by default. Type once, hit every host.
- **NEW — Health check (option 8)**: parallel SSH; reports uptime, disk, memory, Splunk service status and version.

## What's new in v1.4.0
- **FIX — line 310 crash on GO! / RSYNC again**: under `set -u`, the cleanup `RETURN` trap on a `local` variable expanded after the variable had gone out of scope, aborting the script with `tmpdir: unbound variable` after rsync had already succeeded. Fixed by expanding the path at trap-definition time.
- **NEW — chmod 777 after rsync**: files copied to remote hosts are now `chmod -R 777`.
- **NEW — URLs (option 4)**: lists the `host-url` column from `local/hosts.csv`.

---

## Fast Start

### 1) Get the repo
```bash
git clone https://github.com/jameswintermute/lab-speed
cd lab-speed
chmod +x lab-speed.bin
```

### 2) Import the lab credentials (option `i` — recommended)
The fastest path: from the menu, press `i`, paste the credentials table directly from your lab portal webpage (tabs or whitespace both work), then press Enter on an empty line. lab-speed parses the table and writes both `local/hosts.csv` and `local/credentials.txt` for you. The lab clock also starts counting from this moment. See [Import hosts](#import-hosts-option-i--paste-from-the-lab-portal) below for the full walkthrough.

If you'd rather fill them in by hand, the format is:

`local/credentials.txt` (chmod 600):
```bash
username="YOUR_LAB_USERNAME"
password="YOUR_LAB_PASSWORD"
SSHPASS="YOUR_LAB_PASSWORD"   # must match password
```

`local/hosts.csv` — required columns (column order doesn't matter, matched by header name):
- `host-url` — the lab web URL for option 4 (URLs)
- `external-ip` — the IP lab-speed connects to
- `internal-ip` — shown in options 2, 8, and r (optional)
- `function` — friendly label (e.g. `CM`, `IDX1`, `SH1`)

```csv
host-url,external-ip,internal-ip,function
https://example-cm.lab,1.2.3.4,10.0.0.1,CM
https://example-idx1.lab,2.3.5.157,10.0.0.2,IDX1
```

> **Note:** `SSHPASS` is passed via environment variable (not the command line) so it does not appear in the process list.

### 3) Put files you want copied into `files-to-copy/`
Everything in `files-to-copy/` is copied to each lab host. Remote target: `/tmp/lab-speed/`. Permissions are set to `777` after copy so you can read/edit/execute without fighting access.

### 4) Run **GO!** (menu option 1)
This will:
1. Validate inputs (`hosts.csv`, `credentials.txt`, `files-to-copy/`).
2. Provision **all hosts in parallel** — progress is shown live.
3. Run `chmod -R 777` on the remote target on each host.
4. Print a clear end summary and recommend next steps.

### 5) (Recommended) Run **Health check** (option 8)
Right after GO!, run the health check to confirm every host actually came up the way you expect — same status for everything is the green light to start work.

### 6) (At the end of the lab) **Retrieve files** (option `r`)
Pull `/tmp/lab-speed/` back from any hosts you care about — useful for grabbing logs, exercise output, or modified configs before the lab tears down.

---

## Menu overview

```
── setup ──────────────────────────
i) Import hosts from paste
1) GO!         (rsync files to all hosts)

── connect ──────────────────────────
2) SSH         (open host session)
3) Broadcast   (tmux all-hosts session)
4) URLs        (list lab web URLs)

── files ──────────────────────────
5) RSYNC again (laptop → hosts)
r) Retrieve    (hosts → laptop)

── keys ──────────────────────────
6) Install SSH keys  (laptop → host)
m) Mesh SSH keys     (host → host)
p) Show password    (copies to clipboard, hides in 10s)

── checks ──────────────────────────
7) Dependency check
8) Health check

9) Clean up        c) Refresh        q) Quit
```

| Key | Action | Notes |
|-----|--------|-------|
| **i** | **Import hosts** | Paste the lab portal table → writes `hosts.csv` + `credentials.txt`; starts the lab clock |
| 1 | GO! | Provision all hosts in parallel (rsync + chmod 777) |
| 2 | SSH | Function-first host list with both IPs; opens a password-free session |
| 3 | Broadcast | Opens tmux in a new terminal — one pane per host, type once, hit every host |
| 4 | URLs | Lists the `host-url` column from `hosts.csv` (handy for browser tabs) |
| 5 | RSYNC again | Re-push `files-to-copy/` to all hosts (laptop → hosts) |
| **r** | **Retrieve files** | Pull `/tmp/lab-speed/` back from selected hosts (hosts → laptop) |
| 6 | Install SSH keys | Parallel `ssh-copy-id` from **your laptop** to every host |
| m | Mesh SSH keys | Sets up **host → host** passwordless SSH (hub-and-spoke or full mesh) |
| **p** | **Show password** | Displays the lab password for 10s; copies to clipboard if `xclip` / `wl-copy` / `pbcopy` is installed |
| 7 | Dependency check | Verifies required & optional tools are installed |
| 8 | Health check | Parallel SSH; reports uptime, disk, memory, both IPs, Splunk status + version |
| 9 | Clean up | Full lab-session reset: removes `credentials.txt`, `hosts.csv`, and `.lab-start` (confirmation required) |
| **c** | **Refresh** | Redraw the menu and refresh the lab clock |
| q | Quit | |

> **First-run friendliness (v1.11.1+):** options that need lab context (`p`, `1`, `2`, `3`, `4`, `5`, `r`, `6`, `m`, `8`) detect a fresh-clone state and offer to jump to Import instead of dropping you into a manual credentials prompt.

---

## Lab time clock

After option `i` runs, the banner picks up a counter:

```
      v1.11.1 - May 2026. James Wintermute

      Lab time: 01h 23m
      (elapsed since servers listed — press 'c' to refresh)
```

The clock is colored:
- **Dim** (neutral) for the first 3h30m.
- **Yellow** at 3h30m — "approaching limit".
- **Red** at 4h — "past typical cut-off".

It refreshes every time you bounce back to the main menu (no background process). The display shows hours and minutes only — second-level accuracy would be misleading given the refresh model. Press `c` from the menu to force a redraw, or just pick any option and come back.

If your lab has a different cut-off, override the thresholds in `local/credentials.txt`:

```bash
LAB_WARN_SEC=10800    # yellow at 3h
LAB_LIMIT_SEC=12600   # red    at 3h30m
```

Re-running Import resets the clock to zero. If `local/.lab-start` is older than 72 hours (stale from a previous lab), the clock is hidden until you import again. Option 9 (Clean up) also clears the clock as part of the full session reset.

---

## Import hosts (option `i`) — paste from the lab portal

The lab portal typically shows a table like:

```
URL                                           External IP    Internal IP   Admin   Password   SSH User    Status
https://esi3-…-cm1.students.splunk.education  18.219.29.53   10.0.72.87    admin   38pehhg6   sccStudent  Ready
https://esi3-…-mc1.students.splunk.education  3.128.188.138  10.0.76.249   admin   38pehhg6   sccStudent  Ready
…
```

Previously you'd copy this and hand-edit it into `local/hosts.csv` — tabs to commas, removing columns you don't need, adding the function column. **Option `i` automates that:**

1. From the menu, press `i`.
2. Paste the table (tabs or runs of whitespace both work).
3. Press **Enter on an empty line** (or **Ctrl-D**) to finish.
4. lab-speed parses each row, derives the **function** name from the URL (cm1 → CM, idx1 → IDX1, shcd → SHC-Dep, etc), detects the SSH user and lab password, and shows a preview.
5. Confirm with `y` and it writes:
   - `local/hosts.csv` with the parsed rows.
   - `local/credentials.txt` with `username`, `password`, and `SSHPASS` set.
   - `local/.lab-start` with the current timestamp, starting the lab clock.
6. Any existing files are backed up to `.bak-<timestamp>` first — safe to re-run.

### Function-name mapping

| URL token | Function |
|-----------|----------|
| `cm1` / `cm2` | `CM` / `CM2` |
| `mc1` / `mc2` | `MC` / `MC2` |
| `idx1`, `idx2`, … | `IDX1`, `IDX2`, … |
| `sh1`, `sh2`, `sh3` | `SH1`, `SH2`, `SH3` |
| `shcd` | `SHC-Dep` (Search Head Cluster Deployer) |
| `hf1` | `HF1` (Heavy Forwarder) |
| `lm1` | `LM` (License Manager) |
| `ldap` | `LDAP` |
| anything else | uppercase of the URL token verbatim |

If you get an unexpected function name, just edit `local/hosts.csv` afterwards — the rest of the tool only cares about the column values, not how they were derived.

---

## Retrieve files (option `r`) — pull from hosts

At the end of a lab you often want to grab the configs, logs, or exercise output sitting in `/tmp/lab-speed/` on each host. Option `r` does that in parallel:

```
  #  Function               External IP      Internal IP
  -  --------               -----------      -----------
  1) CM                     18.219.29.53     10.0.72.87
  2) MC                     3.128.188.138    10.0.76.249
  3) IDX1                   3.17.141.176     10.0.78.144
  …
Selection: 'a' for all, comma-separated (3,7), ranges (3-5), or combine (3,7-9).
```

Selection syntax:

- `a` — all hosts
- `3` — just host 3
- `3,7,8` — hosts 3, 7, and 8
- `3-5` — hosts 3 through 5
- `1,3-5,7` — combinations work

Files land in `local/retrieved/<timestamp>/<func>/` so multiple retrievals don't clobber each other. The `chmod -R 777` that option 1 applies after rsync means everything is readable for the pull — no permissions friction.

---

## Show password (option `p`) — for pasting back into the portal

Splunk lab portals frequently ask you to re-enter the lab admin password — sometimes several times during a single lab. Option `p` puts the password on screen for 10 seconds (long enough to select & copy, or copy from the clipboard if you've got a clipboard tool installed) and then auto-clears the screen.

```
  Lab password (hides in 10s):

    7rxg9krr

  copied to X clipboard — paste with Ctrl-V (or Cmd-V).

  Auto-clearing in  7s (press Enter to clear now)...
```

The countdown ticks down on a single line, so you don't lose vertical space. Pressing Enter clears it immediately.

### Clipboard support (optional)

If one of these is installed, the password is silently copied to the OS clipboard at the same time as being displayed:

| OS | Tool | Install |
|---|---|---|
| Linux / Wayland | `wl-copy` | `sudo apt install wl-clipboard` |
| Linux / X11 | `xclip` | `sudo apt install xclip` |
| macOS | `pbcopy` | preinstalled |

Without any of those, the option still works — you just select & copy by hand.

---

## Mesh SSH keys (option `m`) — host-to-host passwordless SSH

Option 6 sets up **your laptop → each host**. Option `m` sets up **host → host**, which is what you need for distributed-Splunk operations (cluster-bundle pushes, `splunk diag` collection, ad-hoc `scp` between hosts).

**Hub-and-spoke** (recommended): pick one host as the hub (typically the CM or a SH); the hub gets passwordless SSH to every host including itself. Fast, simple, covers most lab work.

**Full mesh**: every host can SSH to every host. Useful if your exercises require any-host-to-any-host connectivity. The tool warns and asks for confirmation first.

Implementation: lab-speed SSHes to each source host and runs `ssh-keygen` *on the host* if no key exists. The public key is read back; **the private key never leaves the host**. The collected public keys are then appended to `~/.ssh/authorized_keys` on every host in parallel, deduped in place — so re-running is idempotent.

---

## Broadcast (option 3) — the big lab time-saver

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

If tmux isn't installed, option 3 prints an install hint and returns to the menu.

---

## Health check (option 8) — catch silent failures

Runs in parallel against every host and renders a single table:

```
  FUNC     EXT-IP          INT-IP        HOSTNAME     UPTIME          DISK/   MEM     SPLUNK    VER
  ----     ------          ------        --------     ------          -----   ---     ------    ---
  CM       18.118.167.232  10.0.72.87    cm1          2 hours         42G     6.1Gi   up        9.4.1
  MC       3.15.234.123    10.0.76.249   mc1          2 hours         44G     6.3Gi   up        9.4.1
  IDX1     18.118.111.63   10.0.78.144   idx1         2 hours         38G     5.8Gi   up        9.4.1
  IDX2     3.14.148.9      10.0.102.61   UNREACHABLE
  SH1      52.15.252.138   10.0.123.5    sh1          2 hours         41G     6.0Gi   up        9.4.1
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
- `ssh-copy-id` — needed for option 6 (Install SSH keys)
- `tmux` — needed for option 3 (Broadcast)

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

### Import: a row didn't parse
The parser anchors columns to the literal `admin` and `Ready` cells in the lab portal table (introduced in v1.11.0 to fix a swap bug where alphanumeric passwords were being mistaken for the SSH user). If a row is silently dropped, check that:
- It actually contains a URL (not just a hostname).
- The row contains the words `admin` and `Ready` — these are the column anchors.
- The external IP is a public IP and the internal IP is RFC1918 (10.x, 172.16–31.x, 192.168.x).
- The columns aren't merged into one (paste preserved tabs or runs of whitespace).

For non-standard pastes that lack the `admin`/`Ready` anchors, the parser falls back to a heuristic (first identifier wins for SSH user, anything else for password) — which can still swap the two if the password is alphanumeric. If that happens, edit `local/credentials.txt` by hand.

Header rows are auto-skipped if they contain no IP/URL.

### "No lab session yet" appears when picking an option
This is normal on a fresh clone, or after running option 9 (Clean up). The script needs the lab credentials and host list before it can do most things. Accept the offer to run Import (it'll prompt you to paste the lab portal table), and you're ready in a few seconds.

### SSH session opens then immediately closes
Some terminal profiles close the window on exit. Either adjust your terminal profile, or run SSH from the main lab-speed window (no GUI terminal launch).

### rsync fails with "unknown option --mkpath"
lab-speed uses `rsync --mkpath` (rsync ≥ 3.2.3) for a single-connection copy. On older rsync, it falls back to a two-step mkdir + rsync automatically — no action needed. Check your version:
```bash
rsync --version | head -1
```

### A host shows as failed but SSH works manually
Check that the IP in `hosts.csv` is the **external** IP (not internal). Verify username and password in `local/credentials.txt`. Per-host failures are also logged to `logs/lab-speed-<timestamp>.log`.

### Lab time clock seems wrong
The clock anchors to `local/.lab-start` (a hidden file written by option `i`). If it shows a wildly wrong value, either:
- The system clock has drifted — check `timedatectl status`.
- You imported a long time ago and a new lab session was started without a fresh import — re-run option `i` to reset.

If the clock is hidden but you expected it: either you haven't run Import yet, or the `.lab-start` file is >72 hours old (stale-protection). Re-import to refresh it.

### Broadcast: tmux opens but panes are tiny / overlapping
On 7+ hosts, narrow terminal windows can crowd panes. Either:
- Maximise the terminal window before launching option 3, or
- Inside tmux use `Ctrl-b z` to zoom the current pane to full screen (`Ctrl-b z` again to un-zoom).

### Health check: a host reports `?` for every field
Means the SSH succeeded but the remote shell failed to run the probe — usually a non-bash login shell (e.g. some images default to `dash` or have a hardened `rbash`). The tool counts it as "reachable" because SSH worked, but the values aren't usable. Investigate the lab image's default shell.

### Health check: Splunk shows `down` but it's actually running
The probe uses `pgrep -x splunkd` and checks `/opt/splunk/bin/splunk version`. If Splunk is installed somewhere else (`/opt/splunkforwarder`, `/Applications/Splunk`, etc.), the version field will say `-` and status may say `down` even when the service is up. The probe is intentionally simple to keep it fast; for non-standard installs use option 2 to ssh in and check directly.

### Mesh SSH (option `m`): "still prompts for password between hosts"
Three things to check:
1. **Did the keygen step succeed for that source host?** Re-run option `m` and watch the Phase 1 output — any `[WARN]` line names a host where keygen or fetch failed (often a non-bash login shell).
2. **Is `~/.ssh/authorized_keys` permissions tight?** Some sshd configs reject the file if it's group- or world-writable. The remote step explicitly chmods to 600, but if the home directory itself is group-writable sshd will refuse the key. Fix on the affected host: `chmod g-w,o-w ~`.
3. **First connection between two hosts is asking about host-key verification.** That's StrictHostKeyChecking, not authentication — it's a one-time prompt, not a password prompt. Type `yes` or pre-seed `~/.ssh/known_hosts` if that's a problem for your scripts.

---

## License
GPLv3 — see `LICENSE`.
