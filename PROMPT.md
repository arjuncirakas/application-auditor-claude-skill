# Application Auditor: single-file prompt

Not using Claude Code? Paste everything below the line into any capable coding agent (Cursor, Windsurf, Copilot, aider, a raw API call) pointed at your codebase. For Claude Code, install the skill instead. See the README.

---

Run a complete production-readiness audit of this codebase/product. Do not stop for confirmation. Do not summarize. The flat list IS the report.

This is a defensive engagement: the codebase under audit is one the user owns or is authorized to test and modify. Find defects, prove them against the real code, and, when asked to fix, remediate them. Do not produce working exploits, attack live systems outside the user's own localhost, or probe anything outside this codebase.

READ-ONLY AND SAFETY
- Audit mode never edits your source. It reads code/config/docs and writes only its own artifacts (the report and an `.audit/` working dir). Source is modified only when you explicitly ask to fix.
- Live testing, when a local instance is reachable, is LOCALHOST-ONLY. The target host must be loopback or a private RFC-1918 range; a public host is refused with no override. Runtime testing may write disposable, marked (`zzaudit-`) test data to your LOCAL dev database as a manual tester would; it never touches files and never touches production. Confirm the instance is dev (NODE_ENV/APP_ENV, local DB) before any write. Nothing captured at runtime leaves the machine; redact real secrets before they reach the report.

PRINCIPLES
- Trust what the code does, not what it's called. Open the file. Trace the real path: UI → API → data layer → response → render.
- A public claim without implementing code is itself a HIGH finding (README, landing page, docs, security page: every specific promise gets verified).
- One pass is never enough. Audit in repeated passes, each from a DIFFERENT angle, until two consecutive passes surface zero new findings.
- Verify every finding before reporting it: check it against the real code (guard upstream? enforced elsewhere? actually unreachable? does the test pass?). Verify, don't argue it away. If you can't pin it to file:line or URL+selector, drop it; if you can pin it but can't confirm it's safe, keep it and note the uncertainty (a located, uncleared security risk is reported, not dropped).
- A short list means you didn't look hard enough. Real products carry hundreds of findings.

