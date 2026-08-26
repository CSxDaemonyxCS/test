# MTM — Final Close-Out & Decision Ledger (Cowork Work Order)

## 0. OPERATING MODE

Single autonomous pass. You are closing an audit, not opening one.

- **Do the work; don't return a plan for the work.** Every deliverable in §7 must arrive as finished, paste-ready text.
- **Two classes of item, and they are handled differently:**
  - **G, C, L, R items (14 total)** — you decide. Final, committed, defensible answers.
  - **D items (D1–D5, 5 total)** — you do **not** decide. These are mine to own. You lay out the real options, the deciding trade-off, and a clearly-labelled non-binding recommendation. Status stays `معلّق — بانتظار قراري`.
- **Do not ask clarifying questions mid-stream.** If a specific fact from me is genuinely required to close a **non-D** item, tag it `[FLAG-n]` inline and batch all flags in one block at the very end. Maximum 6 flags, each answerable in one line. The D-items are not flags — they are a standing decision brief, and they are not counted against the flag budget.
- **Do the research yourself with live search.** Never instruct me to go look something up.
- **Work order is fixed, and it is not the order of the output:**
  1. Read all attached files and resolve cross-references (§4).
  2. Run all external verification (G1, G2, G3) — these can run in parallel.
  3. *Then* close the C, L, and R items, and build the D-item option sets — in that order, because several C/L/R answers depend on what G3 and G4 turn up, and because the D-item option analysis needs the verified facts in hand.
  4. *Then* write the deliverables, expressing every D-dependent downstream item conditionally (rule 7).
- **Do not edit the attached files.** Produce new files only.
- Produce files, not one long chat message.

## 1. ROLE

Senior mobile-security architect and academic thesis advisor, acting as the accountable technical lead signing off before implementation continues and before the thesis defense — with the explicit exception of the five D-items, where you are the advisor preparing my decision, not the decider.

Your reader holds an **eMAPT** certification with hands-on penetration-testing experience, and is a solo developer with no deep backend specialization. Write the security reasoning at that level: name concrete attack techniques and concrete mitigations. No generic advice, no security-awareness filler.

## 2. PROJECT CONTEXT

**MTM (Medical Teams Management)** — a Flutter-based, multi-tenant SaaS platform coordinating **volunteer medical field teams (مفارز / Mafrazas)**. One tenant holds multiple مفارز, created self-service in-app; 5–8 members per مفرزة. The data is patient-adjacent.

Two coupled domains: field-team coverage/scheduling, and barcode-based (**GS1 DataMatrix**) medical-supply tracking.

**Scale trajectory:** 20 tenants at launch, scaling toward 1,000. Use these numbers to calibrate every quantitative trigger you set — a threshold that fires at launch scale, or never fires before 1,000, is a bad threshold.

**Architecture (locked unless the Blueprint says otherwise):**
- Client: Flutter + Riverpod + Drift + SQLCipher, offline-first via PowerSync
- Backend: NestJS modular monolith; no direct-to-DB client calls; no realtime layer
- Data: PostgreSQL with RLS; transaction-scoped `SET LOCAL` tenant context via `AsyncLocalStorage` / `nestjs-cls`; PgBouncer under threshold-based evaluation
- Crypto: envelope encryption, KEK → per-tenant DEK
- Auth: JWT + DB-backed rotating refresh tokens with device binding; TOTP MFA mandatory for Super Admin and Main Admin
- Ops: audit logging, remote wipe, force sync, break-glass
- **No web admin dashboard.** All admin functionality, including Super Admin, lives in the Flutter app. Any older document proposing a Next.js Super Admin dashboard is superseded.

**Constraints, absolute:**
- Solo developer, AI-assisted. Propose nothing that assumes a second person, an on-call rotation, or a review board.
- Every decision must survive being read aloud twice: once to an academic defense committee, once to a paying institutional customer's procurement team. For D-items, **every option you present** must survive that test, or be marked as failing it.

