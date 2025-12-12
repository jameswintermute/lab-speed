# lab-speed

A rapid lab setup utility for short-lived training labs: generate a disposable SSH key, distribute files via rsync, and apply a helpful shell prompt per host.

> Designed for **speed**: get connected and ready in minutes (even if you only have an hour).

---

## Fast Start (recommended)

### 1) Get the repo

```bash
git clone https://github.com/jameswintermute/lab-speed
cd lab-speed
chmod +x lab-speed.bin
```

### 2) Fill in `local/credentials.txt`

Run the tool once and it will prompt you (and store with chmod 600):

```bash
./lab-speed.bin
```

Or edit manually:

`local/credentials.txt`
```bash
# lab-speed credentials (chmod 600)
username="value"
password="YOUR_LAB_PASSWORD"
SSHPASS="YOUR_LAB_PASSWORD"   # usually same as password
```

### 3) Fill in `local/hosts.csv`

You need at least the columns:
- `external-ip` (or `external_ip`)
- `function` (friendly label shown in menus)

Example:

```csv
host-url,external-ip,internal-ip,function
https://example-cm,1.2.3.4,10.0.0.1,Splunk-CM
https://example-idx1,2.3.5.157,10.0.0.2,Splunk-IDX1
```

### 4) Put files you want copied into `files-to-copy/`

Everything in `files-to-copy/` will be copied to each lab host:

Remote target:
```
/tmp/lab-speed/
```

### 5) Run GO! (option 1)

From the menu:

- **1) Copy ssh-keys, RSYNC files to lab '/tmp', Console setup**

This will:
1. Validate `hosts.csv`
2. Ensure your lab key exists in `local/`
3. Try key auth; if missing and **password SSH is allowed**, it will bootstrap the key using `sshpass`
4. rsync `files-to-copy/` to `/tmp/lab-speed/`
5. Set a clear prompt based on `function` (new shells pick it up)

---

## Dependency check (important)

**Required**
- `ssh` + `ssh-keygen` (package: `openssh-client`)
- `rsync`
- `awk` (package: `gawk` on Ubuntu)
- `sed`

**Recommended**
- `sshpass`  
  Used to automatically install your disposable lab public key **when password SSH is enabled**.

### Install dependencies (with sudo)

```bash
sudo apt-get update && sudo apt-get install -y openssh-client rsync gawk sed sshpass
```

### If you do NOT have admin rights

Ask your lab/instructor/admin to install the missing packages for you:

- `openssh-client`
- `rsync`
- `gawk`
- `sed`
- `sshpass` (recommended for full automation)

lab-speed will print a clear “missing dependencies” message and the package list to request.

---

## How key setup works

lab-speed uses a **dedicated, disposable lab SSH identity** stored inside this repo:

- Private key: `local/lab_speed_ed25519`
- Public key: `local/lab_speed_ed25519.pub`

### Automation rules

- If the lab host supports **password SSH**, lab-speed can use `sshpass` one time to append the public key into:
  ```
  ~/<username>/.ssh/authorized_keys
  ```
  Then all future actions use **key auth** (fast, no prompts).

- If password SSH is **disabled**, lab-speed cannot install keys automatically (no valid auth path).
  In that case it prints the public key and tells you to ask the lab/admin to install it.

---

## lab-connect

Menu option **2) SSH to the servers in our hosts file**

- If key auth is enabled for that host, it opens an SSH session.
- If not, it shows your public key and returns to the menu (no hanging password prompts).

---

## Troubleshooting

### “Permission denied (publickey,…)” / lab-connect closes immediately
Key isn’t installed yet for that host/user.

- If password SSH is enabled: install `sshpass` and run option **1** again.
- If password SSH is disabled: ask lab/admin to install `local/lab_speed_ed25519.pub` into `authorized_keys`.

### “MISSING: sshpass” but I can’t sudo
Ask your lab/instructor/admin to install `sshpass` (and any other missing packages).  
Without `sshpass`, the tool cannot non-interactively bootstrap keys via password.

---

## License

GPLv3 — see `LICENSE`.
