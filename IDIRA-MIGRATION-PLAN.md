# Idira Migration & Certification Plan — cribl-cyberark-rest-io

**Status:** partially implemented — see §0
**Target kit:** `audit-service/siem-integration` (Idira partner integration dev kits)
**Telemetry `it` value:** `SIEM` (fixed by registry — do not invent)
**Date:** 2026-11-19

Sources: `idira-partner-integration-docs/docs/audit-service/siem-integration/certification-checklist.md` (v2.4),
`development-guide.md` (v1.5), `telemetry-reference.md` (v1.3), `naming-crosswalk.md` (v1.0),
`marketplace-standards.md` (v1.11), `journey.md` (v2.0).

---

## 0. Implementation status — 2026-11-19

Shipped in the working tree as **2.0.0**. Not committed.

| Done | Item | Checks |
|:--|:--|:--|
| ✅ | Identity collector, pipeline, route, breaker rules and samples deleted | 4.1, 4.2 |
| ✅ | `x-cybr-telemetry` on all three call sites (token, discover, collect), built from the new `Idira_Telemetry` Variable | 7.1–7.7 |
| ✅ | README: `Report Management` → **Read Only System Administration** | 3.4 |
| ✅ | README: `Allowed REST APIs = .*` → restrict to endpoints called | 6.1 |
| ✅ | README: `jwtRS256`, Client Creds auth method, Advanced claim script added | 9.3 |
| ✅ | README: SIEM key shown once, two-slot disclosure, 7-day retention, rate/page limits | 9.3, 4.9 |
| ✅ | README: field-mapping reference incl. snake_case/camelCase asymmetry | 5.3, 9.4 |
| ✅ | README: scope-of-collection statement | 2.4 |
| ✅ | README: dedup argument on `uuid`; null-stripping clarified as values-only | 5.1, 5.4 |
| ✅ | `package.json` → 2.0.0, `displayName` "Idira REST IO", `idira`/`siem` streamtags added, `cyberark` kept | 9.5 |
| ✅ | All frozen identifiers verified unchanged (§4.1) | — |
| ✅ | **Pre-made Pack Variables** — added `Idira_Secrets`, `Idira_OAuth`, `Idira_Collection` (now 5 total). Collector credential / token-URL / scope / page-size fields read from them; the 1.x `changeme` placeholders are gone. All 5 Variable JSON blobs parse and all 9 collector JS expressions evaluate against stubs | 6.5, 4.4 |
| ✅ | **Cursor pagination** — `pagination.type: response_body`, `attribute: [paging.cursor.cursorRef]`, `maxPages: 0`. Needs a one-run verification (below) | **4.6** |

### Note on Pack export scrubbing

**Cribl's Pack exporter replaces request-header values and client-secret fields with `changeme`.**
Verified empirically against the exported 1.0.0 artifact (`c476ab6:default/jobs.yml`):

| Scrubbed → `changeme` | Preserved verbatim |
|:--|:--|
| `discoverRequestHeaders[].value` | `loginUrl`, `collectUrl`, `discoverUrl` |
| `collectRequestHeaders[].value` | `discoverBody`, `collectBody` |
| `authRequestHeaders[].value` | **`loginBody`** — survived containing `C.Secret(...)` |
| `username`, `password`, `clientSecretParamValue` | `authRequestParams[].value` |

`safeHeaders` does **not** exempt a header: 1.0.0 listed `X-IDAP-NATIVE-CLIENT` in `safeHeaders` and
its header value was still scrubbed.

**Consequence for this Pack.** Of the six fields wired to Variables, three survive export (`loginUrl`
app id, `authRequestParams` scope, `discoverBody` page size + application filter) and three do not
(both `x-api-key` headers, the `Authorization` auth header, plus `clientSecretParamValue`). The three
`x-cybr-telemetry` headers are also scrubbed — **that is 7 Required checks (7.1–7.7) reverting on
every export, and it fails silently because the collector still collects events.** The README's
*After you import the Pack* section is the mitigation: five exact strings to paste, telemetry called
out separately.

**DONE 2026-11-19 — OAuth credentials moved into `loginBody`.** `loginBody` is preserved on export
*and* tolerates `C.Secret(...)`: the 1.0.0 identity collector proved this, carrying
`client_id=${C.Secret('cyberark_api_key','keypair').apiKey}` through export intact. The audit
collector is now `authentication: login` instead of `oauth`, so the client ID and secret live in the
preserved `loginBody` rather than a scrubbed `Authorization` header.

