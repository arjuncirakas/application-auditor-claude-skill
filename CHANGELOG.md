# Changelog

All notable changes to this project are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/); versions follow [SemVer](https://semver.org/).

## [1.1.0] - 2026-06-25

### Changed
- Reframed per-finding checking from "refute/attack the finding" to evidence-grounded verification (confirm the location, trace the guard, run the test that settles it). Removes a confirmation-bias framing that can suppress real findings; adds a tie-break rule so a located, uncleared security risk is reported, not dropped.
- Subagent verification now runs in a fresh context (and a different model where available), to confirm or clear a finding rather than argue it away.
- Stop condition is now an explicit capped heuristic: every lens runs at least once plus a ceiling of further passes; convergence is described as "looked hard enough", not a proof.

### Added
- `.audit-ignore` support: runs skip user-confirmed false positives listed at the repo root (read-only; the audit never writes to it).
- Skill frontmatter: `when_to_use` and `argument-hint` fields.
- Plugin packaging: `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` so the skill installs via `/plugin marketplace add apoorvjain25/production-audit` then `/plugin install production-audit@apoorvjain25`.
- README: honest comparison table vs RepoLens, audit-loop, and `/security-review`; a Limitations section; re-led positioning on the two real differentiators (claim-vs-code and verification-by-default).

### Note
Multi-lens auditing and convergence are prior art (RepoLens, CheckLoop, audit-loop); this release stops implying otherwise and leads with what is actually differentiated.

## [1.0.0] - 2026-06

- Initial release: 24-lens convergence audit, 18 defect classes, fix mode with verification loop, single-file `PROMPT.md` for non-Claude-Code agents, anonymized sample report.
