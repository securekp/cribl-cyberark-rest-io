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

### Set your Pack Variables

**Knowledge → Variables**

**Everything you need to configure lives here.** All five Variables ship pre-made with working
defaults or example syntax, and the collector reads them — so you set values in one place and never
hand-edit a JS expression inside the collector.

#### `CyberArk_Hosts` — required

| Key | Value |
|:--|:--|
| `audit_host` | The **API base URL** from the SIEM integration page (form: `<subdomain>.audit.cyberark.cloud`) |
| `identity_host` | Your Idira Identity **tenant ID** host (form: `<tenant-id>.id.cyberark.cloud`). **Not** the friendly URL of your tenant |

> `identity_host` is **still required** even though this Pack no longer collects Identity logs — the
> OAuth2 bearer token must be retrieved from the Idira Identity (`.id.`) endpoint (check 3.5).

#### `Idira_Secrets` — change only if you renamed your Secrets

Holds the **names** of your Cribl Secrets, never the values. Defaults match the names used since
1.0.0, so an existing deployment needs no change here.

| Key | Default | What it points at |
|:--|:--|:--|
| `api_key` | `cyberark_api_key` | The *API Key and Secret Key* Secret holding the OAuth2 client ID and secret |
| `siem_api_key` | `cyberark_audit_x_api_key` | The *Text* Secret holding the SIEM API key |

If you named your Secrets differently, change them **here** — not in the collector.

#### `Idira_OAuth` — required

| Key | Default | Value |
|:--|:--|:--|
| `app_id` | `cribl_audit_app` | The **Application ID** from your OAuth2 Server web app's Settings tab. Must match exactly, or the token request 404s |
| `scope` | `isp.audit.events:read` | The scope you added on the Scope tab. Leave as-is unless Idira documents otherwise |

#### `Idira_Collection` — optional tuning

| Key | Default | Value |
|:--|:--|:--|
| `page_size` | `500` | Results page size. **500 is the API maximum** |
| `application_code` | `[]` | Optional server-side service filter (check 4.4). Empty array collects every service. Add codes — e.g. `["DPA"]` — to cut volume before it leaves Idira |

The `application_code` filter is wired conditionally: an empty array (or a missing key) sends the
date filter alone, exactly as 1.x did. Populating it adds an `applicationCode` include filter to the
`createQuery` body.

#### `Idira_Telemetry` — required before certification

Every certified integration must send the `x-cybr-telemetry` header on every Idira API request
(checks 7.1–7.7). The Pack builds it from this Variable and attaches it to the token, discover and
collect calls — you only set the values.

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

**Most of the collector configures itself from the Pack Variables.** These fields survive Pack
export and need no attention:

| Field | Resolves to |
|:--|:--|
| Token URL | `` `https://${...identity_host}/oauth2/token/${...app_id}` `` |
| **Token request body** — `grant_type`, `scope`, `client_id`, `client_secret` | from `Idira_OAuth` and `Idira_Secrets` |
| Discover `pageSize` and `applicationCode` filter | from `Idira_Collection` |
| Collect / Discover URLs | from `CyberArk_Hosts.audit_host` |

**The OAuth client credentials need no post-import entry.** The collector uses `authentication:
login`, which puts them in `loginBody` — a field Cribl's exporter preserves. See
*Why the token request uses `loginBody`* below.

**Five fields do not survive export and must be re-entered after import — see the next section.**

Leave **Capture headers** set to `false`. Enabling it writes the bearer token and the SIEM API key
into event payloads, which fails check 3.8.

---

## Why the token request uses `loginBody`

Cribl offers two ways to do OAuth2 `client_credentials` on a REST collector:

| Mode | Credentials live in | Survives Pack export? |
|:--|:--|:--|
| `authentication: oauth` | `authRequestHeaders` → `Authorization: Basic …` | **No** — scrubbed to `changeme` |
| `authentication: login` | `loginBody` → `client_id` / `client_secret` params | **Yes** |

This Pack uses **`login`**, so your customers never paste a credential expression after import.

Both are OAuth 2.0 `client_credentials` (check 3.1) and both send client ID and secret — RFC 6749
§2.3.1 permits either HTTP Basic or request-body parameters. The resolved request matches the one
Idira documents, on the same `.id.` endpoint (check 3.5):

```
POST https://<tenant-id>.id.cyberark.cloud/oauth2/token/<app-id>
grant_type=client_credentials&scope=isp.audit.events:read&client_id=…&client_secret=…
```

The `username` and `password` fields are **unused** — they exist only to satisfy the collector form's
required-field validation, and they stay `changeme`. The real credentials come from the
`Idira_Secrets.api_key` Secret via `loginBody`.

> **If the token request returns `401` or `invalid_client`,** your Idira OAuth2 app requires HTTP
> Basic client authentication rather than body parameters. Roll back by setting **Authentication** to
> `OAuth` and restoring these four fields:
>
> - Client Secret param name: `grant_type`, value: `client_credentials`
> - Auth request **param** `scope`: `C.vars['Idira_OAuth'].scope`
> - Auth request **header** `Authorization`:
>   `"Basic " + C.Encode.base64(C.Secret(C.vars['Idira_Secrets'].api_key, 'keypair').apiKey + ":" + C.Secret(C.vars['Idira_Secrets'].api_key, 'keypair').secretKey)`
>
> That restores 1.x behaviour, at the cost of two extra post-import pastes.