**Academic frame:** cybersecurity graduation project, faculty-supervised (Prof. Dalal), methodology **STRIDE + OWASP MASVS**, defended before a committee. Both frameworks matter — MASVS for the control claims, STRIDE for the threat reasoning. Every decision and every D-option must be mapped to both (§6, rule 11).

## 3. INPUTS AND AUTHORITY ORDER

All planning documents are attached to this session. Resolve conflicts in this order:

1. **`MTM-Master-Blueprint-Final.md`** — authoritative for architecture and locked decisions.
2. **`MAP_WORK.md`** — authoritative for execution sequencing, the dependency graph, and the open architectural decisions it lists.
3. **Module specs** — `MTM-Detachments-Module-Foundation.md` (مفرزة/detachments, D‑16 lock, access-control taxonomy) and `MTM-Workshops-Module-Spec.md` — authoritative within their own modules.
4. **The MTM close-out document** (pasted in §11) — the authoritative work list of what remains open, and the most recent statement of intent. Where it conflicts with an older document, **it wins, and you name the conflict explicitly** in the cross-reference table.

If a document is missing or a named section cannot be located, say so plainly in the cross-reference table. Never reconstruct the contents of a section you could not read.

## 4. MANDATORY PRE-WORK — CROSS-REFERENCE RESOLUTION

The close-out document is written in shorthand that points into the Blueprint and MAP_WORK. **You may not answer a cross-referenced item — or build a D-item option set — without first reading the section it points to.**

Resolve, at minimum:

| Referenced as | Needed for |
|---|---|
| §0.7 assumptions 1, 4, 6 | D5, D2, D4 |
| §6.7 (attestation design) | D1 |
| §7.1, §7.2, §7.3, §7.4 | L3, R2, L4 |
| §9, §9.5 | G1, C2 |
| §10.7 | C1 |
| §12.4, §12.5 | L4 |
| §13, §13.7 | D2, D3 |
| §15 | L2 |
| §21 | G4 — **confirm 21 is actually an unused section number; if it is taken, renumber and say so** |
| Risks 3, 4, 7, 8, 10 | D3, D1, R1, R2, R3 |
| Phases 0–10 | the phase checklist — use the Blueprint/MAP_WORK phase numbering and names, do not invent a parallel scheme |
| The MASVS-STORAGE / MASVS-PRIVACY / MASVS-AUTH tables | G3, G4, L1, R1 |
| Decision 6 (backup/DR) in MAP_WORK | D2, D3 |

**Deliverable:** `00-CROSSREF-RESOLUTION.md` — a table of every reference above with: where you found it, a one-line summary of what it actually says, and status `RESOLVED` / `NOT FOUND`. Anything `NOT FOUND` that you nonetheless had to reason about must be tagged `[افتراض غير متحقّق]` at every point of use in the ledger.

## 5. THE JOB

The close-out document lists **19 open items** in five categories, each already carrying a draft recommendation from a prior review:

- **G1–G4** — verification gaps → **you close these**
- **D1–D5** — undecided decisions → **you brief these, I close them**
- **C1–C2** — items conditioned on a future measurement → **you close these**
- **L1–L5** — deliberate deferrals needing a long-term stance → **you close these**
- **R1–R3** — accepted residual risks → **you close these**

So: **14 items get one final, defensible answer each.** This is a sign-off, not a second opinion.

**5 items (D1–D5) get a decision brief instead** — real options, the deciding trade-off, a non-binding recommendation, and the conditional consequences of each branch. Never a commitment. These are trade-offs that turn on business facts, market access, and money I own; a decision you make for me here is a decision I cannot audit.

## 6. RULES

**1 — Decide, don't enumerate. (G, C, L, R only.)** One answer per item. If the draft recommendation holds, confirm it in a single sentence and move on. If it is weak, incomplete, or wrong, override it and say why in 2–3 sentences. Never restate the draft recommendation back to me. **This rule does not apply to D1–D5** — those are governed by rule 3, where enumeration is the deliverable.

**2 — Verify externally, now, with citations.**

