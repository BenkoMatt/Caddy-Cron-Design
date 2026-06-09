# 02 — Daily VPS Config Snapshot

**Status:** draft
**Created:** 2026-06-09
**Last revised:** 2026-06-09
**Trigger:** Hermes cron (job ID TBD on creation)

## Purpose

A daily, **secrets-excluded** snapshot of the host's operating
configuration — files that determine how the system behaves
(`/etc/`, systemd units, firewall rules, SSH config) — pushed to a
**private** GitHub repo for off-host history. The purpose is
**post-compromise audit and rebuild** — if the host is ever
compromised or wiped, the operator has a daily record of what the
configuration *was* on each prior day, can diff any two days, and
can restore or audit without depending on the live disk.

This is **preservation**, not backup in the disaster-recovery sense.
The snapshot is *intentionally narrow* (config only, not data) and
*intentionally sanitized* (no secrets, ever). It is the smallest
artifact that still answers the question "what was the host
configured to do on date X?"

## Schedule

`0 9 * * *` (09:00 daily, system timezone — currently
`America/New_York` / EDT). Runs 1 hour after the security posture
snapshot (job 01) so the two daily jobs don't compete for
git/network resources.

## Inputs

The host's own filesystem. The job runs entirely locally and
requires no network inputs other than the push to the private
GitHub repo. Specifically:

- `/etc/` — system configuration directory
- `/etc/ssh/sshd_config` — SSH daemon config (read-only check; the
  authorized_keys file is **excluded**, see *Privacy posture*)
- `/etc/ufw/` (or `ufw status` output) — firewall rules
- `/etc/systemd/system/` — custom systemd units
- `/etc/cron.d/`, `/etc/crontab` — system cron (the Hermes-managed
  jobs in `~/.hermes/cron/jobs.json` are out of scope; they are
  version-controlled by Hermes itself)

## Outputs

### Local artifact (host, retained)

- `~/.hermes/state/vps-config-snapshot-<DATE>.tar.gz` — a gzipped
  tarball of the sanitized config tree, retained on disk for 7
  days (old files pruned by the job itself). This is the source of
  truth for the job; the GitHub push is a copy.

### Private artifact (off-host)

A single commit per day pushed to the private repo
`github.com/BenkoMatt/VPS-Config-Snapshots`, branch `main`, containing:

- `snapshots/<DATE>/etc/...` — the sanitized `/etc/` tree
- `snapshots/<DATE>/systemd/...` — the systemd unit files
- `snapshots/<DATE>/firewall.txt` — `ufw status verbose` output
- `snapshots/<DATE>/sshd_config` — sanitized SSH daemon config
- `snapshots/<DATE>/MANIFEST.txt` — list of files captured, with
  sizes and SHA-256 hashes
- `snapshots/<DATE>/sanitization-log.txt` — record of every file
  that was redacted or excluded (so the operator can audit what
  was kept out)

The job also commits and pushes with message
`Config snapshot: YYYY-MM-DD`.

## Privacy posture

This is the **highest-privacy** job in the operator's set. The
artifact is private, not public, and the sanitization must be
defense-in-depth: mistakes here could leak credentials. Specifically:

**Excluded paths and patterns** (the job will not snapshot these,
even if matched by the input list):

- `/etc/ssh/sshd_*` key files
- `/etc/ssh/host_*` key files
- `/etc/letsencrypt/archive/`, `/etc/letsencrypt/live/`, `/etc/letsencrypt/renewal-hooks/` — certificates and private keys
- Any file matching `*.pem`, `*.key`, `*.p12`, `id_*`, `authorized_keys`, `known_hosts`
- `/etc/shadow`, `/etc/gshadow`, `/etc/sudoers.d/*` containing user lists
- `/root/.ssh/`, `/root/.gnupg/`, `/root/.config/` — any operator credential store
- `/etc/cron.d/*` lines containing `password=` or other embedded secrets
- Any file whose first 1KB contains the regex
  `(BEGIN [A-Z ]*PRIVATE KEY|BEGIN OPENSSH PRIVATE KEY|api[_-]?key|secret[_-]?key|aws_access_key_id)`

**Redacted-in-place** (the file is included but sensitive fields
are replaced with `<REDACTED>` before commit):

- `/etc/ssh/sshd_config` — `AuthorizedKeysFile`, `HostKey` lines
  are redacted
- `/etc/cron.d/*` — any `USER=` line containing what looks like a
  token
