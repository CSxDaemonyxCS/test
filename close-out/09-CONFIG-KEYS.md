# 09 — Configuration Keys

**Date:** 24 August 2026 · **Revised:** 25 August 2026 · **Scope:** every runtime key implied by the 19 close-out items, plus the platform-owner web console.

> ### ⚠️ Authority update — 25 August 2026
> **Every `BLOCKED ON` marker in this file is gone.** `CO-D1`–`CO-D5` are decided and all six flags are answered in `12-FINAL-DECISIONS-AND-WEB-ADMIN.md`, which is read above this file. What used to be `D1`–`D5` here is now **`CO-D1`–`CO-D5`** (`12` §12.0).
> **Convention, replacing the old one:** a key no longer carries branch values. Each key carries **one effective value**, the item that decided it, and — where the value is an operational figure rather than a constant — the **named place and date** where it is next checked. A key whose value is chosen at the Phase 0 entry gate says so explicitly and is listed in §9.12 as a boot assertion, so it cannot be shipped empty.
> **New in this revision:** §9.13 (web console) and §9.14 (app ↔ console coupling). Both exist because a public admin surface was added; that surface is residual risk **`R4`** in `04`.

**Legend — `Boundary` column:** `SEC` = a wrong value weakens a security control · `OPS` = a wrong value degrades operation only · `LEGAL` = a wrong value creates a compliance misstatement.

---

