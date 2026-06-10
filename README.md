# Caddy Cron Design

A public, dated design record for the scheduled jobs that run on the
operator's production host. Written by Caddy (Hermes Agent) on behalf
of the operator.

## Project status

**2026-06-10:** Initial renewal **shipped and verified end-to-end.**

- **2 active jobs:** `Caddy Posture Snapshot` (`0 8 * * *`) and
  `Caddy Config Snapshot` (`0 9 * * *`). Both have run clean on every
  scheduled attempt since creation. See each job's
  `## Implementation status` for per-run evidence.
- **7 legacy jobs removed** on 2026-06-10. Pre-removal backup of
  `~/.hermes/cron/jobs.json` was saved on the host. No new jobs have
  been created since.
- **No new design changes** are pending. Three findings from the first
  real runs (`epmd` listener on `*:4369`, Hermes 95-commits-behind
  drift, over-aggressive redaction in `*.service` files) are documented
  under each job's `## Implementation status` and are tracked as
  follow-ups, not blocking changes to this repo.

Future cron work (new jobs, modifications, retirements) should follow
the same workflow: design in this repo, get sign-off, build the
script, wire to Hermes cron, watch one cycle, update the design.
The `caddy-cron-design-publish` skill (on the host) has the full
workflow.

## What this is

Every scheduled job on the operator's host is described here *before*
it is implemented, and the description is updated whenever the job
changes. This is the *design* record — it documents **why** a job
exists, **what** it does, and **how** it is meant to behave. It is
distinct from the job's *output* (which lives in other repos, see
each job's "Outputs" section).

The format of each entry, the privacy rules, and the standing
operator rule that drives this repo are described in the
[`hermes-cron-design-publish`](https://github.com/...)
agent skill — see the operator's `~/.hermes/skills/devops/caddy-cron-design-publish/`
on the host for the full workflow.

## What this is *not*

- **Not a tutorial.** It documents *one* host's job set, not how to
  design cron jobs in general.
- **Not the job output.** Each job's actual reports (posture data,
  config snapshots, etc.) are published elsewhere. This repo
  *describes* those publications; it does not contain them.
- **Not a runtime dashboard.** For current state of a job, ssh to
  the host and run `hermes cron list` (or `crontab -l` for legacy
  system cron entries).

## File layout

```
README.md                                          (this file)
.jobs/                                              (design descriptions, prefixed by creation order)
├── 01-daily-security-posture-snapshot.md
├── 02-daily-vps-config-snapshot.md
└── ...future jobs...
```

Numbered prefix is **creation order**, not priority. The
`NN-` prefix is preserved across edits — even if a job is later
moved, renamed, or retired, the original number stays with the file
so historical references (e.g. "see job 01") continue to resolve.

When a job is **retired**, its file is moved to
`.jobs/retired/NN-<name>.md` and a one-line `RETIRED: YYYY-MM-DD —
<reason>` header is prepended. The file is **not deleted**, so the
design history is preserved.

## Privacy commitment

This repository is **public by design**. The following classes of
information are **never** written here:

- Hostnames, IP addresses, internal network layout, VPN topology
- Usernames, SSH key fingerprints, authentication material of any kind
- Install paths on the operator's host
- Provider names, model names, anything that fingerprints the deployment
- The *content* of any job's output artifacts (this is a design record,
  not a data record)

Public-safe content (and what SHOULD be included):

- The job's name, purpose, schedule, and trigger
- General tool categories (e.g. "uses `ss` for listener enumeration")
- The destination repos / paths a job publishes to
- Privacy posture of the published artifact (what is sanitized, what is not)
- Failure modes the design accounts for
- Cross-references to PR / commit hashes in upstream projects the job
  exercises (these are already public on GitHub)

This mirrors the privacy posture of the
[`BenkoMatt/Hermes-Update-Record`](https://github.com/BenkoMatt/Hermes-Update-Record)
repo, which documents upstream Hermes Agent upgrade history.

## Format of a job description

```markdown
# NN — <job name>

**Status:** draft | active | paused | retired
**Created:** YYYY-MM-DD
**Last revised:** YYYY-MM-DD
**Trigger:** <where the schedule lives>

## Purpose
<one or two sentences on what this job is for, and why it exists>

## Schedule
<cron expression with timezone noted, or natural-language schedule>

## Inputs
<what the job reads from — files, env, network endpoints>

## Outputs
<what the job writes/publishes, including destination repos>

## Privacy posture
<what is and is not in any public artifact; what is sanitized; what is excluded>

## Delivery
<where the job's stdout/deliver target sends results, and what happens
when no messaging platform is wired>

## Failure modes / known issues
<bugs, edge cases, hardcoded values that could break, recovery procedure>

## Replaces
<cross-reference to any previous job this one supersedes; "none" if new>
```

## Author

Caddy — Hermes Agent running on behalf of the operator of this repo.
Commits are made under the operator's GitHub identity with their
explicit consent. The agent and the operator are not the same entity.
