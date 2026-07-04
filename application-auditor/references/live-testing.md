# Live testing: exercising a running localhost instance

Reading code proves *some* findings. A missing auth check, a hardcoded key, a swallowed error are visible on the page. But a whole class of defects only appears at runtime: a race that needs two concurrent requests, a 500 that only fires on a real payload, a security header that is set (or not) by the running server, a spinner that never resolves. This lens drives the actual product like a QA engineer would and records what happens.

**When this lens applies:** only when a running instance is reachable on the local machine. If nothing is running and you cannot start it, skip this lens and tag any finding that *would* need runtime confirmation as `static-inference` in its evidence, so the report is honest about what was proven by reading versus by execution.

**This lens never touches source code.** Audit mode is read-only for the codebase (see SKILL.md). Live testing exercises the product over its own interfaces (HTTP, browser, CLI). It may write to the *local development database* through the app's own actions, exactly as a manual tester clicking around would; it never edits your files.

## Safety rails (hard, non-negotiable)

These gate every runtime action. If any check fails, do not send the request; downgrade the finding to `static-inference` and note why.

1. **Localhost only.** The target host must resolve to a loopback or private address: `localhost`, `127.0.0.0/8`, `::1`, `0.0.0.0`, `*.localhost`, `*.local`, or an RFC-1918 range (`10.*`, `172.16-31.*`, `192.168.*`) that the user has declared in `.audit/target.json`. Any public hostname, any public IP, any `https://` pointing at a real domain: **refuse and stop.** There is no override flag for a non-local host. Production is never a target.
2. **Confirm it is a dev instance, not prod-in-disguise.** Before any write, confirm at least one dev signal: `NODE_ENV`/`APP_ENV` is `development`/`test`, the DB connection string points at a local host, or the user declared `"environment": "dev"` in the config. If the instance looks like it is talking to a production database, stop and ask.
3. **Writes are disposable-data only.** Every write action assumes the local dev DB is throwaway. Prefix created test data with a recognizable marker (e.g. `zzaudit-`) so it is trivially greppable and deletable afterward. Record every write in `.audit/live-writes.log` (method, URL, what was created) so the user can clean up.
4. **No destructive blast radius.** Do not call account-deletion, "delete all", bulk-purge, or billing-charge endpoints with real payment paths, even locally, unless the user explicitly asks for that flow to be tested. Test destructive actions on throwaway records you created, never on seed/demo data you did not.
5. **Rate-limit yourself.** Cap concurrency and total requests per endpoint so the audit does not itself look like the abuse it is testing for. The exception is a deliberate, bounded rate-limit probe (see below), which is announced in its finding.
6. **Never exfiltrate.** Runtime responses stay on the machine. Do not post captured data anywhere. Redact any real secrets/tokens observed in responses before they land in the report or `.audit/` files.

## Finding the target

1. **Read `.audit/target.json` if present.** Shape:
   ```json
   {
     "baseUrl": "http://localhost:3000",
     "environment": "dev",
     "auth": {
       "type": "cookie|bearer|header",
       "loginUrl": "http://localhost:3000/api/auth/login",
       "users": [
         { "role": "admin",  "email": "admin@zzaudit.local",  "password": "..." },
         { "role": "member", "email": "member@zzaudit.local", "password": "..." },
         { "role": "other",  "email": "other@zzaudit.local", "password": "..." }
       ],
       "tokenHeader": "Authorization",
       "tokenPrefix": "Bearer "
     },
     "skipPaths": ["/api/admin/wipe", "/api/billing/charge"],
     "safeToWrite": true
   }
   ```
   `baseUrl` and `environment` are the minimum. Multiple `users` across roles is what makes IDOR and cross-role tests possible: without at least two accounts, authorization testing stays read-inferred.