## 9.1 Tenancy and roles

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_ROLE_MODEL` | enum | **`platform_scoped`** — Super Admin is the platform owner only (`CO-D5`) | Auth module, at role resolution | **SEC** |
| `MTM_SIGNUP_LANDS_AS` | enum | **`main_admin`** — whoever registers a team becomes Main Admin, never Super Admin (`CO-D5`) | Tenant creation endpoint | **SEC** |
| `MTM_RLS_FORCE` | bool | `true` — never configurable to `false` in any environment, including local | Migration assertion, and a CI check that fails the build | **SEC** |
| `MTM_APP_DB_ROLE` | string | `app_role` — must not hold `BYPASSRLS`; must never be the table owner | Connection pool init | **SEC** |
| `MTM_PLATFORM_READ_DB_ROLE` | string | `platform_ops` — `BYPASSRLS` **plus** `default_transaction_read_only = on`; holds no DEK | Break-glass session | **SEC** |
| `MTM_PLATFORM_WRITE_DB_ROLE` | string | `platform_writer` — **new in this revision.** Write grants on exactly five tables: `tenants`, `subscriptions`, `plan_limits`, `feature_flags`, `platform_access_grants`. Nothing else, ever | Console write path | **SEC** |
| `MTM_APP_DB_URL` | url | libpq connection string for the request pool. **Must resolve to `MTM_APP_DB_ROLE`**, asserted by `SELECT current_user` at pool init and again in the Phase 0 gate: a URL that lands on the table owner or on a superuser makes every isolation test read **green while proving nothing**, and no amount of naming discipline detects that. Injected as a secret, never committed, `sslmode=verify-full` | Connection pool init | **SEC** |
| `MTM_MIGRATION_DB_URL` | url | Connection string used **only** by the migration runner and by the gate's catalog inspection. There is deliberately **no `app_owner` URL**: `app_owner` is `NOLOGIN`, so migrations connect as an administrative role and take ownership with `SET LOCAL ROLE app_owner`. Kept separate from `MTM_APP_DB_URL` so the request pool never holds DDL rights | Migration runner; Phase 0 gate | **SEC** |
| `MTM_TENANT_CONTEXT_MODE` | enum | `set_local` — set inside the request transaction via `nestjs-cls`; required because PgBouncer breaks session-scoped settings | Every request | **SEC** |

> A CI test asserts that `MTM_APP_DB_ROLE` is neither the owner of any table in `app` nor a holder of `BYPASSRLS`. The table owner bypasses RLS **implicitly**, with no grant to inspect, which makes this the single easiest way to silently disable tenant isolation.
>
> A second CI test asserts that `platform_writer` can write to **exactly** the five tables above and fails the build on any sixth. The console exists to administer the platform, not to reach into a tenant's operational data, and that boundary is worth a test rather than a paragraph.

> **`MTM_APP_DB_URL` and `MTM_MIGRATION_DB_URL` are a declared addition (26 August 2026), not original rows.** This file named all six roles but no way to reach any of them, which left the Phase 0 gate reading a placeholder. **Naming rule, so no later step has to invent one:** every `MTM_<SCOPE>_DB_ROLE` pairs with an `MTM_<SCOPE>_DB_URL` of the same scope — `MTM_PLATFORM_READ_DB_URL`, `MTM_PLATFORM_WRITE_DB_URL`, `MTM_BACKUP_DB_URL`. Each is added **when the code path that reads it is built**, not before: a key with no reader is documentation, and this file is not documentation. `ai_reader` gets no URL yet because §9.1 names no role key for it either; both arrive with the `analytics` views in Phase 9.
>
> Every URL carries a password, so it is **never printed**: the gate redacts the credential before any failure message reaches a CI log, and the same redaction covers connection **errors**, which is where a DSN most often leaks.

---

## 9.2 Cryptography and key custody

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_KMS_PROVIDER` | enum | **`managed`** — a managed KMS where the KEK is created inside and is non-exportable (`CO-D2`). **No vendor is named in this package**: every outbound host was blocked, so naming a product would mean citing a page I never opened. The provider is chosen at the Phase 0 gate against the six acceptance conditions in `12` §12.1.2 | Boot, `KeyProvider` factory | **SEC** |
| `MTM_KMS_KEK_ID` | string | Provider-specific key identifier. **Set at the Phase 0 gate**, asserted non-empty at boot | Boot | **SEC** |
| `MTM_KMS_ENDPOINT` | url | **Empty** — a managed KMS is reached through the provider SDK, not a configured address | Boot | **SEC** |
| `MTM_KMS_REENCRYPT_ON_ROTATION` | bool | **`true`** — and this is the key most likely to be misunderstood. **Rotating a KEK does not re-wrap existing DEKs.** Old DEKs stay wrapped under the old key version, so rotation without an explicit `ReEncrypt` pass leaves the old version load-bearing forever. Acceptance condition 3 requires this operation to be **executed once on a test key before Phase 0 closes**, not merely documented | Rotation job | **SEC** |
| `MTM_KEK_SECONDARY_REGION` | string | The region holding the KEK copy (`CO-D3`). **Never empty**: losing the KEK means permanently unrecoverable data (Risk 3). This is a key-custody path, not a data path — it does not widen data residency | Key-backup job | **SEC** |
| `MTM_AUDIT_SIGNING_KEY_ID` | string | A **separate** key from the KEK, usage restricted to signing **by policy**, not by naming convention (acceptance condition 4) | Hourly Merkle root job | **SEC** |
| `MTM_COLUMN_CIPHER` | enum | `aes_256_gcm` — applied at the **application** layer. `pgcrypto` is rejected: it would put the key inside the database, defeating the purpose of encrypting a column stored in that database | Encryption service | **SEC** |
| `MTM_BLIND_INDEX_HMAC_KEY_ID` | string | Separate key. Enables exact-match lookup on `patients.name` and `members.national_id_enc` without decryption | Search path | **SEC** |
| `MTM_PASSWORD_HASH` | enum | `argon2id` | Auth module | **SEC** |

> **Fallback, with a trigger rather than a hope:** if the chosen provider fails acceptance condition 1 (KEK non-exportable) or condition 3 (a real `ReEncrypt` operation), the design falls back to the alternative in roadmap §13.7. The check happens **at the Phase 0 entry gate**, before any migration runs, because a KEK decision made after data exists is the most expensive reversal in this system.

---