Post-import burden: **7 fields / 4 values → 5 fields / 2 values.** The credential pastes are gone;
what remains is the `x-api-key` header (×2) and `x-cybr-telemetry` (×3). `username` and `password`
are scrubbed but unused — they exist only for the collector form's required-field validation.

Both modes are OAuth 2.0 `client_credentials` (check 3.1) against the same `.id.` endpoint
(check 3.5); RFC 6749 §2.3.1 permits client credentials via HTTP Basic *or* request-body parameters.
The resolved body was verified to match the dev guide's documented request character for character,
including the unencoded colon in `scope=isp.audit.events:read`.

**Risk and rollback.** If the Idira OAuth2 app requires HTTP Basic client authentication rather than
body parameters, the token request returns `401` / `invalid_client` — a loud, immediate failure, not
a silent one. The README's *Why the token request uses `loginBody`* section carries the exact
four-field rollback to `authentication: oauth`. **This is the one change in 2.0.0 that has not been
run against a live tenant.**

Telemetry cannot be rescued this way; a header is a header.

### Note on the pagination schema

Two pagination schemas exist in a current Stream UI bundle and they are easy to confuse:

| Discriminator | Branch fields | Belongs to |
|:--|:--|:--|
| `paginationMethod` | `paginationAttribute`, `paginationTokenNames`, `maxPages`, `lastPageExpr` | The newer pull **Sources** (OpenAI, Wiz, …) |
| `type` | `attribute`, `maxPages`, `lastPageExpr` | The **REST collector** — what this Pack uses |

The REST collector's `response_body` branch takes `attribute` under the `type` discriminator.

**`nextRelationAttribute: next` is not the cursor field.** It belongs to the
`response_header_link` / `rfc5988` branch, which parses an RFC 5988 `Link` header that the Idira
Audit Service does not send. It is the only next-page-looking key in the block, so it is the obvious
thing to reach for and it is the wrong thing to reach for.

**The inactive-branch keys are retained deliberately** — `pageField`, `sizeField`, `size`,
`zeroIndexed`, `offsetField`, `limitField`, `limit` and `nextRelationAttribute` stay pre-staged with
example values. Cribl ignores keys outside the active branch, the Pack has always shipped this way,
and they serve as in-UI documentation of the available pagination modes: a customer switching mode
finds the fields populated rather than blank. Do not "clean them up".

### Still outstanding

| Blocked on | Item | Checks |
|:--|:--|:--|
| **Live tenant — highest priority** | **Confirm the new `authentication: login` token request works.** The Idira OAuth2 app may require HTTP Basic client auth rather than body parameters; failure is a loud `401` / `invalid_client`. Rollback to `oauth` is in the README. This is the only 2.0.0 change not yet run against a live tenant | 3.1, 3.5 |
| **One run against >500 events** | Confirm the pagination template variable binds. The collect body references `${cursorRef}`; if the engine exposes the full attribute path on your Stream version, the body needs adjusting. Verification steps are in the README's **Pagination** section. Fails safe — a wrong binding collects one page per run, which is the 1.x behaviour | **4.6** |
| **No asset available** | `default/pack.yml` `logo` still holds the CyberArk base64 JPEG. Needs the Idira / Palo Alto Networks mark | 9.5 |
| **Idira Technical Alliances** | `integration_name` and `vendor_name` in `Idira_Telemetry` are defaults, not agreed values | **7.7** |
| **Your decision** | Scope under 2.4 — ISP events only, or include Idira Privileged Access Manager SaaS (decides whether 8.2/8.3 apply). README currently states ISP + Audit Service only | 2.4 |
| **Live tenant** | Functional validation §8 on real events, including the 8.5 stop/generate/restart interruption test | 8.1–8.5 |
| **Process** | Named production or pilot customer; security scan; demonstration; attestation | 2.5, 6.4, 10, 12 |
| **Optional** | Mask function over `customData`/`message`/`command`; `applicationCode` filter Variable; document 401/403 handling | 6.2, 4.4, 4.7 |

---

## 1. Headline findings

Branding is the smallest part of this. Four **Required** gates fail today on build, not on naming:

| # | Gate | State |
|:--|:--|:--|
| 7.1–7.7 | `x-cybr-telemetry` header on every Idira API request | **Absent entirely** — 7 Required checks |
| 4.6 | Pagination to completion via `cursorRef` | **Fails** — `pagination.type: none`; events beyond the first 500-event page are dropped every cycle |
| 3.4 | Service user holds **Read Only System Administration** | **README is wrong** — it instructs `Report Management`, which grants zero audit data with no error raised |
| 6.1 | Least privilege | **README instructs `Allowed REST APIs = .*`** |
| 4.1 / 4.2 | Only documented Audit Service endpoints; `createQuery`/`results` only | **Fails while the identity collector ships** (see §3) |

4.6 is a live data-loss bug, independent of certification: any tenant producing more than 500 audit
events in a 5-minute window silently loses the remainder on every run.

---

## 2. What the pack already gets right

Do not touch these during the rebrand — they are the passing evidence.

- **OAuth2 `client_credentials` against the `.id.` endpoint** (3.1, 3.5) — `loginUrl` uses
  `identity_host`, scope `isp.audit.events:read`.
- **Both required headers on every stream call** (4.3) — `Authorization: Bearer ${token}` and
  `x-api-key` on discover *and* collect.
- **Secrets in a credential store, never plaintext** (3.7, 6.5) — `C.Secret('cyberark_api_key','keypair')`,
  `C.Secret('cyberark_audit_x_api_key','text')`.
- **`captureHeaders: false`** (3.8) — keeps the API key and bearer token out of event payloads.
  **This must stay `false`.**
- **Exponential backoff on 429** (4.8) — `retryRules: backoff`, `multiplier: 2`, `maxIntervalMs: 20000`.
- **Lossless resume** (4.9) — full state tracking with `latestTime` and a `+.001` window advance.
  This is the hardest check in the kit and it is already built.
- **Timestamps preserved in UTC** (5.2) — verified against `data/samples/KLFDbK.json`: raw
  `timestamp: 1770080106394` (milliseconds) is preserved in `_raw` and `_time` resolves to
  `1770080106.394`. The dev guide's seconds-vs-milliseconds warning is real and the breaker's
  auto-detect handles it correctly.
- **`uuid` is collected** — present in `selectedFields` and in the samples, so the stable dedup key
  required by 5.1 is available.

---

## 3. Remove the identity collector — and why

**Certification rationale.** `/Redrock/query` is an Idira **Identity** API, not an Audit Service
endpoint. Checks 4.1 and 4.2 are Required and read "only documented and supported Idira **Audit
Service** endpoints" and "no undocumented or scraped endpoints". The one place `Redrock/query` is
sanctioned anywhere in the Idira corpus is the `identity-access-management/identity-governance`
kit — which is **Preview**, publishes no development guide, carries a different `it` value
(`IdentityGovernance`), and is not open for Marketplace listing. The identity flow is therefore not
certifiable under any kit today, and shipping it puts 4.1/4.2 at risk for the whole submission.

### Files to change

| File | Action |
|:--|:--|
| `default/jobs.yml` | Delete the `cyberark_identity_logs` block (lines 1–117) |
| `default/pipelines/cyberark_identity_parsing/` | Delete the directory |
| `default/pipelines/route.yml` | Delete route `4jWdMW` |
| `default/breakers.yml` | In `CyberArk_API_Ruleset`, delete the `identity_logs` and `identity_error` rules. Keep `audit_logs` |
| `default/samples.yml` + `data/samples/` | Delete `AYYkme` and `0ji3OK` entries and their JSON files (the two identity samples). Keep `KLFDbK` |
| `README.md` | Delete the "Identity Security Platform Credentials" and "Identity Logs Collector" sections |

### Two traps

1. **`identity_host` must stay in `default/vars.yml`.** The *audit* collector's `loginUrl`
   (`default/jobs.yml:205`) uses `C.vars['CyberArk_Hosts'].identity_host` for the OAuth token
   endpoint — that is exactly what check 3.5 grades. Deleting the key because "the identity
   collector is gone" breaks authentication for the flow you are keeping.
2. **`CyberArk_Audit` breaker is dead config but was historically live.** `jobs.yml` references
   `CyberArk_API_Ruleset`; the shipped samples were captured under `CyberArk_Audit:audit_logs`. It is
   unreferenced today, but a customer may have attached it to a source of their own. Leave it in place,
   or remove it at 2.0.0 with an explicit release note.

