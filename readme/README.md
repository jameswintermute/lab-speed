# lab-speed

A rapid lab setup utility for short‑lived training labs: distribute files to lab hosts and open SSH sessions quickly.

> Designed for **speed**: get connected and ready in minutes.

---

## Fast Start

### 1) Get the repo

```bash
git clone https://github.com/jameswintermute/lab-speed
cd lab-speed
chmod +x lab-speed.bin
```

### 2) Fill in `local/credentials.txt`

Run the tool once and it will prompt you (and store with `chmod 600`):

```bash
./lab-speed.bin
```

Or edit manually:

`local/credentials.txt`
```bash
# lab-speed credentials (chmod 600)
username="YOUR_LAB_USERNAME"
password="YOUR_LAB_PASSWORD"
SSHPASS="YOUR_LAB_PASSWORD"   # usually same as password
```

### 3) Fill in `local/hosts.csv`

Required columns:
- `external-ip` (or `external_ip`)
- `function` (friendly label shown in menus)

Example:

```csv
host-url,external-ip,internal-ip,function
https://example-cm,1.2.3.4,10.0.0.1,Splunk-CM
https://example-idx1,2.3.5.157,10.0.0.2,Splunk-IDX1
```

### 4) Put files you want copied into `files-to-copy/`

Everything in `files-to-copy/` is copied to each lab host:

Remote target:
```
/tmp/lab-speed/
```

### 5) Run **GO!** (menu option 1)

From the menu:

- **1) GO! (install ssh key, RSYNC files)**

This will:
1. Validate inputs (`hosts.csv`, `credentials.txt`, `files-to-copy/`)
2. Provision each host by copying `files-to-copy/` → `/tmp/lab-speed/`
3. Print a clear end summary (how many hosts succeeded) and recommend next steps

---

## Menu overview

- **1) GO!** — fast provision all hosts (with a progress meter)
- **2) SSH to host** — shows a function‑first list, then opens an SSH session (in a new terminal when available)
- **3) RSYNC again** — re-push `files-to-copy/` to all hosts (progress meter)
- **7) Clean up** — remove local credentials file

---

## Dependency check

**Required**
- `bash`
- `ssh` (package: `openssh-client`)
- `rsync`
- `awk`, `sed`

**Recommended / used for non-interactive runs**
- `sshpass`

### Install dependencies (Ubuntu/Debian)

```bash
sudo apt-get update && sudo apt-get install -y openssh-client rsync gawk sed sshpass
```

If you do NOT have admin rights, ask your lab/instructor/admin to install:
- `openssh-client`, `rsync`, `gawk`, `sed`, `sshpass`

---

## Troubleshooting

### “hostname contains invalid characters”
Usually caused by hidden `\r` characters in `hosts.csv` (Windows line endings).

Fix:
```bash
sed -i 's/\r$//' local/hosts.csv
```

### SSH session opens then immediately closes
If your terminal emulator opens a new window and closes on exit, that is expected.
If you want it to stay open, run SSH from the main window (or adjust your terminal profile).

---

## License

GPLv3 — see `LICENSE`.