## 9.3 Data residency, backup, recovery

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_DATA_REGION` | string | **One named region** (`CO-D3`), named verbatim in §21.4 and in the DPA. **Must match the contract text exactly.** Chosen at the Phase 0 gate; asserted non-empty at boot | Boot; printed in the platform-health view | **LEGAL** |
| `MTM_BACKUP_MODE` | enum | **`daily_wal`** — daily full backup plus continuous WAL archiving, with the KEK replicated to `MTM_KEK_SECONDARY_REGION` (`CO-D3`) | Backup job | **SEC** |
| `MTM_DECLARED_RPO_MINUTES` | int | **`1440`** for the daily backup, **`5`** for the WAL tail. Declared in the security questionnaire as a **range, not a single flattering number** | Documentation assertion | **LEGAL** |
| `MTM_DECLARED_RTO_HOURS` | int | **Hours, not minutes.** A single-region single-operator restore is a manual procedure; claiming minutes would be a false statement (`CO-D3`). Upgrade trigger: the first tenant that contractually requires RTO < 1 hour, **or** 250 paying tenants — checked in the contract review checklist | Documentation assertion | **LEGAL** |
| `MTM_BACKUP_DB_ROLE` | string | `backup_role` — configured with `row_security = off` as a **tripwire**: that setting does not bypass row security, it raises an error if any query's rows would be filtered, so a dump run under the wrong role **fails loudly** instead of quietly emitting a partial backup | Backup job | **SEC** |
| `MTM_WORM_RETENTION_DAYS` | int | Set longer than one full backup cycle. Directly bounds how far back R3 exposure reaches | Audit archival | **SEC** |
| `MTM_RESTORE_TEST_DATES` | list | `2026-10-01, 2027-01-01, 2027-04-01, 2027-07-01` — each run **must** include decrypting at least one encrypted column. A restore that was never tested is not a backup | Ops runbook reminder | **OPS** |

---

## 9.4 Sync

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_SYNC_ENGINE` | enum | **`cursor_delta`** — the opaque-cursor delta protocol already specified in roadmap §8.6. **PowerSync is rejected** (`12`/`FLAG-1`): no authority document mentions it, it introduces a second authorisation path to Postgres outside the request transaction, and it adds a sub-processor to the DPA — widening the one item that blocks sales (`G4`) | Boot | **SEC** |
| `MTM_SYNC_CACHE_EXCLUDE` | list | `patients.condition` — **non-negotiable, and now enforceable as a config assertion** rather than as a reviewed third-party rule set, because the sync engine is mine. This is the concrete payoff of the `FLAG-1` decision | Sync query builder | **SEC** |
| `MTM_LOCAL_KEY_TTL_HOURS` | int | Bounds R1 exposure. Lowering it is the cheapest R1 mitigation; the cost is shorter offline operation, not money | Local key check on app open | **SEC** |
| `MTM_SYNC_POLL_INTERVAL_SECONDS` | int | Polling interval. **Push is a signal, pull is truth** — a push notification only accelerates a pull that would have happened anyway, so no state change ever depends on a notification arriving | Client scheduler | **OPS** |
| `MTM_IDEMPOTENCY_TTL_HOURS` | int | Retention for idempotency keys on mutating sync operations | Sync endpoint | **OPS** |

---

