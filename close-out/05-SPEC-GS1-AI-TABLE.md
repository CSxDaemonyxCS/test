# 05 — GS1 Application Identifier Table (Implementation Reference)

**Date:** 24 August 2026 · **Status:** **PARTIAL — not verified against the primary specification**
**Related ledger item:** G2 · **Field-measurement protocol:** `06` · **Config keys:** `09`

---

## 0. Verification status — read this before writing any parser code

Every factual claim in this file about GS1 encoding comes from **model training knowledge**. I did **not** open `ref.gs1.org`, `www.gs1.org`, or any other GS1 page in this session — all outbound network access is blocked at the environment level (verbatim errors in `07`). Per the citation discipline in force:

- **No URL is cited**, because no page was fetched. A fabricated citation would be worse than this disclosure.
- Every row below carries `UNVERIFIED`.
- **No secondary source has been promoted to primary.** I did not substitute a blog post, vendor SDK doc, or Wikipedia article for the specification.
- **No local DataMatrix prevalence figure appears anywhere in this file.** That number comes from the field measurement in `06`, not from an estimate.

**What this means practically:** the parser is written so that a wrong assumption in this table produces a **rejection, never a silent misread**. Correctness of this table is a *convenience* property; safety comes from the fail-closed design in §3.

---

## 1. Application Identifiers in scope

| AI | Meaning | Data format (per training knowledge) | Length | Verification |
|---|---|---|---|---|
| `01` | GTIN — Global Trade Item Number | Numeric | Fixed, 14 digits | `UNVERIFIED` |
| `10` | Batch / Lot number | Alphanumeric | **Variable**, up to 20 characters | `UNVERIFIED` |
| `17` | Expiration date | Numeric `YYMMDD` | Fixed, 6 digits | `UNVERIFIED` |
| `21` | Serial number | Alphanumeric | **Variable**, up to 20 characters | `UNVERIFIED` |

**Nothing outside this set is parsed.** Any other AI encountered in a scanned symbol causes rejection (§3), not a best-effort attempt. This is deliberate: an AI I have not verified is an AI I must not interpret.

---

## 2. Structural rules the parser depends on

| # | Rule (per training knowledge) | Why the parser needs it | Verification |
|---|---|---|---|
| 2.1 | **FNC1 in the first position** marks the symbol as GS1-formatted | This is the single discriminator between "a GS1 DataMatrix" and "some other DataMatrix that happens to start with digits". Without it, a plain serial-number label starting `01…` decodes as a GTIN | `UNVERIFIED` |
| 2.2 | A **variable-length** field is terminated either by a **GS separator (ASCII 29)** or by end-of-data | Without this, `10` (lot) greedily swallows the following `17` (expiry) and both fields come out wrong — the exact failure that puts the wrong expiry date on a dispensed drug | `UNVERIFIED` |
| 2.3 | **Fixed-length** fields (`01`, `17`) are **not** followed by a separator | If the parser expects one, every fixed field misaligns by one byte | `UNVERIFIED` |
| 2.4 | Expiry `YYMMDD` with `DD = 00` means "end of that month" | Storing `00` as a day yields an invalid date; storing it as day 1 silently shortens shelf life by up to a month | `UNVERIFIED` |
| 2.5 | The two-digit year uses a sliding century window | A naive `20YY` breaks for long-dated stock and for expired stock across a century boundary | `UNVERIFIED` |

**Rules 2.1 and 2.2 are the two that actually matter for patient safety.** Both are enforced as hard rejections, so an error in my understanding of them yields a refused scan, not a wrong lot number in `stock_movements`.

---

## 3. Fail-closed parser contract (this part IS verified — it is my own design)

The parser is a total function returning either a fully-populated result or a typed rejection. There is no partial success and no "best effort".

