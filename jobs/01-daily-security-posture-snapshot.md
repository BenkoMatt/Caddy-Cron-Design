# 01 — Daily Security Posture Snapshot

**Status:** draft
**Created:** 2026-06-09
**Last revised:** 2026-06-09
**Trigger:** Hermes cron (job ID TBD on creation)

## Purpose

A read-only daily check of the host's external security posture:
what is listening, what is exposed, what changed since yesterday, and
whether the agent itself is healthy. Produces a single sanitized
markdown report that is published to a public GitHub repo for
**public accountability** — anyone can see that the host is hardened
and staying hardened over time.

This is **observability**, not a security audit. The job does not
detect attacks, does not block traffic, does not modify configuration.
It surfaces state.

## Schedule

`0 8 * * *` (08:00 daily, system timezone — currently
`America/New_York` / EDT).

## Inputs

The host's own system state. The job runs entirely locally and
requires no network inputs. Specifically:

- `ss -tln` — listening TCP sockets
- `df -h /` — disk usage on the root filesystem
- `systemctl list-units --type=service --state=running` — active services count
- `apt list --upgradable` — pending package updates
- `tailscale status` — Tailscale peer status (top 10 entries)
- `hermes --version` — agent version string

## Outputs

### Local artifacts (host, ephemeral)

- `~/.hermes/state/caddy-posture-snapshot.log` — full unredacted
  log of the 6 checks. Excluded from any public artifact.
- `~/.hermes/state/caddy-posture-snapshot.json` — 7-field JSON
  snapshot (date, public_listeners, ssh_tailscale_only, disk_used,
  upgradable_packages, public_report path, log path). Excluded from
  any public artifact.

Both files live under `~/.hermes/state/` (not `/tmp`) so they
survive reboots. They are rotated / pruned by the system (not this
job); if a long retention window is needed, the operator can add a
maintenance cron job in the future.

### Public artifact

A single markdown file per day pushed to the public repo
`github.com/BenkoMatt/VPS-Daily-Posture`, branch `main`, at path
`reports/YYYY-MM-DD.md`. The file contains:

- A 1-line "Generated" timestamp
- A 1-line "Agent" attribution
- A 4-row "Posture Summary" table (listeners, SSH on Tailscale, disk,
  pending updates)
- A "Security Audit" section that honestly states the audit status
  (see *Failure modes*).
- A "Notes" section linking to the upstream hardening guide

The job also commits and pushes the file with message
`Daily security report: YYYY-MM-DD`.

## Privacy posture

The public artifact is sanitized. Specifically, the job:

- **Excludes** the host's Tailscale IP and IPv6 prefix from any
  public output. Listener filtering uses `tailscale ip -4` to discover
  the address at runtime rather than a hardcoded value, so the
  report stays correct if the CGNAT IP changes.
- **Excludes** internal paths, usernames, and any other host-specific
  identifiers.
- **Includes** only aggregate counts (listener count, service count,
  upgradable package count, disk percentage) — never the full
  `ss -tln` output or `systemctl` listing.
- **Attribution** is "Caddy (Hermes — self-hosted)" — does not name
  the operator, the host, or the provider.

## Delivery

`deliver: "local"` (Hermes cron). The job runs entirely as a script
(no LLM), produces its own stdout summary, and the stdout is saved
to `~/.hermes/cron/output/<job-id>/` for post-hoc inspection.

**Why not `deliver: "origin"`:** the operator's `hermes setup` does
not currently have a messaging platform (Telegram, Discord, Slack,
etc.) wired. `deliver="origin"` would silently fail with "no
delivery target resolved" and the operator would never see the
result. Using `local` makes the silent-success-on-no-platform
behavior explicit: the work happens, the report lands on GitHub, the
log lands on disk — that's all the delivery surface we have right
now. If a platform is wired later, this field can be changed to
`origin` in a one-line update.

## Failure modes / known issues

- **Hardcoded dependency on `apt`** — the package-updates check will
  produce "0" on non-Debian-family systems. For this host (Ubuntu
  24.04) it is correct.
- **Tailscale filtering uses `tailscale ip -4`** — if `tailscale`
  is not on PATH, the filtering falls back to "no exclusion," which
  would over-count public listeners. Mitigated by the fact that
  Tailscale is the only consumer of the CGNAT interface and the
  fall-through case would obviously include a Tailscale listener,
  making the error self-evident in the log.
- **Security Audit section is currently an honest placeholder** —
  the job does not run an integrated audit tool. The "Security
  Audit" section in the public report says *"Not run — no
  integrated audit tool on this host"* rather than echoing
  `0/0/0` (which would be a misleading lie). The section is the
  integration point if/when a real audit tool is configured. The
  design is honest about this.
- **No diff against previous day** — the report shows today's state
  only. Comparing two consecutive days' reports is left to the
  reader. A "diff vs. yesterday" section could be added in a future
  revision if the operator wants trend visibility.
- **Single timezone** — the schedule is in the host's system
  timezone. If the host's timezone changes, the schedule's wall-clock
  meaning changes too. Documented in the operator's
  `hermes-cron-jobs` skill.

## Implementation status

- **2026-06-09:** Script built, tested locally, real-world run
  against the live host completed. First report published to
  `github.com/BenkoMatt/VPS-Daily-Posture` at commit `df9a03f`,
  `reports/2026-06-09.md`. End-to-end push verified.
- **2026-06-09:** First run surfaced a real finding: 1 non-local
  TCP listener on `*:4369` (Erlang Port Mapper / `epmd`) — bound
  to all interfaces, not just the Tailscale interface. This
  contradicts the design's "expected: 0 public-facing listeners"
  assumption. Investigated separately, not blocking this job.
- **2026-06-09:** `hermes --version` reported 95 commits behind
  upstream (was 4 commits behind on 2026-06-08). The version
  string is unchanged (`v0.16.0`); the commit count behind
  indicates an upstream refspec shift. Also investigated
  separately, not blocking this job.
- **Status:** ✅ ready for production scheduling.