## 9.5 Attendance lock

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_ATTENDANCE_LOCK_NAME` | string | **`attendance_lock`** (`CO-D4`). The name `d16_lock` is retired: **no source document defines what the 16 meant.** The name appears in audit-log rows, so it is fixed before any data exists, not after | Audit writer | **OPS** |
| `MTM_ATTENDANCE_LOCK_SELF_GRACE_MINUTES` | int | **`60`** — the one value carried verbatim from a source document (detachments file §3, "1h grace"). Checked **client and server**, server authoritative | Lock check | **OPS** |
| `MTM_ATTENDANCE_LOCK_EDIT_WINDOW_DAYS` | int | **`7`** calendar days from the occurrence date. **Derived, not invented:** the only scheduling surface in the product is the weekly schedule (§9), so the correction window is exactly one full weekly operational cycle. Per-tenant configurable | Lock check | **SEC** |
| `MTM_ATTENDANCE_LOCK_EDIT_WINDOW_MAX_DAYS` | int | **`30`** — a hard platform ceiling. A tenant configuration above it is **rejected**, not clamped silently. Without a ceiling a tenant could turn the attendance record into an indefinitely editable document | Config validation | **SEC** |
| `MTM_ATTENDANCE_LOCK_EDIT_CAPABILITY` | string | **`recordAttendance`** — held at that detachment's scope. **No sixth capability is invented:** the grants stay five (`recordAttendance`, `registerPatients`, `managePeople`, `manageStructure`, `viewStats`), and what the detachments file calls "permission override" is read as holding this one at that scope | Guard | **SEC** |
| `MTM_ATTENDANCE_CORRECTIONS_APPEND_ONLY` | bool | **`true`** — after day 7 there is **no overwrite at all**. A correction is an additional row in `attendance_corrections` referencing the original, and the original is never destroyed. Same append-only principle as `stock_movements` (§10.2) | Correction endpoint | **SEC** |
| `MTM_ATTENDANCE_CORRECTION_ROLE` | enum | **`main_admin`** — creating a post-window correction is a tenant-level accountability act, so it is bounded by **role**, not by a detachment-scoped grant | Guard | **SEC** |

> **Naming, so a wildcard reference resolves unambiguously:** every key governing the lock itself sits under **`MTM_ATTENDANCE_LOCK_*`** (five keys above), and the two keys governing the corrections table sit under **`MTM_ATTENDANCE_CORRECTION*`**. `01` and `12` refer to this family as `MTM_ATTENDANCE_LOCK_*`; that glob covers the lock keys and **not** the corrections keys, which is correct — the corrections table is what exists *after* the lock, not part of it.

> Every edit inside the 7-day window writes an audit row containing the actor, the value before, and the value after. **What this does and does not do, stated precisely:** the short window and the corrections table **narrow** what can later be misrepresented. They do **not** close `R3`, which is about the operator's ability to rebuild and re-sign the chain, and which stays open until `L4` ships. Any wording that says "solved" is a documentation error.

---

## 9.6 Attestation, pinning, resilience

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_DISTRIBUTION_CHANNEL` | enum | **`official_stores`** — Google Play **and** Apple App Store, both in v1 (`CO-D1`, `12`/`FLAG-5`). Sideloading is not a supported channel. Upgrade path to a dual model exists but is not built: its trigger is the first tenant that contractually forbids public stores, checked in the contract review checklist | Build config | **SEC** |
| `MTM_ATTEST_REQUIRE_APP_RECOGNITION` | bool | **`true`** (Android) — enforceable precisely because distribution is store-only. Off-store this verdict is always `UNRECOGNIZED_VERSION`, which is why the two decisions had to be made together | Attestation guard | **SEC** |
| `MTM_ATTEST_REQUIRE_LICENSING` | bool | **`true`** (Android) — off-store the verdict is always `UNLICENSED` | Attestation guard | **SEC** |
| `MTM_ATTEST_REQUIRE_APP_ACCESS_RISK` | bool | **`true`** (Android) — available on the store path; not evaluated off-store | Attestation guard | **SEC** |
| `MTM_ATTEST_REQUIRE_DEVICE_INTEGRITY` | bool | `true` — plus `MEETS_STRONG_INTEGRITY` required for break-glass-adjacent actions | Attestation guard | **SEC** |
| `MTM_ATTEST_TREAT_UNEVALUATED_AS_FAIL` | bool | `false` — `UNEVALUATED` is not a failure signal; treating it as one locks out legitimate devices | Attestation guard | **SEC** |
| `MTM_ATTEST_IOS_MODE` | enum | **`app_attest`** — `DCAppAttestService`. **Declared asymmetry, not a gap to be papered over:** the App Attest reference makes **no jailbreak, root, or tampering claim**, and that absence is itself **undeclared** by the vendor — I did not open the page (`07`). So iOS and Android guarantees under MASVS-RESILIENCE are **not equivalent**, and no wording may imply they are | Attestation guard | **SEC** |
| `MTM_PINNING_LAYER` | enum | `dart` — fixed. An Android `network_security_config.xml` `<pin-set>` does **not** cover `dart:io`, which ships its own BoringSSL, so a pin declared only there is a pin that does not exist | HTTP client init | **SEC** |
| `MTM_PIN_SPKI_PRIMARY` / `_BACKUP` | string | SPKI hashes. **`_BACKUP` must be shipped before it is needed** — a wrong or missing pin bricks every install (Risk 4) | HTTP client init | **SEC** |
| `MTM_PIN_KILL_SWITCH_ENABLED` | bool | `true` — the only recovery path from a bad pin | HTTP client init | **SEC** |
| `MTM_MIN_SUPPORTED_APP_VERSION` | semver | Returned by the API on every authenticated call. A client below it shows a **forced-upgrade screen** and makes no further calls. **This mechanism is only trustworthy because `MTM_DISTRIBUTION_CHANNEL = official_stores`** — a forced upgrade the user cannot obtain is a lockout, and store distribution is what guarantees the newer build is actually reachable | Client bootstrap | **SEC** |