- `/etc/systemd/system/*.service` — `Environment=` and
  `EnvironmentFile=` lines are redacted
- `/etc/environment` — fully redacted, value-by-value

**Sanitization log** is mandatory: every redaction and exclusion
is recorded in `snapshots/<DATE>/sanitization-log.txt` so the
operator can verify nothing was missed. If the sanitization step
fails for any reason, the snapshot is **not pushed** — only the
local tarball is kept, and the job exits with code 1.

## Delivery

`deliver: "local"` (Hermes cron). Same rationale as job 01: no
messaging platform is wired, and `local` makes the silent
delivery explicit. The local tarball and the `sanitization-log.txt`
in the next run give the operator a way to verify the job worked.

## Failure modes / known issues

- **Sanitization is the highest-risk part of this job.** A bug in
  the exclusion regex could leak a secret to a private (but
  internet-reachable) GitHub repo. Mitigations:
  - Unit tests for the sanitization logic run as part of the
    job's pre-flight check.
  - `sanitization-log.txt` is human-readable and reviewed by
    the operator on a sampling basis.
  - The `deliver="local"` setting means the push result is
    inspectable on disk; the operator can `cd` to the
    config-snapshot working tree and `git diff` before merge if
    desired.
- **Disk retention of 7 days is a local policy** — the job prunes
  its own `~/.hermes/state/vps-config-snapshot-*.tar.gz` files
  older than 7 days. GitHub retains the full history indefinitely.
  If the operator wants shorter or longer retention, this is a
  one-line change in the job.
- **Hermes-managed cron state is not snapshotted** by this job,
  by design. The Hermes cron registry is
  `~/.hermes/cron/jobs.json`, which is part of the agent's
  identity and is restored from the meta-repo
  `BenkoMatt/Caddy-Cron-Design` and the operator's memory
  entries, not from the config snapshot. This is a deliberate
  separation of concerns.
- **`ufw status verbose` output may contain IPs** that were
  configured as allow-rules. These are scrubbed (the IP octets
  are replaced with `<IP>`) before commit, but the scrub is
  per-line, so a misformatted rule could slip through. The
  sanitization log records the line numbers that were scrubbed,
  making it auditable.
- **Systemd unit `ExecStart=` lines that reference a credential
  file path** are not currently detected as credentials. If the
  operator adds a service in the future that puts secrets on
  the command line, the sanitization would not catch it. This
  is a known limitation; the operator is responsible for
  reviewing new unit files.

## Replaces

This is a **new** job. It does not supersede any existing cron
entry, but it is a more disciplined successor to the
**intent** behind the old `daily-backup.sh` / `caddy-public-sync.sh`
job, which conflated config-preservation with public-memory-publishing
into a single daily report. Those jobs are paused on 2026-06-09 in
preparation for retirement; their `daily-backup` intent is replaced
by *this* job, while their public-memory-publishing intent is
*dropped* (see job 01 for that side of the change).

## Repo history note

The destination repo `BenkoMatt/VPS-Config-Snapshots` was created
private on 2026-06-09. During agent setup, **two test commits
authored by Caddy** were pushed to the repo before the first real
job run:

1. `fa1ba09 test: verify Caddy can write to VPS-Config-Snapshots`
   — pushed as a write-permission check; the SSH `ls-remote`
   returned empty for the empty private repo, which was
   misinterpreted as a write-permission failure, leading to a
   no-op empty commit push to verify access.
2. `519fef1 temp: place new default branch` — pushed as part of
   an attempted cleanup of the first commit; the cleanup was
   abandoned when the second commit itself was determined to
   also be undesirable, and the operator chose to leave both
   in place rather than continue iterating.

**Both commits are empty (no tree changes), authored by Caddy
under the operator's GitHub identity, and contain no sensitive
content.** They are the parent commits of the first real config
snapshot, and will appear in `git log` ahead of every legitimate
snapshot. They are documented here for transparency; the operator
chose Option B ("Leave both my bad commits in place") over
deleting the repo and recreating it, accepting the test commits
as the historical root of the repo.

**Lesson recorded** in the
`caddy-cron-design-publish` skill (and its parent `hermes-cron-jobs`
skill): an SSH `ls-remote` returning empty is **not** a reliable
indicator that a write will fail. For private repos in particular,
the correct pre-write check is a `git push --dry-run` or an
explicit operator confirmation, not a no-op push that becomes
a real commit.
