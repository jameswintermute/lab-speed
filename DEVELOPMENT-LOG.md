# lab-speed — future development log

Working notes for ideas that have been discussed and designed but
not yet shipped. When work resumes, start here.

---

## Sudoless setup (proposed option `s`)

### Origin
During v1.11.4 testing in May 2026, the user reported that the lab
images prompt for a sudo password on every command — e.g. `sudo -iu splunk`
asks for the password, and you hit that prompt many times across a four-hour
lab. The Show password option (`p`) makes it slightly easier (paste from
clipboard) but is still friction on every escalation.

### Goal
Eliminate sudo password prompts on lab hosts for the duration of the lab
session, in a single menu action that runs in parallel across all hosts.

### Proposed mechanism
Install a sudoers drop-in at `/etc/sudoers.d/lab-speed-nopasswd` containing:
```
<username> ALL=(ALL) NOPASSWD: ALL
```
After installation, all subsequent sudo invocations by the student user
succeed without a prompt. Reversal is a one-liner:
```
sudo rm /etc/sudoers.d/lab-speed-nopasswd
```

### Implementation design (drafted but not coded)
- New menu option `s) Sudoless setup` in the **keys** section, adjacent to
  the other auth-related options (6 Install SSH keys, m Mesh SSH keys, p
  Show password).
- Confirms with the user explicitly before doing anything. Warning text
  along the lines of:
    "This is a deliberate loosening of host security. Only run this on
     short-lived training labs that will be torn down."
- Runs in parallel across all hosts (like option 6 / m / 8 / r).
- Per-host remote script:
  1. Write the sudoers content to a temp file owned by the student.
  2. Pipe the known SSHPASS to `sudo -S` to call `visudo -c -f <tmpfile>`
     to validate. If validation fails, the host is reported failed and the
     live /etc/sudoers.d/ is not touched.
  3. If valid, `sudo install -m 0440 -o root -g root <tmpfile>
     /etc/sudoers.d/lab-speed-nopasswd`.
  4. Remove the temp file.
- The mode 0440 is what sudo requires for files in /etc/sudoers.d/.

### Why use `sudo -S` instead of just typing the password
The script already exports SSHPASS for sshpass. Piping it through stdin to
`sudo -S` lets us run the entire install non-interactively — one menu click,
no per-host typing. The user only types their password during the original
Import (option `i`).

### Trade-offs and concerns
1. **Security posture.** This meaningfully reduces the security of the host.
   It is only acceptable in the context of throwaway training labs. The
   warning text must make this unambiguous, and the option should be
   labelled clearly so it can't be triggered accidentally.

2. **The first sudo invocation still needs the password.** No way around
   that. But once SSHPASS is exported, the remote script uses `sudo -S`
   with stdin, so the user doesn't see a prompt during install.

3. **visudo validation is essential.** A bad sudoers file can lock you out
   of sudo entirely. Writing to a separate file in `/etc/sudoers.d/`
   isolates the blast radius (the main /etc/sudoers is untouched) and
   `visudo -c` catches syntax errors before they hit the live config.

4. **Cleanup interaction.** Option 9 (Clean up) currently resets local
   lab-speed state but does not undo sudoless setup on remote hosts. That's
   fine: the lab itself is torn down at the end of the session. Mentioning
   this in the README troubleshooting section is enough.

### Why we deferred
The user asked to pause development at this point in May 2026 because the
existing v1.11.4 is in a known-good state and they may not need the next
lab for a while. Significant new functionality risks introducing bugs that
won't be discoverable until the next lab session, and other users may pull
the repo in the meantime. Better to ship sudoless setup as a coherent v1.12
release when there's an actual lab to test against.

### Resume checklist when work restarts
- [ ] Drop function `sudoless_setup` into the script (after `install_ssh_keys`).
- [ ] Add `require_lab_session` guard at the top.
- [ ] Use the existing `read_hosts_full` helper for the host list.
- [ ] Use the existing parallel-execution pattern (see `install_ssh_keys`
      or `health_check` for a template — tmpdir, per-host result files,
      progress bar, wait for pids).
- [ ] Wire `s|S) sudoless_setup` into the menu `case`.
- [ ] Add the menu line: `printf "  %s\n" "s) Sudoless setup   (no more
      'sudo' password prompts during the lab)"`.
- [ ] Update the README — new section explaining the feature, the warning,
      and the reversal command. Add it to the menu overview table.
- [ ] Bump version to v1.12.0 (this is a feature release, not a patch).
- [ ] Test against a real lab — verify visudo validation works, verify
      reversal works, verify it survives `sudo -iu splunk`.

---

## Other ideas mentioned in earlier conversations

These came up during design discussions and were not built. Listed here so
they don't get lost.

### Auto-detect lab cut-off time
The lab clock currently has hard-coded warn/limit thresholds (with override
env vars). Some lab portals show the actual cut-off time on screen. Could
the user paste that into the Import flow, and have lab-speed compute the
real countdown? Probably not worth it — the override env vars cover the
case where 4h is wrong.

### Resume mode after Clean up
If the user runs Clean up by mistake, the Import backup `.bak-<timestamp>`
files contain everything needed to restore. A `--restore` flag or menu
option could find the most recent backup and re-instate it. Low priority
since the user can do this manually with `cp local/credentials.txt.bak-X
local/credentials.txt`.

### Web UI mode
A localhost web interface that shows live host status, lab clock, and
provides the same actions as the menu. Significant work, probably not
worth it for a tool that does its job in a terminal.