> Recorded as design intent, not a key: `--obfuscate --split-debug-info` is **friction under MASVS-RESILIENCE, not a control**, and enum names are not obfuscated. No security property may be argued from it. This is exactly why console-code exclusion from the mobile bundle is verified by a **bundle inspection** in §9.12 rather than by obfuscation.

---

## 9.7 Scanner (GS1)

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_SCANNER_ENABLED` | bool | `false` until the C1 protocol in `06` **passes**. No exception | Feature flag | **OPS** |
| `MTM_GS1_ALLOWED_AIS` | list | `01, 10, 17, 21` — anything else is rejected, never interpreted | Parser | **SEC** |
| `MTM_GS1_REQUIRE_FNC1` | bool | `true` — this is the only discriminator between a GS1 symbol and an unrelated DataMatrix | Parser | **SEC** |
| `MTM_GS1_REQUIRE_CATALOGUE_MATCH` | bool | `true` — decoded GTIN must exist in **this tenant's** catalogue, checked under RLS | Parser, post-decode | **SEC** |
| `MTM_GS1_MAX_ATTEMPTS` | int | `3`, then fall through to manual entry | Scanner UI | **OPS** |

> `G2` stays **PARTIAL** and `C1` stays unmeasured. Neither blocks a phase. The guarantee here is behavioural, not bibliographic: the parser rejects what it does not recognise, so an unverified AI number cannot become a silent misread.

---

## 9.8 Staffing threshold

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_MIN_STAFF_ENFORCEMENT` | enum | `alert_only` — per C1's sibling decision C2. `block` becomes permissible only when the qualifications model has shipped (Phase 10) **and** `min_staff` is set on ≥80% of active templates | Shift validation | **OPS** |
| `MTM_MIN_STAFF_ALERT_HOURS` | list | `48, 24, 6` | Notification scheduler | **OPS** |

---

## 9.9 Platform operations and break-glass

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_BREAKGLASS_TTL_MINUTES` | int | `60`; lowering to `15` is the cheapest R2 mitigation | Grant issuance | **SEC** |
| `MTM_BREAKGLASS_REQUIRE_JUSTIFICATION` | bool | `true` — free text, non-empty, stored on the grant row | Grant issuance | **SEC** |
| `MTM_BREAKGLASS_REQUIRE_STEPUP_MFA` | bool | `true` — on the console this means a **second passkey assertion**, not a TOTP code | Grant issuance | **SEC** |
| `MTM_BREAKGLASS_AUDIT_EVERY_QUERY` | bool | `true` — auditing the **grant** alone proves access was authorised, not what was read | Session hook | **SEC** |
| `MTM_BREAKGLASS_NOTIFY_TARGET` | enum | **`tenant_main_admin`** (`CO-D5`) — a party genuinely independent of me, which is what makes the oversight mean anything | Grant issuance | **SEC** |
| `MTM_BREAKGLASS_NOTIFY_CHANNELS` | list | **`email, in_app, audit_row`** — three channels, because a notification that can fail silently is not oversight. The `audit_row` channel is the one that cannot fail: **push is a signal, pull is truth**, and the app pulls active grants on every open | Grant issuance | **SEC** |
| `MTM_PLATFORM_OPS_READ_ONLY` | bool | `true`, backed by `default_transaction_read_only = on` on the role itself, so the guarantee lives in the database rather than in application code | Role definition | **SEC** |
| `MTM_BREAKGLASS_MONTHLY_ALERT_THRESHOLD` | int | `15` — feeds the R2 mitigation. Counted from `platform_access_grants` | Monthly job, 1st of month | **OPS** |

---

## 9.10 Audit integrity

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_AUDIT_MERKLE_INTERVAL_MINUTES` | int | `60` | Root-signing job | **SEC** |
| `MTM_AUDIT_EXTERNAL_TIMESTAMP_ENABLED` | bool | `false` in v1. **Activation trigger (`L4`, unchanged by any decision here):** the first written tenant request **or** 150 paying tenants, whichever comes first — counted on the 1st of each month from the subscriptions table | Root-signing job | **SEC** |
| `MTM_AUDIT_TSA_URL` | url | Empty until the key above turns `true`. RFC 3161 | Root-signing job | **SEC** |

