---
name: "sbc-to-clarosvtest"
description: TEST BUILD of the Claros SBC-to-XML skill, for side-by-side comparison with sbc-to-clarosv5. Use ONLY when the user explicitly invokes "/sbc-to-clarosvtest" or names "sbc-to-clarosvtest" or "vTest". Builds a Claros-compatible XML plan file from carrier Summary of Benefits and Coverage (SBC) PDFs (PPO, HDHP, HMO, EPO, and multi-tier plans with 3+ network levels), using a Claros XML as a structural reference. Do NOT trigger on general Claros or SBC requests without that explicit invocation; those belong to sbc-to-clarosv5.
---

# sbc-to-clarosvtest

**Version 5.1-test** (vTest, built on v5.1: the verified service crosswalk and PP display fix, plus the November 4 evaluator findings; version history: `references/changelog.md`). A test build for side-by-side comparison with `sbc-to-clarosv5`; it does not supersede it.

Build a Claros-compatible XML plan file from carrier SBC PDFs, using an existing Claros XML as a structural reference. The output preserves every structural element of the reference so Claros accepts the upload, while the plan design (deductibles, MOOPs, copays, coinsurance) reflects the new SBCs.

## When this skill applies

The user has:
1. One or more SBC PDFs (typically a medical PPO and/or HDHP for the same client)
2. A goal of uploading the new plan to Claros for actuarial modeling

