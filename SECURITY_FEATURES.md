# Security features I built into Project Scout

A complete catalog, for my portfolio, of every security control in the Project Scout system: the student project-idea
bots and the private TLDP Discord server. The source code lives at **github.com/tldpprojectscout/project-scout**; this
document records what I designed, why, and how it was verified. It makes no claim of a numeric security score,
malware-free results, or guaranteed legal or privacy safety.

---

## 1. The problem I was solving

The bot's raw material is public GitHub search, and public GitHub search returns real malicious content: wallet
drainers, credential stealers, fake "download" repositories pointing at password-protected archives, forex and
account-farming tools, and star-farmed clone networks. A code review found a wallet drainer and a deceptive Ghostfolio
clone that had already reached the catalogue. Hand an undergraduate a link and call it a project idea, and some of
them will clone and run it on their laptop.

**Design goal:** a student only ever sees repositories that have completed automated checks, and even those are meant
to be run in a disposable sandbox, not on a personal machine. Risk is minimized, not eliminated.

## 2. Threat model

- **Untrusted input.** Every repository, README, description, issue title and event listing is attacker-controlled
  data. It is treated as data, never as instructions to a scanner or an AI.
- **Adversaries.** Repo authors seeding malware or scams into search results; anyone trying to smuggle a malicious
  link or fake mention into a Discord message; outsiders trying to join the private student server; malicious
  dependencies inside an otherwise ordinary repo.
- **Assets.** Students' machines, the private roster, the bot's credentials, and the integrity of what students are
  told is "screened."

---

## 3. The single publishing gate

Discovery (finding candidates) is separated from delivery (what students see). One function, `gate.eligible`, is the
only decision every student-visible path calls. A repository is shown only if **all** hold:

1. It is not on the **quarantine** list.
2. It has a **passing screening record** whose reviewed commit SHA matches, under the current policy version.
3. The record has **not expired** (14 days for fresh repos, 30 for curated references).
4. For a fresh-repo row, it was **pushed within 30 days**.

Anything that fails, times out, is incomplete, or is unsupported is **withheld**. Checks are never silently downgraded
to make a repo pass. If nothing qualifies, the feed offers a self-contained synthetic project idea with no external
links rather than an unchecked repo. Every repo message ends with: *"Automated checks completed; not a safety
guarantee."*

## 4. Isolated, no-execution code scanning

The only stage that touches an untrusted repo. For each candidate it:

- **Pins the commit** with `git ls-remote`, then fetches and checks out **that exact SHA** (not whatever the branch
  points at later), and verifies the scanned commit matches the pinned one.
- Clones into a throwaway directory with an **empty HOME**: no credential helpers, no git hooks, no LFS smudge, no
  submodules, blobs over 2 MB skipped, non-https protocols disabled.
- **Never executes anything** in the repository: no install scripts, no build, no tests, no GitHub Actions workflows.
- Deletes any `.semgrepignore` so a hostile repo cannot hide files from the scan.
- Discards matched source text; only rule ids, paths and line numbers are kept, so a secret found in a repo is never
  copied into a report or a message.

### The four detection engines (all required)

| Engine | What it catches |
|---|---|
| **Semgrep** with a custom malware-behaviour ruleset + `p/security-audit` + `p/secrets` | Decode-then-exec, download-and-run, credential and wallet theft, webhook/Telegram exfiltration, reverse shells, persistence, antivirus tampering, miners, keyloggers, committed secrets |
| **YARA** byte-signatures | Renamed executables, obfuscated payloads, and stealer strings in any file type (what language parsers skip) |
| **ClamAV** | Known-malware signatures |
| **osv-scanner** | Malicious and vulnerable dependencies (where much real GitHub malware lives) |

**Fail-closed rule:** all four must run for a pass. A missing tool, missing signature database, timeout, engine
error, malformed output, or a code repo where zero files were scanned yields *incomplete*, never *pass*. Exit codes
are read per tool. A real detection blocks even if another engine also failed.

**Blocking thresholds (conservative, documented):** any malware-rule hit, any YARA match, any ClamAV signature, any
secrets finding, or a malicious-package advisory withholds the repo. Ordinary vulnerabilities and security-audit
findings are advisory only (vulnerable code is a learning topic, not a student-safety issue). Findings are never
suppressed to make a check pass.