> Public transparency-log style anchoring (RFC 9162) is **out of scope for a solo operator and is not claimed**. `R3` stays open and declared until `L4` ships.

---

## 9.11 Billing, trial, growth triggers

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_TRIAL_DAYS` | int | `3` | Signup | **OPS** |
| `MTM_TRIAL_KILL_SWITCH` | bool | Platform-owner control, exercised from the console | Platform settings | **OPS** |
| `MTM_PLAN_PRICES_USD` | list | **`10, 20, 30, 50`** — Blueprint §11 prices stand for v1 (`12`/`FLAG-3`) | Billing module | **OPS** |
| `MTM_PAYMENT_GATEWAY_ENABLED` | bool | `false`. Turning it on is what puts the system **into PCI scope**, which is why the trigger is a measured workload figure and not a feeling | Billing module | **LEGAL** |
| `MTM_MANUAL_ACTIVATION_MONTHLY_THRESHOLD` | int | `40` — the `L2` trigger, counted from activation rows, checked on the 1st of each month. **Deliberately workload-based, not price-based**, so changing a price never silently invalidates the trigger | Monthly job | **OPS** |
| `MTM_WEBAUTHN_MANDATORY_TENANT_COUNT` | int | **`100`** (`L1`, resolved by `CO-D1`) — store-only distribution means a platform credential manager is available on every supported device, so mandating WebAuthn cannot lock a tenant admin out | Monthly job | **SEC** |
| `MTM_WEBAUTHN_PLATFORM_OWNER_ENABLED` | bool | **`true` from Phase 1**, not Phase 7 (moved by `12` §12.3.4). One account, no adoption cost, highest-value target — and on the console it is the **only** accepted factor | Auth module | **SEC** |
| `MTM_WORKSHOP_CONFLICT_DETECTION` | bool | `false` in v1 (`L5`). Enabling requires adding a time range to workshop rows; the `EXCLUDE USING gist` constraint keys on `member_id` and starts working immediately once it exists | Shift validation | **OPS** |

---

## 9.12 Release gates enforced as assertions, not documentation

These are not tunable. Each is a build-or-boot failure.

| Assertion | Fails when |
|---|---|
| `app_role` holds no `BYPASSRLS` and owns no table in `app` | Either condition is violated |
| `FORCE ROW LEVEL SECURITY` is set on every table carrying `tenant_id` | Any table is missing it |
| `platform_writer` can write to exactly the five tables in §9.1 | It can write to a sixth |
| Cross-tenant isolation tests pass | Any test fails — this is the Phase 0 gate |
| `MTM_SYNC_CACHE_EXCLUDE` contains `patients.condition` | It does not |
| `MTM_PIN_SPKI_BACKUP` is non-empty and differs from primary | Empty or identical |
| `MTM_KMS_KEK_ID`, `MTM_DATA_REGION`, `MTM_KEK_SECONDARY_REGION` are all non-empty | Any is empty at boot |
| `MTM_ATTENDANCE_LOCK_EDIT_WINDOW_DAYS` ≤ `MTM_ATTENDANCE_LOCK_EDIT_WINDOW_MAX_DAYS` | A tenant configuration exceeds the ceiling |
| **`V-WEB-1`** — the served admin response headers contain **zero** `unsafe-` tokens | Any `unsafe-inline` or `unsafe-eval` appears in the header as actually served |
| **`V-WEB-2`** — a scripted pass over every console screen records **zero** CSP violation reports | Any violation is recorded |
| **`V-WEB-3`** — the mobile release bundle contains **no** `admin/` code | Any admin symbol or asset is found by bundle inspection |
| Admin routes are **not mounted** on the tenant API host | Any admin route answers on `api.<domain>` |
| No key carries the marker token `BLOCKED ON` + an item id | The token appears **(a)** anywhere in the resolved effective config, or **(b)** in this file on a line that starts a key row (`` | `MTM_ ``). Prose lines are out of scope by construction |