---

## 4. Breaking-change contract

The compatibility surface of a Cribl pack is **every identifier the customer's own config references
by name**. Rebrand the display layer; freeze the identifier layer.

### 4.1 Freeze — renaming any of these breaks existing deployments

| Identifier | Referenced by | What breaks if renamed |
|:--|:--|:--|
| Pack ID `cribl-cyberark-rest-io` | install namespace; parent routes as `pack:cribl-cyberark-rest-io` | Installs as a **second, separate pack**. Parent routes keep pointing at the old ID, and `savedState.restCollectorState` does not migrate — the collector falls back to `initialValue` (`2026-01-01`) or to `earliest`, producing either a massive re-collect or a silent gap |
| Job ID `cyberark_audit_logs` | `route.yml` filter `__inputId.endsWith('cyberark_audit_logs')`; `source` field; saved state; schedule | Every customer downstream filter, dashboard, and Search query keyed on `__inputId` or `source` stops matching. State is keyed on the job ID, so collection restarts from scratch |
| Breaker `CyberArk_API_Ruleset` | `input.breakerRulesets` in both collectors | Events arrive unbroken. Customers may also have attached it to their own sources |
| Pipeline `cyberark_audit_parsing` | `route.yml` | Route resolves to nothing; events pass through unparsed |
| Variable `CyberArk_Hosts` and keys `audit_host`, `identity_host` | 5 expressions in `jobs.yml` | Any customer who overrode the variable value loses it. All five references must change atomically |
| Secrets `cyberark_api_key`, `cyberark_audit_x_api_key` | live in the customer's **worker group**, outside the pack | Every existing customer must recreate their secrets by hand |
| streamtag `cyberark` | customer streamtag filters | **Add** `idira` and `siem`; do not remove `cyberark` |

### 4.2 Change freely — display only, zero compatibility surface

- `package.json` → `displayName`, `description`, `version`, `author`
- `package.json` → `tags.streamtags` (additive only)
- `default/pack.yml` → `logo` (base64 — swap for the Idira / Palo Alto Networks mark)
- `README.md` → full rewrite
- Collector `description` fields in `jobs.yml` (UI labels, not IDs)
- Route `name` fields in `route.yml` (display, not referenced)
- Function `description` fields in the pipelines

### 4.3 Must NOT be rebranded — Idira's explicit rule

`naming-crosswalk.md` §3: *product names changed; hostnames, header keys, API paths, response
payloads, and role names did not.* The dev guide adds: "These endpoints currently retain the
`cyberark.cloud` domain."

Leave all of these exactly as written:

- `*.audit.cyberark.cloud` and `*.id.cyberark.cloud` in `vars.yml` — do **not** "correct" to an idira domain
- `docs.cyberark.com` documentation links in the README — the portal has not moved
- `x-cybr-telemetry`, `x-api-key`, `Authorization` header keys
- API paths `/api/audits/stream/createQuery`, `/api/audits/stream/results`, `/oauth2/token/<app-id>`
- Scope `isp.audit.events:read`; claim `aud=cyberark.isp.audit`
- The role name **Read Only System Administration** — character for character, or assignment fails
- Response field names (`applicationCode`, `auditCode`, `customData`, …) — payloads were not renamed

### 4.4 Versioning

Removing the identity collector is a breaking removal → **2.0.0**.

**Keeping the pack ID (decision 1) makes the release shape matter more, not less.** A preserved ID
means existing installs upgrade *in place* — so a customer who upgrades inherits the identity
collector's removal silently: the job disappears, its saved state (`latestTime`) is discarded, and any
downstream route of theirs filtering on `cyberark_identity_logs` goes quiet with no error. Had the ID
changed, the old pack would have sat untouched alongside the new one and absorbed that shock. It no
longer does.

Two routes:

- **Recommended — two-step.** `1.1.0`: add telemetry, fix pagination, fix the README role and scope
  errors, **disable** the identity collector (`schedule.enabled: false` is already the shipped
  default) and add a deprecation notice to the README. All non-breaking, and it gives anyone running
  the identity flow a release in which to notice before the job is gone.
  `2.0.0`: delete the identity collector and its artifacts per §3.
