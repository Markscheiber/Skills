# Known Issues and Landmines

These are upload failures that have actually happened in the field. Address every one before delivering an XML.

## 0. Encoding declaration quirk in Claros exports

**Symptom**: `xml.etree.ElementTree.ParseError: encoding specified in XML declaration is incorrect`

**Cause**: Files exported from Claros include an XML declaration like `<?xml version="1.0" encoding="utf-16"?>` but the actual bytes are UTF-8 (or ASCII). Python's parser refuses because the declared encoding doesn't match the bytes.

**Fix**: Strip the declaration before parsing. The bundled `load_reference()` in `build_claros_xml.py` does this automatically:
```python
import re
content = re.sub(r'<\?xml[^?]*\?>\s*', '', content, count=1)
tree = ET.ElementTree(ET.fromstring(content))
```

When writing the OUTPUT XML, don't add an XML declaration at all. Claros accepts files without one (the Ada County reference and our Futura output both lack one and load fine).

## 1. Missing XML namespaces

**Symptom**: Claros rejects the file with "Data could not be loaded" or similar.

**Cause**: Python's `xml.etree.ElementTree` strips `xmlns:xsi="..."` and `xmlns:xsd="..."` from the root element during parse/write because they aren't actually used in element or attribute names. Claros validates against a schema and requires these declarations.

**Fix**: Two-part solution:
1. Register the namespaces BEFORE parsing:
   ```python
   ET.register_namespace("xsi", "http://www.w3.org/2001/XMLSchema-instance")
   ET.register_namespace("xsd", "http://www.w3.org/2001/XMLSchema")
   ```
2. Inject them back into the root tag during string serialization (registration alone is not enough):
   ```python
   xml_str = ET.tostring(root, encoding="unicode")
   xml_str = xml_str.replace(
       '<Scenarios ',
       '<Scenarios xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" ',
       1
   )
   ```

**How to verify**: First 200 characters of the output file should contain both `xmlns:xsi=` and `xmlns:xsd=`.

## 2. Wrong line endings

**Symptom**: File looks correct but Claros rejects it. May be silent or generic error.

**Cause**: Reference XMLs are written with CRLF (Windows) line terminators. Python writes LF (Unix) by default. Claros expects CRLF.

**Fix**: Convert before writing:
```python
xml_str = xml_str.replace("\r\n", "\n").replace("\n", "\r\n")
with open(out_path, "w", encoding="utf-8", newline="") as f:
    f.write(xml_str)
```

The `newline=""` is critical — without it, Python re-translates `\n` back to the platform default and you get LF on Linux/Mac.

**How to verify**: Run `file <output>.xml` — should say "with CRLF line terminators".

## 3. Zero/empty values where Claros does math

**Symptom**: "Data could not be loaded" upload error even when XML is structurally perfect.

**Cause**: Claros performs actuarial calculations that divide by census, enrollment, and underwriting fields. When these are zero or empty, the math fails silently and the upload is rejected.

The fields that break:
- `CensusType1.TotalEE` and `EEOnlyPct`
- `Enrollment.AdultEquivalent` and `DependentPerEmployee`
- `StopLoss` PEPM fees (any of the `sf_*PEPM` or `SF_*` fields)
- `StopLoss` percentages (`fi_Administration`, `sl_Profit`, etc.)
- `ReimbursementAccount.EmployeeContribution`
- `Normalization.TotalEE` (if 0 with NormalizationFactor=1, ratio breaks)

**Fix**: Use minimal placeholders, not zeros:
- `1` for counts
- `1` (= 100%) for `EEOnlyPct` if not known
- `1` for dollar fees ($1 placeholder PEPM)
- `0.01` for percentages (1% placeholder)
- `0` only for clearly additive fields like `sf_PharmacyRebatePEPM` (a $0 rebate is fine)

The rule of thumb: if a field can plausibly appear in a denominator, give it at least `1`.