**On the last row, and on making it actually runnable.** The check has two greps, and both are stated above rather than implied: one over the resolved config dump, one over this file **restricted to key rows**. It is deliberately *not* a free grep over the whole file, because this file discusses the marker in three places while carrying none in a single key row — a check written loosely enough to trip over its own documentation is a check that gets disabled, and a disabled check is worse than none. **The scope restriction is the check, not an exemption from it.**

Its original purpose was to make it structurally impossible to ship a build in which one of the five pending decisions had been silently resolved by whoever filled in a blank. That purpose is served: **zero markers remain.** The assertion stays as a **ratchet** — it prevents the pattern from returning, rather than commemorating that it once existed.

**On `V-WEB-1`–`V-WEB-3`.** These three exist because the one genuinely unverifiable claim in the console design — what Flutter Web requires of a CSP — could not be checked against vendor documentation from this environment. Rather than write an unsourced assertion, the claim is converted into a **local machine-checkable gate**: whatever the framework turns out to need, the build fails if the served header carries an `unsafe-` token or if any screen reports a violation. An untestable citation is replaced by a testable build failure.

---

## 9.13 Web console (platform owner only)

Introduced 25 August 2026 by `12` §12.3. This section is the reason `R4` exists in `04`.

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_ADMIN_ORIGIN` | url | **`https://admin.<domain>`** — a separate origin from the tenant API host. The API is **reverse-proxied same-origin** under `/api/`, which removes CORS and preflight entirely and is what allows the `__Host-` cookie prefix to work | Boot; console build | **SEC** |
| `MTM_ADMIN_ROUTES_MOUNTED_ON_TENANT_HOST` | bool | **`false`** — asserted, not assumed. Admin routes are not registered on `api.<domain>` at all, so there is nothing there to authorise incorrectly | Router init + §9.12 assertion | **SEC** |
| `MTM_ADMIN_BUILD_TARGET` | string | `lib/main_admin.dart` — a separate Flutter Web entry point. Shared with mobile: a models package and the OpenAPI-generated client. Shared with mobile: **nothing else** | Build config | **SEC** |
| `MTM_ADMIN_AUTH_FACTOR` | enum | **`passkey_only`** — origin-bound WebAuthn. **TOTP is not accepted on the console.** The phishable factor is removed from the flow rather than layered on top of it | Console auth | **SEC** |
| `MTM_ADMIN_REQUIRE_UV` | bool | `true` — `userVerification: required` | Console auth | **SEC** |
| `MTM_ADMIN_MIN_REGISTERED_PASSKEYS` | int | **`2`**, on two separate devices, enforced from day one. This is the self-lockout mitigation, and it is the same failure mode as a wrong certificate pin: a control that locks out its only legitimate user is an outage, not a control | Console auth setup | **SEC** |
| `MTM_ADMIN_RECOVERY_CODES` | int | `10`, single-use, shown once | Console auth setup | **SEC** |
| `MTM_ADMIN_IP_ALLOWLIST_ENABLED` | bool | **`false` — explicitly cancelled, not merely unset.** It produces exactly the wrong-pin failure mode with no offsetting gain for a single operator who travels | Console auth | **SEC** |
| `MTM_ADMIN_ACCESS_TOKEN_MINUTES` | int | `10`. Held **in memory only** — never `localStorage`, never `sessionStorage`. **Honest caveat:** in-memory storage is not XSS immunity; it shortens the window and forbids silent persistence, which is a different and smaller claim | Console auth | **SEC** |
| `MTM_ADMIN_REFRESH_COOKIE_NAME` | string | **`__Host-mtm_admin_rt`** — the `__Host-` prefix forces `Secure` and `Path=/` and forbids a `Domain` attribute, so the cookie cannot be scoped to a parent domain shared with tenant hosts | Console auth | **SEC** |
| `MTM_ADMIN_REFRESH_COOKIE_FLAGS` | string | `HttpOnly; Secure; SameSite=Strict` | Console auth | **SEC** |
| `MTM_ADMIN_CSRF_MODE` | enum | `double_submit_plus_origin` — applied on the **single** cookie-authenticated endpoint (refresh). Everything else uses the in-memory bearer token and is not cookie-authenticated at all | Console auth | **SEC** |
| `MTM_ADMIN_SESSION_ABSOLUTE_HOURS` | int | `8` | Console auth | **SEC** |
| `MTM_ADMIN_SESSION_IDLE_MINUTES` | int | `15` | Console auth | **SEC** |
| `MTM_ADMIN_CSP` | string | `default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; connect-src 'self'; font-src 'self'; frame-ancestors 'none'; base-uri 'none'; object-src 'none'; form-action 'self'` — **zero `unsafe-` tokens**, enforced by `V-WEB-1` | Response headers | **SEC** |
| `MTM_ADMIN_SECURITY_HEADERS` | string | `Strict-Transport-Security: max-age=63072000; includeSubDomains` · `Cross-Origin-Opener-Policy: same-origin` · `Referrer-Policy: no-referrer` · `X-Content-Type-Options: nosniff` · a `Permissions-Policy` denying camera, microphone, geolocation and USB | Response headers | **SEC** |
| `MTM_ADMIN_FAILED_AUTH_ALERT_THRESHOLD` | int | **`1`** — the cheapest `R4` mitigation and the most interesting one. There is exactly **one** legitimate admin account, so there is no baseline noise: any failed admin authentication is signal. **The same property that produces `R2` (no separation of duties) produces detection precision that is unavailable to a multi-admin organisation** | Auth failure hook | **SEC** |
| `MTM_ADMIN_ENUMERATION_SAFE_RESPONSES` | bool | `true` — responses never reveal whether an admin account exists | Console auth | **SEC** |
| `MTM_ADMIN_SHOWS_TENANT_OPERATIONAL_DATA` | bool | **`false`** — ten platform-owner surfaces only (tenant list and health, subscriptions and manual activation, plan limits, trial, per-tenant feature flags, tenant create/suspend/delete, break-glass, audit search and Merkle verification status, platform metrics, report export). **Zero PHI reaches the browser structurally**, because the console's database role holds no tenant DEK — a cryptographic boundary, not a UI one | Console route guard | **SEC** |