- **G1 — Related Work.** Research **Aladtec, Vector Solutions (Vector Scheduling), Kronos/UKG TeleStaff, Rosterfy, Resgrid, and D4H**. These are close comparators — public-safety/EMS scheduling, volunteer management, and emergency-response readiness — so the comparison must be specific, not generic. For each product, establish: deployment model (SaaS / on-prem / mobile-first); tenancy model; the core scheduling primitive (shift template vs. self-scheduling vs. availability bidding vs. callout/paging); whether coverage is **credential/qualification-aware**; offline capability on mobile; whether consumables/inventory/barcode tracking exists at all; published security posture and certifications (SOC 2, HIPAA, ISO 27001); target sector; and pricing/licensing visibility. Then produce the **actual Related Work section text**: a comparison table, 500–800 words of Arabic narrative, and a closing gap statement naming precisely what none of them combine. Do not overclaim novelty — a committee punishes that harder than a modest claim. Where a vendor's public documentation does not state something, write "غير معلن" rather than inferring it.
- **G2 — GS1 Application Identifiers (numbers only).** Pull the exact AIs relevant to this project from the **GS1 General Specifications** — the primary source, cited by edition/version and section number, not a secondary summary or blog. For each AI, give: the AI code, the official data title, format (numeric/alphanumeric, fixed vs. variable length), whether it is a predefined-length AI, and whether an FNC1/`GS` separator is required after it. Cover at minimum the healthcare-relevant set the project needs, and state which are variable-length — that is a direct parsing requirement in the scanner code, not trivia. Also state the FNC1-in-first-position rule for GS1 DataMatrix. If the specification PDF is not retrievable, mark the item `PARTIAL` and say so; do **not** promote a secondary source to primary. **Do not estimate local DataMatrix prevalence** — that is C1's field measurement, not a research task.
- **G3 — iOS platform claims.** Verify directly against `developer.apple.com`: (a) whether any API allows a **non-MDM third-party app to remotely wipe** its own data or the device; (b) the actual documented behavior of **`kSecAttrAccessibleWhenUnlockedThisDeviceOnly`** — accessibility conditions, backup/restore behavior, and device-migration behavior. Then add (c), which the draft recommendation misses: **what Apple does positively offer instead**, so the MASVS-STORAGE row carries a real control rather than only an absence — e.g. server-signalled key destruction on next launch/foreground, keychain item deletion, file Data Protection classes. Cite each finding with a URL and access date.
- **Citation discipline, absolute.** If you did not actually fetch a page, do not cite it. Every external claim carries a URL and an access date. If something cannot be verified, write "غير قابل للتحقّق" and state what you tried. A fabricated citation destroys the thesis; an honest gap does not.

**3 — D-items: brief me, do not decide for me.** For each of D1–D5:

- Status is `معلّق — بانتظار قراري`. Never `مؤكَّدة`, `مُعدَّلة`, or `مُلغاة` — those verdicts belong to items you close.
- State **the deciding trade-off in ≤12 words**. This is the single sentence I should be able to read and know what I'm actually choosing between.
- Present **the real options** — 2 to 4, whatever genuinely exists. Not strawmen: if the draft recommendation is one of the live options, include it as one and evaluate it on the same axes as the rest. If an option looks obvious to you, it still gets presented with its costs, because "obvious" is doing work I need to see.
- Give a **non-binding recommendation**, explicitly labelled as such, with the reason in 1–2 sentences. Recommending is expected; committing is not.
- State the **conditional downstream consequences** of each branch (rule 7).
- State the **decision deadline** — the phase at which this becomes blocking, so I know which of the five I have to answer first.
- Do not resolve a D-item by declaring one option "the default." The only permitted use of a default is narrow: where a downstream deliverable physically cannot be written without assuming a branch, name that branch as the working assumption **for that deliverable only**, and state exactly what gets rewritten if I choose otherwise.

**Verified facts are still yours to establish.** Where a D-item has a factual component — what the Blueprint actually says, what Play Integrity actually returns outside the Play Store, whether a given provider actually offers managed KMS — verify and state it as fact. The open part is the choice, not the evidence. Do not hide behind "pending" to avoid research.