**How to verify**: After building, print every field touched and confirm none is `"0"` or `""` unless you've explicitly verified it's safe.

## 4. Deleted elements vs zeroed values

**Symptom**: Subtle data validation error.

**Cause**: Removing optional elements (like `<Zip>` rows in `EmployeesInZip`) can cause Claros's import to fail if it's expecting a certain element structure.

**Fix**: Don't delete elements. Zero out their values instead.
```python
# WRONG
for z in zips.findall("Zip"):
    zips.remove(z)

# RIGHT
for z in zips.findall("Zip"):
    z.set("EnrolledEmp", "0")
```

**How to verify**: Element count by tag should match the reference exactly. The `signature()` walker below catches this.

## 5. UI accordion out of sync with PlanElement

**Symptom**: File loads, but Claros displays wrong copays/deductibles even though calculations may be correct.

**Cause**: The XML has both a calculation layer (`Plan/SubPlan/PlanElement`) and a display layer (`Scenario/MedCategoryParentNode/PlanDesignAccordion1`). Updating one without the other creates inconsistent display.

**Fix**: Whenever you change a PlanElement's copay or deductible-applies, update the matching PlanDesignAccordion1 row. See `field_mapping.md` for the mapping.

## 6. Embedded vs aggregate deductible (every plan type, including HDHPs)

**Symptom**: The file loads, but Claros shows an embedded plan as aggregate (or the reverse), so family deductible and OOP calculations come out wrong.

**Cause**: The flag was set by assumption (plan type, a default, or a compliance belief) instead of the SBC's family-deductible wording. Claros encoding: `EmbededDeductible="N"` = embedded, `"Y"` = aggregate (verified November 2026). Note that no law requires an embedded DEDUCTIBLE: the ACA embedding rule requires an individual out-of-pocket MAXIMUM inside family coverage, which is a MOOP rule, so an aggregate deductible HDHP is compliant and must be built as aggregate.

**Fix**: Read every plan's flag from its SBC ("each family member must meet their own individual deductible" = embedded, `embedded_deductible=True`; "aggregate" / family met only as a whole = `False`). HDHPs default to embedded only when the key is omitted; set it explicitly whenever the SBC states it.


## 7. HDHP Rx copay must be zeroed, not carried from the SBC

**Symptom**: Two runs of the same client produce different HDHP Rx tiers — one with `Copay1Min="15/30/45"`, one with `Copay1Min="0"`. Inconsistent, and the non-zero version is wrong.

**Cause**: On an HSA-qualified HDHP the member pays the full cost of every prescription until the medical deductible is met — there is NO separate Rx copay. SBCs often still print a post-deductible copay (e.g. "$15 generic after deductible"), and it's tempting to copy that number into `Copay1Min`. That double-counts: the deductible already applies, so a copay on top is incorrect. Setting `Deductible2Applies="Y"` alone is not enough; the copay value itself must be `0`.

**Fix**: For any `plan_type="HDHP"`, force `Copay1Min="0"` on all 16 Rx PlanElements AND `CopayMinT4="0"` on the matching accordion rows, regardless of what the SBC lists or the plan dict carries. Keep `Deductible1/2/3Applies="Y"` and `CoInsRate="1"`. The build script now does this automatically in the HDHP Rx branch and zeroes the accordion copay to match. Do not re-introduce the SBC's post-deductible copay for an HDHP.

**How to verify**: In the output, every `MedicalGrouping="RX"` PlanElement inside the HDHP scenario has `Copay1Min="0"`, and every `TierType="T4"` accordion row in that scenario has `CopayMinT4="0"`.


## 8. MOOP value leaking into the deductible field (HDHP)

**Symptom**: An HDHP named "$6800" ships with `Deductible2Individual="8300"` — the deductible field is holding the MOOP number. Family deductible and the T3 `FamDed` accordion inherit the same wrong value.