> **What the console does not replace.** The server shell **remains** for incidents, migrations, and deep log analysis. The console replaces **daily administration**, not incident tooling. Any sentence claiming "no shell access" is false and is not written. This is item 10 of `04` §4.4.

---

## 9.14 App ↔ console coupling

The console writes; the app must render the consequence. Version skew between a browser that updates instantly and a mobile build that updates on a store's schedule is the most likely source of real bugs here, so every unknown value **fails closed**.

| Key | Type | Value | Read at | Boundary |
|---|---|---|---|---|
| `MTM_APP_UNKNOWN_TENANT_STATUS_BEHAVIOUR` | enum | **`treat_as_suspended`** — an app build that receives a `tenants.status` value it does not recognise must not guess "probably fine" | Client bootstrap | **SEC** |
| `MTM_APP_UNKNOWN_FEATURE_FLAG_BEHAVIOUR` | enum | **`off`** — an unrecognised flag is off, never on | Client feature gate | **SEC** |
| `MTM_APP_PLAN_LIMIT_RESPONSE_CODES` | list | **`402, 403`** — the API answers a plan-limit breach with these; the app renders a plan-limit screen rather than a generic error. RFC 9457 problem details carry the specific limit | Client error handler | **OPS** |
| `MTM_APP_GRANT_PULL_ON_OPEN` | bool | **`true`** — the app pulls active break-glass grants and tenant status on every open. **Push is a signal, pull is truth**: no consequence of a console write depends on a notification being delivered | Client bootstrap | **SEC** |
| `MTM_APP_FORCED_UPGRADE_ON_MIN_VERSION` | bool | **`true`** — see `MTM_MIN_SUPPORTED_APP_VERSION` in §9.6. Available only because distribution is store-only | Client bootstrap | **SEC** |

> The seven-row mapping from each console write, to the API response that carries it, to the **named app screen** that must render it, is in `12` §12.3.8. That table is the anti-bug artefact of this whole change: a console that can set a state the app has no screen for is a bug factory, and the mapping is what makes that condition checkable rather than hopeful.