**3b — The FLAG mechanism, unchanged and separate.** `[FLAG-n]` is for a single missing fact that blocks a **non-D** item — a version number, a provider name, a date. Maximum 6, batched at the end, one line each. D-items are not flags: they are open decisions by design, they belong in the ledger with full option analysis, and they do not consume the flag budget. Do not collapse a D-item into a flag, and do not turn a flag into a D-item.

**4 — C-items: no invented numbers.** Finalize the measurement protocol instead: sample size and sampling method, pass/fail threshold, what happens on fail, what happens on pass, and exactly what gets recorded in both cases (including the fields of the data sheet). The protocol must be executable by one person in a day.

**5 — L-items: a quantitative trigger, or it isn't a decision.** Every deferral ends on a number — tenant count, paying-tenant count, screen count, support-hours-per-week, monthly break-glass events. "Later," "when it makes sense," or "in a future phase" is a failed answer. Calibrate against the 20 → 1,000 tenant trajectory and state where the trigger is checked (a dashboard query, a monthly review line in the runbook). Where a trigger's value depends on a D-item outcome, express the trigger conditionally rather than picking one.

**6 — R-items: disclose, don't dissolve.** For each, write the final honest disclosure paragraph in the register format of §8.4 — suitable for direct paste into the thesis risk register — plus the single cheapest mitigation that measurably narrows the risk. Do not quietly convert a residual risk into a solved control.

**7 — Surface second-order conflicts yourself, and keep D-dependencies open.** Do not wait for me to spot them.

**The conditional rule, which overrides convenience everywhere:** any G, C, L, or R answer, any conflict-matrix row, any phase-checklist entry, and any config key whose correct value depends on how a D-item lands must be written as a branch — `إذا [خيار أ] → X، إذا [خيار ب] → Y` — not silently resolved to whichever branch you'd have picked. Silently resolving a D-dependency is the single worst failure mode in this document, because it produces a deliverable that looks finished and is quietly built on a decision I never made. Where a downstream deliverable genuinely cannot be written without assuming a branch, label the assumed branch, and state what has to be rewritten under the alternative.

Check and report on these pairs at minimum, then add any others you find:

- **D1 (distribution) →** §6.7 attestation design; which Play Integrity verdict fields are even evaluable outside the Play Store; minimum-version enforcement → risk 4 (certificate pin rotation); L1
- **D2 (hosting/KMS) →** G4 data residency clause; D3 KEK custody; MAP_WORK Decision 6 (backup/DR); PowerSync service region
- **D3 (KEK custody/succession) →** risk 3; G4 processor obligations; key-recovery vs. backup story; R3
- **D4 (D‑16 semantics) →** the server-side D‑16 mirror in the Detachments module; per-tenant config with a server-enforced maximum
- **D5 (Super Admin scope) →** R2 (who oversees Super Admin); L4 (which identity reaches the analytics views); the RLS bypass path; MFA scope — see rule 12
- C1 (barcode measurement) → G2; the feature flag; manual entry remaining the fully-functional primary path
- C2 (coverage minimum) → the qualifications model; the risk of enforcing a block on a criterion that does not yet exist
- L1 (WebAuthn trigger) → D1; TOTP phishing surface growth vs. tenant count
- L2 (no payment gateway) → §15; keeping PCI scope at SAQ-A
- L3 (no web panel) → the earlier review's request for a support-load threshold; Flutter Desktop as the escape hatch; CLI in phase 9; R3
- R1 (self-shred TTL) → PowerSync local cache scope; the remote-wipe design; MASVS-STORAGE
- R2 (break-glass oversight) → summary volume at 1,000 tenants; append-only log integrity; **D5**
- R3 (solo developer) → D3 bus factor; the phase-8 runbook

**8 — End-state text only.** Every deliverable is text I paste directly into the thesis or the codebase: the actual Related Work section, the actual legal-basis section, the actual risk-register entries, the actual AI parsing table. A sentence that tells me to write something later is a failed deliverable. **The one permitted exception:** text whose wording genuinely depends on an open D-item. There, either give the short version of both branches, or state plainly which single sentence gets written once the D-item lands — never a vague placeholder.

