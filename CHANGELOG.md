# Changelog

All notable changes to this project are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/); versions follow [SemVer](https://semver.org/).

## [1.2.0] - 2026-07-04

### Added
- **Live testing (localhost) lens** ([references/live-testing.md](application-auditor/references/live-testing.md)): when a local instance is reachable, the audit drives the running product to prove findings — every endpoint hit, auth matrix replayed across real sessions, IDOR by id-swap, bounded rate-limit and concurrency-race probes, hostile file uploads, security-header and error-leak inspection. Findings are tagged `runtime-confirmed` / `static-inference` / `runtime-cleared`, and the reproducing request is the evidence. Target via auto-detected dev ports or `.audit/target.json`.
- **Hard safety rails for runtime testing**: localhost/private hosts only with no override, production never a target, dev-instance confirmation before any write, disposable `zzaudit-`-marked test data to the local dev DB only, a `.audit/live-writes.log` cleanup trail, and no exfiltration of captured data.
- **Mechanical-tools pass before the reading passes**: `gitleaks`/`trufflehog` over the tree *and git history*, ecosystem CVE audits, `semgrep`, full `tsc`/lint. Tool hits are candidates still verified against real code.
- **Expanded attack-class coverage**: SSRF, path traversal, unsafe deserialization / prototype pollution, XXE, open redirects, crypto misuse (fast password hashes, JWT `alg:none`, non-constant-time compares, `Math.random` tokens), mass assignment, and business-logic/money bugs — across audit-angles, taxonomy, and PROMPT.md.
- **Disk-persisted run state** in `.audit/` (`inventory.md`, `findings.jsonl`, `ledger.md`, `live-writes.log`) so long, compaction-prone runs survive and can resume; and the report is written to `audit-report.md`, not just printed.
- **Inventory × lens coverage matrix**: convergence now requires full coverage, not just two quiet passes.
- **Blind verifier**: verification subagents receive only the finding + code access, never the finder's reasoning, removing confirmation anchoring.

### Changed
- **Explicit read-only guarantee**: audit mode never edits source; only `fix` mode does, and only after you ask — now on a fresh branch committed wave by wave.
- CRITICAL/HIGH rows now carry an `evidence:` clause (reproducing request, traced code path, or flagging tool).
- CRITICAL-dense lenses run first so a truncated run still surfaces the worst.
- `.audit-ignore` matching is path + issue-tag + content, not a bare line number that goes stale on the first edit.

## [1.1.0] - 2026-06-25

### Changed
- Reframed per-finding checking from "refute/attack the finding" to evidence-grounded verification (confirm the location, trace the guard, run the test that settles it). Removes a confirmation-bias framing that can suppress real findings; adds a tie-break rule so a located, uncleared security risk is reported, not dropped.
- Subagent verification now runs in a fresh context (and a different model where available), to confirm or clear a finding rather than argue it away.
- Stop condition is now an explicit capped heuristic: every lens runs at least once plus a ceiling of further passes; convergence is described as "looked hard enough", not a proof.

### Added
- `.audit-ignore` support: runs skip user-confirmed false positives listed at the repo root (read-only; the audit never writes to it).
- Skill frontmatter: `when_to_use` and `argument-hint` fields.
- Plugin packaging: `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` so the skill installs via `/plugin marketplace add arjuncirakas/application-auditor-claude-skill` then `/plugin install application-auditor@arjuncirakas`.
- README: honest comparison table vs RepoLens, audit-loop, and `/security-review`; a Limitations section; re-led positioning on the two real differentiators (claim-vs-code and verification-by-default).

### Note
Multi-lens auditing and convergence are prior art (RepoLens, CheckLoop, audit-loop); this release stops implying otherwise and leads with what is actually differentiated.

## [1.0.0] - 2026-06

- Initial release: 24-lens convergence audit, 18 defect classes, fix mode with verification loop, single-file `PROMPT.md` for non-Claude-Code agents, anonymized sample report.