```
parseGs1(raw: bytes) -> Result<Gs1Payload, Gs1Rejection>

REJECT if:
  REJ-1  FNC1 is absent from the first position          -> NOT_GS1_FORMAT
  REJ-2  any AI encountered is not in {01, 10, 17, 21}   -> UNKNOWN_AI
  REJ-3  a fixed-length field is short or non-numeric    -> MALFORMED_FIXED_FIELD
  REJ-4  a variable-length field exceeds 20 characters   -> FIELD_TOO_LONG
  REJ-5  a variable-length field is unterminated
         and is not the final field                      -> MISSING_SEPARATOR
  REJ-6  expiry does not resolve to a real calendar date -> INVALID_DATE
  REJ-7  AI 01 appears zero times or more than once      -> GTIN_CARDINALITY
  REJ-8  the same AI appears twice                       -> DUPLICATE_AI
  REJ-9  decoded GTIN is not present in this tenant's
         product catalogue                               -> UNKNOWN_PRODUCT

On ANY rejection:
  - do NOT write to stock_movements
  - do NOT pre-fill any form field with partial data
  - surface the rejection code to the user
  - fall through to MANUAL ENTRY
```

**Manual entry is a primary path, not a fallback** (Roadmap §10.7). The scanner is an accelerator layered on top of a workflow that is complete without it. This is what makes G2's unverified status survivable: the worst outcome of a wrong assumption in §1–§2 is that scanning stops working and staff type, which is exactly what they do today.

**REJ-9 deserves emphasis.** Validating the decoded GTIN against the tenant's own catalogue is a tenant-scoped check that runs under RLS. It converts "any barcode in the world" into "a barcode for a product this tenant actually stocks", which closes the injection surface far more effectively than any format check.

> **Naming note:** these rejection codes are prefixed `REJ-` deliberately. `R1`–`R3` are reserved throughout this close-out package for the three **residual risks** in `01` and `04`, and a parser rule sharing those labels would be a reference collision in the thesis.

---

## 4. Negative test suite — required before the scanner ships

These tests encode the failure modes, so they stay valid even if §1–§2 turn out to be wrong in detail.

| Test | Input shape | Expected |
|---|---|---|
| T1 | Valid DataMatrix **without** FNC1 in first position | `NOT_GS1_FORMAT`, manual entry offered |
| T2 | AI `30` (a quantity AI, out of scope) present | `UNKNOWN_AI` |
| T3 | AI `10` of 24 characters | `FIELD_TOO_LONG` |
| T4 | AI `10` immediately followed by `17` with **no separator** | `MISSING_SEPARATOR` — must **not** produce a lot value containing `17…` |
| T5 | `17` = `260231` (31 February) | `INVALID_DATE` |
| T6 | `17` = `2612 00` (day `00`) | Resolves to **31 December 2026**, not 1 December |
| T7 | Two `01` fields | `GTIN_CARDINALITY` |
| T8 | Valid GTIN not in the tenant catalogue | `UNKNOWN_PRODUCT` |
| T9 | Valid symbol belonging to **another tenant's** catalogue GTIN | `UNKNOWN_PRODUCT` — proves RLS scoping of REJ-9 |
| T10 | Truncated symbol: `01` present, `17` cut mid-field | `MALFORMED_FIXED_FIELD`, and **nothing written** |

**Cost:** 0.5 developer-days for the suite. It is the cheapest item in this entire close-out and it is what lets the thesis say "the unverified assumption is contained by design" rather than "the assumption is probably fine".

---

## 5. What goes in the thesis, verbatim

> The medication scanner reads GS1-structured DataMatrix symbols using application identifiers `01`, `10`, `17`, and `21`. These identifier numbers and their length rules **could not be verified against the primary GS1 specification in the research environment**; they are therefore marked as unverified in the implementation reference, and the parser is designed so that any deviation from the assumed structure produces an explicit rejection with a typed error code rather than a partial or best-effort interpretation. Manual entry is a primary input path, not a fallback, so scanner rejection degrades throughput and never data integrity. No claim of GS1 conformance certification is made.

**Never write** "GS1 compliant" or "GS1 certified" anywhere. Write "reads GS1-structured DataMatrix symbols". The first is a claim about a certification I do not hold; the second is a description of behaviour.
