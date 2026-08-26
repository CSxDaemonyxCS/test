# 06 — Field Measurement Protocol: C1 (DataMatrix Viability)

**Date:** 24 August 2026 · **Ledger item:** C1 · **Executable by:** one person, one day, no code
**Pre-registration:** the sample size, sampling method, and pass threshold below were fixed on **24 August 2026, before any packaging was observed.** They must not be changed after data collection begins. If they are changed, the run is void and restarts.

---

## 1. The question this measures

> On the medication packaging this team actually receives and dispenses, how often can a phone camera extract **both a lot number and an expiry date** from a GS1-structured DataMatrix symbol?

**Not** "does the box have a 2D barcode on it". A symbol that decodes but carries only a GTIN is a **failure** for this protocol, because GTIN alone drives no feature: FEFO needs expiry, and recall handling needs lot. The whole business case for the scanner is those two fields.

**No prevalence figure is asserted anywhere in this close-out.** This protocol produces the number. Estimating it would be inventing the very fact the protocol exists to establish.

---

## 2. Sample: n = 50 distinct SKUs, stratified

| Stratum | n | Selection rule |
|---|---|---|
| **A — most dispensed** | **20** | The 20 SKUs with the highest dispensing count over the last 90 days. If no dispensing history exists yet, substitute the 20 the pharmacy lead names as most-used, and record that substitution on the data sheet |
| **B — random from current stock** | **15** | Number every remaining distinct SKU on the shelves, then draw 15 using a random number generator. **Do not** pick by convenience, position, or by which boxes look like they have barcodes |
| **C — newest arrivals** | **15** | The 15 most recently received distinct SKUs, newest first |

**Distinct SKU, not distinct box.** Ten boxes of the same product count once. If a SKU appears in more than one stratum, keep it in the highest stratum (A > C > B) and draw a replacement for the lower one.

**Why stratified rather than a flat sample:** a flat unweighted sample can pass on a long tail of rarely-touched items while failing on everything staff handle daily. That is precisely the outcome where the feature ships, disappoints on first contact, and gets abandoned in a week. Stratum A carries the pass/fail decision for that reason. Stratum C exists because packaging changes over time — if C outperforms A markedly, the trend is favourable and that belongs in the report even though it does not change the verdict.

---

## 3. Per-SKU procedure

For each of the 50 SKUs, take one representative box and:

1. **Look for a 2D symbol.** Record `symbol_present` = yes/no. If no, the SKU is a failure; skip to the next.
2. **Scan it** with the phone that will actually be used in the field (not a lab device), under **normal ward lighting**, held naturally. Allow up to **3 attempts** and record how many were needed.
3. **Record what decoded**, field by field: GTIN, lot, expiry, serial — each present or absent.
4. **Verify the expiry against the human-readable print** on the box. A decoded expiry that disagrees with the printed one is a **failure**, and it is the single most important observation in the whole run — flag it in the notes.
5. **Record label condition:** intact / creased / partially covered by a pharmacy sticker. Pharmacy stickers covering the symbol are a real-world failure mode and count as a failure, not as an excluded case.

**Total time:** roughly 3–5 minutes per SKU including data entry ⇒ **3–4 hours for all 50**, plus setup. One person, one day, comfortably.

---

## 4. Data sheet

One row per SKU. Paper or a spreadsheet; either is fine.

```
sku_id, product_name, stratum(A|B|C), dispense_rank_90d,
symbol_present(Y|N), attempts(1|2|3|failed),
gtin_decoded(Y|N), lot_decoded(Y|N), expiry_decoded(Y|N), serial_decoded(Y|N),
expiry_matches_printed(Y|N|NA),
label_condition(intact|creased|covered),
both_fields(Y|N),          <-- lot_decoded == Y AND expiry_decoded == Y
notes
```

`both_fields` is the only derived column and the only one the decision rule reads.

**Also record once, at the top of the sheet:** date of the run, phone model and OS version, scanning app used, lighting conditions, name of the person running it, and whether stratum A used dispensing history or the pharmacy lead's naming.

---

## 5. Decision rule — fixed in advance

> **PASS** if **≥ 70% of stratum A** (that is, **≥ 14 of the 20 most-dispensed SKUs**) have `both_fields = Y`.

Strata B and C are **recorded and reported but do not affect the verdict.** They characterise the tail and the trend; they do not get to rescue a failing result on daily-use items.

**Secondary metrics — reported, never part of pass/fail:** overall `both_fields` rate across all 50; mean attempts among successes; count of `label_condition = covered`; count of any `expiry_matches_printed = N`.

**One override, and only one:** **a single instance of `expiry_matches_printed = N` fails the entire run regardless of the 70% figure.** A scanner that can print the wrong expiry date onto a medication record is worse than no scanner. If this occurs, the cause must be found and eliminated before any re-run.

---

## 6. On PASS

1. Build the scanner path: **3 developer-days** (parser from `05`, camera integration, the ten negative tests, reconciliation against the tenant catalogue).
2. Ship it as an **accelerator over manual entry**, never as a replacement. Manual entry stays the primary path (Roadmap §10.7).
3. Put the measured percentage, the date, and `n = 50` in the thesis. Report the observed number, whatever it is — not a rounded or flattering version.
4. Re-run this protocol **once per year** using the same strata definitions, because packaging changes and a stale measurement is a stale claim.

## 7. On FAIL

1. **Do not build the scanner path.** Zero developer-days spent. This is the point of measuring first.
2. Manual entry remains the sole input path and needs no change — it was never designed as a fallback.
3. Record the failure and the measured percentage in the thesis. **A negative result reported honestly is a finding**; it is direct evidence for the design principle that the accelerator was correctly gated behind a threshold declared in advance.
4. Re-run only when a concrete trigger occurs: **a supplier change**, or **≥ 20 new SKUs entering stock** since the last run. Not on a hunch.

## 8. Ambiguous outcome — decided in advance so it cannot be argued later

If stratum A lands at **exactly 13 of 20 (65%)**, the run is a **FAIL**. There is no rounding up, no "close enough", and no second look at strata B and C to change the verdict. Fixing this in advance is what makes the threshold meaningful rather than decorative.

---

## 9. Why this protocol is defensible to a committee

Three properties, and each of them answers a specific hostile question:

- **The threshold was declared before observation** (§0), which forecloses "did you pick the number after seeing the data?"
- **The decision metric is weighted toward operational reality** (stratum A), which answers "why does this measurement predict adoption rather than just describe packaging?"
- **The failure branch costs zero developer-days and changes no workflow**, which answers "what did you risk by not knowing?" — the honest answer being: nothing, because the feature was gated, not assumed.
