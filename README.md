# Project Scout — build record

A record, for my own reference, of what Project Scout is and what I built. The source code lives in the project's
machine account: **https://github.com/tldpprojectscout/project-scout** (transferred there from this account so the bot
owns its own repository and low-privilege token). This repo is documentation only.

## What it is

Project Scout is a bot system that finds GitHub project ideas, open-source contribution opportunities and research code
for undergraduate students across seven majors (Quant, Finance/FinTech, Software Engineering, Cybersecurity, Data
Analytics, Project Management, Digital Marketing), and delivers them through Telegram and a private Discord server for
the Baruch TLDP cohort. It runs on GitHub Actions and Cloudflare Workers.

## What it does

- **Feed:** every six hours it searches GitHub, screens each candidate, and posts fresh repositories per major into
  Telegram and per-major Discord forums. Only new items are sent.
- **On demand:** students run `/scout` in Discord (and slash-style commands in Telegram) to pull repos, open
  good-first-issues, research code, mission-driven org repos, NYC hackathons, and case-study collections.
- **Learning structure:** four-stage learning paths per major, course-aligned searches (Baruch MFE, FIN, PM tools),
  the 8 CISSP domains for cybersecurity, and difficulty levels sorted for undergraduates (Beginner by default, with a
  clear opt-in to Intermediate or an Undergraduate Challenge).
- **Private cohort server:** a gated Discord for the 45 TLDP students with per-major channels, a spring hackathon
  feed, a capstone channel, a case-studies library, interview code-review practice, and a "set up your sandbox first"
  onboarding step.

## Architecture

- **Discovery vs. delivery are separated.** A discovery stage searches and screens candidates in an isolated
  environment; a publish stage re-validates everything offline and writes a screened snapshot (`published.json`).
- **The Cloudflare Workers render only that snapshot.** They make no live GitHub calls and pause (fail closed) if
  screening is stale, degraded, or missing.
- **GitHub Actions** run the discovery and publish jobs with least privilege: the untrusted scanning job has no
  Discord/Telegram secrets and no write credentials; a separate job publishes and commits state.

## Security engineering (the part I care about most)

Public GitHub search returns real malicious content — wallet drainers, credential stealers, fake "download"
repositories. The whole point of the security work is that a student only ever sees repositories that have completed
automated checks, and even those are meant to be run in a disposable sandbox, not on a laptop.

- **One publishing gate.** Every student-visible path goes through a single eligibility check: not quarantined, a
  current passing screening record with a valid reviewed commit SHA, adequate scan coverage, and (for fresh repos)
  pushed within 30 days. Anything that fails, times out, is incomplete, or is unsupported is withheld.
- **Isolated, no-execution scanning.** Each repo is cloned at a pinned commit into a throwaway environment with no
  tokens and no git hooks; nothing in the repo is ever executed. Four engines run: Semgrep (a custom malware-behaviour
  ruleset plus security-audit and secrets), YARA byte-signatures, ClamAV, and osv-scanner for malicious dependencies.
  A hit blocks; a missing or failed engine makes the result incomplete, never a pass.
- **README link and install-instruction screening**, with an SSRF-guarded resolver that refuses private, loopback,
  link-local and cloud-metadata addresses and re-validates every redirect hop.
- **Quarantine** of named incident repositories and clone-name patterns.
- **Student-side containment:** a hardened dev-container template and a Colab notebook so students run untrusted code
  off their own machines.
- **Roster-gated enrollment** using unique single-use invites, with no self-typed-name oracle and no roster exposure.

### Review findings I addressed (HIGH)

1. Required scanners fail closed — a missing tool, missing signature database, timeout, malformed output, or zero files
   scanned yields "incomplete," never "pass."
2. Reviewed-commit integrity — the scanner fetches and checks out the exact pinned commit, the deep vet and link screen
   read that same commit, cached records are re-checked against the current SHA before reuse, and student links point
   at the reviewed commit.
3. README link screening is enforced in the gate, at the reviewed commit.
4. The published snapshot is reconstructed from the authoritative screening records and independently validated section
   by section before it is written; on any failure the previous snapshot is preserved.

## Verification

- 84 Python tests and 19 Cloudflare Worker tests pass; Semgrep reports no ERROR findings in the project's own code.
- ClamAV and osv-scanner run only in CI on GitHub; they are wired and unit-tested with mocked failures rather than
  exercised end to end locally.

## Honest limits

Heuristics miss malicious projects and reject legitimate ones. Static analysis, signature scanning and link checks are
all evadable. This is built for a supervised student pilot with staff in the loop; it reduces risk substantially, but
it makes no promise of a numeric security score, malware-free recommendations, or guaranteed legal or privacy safety.
The student sandbox is what makes the residual risk acceptable.

## Where things live

- **Code:** https://github.com/tldpprojectscout/project-scout (the bot's machine account).
- **Case-study library:** https://github.com/tldpprojectscout/tldp-case-studies.
- **This repo:** a documentation record kept under my personal account.

## The Discord server

See **[DISCORD_SERVER.md](DISCORD_SERVER.md)** for a documented walkthrough (with screenshots) of the private TLDP Discord server I designed and built from code.