- **Single-step.** Straight to `2.0.0` with a migration note. Defensible — the identity flow cannot
  be certified under this kit regardless — but with the pack ID preserved, the removal lands on live
  deployments without warning.

---

## 5. Required work, by checklist item

### 5.1 Telemetry — 7 Required checks (7.1–7.7)

The value is a constant per release: Base64url of
`in=<name>&it=SIEM&iv=<version>&vn=<legal entity>`. Compute once, store in a pack Variable, reference
from headers.

Add the header to all three header arrays in `jobs.yml`:
`discoverRequestHeaders`, `collectRequestHeaders`, and `authRequestHeaders` (the token endpoint is an
Idira Identity API, and the requirement is "every request to an Idira API, including retries").

Two traps from `telemetry-reference.md`:

- **Padded Base64url.** Keep the `=` padding. Node's `.toString("base64url")` strips it and is wrong here.
- **Alphabet.** Assert against test vector 2 (`vn=NTT データ`) — it is the only vector that
  distinguishes Base64url from plain Base64, and plain Base64 passes everything else.

**Action for a human, not code:** check 7.7 requires `in`, `it` and `vn` to be *agreed with the Idira
Technical Alliances contact before submission*. `it=SIEM` is fixed. `in` and `vn` (Cribl's legal
entity name) must be confirmed by email, not chosen here.

### 5.2 Pagination to completion — check 4.6

Today: `collector.conf.pagination.type: none`, and the breaker's `jsonArrayField: data` discards the
`paging` wrapper, so the next-page `cursorRef` is never read. The config already carries an unused
`nextRelationAttribute: next`.

The `results` response is
`{"data": [...], "paging": {"cursor": {"cursorRef": "<next>"}}}`. The collector must pass each
returned `cursorRef` into the next `results` call until the pages are exhausted.

**Verify the exact pagination mode and attribute path against the REST collector UI in your Stream
version before editing the YAML** — do not guess the enum value from the dead `nextRelationAttribute`
key. Test against a tenant generating more than 500 events in one window; a short test passes either way.

### 5.3 README corrections — checks 3.4, 6.1, 9.3

| Current README text | Replace with |
|:--|:--|
| "Create and assign a role (cribl_audit_access) with **Report Management**" | **Read Only System Administration.** Report Management grants no audit data, every query returns zero events, and nothing raises an error |
| "set Allowed REST APIs to `.*`" | Restrict to the audit stream paths actually called. `.*` is graded as a least-privilege failure (6.1) |
| *(absent)* | OAuth2 Server web app: **Token Type `jwtRS256`**, **Client Creds** auth method |
| *(absent)* | Advanced tab claim script — **without it the token is rejected by Audit**:<br>`setClaim('tenant_id', TenantData.Get("CybrTenantID"));`<br>`setClaim('aud', 'cyberark.isp.audit');` |
| *(absent)* | The SIEM API key is **shown once** and cannot be re-displayed. A tenant holds **at most two** third-party SIEM integrations — **this pack consumes one of the customer's two slots** (9.3 grades this disclosure) |
| *(absent)* | The Audit Service API retains only the **previous 7 days** (4.9 grades this statement) |
| *(absent)* | Limits: stream APIs callable **once a minute**; `results` pages at **500** events |

### 5.4 Documentation deliverables — checks 2.4, 5.1, 5.3, 5.4, 9.4

- **2.4 — declare scope.** Which event categories and which Idira services. This also decides whether
  checks 8.2 and 8.3 apply: scoped to Identity Security Platform events only, you certify without a
  PAM SaaS tenant. Scoped to include **Idira Privileged Access Manager SaaS**, you must demonstrate
  credential retrieval and session initiation on real events.
- **9.4 / 5.3 — publish a field-mapping reference.** Idira field → Cribl field. Record the
  asymmetry: `selectedFields` in the request are **snake_case** (`application_code`) while the
  response is **camelCase** (`applicationCode`). Note `timestamp` in milliseconds and `customData`
  as a nested object.
- **5.1 — state the dedup argument.** `uuid` is the stable identifier. Either document how the
  state-tracking window (`state.latestTime + .001`) prevents cross-cycle overlap, or add explicit
  dedup. A reviewer will ask; silence reads as "not handled".