## 5. Content policy, dual-use handling, and quarantine

- **Prohibited wording** (wallet drainers, stealers, phishing kits, cracked software, account farms, carding) fails any
  repo, curated or not.
- **Dual-use offensive tooling** (RATs, C2 frameworks, keyloggers, booters, exploit kits) is kept out of the general
  student feed, without treating every legitimate security tool as malware: curated, allow-listed security projects
  such as OWASP tools are not flagged.
- **Quarantine** (`quarantine.json`): the named incident repositories and a clone-name pattern can never be published,
  regardless of what any scan says.

## 6. Deep vetting of repository signals

Before the code scan, each repo is checked for the signals that identify scam and star-farm repositories: binaries or
archives in the root, README-only shells, bought stars on very young repos, brand-new owner accounts, throwaway owner
names, the same repo name under many owners (clone farms), commit history, and OpenSSF Scorecard signals. Established,
well-followed projects are exempted from the heuristics that would otherwise flag legitimate popular repos.

## 7. README link and install-instruction screening (SSRF-guarded)

Every README is screened **at the reviewed commit**. Blocking findings: file-sharing hosts, executable or archive
downloads from unrelated domains, credentials embedded in a URL, and instructions to disable antivirus, use an archive
password, run an installer as administrator, or pipe `curl` into a shell. URL shorteners warn. A README that exists but
cannot be read makes the result incomplete.

If a shortener is ever resolved, the resolver is **SSRF-guarded**: it refuses private, loopback, link-local,
carrier-grade-NAT and cloud-metadata addresses; re-validates **every redirect hop** against the resolved IP; connects to
the validated address rather than the hostname (so DNS cannot change between check and connect); caps redirects and
header size; and never downloads a body. Resolution is off by default.

## 8. Screening records and reviewed-commit integrity

Each record in `screening.json` holds the repository identity, the **reviewed commit SHA**, scan time, Semgrep version
and rule-file SHA-256, coverage (languages parsed, files scanned, which engines ran), result and reasons, and an
expiry. Integrity rules:

- Before a cached record is reused, the current default-branch SHA is re-checked; a changed commit forces a re-screen.
- An unreachable SHA check is never treated as "still valid."
- Every passing record and every published row carries a valid reviewed SHA, and student links point at that exact
  commit, not a mutable default branch.

## 9. Independent validation of the published snapshot

