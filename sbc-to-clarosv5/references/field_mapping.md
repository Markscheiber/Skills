# SBC-to-XML Field Mapping Reference

Exact XPath locations in the Claros XML for each value found in a Summary of Benefits and Coverage (SBC) PDF.

## Top-level scenario fields

| SBC location | XML location | Notes |
|--------------|--------------|-------|
| Plan name | `Scenario/@ScenarioName` | Fixed pattern: `<Network> <PlanType> <IndivDeductible>` — NO client name (the client is identified by the file name and the Claros case), digits only, NO `$` sign, e.g. `ASC PPO 1000`, `ASC HSA HDHP 6800`, or just `PPO 1000` for single-network clients. The plan-type token must match the real design and never be swapped between plans (an HSA plan is `HSA HDHP`, not `HSA PPO`). Confirm names with the user once, then reuse verbatim across runs. |
| Coverage Period (e.g. 1/1/2026) | `Scenario/@TrendDate` | Format MM/DD/YYYY |
| Coverage Period | `Scenario/Plan/Normalization/@NormalizeDate` | Same date as TrendDate |
| Plan year | `Scenario/Plan/Normalization/@ManualForDataYear` | Just YYYY. **Non-calendar plan years use the plan year START year** (builder decision, Oct 2026): a 12/01/2025-11/30/2026 plan year is `2025`. State the choice in the output presentation so the user sees it. |

## Medical deductibles and MOOPs

The XML separates in-network, out-of-network, and Rx into SubPlans under `Scenario/Plan/`. Find each by `@SubPlanId`:

| SubPlanId | NetworkId | What it represents |
|-----------|-----------|-------------------|
| `1` | `1000` | In-network (Tier 1) |
| `3` | `3000` | Out-of-network (Tier 3) |
| `4` | `4000` | Rx (Ada-style references only — see below) |

### Two reference structures

Different Claros XMLs organize Rx differently:

- **Ada-style**: Has a separate SubPlan `id=4` containing the 16 Rx PlanElements, with its own Rx Deductible/MOOP. Used in older or migrated client files.
- **Blank-style** (Claros export default): No SubPlan `id=4`. The 16 Rx PlanElements are folded into SubPlan `id=1` (in-network). Rx subject to the medical deductible.

The build script auto-detects which structure is in use via `get_rx_planelements()`. Both structures are valid and both load correctly.

Within each SubPlan, the deductible/MOOP element is `SubPlan/Deductible/Deduct`.

| SBC field | XML attribute on `<Deduct>` |
|-----------|------------------------------|
| Individual deductible | `Deductible2Individual` |
| Family deductible | `Deductible2Family`, `Deductible2EESP`, `Deductible2EE1C`, `Deductible2EECC` (all four equal to family) |
| Individual MOOP | `MOOPIndividual` |
| Family MOOP | `MOOPFamily`, `MOOPEESP`, `MOOPEE1C`, `MOOPEECC` (all four equal to family) |
| Plan coinsurance % (= 100% - member %) | `CoInsPlanPaysPerc` (decimal: 0.80 for "20% coinsurance") |
| Family deductible behavior (read from the SBC wording, ANY plan type, never by plan type) | `EmbededDeductible`: `"N"` = embedded ("each family member must meet their own individual deductible"), `"Y"` = aggregate (family deductible met only as a whole). Verified against a Claros export, November 2026. |

### Coinsurance conversion

SBCs state member coinsurance ("20% coinsurance" means member pays 20%). The XML uses **plan-pays** percentage:

| SBC says | XML value |
|----------|-----------|
| 20% coinsurance | `CoInsPlanPaysPerc="0.80"` |
| 30% coinsurance | `CoInsPlanPaysPerc="0.70"` |
| 40% coinsurance | `CoInsPlanPaysPerc="0.60"` |
| 50% coinsurance | `CoInsPlanPaysPerc="0.50"` |

## Service-level copays (office visits and beyond)

For PPO-style plans with fixed copays, update the matching `<PlanElement>` rows in SubPlanId 1:

| SBC field | UIMedCatId_Sequence | MedicalGrouping |
|-----------|---------------------|------------------|
| Primary care visit | `509` | `PH` |
| Specialist visit | `511` | `PH` |
| Every OTHER physician-grouping row | `501`-`514` (the other 12) | `PH` |