**9 — Cost, on every decision.** Each entry states implementation cost in **developer-days** (solo, AI-assisted) and **reversal cost** as `رخيص` / `متوسّط` / `مكلف` — how expensive it is to change this decision after it ships. A sign-off without cost is an opinion. **For D-items, cost and reversal cost are stated per option** — that asymmetry is usually the thing that actually decides the call.

**10 — Defensibility test.** Each entry ends with two one-line rows: the hardest question a **committee** would ask and its answer; the hardest question a **procurement team** would ask and its answer. One line each — a stress test, not an essay. **For D-items, run the test on every option**, one line per question per option. If an option fails either test, say so — that is the most useful thing you can tell me.

**11 — Map every decision to STRIDE and MASVS.** Name the STRIDE category or categories addressed, and the specific MASVS category (MASVS-STORAGE, -CRYPTO, -AUTH, -NETWORK, -PLATFORM, -CODE, -RESILIENCE, -PRIVACY). For D-items, map **each option**, since options often differ in which control they even claim. If something maps to neither, say so — that's a signal it belongs in the product backlog, not the security chapter.

**12 — D5: verify the contradiction, present the resolutions, don't pick one.** The close-out document marks D5 closed, and I do not want closed items reopened on preference alone. But there is an actual contradiction: `MTM-Detachments-Module-Foundation.md` treats **tenant-level** Super Admin as an interpretation and notes that platform-ops-only Super Admin would mean the signer-upper lands as Main Admin instead — while the earlier architecture review describes Super Admin as the **platform-level cross-tenant query path** for platform administration and AI-training aggregation. Both readings cannot be true.

Your job on D5 is split:
- **Factual, and you must settle it:** state which reading the Blueprint actually supports, quoting it. This is evidence, not preference.
- **Open, and you must not settle it:** which role model MTM adopts — tenant-level Super Admin, platform-only Super Admin, or two distinct roles with a named privilege boundary. Present these as options with the privilege boundary spelled out for each.
- **Propagate conditionally:** R2 and L4 both change shape depending on the answer. Write both under the conditional rule of rule 7, and do not let either quietly assume a reading.

**13 — No new scope.** Do not introduce a new subsystem, service, or vendor dependency unless you state its dev-day cost and what it replaces. A solo developer's calendar is the binding constraint, not the threat model.

**14 — Absolute dates only.** No "in six months," no "next semester." Convert every date to an absolute one. Cite access dates as actual retrieval dates.

**15 — Length discipline.** Decision prose ≤180 words per closed item; D-item briefs ≤320 words each including the options table; paste-ready thesis/spec text excluded from both caps. Total ledger ≤7,000 words. Density over volume — an unusable 20,000-word document is a failed deliverable.

## 7. DELIVERABLES

Produce these as separate files:

| File | Contents | Language |
|---|---|---|
| `00-CROSSREF-RESOLUTION.md` | Cross-reference table from §4 | Arabic |
| `01-FINAL-DECISION-LEDGER.md` | All 19 entries, G1→R3 — 14 closed (schema §8.1a), 5 pending decision briefs (schema §8.1b). The primary deliverable. | Arabic |
| `02-THESIS-RELATED-WORK.md` | G1: comparison table + 500–800-word narrative + gap statement + citation list | Arabic (citations in Latin script) |
| `03-THESIS-SECTION-21-LEGAL-BASIS.md` | G4: the complete section, contents per §8.3. Flag any clause whose wording depends on D2 (hosting region) and give both branches. | Arabic |
| `04-THESIS-RISK-REGISTER.md` | R1–R3 plus any residual risk newly created by a G/C/L decision, schema §8.4. R2 written conditionally on D5. | Arabic |
| `05-SPEC-GS1-AI-TABLE.md` | G2: AI table + parsing/separator implications + spec citation | **English** |
| `06-MEASUREMENT-PROTOCOL-C1.md` | C1: executable field protocol + blank data sheet + recording rules | **English** |
| `07-VERIFICATION-LEDGER.md` | Every external claim: claim, URL, access date, status `VERIFIED` / `PARTIAL` / `UNVERIFIABLE` | Arabic, URLs verbatim |
| `08-CONFLICT-MATRIX.md` | Second-order conflicts, schema §8.5. D-dependent rows written as branches. | Arabic |
| `09-CONFIG-KEYS.md` | Every concrete parameter as one implementable table, schema §8.6. D-dependent keys carry both branch values. | **English** |
| `10-PHASE-CHECKLIST.md` | Phase 0 → Phase 10, using the Blueprint's own phase names, with the sequencing dependencies you found — plus a **decision-deadline row per D-item** marking the phase at which my answer becomes blocking, and which tasks are gated behind it | Arabic |
| `11-FLAGS-AND-SELFCHECK.md` | Batched non-D flags (max 6) + the pending-D summary + the coverage self-check table of §8.7 | Arabic |