**Cause**: On the SBC, the deductible and the out-of-pocket maximum sit next to each other, and during extraction the MOOP gets read into the deductible slot. Nothing downstream catches it because both are valid dollar amounts.

**Fix**: Two safeguards. (1) When extracting, cross-check the plan name against the deductible — a plan called "$6800" should have a `$6,800` individual deductible. (2) The build script's `validate()` now asserts that every HDHP's individual and family deductible is strictly LESS than its matching MOOP, and fails the build with the offending plan named if `deductible >= MOOP`. If a plan genuinely has deductible == MOOP (a true 100%-after-deductible design, rare), override the check deliberately and note it to the user.

**How to verify**: Build completes past the "HDHP deductibles are below their MOOPs" validation line. Spot-check that `Deductible2Individual` matches the number in the plan's own name.

## Validation script (standalone spot-check)

**This is NOT a substitute for a build's validation.** Never use it to clear a
file this skill produced: a build is validated only by a clean `main()` run
printing `All checks passed` (see SKILL.md, "Validation before delivering").
This snippet covers 3 of the 10 checks and exists ONLY for spot-checking a
file from OUTSIDE a build — e.g. XML someone hands you.

The build script's `validate()` runs an attribute-aware version of the
signature check below automatically on every build (with one narrow, documented
copay-attribute exemption - see `SIG_ATTR_ADD_EXEMPT` in
`references/build_script.md`), plus nine more checks.

```python
import xml.etree.ElementTree as ET

def signature(elem):
    """Return tag + sorted attribute keys for elem and all descendants."""
    sig = [(elem.tag, tuple(sorted(elem.attrib.keys())))]
    for child in elem:
        sig.extend(signature(child))
    return sig

ref = ET.parse("reference.xml").getroot()
out = ET.parse("output.xml").getroot()

# Check 1: Structural match per scenario
for i, (rs, os_) in enumerate(zip(ref.findall("Scenario"), out.findall("Scenario"))):
    rsig = signature(rs)
    osig = signature(os_)
    assert rsig == osig, f"Scenario {i}: structural mismatch"
    print(f"Scenario {i}: {len(rsig)} elements match")

# Check 2: Namespaces present
with open("output.xml", "rb") as f:
    head = f.read(300)
assert b"xmlns:xsi" in head, "Missing xmlns:xsi"
assert b"xmlns:xsd" in head, "Missing xmlns:xsd"

# Check 3: CRLF line endings
with open("output.xml", "rb") as f:
    data = f.read()
assert b"\r\n" in data, "File is not CRLF-terminated"
print("All structural checks passed")
```

If any check fails, do not deliver the file. Fix and re-run.


## 9. Coinsurance formatting drift (1 vs 1.0, 0.8 vs 0.80)

**Symptom**: Two runs of the same plan produce files that differ only in how coinsurance is written — one has `CoInsRate="1"` and `CoInsPlanPaysPerc="0.8"`, the other has `CoInsRate="1.0"` and `CoInsPlanPaysPerc="0.80"`. Same meaning, different text, spurious diffs, and it makes consistency testing noisy.

**Cause**: Writing the value with a bare `str()` on a Python float emits `1.0`, while an int emits `1`. Whether the plan dict happened to carry `1` or `1.0` decided the output.

**Fix**: All coinsurance / plan-pays / coinsurance-percent values go through the `coins()` helper in the build script, which formats whole numbers as bare integers and fractions without trailing zeros — matching the untouched Claros reference XML, which uses the integer form (`CoInsRate="1"`). Never write these with a raw `str()`.

**How to verify**: `grep -o 'CoInsRate="[^"]*"'` on the output shows only forms like `1`, `0.8`, `0.6` — never `1.0` or `0.80`.


## 10. HDHP Rx-deductible rule leaking into PPO scenarios