- **5.4 — clarify the null-stripping.** The pipelines drop fields whose *value* is null or empty.
  Say so explicitly, and say that no event and no unmapped field is ever discarded — "redact any
  fields with empty values" currently reads like silent dropping, which is what 5.4 fails.

### 5.5 Recommended, not gating

- **6.2** — add an optional, disabled-by-default Mask function over `customData`, `message` and
  `command` so credential values appearing in a "Fetch credential" event's context are not re-emitted.
- **4.4** — expose `filterModel.applicationCode` as a pack Variable so customers filter server-side
  rather than pulling everything.
- **4.7** — confirm and document `401` / `403` handling. `retryRules.codes` is `[429, 503]`; the
  collector's own re-auth path covers `401` but that needs stating as evidence.

---

## 6. Marketplace listing (stage 5, after certification)

Verbatim from `marketplace-standards.md` — these are fixed values, not free text:

- **Bucket:** Security Analytics (SIEM Integration)
- **Title format:** `<Vendor name> <Vendor product> <Use Case/solution>` — enforced at review
- **Short description (supplied, substitute the placeholder):**
  `Feed Idira Audit logs into Cribl for security analytics.`
- **Idira Solution:** `Security Information & Event Management Integration`
- **Idira Products:** `Idira Identity Security Platform, Audit Service`
- **Overview:** exactly three headings — Business Problem, Solution, Key Features. No others.
- **Documentation asset:** named exactly `Configuration guide`, sentence case, supplied as an
  **HTML link** (a URL), not an uploaded file.
- **Submission:** email to `cybr-bizdevtech@paloaltonetworks.com` with the tracking reference in the
  subject. There is no upload portal.

---

## 7. Non-engineering prerequisites

- **2.5 — a named production or pilot customer** before certification. This is a Required gate and it
  is not something a code change satisfies.
- **§10 — demonstration.** The checklist grades it Required; `journey.md` §6 and
  `security-baseline-reference.md` §3 record it as highly recommended and **non-blocking**, and
  `journey.md` is authoritative. Deliver one anyway — a 2–5 minute video, or a live call.
- **§8 — functional validation on real events.** Synthetic payloads do not count. Check **8.5** is
  the one that is easy to miss: stop the collector, generate events, restart, and prove no gap *and*
  no duplicates. The pack's state tracking should pass this; it has to be demonstrated.
- **6.4 — security scan** reviewed and addressed.
- **§12 — attestation** filled in: partner name, integration name, version, date, contact.

---

## 8. Effort

| Work | Estimate |
|:--|:--|
| Telemetry header (compute value, 3 header arrays, pack Variable, vectors) | ~1 hour |
| Identity collector removal | ~1 hour |
| Branding — package.json, pack.yml logo, descriptions | ~1 hour |
| README rewrite incl. the §5.3 corrections | ~3–4 hours |
| Pagination fix | ~2 hours, plus a live tenant with >500 events/window to verify |
| Field-mapping reference + scope declaration | ~2–3 hours |
| Functional validation §8 on a live tenant, incl. the 8.5 interruption test | ~half a day |

**~2–3 days of focused engineering**, excluding the demo recording and the named-customer prerequisite.

---

## 9. Open decisions

1. ~~**Pack ID.**~~ **DECIDED 2026-11-19 — keep `cribl-cyberark-rest-io`.** The upgrade path stays
   intact and no customer has to recreate secrets or lose collector state. The ID is stale against
   Idira's naming, which is accepted: check 9.5 grades *documentation* terminology, not the pack's
   internal identifier. See §4.4 for the one consequence this creates.
2. **Scope under 2.4.** Identity Security Platform events only, or include Idira Privileged Access
   Manager SaaS? Decides whether checks 8.2/8.3 apply and whether a PAM SaaS tenant is needed.
3. **`in` and `vn` telemetry values** — must be agreed with the Idira Technical Alliances contact
   before submission (7.7).
4. ~~**Release shape**~~ **DECIDED 2026-11-19 — straight to `2.0.0`.** The identity collector and its
   artifacts are deleted in one release, with the removal called out in the README release notes.
   Accepted consequence (§4.4): because the pack ID is preserved, an in-place upgrade removes the
   identity job and discards its saved state without a prior deprecation release. Any customer route
   filtering on `cyberark_identity_logs` goes quiet silently, so the release note must say so
   explicitly — it is the only warning they get.