Before `published.json` is written, the publisher **reconstructs** it from the authoritative screening records (never
trusting screened fields carried in the build artifact's rows) and re-validates **every** repository-bearing section:
the fresh feed, org recommendations, curated starters, cyber-domain repos, case collections, and learning paths. Any
malformed record, missing engine, incomplete coverage, expired scan, missing or mismatched SHA, or quarantined repo
rejects the whole snapshot, the **previous** snapshot is preserved, and the run fails so staff are alerted.

## 10. Safe delivery: the Cloudflare Workers

The Telegram and Discord Workers are pure renderers of the screened snapshot:

- **No live search.** They make no GitHub or GitLab calls at all; the unscreened GitLab fallback was removed.
- **Fail closed.** If the snapshot is missing, degraded, stale (no screening in 36 hours), malformed, or produced under
  an incompatible policy version, every repo lane returns a pause message instead of guessing.
- **Link allowlist.** A link is rendered only for HTTPS destinations on an allowlist of known hosts; anything else is
  shown as plain escaped text.
- **Consistent escaping** of every piece of third-party text (descriptions, titles, todo items, keyword echoes) so a
  repo name can never smuggle a masked link or a fake mention.
- **Automatic mentions disabled** on every posting path (`allowed_mentions: {parse: []}`), so `@everyone` in a
  description pings nobody.
- **Message limits handled** by dropping whole rows, never cutting a link or the safety label mid-message.
- **No internal error text** reaches students.

Discord command endpoint controls: Ed25519 **signature verification** retained; the configured **guild ID** enforced;
command names, option names and enum values validated; keyword input capped at 60 characters with a safe character
set; the caller must hold the Student or Staff role (missing configuration fails closed); a **timestamp skew** check;
and best-effort **replay** and per-user **rate limiting**.

## 11. Least-privilege CI/CD

GitHub Actions is split into two jobs with a trust boundary between them:

- **screen** (untrusted): searches and clones third-party repos and runs the engines. It has `contents: read`, no git
  credential persisted, **no Discord or Telegram secrets**, and only a no-permission machine-account search token. It
  cannot post to students or write to the repository.
- **publish** (trusted): downloads the screen artifact, re-validates offline, sends, and commits state. It never
  clones or executes anything.

Every third-party Action is pinned to a reviewed **commit SHA**; `workflow_dispatch` input is a fixed choice passed
only through an environment variable, never interpolated into a shell line. A separate `security.yml` workflow scans
the project's own code with Semgrep, osv-scanner and gitleaks on every push.

## 12. Student-side containment: sandboxes

Because no scanner catches everything, the most reliable control is that students never run untrusted code on their
own machines. I built a **hardened dev-container template** (non-root user, capped CPU and memory,
`no-new-privileges`) for GitHub Codespaces or local Docker/Podman, and a **Google Colab starter notebook** that walks
through clone, read, run, and reset. In the Discord server the "set up your sandbox" step is deliberately the first
thing a student does. The rules taught alongside it: public or synthetic data only, never put a secret or `.env` in
a sandbox, treat it as disposable, and a passed check is not proof code is safe to run.

## 13. Student identity and privacy

- **Name matching removed.** Typing a name is not proof of identity, so `/verify` no longer grants access by name and
  exposes no roster oracle.
- **Roster-gated enrollment.** The staff roster is a distribution list for **unique, single-use, expiring** Discord
  invites (unpredictable, replay-proof, time-boxed); staff grant the Student role to those who join.
- **Data minimization.** The roster lives only in a gitignored local file and is never printed, logged, committed, or
  sent to a service. Verification responses are private (ephemeral).
- **No insecure fallback.** If enrollment configuration is missing, the system fails closed rather than reverting to
  name-only verification.

## 14. Discord server security

- All learning content **gated behind the Student role**; only START HERE is visible to newcomers.
- **Verified email required** before posting (verification level Medium).
- **Invite creation restricted to staff**; `@everyone` cannot invite.
- **Least-privilege roles**: staff and students separated; the automation bot sits below staff and holds only the
  permissions it needs, documented separately as runtime versus one-time setup permissions.
- The entire server is **provisioned from code**, so its security configuration is reviewable and reproducible.

## 15. The four high-priority review findings I addressed

1. **Required scanners fail closed** (missing tool, database, timeout, malformed output, or zero coverage never pass).
2. **Reviewed-commit integrity** (exact-SHA fetch, same-commit vetting, SHA re-check before reuse, commit-pinned links).
3. **README link screening enforced inside the gate**, at the reviewed commit.
4. **Independent, section-by-section validation** of the published snapshot with previous-snapshot preservation.

## 16. Verification

- **84 Python tests and 19 Cloudflare Worker tests** pass, covering quarantine, expired and changed-SHA records,
  freshness boundaries, engine failures (missing, error, timeout, malformed), suspicious links and unsafe install
  instructions, SSRF refusals and redirect re-validation, mention and masked-link injection, wrong-guild and
  impersonation attempts, artifact tampering, and index-only publication.
- **Semgrep** (security-audit, secrets, Python, JavaScript, GitHub Actions rulesets) reports **zero error-level
  findings** in the project's own code.
- **Real end-to-end screens** of live repositories produced valid records with genuine reviewed SHAs and engine
  coverage; quarantined repositories were failed without being scanned.
- **Not verified locally:** ClamAV and osv-scanner run only in CI on GitHub and were unit-tested with mocked failures
  rather than exercised end to end on this machine.

## 17. Honest limits

Heuristics miss malicious projects and reject legitimate ones. Static analysis, signature scanning and link checks are
all evadable. A repository can change after it was screened (records expire in 14 days). Worker replay and rate-limit
state is best-effort per isolate. This system is built for a **supervised** student pilot with staff in the loop. It
reduces risk substantially; it does not remove it, and the student sandbox is what makes the residual risk acceptable.
