----
# Idira REST IO
----

## About this Pack

This Pack ingests platform-wide audit events from the **Idira Audit Service** (Palo Alto Networks
Idira Identity Security Platform) into Cribl Stream, using the documented SIEM Integration API.

It provides one REST collector, its event breaker ruleset, and a route and pipeline for processing
Idira audit events.

The pre-built pipeline removes fields whose **value** is null or empty, to give a general starting
point for event optimization. **No event and no unmapped field is ever discarded** — only empty
values are stripped. Further processing or enrichment may be desired for your use case.

> **Product naming.** Idira was formerly CyberArk. Product names changed; **hostnames, header keys,
> API paths, response payloads and role names did not.** This Pack therefore still uses
> `*.audit.cyberark.cloud` and `*.id.cyberark.cloud` hostnames, the `x-api-key` and
> `x-cybr-telemetry` header keys, the `/api/audits/stream/` paths, and `docs.cyberark.com`
> documentation links. **These are correct — do not change them to an `idira` domain.**

---

## ⚠️ Upgrading from 1.x — read this first

**Version 2.0.0 removes the Identity Security Platform collector (`cyberark_identity_logs`).**

Because the Pack ID is unchanged, an upgrade applies **in place**. On upgrade:

- The `cyberark_identity_logs` collection job is **deleted**, along with its saved state
  (`latestTime`). It will not resume.
- The `cyberark_identity_parsing` pipeline and its route are **deleted**.
- Any route **in your own Worker Group** that filters on `cyberark_identity_logs` will stop matching
  and go quiet **without raising an error**.

If you depend on Identity (`/Redrock/query`) collection, **export that collector's configuration
before upgrading** and recreate it as a standalone Source outside this Pack.

**Why it was removed:** `/Redrock/query` is an Idira *Identity* API, not an Audit Service endpoint.
The Idira Audit Service SIEM certification checklist requires that only documented Audit Service
endpoints are used (checks 4.1 and 4.2), so the Identity flow cannot be certified under this kit.

---

## Scope of audit data collected

*(Certification check 2.4 — confirm this matches your deployment before submission.)*

- **Idira services:** Idira Identity Security Platform, Audit Service
- **Event categories:** all platform-wide audit events returned by the SIEM Integration API, filtered
  only by time window. No `applicationCode` filter is applied by default.

If your deployment also ingests **Idira Privileged Access Manager SaaS** events, say so in your
certification submission — checks 8.2 (credential retrieval) and 8.3 (session initiation) then apply
and must be validated against real events.

---

## API limits you need to plan for

| Limit | Value |
|:--|:--|
| Call rate | The stream APIs may be called **once a minute**. This Pack defaults to every 5 minutes |
| Page size | `results` returns pages of **500** events |
| Retention | The SIEM Integration API returns events from the **previous 7 days only**. An outage longer than 7 days loses events permanently |
| SIEM integrations per tenant | **Two.** This Pack consumes one of your two slots |

---

## Deployment

### Step 1 — Create the Idira OAuth2 Server web app

In **Identity Administration → Manage → Inventory → Targets → Applications → Add Web Apps**, on the
**Custom** tab, add the **OAuth2 Server** web app.

| Tab | Setting |
|:--|:--|
| **Settings** | Enter an **Application ID**. This is the last path segment of the token URL — this Pack expects `cribl_audit_app` |
| **Tokens** | **Token Type** = `jwtRS256`; ensure the **Client Creds** auth method is selected |
| **Scope** | Add a scope named `isp.audit.events:read` |
| **Advanced** | Paste the claim script below and save |

```
setClaim('tenant_id', TenantData.Get("CybrTenantID"));
setClaim('aud', 'cyberark.isp.audit');
```

> **The Advanced claim script is not optional.** Without it the token is issued but **rejected by the
> Audit Service**.

**Restrict Allowed REST APIs** to the endpoints this Pack actually calls. Do **not** set `.*` —
least privilege is a graded requirement (check 6.1).

### Step 2 — Create the dedicated service user

1. **Inventory → Identities → Users → Add User.** Under **Status**, select
   **is OAuth confidential client**. Create the user (for example `svc_cribl_audit`) and
   **note the full UPN** (e.g. `svc_cribl_audit@cyberark.cloud.12345`).
2. In **Users → Sets → All Service Users**, open the user, select **Application Settings**, and add
   the OAuth2 Server web app.
3. Grant the user the **Read Only System Administration** role right.
4. Back on the OAuth2 Server web app, open **Permissions**, add the service user, and select
   **Grant**, **View**, **Run** and **Automatically Deploy**.