PROCESS
1. Silently build a complete feature inventory first: every page/route/screen (including admin pages and settings toggles); every interactive UI element and state (forms + validation, modals, toasts, dropdowns, search/filter/sort, bulk actions, keyboard shortcuts, undo, pagination; loading/empty/error/offline/404/500); auth surfaces (signup, login, password reset, OAuth/SSO, MFA, sessions, invitations, roles, account deletion); every CLI command, API endpoint, webhook, integration, background job; file flows (upload/download, import/export, previews); outbound comms (emails, push, SMS, in-app notifications); billing flows (checkout, trials, upgrades, payment failure, cancellation); public surfaces (share links, embeds, sitemap/robots, OG/meta, deep links); realtime channels; static assets (icons, images, animations, fonts, favicons); LLM prompts and model configs; plus every claim in the README/docs/landing/legal/security pages, across every locale, theme, viewport, and platform shipped. This is your coverage checklist: audit every item, however small.
1b. Persist state to disk so a long, compaction-prone run survives: create `.audit/` at the repo root with `inventory.md` (coverage checklist), `findings.jsonl` (append one line per confirmed finding; de-dupe against this file), `ledger.md` (pass ledger + inventory x lens coverage matrix), `live-writes.log`. These are the audit's own files, not your source.
2. Run mechanical analyzers first, then the project's own gates. Deterministic tools find whole classes faster than reading: `gitleaks`/`trufflehog` over the tree AND git history (deleted-but-committed secrets), `npm/pnpm/yarn audit` / `pip-audit` / `govulncheck` / `cargo audit` (CVEs), `semgrep --config auto` (injection, SSRF, path traversal, crypto), `tsc --noEmit` on every tsconfig, `eslint`/`ruff`/`mypy`. Then the gates: clean install, full build, typecheck, lint, complete test suite. Every failure or ignored warning is a finding, and so is a gate configured so it cannot fail (ignoreBuildErrors, `|| true`, skipped tests, suppressions). Tool hits are candidates: still verify each against real code before it makes the list.
3. Discovery loop. Each pass sweeps the whole inventory through ONE lens. Never repeat a lens; append only NEW findings (de-duplicate by file+issue) and keep a pass ledger (`pass #, lens, new findings`). The lenses:
   - Subsystem sweep: one subsystem at a time (auth, billing, search, ...), entry points traced to storage and back. Best first pass.
   - Attack-class: IDOR, cross-tenant isolation, client-only auth, SQL/NoSQL/XSS/command/template/prompt injection, SSRF (server-side fetch of user URLs hitting internal/metadata hosts), path traversal, unsafe deserialization / prototype pollution, XXE, open redirects, crypto misuse (fast password hashes, JWT alg:none, non-constant-time compares, Math.random tokens), mass assignment (req.body setting role/isAdmin/price), business-logic & money (client-trusted prices, negative amounts, float-currency drift, replayable coupons), exposed secrets (incl. git history), unverified webhooks, missing security headers, share-link enforcement after revocation.
   - Auth & permissions deep-dive: full role x action matrix enforced server-side; session expiry, rotation, server-side logout invalidation; token signature/expiry validation and revocation; single-use expiring reset tokens; no user enumeration; MFA recovery bypasses; demotion strips access immediately; permission caches never serve stale grants; safe default permissions on new resources.
   - Claim-vs-code: every specific public promise (numbers, guarantees, feature names) traced to the code that delivers it; staged demo data presented as real counts too.
   - Data-shape: zero/one/partial/huge data, unicode/emoji/RTL, very long strings, 100k rows; pagination without limits; UI assuming at least one element.
   - Platform divergence & responsiveness: web vs mobile vs CLI vs API parity; dark mode; every page at phone/tablet/desktop widths (overflow, fixed widths, broken breakpoints, touch targets).
   - Lifecycle: signup → onboarding → daily use → plan change → offboarding → deletion; retention promises actually honored, revoked integrations actually stop, deletion cascades.
   - Write-path integrity: external-effect writes idempotent under retry; non-atomic sibling writes (record live before its permission row).
   - Failure-mode: each dependency down/slow/garbage: timeouts, backoff, swallowed errors, dead-letter paths, malformed-response parsing.
   - Dead-and-stale: docs for removed features, TODO/FIXME in prod paths, flags off with live marketing, old screenshots, drifted migrations.
   - Gate-escape: defects that pass the gates run in step 2 (type errors outside the typecheck path, generated code outside CI, assertion-free tests).
   - Perf: N+1, missing indexes, no-LIMIT queries, unbounded loops, render waterfalls, bundle size; Core Web Vitals, unoptimized images, render-blocking assets, missing cache headers; calls without timeout or cap.
   - A11y & UX-jank: focus order, ARIA, contrast, reduce-motion, skeleton/empty/error states, forms losing input, destructive actions without confirm or undo; animations that stutter, never finish, or block input; spinners with no failure path.
   - Content & copy: typos, placeholder text, jargon on non-technical pages, stack traces/JSON/NaN rendered to users; every number, stat, and price identical across landing/pricing/docs/app.
   - Asset & icon integrity: broken images/icons, mixed icon sets, icons contradicting their action, missing favicon/OG/touch icons, missing alt text, fonts that flash or never load.
   - Connection & wiring: frontend calls to dead/renamed endpoints, hardcoded localhost/staging URLs, secrets in the client bundle, CORS wider than needed; DB pool size vs concurrency, leaked connections, no acquire timeout, missing TLS; test keys in prod, env mismatches between duplicated configs.
   - LLM & prompt quality: prompts contradicting their output parsers, unvalidated model output with no fallback, missing token/cost/timeout caps, deprecated model IDs, near-duplicate prompts drifted apart, PII to providers, user content concatenated where instructions live.
   - Resource leaks & long-running drift: listeners/intervals/observers never cleaned up, unbounded in-memory caches, unclosed file/socket/DB handles, workers growing with uptime, temp files accumulating, logs without rotation.
   - Observability & ops: errors swallowed untracked, unlogged critical paths, PII in logs, no health checks, no graceful shutdown, no alerts on job failures, stack traces or source maps in prod.
   - Abuse & limits: no rate limits on auth/email/expensive endpoints, unbounded uploads and payloads, uncapped LLM spend, missing storage quotas, no bot resistance where it matters.
   - Config & environment: env vars unvalidated at boot, dev defaults in prod, debug mode reachable, drifted duplicate configs, untested flag states, real secrets in committed examples.
   - Dependency & supply-chain: known CVEs in the lockfile, deprecated packages, unpinned versions, duplicate versions of one library, install scripts, license conflicts.
   - Caching correctness: cache keys missing tenant/user scope, stale after writes, authorization cached past revocation, CDN caching authenticated responses, localStorage surviving logout.
   - Concurrency & races: double-submit, two tabs, two workers on one job, webhooks delivered twice, check-then-act without unique constraints, last-write-wins edits, non-atomic counters, job locks that don't expire.
   - Live testing (only when a local instance is reachable): drive the running product. Hit every endpoint; submit every form with valid/empty/oversized/malformed input; run the auth matrix across real sessions (replay each request as every role, change ids for IDOR); fire bounded probes for missing rate limits and concurrency races; upload hostile files; inspect security headers and error surfaces for leaks. Safety rails: localhost-only, dev-DB-only, disposable marked test data, never production, never destructive endpoints on data you didn't create. Tag each finding runtime-confirmed / static-inference / runtime-cleared; the exact reproducing request is the evidence.