## 8. SCHEMAS

### 8.1a Ledger entry — closed items (G1–G4, C1–C2, L1–L5, R1–R3)

```
### [CODE] <short title>
**القرار النهائي:** <1–3 sentences, decisive>
**الحكم على التوصية المسبقة:** مؤكَّدة | مُعدَّلة | مُلغاة — <why, only if not confirmed>
**القيمة/المعامل:** <TTL in days, threshold %, config key name, role name — or "لا ينطبق">
**STRIDE:** <categories>   **MASVS:** <categories>
**الكلفة:** <n dev-days>   **كلفة التراجع:** رخيص | متوسّط | مكلف
**التبعية:** <items this constrains or is constrained by; if a D-item, state the branch condition>
**سؤال اللجنة:** <one line> → <answer, one line>
**سؤال المشتريات:** <one line> → <answer, one line>
**نص جاهز للّصق:** <the actual text, or a pointer to the file that carries it>
```

### 8.1b Ledger entry — pending decisions (D1–D5)

```
### [D#] <short title>
**الحالة:** معلّق — بانتظار قراري
**المفاضلة الحاسمة:** <the deciding trade-off, ≤12 words>
**الوقائع المتحقّقة:** <what you established as fact — Blueprint wording, platform behavior, provider capability — with citations>

**الخيارات والمفاضلة:**
| الخيار | ما يمنحه | ما يفوّته أو يكلّفه | STRIDE / MASVS | الكلفة (أيام) | كلفة التراجع | سؤال اللجنة → الجواب | سؤال المشتريات → الجواب |
|---|---|---|---|---|---|---|---|

**التبعيات المشروطة:** إذا [خيار أ] → <what changes>، إذا [خيار ب] → <what changes>
**توصيتي (غير ملزمة):** <1–2 sentences + the reason. Labelled as advice, never as the decision.>
**الفرع الافتراضي للعمل المتقدّم:** <only where a downstream deliverable cannot be written without assuming a branch — name the branch, the deliverables riding on it, and exactly what gets rewritten if I choose otherwise. Otherwise write "لا حاجة — لا شيء متقدّم يعتمد عليه".>
**موعد الحسم:** <the phase at which this becomes blocking, and what is gated behind it>
```

### 8.2 Related Work comparison table columns

Product | Sector | Deployment | Tenancy | Scheduling primitive | Qualification-aware coverage | Mobile offline | Consumables/barcode | Published certifications | MTM's difference

### 8.3 Section 21 required contents

Controller/Processor allocation (tenant = **الجهة المسيطرة (Controller)**, you = **الجهة المعالِجة (Processor)**) and why that allocation is both operationally and legally correct; legal basis for processing; handling of health-adjacent data as **بيانات فئة خاصة (special-category data)**; data residency with `[HOSTING_REGION]` as an explicit named placeholder — and marked as conditional on **D2**; retention and deletion; a stated **breach-notification window in hours**; sub-processor register; data-subject requests routed through the tenant; cross-border transfer stance; a one-page **اتفاقية معالجة البيانات (DPA)** outline; and the mapping to **MASVS-PRIVACY**.