**Symptom**: In a multi-plan file, more Rx tiers are marked `Deductible2Applies="Y"` than the plan designs call for. E.g. a file with one PPO (no Rx deductible) + one HDHP shows 40 Rx PlanElements subject to deductible when it should show 16, and 20 T4 accordion rows `Deductd2="True"` when it should show 8.

**Cause**: The rule "all Rx is subject to the medical deductible" is HDHP-only (IRS requirement, see #7). It's easy to over-generalize and apply it to PPO scenarios too. Note the reverse is NOT a rule: PPOs legitimately apply a deductible to some Rx tiers (a separate brand-drug Rx deductible, or "copay after the deductible" wording on the SBC) — when the SBC says so, the tier's `*_after_ded` flag is set `True` and `Deductible2Applies="Y"` is correct on a PPO.

**Fix**: The plan dicts' `*_after_ded` flags are the source of truth. The build script's `validate()` counts Rx PlanElements with `Deductible2Applies="Y"` and T4 accordion rows with `Deductd2="True"` in the OUTPUT and asserts they equal the expectation derived from the dicts: 16 PlanElements + 8 T4 rows per HDHP (forced regardless of flags), plus 2 PlanElements + 1 T4 row per non-HDHP tier flagged `after_ded=True`. A higher count means a deductible rule leaked into a tier whose dict says `False`; a lower count means a flagged tier never got written. Either direction hard-fails the build.

**How to verify**: Build prints "Rx deductible application matches plan dicts" with the counts. Change one `*_after_ded` flag and the expected count moves with it.


## 11. Coinsurance when deductible equals MOOP (derive 100%, don't read the SBC)

**Symptom**: Two runs of the same plan disagree on the HDHP in-network coinsurance — one writes `CoInsPlanPaysPerc="0.8"` (plan pays 80%), the other `"1"` (100%). The plan's in-network deductible equals its in-network MOOP.

**Cause**: When a tier's deductible equals its MOOP, the member hits the out-of-pocket maximum the instant the deductible is satisfied, so the plan pays 100% of everything after that — there is no coinsurance band, it has zero width. But the SBC still prints a coinsurance figure (e.g. "20% coinsurance"), and reading that figure literally and writing 80% is wrong for this structure. The two runs diverged on whether to trust the printed coinsurance or reason from the deductible/MOOP relationship.

**Fix**: Derive it. The build script checks, per tier, whether `deductible == MOOP`, and if so forces that tier's plan-pays to 1.0 regardless of the plan dict or the SBC line. Coinsurance is a function of the deductible/MOOP structure here, not a value to transcribe.

**How to verify**: For any tier where `Deductible2Individual == MOOPIndividual`, confirm `CoInsPlanPaysPerc="1"` on that Deduct block.


## 12. HDHP OON coinsurance carried over from the PPO (60% vs 80%)

**Symptom**: The HDHP's out-of-network band pays 60% in the file, but the HDHP's SBC says 80%. The 60% usually matches a PPO scenario's OON rate in the same file.

**Cause**: The OON coinsurance is read from the wrong plan — the model carries a PPO's OON rate (40% member / 60% plan) onto the HDHP instead of reading the HDHP's own OON figure. Unlike the in-network 100%-after-deductible case (#11), this value is NOT derivable from the deductible/MOOP numbers: a deductible below its MOOP tells you a coinsurance band exists, but not what percentage the plan pays in it. That percentage must be read off the HDHP's SBC.

**Fix**: One guard in `validate()` flags the suspicious pattern (WARN, not auto-fail, since a genuine split is possible): the HDHP OON band coinsurance differs from the HDHP's own in-net band. When it fires, the build prints a CONFIRM block, and the skill asks the user to verify the HDHP OON coinsurance against the SBC before the file is delivered. Do not auto-derive this value — read it.

*(v3.2 note: a former guard (b) also fired when the HDHP's OON rate MATCHED a PPO's OON rate in the same file. With a single carrier and one OON schedule across all plans — the normal case — that match is expected, and the guard fired on a correct file in POC testing. It was removed; the carried-rate error it targeted still trips guard (a) whenever the carried rate differs from the HDHP's own in-net band, which is the actual error signature.)*

**How to verify**: HDHP OON `CoInsPlanPaysPerc` matches the plan-pays percentage on the HDHP's own SBC OON column (e.g. `0.8` for an 80%-plan-pays OON band), and is not silently copied from a PPO.


## 13. Shell pre-flagged copay elements left at the `$1` placeholder

**Symptom**: After building, a rich-tier service (often ER, an outpatient
surgery, imaging, or lab line) shows a `$1` copay in Claros.

**Cause**: Claros shells frequently pre-flag certain Tier 1 services (grouping
`OP`/`RAD`/`LP`) as copay-type with a `$1` placeholder (`Copay1Flag="Y"`,
`Copay1Min="1"`), anticipating that the tier uses copays. Any logic that writes
coinsurance ONLY to elements that are NOT already flagged as copays will skip
these and leave the `$1` in the file. The shell's copay flags are a generic
template, not client-specific (a shell may flag Lab as a copay when the client's
rich-tier lab is actually no-charge), so never trust them as the client's design.

**Fix**: Handle EVERY element in the tier explicitly - populate the real copay OR
convert it to coinsurance. The build script's Tier 1 loop does this. After
building, assert no stray placeholders remain:
```python
stray = [pe.get("UIMedCatId_Sequence")
         for sc in root.findall("Scenario")
         for sp in sc.find("Plan").findall("SubPlan")
         for pe in sp.findall("PlanElement")
         if pe.get("Copay1Flag") == "Y" and pe.get("Copay1Min") == "1"]
assert not stray, f"stray $1 placeholder copays: {stray}"
```

**How to verify**: The build script's `validate()` now runs this exact assertion
(check 9) on every build and hard-fails if any placeholder survives; the build
prints "No stray $1 placeholder copays" when clean.

## 14. Multi-tier (3-network-tier) plans

**Symptom**: A plan has more than two network levels — either Pattern A
(rich/domestic tier plus broad tier plus OON, four real levels) or Pattern B
(one network but a sub-network copay split, e.g. BCI ChoiceDocs) — but the
output only has two tiers, or Tier 2 is empty, or ER/urgent-care copays are
missing from Tier 2 and OON.

**Cause**: The DEFAULT reference (`assets/blank_reference_2tier.xml`) has no
SubPlan 2. Building a multi-tier plan against it silently drops the middle
tier. Also, a few copays (ER, urgent care, ambulance) commonly apply in EVERY
tier, not just the rich tier, and are easy to leave as coinsurance in Tier
2/OON.

**Fix**: Use a 3-tier reference — the skill bundles one
(`assets/blank_reference_3tier.xml`, set `REF_FILE = DEFAULT_REF_3TIER`), or a
client's own 3-tier Claros export — and supply `t2_*` fields plus
`cross_tier_copays`. The reference choice is confirmed with the user on every
build ("The tier decision" in SKILL.md), never defaulted. Full methodology,
the 4-to-3 mapping decision, the accordion pattern, and a worked example are
in `references/multi_tier.md`; Pattern B specifically is section 10. Read it
whenever an SBC/SPD shows more than two in-network levels of either kind.

## 15. Preventive care overwritten by the tier loops (v3.1, fixed in v3.2)

**Symptom**: Preventive Care / Immunizations show coinsurance-after-deductible in Claros instead of "no charge, deductible does not apply." Occurred in **10 of 10 real-client POC tests** on v3.1, on every plan type.

**Cause**: The blank Claros export ships `PH` sequences 504/510 already covered-full (all `DeductibleNApplies="N"`, `CoInsRate="1"`) — and the v3.1 tier loops overwrote them with the tier default (`PH` is in `MED_GROUPINGS`, and no branch carved preventive out). Every validation check verified the BUILD, not the benefit content, so the wrong file passed all checks.

**Fix (v3.2)**: `PREVENTIVE_SEQS = ("504", "510")` branch runs FIRST in every in-network tier loop (SubPlans 1 and 2, all plan types including HDHP per the IRS safe harbor), writing covered-full via `set_planelement_covered_full()`. Validation **check 11** hard-fails any in-network preventive row not covered in full, so the v3.1 failure mode can no longer ship. OON preventive deliberately keeps the tier default (not ACA-mandated; SBCs mix it). Escape hatch for genuine non-ACA designs: `preventive_in_net_covered=False` in the plan dict, flagged to the user.

**Lesson**: the shell's own defaults are evidence. Before writing a grouping wholesale, check whether the blank export treats any of its rows specially.


## 16. Copay + coinsurance encoding: member share lives in `Copay1Perc`, not `CoInsRate`

**Symptom**: A "$100 Copay/visit, 30% Cost Sharing" ER line feeds the actuarial engine the wrong design; or an Rx specialty tier "shows 100% with the coinsurance in Participant %/Srv" and gets reported as a bug.

**Cause / verified encoding**: a verified hand-built export (2021 GAP/MOU, sequence 307) encodes copay+coinsurance as `Copay1Min="100"`, **`Copay1Perc="0.3"` (the MEMBER share)**, `CoInsRate="1"`, deductibles off — mirrored to the accordion as `CopayMinTn=100`, `ParticipantPctTn=30`, `CoinsurancePlanPctTn=100`. A v3.1 code path instead wrote the plan-pays rate into `CoInsRate` and forced `Deductible2Applies="Y"`, a materially different design. The same convention explains the Rx specialty display: `ParticipantPctT4` holding the member share while the plan percentage shows 100 is CORRECT, not a defect — say so in the output presentation so testers stop reporting it.

**Open question**: the verified export left the deductible OFF on that ER line even though its SBC printed "after Deductible." Combining `Copay1Perc` with `Deductible2Applies="Y"` is a composition of individually verified mechanics, not itself export-verified; the build supports it but flags it for in-Claros verification.


## 17. Separate Rx deductible: element-level flags vs SubPlan-level deductible (UNVERIFIED)

**Symptom / risk**: On a plan with its own prescription deductible (e.g. "$500 person for Prescription Drugs"), it is not yet verified how Claros expects the two layers to interact. The verified 2021 export carries the Rx deductible at the SubPlan 4 level (`Deductible2Individual="500"`) while leaving **all 16 Rx PlanElements at `DeductibleNApplies="N"`** — even though its SBC exempted generic but subjected brand tiers. The build (and check 6) instead drive `Deductible2Applies` per element from the `*_after_ded` flags.

**Until verified**: when an SBC carries a separate Rx deductible, set the `*_after_ded` flags from the SBC as usual, but list in the output presentation: "this plan has its own Rx deductible; confirm in Claros that the Rx deductible amount and the per-tier deductible checkboxes display as intended." If check 6 fires on such a plan and the flags genuinely match the SBC, the discrepancy may be this convention — confirm against a fresh Claros export of a separate-Rx-deductible plan and update this entry. Note the separate Rx MOOP (e.g. "$2,000/$4,000 for Prescription Drugs") has no verified home either; flag it as a manual Claros item.


## 18. Facility copays placed, half-placed, or skipped depending on the session (v3, fixed in v4.0)

**Symptom**: Three v3 sessions on identical BCBSOK SBCs, with identical intake answers, produced three different treatments of the inpatient facility copay (IP `101`-`105`) and the outpatient surgery facility fee (OP `309`/`311`): two placed them as copay + coinsurance with the deductible waived, one left all seven sequences at tier-default coinsurance and called them manual Claros items. 162 of the run-to-run's 222 attribute differences were this one decision. A second session put the IP copays into `t1_flat_copays`, which writes `IPCopayPerDay="N"` on the elements while the IP accordion row stayed coinsurance, and v3 validation passed it.

**Cause**: (a) the crosswalk marks these sequences "Client fill", and the stop rule fires on "NO confirmed sequence" without saying whether Client fill counts as confirmed, so each session decided; (b) on 2-tier plans there was no documented home for an inpatient copay (`t1_ip_copay` was described as a multi-tier rich-tier field only), so each session improvised one.

**Fix (v4.0)**: the session no longer decides. Copay-style plans must carry `facility_lines` (the SBC's hospital-stay, mental-health-inpatient, childbirth-facility and outpatient-surgery facility rows, transcribed). `resolve_facility_read()` places them under one fixed rule: IP placed when the three IP rows agree, otherwise the build STOPS for the operator's answer to pinned question F1; OP surgery placed on both `309` and `311` from one row, on each from a source-labeled hospital/ASC split, and on neither (flagged) from an unlabeled split; OP Other `308` from an explicit `op_other` line, otherwise the same value as `309` (builder decision; a deliberate change from v3's tier default). Facility sequences in any other per-sequence map are refused before the build starts, and **check 12** hard-fails any built facility row that differs from the resolved read. The resolved read prints under "TREATED AS A READ: FACILITY ROWS" for the output presentation.


## 19. Inpatient display row disagrees with the calculation rows (v3, fixed in v4.0)

**Symptom**: Claros calculates one inpatient design and displays another. Ben Choate's Run A wrote `IPCopayPerDay="D"` on the IP accordion row and `"A"` on the five IP PlanElements; Run B wrote IP copays with `IPCopayPerDay="N"` on the PlanElements under an accordion row showing 90% coinsurance and no copay. Both passed v3 validation. A reviewer checks the display, so the display is the worst place to be wrong.

**Cause**: the basis was hard-coded as `"D"` in two separate functions (a session patched one), there was no Per Admit option at all, and check 8 neither compared `IPCopayPerDay` nor checked copay-typed IP rows against the rollup (the exemption meant for ER under OP covered IP too).

**Fix (v4.0)**: `IP_BASIS_CODES` is the single mapping; both writers take the code as a required argument; inpatient copay lines in `facility_lines` carry a transcribed `basis`; check 8 has a dedicated IP block with no copay exemption that compares copay, member share, plan pct, deductible checkboxes and `IPCopayPerDay`, and rejects any IP copay row whose code is not Per Admit or Per Day.

**Mapping status and the BASIS TEST.** Per Admit = `"A"` rests on the blank shells shipping `"A"` with the UI defaulting to Per Admit; Per Day = `"D"` rests on the CRMC client fill. Ben's Run A file (display `"D"`, calculation `"A"`) displayed Per Admit, which fits the mapping only if Claros sets the radio from the calculation rows. To close it: import the three-scenario BASIS TEST file into a scratch Claros case and read the Inpatient (IP) radio on each. Scenario 1 (`A` on both layers) should read Per Admit and Scenario 2 (`D` on both) Per Day; if they read the other way, swap the two codes in `IP_BASIS_CODES` and nowhere else. Scenario 3 (display `A`, calculation `D`) shows which layer the radio follows. Record the result here.


## Diagnosing future upload failures

When the user reports a Claros upload error:

1. Get the exact error message Claros displays. "Data could not be loaded" is generic and could be any of the issues above.
2. Diff the output against the reference using the value-level walker:
   ```python
   def walk(elem, path=""):
       items = [(path + "/" + elem.tag, elem.attrib)]
       for i, child in enumerate(elem):
           items.extend(walk(child, f"{path}/{elem.tag}[{i}]"))
       return items
   ```
   Look for any field where `out` has `0` or `""` and `ref` had a real number — that's almost always the culprit.
3. Compare the structural signature too — element counts must match.
4. Last resort: try uploading the unmodified reference XML to verify Claros itself is working, then bisect the changes you made.