4. Stop discovery only when two consecutive diverse passes find nothing new AND the inventory x lens coverage matrix is full (a "0 new" pass that skipped half the inventory is incomplete, not quiet). Cap the loop: run every applicable lens at least once plus a ceiling of a few more passes; if you hit the cap first, report what you have and say so. Order CRITICAL-dense lenses (attack-class, auth, caching, write-path) early so a truncated run still surfaces the worst first. Convergence is a strong heuristic for "looked hard enough", not a proof nothing remains. Before reporting, read `.audit-ignore` at the repo root if present and skip findings matching a confirmed-false-positive entry there (format `path:line  issue-tag  # reason`, matched on path + issue-tag + content, not bare line number); never add entries yourself.
5. If your harness supports subagents or parallel tasks, fan out one finder per lens/subsystem and route every candidate through a BLIND verifier in a fresh context (a different model if available): the verifier gets only the finding + code access, never the finder's reasoning (which anchors it toward confirming), and is prompted to confirm or clear against the code and the running instance, not to argue it away. If not, run lenses sequentially yourself with the same verification step.

SCOPING
The user may scope the run (one subsystem like src/billing, one lens family like security, or docs-vs-code). Apply the same loop, lenses, and rules to the narrowed inventory. No scope given = the whole product.

PER-FEATURE CHECKS
Implemented end-to-end or stubbed; empty/partial/huge data; loading/error/empty states; server-side authorization on every path that reaches the data, correct for every role (not just the happy path); external writes idempotent and audited; parity across every surface the product ships.

OUTPUT FORMAT
Write the report to `audit-report.md` at the repo root (the deliverable), and also print it unless it is too large. A single flat list. No preamble, no overview, no closing summary. Each row, 1-2 lines max:

[SEVERITY] [AREA] path/to/file.ts:123 - what is wrong - one-line fix

For CRITICAL/HIGH rows append an `evidence:` clause (the reproducing request for runtime-confirmed, the code path traced for static-inference, or the tool that flagged it).

For example:
[CRITICAL] [SECURITY] src/lib/cache.ts:21 - dashboard cache key omits the workspace id; one tenant's data served to another - add the tenant to the key
[HIGH] [CONTENT] landing/security.html §hero - claims "AES-256 encryption at rest"; no encryption configured in the storage layer - implement it or remove the claim
[MEDIUM] [PERF] src/dashboard/page.tsx:61 - members fetched per project in a loop (N+1) - one grouped query

SEVERITY = CRITICAL (data loss, breach, broken core flow, crash) / HIGH (claimed feature broken or missing, security weakness, silent failure) / MEDIUM (degraded behavior, edge-case failure, real inconsistency) / LOW (minor bug, polish) / IMPROVEMENT (concrete upgrade only, no hedging).
AREA = whatever fits the product: FRONTEND BACKEND API DB SECURITY AUTH DOCS CONTENT UX A11Y PERF RELIABILITY DATA BUILD MOBILE INTEGRATION CONFIG AI.
Order CRITICAL → HIGH → MEDIUM → LOW → IMPROVEMENT; group by AREA within severity.

HARD RULES
- Every row has file:line or URL+selector. No location means drop the row.
- No "consider/might/could/potentially". Concrete defects, concrete fixes.
- Don't skip small features or known-broken areas; same depth everywhere, no commentary.
- If you run out of context, end with exactly one line: TRUNCATED AT [area] - [N] inventory items unchecked. Nothing after it.

IF ASKED TO FIX
Fix mode is the only mode that edits source. Branch first and commit after each gated wave (a crash mid-run must not lose work, and the over-reach review needs a clean diff). Fix in severity waves (1: CRITICAL+HIGH, 2: MEDIUM, 3: LOW+IMPROVEMENT), gating each wave on green clean build + typecheck + lint + full test suite, then reviewing every change for over-reach (anything changed beyond its finding gets reverted). For a runtime-confirmed finding, add its reproducing request as a regression test before fixing. Deferrals are listed with a reason, never silently dropped. Then run a verification loop: re-audit with fresh angles (regression, fix-completeness, over-reach; the same mistake is almost never made once) and fix what surfaces, round after round, until two consecutive passes find zero CRITICAL and zero HIGH. Expect 10+ rounds on a real codebase; cap at a sane ceiling and report any still-open CRITICAL/HIGH rather than looping forever.