---

## ⚠️ After you import the Pack — 5 fields, 2 values

**Cribl's Pack exporter replaces request-header values and client-secret fields with `changeme`.**
This is deliberate — it stops credentials leaking into a distributed Pack — and it is not something
this Pack can configure around. `safeHeaders` does **not** exempt a header from it.

The exporter scrubs exactly these, and nothing else:

- `discoverRequestHeaders[].value`
- `collectRequestHeaders[].value`
- `authRequestHeaders[].value`
- `username`, `password`, `clientSecretParamValue`

URLs, request **bodies**, and request **params** are preserved, which is why everything in the table
above survives.

Five fields are affected, but only **two distinct strings** — both `x-api-key` headers take the same
value, and all three `x-cybr-telemetry` headers take the same value.

`username` and `password` are also scrubbed. **Leave them as `changeme`** — they are unused.

### Value 1 — the SIEM API key header

Paste into the `x-api-key` header on **Discover** and on **Collect**:

```
C.Secret(C.vars['Idira_Secrets'].siem_api_key, 'text').value
```

### Value 2 — and the one people miss: `x-cybr-telemetry`

`x-cybr-telemetry` is a header, so **it is scrubbed too — in all three places.** It is also seven
Required certification checks (7.1–7.7), and a scrubbed value fails them silently: the collector
still runs and still collects events.

Paste this identical string into the `x-cybr-telemetry` header on **Discover**, on **Collect**, and
on **Authentication**:

```
C.Encode.base64(`in=${C.vars['Idira_Telemetry'].integration_name}&it=SIEM&iv=${C.vars['Idira_Telemetry'].integration_version}&vn=${C.vars['Idira_Telemetry'].vendor_name}`).replace(/\+/g,'-').replace(/\//g,'_')
```

It reads from the `Idira_Telemetry` Variable, so once pasted you change the name, version and vendor
in Knowledge → Variables and never touch the header again.

**Verify it before certification.** Decode the header from a captured request and confirm it says
what you meant — with the shipped defaults it is:

```
plaintext : in=Cribl Stream Security Analytics&it=SIEM&iv=2.0.0&vn=Cribl, Inc.
encoded   : aW49Q3JpYmwgU3RyZWFtIFNlY3VyaXR5IEFuYWx5dGljcyZpdD1TSUVNJml2PTIuMC4wJnZuPUNyaWJsLCBJbmMu
```

### Checklist

| # | Where | Field | Paste |
|:--|:--|:--|:--|
| 1 | Discover | `x-api-key` | Value 1 |
| 2 | Discover | `x-cybr-telemetry` | Value 2 |
| 3 | Collect | `x-api-key` | Value 1 |
| 4 | Collect | `x-cybr-telemetry` | Value 2 |
| 5 | Authentication | `x-cybr-telemetry` | Value 2 |
| — | Authentication | `username`, `password` | leave as `changeme` — unused |

> **Note for maintainers.** The Git repository keeps the real expressions in these fields rather than
> `changeme`, so a Pack installed directly from source works without this step. Only the **exported**
> `.crbl` artifact carries `changeme`. If you re-export and commit, expect these five header values
> to flip back to `changeme` in the diff — that is the exporter, not a regression.

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
- **Three new pre-made Pack Variables** — `Idira_Secrets` (Cribl Secret *names*), `Idira_OAuth`
  (`app_id`, `scope`) and `Idira_Collection` (`page_size`, optional `application_code` filter) —
  joining `CyberArk_Hosts` and `Idira_Telemetry`. The collector's credential, token-URL, scope and
  page-size fields now read from them, so configuration is set in Variables rather than by pasting
  JS into collector fields that previously shipped as `changeme`.
- Field mapping reference, including the snake_case/camelCase asymmetry and the five requested fields
  the API never returns (checks 5.3, 9.4).
- Scope-of-collection statement (check 2.4), API limits, 7-day retention note (check 4.9), and the
  two-SIEM-integrations-per-tenant disclosure (check 9.3).
- *After you import the Pack* section — Cribl's exporter scrubs request-header values to `changeme`,
  so five fields need re-entry on every install. Telemetry is called out separately because a
  scrubbed value fails seven Required checks while the collector keeps working.

**Changed**
- **Token request now uses `authentication: login` rather than `oauth`.** The OAuth2 client ID and
  secret moved from the `Authorization` auth header into `loginBody`, because Cribl's Pack exporter
  scrubs header values and preserves bodies. Same `client_credentials` grant, same `.id.` token
  endpoint, same scope — only the client-authentication style changes (RFC 6749 §2.3.1 permits
  either). Post-import field entry drops from 7 fields to 5, and from 4 distinct values to 2.
  Rollback instructions are in **Why the token request uses `loginBody`**.

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