Two blank Claros XML references are bundled: `assets/blank_reference_2tier.xml` (SubPlans 1 + 3, the default for ordinary single-in-network plans) and `assets/blank_reference_3tier.xml` (SubPlans 1 + 2 + 3, for any multi-tier build — set `REF_FILE = DEFAULT_REF_3TIER`). The user may also provide a different reference file (typically a prior client's Claros XML) — if they do, use theirs instead and pass it to the build script as `REF_FILE`.

## High-level workflow

1. Read the reference guidance in `references/field_mapping.md` and `references/known_issues.md` BEFORE writing any code. The known_issues file documents landmines that have caused real upload failures — skipping it will produce a file that Claros rejects. **If the SBC/SPD shows more than two network levels of ANY kind**, ALSO read `references/multi_tier.md` first. That includes BOTH multi-tier patterns: (A) provider-steerage plans with a rich/domestic tier plus a broad tier plus OON (different ded/MOOP/coins per tier, e.g. UMR/CRMC), and (B) sub-network copay splits where one in-network copay cell prints two rates (e.g. BCI "ChoiceDocs = $X; All other In-Network = $Y") — same ded/MOOP/coins across both in-network levels, only office/urgent-care copays differ. The reference covers the 4-to-3 tier mapping, the Pattern B configuration (t2_pcp_copay / t2_specialist_copay / t2_flat_copays), single-in-network plans inside a 3-tier file (tier2_active / tier_split), the service-ID crosswalk, rich-tier and cross-tier copays, and the accordion pattern.
2. Ask the user clarifying questions (see "Clarifying questions" below) before parsing.
3. Extract the SBC plan design values into a small dictionary per plan.
4. Extract and run the build script WITHOUT editing it (see "Build script usage" below). The script is stored as inert text in `references/build_script.md`; extract it with the one command given there rather than reading the whole file or retyping it, and run it through a small `run_build.py` that sets only `REF_FILE`, `OUT_FILE`, the three date constants and `PLAN_DICTS`. Never change the script's logic. If a plan design cannot be expressed through the plan dict, STOP and tell the user exactly what cannot be modeled; do not patch the script to make it fit.
5. Validate the output against the reference (structural match check, namespace check, CRLF check).
6. Present the final XML file to the user with a summary of placeholder fields they will need to update in Claros.

## The tier decision — ask this on EVERY build, before anything else

Two blank references ship with this skill and the wrong one cannot be recovered from: a multi-tier plan built on the 2-tier reference silently drops the middle tier and models coverage the plan does not have. The reference choice is therefore a confirmed decision on every single build, never a default you fall into.

| Reference | Set `REF_FILE` to | Structure | Use for |
|-----------|-------------------|-----------|---------|
| `assets/blank_reference_2tier.xml` | `DEFAULT_REF` | SubPlans 1 + 3 | One in-network level plus OON. Ordinary PPO / HDHP / HMO / EPO. |
| `assets/blank_reference_3tier.xml` | `DEFAULT_REF_3TIER` | SubPlans 1 + 2 + 3 | Any plan with more than two network levels: Pattern A steerage, or Pattern B sub-network copay split. |

A client's own Claros export may be substituted for either by pointing `REF_FILE` at its path.

**How to ask.** Read the SBCs first, form your own read, then put the question to the user with your read as the recommendation. Do not ask blind (most users do not think of their plan in terms of Claros SubPlans), and do not decide silently. The question is asked even when the answer looks obvious.

The read is mechanical. Scan every SBC's benefit table for network columns and for any single in-network cell printing two rates:

- **One in-network column, one rate per cell** → 2-tier. Recommend `DEFAULT_REF`.
- **A rich/domestic or preferred tier alongside a broad tier** (different deductible, MOOP, or coinsurance per tier) → Pattern A, 3-tier. Recommend `DEFAULT_REF_3TIER`.
- **One shared in-network deductible/MOOP/coinsurance, but an office-visit or urgent-care cell printing two copays** (e.g. BCI "ChoiceDocs $X; All other In-Network $Y") → Pattern B, 3-tier. Recommend `DEFAULT_REF_3TIER`.
- **Mixed file** (one plan has a sub-network split, another does not, typically the HDHP) → 3-tier for the whole file. Build the non-split plan as an effective 2-tier scenario inside it: `tier2_active=False`, `t2_*` mirroring Tier 1, zero Tier 2 weight in `tier_split`. Never split a client's plans across two reference structures in one upload.

Put it to the user roughly like this, substituting what you actually found:

> I read the [client] SBCs as **[2-tier / 3-tier, Pattern A / 3-tier, Pattern B]** because [the specific cell or column that decided it, quoted from the SBC]. That means building on `blank_reference_[2tier/3tier].xml`. Does that match how the plan actually works, or is there a network level the SBC does not show?

If the user corrects you, take the correction. The SBC is a summary and does not always show a steerage tier that the SPD does. If the user is unsure, ask for the SPD before building rather than guessing, and read `references/multi_tier.md` before going further.

Once confirmed, set `REF_FILE` explicitly in `run_build.py`, and state the reference you used in the output presentation so the choice is visible in the record.

## Clarifying questions to ask first

After the tier decision above, ask the user (use `ask_user_input_v0` if available, otherwise inline). **`ask_user_input_v0` ENDS YOUR TURN**: any question typed in chat prose immediately before the tool call is never answerable — the user only sees and responds to the tool's options. Everything you need answered goes INSIDE the tool call, or none of it does; never mix the two in one message. If a gating question goes unanswered (tool failure, user skips it), STOP and re-ask — do not proceed on a labeled fallback. Both failure modes happened in POC testing and shipped a build past an unanswered gate.

1. **Scenario contents**: The output is ALWAYS one file, with every plan as a scenario inside it (`TotalPlans=N`) — never separate files per plan. Confirm which plans go in the file and in what order (the order sets the scenario order in Claros).
2. **Effective date**: What date should appear in `TrendDate` and the file name? SBCs typically use 1/1/YYYY but Claros file naming convention is often YYYY-10-1 — confirm which the user wants. For a NON-CALENDAR plan year (e.g. 12/01/2025–11/30/2026), `ManualForDataYear` takes the plan year's START year (`2025` in that example) — a builder decision, not a guess; state it in the output presentation.
3. **Placeholder strategy** (NOT asked; stated under "Treated as a read") for fields not in the SBC (census, enrollment, stop loss, employee contributions): use minimal placeholders ($1 fees, 1 EE, 1% admin) so the file loads, then update in Claros. This is the recommended default. Explain that all-zero values cause Claros upload errors due to divide-by-zero in actuarial calculations.
4. **OON deductible**: SBCs sometimes say "all Networks" (in-net = OON) and sometimes split them. Confirm interpretation if ambiguous.
5. **Scenario naming convention**: Confirm the exact scenario names with the user once, then reuse them verbatim. To keep runs consistent, follow a fixed pattern and do not vary it between builds: `<Network> <PlanType> <DeductibleAmount>`, e.g. `ASC PPO 1000`, `ASC HSA HDHP 6800` (omit the network token when the client has a single network, e.g. `PPO 1000`). Rules: (a) NO client name inside scenario names — the client is identified by the file name (`<Client>_Eff_<date>.xml`) and the Claros case, so repeating it in every plan name is noise; (b) no `$` sign anywhere in the name; (c) the deductible number is the individual in-network deductible, digits only; (d) the plan-type token (`PPO`, `HDHP`, `HMO`, `EPO`) must match the actual plan design and must never be swapped between plans (an HSA plan is `HSA HDHP`, not `HSA PPO`); (e) if the client uses an HSA-qualified HDHP, write `HSA HDHP`, not just `HDHP`, only if the SBC/name calls it HSA. When in doubt, mirror the client's own plan labels from the SBC exactly.
6. **Multi-tier follow-ups** (only when "The tier decision" above landed on 3-tier): the reference itself is already settled, so this question covers only what comes after it. For Pattern A, confirm the 4-to-3 mapping (which real network level gets dropped; the usual answer is Claros Tier 1 = richest, Tier 2 = broad, OON = OON, dropping the preferred tier). For either pattern, tell the user the tier utilization split (`tier_split`) will ship as a placeholder they must set in Claros from actual claims routing, because it is never in an SBC. Offer a starting default rather than bare `$1`-style values: a verified hand-built client file used 95/0/5 (in-net / middle / OON) for an effectively single-in-network plan — propose a sensible split like that, let the user override, and still flag it as a set-in-Claros item. If the user would rather collapse a genuinely multi-tier plan down to 2-tier for simplicity, that is their call to make explicitly, not yours to make silently: ask which level's copays to load and record the collapse as a judgment call in the output presentation.
7. **Embedded vs aggregate deductible — EVERY plan type, not just HDHPs**: this is a plan fact read off the SBC's family-deductible language, and the build refuses to start ANY plan without it, HDHPs included (`embedded_deductible` is a REQUIRED plan-dict key; earlier versions set this flag by plan type instead of reading the SBC). Read rule: "each family member must meet their own individual Deductible" / "per person" = embedded (`True`); family deductible met only in aggregate = aggregate (`False`). Read it per plan and confirm your read with the user when the SBC's wording is ambiguous. The plan dict always uses True = embedded; the build writes Claros's inverted letter (`"N"` = embedded, `"Y"` = aggregate), so never reason about the letter directly. If the user overrides the SBC wording, their call governs and gets recorded as a judgment call. Read the HDHP's SBC the same way as any other plan and set `embedded_deductible=False` when it reads aggregate (e.g. "100% aggregate"). No law requires an embedded deductible: the ACA embedding rule applies to the out-of-pocket MAXIMUM (an individual MOOP inside family coverage), not the deductible, so never override an SBC that says aggregate on compliance grounds.
8. **HDHP coinsurance** (in-net and OON; asked only when check 7's CONFIRM block prints, otherwise stated under "Treated as a read"): read each band's plan-pays percentage off the HDHP's OWN SBC, not from a PPO in the same file. Two rules: (a) when a tier's deductible equals its MOOP, the plan pays 100% after deductible for that tier (derived automatically, no coinsurance band exists); (b) when a tier's deductible is below its MOOP, there IS a band and the plan-pays percentage must be read from the SBC — it is NOT derivable from the numbers. The build's coinsurance check flags an OON rate that differs from the HDHP's own in-net band; **if that CONFIRM block prints, ask the user to verify the HDHP OON coinsurance against the SBC before delivering.** (A former second guard that compared the HDHP's OON rate against the PPO's was removed in v3.2 — one carrier, one OON schedule is the normal case and it fired on correct files.) See known_issues.md #12.
9. **Copay inventory — walk the whole SBC benefit table, then confirm it**: list every line that prints (a) a dollar copay, (b) a coinsurance rate different from the plan's default, or (c) "Deductible does not apply" on a coinsurance line. Map each to its home: office copays fill the `PH` grouping automatically; the four facility rows (hospital stay facility fee, mental health inpatient, childbirth/delivery facility, outpatient surgery facility fee) are transcribed into `facility_lines` exactly as printed and the BUILD decides their sequences (see "Facility rows" below; do not place them yourself); ER (`307`), urgent care (`312`) and every other named service row go in `t1_flat_copays` (copay) or `t1_service_coins` (off-default or deductible-waived coinsurance); since v5.1 EVERY medical sequence is verified by name (`field_mapping.md` crosswalk: e.g. complex imaging `601`/`602`, routine imaging `604`, lab `201`, PH Therapy `514`, PH Chiro `502`, PH Mental Health `507`, PH Anesthesia `501`, PH Surgery `513`, DME `404`, ambulance `402`, hospice `406`). The PH rows default to the specialist copay, so walk them by name and override any the SBC prices differently (rehab therapy and physician/surgeon fees are often coinsurance). What still stops is only what the XML cannot express on one row (a place-of-service split on one SBC line such as "office visit copay or 30% coinsurance", day-band benefits, telehealth, OON-only exceptions on a 2-tier build): it is NOT guessed and NOT silently coinsured — list it to the user now and again as a set-manually-in-Claros item in the output presentation. Show the user the inventory table and get confirmation before building. Six of ten v3.1 POC failures were copays the SBC printed and the build never placed; this question plus the post-build coverage table are the two ends of the countermeasure.

### How to batch these questions (fixed rounds, every build)

`ask_user_input_v0` takes at most 3 questions per call and ends the turn, so the questions always go in these rounds, in this order. Do not reorder, merge, or split them differently between runs; the same SBCs should produce the same questions in the same order.

- **Round 1 (one tool call):** The tier decision (your read and the deciding SBC cell go in the question text itself); question 1 (which plans, in what order); question 2 (effective date).
- **Round 2 (one tool call):** question 7 (embedded vs aggregate, one question stating your read for every plan); question 5 (scenario names, proposed in the fixed pattern); question 4 (OON deductible) only when the SBC is ambiguous, otherwise stated under "Treated as a read".
- **Round 3 (one tool call, 3-tier builds only):** question 6 (the 4-to-3 mapping for Pattern A, and the `tier_split` starting default).
- **Round 4 (inventory):** question 9. Show the copay inventory table in chat, then put one confirm question in the tool call with the options "Inventory is correct, build it" and "Something is missing or wrong". A table shown for review is not a question, so this does not break the no-prose-questions rule above.

Skip a round only when none of its questions apply. Pinned question F1 and question 8 are asked only when their triggers fire, each in its own call. Question 3 is never asked.

## Treated as a read (stated to the operator, never asked)

These decisions are made by a fixed rule, not by the session and not by the operator. Do NOT turn any of them into a question. State the result in the intake message under this exact heading, **Treated as a read**, so the operator sees what was decided, and repeat it in the output presentation.

- **Placeholder strategy (question 3).** Census, enrollment, stop-loss and contribution fields get the minimal placeholders in `references/field_mapping.md`, "Fields to placeholder", never zeros. State it as: "Placeholders: minimal non-zero values so the file loads; update census, enrollment and stop loss in Claros."
- **HDHP coinsurance (question 8).** Each HDHP band's plan-pays rate is read from the HDHP's own SBC (100% where deductible equals MOOP). State the rates read. Ask only if check 7 prints its CONFIRM block.
- **Facility rows (IP 101-105, OP 308/309/311).** Read from the SBC into `facility_lines` and placed by the build under the rule in `references/field_mapping.md`, "Facility read". State it as: "Facility rows: inpatient [value, including per admit or per day] on all five inpatient rows; outpatient surgery [value] on hospital and ASC [or: labeled split / unlabeled split, flagged]; outpatient other [value] (explicit line, or the same as outpatient surgery hospital)." The only exception is a disagreement among the three inpatient rows, which triggers pinned question F1 below.

## Pinned conditional questions

A pinned conditional question is asked ONLY when its trigger fires, and when it fires it is asked with exactly the wording and options below (through `ask_user_input_v0`, inside the tool call, under the rule in "Clarifying questions to ask first"). Only the bracketed slot in the question text is filled in; the option text is never reworded, reordered, or shortened, so the same letter always means the same answer.

**F1. Inpatient facility rows disagree.** Trigger: the build stops with `STOP - ASK THE OPERATOR (pinned question F1)`, or your `facility_lines` read already shows the three inpatient rows differ. Ask before building:

> Question: "The SBC prints different inpatient facility cost sharing: [each of the three rows with its value, as the build printed it]. Claros cannot place these per inpatient subtype. What should all five inpatient rows carry?"
> Options, in this order:
> 1. "Hospital-stay value on all five (Recommended)"
> 2. "Tier default coinsurance on all five"
> 3. "One value I will specify"

Record the answer as `facility_lines["ip_discrepancy_choice"]`: option 1 = `"hospital_stay_all"`, option 2 = `"tier_default"`, option 3 = `{"custom": <the value the operator gives>}` (ask for the value in the same form as a facility line if they did not give it). Option 1 is recommended because the hospital-stay row covers medical and surgical admissions, the bulk of inpatient spend. Whichever is chosen, the build lists the differing rows as set-manually-in-Claros items. Never set `ip_discrepancy_choice` without the operator's answer, and never leave one in place when the rows agree (the build refuses a stale answer).

## Critical landmines (read `references/known_issues.md` for details)

**Source-document gate — run BEFORE any extraction.** Confirm each uploaded PDF is a MEDICAL SBC (or SPD) with selectable text. If a document is image-only/scanned, is a dental/vision/other non-medical summary, or has values that will not extract cleanly (garbled numbers, cut-off tables), STOP and tell the user what you got and what you need — do not infer, approximate, or OCR-guess dollar values into a plan dict. Every downstream check validates the BUILD, not the READING: a mis-extracted deductible passes all twelve checks and ships a wrong actuarial model. Two related rules: when both an SBC and SPD are supplied, state in the output which values came from which document; and when the SBC is silent on a required value (commonly the OON MOOP), ask rather than guess — the output-presentation section requires flagging that judgment call anyway. If the SBC and SPD disagree on the same value, do not pick one - present both values to the user and ask which governs before building.


These five issues have all caused "Data could not be loaded" errors or other failures in Claros uploads. Every output must address them:

1. **XML namespaces must be re-injected**: Python's ElementTree strips `xmlns:xsi` and `xmlns:xsd` from the root unless explicitly handled. Always add them back to the root tag during serialization.
2. **CRLF line endings required**: Claros expects Windows-style line endings. Convert LF to CRLF before writing.
3. **Minimal placeholders, never zeros**, for fields Claros uses in math: `TotalEE`, `EEOnlyPct`, `AdultEquivalent`, `DependentPerEmployee`, all `StopLoss` fees, all `ReimbursementAccount` contributions. Zeros cause divide-by-zero failures.
4. **Don't delete elements** like `<Zip>` entries — zero out their values instead. Claros may validate against an expected element count.
5. **Encoding declaration quirk**: Claros exports XML with a declaration claiming `encoding="utf-16"` but the bytes are actually UTF-8. The `load_reference()` function in the build script strips this declaration before parsing. The output should be written WITHOUT an XML declaration to match the format that Claros consistently accepts.

## Plan design transformation rules

These are the SBC values that change in the XML. See `references/field_mapping.md` for exact XPath locations and SBC sources.

### Medical deductibles and MOOPs

The `<Plan>/<SubPlan>/<Deductible>/<Deduct>` element holds the plan-level deductibles and MOOPs:
- `SubPlanId="1"` (NetworkId="1000") = in-network Tier 1
- `SubPlanId="3"` (NetworkId="3000") = out-of-network Tier 3
- `SubPlanId="4"` (NetworkId="4000") = Rx (only present in Ada-style references; in blank/exported references, Rx is folded into SubPlan 1)

The build script auto-detects which structure the reference uses. Don't worry about which one is in front of you — `get_rx_planelements()` finds the right SubPlan.

Update `Deductible2Individual`, `Deductible2Family`, `MOOPIndividual`, `MOOPFamily` and the matching `EESP`/`EE1C`/`EECC` family-tier copies. Set `CoInsPlanPaysPerc` to the plan-pays percentage (0.80 if SBC says "20% coinsurance", 0.60 if SBC says "40% coinsurance"). **Exception — derive, don't read:** when a tier's deductible equals its MOOP, the plan pays 100% of claims after the deductible (the MOOP is reached the moment the deductible is paid, leaving no coinsurance band), so set that tier's plan-pays to 1.0 regardless of any coinsurance line the SBC prints. The build script derives this automatically. See known_issues.md #11.

`EmbededDeductible` comes from the plan dict's `embedded_deductible` key, and Claros encodes it inverted: `"N"` = embedded, `"Y"` = aggregate (verified against a Claros export, November 2026; the build handles the inversion, so plan dicts keep True = embedded) — REQUIRED for every plan type, HDHPs included (the build raises a KeyError without it; see clarifying question 7 for the SBC read rule). Earlier versions set this flag by plan type instead of reading the SBC; never default this value. Supported `plan_type` values are exactly `PPO`, `HMO`, `EPO`, and `HDHP`; the build refuses to start on anything else.

### Rx tiers with coinsurance plus a dollar maximum

Some SBCs price specialty drugs as a percentage with a dollar cap, e.g. "25% up to $215 maximum after deductible." The skill supports this by allowing the Rx tier value in the plan dict to be either:
- A scalar (e.g. `"rx_specialty_retail": 50`) - treated as a fixed copay
- A dict (e.g. `"rx_specialty_retail": {"coins": 0.25, "max": 215}`) - treated as a coinsurance with a per-script dollar cap

When the dict form is used, the script sets `Copay1Perc` and `Copay1Max` on the PlanElement (instead of `Copay1Min`) and sets `ParticipantPctT4` and `CopayMaxT4` on the matching PlanDesignAccordion1 row.

If the SBC lists only one "Specialty drugs" line (no separate mail-order specialty), the skill mirrors the same SBC value to BOTH retail and mail specialty tiers. Some carriers (e.g. Medical Mutual) route specialty drugs only through specialty pharmacies, making mail-order specialty effectively not covered — the build does NOT model that case, because the correct "not covered" representation in the Claros XML has not been verified against a real export. When an SBC says mail specialty is not covered, build the file with the mirror as-is and explicitly list "mail-order specialty shows retail specialty pricing — mark not covered in Claros" in the placeholder-fields summary presented to the user. (A prior version documented a `MAIL_SPECIALTY_NOT_COVERED` flag that no code ever read; it has been removed rather than left as a false promise.)

### Service-level copays and coinsurance (copay-style plans: PPO, HMO, EPO)

This layer is where 10 of 10 v3.1 POC builds failed — read it fully, then run clarifying question 9's copay inventory against the whole SBC benefit table.

**Office copays fill the ENTIRE physician grouping.** The `PH` grouping holds 14 service rows (PCP `509`, Specialist `511`, preventive `504`/`510`, plus 10 unnamed physician rows: office surgery, therapy visits, MH office, allergy, etc.). A verified hand-built client export fills the office copay across ALL of them, not just 509/511 — copaying two rows and coinsuring the other twelve is what testers saw as "the Specialist copays did not fill in the drop down / Physician other." The build does this automatically: 509 takes `pcp_copay`, 511 takes `specialist_copay`, the other non-preventive rows take the specialist copay (override wholesale with `ph_other_copay`, or per-sequence via `t1_flat_copays`). A scalar `0` for `pcp_copay`/`specialist_copay` means NO office copay (coinsurance-only design) and the rows fall to the tier default rather than a free-care $0 copay.

**Copays beyond the office — `t1_flat_copays` is a first-class 2-tier field, NOT multi-tier-only.** Nearly every PPO/HMO/EPO SBC prints per-visit copays past the office lines: ER (`307`) and urgent care (`312`) at minimum. Place every dollar copay the inventory found, e.g. `"t1_flat_copays": {"307": {"copay": 250, "plan_pays": 0.80}, "312": 50}`. Omitting them ships those services as coinsurance with no error.

**Facility rows: `facility_lines` (REQUIRED on every PPO/HMO/EPO, 2-tier and multi-tier, v4.0).** Transcribe the SBC's four facility rows into the plan dict, one key each, exactly as printed. Do not decide where they go: the build's facility read places them under one fixed rule, so identical SBCs always produce identical facility rows. Under v3, three sessions on the same SBCs placed, half-placed, and skipped these rows, which was 162 of 222 run-to-run differences (known_issues.md #18). Example for a row printing "$150 copay/admission, 10% coinsurance, deductible does not apply" on all three inpatient rows and "$100 copay, 10% coinsurance, deductible does not apply" for outpatient surgery:

```python
"facility_lines": {
    "ip_hospital_stay": {"copay": 150, "plan_pays": 0.90, "basis": "per_admit"},
    "ip_mental_health": {"copay": 150, "plan_pays": 0.90, "basis": "per_admit"},
    "ip_childbirth":    {"copay": 150, "plan_pays": 0.90, "basis": "per_admit"},
    "op_surgery":       {"copay": 100, "plan_pays": 0.90},
},
```

**Inpatient copay basis (Per Admit / Per Day).** Every inpatient copay line carries `"basis"`, transcribed from the SBC's wording: `"per_admit"` for per admission / per stay / per confinement, `"per_day"` for per day, and `"not_stated"` when the row gives a copay with neither. `"not_stated"` builds Per Admit (the Claros default) and is flagged for confirmation in Claros (builder decision, October 2026). The basis is part of the row's value, so three inpatient rows that differ only in basis trigger pinned question F1. The build writes ONE Claros code to both the calculation rows and the display row from a single mapping (`IP_BASIS_CODES`), and check 8 fails any file where they differ. A "$X per day up to $Y per admission" cap still cannot be modeled; build `"per_day"` and flag the cap.

`None` means the row prints the tier's ordinary coinsurance after the deductible. A labeled hospital/ASC split uses `op_surgery_hospital` + `op_surgery_asc`; an unlabeled split is `"op_surgery": "UNLABELED_SPLIT"`; OP Other (`308`) takes an explicit `op_other` line when a source line names it; without one, it carries the same value as OP Surgery - Hospital (`309`). The build refuses `t1_ip_copay` set directly, any facility sequence (`101`-`105`, `308`, `309`, `311`) keyed into `t1_flat_copays` or `t1_service_coins`, and IP sequences in any per-sequence map. Full rule and row wording: `references/field_mapping.md`, "Facility read". HDHPs do not carry `facility_lines`.

**Copays that add coinsurance and/or keep the deductible.** For an SBC line like "$100 Copay/visit, 30% Cost Sharing," write a dict anywhere a copay is accepted: `{"copay": 100, "plan_pays": 0.7}`. VERIFIED ENCODING (2021 GAP/MOU export, ER row): the member share goes to `Copay1Perc` (`"0.3"`) with `CoInsRate` staying `"1"` — never write the plan-pays rate into `CoInsRate` for these (a prior version did, which feeds the actuarial engine a different design). Accordion mirror: copay in `CopayMinTn`, member share in `ParticipantPctTn`, `CoinsurancePlanPctTn=100`; check 8 verifies it. Adding `"after_ded": True` sets `Deductible2Applies="Y"` — supported, but the verified export left the deductible OFF even where its SBC said "after Deductible," so flag after-ded copay+coins combos as verify-in-Claros items in the output presentation. A scalar keeps the historical meaning (deductible waived, plan pays 100% after the copay).

**Per-service coinsurance exceptions — `t1_service_coins`.** For services whose coinsurance differs from the tier default (DME `404` at 50% under a 70% plan) or that waive the deductible ("30% Cost Sharing, Deductible does not apply" on diagnostic x-ray/blood work — a recurring Cigna/BCBS pattern v3.1 could not express): `"t1_service_coins": {"404": 0.50, "604": {"plan_pays": 0.70, "after_ded": False}}`. Check 8 verifies these elements against the declared overrides. Use the verified names in `field_mapping.md`; a line the XML cannot express on one row follows the stop rule there (tell the user, never guess).

For HDHPs (no upfront copay), leave `Copay1Flag="N"` and set all three `DeductibleNApplies="Y"` with `CoInsRate` = the HDHP's own in-network plan-pays rate from its SBC (question 8) — the build does this for every HDHP element except preventive.

### Preventive care — covered in full, in-network, on EVERY plan type

`PH` sequences `504`/`510` (preventive/immunizations) are covered at 100% with no deductible in-network on ACA plans, HDHPs included (IRS safe harbor). The build writes them covered-full in SubPlans 1 and 2 BEFORE any grouping fill, and validation check 11 hard-fails any in-network preventive row that is not — this was the universal v3.1 failure (the blank export even ships these rows correct; the tier loops overwrote them). OON preventive is not mandated and keeps the OON tier default; if the SBC's OON column says otherwise, note it as a judgment call. Escape hatch for genuine non-ACA designs whose SBC literally prints preventive cost sharing: `preventive_in_net_covered=False` in that plan dict, flagged to the user. Full detail: `field_mapping.md`, "Preventive care."

### HMO / EPO plans with no out-of-network coverage

Some HMO and EPO designs have NO out-of-network benefit at all. The correct
"not covered" representation for the MEDICAL OON tier has not been verified
against a real Claros export, so do not invent one. Rule:

1. Detect it at the source-document gate: if the SBC's OON column says
   "Not covered", "No coverage", or shows no OON deductible/MOOP, flag it
   before extraction.
2. ASK THE USER which convention to build (do not choose silently):
   (a) mirror the in-network deductible/MOOP into the OON fields, or
   (b) use a high sentinel (e.g. ded 50000 / MOOP 50000).
3. Whichever is chosen, the output presentation MUST list it as a judgment
   call: "OON tier shows placeholder values - this plan has no OON coverage;
   confirm the correct representation in Claros after upload."

Never leave the reference shell's OON template values in place for a
no-OON plan, and never present the file without the flag above.

### Rx copays

The Rx PlanElements (16 rows; in SubPlan 4 in Ada-style references, folded into SubPlan 1 in the bundled blank references, and found automatically by `get_rx_planelements()`) cover the Generic/BrandFormulary/BrandNonFormulary/Specialty × Retail/Mail × New/Maintenance grid. See `references/field_mapping.md` for the full Sequence ID map.

For each, set `Copay1Min` to the SBC copay and `Deductible2Applies="Y"` if the SBC says "after the deductible has been met", otherwise `"N"`.

**Plans with their own prescription deductible** (e.g. "$500 person for Prescription Drugs"): how Claros expects the SubPlan-4-level deductible and the per-element checkboxes to interact is NOT yet verified — a verified hand-built export carried the $500 at the SubPlan level with all 16 element flags `"N"`, while this build drives the flags from the `*_after_ded` keys. Set the flags from the SBC as usual, and ALWAYS list in the output presentation: "this plan has its own Rx deductible — confirm the Rx deductible amount and per-tier deductible checkboxes in Claros after upload." A separate Rx MOOP has no verified home either; flag it as a manual Claros item. See known_issues.md #17.

For HDHPs, ALL Rx tiers are subject to the medical deductible per IRS rules — set all to `Deductible2Applies="Y"` AND set `Copay1Min="0"` on every Rx tier (and `CopayMinT4="0"` on the accordion). On an HSA HDHP the member pays full cost until the deductible is met, so there is no separate Rx copay; do not carry the SBC's post-deductible copay into the copay field. The build script enforces this automatically. See known_issues.md #7.

### UI accordion mirroring

For every change to the `<PlanElement>` rows (which drive the actuarial math), make the matching change to the `<PlanDesignAccordion1>` rows (which drive the Claros UI display). Otherwise the file loads but displays wrong values to the user.

The UI mirror lives at `Scenario/MedCategoryParentNode/PlanDesignAccordion1ParentNodes`. Family deductibles/MOOPs go in the sibling `PlanDesignAccordion2ParentNodes`.

**Dropdown inheritance (parent rows and child services).** Each `PlanDesignAccordion1` row is the dropdown header the user sees; the `PlanElement` rows sharing its `MedicalGrouping` are the services inside it. The rule the build follows: a value supplied for a specific `UIMedCatId_Sequence` (via `t1_flat_copays` / `cross_tier_copays` / the PCP-Specialist keys) wins for that service; every other service in the grouping inherits the parent row's tier value; a parent with no instruction takes the tier's default coinsurance. The mapping lives in `MEDTYPE_TO_GROUPINGS` / `MEDTYPE_TO_SEQ` in the build script — note `PP` and `SP` both map to grouping `PH` and are pinned to sequences 509/511, lab (`LP`) rolls up under `O`, and `GBL` rollup rows are deliberately never touched. A shell containing a grouping outside this map hard-fails check 10 rather than shipping template values.

**Mirroring includes all three deductible checkboxes.** `Deductd1T{n}`, `Deductd2T{n}`, and `Deductd3T{n}` on the accordion row must carry exactly the same values as `Deductible1Applies`, `Deductible2Applies`, and `Deductible3Applies` on the matching PlanElements — never a single blanket value, and never the shell's leftover defaults. The blank Claros export ships every accordion row with all three set to `True`; any row not explicitly rewritten keeps that default and will display a deductible the calc layer does not apply.

## Multi-tier plans (3+ network levels)

Whenever "The tier decision" lands on 3-tier (Pattern A provider steerage, or Pattern B sub-network copay split), read `references/multi_tier.md` in full before building. It holds the 4-to-3 mapping, the SPD coinsurance rule, rich-tier and cross-tier copays, Pattern B's `t2_*` fields, `tier2_active` for single-in-network plans inside a 3-tier file, `tier_split`, and the accordion pattern. Two rules apply even before you open it: never build a multi-tier plan on the 2-tier reference (the middle tier is silently dropped), and always deliver `tier_split` as a flagged placeholder the user sets in Claros.

## Fields to placeholder (not from SBC)

The build writes minimal non-zero placeholders for census, enrollment, stop-loss and contribution fields automatically (zeros cause divide-by-zero failures in Claros). The full table is in `references/field_mapping.md`, "Fields to placeholder". After upload, the user updates these in Claros with real census, enrollment, and underwriting data.

## Build script usage

The proven working transformation lives in `references/build_script.md` (there is no standalone `.py` file, by design; that file explains why and how to extract it). Once you have written it out to `build_claros_xml.py`, it takes:
- A reference XML file path
- A list of plan dictionaries (one per scenario)
- An output path

Each plan dictionary has the SBC values:
```python
{
    "scenario_name": "PPO 1000",  # plan name only - NO client name (see clarifying question 5)
    "plan_type": "PPO",  # PPO / HMO / EPO (copay-style) or HDHP
    "in_net_ded_ind": 1000, "in_net_ded_fam": 2000,
    "in_net_moop_ind": 4000, "in_net_moop_fam": 8000,
    "oon_ded_ind": 1000, "oon_ded_fam": 2000,
    "oon_moop_ind": 6000, "oon_moop_fam": 12000,
    "in_net_plan_pays": 0.80,
    "oon_plan_pays": 0.60,
    "embedded_deductible": True,  # REQUIRED for every plan type, from the SBC's
                                  # family-deductible language (question 7)
    "pcp_copay": 20, "specialist_copay": 50,  # 0 for HDHP; fills the whole PH grouping
    # EVERY dollar copay the SBC prints beyond office visits (question 9's
    # inventory) - ER (307) / urgent care (312) at minimum on most PPOs.
    # Dict form = copay + coinsurance ("$250 copay, 20% coinsurance"):
    "t1_flat_copays": {"307": {"copay": 250, "plan_pays": 0.80}, "312": 50},
    # REQUIRED for PPO/HMO/EPO: the SBC's facility rows, transcribed (None =
    # tier default coinsurance). The build places them; see "Facility rows".
    "facility_lines": {"ip_hospital_stay": None, "ip_mental_health": None,
                       "ip_childbirth": None, "op_surgery": None},
    # Coinsurance exceptions: off-default rates or deductible-waived lines:
    # "t1_service_coins": {"404": 0.50, "604": {"plan_pays": 0.80, "after_ded": False}},  # 404 DME, 604 routine imaging
    # Rx tiers: scalar = fixed copay, dict = coins-with-max
    "rx_generic_retail": 10, "rx_generic_retail_after_ded": False,
    "rx_brand_form_retail": 30, "rx_brand_form_retail_after_ded": True,
    "rx_brand_nonform_retail": 50, "rx_brand_nonform_retail_after_ded": True,
    "rx_specialty_retail": {"coins": 0.25, "max": 215},
    "rx_specialty_retail_after_ded": True,
    "rx_generic_mail": 30, "rx_generic_mail_after_ded": False,
    "rx_brand_form_mail": 90, "rx_brand_form_mail_after_ded": True,
    "rx_brand_nonform_mail": 150, "rx_brand_nonform_mail_after_ded": True,
    "rx_specialty_mail": {"coins": 0.25, "max": 215},
    "rx_specialty_mail_after_ded": True,
}
```

For HDHPs, set `pcp_copay=0` and `specialist_copay=0`, and set ALL `*_after_ded` flags to `True` regardless of what the SBC says (IRS rule).

**How to run it (do not read the whole build script, do not retype it, do not edit it).**

1. Extract the script with one command. `<skill-dir>` is the folder this SKILL.md is in:
   ```bash
   awk '/BEGIN EMBEDDED PYTHON/{f=1}/END EMBEDDED PYTHON/{f=0}f' <skill-dir>/references/build_script.md \
     | awk '/^```python/{c=1;next}/^```/{c=0}c' > build_claros_xml.py
   python3 -c "import ast; ast.parse(open('build_claros_xml.py').read())" && wc -l build_claros_xml.py   # about 2,250 lines
   ```
2. For the full plan-dict field reference and the commented 2-tier, multi-tier and HDHP examples, view ONLY the top of the extracted file, through the closing `]` of the example `PLAN_DICTS` (about the first 335 lines). Nothing below that needs reading to build a file.
3. Write `run_build.py` with the settings for this client, then run `python3 run_build.py`:
   ```python
   import build_claros_xml as b
   b.REF_FILE = b.DEFAULT_REF          # or b.DEFAULT_REF_3TIER, or a client export path (The tier decision)
   b.OUT_FILE = "/mnt/user-data/outputs/<Client>_Eff_<YYYY-M-D>.xml"
   b.DATA_YEAR = "2026"
   b.TREND_DATE = "01/01/2026"         # MM/DD/YYYY
   b.NORMALIZE_DATE = "01/01/2026"
   b.PLAN_DICTS = [ ... ]              # one dict per scenario, in Claros order
   b.main()                            # runs every check; writes OUT_FILE only on a pass
   ```
   These are the only values a session sets. To fix a wrong value, edit `run_build.py` and re-run.

## Validation before delivering — MANDATORY, NON-SKIPPABLE

Validation is a HARD GATE, not an optional final polish. The build script writes
to a temporary file, runs `validate()`, and only promotes it to the real output
path if every check passes. **If validation fails, the script deletes the temp
file and exits — no XML exists to deliver.** This is deliberate.

Three rules you MUST follow:

1. **Always run the build script's `main()` (which calls `validate()`).** Never
   hand-write or hand-edit an output XML that bypasses the script, and never run
   a stripped-down version that skips validation. If you produce XML by any path
   other than a clean `main()` run that prints `All checks passed`, you have not
   validated it.
2. **Never work around a failing check.** If `validate()` fires (e.g. "HDHP
   deductible >= MOOP"), that is the skill telling you the INPUT is wrong — the
   deductible and MOOP almost certainly got swapped during SBC extraction. Fix
   the plan dict by re-reading the SBC, then re-run. Do NOT edit the assertion,
   wrap it in try/except, force the values through, or otherwise make the error
   pass. A check that fires is doing its job.
3. **Do not deliver until the run prints `All checks passed`.** If it doesn't,
   there is no file to deliver anyway — investigate and re-run.

The checks that run (numbered one-to-one with `validate()` in the build script — when a failure message cites a check number, it is THIS list):

1. **Structural match (attribute-aware)**: every scenario must contain exactly the reference's element tree AND, element by element, its attribute keys. Dropping an attribute the reference carries is always a hard fail; the build may only ADD the attributes named in `SIG_ATTR_ADD_EXEMPT` (census placeholders, deductible/MOOP values, the `PlanElement` copay family, stop-loss fees — all discovered empirically). The copay family is export-verified as Copay1 (v5.1); the entry stays because the blank shell carries no Copay1 attributes for the build to overwrite.
2. **Namespaces + declaration**: `xmlns:xsi` and `xmlns:xsd` present on root, no XML declaration.
3. **CRLF line endings**: file must be CRLF-terminated throughout.
4. **Denominator guard**: no zero/empty in census/enrollment fields Claros divides by.
5. **HDHP deductible < MOOP**: every HDHP's individual and family deductible must be strictly below its MOOP; equal values almost always mean the MOOP leaked into the deductible (see known_issues.md #8).
6. **Rx deductible-application count**: the number of Rx PlanElements carrying `Deductible2Applies="Y"` must equal what the plan dicts imply — HDHPs all 16, copay-style plans exactly the tiers flagged `*_after_ded`. ABOVE expected means the HDHP rule leaked into a PPO scenario (known_issues.md #10); BELOW means a flagged tier never got written.
7. **HDHP coinsurance sanity (WARN + PROMPT, not auto-fail)**: flags an OON rate that differs from the HDHP's own in-net band. If the CONFIRM block prints, verify against the SBC before delivering (see known_issues.md #12 and clarifying question 8). The former second guard comparing against the PPO's OON rate was removed in v3.2 (false-positives on the normal one-carrier case).
8. **UI accordion mirror**: every `PlanDesignAccordion1` row's copay, coinsurance percentage, ParticipantPct (member share of a copay+coins combination, mirroring `Copay1Perc`), and all three deductible checkboxes must equal the matching design PlanElements (copay-typed exception elements under a coinsurance rollup are allowed; `t1_service_coins` override elements are verified against their DECLARED values rather than the rollup; Utilization shadows are excluded). Catches the loads-fine-displays-wrong drift of known_issues.md #5. **Inpatient is strict (v4.0):** there is no copay exemption for the IP group. When the IP rows carry a copay, the IP display row must show the same copay, member share and deductible checkboxes, plan pct 100, and the same `IPCopayPerDay` (Per Admit / Per Day) code; when they are coinsurance, it must show no copay and the same rate. **PP/SP rows (v5.1):** `PP` is the Physician Other rollup and mirrors `508`, `SP` mirrors `511` (known_issues.md #20). This is the check that v3 skipped and that passed Ben Choate's Run A and Run B files (known_issues.md #19).
9. **Stray $1 placeholders**: no PlanElement may survive with the shell's `Copay1Flag="Y"` / `Copay1Min="1"` template placeholder (see known_issues.md #13); matters most with client-supplied reference exports.
10. **Coverage gate**: every design PlanElement must belong to a grouping the tier loops write (`GBL` exempt), every design SubPlan must be one the loops visit, and every Rx element must be in `RX_SEQ_MAP` — a client shell with anything else hard-fails instead of shipping shell template values.
11. **In-network preventive covered in full (hard fail, new in v3.2)**: every preventive element (sequences 504/510) in SubPlans 1 and 2 must be `Copay1Flag="N"`, all three deductible flags `"N"`, `CoInsRate="1"` — the ACA requirement a tier loop silently violated in 10 of 10 v3.1 POC tests. A scenario opts out only via `preventive_in_net_covered=False` (rare non-ACA designs, flagged to the user).
12. **Facility read applied (hard fail, new in v4.0)**: every facility sequence (IP `101`-`105`, OP `308`/`309`/`311`) in the Tier 1 SubPlan of a copay-style scenario must carry exactly what the facility read resolved from `facility_lines`, or the plain tier default when the row was `None` or flagged. Before the build starts, the facility read also refuses a missing or malformed `facility_lines`, a directly set `t1_ip_copay`, facility sequences in other per-sequence maps, and an inpatient disagreement with no F1 answer.

After a passing run, the script prints each scenario's deductibles, MOOPs, plan-pays percentages, and embedded flag, followed by the **SERVICE COVERAGE table**: every sequence's actual treatment (copay / copay+coins / coinsurance / covered-full, with deductible handling) per tier. **Walk the SBC's benefit table line by line against that coverage table before presenting** — the numbered checks validate the BUILD, not the READING, and a copay the SBC prints but the table doesn't show was never extracted (the exact miss behind six of ten v3.1 POC failures). A mismatch means fixing the plan dict and re-running, never hand-editing the XML.

## Output presentation

Use `present_files` to deliver the XML. In your message, list:
- File name and effective date
- Each scenario name and key plan design values (ded, MOOP, embedded flag, plan-pays %)
- The placeholder fields the user needs to update in Claros after upload

Don't editorialize on values that are explicit in the SBC; do flag any judgment calls (OON deductible interpretation, embedded vs aggregate decision, OON MOOP when SBC is silent, OON preventive left at the tier default when the SBC's OON column says otherwise, a non-calendar `ManualForDataYear` set to the plan year's start year). On any 3-tier build, ALWAYS flag the tier utilization split (`tier_split` -> PlanDesignAccordion1GlobalNode) as a placeholder the user must set in Claros from actual claims routing, and on Pattern B flag anything the XML cannot hold (separate Rx MOOP when Rx is folded into SubPlan 1; HDHP ER copay).

On every copay-style build, ALWAYS include the **Treated as a read** block, copied from the build's `TREATED AS A READ: FACILITY ROWS` output, and put each of its `MANUAL IN CLAROS` lines into the unmapped list below.

Whenever the build prints `DISPLAY NOTE: OUTPATIENT (OP) ROW`, include that note in the presentation VERBATIM, with its "Applies to" list. It tells the reviewer that the OP dropdown row showing coinsurance over ER, urgent care and outpatient surgery copays is Claros's own convention, not an error (builder decision, October 2026).

Four more items this section always carries when they apply:
- **Inpatient copay basis**: whenever any inpatient row carries a copay, add "Confirm the Per Admit / Per Day setting on the inpatient rows in Claros." Per Admit (`"A"`) is export-verified (v5.1); Per Day (`"D"`) is not yet verified against a Claros export (known_issues.md #19), so the flag matters most on per-day copays.
- **Unmapped SBC lines**: every copay or off-default coinsurance the inventory (question 9) could not place on a confirmed sequence, quoted from the SBC, as "set manually in Claros after upload." Never deliver silently without these.
- **Display note for copay+coins and specialty-coins tiers**: Claros shows these with the MEMBER share in "Participant %/Srv" while the plan percentage column reads 100 — that is the verified-correct display, not an error (a POC tester reported it as a bug). Say so preemptively whenever the build used `Copay1Perc`/`ParticipantPct`, and list any `after_ded: True` copay+coins combination as a verify-in-Claros item (that exact combination is not export-verified).
- **Separate Rx deductible**: when the plan has its own prescription deductible or Rx MOOP, the known_issues.md #17 confirmation items.


---

## Build script location

Because of enterprise restrictions there is no loose `.py` file in this skill. The full build script (`build_claros_xml.py`, about 2,250 lines) lives in **`references/build_script.md`** as inert double-fenced text. Extract it with the command in "Build script usage" rather than reading the file into context, and never retype it from memory or reconstruct it partially.

---

## Review decisions (accepted, do not re-flag)

From the September 1, 2026 AI Quality Evaluator run:

- *SPD not added to frontmatter triggers.* Builder relies on explicit `/sbc-to-claros...` invocation; broader trigger phrasing is not wanted.
- *2026 constants left as-is* (`DATA_YEAR`, `TREND_DATE`, `NORMALIZE_DATE`; the $9,200 ACA MOOP cap formerly in clarifying question 7 was removed in v5.0). Builder updates these manually at each plan-year rollover.

## Changelog

Version history (v2 through the current release, with the reason behind each fix) lives in `references/changelog.md`. It is a maintainer record, not build guidance, so it does not need to be read to build a file.