**The office copay fills the ENTIRE `PH` grouping, not just 509/511.** Confirmed
against a verified hand-built client export (2021 GAP/MOU): all 14 `PH`
sequences carry the office copay; only preventive (504/510, see below) differs.
The build script does this automatically (default fill = the specialist copay;
override with `ph_other_copay` or per-sequence `t1_flat_copays`). Copaying only
509/511 and coinsuring the other 12 was the v3.1 behavior testers reported as
"the Specialist copays did not fill in the drop down / Physician other".

The build also mirrors the `<PlanDesignAccordion1>` rows under `Scenario/MedCategoryParentNode/PlanDesignAccordion1ParentNodes`: `MedType="SP"` is the Specialty Physician rollup (mirrors `511`) and `MedType="PP"` is the **Physician Other(PH) rollup, not PCP** (mirrors `508`). PCP `509` has no display row (verified v5.1; known_issues.md #20).

For copay-based services, set on the PlanElement:
- `Copay1Flag="Y"`
- `Copay1Min="<copay amount>"`
- `Deductible1Applies="N"`, `Deductible2Applies="N"`, `Deductible3Applies="N"`
- `CoInsRate="1"` (plan pays 100% after copay)

**Copay + coinsurance combined** (e.g. "$100 Copay/visit, 30% Cost Sharing" on
an ER line) — verified encoding from the 2021 GAP/MOU export, sequence 307:
- `Copay1Min="100"`, `Copay1Perc="0.3"` (the **member** share), `CoInsRate="1"`
- Do NOT write the plan-pays rate into `CoInsRate` for these: `Copay1Perc`
  carries the coinsurance component and `CoInsRate` stays `1`.
- Accordion mirror: `CopayMinTn=100`, `ParticipantPctTn=30`,
  `CoinsurancePlanPctTn=100`.
- The verified export left all three `DeductibleNApplies="N"` even though its
  SBC printed "after Deductible" on that line; applying `Deductible2Applies="Y"`
  alongside `Copay1Perc` is a composition of verified mechanics but not itself
  export-verified — flag it as a verify-in-Claros item when used.

For HDHP (no upfront copay):
- `Copay1Flag="N"`
- All three `DeductibleNApplies="Y"`
- `CoInsRate="0.80"` (or whatever the plan-pays % is)

## Preventive care (sequences 504 / 510)

`PH` sequences **504 and 510** are preventive care / immunizations, confirmed
from two independent sources: the blank Claros export ships exactly these two
with all three `DeductibleNApplies="N"` and `CoInsRate="1"` while every other
`PH` row ships `Y/Y/Y`, and the verified 2021 hand build leaves exactly these
two covered-full in the OON SubPlan. ACA plans cover them at **100% in-network
with no deductible on every plan type, HDHPs included** (IRS safe harbor). The
build writes them covered-full in SubPlans 1 and 2 before any grouping fill,
and validation check 11 hard-fails any in-network preventive row that is not
`Copay1Flag="N"` / all-`N` deductibles / `CoInsRate="1"` (escape hatch:
`preventive_in_net_covered=False` for the rare non-ACA design whose SBC
literally prints preventive cost sharing).

**OON preventive is a judgment call**: it is not ACA-mandated, SBCs often mix it
("No charge for listed immunizations, 50% after Deductible for preventive"),
and Claros cannot split 504 from 510 semantically without confirmed names. The
build leaves OON preventive at the OON tier default; note it in the output
presentation if the SBC's OON preventive line says otherwise.

## Medical service-ID crosswalk (all groupings) - FULLY VERIFIED (v5.1)

The XML identifies each medical service ONLY by an internal Claros ID
(`UIMedCatId_Sequence`) plus its `MedicalGrouping`; it carries no service names.
**Every medical sequence below is verified** by a pinned Claros export (v5.1,
November 2026, AZ Blue PPO 5000 80 scenario): a distinct copay was hand-keyed on
every named line of the Claros "Tier Copays" screen, the scenario was exported,
and each value was read back off exactly one PlanElement. The same names live
in the build script as `SERVICE_NAMES`, and the post-build SERVICE COVERAGE
table now prints them beside each sequence.

| Grouping | Seq | Claros UI line | Under UI rollup (accordion row) |
|----------|-----|----------------|----------------------------------|
| `IP`  | `102` | IP Medical | Inpatient(IP) -> accordion `IP` |
| `IP`  | `105` | IP Surgical | Inpatient(IP) |
| `IP`  | `101` | IP Maternity | Inpatient(IP) |
| `IP`  | `103` | IP MHSA (mental health / substance abuse) | Inpatient(IP) |
| `IP`  | `104` | IP NICU | Inpatient(IP) |
| `OP`  | `309` | OP Surgery - Hospital | Outpatient(OP) -> accordion `OP` |
| `OP`  | `311` | OP Surgery - ASC | Outpatient(OP) |
| `OP`  | `308` | OP Other | Outpatient(OP) |
| `OP`  | `307` | ER(OP) | own line, no accordion row |
| `OP`  | `312` | Urgent Care(OP) | own line, no accordion row |
| `RAD` | `601` | RAD Complex OP Hospital (CT/MRI/PET at a hospital outpatient dept) | Radiology Complex (RAD) -> accordion `RAD` |
| `RAD` | `602` | RAD Complex OP Imaging Center (CT/MRI/PET at a freestanding center) | Radiology Complex (RAD) |
| `RAD` | `604` | Radiology Routine (x-ray, ultrasound) | own line, no accordion row |
| `LP`  | `201` | Lab Pathology | own line, no accordion row |
| `PH`  | `509` | Primary Physician - PCP | own line, **no accordion row** |
| `PH`  | `511` | PH Special Visit (the specialist office visit) | Specialty Physician -> accordion `SP` |
| `PH`  | `503` | PH Consultation | Specialty Physician |
| `PH`  | `506` | PH Maternity (prenatal/postnatal office) | Specialty Physician |
| `PH`  | `514` | PH Therapy (PT/OT/ST visits) | Specialty Physician |
| `PH`  | `507` | PH Mental Health (outpatient office) | Specialty Physician |
| `PH`  | `512` | PH Substance Abuse (outpatient office) | Specialty Physician |
| `PH`  | `502` | PH Chiro | Specialty Physician |
| `PH`  | `501` | PH Anesthesia | Physician Other(PH) -> accordion `PP` |
| `PH`  | `504` | PH Immunizations | Physician Other(PH) |
| `PH`  | `510` | PH Preventive | Physician Other(PH) |
| `PH`  | `513` | PH Surgery (physician/surgeon fees) | Physician Other(PH) |
| `PH`  | `508` | PH Other | Physician Other(PH) |
| `O`   | `404` | O DME | Other -> accordion `O` |
| `O`   | `407` | O Rx - Non - PBM (medical-benefit drugs) | Other |
| `O`   | `402` | O Ambulance | Other |
| `O`   | `405` | O Home Health | Other |
| `O`   | `403` | O Dialysis Facility | Other |
| `O`   | `408` | O SNF (skilled nursing facility) | Other |
| `O`   | `406` | O Hospice | Other |
| `O`   | `401` | O All Other | Other |

**Accordion rollup rows are DISPLAY ONLY.** The values keyed on the UI rollup
rows (Inpatient, Outpatient, Specialty Physician, Physician Other, Other) landed
on `PlanDesignAccordion1` rows and on NO PlanElement; Claros calculates from the
PlanElements. Note the trap: **accordion `MedType="PP"` is the "Physician
Other(PH)" rollup, not PCP** (PCP 509 has no display row). The build mirrors PP
from `508` (PH Other) and SP from `511` (PH Special Visit); see known_issues.md #20.

The same export also verified: the copay field family is **Copay1**
(`Copay1Flag="Y"` + `Copay1Min`, `Copay2Flag="N"`), and an inpatient copay with
the Per Admit radio writes `IPCopayPerDay="A"` / `IPDayAdmit="A"` on the IP
elements and `"N"` on everything else, exactly what the build writes.

**Placement rule (v5.1 - fixed, never a session judgment):**

| Sequences | Placement |
|-----------|-----------|
| Facility rows: IP `101`-`105`, OP `308` `309` `311` | ONLY by the build's facility read from `facility_lines` ("Facility read" below). Never keyed into `t1_flat_copays` / `t1_service_coins` (the build refuses it). The names are now known, but the SBC's three inpatient rows still do not map one-to-one onto five subtypes, so the one-value rule stands. |
| Office copays `509` / `511` | `pcp_copay` / `specialist_copay` keys. |
| Every other PH row | Receives the PH grouping fill (the specialist copay) by default. Override BY NAME per the SBC: a row the SBC prices differently goes in `t1_flat_copays` (a different copay) or `t1_service_coins` (coinsurance), e.g. PH Therapy `514` when the SBC prints rehab at coinsurance, PH Anesthesia `501` / PH Surgery `513` when physician/surgeon fees are coinsurance. |
| ER `307`, UC `312`, RAD `601` `602` `604`, LP `201`, O `401`-`408` | Place directly from the SBC line: `t1_flat_copays` for a copay, `t1_service_coins` for an off-default or deductible-waived coinsurance (e.g. hospice no charge = `{"406": {"plan_pays": 1.0, "after_ded": False}}`), `cross_tier_copays` for a copay that also applies OON. |
| Rx `701`-`716` | `RX_SEQ_MAP` (verified twice). |

**What still stops (set manually in Claros, never guessed):** the sequence is
known for everything now, so the stop rule covers only SBC lines the XML cannot
EXPRESS on one row:
- **Place-of-service splits on one SBC line** ("office visit copay or 30%
  coinsurance" for x-ray, lab, DME, maternity office): Claros has one row per
  service, not per setting. Leave the row at its default and list the line.
- **Day-band benefits** (LTAC 50% after day 100, EAR/SNF 50% after day 60/90).
- **Lines with no Claros row** (telehealth copays, Prosano-style clinic
  sub-networks on a 2-tier build).
- **OON-only exceptions on a 2-tier build** (e.g. ambulance OON at the
  in-network rate after the in-network deductible): there is no OON per-service
  coinsurance map; only `cross_tier_copays` (copays) reaches OON.

## Facility read (v4.0)

The SBC rows that feed `facility_lines` on every PPO/HMO/EPO plan dict (2-tier and multi-tier alike; not used on HDHPs):

| SBC row (standard SBC template wording) | `facility_lines` key | Sequences |
|------------------------------------------|----------------------|-----------|
| If you have a hospital stay: Facility fee (e.g., hospital room) | `ip_hospital_stay` | IP group |
| If you need mental health, behavioral health, or substance abuse services: Inpatient services | `ip_mental_health` | IP group |
| If you are pregnant: Childbirth/delivery facility services | `ip_childbirth` | IP group |
| If you have outpatient surgery: Facility fee (e.g., ambulatory surgery center) | `op_surgery` | `309` + `311` |
| The same row when the source LABELS hospital and ASC separately | `op_surgery_hospital` / `op_surgery_asc` | `309` / `311` |
| A source line explicitly naming outpatient other (SPD-style designs, e.g. CRMC) | `op_other` (optional; without it `308` takes `309`'s value) | `308` |

Each value is transcribed, not interpreted: `None` when the row prints the tier's ordinary coinsurance after the deductible, a copay scalar or `{"copay": X, "plan_pays": P, "after_ded": B}` when it prints a copay (an INPATIENT copay must be the dict form and add `"basis": "per_admit" | "per_day" | "not_stated"`, from the SBC's wording), or `{"plan_pays": P, "after_ded": B}` when it prints an off-default or deductible-waived coinsurance with no copay. Physician/surgeon-fee cells in those rows are NOT facility rows; a copay or off-default rate there is an unmapped line under the stop rule.

**The rule (applied by `resolve_facility_read()` in the build, not by the session):**

1. **IP group.** The three IP rows are compared as values (copay, coinsurance component, deductible treatment). If all three agree, that value goes on all five IP sequences. If they differ, the build STOPS and the operator answers pinned question F1 (SKILL.md); the answer is recorded as `ip_discrepancy_choice` and the differing rows are listed as set-manually-in-Claros items. A per-subtype placement is never attempted, because subtype order is not guaranteed.
2. **OP surgery.** One row covering both settings goes on `309` and `311`. A source that labels hospital and ASC separately places each on its own sequence. A source that prints differing values WITHOUT saying which setting is which is recorded as `"op_surgery": "UNLABELED_SPLIT"`: neither sequence is placed and both are listed as set-manually-in-Claros items.
3. **OP Other.** An explicit `op_other` line is placed on `308`. Without one, `308` carries the SAME value as OP Surgery - Hospital (`309`): the one-row value, or the labeled hospital value (builder decision, October 2026). When `309` is not placed (unlabeled split), `308` is not placed either and is flagged with it.

Validation check 12 hard-fails any built facility row that differs from this read, and the build prints the read under "TREATED AS A READ: FACILITY ROWS" for the output presentation.

Notes:
- These IDs are Claros-internal and consistent across exports (the 2021 GAP/MOU
  export and the current blank export carry identical sequence sets).
- The Rx sequence map (`RX_SEQ_MAP` in the build script) is **verified twice**:
  client fill + the 2021 export's accordion rows (701-704 = Mail, 705-708 =
  Retail, each +8 twin = the New/Maintenance split).
- **Stop rule (v5.1)**: every sequence is now named, so a line is a
  set-manually-in-Claros item ONLY when the XML cannot express it on one row
  (place-of-service split on one SBC line, day-band benefits, lines with no
  Claros row, OON-only exceptions on a 2-tier build; see the crosswalk above).
  List each as "the SBC prints [line]; Claros cannot hold it on one service
  row - set it manually in Claros after upload." Never silently fold such a
  line into the grouping fill or tier coinsurance.
- **Re-pinning procedure** (if a future Claros release changes the screen):
  open any PPO scenario, hand-key a DISTINCT copay with the deductible waived
  on every named line of the in-network "Tier Copays" screen (use odd values
  like 101, 102, ... that cannot collide with real copays, including the
  rollup rows and every collapsed dropdown such as Specialty Physician), keep a
  key of line name -> value, export, and read which sequence carries which
  value. This is how the v5.1 crosswalk was built.
- Copay placement fields (per PlanElement): fixed copay via `Copay1Flag="Y"` +
  `Copay1Min`; inpatient copay adds `IPCopayPerDay` = the basis code from
  `IP_BASIS_CODES` (Per Admit `"A"`, Per Day `"D"`; the same code is written to
  the IP accordion row) + `IPDayAdmit="A"` (export-verified alongside Per
  Admit in v5.1; its effect under Per Day is still unverified); non-IP rows carry `IPCopayPerDay="N"`; deductible-waived copays set all `Deductible*Applies="N"` and
  `CoInsRate="1"` (plan pays 100% after copay).

## Rx tier copays

The Rx subplan (`SubPlanId="4"`) has 16 PlanElement rows. Each combination of {Generic / Brand Formulary / Brand Non-Formulary / Specialty} × {Retail / Mail} × {New / Maintenance} has a unique sequence ID:

| Description | Sequence ID |
|-------------|-------------|
| Retail Generic (new) | `715` |
| Retail Generic (maintenance) | `707` |
| Retail Brand Formulary (new) | `713` |
| Retail Brand Formulary (maintenance) | `705` |
| Retail Brand Non-Formulary (new) | `714` |
| Retail Brand Non-Formulary (maintenance) | `706` |
| Retail Specialty (new) | `716` |
| Retail Specialty (maintenance) | `708` |
| Mail Generic (new) | `711` |
| Mail Generic (maintenance) | `703` |
| Mail Brand Formulary (new) | `709` |
| Mail Brand Formulary (maintenance) | `701` |
| Mail Brand Non-Formulary (new) | `710` |
| Mail Brand Non-Formulary (maintenance) | `702` |
| Mail Specialty (new) | `712` |
| Mail Specialty (maintenance) | `704` |

Maintenance and new-Rx use the same copay in most plans — the SBC doesn't typically distinguish.

For each Rx PlanElement, set:
- `Copay1Flag="Y"`
- `Copay1Min="<retail or mail copay from SBC>"`
- `Deductible1Applies="N"`
- `Deductible2Applies="Y"` if SBC says "after the deductible has been met", else `"N"`
- `Deductible3Applies="N"`
- `CoInsRate="1"`

### Rx tier as coinsurance with a dollar maximum

When the SBC prices a tier as a percentage with a per-script dollar cap (e.g. "Specialty drugs: 25% up to $215 maximum after deductible"), use a different attribute set on the PlanElement:

| Attribute | Value | Notes |
|-----------|-------|-------|
| `Copay1Flag` | `Y` | |
| `Copay1Perc` | decimal coinsurance (e.g. `0.25`) | Member percentage |
| `Copay1Max` | dollar cap (e.g. `215`) | Per-script maximum |
| `Copay1Min` | DELETE the attribute | Do not set a fixed copay |
| `Deductible2Applies` | `Y` or `N` | Per SBC |
| `CoInsRate` | `1` | |

Matching `PlanDesignAccordion1` row gets:
- `CopayMinT4=""` (empty)
- `ParticipantPctT4="25.0"` (the member percentage as a whole number)
- `CopayMaxT4="215"` (the dollar cap)
- `Deductd2T4="True"` if after deductible

### Rx tier as not covered

When a tier is genuinely not covered (or the SBC is silent on mail specialty and the carrier doesn't route specialty through mail), set:
- PlanElement: `Copay1Perc="1.0"`, no `Copay1Min` or `Copay1Max`, all `DeductibleNApplies="N"`
- Accordion: `ParticipantPctT4="100.0"`, `CopayMaxT4=""`, `Deductd2T4="False"`

**HDHP rule (IRS-mandated)**: ALL Rx tiers must have `Deductible2Applies="Y"` regardless of what the SBC literally says, because HDHPs require all non-preventive Rx to be subject to deductible for HSA eligibility. In addition, ALL HDHP Rx tiers must have `Copay1Min="0"` (and accordion `CopayMinT4="0"`): the member pays full cost until the deductible is met, so there is NO separate Rx copay. Do NOT copy the SBC's post-deductible copay (e.g. "$15 generic after deductible") into the copay field — that double-counts the deductible. See known_issues.md #7.

Also update the matching `<PlanDesignAccordion1>` rows where `TierType="T4"` and the `RxType`/`RxSubType` attributes match. Set `CopayMinT4` and `Deductd2T4` accordingly.

## UI display mirrors (PlanDesignAccordion)

The `MedCategoryParentNode` section under each Scenario contains UI-facing copies of the plan design that must be kept in sync with the PlanElement values, otherwise Claros displays wrong information even though the math is correct.

### PlanDesignAccordion1 (per-service copays/coins)

Under `Scenario/MedCategoryParentNode/PlanDesignAccordion1ParentNodes`, rows are keyed by `TierType` and either `MedType` (medical) or `RxType`/`RxSubType` (Rx).

For each row, set:
- `CopayMinT<N>` where N is the tier number (1, 3, or 4)
- `CoinsurancePlanPctT<N>` (whole number, e.g., `80` for 80%)
- `Deductd1T<N>`, `Deductd2T<N>`, `Deductd3T<N>` ("True" or "False")

### PlanDesignAccordion2 (family ded/MOOP)

Under `Scenario/MedCategoryParentNode/PlanDesignAccordion2ParentNodes`, three rows exist:
- `TierType="T1"` for in-network
- `TierType="T3"` for out-of-network
- `TierType="T4"` for Rx (typically blank if no separate Rx ded/MOOP)

Set `FamDed` and `FamMoop` to the family deductible and MOOP from the SBC.

## Common SBC plan types and their characteristics

### PPO
- Copay-based for office visits ($20-50 PCP, $40-80 Specialist typical)
- Coinsurance 70/80/90% plan-pays in-network, 50/60/70% OON
- Embedded deductible: read from the SBC's family-deductible wording, never assumed by plan type (`EmbededDeductible="N"` = embedded, `"Y"` = aggregate)
- Generic Rx usually NOT subject to deductible
- Brand and Specialty Rx usually subject to deductible

### HDHP
- No copays for office visits — everything subject to deductible
- Plan-pays % only applies after deductible
- Embedded deductible: read from the SBC like any plan (`embedded_deductible` is required; embedded writes `EmbededDeductible="N"`, aggregate writes `"Y"`). Aggregate deductible HDHPs are compliant (the ACA embedding rule covers the MOOP, not the deductible)
- ALL Rx subject to deductible (IRS rule for HSA eligibility)
- Family deductible 2x individual is typical
- 2025 minimum HDHP deductible: $1,650 individual / $3,300 family
- 2025 maximum HDHP MOOP: $8,300 individual / $16,600 family
- For 2026 effective dates, verify current IRS limits

### EPO / HMO
- Similar to PPO for in-network
- May have no out-of-network coverage at all (set OON to "Not covered" or use a very high deductible)

## What the SBC doesn't tell you

These need separate sources or placeholders:
- Census (TotalEE, age/gender breakdown)
- Enrollment by tier (EE / EE+SP / EE+CH / Family)
- ZIP distribution
- Stop loss specifics (specific deductible, aggregate corridor)
- Premium / underwriting fees (PEPM admin, profit, tax, commission)
- Rx PBM-specific data (rebates, dispensing fees)
- Network discounts (these are pre-populated in the reference and typically kept)

Use minimal placeholders for these (see `SKILL.md` "Fields to placeholder" table).
