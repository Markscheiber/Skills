# Multi-Tier (3-Network-Tier) Plans

How to build a Claros XML for a plan that has MORE than two network levels.
There are TWO distinct patterns that both land on a 3-tier Claros build, and
they are configured differently:

- **Pattern A - provider steerage** (sections 1-9): a rich/domestic tier + a
  broad tier + OON, sometimes with an extra preferred tier. Common on
  hospital-sponsored plans (UMR/Wausau, e.g. CRMC). Tiers have DIFFERENT
  deductibles, MOOPs, and coinsurance.
- **Pattern B - sub-network copay split** (section 10): ONE network with a
  preferred provider sub-list (e.g. BCI "ChoiceDocs"). Both in-network levels
  share the SAME deductible, MOOP, and coinsurance; only the office-visit /
  urgent-care copays differ. Verified on the Zoroco Packaging build.

Both patterns REQUIRE a 3-tier Claros reference (SubPlans 1, 2, 3). The
bundled blank reference is 2-tier; never build either pattern on it.

Read this in addition to `field_mapping.md` (which holds the service-ID
crosswalk) and `known_issues.md` (which holds the copay-flag trap).

## 1. Recognize the situation

A single medical plan with FOUR real network levels on the SBC / SPD, typically:
1. A top **Platinum / domestic / employed-provider** tier (richest benefits)
2. A preferred network tier (carrier often calls it "Tier 1")
3. A broad network tier (carrier often calls it "Tier 2")
4. **Out-of-network (OON)**

Claros holds only **THREE** network slots per plan, so you must map 4 real
levels onto 3 Claros slots.

## 2. Claros's three network slots

| Claros slot | SubPlanId | NetworkId | Accordion TierType |
|-------------|-----------|-----------|--------------------|
| Tier 1      | `1`       | `1000`    | `T1`               |
| Tier 2      | `2`       | `2000`    | `T2`               |
| OON         | `3`       | `3000`    | `T3`               |
| Rx          | `4`       | `4000`    | `T4`               |

The bundled `assets/blank_reference_2tier.xml` is a **2-tier** export (SubPlans
1 and 3 only, no SubPlan 2). It will NOT produce a middle tier. For a multi-tier
plan use the bundled **`assets/blank_reference_3tier.xml`** (a blank 3-tier
Claros export, SubPlans 1 + 2 + 3; set `REF_FILE = DEFAULT_REF_3TIER` in the
build script), or pass a client's own 3-tier Claros export as `REF_FILE`. The
build script auto-detects SubPlan 2 and only writes Tier 2 when it exists.

## 3. Decide the 4 -> 3 mapping (ASK THE USER)

You cannot keep all four real tiers. Confirm with the user which three to model.
The common, recommended choice:

- **Claros Tier 1 = the richest tier** (Platinum / domestic)
- **Claros Tier 2 = the broad / leaner network tier** (the carrier's "Tier 2")
- **Claros OON = OON**
- The middle preferred tier (carrier's "Tier 1") is **dropped**

Always confirm before building. The user may want a different pairing (e.g. keep
the preferred tier and drop Platinum). Also confirm one-file-multiple-scenarios
vs separate files, and the effective/trend date, as in the normal workflow.

## 4. Coinsurance source: the SPD beats the SBC

The SBC frequently does NOT state the rich tier's default coinsurance. It only
shows exceptions (e.g. a single "Durable medical equipment 20%" line). Do NOT
infer the whole-tier default from one exception.

Get the true default from the SPD's **"Plan Participation Rate, Unless Otherwise
Stated Below"** row for each tier column. Real example (CRMC Plus): the SBC's DME
line implied 80%, but the SPD's plan-participation default was **100%** for the
Platinum tier (DME was just the "otherwise stated" exception). Using 80% would
have understated the plan everywhere. Always cross-check the SPD when available.

Set `in_net_plan_pays` / `t2_plan_pays` / `oon_plan_pays` from those SPD defaults.

## 5. Tier 1 (rich tier) copays

The rich tier usually uses **copays** for facility and office services, then the
plan pays 100% after the copay. The build script places these from three
plan-dict fields (`facility_lines` is required on every copay-style plan; the
other two are optional):

- `facility_lines` (v4.0, REQUIRED on every PPO/HMO/EPO, multi-tier or not)
  -> the inpatient and outpatient-surgery facility rows, transcribed from the
  SPD/SBC; the build resolves them onto IP `101`-`105`, OP `309`/`311` and
  OP `308` (an explicit `op_other` line, else `309`'s value) under the fixed rule in field_mapping.md
  "Facility read". A uniform inpatient copay is written internally as
  `t1_ip_copay`, the inpatient facility copay below; never set
  `t1_ip_copay` yourself (the build refuses it). Each inpatient copay line
  states its `basis` (`per_day` for CRMC-style per-day designs). Written with
  `IPCopayPerDay` from `IP_BASIS_CODES` (Per Day = `"D"`, Per Admit = `"A"`, the
  same code on the accordion), `IPDayAdmit="A"`, deductible waived, `CoInsRate="1"` on
  every IP element. NOTE: Claros's copay table has no per-admission cap field, so
  a "$X per day up to $Y per admission" cap CANNOT be modeled here. Flag the cap
  to the user as a Claros-side note.
- `t1_flat_copays` -> `{service_id: copay}` per-visit copays, deductible waived,
  `CoInsRate="1"`. Typical rich-tier set: ER and Urgent Care (OP Surgery and
  OP Other now go in `facility_lines`). `509` (PCP) and `511` (Specialist) default to `pcp_copay` /
  `specialist_copay`, so you only add the facility/ER lines here.
- Services with NO copay (OP Surgery Hospital, Radiology, Lab, Other) are set
  to coinsurance at `in_net_plan_pays` — EXCEPT the `PH` physician grouping,
  which fills with the office copay (v3.2: specialist copay by default,
  `ph_other_copay` to override; preventive 504/510 are carved out covered-full
  first), and except any sequence listed in `t1_service_coins`. On a rich tier
  where the SPD prices non-office physician services as coinsurance rather than
  the office copay, set `"ph_other_copay": 0` — a scalar zero means "no copay
  here," which disables the fill so the non-office `PH` rows take the tier
  coinsurance while 509/511 keep their pcp/specialist copays. Confirm with the
  user which treatment the SPD intends before choosing.

Service IDs come from the crosswalk in `field_mapping.md` (e.g. `307`=ER,
`311`=OP Surgery ASC, `312`=Urgent Care, IP=`101`-`105`).

### Accordion rollup pattern (do it this way)

The summary accordion (`PlanDesignAccordion1`) has ONE row per MedType
(IP/OP/RAD/SP/PP/O) and can hold only one copay per grouping. Claros itself
produces this convention, and the script mirrors it:

- **IP, PP (PCP), SP (Specialist)** rows show the copay (`CopayMinT1`,
  `CoinsurancePlanPctT1=100`, `Deductd*T1=False`; IP also `IPCopayPerDay="D"`).
- **OP, RAD, O** rows leave the copay blank and show coinsurance at the tier rate.

The individual OP copays (ER, OP Surgery ASC, Urgent Care) live ONLY on the
PlanElements. Claros's detailed copay screen reads the PlanElements and shows
each named service; the accordion is just a rollup. Do not try to force a single
OP copay into the accordion.

## 6. Tier 2 and OON

Most services in the broad tier and OON are **coinsurance** at the tier plan-pays
rate (e.g. Plus: 80% Tier 2 / 60% OON; Basic: 70% / 50%). The script sets these
automatically from `t2_plan_pays` and `oon_plan_pays`.

### Cross-tier copays (easy to miss)

A few services carry the SAME flat copay in EVERY tier, including Tier 2 and OON,
not just the rich tier. Check the ER, Urgent Care, and Ambulance rows across ALL
tier columns of the SBC/SPD. If a copay shows in every column, put it in
`cross_tier_copays` `{service_id: copay}`; the script places it (deductible
waived, plan pays 100% after copay) on that PlanElement in **Tier 1, Tier 2, and
OON**. Everything else stays coinsurance.

Real example (CRMC PPOs): ER `$500` in all four tiers; Urgent Care `$75` in
Tier 1/Tier 2/OON (Platinum N/A). So `cross_tier_copays = {"307": 500,
"312": 75}`. Ambulance `$100` in all tiers lives in the "Other" grouping, whose
sub-mapping is not yet confirmed (see `field_mapping.md`); identify the specific
"O" service ID to place it.

On HDHP/CDHP plans, ER and Urgent Care are usually coinsurance in every tier
(CRMC CDHP: ER 85% all tiers; Urgent Care 90/90/70/50), so there are no
cross-tier copays there. Leave everything coinsurance and set no copay fields.

## 7. The shell copay-flag trap

Claros shells often pre-flag certain rich-tier services (OP/RAD/LP) as
copay-type with a `$1` placeholder (`Copay1Flag="Y"`, `Copay1Min="1"`),
anticipating that the tier uses copays. Any logic that writes coinsurance ONLY to
elements not already flagged as copays will SKIP these and leave the `$1` in the
file. The build script now handles every element explicitly (real copay OR
coinsurance). After building, verify NO `Copay1Min="1"` remain in any tier. See
`known_issues.md`, issue 6.

## 8. Pharmacy

The SPD may defer the pharmacy grid to the PBM (e.g. CVS/Caremark) and not
contain the copay tiers at all: in that case use the SBC for Rx. Watch for
specialty-rate discrepancies between the SBC and the SPD narrative (CRMC: SBC said
specialty 20% / $250 max; SPD PrudentRx section said 30%). Flag these for the user
to confirm with the PBM rather than guessing.

## 9. Worked example: CRMC (effective 1/1/2026)

Three plans (Plus PPO, Basic PPO, CDHP), each with 4 real tiers (Platinum /
preferred Tier 1 / broad Tier 2 / OON). Mapping chosen with the client: Claros
T1 = Platinum, Claros T2 = broad Tier 2, OON = OON (preferred tier dropped).

Coinsurance (plan-pays), from the SPD plan-participation defaults:

| Plan  | T1 (Platinum) | T2 (broad) | OON |
|-------|---------------|------------|-----|
| Plus  | 100%          | 80%        | 60% |
| Basic | 90%           | 70%        | 50% |
| CDHP  | 90%           | 70%        | 50% |

Tier 1 (Platinum) copays:

| Service                         | Plus | Basic |
|---------------------------------|------|-------|
| Inpatient facility (per day, all 5 IP subtypes) | $150 | $350 |
| OP Surgery ASC (`311`)          | $120 | $250  |
| OP Surgery Hospital (`309`)     | coins| coins |
| OP Other (`308`)                | $15  | $20   |
| ER (`307`)                      | $500 | $500  |
| Urgent Care (`312`)             | $0   | $0    |
| PCP (`509`)                     | $15  | $20   |
| Specialist (`511`)              | $55  | $60   |
| Radiology / Lab / Other         | coins| coins |

CDHP Tier 1: all coinsurance, no copays.

Cross-tier copays (Tier 2 and OON, both PPOs): ER `$500`, Urgent Care `$75`.
(Ambulance `$100` all tiers is in the "Other" grouping - identify the O-sequence.)

Plan-dict fields used (see the commented multi-tier example in the embedded build
script): `facility_lines` (IP rows all `150`/`350` with `basis: "per_day"`, `op_surgery_hospital: None`,
`op_surgery_asc: 120`/`250`, `op_other: 15`/`20`), `t1_flat_copays` `{307,312,509,511}`,
`cross_tier_copays` `{307,312}`, plus the `t2_*` deductible / MOOP / plan-pays
fields on each PPO. The CDHP carries `t2_*` but no copay maps.

### Modeling limits to flag to the user (Claros can't hold these in the XML)

- Per-admission inpatient caps ($X/day up to $Y/admission): per-day copay only.
- Within-tier coinsurance exceptions (e.g. DME at a different rate than the tier
  default): the tier default is used; the exception flattens.
- Cross-tier deductible rules (e.g. "the Platinum deductible applies to all tiers
  for ER"): each Claros tier has its own deductible.
- Deductible / OOP cross-accumulation across tiers, and (CDHP) medical + Rx
  sharing one deductible / OOP: these are Claros scenario settings, not XML
  fields; confirm they are toggled in Claros after upload.

## 10. Pattern B: sub-network copay split (ChoiceDocs-style)

Recognize it on the SBC: the in-network copay cell reads like
"ChoiceDocs = $10 Copay/visit; All other In-Network = $30 Copay/visit" while
every other service shows a single in-network coinsurance. There is one
network, one in-network deductible/MOOP, one in-network coinsurance - only the
PCP / specialist (and sometimes urgent care) copays split by sub-network.

Offer the user the choice (collapse was the historical v2 behavior):
1. **Collapse** to a 2-tier build on the bundled reference, loading either the
   sub-network or standard copays into Tier 1 (ask which), or
2. **True 3-tier** on the bundled `blank_reference_3tier.xml` or a
   user-supplied 3-tier reference (recommended when the copay spread matters
   to the model).

### True 3-tier configuration

- Claros Tier 1 = the preferred sub-network (ChoiceDocs), Tier 2 = all other
  in-network, OON = OON.
- Set `t2_ded_ind/fam`, `t2_moop_ind/fam`, `t2_plan_pays` IDENTICAL to the
  Tier 1 values - one shared accumulator, so the numbers are the same.
- Put the Tier 2 office copays in `t2_pcp_copay` / `t2_specialist_copay`, and
  any other both-tier copays (urgent care) in `t2_flat_copays` (plus
  `t1_flat_copays` for the Tier 1 side).
- Copays that ALSO apply out-of-network (Zoroco: ER $100 + deductible + 20% in
  every tier) go in `cross_tier_copays`, dict form when the deductible and
  coinsurance still apply: `{"307": {"copay": 100, "after_ded": True,
  "plan_pays": 0.80}}`. Copays that stop at the network edge (urgent care) go
  in the t1/t2 flat maps ONLY, never in cross_tier, or they leak into OON.
- The build writes Tier 2 PP/SP accordion rows as copays (mirroring Tier 1's
  pattern) whenever `t2_pcp_copay`/`t2_specialist_copay` are present; steerage
  plans omit them and Tier 2 stays coinsurance-only.

### Mixed files: a single-in-network plan inside a 3-tier file

An HDHP (or any plan) with NO sub-network split still occupies a 3-tier shell
when it shares the file with Pattern B PPOs. Build it as an effective 2-tier
scenario:

- `"tier2_active": False` -> writes `<Load InTier2="N">` so Claros switches the
  middle tier off for that scenario.
- Still supply `t2_*` fields MIRRORING Tier 1 exactly (the structural check
  requires SubPlan 2 to be populated; mirrored values are harmless at zero
  weight).
- `"tier_split": (95, 0, 5)` (or similar) zeroes Tier 2's utilization weight.

### Tier utilization split (`tier_split`) is ALWAYS a placeholder

`PlanDesignAccordion1GlobalNode InTier1/InTier2/OONTier3` is the percentage of
claims Claros routes to each tier. NO SBC contains this. On Pattern B plans it
materially drives the model (it decides how much claims volume gets the
ChoiceDocs copays vs the standard copays). Write an explicit placeholder (e.g.
`(50, 45, 5)`), and ALWAYS list it in the output presentation as a value the
user must set in Claros from actual claims routing. Never present a Pattern B
file without this flag - leaving the shell default (often 95/0/5) silently
models the whole plan at Tier 1 rates.

### Pattern B modeling limits (flag to the user)

Same as Pattern A section "Modeling limits", plus: a separate Rx out-of-pocket
maximum (e.g. $2,000/$4,000) cannot be represented when the reference folds Rx
into SubPlan 1 (no Rx MOOP field), and an HDHP's ER copay is not modeled (the
HDHP branch is coinsurance-only per IRS/HSA design).