2. **Auto-detect if there is no config.** Probe common dev ports on loopback and see what answers: `3000 3001 4000 5000 5173 8000 8080 8888 9000` (web/API), plus framework hints from the repo (Next `3000`, Vite `5173`, Django `8000`, Rails `3000`, Flask `5000`, Spring `8080`). A port that returns HTML or JSON is a candidate. Confirm the guess against the repo (does the detected app match this codebase?) before testing it.

3. **Offer to start it.** If nothing is running, look for the start command (`package.json` scripts, `Procfile`, `docker-compose.yml`, `Makefile`) and offer to launch it in the background against a dev/test DB. Do not start anything that would connect to a production datastore.

4. **If auth is needed and no credentials exist,** ask once for a dev login (or a way to create test users). Without auth you can still test all unauthenticated surfaces; say so in the ledger.

## What to exercise (every action, not a sample)

Drive this off the Phase 0 inventory. For each item, actually do it:

- **Every endpoint / route.** Hit it. Record status, latency, and response shape. A 500 on a documented endpoint is a finding; so is a 200 that should have been a 401.
- **Every form and mutation.** Submit valid, empty, oversized, malformed, unicode/emoji, and boundary inputs. Watch for: unhandled 500s, validation that exists on the client but not the server (submit past the client via a direct request), input silently truncated, success responses on invalid data.
- **Auth matrix, for real.** Log in as each role. For every endpoint, replay the *exact* request as each other role and as no one. Whose data comes back? Change an ID in a request (IDOR): does another user's / tenant's object return? This is the single most valuable thing live testing does, because it needs multiple real sessions.
- **Session lifecycle.** Log in, capture the session; change the password; confirm the old session is actually dead. Log out; replay the old cookie/token; confirm it is rejected server-side. Let a token expire (or forge an expired/`alg:none` one) and confirm rejection.
- **Rate limits & abuse (bounded).** Fire a small, announced burst (e.g. 20 requests) at login, password-reset, signup, and invite/email endpoints. No throttling appearing is a HIGH finding. Keep it bounded; the goal is to observe the *absence* of a limit, not to actually flood anything.
- **Concurrency & races.** Send two identical writes simultaneously (double-submit, coupon redeem, seat purchase). Did you get two records, a double charge, an over-redeemed coupon? This is unprovable by reading; here it is one parallel request pair.
- **File flows.** Upload oversized files, wrong MIME types, files with path-traversal names (`../../etc/passwd`), a polyglot. Download another user's file id.
- **Security headers & transport.** Inspect response headers: `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options`, `Cache-Control: private` on authenticated responses, cookie flags (`HttpOnly`, `Secure`, `SameSite`). Missing/misconfigured is a finding.
- **Error surfaces.** Trigger 404, 500, malformed JSON, wrong content-type. Does the response leak a stack trace, framework version, SQL error, or internal path?
- **The browser layer (if a headless browser is available).** Load each page at phone/tablet/desktop widths; capture console errors, failed network requests, layout overflow, and elements that render but do nothing. Drive the destructive-action confirm/undo flows. Watch spinners: does the failure path ever surface, or does it spin forever? A screenshot per broken state is strong evidence.

## Turning a runtime observation into a finding

Every live finding carries reproducible evidence: the exact request (method, path, headers minus secrets, body) and the observed response (status + the telling part of the body/headers), or the browser step + screenshot. That request *is* the proof and, in fix mode, becomes the regression test for free.

Tag each finding's evidence with how it was established:
- `runtime-confirmed` - reproduced against the running instance (strongest).
- `static-inference` - read from code only; runtime not reachable or not safe to test.
- `runtime-cleared` - code looked wrong but the running server actually handles it; drop or downgrade, and say the running instance disproved it.

A read-inferred finding that the live instance contradicts gets cleared honestly: the running server is the tie-breaker, in both directions.

## After the run

- Write `.audit/live-writes.log` listing every record created, so cleanup is a grep for the `zzaudit-` marker.
- Note in the ledger which lenses ran live vs static, and how many findings were `runtime-confirmed` vs `static-inference`. That ratio tells the user how much of the report is proven against the real product.