**Regime:** write it GDPR-shaped and regime-neutral. Use GDPR's vocabulary and its strictest defaults, and state explicitly that these are adopted as international best practice where no binding local statute applies — with `[JURISDICTION]` as a named placeholder. Do not assert that any specific law binds this system.

### 8.4 Risk register entry

```
| المعرّف | الوصف | STRIDE | MASVS | الاحتمال | الأثر | لماذا لا تُحلّ بنيويا | التخفيف الأرخص المعتمد | الخطر المتبقي بعد التخفيف | مُشغِّل إعادة النظر |
```
Followed by the disclosure paragraph — written to be read aloud in a defense without flinching. Where a row's content depends on an open D-item, write both branches inline rather than choosing.

### 8.5 Conflict matrix

```
| البند أ | البند ب | اتجاه القيد | ما ينكسر إن تُجاهل | الحسم أو الفرع الشرطي |
```
The last column carries a decision for closed-item pairs, and an `إذا … → …` branch for any pair touching D1–D5.

### 8.6 Config keys table

```
| Key | Value | Conditional on | Where enforced (server / tenant row / client flag) | Server-enforced max? | Item | Phase |
```
Name keys consistent with the existing codebase conventions. Anything security-relevant must be server-enforced; say so explicitly where a client-side value is only a hint. If a key's value depends on a D-item, put the D-code in `Conditional on` and give one row per branch — never a single averaged value.

### 8.7 Coverage self-check

A 19-row table. Column sets differ by item class:

- **Closed rows (G, C, L, R — 14 rows):** `قرار نهائي؟` `معامل محدّد؟` `نص جاهز؟` `STRIDE+MASVS؟` `كلفة؟` `تعارضات فُحصت؟`
- **Pending rows (D1–D5 — 5 rows):** `قرار نهائي؟` must read **`لا — معلّق`**, and that is a **pass**, not a failure. Then: `خيارات مطروحة؟` `المفاضلة معلنة؟` `توصية غير ملزمة؟` `تبعيات مشروطة؟` `موعد حسم؟` A D-row showing a committed decision is the failure condition here.

Then three closing lines: any item that ended `UNVERIFIABLE` or `NOT FOUND`; a list of every downstream item written on an assumed D-branch, with the branch named; and one line confirming all 19 codes are present.

## 9. LANGUAGE

The ledger and all thesis-facing text in **Arabic**, matching the technical register of the source document — English technical terms in parentheses exactly as the source does, e.g. "الجهة المسيطرة (Controller)". Code-facing files (`05`, `06`, `09`) in **English**, since they paste into source files and comments. Reasoning and searching in English. URLs, product names, and citations always verbatim in Latin script.

## 10. FAILURE MODES

The deliverable fails if it contains any of the following:

- **A D-item answered with a committed decision**, or a D-status written as `مؤكَّدة` / `مُعدَّلة` / `مُلغاة`
- **A G, C, L, or R answer, conflict row, phase entry, or config value that silently assumes a D-outcome** instead of branching — the worst failure in this document, because it looks finished
- A D-item presented with strawman options, or with the recommendation dressed up as the decision
- The phrases "consider", "you may want to", "it depends", "should be investigated", "best practice suggests" without a citation, "in a future phase" without a number, or "TODO"
- A citation for a page that was not actually fetched
- A reconstructed summary of a Blueprint section that could not be found
- **A closed item (G, C, L, R) answered with two options instead of one decision** — this applies to the 14 closed items only; D1–D5 must present options, and doing so is correct there
- A deferral without a numeric trigger
- A residual risk quietly upgraded into a solved control
- A recommendation that requires a second person
- A D-item collapsed into a `[FLAG-n]`, or a flag inflated into a D-item
- Any of the 19 codes missing from the ledger

## 11. SOURCE DOCUMENT

MTM — open / pending / deferred items, with draft recommendations:

[PASTE THE FULL MTM CLOSE-OUT DOCUMENT HERE]
