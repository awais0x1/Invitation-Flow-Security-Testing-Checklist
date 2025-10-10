# Invitation Flow Security Testing Checklist

---

## Table of contents
- [Critical tests](#critical-tests)
- [High / Important tests](#high--important-tests)
- [Medium tests](#medium-tests)
- [Lower-priority tests](#lower-priority-tests)
- [Detailed scenario tests](#detailed-scenario-tests)
- [Reporting template](#reporting-template)

---

# Critical tests

- [ ] **Token not bound to invited email / account**
  - **What:** Invite token can be accepted by an account other than the invited email.
  - **Steps:** Create invite for `victim@example.com`. While logged in as attacker, open `https://target/invite/accept?token=<TOKEN>` or POST accept endpoint with token.
  - **PoC:** `POST /api/invites/accept {"token":"<TOKEN>"}`
  - **Bad:** Attacker joins as invited user.
  - **Impact:** Account takeover / unauthorized access.
  - **Fix:** Bind token to invited email and require verification on accept.

- [ ] **Role elevation via invite (create or accept time)**
  - **What:** Client-supplied `role` is trusted and attacker can request higher role.
  - **Steps:** Intercept `POST /api/invites` or `POST /api/invites/accept`, change `"role":"member"` → `"role":"admin"`.
  - **PoC:** `POST /api/invites {"email":"x@x.com","role":"admin"}`
  - **Bad:** Invite or acceptance results in admin-level membership.
  - **Impact:** Privilege escalation — critical.
  - **Fix:** Server-side role validation (only admins can create admin-level invites) and ignore client-provided role on accept.

- [ ] **Deleted/Revoked invite token still usable**
  - **What:** Invite deleted/revoked, but token still grants access.
  - **Steps:** Create invite, capture token, delete/revoke invite, then attempt accept with token.
  - **PoC:** `DELETE /api/invites/{id}` then `POST /api/invites/accept {"token":"<TOKEN>"}`
  - **Bad:** Accept succeeds after deletion.
  - **Impact:** Revocation bypass, persistent unauthorized access.
  - **Fix:** On `accept`, check DB invite `status`/`revoked` and invalidate tokens immediately on revoke.

- [ ] **Invite token reuse / multiple-accepts**
  - **What:** Tokens are multi-use and allow repeated joins.
  - **Steps:** Accept invite once, then replay accept with same token to add more accounts.
  - **PoC:** Replay `POST /api/invites/accept {"token":"<TOKEN>"}`
  - **Bad:** Multiple successes.
  - **Impact:** Mass unauthorized joins or confusion.
  - **Fix:** Mark token consumed on first use; single-use semantics.

- [ ] **Cross-tenant invite acceptance (org_id tampering)**
  - **What:** Attacker modifies `org_id`/`team_id` during accept to join a different tenant.
  - **Steps:** Modify `team_id` in accept request or accept token in a different org context.
  - **Bad:** Attacker joins an org they shouldn't.
  - **Impact:** Cross-tenant data exposure.
  - **Fix:** Validate invite → org mapping server-side; enforce RBAC.

---

# High / Important tests

- [ ] **Invite role changes not propagated (old role persists)**
  - **What:** Invite originally `admin`, then changed to `member`. Token still grants `admin`.
  - **Steps:** Create invite with role A, PATCH invite to role B, accept with original token.
  - **Bad:** Accepted role = original role A.
  - **Impact:** Privilege escalation.
  - **Fix:** Invalidate tokens on invite edits or use DB-stored role at accept.

- [ ] **Accept without authentication (open accept)**
  - **What:** Accept endpoint doesn't require authentication and may attach the invitation to the wrong session.
  - **Steps:** Visit accept link while unauthenticated or as another user.
  - **Bad:** Wrong account becomes member.
  - **Impact:** Unauthorized joins.
  - **Fix:** Require authentication & email verification or one-time ownership checks at accept.

- [ ] **Predictable invite IDs / IDOR (enumeration)**
  - **What:** Invite IDs or endpoints are sequential/guessable.
  - **Steps:** Iterate `/api/invites/1000..1010` and observe details.
  - **Bad:** Access to other invites or PII.
  - **Impact:** Reconnaissance & abuse.
  - **Fix:** Use opaque UUIDs; require auth and proper permissions.

- [ ] **Webhook / payment callback forging (fake upgrade)**
  - **What:** Webhook endpoints (payment/provider callbacks) lack signature verification.
  - **Steps:** Replay recorded webhook payload; observe subscription status change (only in test environment or in-scope conditions).
  - **Bad:** Account marked as paid without payment.
  - **Impact:** Revenue loss, feature abuse.
  - **Fix:** Verify webhook signatures; validate with payment provider.

- [ ] **Race: accept vs revoke (timing window)**
  - **What:** Accept request competes with revoke/delete and may succeed.
  - **Steps:** Send concurrent `DELETE /invites/{id}` and `POST /invites/accept` in rapid succession or parallel.
  - **Bad:** Accept succeeds after deletion.
  - **Impact:** Revocation bypass.
  - **Fix:** Atomic state transitions, optimistic locking, verify invite status at acceptance time.

- [ ] **Inviting existing users overwrites roles**
  - **What:** Inviting an already-registered user overwrites their role without consent.
  - **Steps:** Invite an existing member and observe whether role changes.
  - **Bad:** Existing account roles changed without consent.
  - **Impact:** Privilege changes, DoS of admins.
  - **Fix:** Require explicit consent for role changes or refuse invite for existing accounts.

---

# Medium tests

- [ ] **Email canonicalization / plus-address ambiguity**
  - **What:** `alice@example.com` vs `alice+tag@example.com` treated inconsistently.
  - **Steps:** Invite `alice@example.com`, accept as `alice+tag@example.com` or case variations.
  - **Bad:** Duplicate accounts or acceptance by unintended user.
  - **Impact:** Account split/takeover.
  - **Fix:** Normalize/verify emails server-side.

- [ ] **Token stored in browser history / referer leaks**
  - **What:** Token in GET URL gets sent in Referer to third parties.
  - **Steps:** Visit invite link while page includes third-party assets; inspect outbound Referer headers.
  - **Bad:** Token leaked to third parties via Referer.
  - **Impact:** Token theft → reuse.
  - **Fix:** Put tokens in POST body, short TTL, set Referrer-Policy, avoid third-party assets on accept page.

- [ ] **PII in invite URLs (email visible in query)**
  - **What:** Email included in `?email=` parameter in URLs.
  - **Steps:** Inspect invite link; check logs and referer exposures.
  - **Bad:** PII leaked via referrer or logs.
  - **Impact:** Privacy breach.
  - **Fix:** Remove PII from URLs; use opaque tokens.

- [ ] **Invite message / note XSS**
  - **What:** Invite messages allow HTML/JS.
  - **Steps:** Submit `<img onerror=alert(1)>` in invite message; open email or admin UI.
  - **Bad:** Script runs in inbox or admin UI.
  - **Impact:** XSS / phishing.
  - **Fix:** Sanitize/escape message content.

- [ ] **CSRF on invite creation (mass invites)**
  - **What:** Lack of CSRF on invite creation allows attacker pages to trigger invites from admin’s browser.
  - **Steps:** Create HTML page that auto-POSTs `POST /api/invites` and trick an admin into visiting.
  - **Bad:** Mass invites sent.
  - **Impact:** Reputation loss, spam.
  - **Fix:** Add CSRF tokens, SameSite cookies, admin confirmation flows.

- [ ] **Role escalation via accept parameters**
  - **What:** Accept API accepts a `role` param and trusts it.
  - **Steps:** Intercept accept request and change `"role":"admin"`.
  - **Bad:** Role granted by accept differs from server-stored role.
  - **Impact:** Privilege escalation.
  - **Fix:** Server must ignore client-provided role; use DB-stored role.

- [ ] **Invite creation by low-privilege user (broken RBAC)**
  - **What:** Non-admins can create invites by hitting API.
  - **Steps:** As normal user call `POST /api/invites` to invite someone to org.
  - **Bad:** Invite created without proper authorization.
  - **Impact:** Unauthorized org population, phishing, lateral movement.
  - **Fix:** Enforce server-side RBAC and scopes.

- [ ] **Invite acceptance without authentication (open accept)**
  - **What:** Accept endpoint doesn’t require auth and binds to an attacker session.
  - **Steps:** Use invite link while logged out or logged-in as attacker and see which account becomes member.
  - **Bad:** Attacker becomes member / links accounts.
  - **Impact:** Account takeover / unauthorized join.
  - **Fix:** Require email ownership verification or authenticated accept flow with matching email.

- [ ] **Invite acceptance binding to wrong org (cross-tenant join)**
  - **What:** Accepting invite can be redirected to another org by tampering `org_id`/`team_id`.
  - **Steps:** Modify `team_id` in accept request to another org’s team.
  - **Bad:** Attacker joins a different org.
  - **Impact:** Cross-tenant membership, data leaks.
  - **Fix:** Validate invite→org mapping server-side.

- [ ] **Invites to existing users overwrite roles (downgrade/upgrade)**
  - **What:** Sending an invite to an email already in system changes that user’s role.
  - **Steps:** Invite an existing member with different role and observe role change.
  - **Bad:** Existing user’s role changed without consent.
  - **Impact:** Privilege escalation/demotion, DoS of admins.
  - **Fix:** Handle invites to existing users gracefully; do not overwrite roles without consent.

- [ ] **Unvalidated redirect after accept (open redirect / phishing)**
  - **What:** `redirect_uri` param not validated.
  - **Steps:** Accept invite with `redirect_uri=https://attacker.com` and see redirect.
  - **Bad:** Victim redirected to attacker site; possible token capture.
  - **Impact:** Phishing, credential stealing.
  - **Fix:** Whitelist redirect URIs.

- [ ] **Webhook / callback allowing SSRF or token leak**
  - **What:** Invite flow triggers callback to user-supplied URL.
  - **Steps:** Supply `callback_url` pointing to internal host or attacker and trigger invite.
  - **Bad:** Server fetches attacker/internal URL; tokens may leak.
  - **Impact:** SSRF, internal scanning, token exfiltration.
  - **Fix:** Whitelist/validate callback hosts; sanitize payloads.

- [ ] **Invite message/subject injection (XSS / social engineering)**
  - **What:** Invite content accepts HTML/JS.
  - **Steps:** Put `<img onerror=...>` in invite note and view email or web console.
  - **Bad:** XSS in admin console or inbox; phishing.
  - **Impact:** Account compromise / phishing.
  - **Fix:** Sanitize input; encode in outputs.

- [ ] **Old tokens valid after password/email changes (no revocation)**
  - **What:** Tokens issued earlier still valid after sensitive changes.
  - **Steps:** Request reset token, change email/password, then use old token.
  - **Bad:** Old token still works.
  - **Impact:** Account takeover after user tries to secure account.
  - **Fix:** Revoke older tokens on sensitive changes; check token `iat` vs `last_sensitive_change`.

- [ ] **Invite deletion soft-delete (orphaned tokens)**
  - **What:** Deleting invite only hides it in UI; token still valid.
  - **Steps:** DELETE invite record and try accept with saved token.
  - **Bad:** Accept succeeds despite deletion.
  - **Impact:** Revocation ineffective — attacker can still join.
  - **Fix:** Hard revoke tokens; keep `revoked` flag and verify on accept.

- [ ] **Race: accept & revoke timing window**
  - **What:** Accept happens concurrently with revoke and wins.
  - **Steps:** Automate simultaneous `DELETE /invites/{id}` and `POST /invites/accept` many times.
  - **Bad:** Accept sometimes succeeds after revoke.
  - **Impact:** Revocation bypass (timing-based).
  - **Fix:** Use transactional checks, optimistic locking, and atomic state transitions.

- [ ] **Bulk-invite abuse (CSRF / spam)**
  - **What:** No CSRF or anti-automation allows mass invite spamming via unsuspecting admin’s browser.
  - **Steps:** Create HTML page that auto-POSTs `POST /api/invites` and load as admin.
  - **Bad:** Mass invites sent; mailbox flooding.
  - **Impact:** Reputation loss, spam.
  - **Fix:** CSRF tokens, SameSite cookies, admin confirmation flows, rate limiting.

- [ ] **Token contained in email attachments / images (exfil via image proxy)**
  - **What:** Email client loads images from remote URLs with token in query.
  - **Steps:** Invite link includes `img src="https://attacker/track?token=..."` or includes token in tracking.
  - **Bad:** Token capture by remote server.
  - **Impact:** Token capture and reuse.
  - **Fix:** Avoid embedding sensitive tokens in image or public URLs.

- [ ] **SSO/SSP invite binding failure (account link confusion)**
  - **What:** Invite acceptance via SSO creates separate accounts or links wrong account.
  - **Steps:** Invite user who uses SSO; accept using SSO vs email flow and compare account linking/roles.
  - **Bad:** Account duplication or wrong account linked.
  - **Impact:** Account duplication, takeover, role confusion.
  - **Fix:** Properly link SSO identity to invite email, require verification and account linking consent.

---

# Detailed scenario tests

## 1) Invitation token still works after role change

**What this means:** Admin creates invite for `victim@example.com` with `role=admin`. Later Admin edits the invite (or the org changes intended role) to `role=member`. The token — when accepted — still grants the original (old) role (`admin`) instead of the new role (`member`).

**Steps to test**

1. **Create invite with elevated role**
   - As **AdminA** create invite with role `admin`. Capture `invite_id` and `INVITE_TOKEN` or email link.

2. **Modify the invite role (server-side change)**
   - While the invite is still pending (not accepted), have **AdminA** change the invite role to `member` via the UI or API:

   ```http
   PATCH /api/invites/{invite_id}
   Authorization: Bearer <AdminA_JWT>
   Content-Type: application/json

   { "role": "member" }
   ```

   - Capture the PATCH response — it should indicate success and the invite now shows `role: member`.

3. **Accept the invite using the original token**
   - In a different browser/session (or as the InviteTarget), open or POST the original token:

   ```http
   POST /api/invites/accept HTTP/1.1
   Content-Type: application/json

   { "token": "<INVITE_TOKEN>" }
   ```

   - Or visit `GET /invite/accept?token=<INVITE_TOKEN>` if web UI.

4. **Observe granted role**
   - After accept, inspect the returned object or query `GET /api/users/me` or admin UI to see what role the newly-joined account has.
   - Evidence to capture: response JSON from accept request, `GET /api/orgs/{org}/members` showing role, any session JWT claims (if JWT returned).

**Success condition (vulnerable)**

- The account is created/added with **role: admin** (original role) even though the invite was changed to `member`. Or the accept response contains `"role":"admin"` or JWT contains `role=admin`.

**Variations to try**

- Flip order: create invite for `member`, then change to `admin`, see if accept gives `admin`.
- Change role multiple times and accept — see which one wins.
- Try editing role by sending `PATCH` with another `admin`’s credentials (privilege distinction).

**Impact**

- **High/Critical**: attackers can be granted higher privileges than intended (e.g., admin) despite owner changes — privilege escalation.

**Why this happens (likely)**

- Server sets the role **once** at invite creation and stores it in token or signs it into token payload; patching invite does not update or invalidate previous token/record; accept uses token data rather than current DB invite state.

**Remediation (what to recommend)**

- On `accept`, use server-side invite record to determine role (lookup `invite_id`) — ignore any role value embedded in token.
- If role is updated, update stored invite record and/or **invalidate previously issued token** (rotate token or set `valid=false` and issue new token if needed).
- Do not encode role in token claims without re-checking DB state on accept; sign tokens with short TTL and verify DB state on accept.


## 2) Deleted admin invitation token still grants access

**What this means:** Admin creates an invite (role=`admin`), then deletes or revokes the invite (e.g., `DELETE /api/invites/{id}`). Despite deletion, the previously-issued `INVITE_TOKEN` remains valid and allows acceptance.

**Steps to test**

1. **Create admin invite** (same as above), save `invite_id` and `INVITE_TOKEN`.

2. **Delete / revoke the invite**
   - As AdminA or an admin UI action:

   ```http
   DELETE /api/invites/{invite_id}
   Authorization: Bearer <AdminA_JWT>
   ```

   - Capture the delete response (e.g., `204 No Content` or `200 { deleted: true }`).

3. **Attempt to accept using the old token**
   - Immediately after deletion:

   ```http
   POST /api/invites/accept
   Content-Type: application/json

   { "token": "<INVITE_TOKEN>" }
   ```

   - Or visit `GET /invite/accept?token=<INVITE_TOKEN>`.

4. **Observe outcome**
   - If the accept still succeeds, that confirms the vulnerability. Capture accept response and resulting account membership.

**Race condition test (stronger proof)**

- Test timing: send the accept request **at the same time** as the delete operation (or shortly after). This proves whether deletion and accept are not atomic or the accept uses cached/independent validation.

- Example (simple parallel test using curl & xargs; be careful):

```bash
# send delete
curl -s -X DELETE "https://app.example.com/api/invites/<invite_id>" -H "Authorization: Bearer <AdminA_JWT>" &
# immediately send accept
curl -s -X POST "https://app.example.com/api/invites/accept" -d '{"token":"<INVITE_TOKEN>"}' -H "Content-Type: application/json"
```

Repeat several times to increase race chance.

**Success condition (vulnerable)**

- You can accept and join the org even though invite was deleted. Evidence: accept response, new account membership, session token or ability to call admin-only APIs.

**Impact**

- **High/Critical:** revocation does not work — attackers can use stale tokens to join as admin even after owner attempted to revoke. Persistent unauthorized access and privilege escalation.

---

# Reporting template
Use this template when filing an issue in HackerOne/Bugcrowd/GitHub issue.

**Title:** [Business Logic] <Short summary: e.g., Invite token valid after revoke/downgrade>

**Summary:** One-line summary of the issue and impact.

**Steps to reproduce:**
1. (Actor) Create an invite with role X — `POST /api/invites` — capture `invite_id` and `token`.
2. (Actor) PATCH or DELETE the invite — `PATCH /api/invites/{id}` or `DELETE /api/invites/{id}`.
3. (Attacker) Use saved token to `POST /api/invites/accept {"token":"<TOKEN>"}`.

**Actual result:** Describe the observed behavior and attach Burp requests & responses.

**Expected result:** Invite accept should fail or grant the updated role; tokens revoked on delete/patch.

**Impact:** Explain what an attacker could do (e.g., gain admin access, bypass revocation).

**PoC:** Include exact HTTP requests (with placeholders) and responses.

**Suggested fix:** List recommended server-side checks and remediation steps.

**Severity:** (Critical / High / Medium) with rationale.

---

## Notes & references
- OWASP Web Security Testing Guide — Business Logic testing.
- PortSwigger labs & writeups on logic flaws & race conditions.