> ### ⚠️ The role right decides whether any data comes back
>
> The service user **must** hold **Read Only System Administration** — exactly as written.
>
> **`Report Management` does not grant access to audit event data.** With only Report Management the
> collector authenticates successfully, the connection test passes, and **every query returns zero
> events with no error raised.** If you configured this Pack at version 1.x using Report Management,
> this is why you saw no data.
>
> Grant nothing broader than Read Only System Administration (check 3.4 and check 6.1).

Reference: [Export audit activities to SIEM applications](https://docs.cyberark.com/setup/latest/en/content/setup-space/ecosystem-integration/integrate-siem-main.htm)

### Step 3 — Generate the SIEM API key

You need an administrator whose role has the **Show Export to SIEM in Administration space** right.

1. Go to the **Setup** space → **Integrations → Export to SIEM**.
2. Click **Create → Create SIEM integration**, enter a name and optional description, click **Apply**.
3. **Copy the API key immediately — it is shown once and cannot be re-displayed.**
4. Also copy the **API base URL** shown for the integration. This is your `audit_host`.

A SIEM integration's name and description cannot be edited afterwards, and a tenant can hold at most
**two** third-party SIEM integrations.

Reference: [Integrate Audit with third-party SIEM applications](https://docs.cyberark.com/setup/latest/en/content/siem-integration/siem-export-3rd-party.htm)

---

## Configure Cribl Stream

### Create the Cribl Secrets

Navigate to **Worker Group Settings → Security → Secrets** and create two Secrets.

Secret names are **unchanged from 1.x** so existing deployments do not have to recreate them.

| Name | Type | Value |
|:--|:--|:--|
| `cyberark_api_key` | API Key and Secret Key | The Client ID and Client Secret of the `svc_cribl_audit` service user |
| `cyberark_audit_x_api_key` | Text | The API key from the SIEM integration (Step 3) |

If you name your Secrets differently, update the Secret references in the REST collector to match.

### Set your Idira hosts

**Knowledge → Variables → `CyberArk_Hosts`**

| Key | Value |
|:--|:--|
| `audit_host` | The **API base URL** from the SIEM integration page (form: `<subdomain>.audit.cyberark.cloud`) |
| `identity_host` | Your Idira Identity **tenant ID** host (form: `<tenant-id>.id.cyberark.cloud`). **Not** the friendly URL of your tenant |

> `identity_host` is **still required** even though this Pack no longer collects Identity logs — the
> OAuth2 bearer token must be retrieved from the Idira Identity (`.id.`) endpoint (check 3.5).

### Set your telemetry values

**Knowledge → Variables → `Idira_Telemetry`**

Every certified integration must send the `x-cybr-telemetry` header on every Idira API request
(checks 7.1–7.7). This Pack builds it automatically from this Variable and attaches it to the token,
discover, and collect calls — you only set the values.

| Key | Value |
|:--|:--|
| `integration_name` | Your integration name as it appears in the Marketplace listing |
| `integration_version` | Must match the released Pack version (`2.0.0`) |
| `vendor_name` | Your legal entity name |

`it` is fixed at `SIEM` by the Idira IntegrationType registry and is set in the header expression —
do not change it.

> **Before certification:** `integration_name` and `vendor_name` must be **agreed with your Idira
> Technical Alliances contact** (check 7.7). The shipped defaults are placeholders, not approved
> values. Never use a value containing `&` or `=` — no escaping rule is defined for them.

**Verify the encoded header before you submit.** With the shipped defaults the plaintext and encoded
value are:

```
plaintext : in=Cribl Stream Security Analytics&it=SIEM&iv=2.0.0&vn=Cribl, Inc.
encoded   : aW49Q3JpYmwgU3RyZWFtIFNlY3VyaXR5IEFuYWx5dGljcyZpdD1TSUVNJml2PTIuMC4wJnZuPUNyaWJsLCBJbmMu
```

The encoding is **padded Base64url** (RFC 4648 §5). The expression applies the `+`→`-` and `/`→`_`
replacements; those are a no-op for the values above but are required, because a non-ASCII
`vendor_name` will produce characters that differ between the two alphabets.

### Configure the audit collector

**Sources → REST → `cyberark_audit_logs`**

The job ID is retained as `cyberark_audit_logs` for backward compatibility — your existing route
filters and downstream Search queries keyed on `__inputId` or `source` keep working.

- **Discover**
  - Review the **Discover POST body** and adjust `pageSize` or `selectedFields` as desired. Adding an
    `applicationCode` filter here reduces volume server-side (check 4.4).
  - Set the **x-api-key** header value to: `C.Secret("cyberark_audit_x_api_key", "text").value`
- **Collect**
  - Set the **x-api-key** header value to: `C.Secret("cyberark_audit_x_api_key", "text").value`
- **Authentication**
  - Set the **Client Secret value** to the text: `client_credentials`
  - Set the **Authorization** authentication header value to:
    `"Basic " + C.Encode.base64(C.Secret("cyberark_api_key", "keypair").apiKey + ":" + C.Secret("cyberark_api_key", "keypair").secretKey)`

Leave **Capture headers** set to `false`. Enabling it writes the bearer token and the SIEM API key
into event payloads, which fails check 3.8.

**Time filters** are pre-configured to work with either the *Earliest/Latest* run-time values or full
State Tracking, so no change is normally needed. State Tracking is what makes a transient failure
resume from the last successful position rather than skipping forward (check 4.9). For how State
Tracking works, see the how-to videos on the [Cribl University YouTube](https://www.youtube.com/@Cribl_University/videos) channel.

### Final steps

1. Configure your desired Destination(s).
2. Review the route and pipeline, and transform or enrich events as desired.
3. Enable **Scheduling** on the collector.

---

## Field mapping reference

*(Certification checks 5.3 and 9.4.)*

**The request and the response use different casing.** `selectedFields` in the Discover POST body are
**snake_case**; the fields in the response are **camelCase**. `application_code` is requested,
`applicationCode` is returned.

| Idira response field | Type / notes |
|:--|:--|
| `uuid` | **Stable event identifier.** Used for deduplication (check 5.1) |
| `timestamp` | **Unix epoch in milliseconds** (e.g. `1770080106394`). The Idira docs describe this field as seconds while their own example is milliseconds — the breaker auto-detects the unit. Preserved as emitted, in UTC (check 5.2) |
| `tenantId` | String |
| `applicationCode` | Service code, e.g. `IDP` |
| `serviceName` / `component` | e.g. `Identity` |
| `auditCode` | Per-service audit code, e.g. `IDP1302`. See [Audit events and codes](https://docs.cyberark.com/audit-and-reports/latest/en/content/audit/product-audits/audit_events_and_codes.htm) |
| `auditType` | e.g. `Info` |
| `actionType` / `action` | e.g. `End`, `Outbound provisioning sync completed` |
| `username` / `userId` | Actor |
| `source` | Source IP |
| `message` | Event message key |
| `customData` | **Nested object** of title/value pairs, not a flat string |
| `target`, `targetAccount`, `targetPlatform`, `safe`, `sessionId`, `accountId`, `accountName`, `accessMethod`, `identityType`, `correlationId`, `command`, `cloudProvider`, `cloudAssets`, `cloudIdentities`, `cloudWorkspacesAndRoles` | Present on every event; frequently `null` |

### Two measured caveats

Verified against all 28 events in `data/samples/sample-audit.log`:

1. **Five requested fields are never returned.** `arrival_timestamp`, `checksum`, `cloud_roles`,
   `cloud_workspaces` and `vaulted_accounts` are in `selectedFields` but appear in no response. Do
   not build downstream logic that depends on them.
2. **Two fields are returned without being requested:** `isDr` and `originRegion`. They are passed
   through rather than dropped (check 5.4).

### Deduplication

`uuid` is the stable identifier for deduplication (check 5.1). Cross-cycle overlap is prevented at the
query level: State Tracking advances the window to `state.latestTime + .001` on each run, so a given
event is requested in exactly one collection cycle. If you replay a window manually with
*Earliest/Latest*, deduplicate on `uuid` at your Destination.

---

## Pagination

The `results` response has the shape:

```json
{ "data": [ ...events... ], "paging": { "cursor": { "cursorRef": "<next>" } } }
```

The collector follows that cursor to exhaust every page (certification check 4.6):

```yaml
pagination:
  type: response_body
  attribute:
    - paging.cursor.cursorRef
  maxPages: 0          # 0 = retrieve all pages
  lastPageExpr: ""      # termination handled by stopOnEmptyResults
```

The first `results` call uses the `cursorRef` returned by Discover (`createQuery`); each subsequent
page uses the `cursorRef` from the previous response. Collection stops when a page returns no
events, via `stopOnEmptyResults: true`.

> ### ⚠️ Verify pagination on your first run
>
> Confirm the collector actually makes more than one `results` call before you rely on it — and
> before certification, since check 4.6 is a gate.
>
> 1. Pick a time window you know contains **more than 500** audit events (use *Run* with explicit
>    Earliest/Latest rather than waiting for the schedule).
> 2. Open the job's **Task** logs and count the requests to `/api/audits/stream/results`.
> 3. **Expected:** more than one `results` call, and a total event count above 500.
> 4. **If you see exactly one call and exactly 500 events,** the cursor template variable is not
>    binding. Open the collector's **Collect → POST body** and check which variable name the
>    pagination engine supplies for the `paging.cursor.cursorRef` attribute — the body currently
>    references `${cursorRef}`, and on some Stream versions the engine exposes the full attribute
>    path instead. Adjust the body or add the mapped name to the pagination configuration.
>
> This fails safe: if the binding is wrong you collect one page per run, which is the 1.x behaviour,
> not corrupted data.

---

## Known limitations

| # | Limitation |
|:--|:--|
| 1 | The `CyberArk_Audit` breaker ruleset is retained but **unused** — `cyberark_audit_logs` uses `CyberArk_API_Ruleset`. It is kept in case an existing deployment attached it to a Source of its own |

---

## Upgrades

Upgrading certain Cribl Packs using the same Pack ID can have unintended consequences. See
[Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading) for details, and read the
**Upgrading from 1.x** warning at the top of this file.

## Release Notes

### Version 2.0.0 — 2026-11-19

**Breaking**
- **Removed the Identity Security Platform collector** (`cyberark_identity_logs`), its pipeline
  (`cyberark_identity_parsing`), its route, its breaker rules, and its samples. `/Redrock/query` is an
  Idira Identity API rather than an Audit Service endpoint and cannot be certified under the Audit
  Service SIEM kit (checks 4.1, 4.2). **Saved state for that job is discarded on upgrade and any
  Worker Group route filtering on `cyberark_identity_logs` goes quiet without error.**

**Added**
- `x-cybr-telemetry` header on the token, discover and collect calls, built from the new
  `Idira_Telemetry` Variable (checks 7.1–7.7).
- **Cursor pagination.** `pagination.type` changed from `none` to `response_body`, following
  `paging.cursor.cursorRef` with `maxPages: 0`. 1.x collected only the first 500-event page per run
  and silently discarded the remainder (check 4.6). See **Pagination** above for the one-run
  verification step.
- Field mapping reference, including the snake_case/camelCase asymmetry and the five requested fields
  the API never returns (checks 5.3, 9.4).
- Scope-of-collection statement (check 2.4), API limits, 7-day retention note (check 4.9), and the
  two-SIEM-integrations-per-tenant disclosure (check 9.3).

**Fixed**
- **Documentation named the wrong role right.** 1.x instructed `Report Management`, which grants no
  access to audit event data — every query returns zero events with no error. The required right is
  **Read Only System Administration** (check 3.4).
- Documentation instructed setting **Allowed REST APIs** to `.*`; it now instructs restricting to the
  endpoints called (check 6.1).
- Added the OAuth2 web app configuration that 1.x omitted: token type `jwtRS256`, the Client Creds
  auth method, and the Advanced-tab claim script, **without which the token is rejected by the Audit
  Service** (check 9.3).

**Unchanged (deliberately — backward compatibility)**
- Pack ID `cribl-cyberark-rest-io`, job ID `cyberark_audit_logs`, pipeline `cyberark_audit_parsing`,
  breaker rulesets `CyberArk_API_Ruleset` and `CyberArk_Audit`, Variable `CyberArk_Hosts`, and the
  Secret names `cyberark_api_key` and `cyberark_audit_x_api_key`.

### Version 1.0.0 — 2026-02-09
Initial Release

### Version 0.9.3 — 2026-02-07
Internal beta

---

## Certification status

This Pack targets the Idira `audit-service/siem-integration` development kit
(`it` value `SIEM`). It is **not yet certified**. Outstanding items are tracked in
[`IDIRA-MIGRATION-PLAN.md`](IDIRA-MIGRATION-PLAN.md).

Nothing in this Pack is Idira-approved; certification is a human review.

## Contributing to the Pack
To contribute to this Pack, or to report any issues or enhancement requests, please connect with
Kelsey Prior (cribl.io) or Randy Corelli (cribl.io) on
[Cribl Community Slack](https://cribl-community.slack.com).

## Contact
Email <kprior@cribl.io> or <rcorelli@cribl.io>, or find us on
[Cribl Community Slack](https://cribl-community.slack.com).

## License
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE).
