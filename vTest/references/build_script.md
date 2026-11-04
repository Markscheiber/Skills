# Build script: build_claros_xml.py

This skill is packaged for environments that restrict standalone Python (`.py`)
files. Instead of shipping a loose script, the full build script is stored below
as **inert text**: it sits inside a double fence (an outer `~~~~~` block wrapping
an inner ```` ```python ```` block), so nothing parses or runs it automatically.
It is documentation until you deliberately extract it.

**To run it, Claude does the following:**

1. Read the block between the `BEGIN EMBEDDED PYTHON` and `END EMBEDDED PYTHON` markers below.
2. Take the inner ```` ```python ```` fenced code (drop the outer `~~~~~` fence and the inner ```` ``` ```` fence lines).
3. Write that code verbatim to `build_claros_xml.py` in your working directory.
4. Run it with `python3 build_claros_xml.py` (or import its functions), adapting `PLAN_DICTS`, `OUT_FILE`, and `REF_FILE` as SKILL.md describes.

**Setting `REF_FILE` is not optional.** Two references ship with this skill and the script resolves both automatically:

| Reference | Constant | Structure | Use for |
|-----------|----------|-----------|---------|
| `assets/blank_reference_2tier.xml` | `DEFAULT_REF` | SubPlans 1 + 3 | Ordinary single-in-network plans (plain PPO / HDHP / HMO / EPO) |
| `assets/blank_reference_3tier.xml` | `DEFAULT_REF_3TIER` | SubPlans 1 + 2 + 3 | ANY multi-tier plan, Pattern A steerage or Pattern B sub-network copay split |

A client's own Claros export may be substituted for either by pointing `REF_FILE` at its path. Confirm the choice with the user before running (SKILL.md, "The tier decision") and never leave it on the default out of habit: a multi-tier plan built on the 2-tier reference silently drops the middle tier.

Both paths resolve through `_asset()`, which searches every known install root (`/mnt/skills/plugins/`, `/mnt/skills/user/`, an `assets/` folder beside the script, and a bare `assets/`). If none matches, the build stops at `load_reference()` and prints the full search list.

Do not run the script from inside this markdown; always extract it to a real `.py` file first, then execute that file. Read SKILL.md's "Validation before delivering" section before running — validation is a hard gate, not optional.

<!-- BEGIN EMBEDDED PYTHON: build_claros_xml.py -->
~~~~~
```python
"""
Build a Claros-compatible XML plan file from SBC-derived plan dictionaries,
using a Claros XML as a structural reference.

Default reference: the bundled blank export, resolved automatically (the
installed skill's assets/blank_reference_2tier.xml, falling back to an assets/
folder beside this script; a 2-network-tier structure: in-network SubPlan 1 +
OON SubPlan 3). For MULTI-TIER (3-network-tier) plans set
REF_FILE = DEFAULT_REF_3TIER, the bundled blank 3-tier export
(assets/blank_reference_3tier.xml, SubPlans 1 + 2 + 3), or point REF_FILE at a
client's own 3-tier Claros export. See references/multi_tier.md.

Two build paths, auto-selected by the reference structure plus plan-dict fields:
  - Simple 2-tier: reference has SubPlans 1 and 3 only; plan dict has in_net_*
    and oon_* fields (plus pcp/specialist copays for PPOs). No Tier 2 is written.
  - Multi-tier (3-tier): reference has SubPlans 1, 2 and 3; plan dict ALSO has
    t2_* fields and, for rich-tier copays, facility_lines / t1_flat_copays /
    cross_tier_copays. See the commented example below and multi_tier.md.

The script handles: xmlns:xsi/xsd preservation, CRLF endings, the utf-16-declared
but UTF-8-content quirk, Rx-location auto-detection, scenario auto-cloning, and
minimal non-zero placeholders. Read references/known_issues.md and
references/field_mapping.md before modifying.
"""

import copy
import os
import re
import xml.etree.ElementTree as ET

# =============================================================================
# Configuration - adjust per client/upload
# =============================================================================
# The workflow extracts this script to the WORKING directory, so a path
# relative to the script (../assets) resolves to a directory that does not
# exist and the default build dies at load_reference() before any validation
# runs. A single hardcoded install root has the same problem: this skill has
# shipped from more than one location (/mnt/skills/user/ and
# /mnt/skills/plugins/), and whichever one is hardcoded is wrong in the other.
# Search every known root in order, then fall back to an assets/ folder next to
# the script for anyone who copied the whole package somewhere else. If nothing
# matches, raise with the full search list so the fix is obvious.
SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
_SKILL_ASSET_DIRS = [
    "/mnt/skills/plugins/sbc-to-clarosv5/assets",
    "/mnt/skills/user/sbc-to-clarosv5/assets",
    os.path.join(SCRIPT_DIR, "assets"),
    "assets",
]

def _asset(name):
    for d in _SKILL_ASSET_DIRS:
        p = os.path.join(d, name)
        if os.path.exists(p):
            return p
    raise FileNotFoundError(
        f"Bundled reference {name!r} not found. Searched:\n  "
        + "\n  ".join(_SKILL_ASSET_DIRS)
        + "\nFix: copy the skill's assets/ folder next to build_claros_xml.py, "
          "or set REF_FILE to an explicit path to a Claros XML export.")

# TWO bundled references - pick by network structure, never by habit:
#   blank_reference_2tier.xml - SubPlans 1 + 3 only. Default for ordinary
#       one-in-network-level plans (plain PPO/HDHP/HMO/EPO).
#   blank_reference_3tier.xml - SubPlans 1 + 2 + 3 (blank 3-tier Claros export,
#       added after the Zoroco ChoiceDocs build). Use for ANY multi-tier plan:
#       Pattern A steerage or Pattern B sub-network copay split. A client's own
#       3-tier Claros export may still be substituted via REF_FILE.
# Resolved tolerantly at import: a missing bundle should fail when the build
# actually tries to LOAD that reference, with the full search list in the
# message, not at import time on the one you were not going to use.
def _asset_or_missing(name):
    try:
        return _asset(name)
    except FileNotFoundError as e:
        return f"<<MISSING:{name}>> {e}"

DEFAULT_REF = _asset_or_missing("blank_reference_2tier.xml")
DEFAULT_REF_3TIER = _asset_or_missing("blank_reference_3tier.xml")

# REF_FILE is set by the TIER DECISION (SKILL.md, "The tier decision"). Do not
# leave it on the default out of habit - the user is asked to confirm 2-tier vs
# 3-tier on EVERY build, and a multi-tier plan built on the 2-tier reference
# silently drops the middle tier.
REF_FILE = DEFAULT_REF                      # 2-tier confirmed; set to DEFAULT_REF_3TIER for multi-tier
# Output naming rule (confirm the client name once, then reuse verbatim):
#   <Client>_Eff_<YYYY-M-D>.xml  - effective date matches the scenario names.
OUT_FILE = "/mnt/user-data/outputs/Client_Eff_YYYY-M-D.xml"

DATA_YEAR = "2026"
TREND_DATE = "01/01/2026"                   # MM/DD/YYYY
NORMALIZE_DATE = "01/01/2026"

# Mail-order specialty when the carrier routes specialty only through specialty
# pharmacies (e.g. Medical Mutual of Ohio): there is NO "not covered" flag in
# this build, because the correct Claros representation has not been verified
# against a real export. The build MIRRORS the SBC's single specialty value into
# both retail and mail specialty tiers. When the SBC says mail specialty is not
# covered, build with the mirror as-is and add "mail-order specialty shows
# retail specialty pricing - mark not covered in Claros" to the placeholder-
# fields summary presented to the user. (A prior version documented a
# MAIL_SPECIALTY_NOT_COVERED flag no code ever read; it was removed on purpose.)

# ---------------------------------------------------------------------------
# PLAN_DICTS - one dict per scenario. Replace with the client's values.
#
# Fields on EVERY plan:
#   scenario_name (plan name only, NO client name - client lives in the FILE
#   name), plan_type ("PPO"/"HMO"/"EPO"/"HDHP"), in_net_ded_ind/fam,
#   in_net_moop_ind/fam, in_net_plan_pays, oon_ded_ind/fam, oon_moop_ind/fam,
#   oon_plan_pays, pcp_copay, specialist_copay (0 for HDHP), the Rx tiers, and
#   embedded_deductible (REQUIRED for EVERY plan type, HDHPs included - True
#   when the SBC says each family member meets their own individual deductible,
#   False when the family deductible is aggregate-only).
#
# Service-level fields (2-TIER PLANS TOO - these are NOT multi-tier-only;
# omitting them on a plan whose SBC prints the copays ships those services as
# coinsurance, the #1 accuracy failure in POC testing):
#   facility_lines     -> REQUIRED for PPO/HMO/EPO (v4.0). The SBC's facility
#                         rows, transcribed one key per row; the BUILD (not
#                         the operator, not the session) decides which Claros
#                         sequences they land on - see resolve_facility_read().
#                         Inpatient group (IP 101-105), all three keys required:
#                           "ip_hospital_stay": "If you have a hospital stay -
#                                               Facility fee" row
#                           "ip_mental_health": "mental health / behavioral
#                                               health / substance abuse -
#                                               Inpatient services" row
#                           "ip_childbirth":    "If you are pregnant -
#                                               Childbirth/delivery facility" row
#                         Outpatient surgery facility, EXACTLY ONE form:
#                           "op_surgery": <line>   one row covering both
#                                                  settings -> 309 AND 311
#                           "op_surgery_hospital" + "op_surgery_asc": the
#                                                  source LABELS the settings
#                                                  -> 309 / 311 respectively
#                           "op_surgery": "UNLABELED_SPLIT"  the source prints
#                                                  differing values without
#                                                  saying which is which ->
#                                                  neither placed, both flagged
#                         Optional:
#                           "op_other": <line>     OP Other (308), when a source
#                                                  line explicitly names it.
#                                                  Without it, 308 carries the
#                                                  SAME value as 309.
#                           "ip_discrepancy_choice": the operator's answer to the
#                                                  pinned discrepancy question
#                                                  (SKILL.md) - ONLY when the
#                                                  three IP rows differ:
#                                                  "hospital_stay_all" |
#                                                  "tier_default" |
#                                                  {"custom": <line>}
#                         <line> is None (the row prints the tier's default
#                         coinsurance, deductible applies), a copay (scalar or
#                         copay_spec dict), or a coinsurance-only override
#                         {"plan_pays": 0.7, "after_ded": False}.
#                         An INPATIENT copay line must be a dict carrying
#                         "basis": "per_admit" | "per_day" | "not_stated", read
#                         from the SBC wording (v4.0, finding D2), e.g.
#                         {"copay": 150, "plan_pays": 0.9, "basis": "per_admit"}.
#                         Facility sequences (101-105, 308, 309, 311) may NOT
#                         also appear in t1_flat_copays / t1_service_coins, and
#                         IP sequences may not appear in ANY per-sequence map:
#                         one source of truth per sequence.
#   t1_flat_copays     -> {service_id: copay} per-visit copays beyond the
#                         office visits and the facility rows above. Walk the
#                         SBC's full benefit table and place EVERY dollar copay
#                         it prints: ER (307) and urgent care (312) at minimum
#                         on most PPOs. Values are scalars or copay_spec dicts
#                         (see copay_spec()).
#   t1_service_coins   -> {service_id: plan_pays} coinsurance overrides for
#                         services whose rate differs from the tier default
#                         (e.g. DME at 50% under a 70% plan), or
#                         {"plan_pays": 0.8, "after_ded": False} for
#                         "X% Cost Sharing, Deductible does not apply" lines.
#   ph_other_copay     -> overrides the PH grouping fill (the 12 physician
#                         rows beyond PCP/Specialist default to the specialist
#                         copay; see the Tier 1 loop). Rarely needed.
#   preventive_in_net_covered -> defaults True (ACA: sequences 504/510 covered
#                         100% in-network, no deductible, on EVERY plan type
#                         including HDHPs). Set False ONLY when the SBC
#                         literally prints preventive cost sharing; check 11
#                         enforces it otherwise.
#
# MULTI-TIER (3-network-tier) plans ADD (see references/multi_tier.md):
#   t2_ded_ind/fam, t2_moop_ind/fam, t2_plan_pays   -> the middle (broad) tier
#   (Inpatient and OP-surgery facility copays come from facility_lines on
#   EVERY copay-style plan, 2-tier or multi-tier. t1_ip_copay is now an
#   internal key written only by resolve_facility_read(); setting it directly
#   is refused.)
#                         ANY copay value (pcp/specialist/facility/flat/cross) is a
#                         scalar (copay, deductible waived, plan pays 100% after)
#                         OR {"copay": X, "plan_pays": 0.7, "after_ded": False}
#                         when the SBC combines coinsurance (and/or the
#                         deductible) with the copay, e.g. "$100 Copay/visit,
#                         30% Cost Sharing". The member share is written to
#                         Copay1Perc (export-verified encoding; see copay_spec()).
#   t1_flat_copays     -> {service_id: copay} rich-tier per-visit copays, e.g.
#                         {"307": ER, "312": Urgent Care}. 509/511 default to
#                         pcp/specialist. OP Other / OP Surgery go in
#                         facility_lines, not here.
#   cross_tier_copays  -> {service_id: copay} copays that apply in EVERY tier
#                         (Tier 2 and OON too), e.g. {"307": 500, "312": 75}.
#   Take the rich-tier coinsurance (in_net_plan_pays) from the SPD "Plan
#   Participation Rate, Unless Otherwise Stated" row, NOT from an SBC exception.
#   Service-ID crosswalk lives in references/field_mapping.md.
#
#   Sub-network copay split fields (ChoiceDocs pattern; all OPTIONAL):
#   t2_pcp_copay / t2_specialist_copay -> Tier 2 office copays when the two
#                         in-network levels differ only on copays.
#   t2_flat_copays     -> {service_id: copay} per-visit copays for Tier 2
#                         (checked before cross_tier_copays).
#   tier2_active       -> False switches Tier 2 OFF via <Load InTier2="N">
#                         (single-in-network plan inside a 3-tier file; still
#                         supply t2_* mirroring Tier 1).
#   tier_split         -> (t1, t2, oon) utilization percentages written to
#                         PlanDesignAccordion1GlobalNode. NOT in the SBC -
#                         placeholder the user must set in Claros; flag it.
# ---------------------------------------------------------------------------
PLAN_DICTS = [
    {   # ---- Simple 2-tier PPO (works with the bundled blank reference) ----
        "scenario_name": "PPO 1000",       # plan name only - NO client name
        "plan_type": "PPO",
        "in_net_ded_ind": 1000, "in_net_ded_fam": 2000,
        "in_net_moop_ind": 4000, "in_net_moop_fam": 8000,
        "in_net_plan_pays": 0.80,
        "oon_ded_ind": 2000, "oon_ded_fam": 4000,
        "oon_moop_ind": 8000, "oon_moop_fam": 16000,
        "oon_plan_pays": 0.60,
        "embedded_deductible": True,   # REQUIRED for PPO/HMO/EPO - from the SBC's
                                       # family-deductible language, never defaulted
        "pcp_copay": 20, "specialist_copay": 50,
        # Every dollar copay the SBC prints beyond office visits goes here -
        # ER (307) / urgent care (312) at minimum on most PPOs. ER combining a
        # copay with coinsurance uses the dict form:
        "t1_flat_copays": {"307": {"copay": 250, "plan_pays": 0.80}, "312": 50},
        # Services whose coinsurance differs from the tier default, or waives
        # the deductible ("30% Cost Sharing, Deductible does not apply"):
        # "t1_service_coins": {"403": 0.50, "601": {"plan_pays": 0.80, "after_ded": False}},
        # REQUIRED: the SBC's facility rows (None = the row prints the tier
        # default coinsurance). E.g. "$150 copay/admission, 10% coinsurance,
        # deductible does not apply" is
        # {"copay": 150, "plan_pays": 0.90, "basis": "per_admit"}.
        "facility_lines": {
            "ip_hospital_stay": None, "ip_mental_health": None, "ip_childbirth": None,
            "op_surgery": None,
        },
        "rx_generic_retail": 10,        "rx_generic_retail_after_ded": False,
        "rx_brand_form_retail": 35,     "rx_brand_form_retail_after_ded": True,
        "rx_brand_nonform_retail": 60,  "rx_brand_nonform_retail_after_ded": True,
        "rx_specialty_retail": {"coins": 0.25, "max": 250}, "rx_specialty_retail_after_ded": True,
        "rx_generic_mail": 20,          "rx_generic_mail_after_ded": False,
        "rx_brand_form_mail": 70,       "rx_brand_form_mail_after_ded": True,
        "rx_brand_nonform_mail": 120,   "rx_brand_nonform_mail_after_ded": True,
        "rx_specialty_mail": {"coins": 0.25, "max": 250}, "rx_specialty_mail_after_ded": True,
    },
    # ---- MULTI-TIER PPO example (uncomment; needs a 3-tier reference) --------
    # See references/multi_tier.md for the full CRMC worked example.
    # {
    #     "scenario_name": "Platinum PPO 1000",   # plan name only - NO client name
    #     "plan_type": "PPO",
    #     "embedded_deductible": True,            # REQUIRED for PPO/HMO/EPO
    #     "in_net_ded_ind": 500,  "in_net_ded_fam": 1000,       # Tier 1 = rich / Platinum
    #     "in_net_moop_ind": 3000, "in_net_moop_fam": 6000,
    #     "in_net_plan_pays": 1.00,                             # from SPD default, not an SBC line
    #     "pcp_copay": 15, "specialist_copay": 55,
    #     "facility_lines": {                                   # SPD facility rows
    #         "ip_hospital_stay": {"copay": 150, "basis": "per_day"},   # SPD: per day
    #         "ip_mental_health": {"copay": 150, "basis": "per_day"},
    #         "ip_childbirth":    {"copay": 150, "basis": "per_day"},
    #         "op_surgery_hospital": None, "op_surgery_asc": 120,  # labeled split
    #         "op_other": 15,
    #     },
    #     "t1_flat_copays": {"307": 500, "312": 0},
    #     "cross_tier_copays": {"307": 500, "312": 75},         # ER / Urgent Care in every tier
    #     "t2_ded_ind": 1000, "t2_ded_fam": 2000,              # Tier 2 = broad
    #     "t2_moop_ind": 4000, "t2_moop_fam": 8000,
    #     "t2_plan_pays": 0.80,
    #     "oon_ded_ind": 2500, "oon_ded_fam": 5000,
    #     "oon_moop_ind": 7500, "oon_moop_fam": 15000,
    #     "oon_plan_pays": 0.60,
    #     # ... plus the same rx_* tiers as above ...
    # },
    # ---- SUB-NETWORK COPAY SPLIT example (ChoiceDocs pattern; 3-tier ref) -----
    # The two in-network levels are the SAME network with identical ded/MOOP/
    # coinsurance; only office-visit and urgent-care copays differ. Verified on
    # the Zoroco Packaging build (BCI ChoiceDocs, eff 1/1/2026).
    # {
    #     "scenario_name": "ASC PPO 1000",
    #     "plan_type": "PPO",
    #     "tier2_active": True, "tier_split": (50, 45, 5),   # PLACEHOLDER split - user sets in Claros
    #     "in_net_ded_ind": 1000, "in_net_ded_fam": 2000,    # Tier 1 = rich sub-network (ChoiceDocs)
    #     "in_net_moop_ind": 4000, "in_net_moop_fam": 8000,
    #     "in_net_plan_pays": 0.80,
    #     "embedded_deductible": True,            # REQUIRED for PPO/HMO/EPO
    #     "pcp_copay": 10, "specialist_copay": 30,
    #     "t1_flat_copays": {"307": {"copay": 100, "after_ded": True, "plan_pays": 0.80}, "312": 50},
    #     "facility_lines": {"ip_hospital_stay": None, "ip_mental_health": None,
    #                        "ip_childbirth": None, "op_surgery": None},   # REQUIRED
    #     "t2_ded_ind": 1000, "t2_ded_fam": 2000,            # Tier 2 = all other in-network
    #     "t2_moop_ind": 4000, "t2_moop_fam": 8000,          #   (same ded/MOOP/coins as Tier 1)
    #     "t2_plan_pays": 0.80,
    #     "t2_pcp_copay": 30, "t2_specialist_copay": 50,     # only the copays differ
    #     "t2_flat_copays": {"307": {"copay": 100, "after_ded": True, "plan_pays": 0.80}, "312": 50},
    #     "cross_tier_copays": {"307": {"copay": 100, "after_ded": True, "plan_pays": 0.80}},  # ER OON too
    #     "oon_ded_ind": 1000, "oon_ded_fam": 2000,
    #     "oon_moop_ind": 6000, "oon_moop_fam": 12000,
    #     "oon_plan_pays": 0.60,
    #     # ... plus the same rx_* tiers as above ...
    # },
    {   # ---- HDHP ----
        "scenario_name": "HSA HDHP 3400",  # plan name only - NO client name
        "plan_type": "HDHP",
        "embedded_deductible": True,   # REQUIRED - from the SBC's family-deductible
                                       # language; False for an aggregate HDHP
        "in_net_ded_ind": 3400, "in_net_ded_fam": 6800,
        "in_net_moop_ind": 5000, "in_net_moop_fam": 10000,
        "in_net_plan_pays": 0.80,
        "oon_ded_ind": 6800, "oon_ded_fam": 13600,
        "oon_moop_ind": 7500, "oon_moop_fam": 15000,
        "oon_plan_pays": 0.60,
        "pcp_copay": 0, "specialist_copay": 0,
        "rx_generic_retail": 10,        "rx_generic_retail_after_ded": True,
        "rx_brand_form_retail": 35,     "rx_brand_form_retail_after_ded": True,
        "rx_brand_nonform_retail": 60,  "rx_brand_nonform_retail_after_ded": True,
        "rx_specialty_retail": {"coins": 0.25, "max": 250}, "rx_specialty_retail_after_ded": True,
        "rx_generic_mail": 20,          "rx_generic_mail_after_ded": True,
        "rx_brand_form_mail": 70,       "rx_brand_form_mail_after_ded": True,
        "rx_brand_nonform_mail": 120,   "rx_brand_nonform_mail_after_ded": True,
        "rx_specialty_mail": {"coins": 0.25, "max": 250}, "rx_specialty_mail_after_ded": True,
    },
]

RX_SEQ_MAP = {
    "rx_generic_retail":       ["715", "707"],
    "rx_brand_form_retail":    ["713", "705"],
    "rx_brand_nonform_retail": ["714", "706"],
    "rx_specialty_retail":     ["716", "708"],
    "rx_generic_mail":         ["711", "703"],
    "rx_brand_form_mail":      ["709", "701"],
    "rx_brand_nonform_mail":   ["710", "702"],
    "rx_specialty_mail":       ["712", "704"],
}
RX_ACCORDION_MAP = {
    "rx_generic_retail":       ("RxRetail", "Generic"),
    "rx_brand_form_retail":    ("RxRetail", "BrandFormulary"),
    "rx_brand_nonform_retail": ("RxRetail", "BrandNonFormulary"),
    "rx_specialty_retail":     ("RxRetail", "Speciality"),
    "rx_generic_mail":         ("RxMail", "Generic"),
    "rx_brand_form_mail":      ("RxMail", "BrandFormulary"),
    "rx_brand_nonform_mail":   ("RxMail", "BrandNonFormulary"),
    "rx_specialty_mail":       ("RxMail", "Speciality"),
}

MED_ALL = ("IP", "OP", "RAD", "SP", "PP", "O")
MED_GROUPINGS = ("IP", "OP", "RAD", "LP", "PH", "O")

# Plan types. Copay-style designs (PPO/HMO/EPO) carry office-visit copays and
# may carry flat/per-day facility copays; HDHP is coinsurance-only with the
# IRS Rx-deductible rule and the embedded-deductible default. Anything else is
# rejected before the build starts - an unknown plan_type used to fall into the
# HDHP branch silently, dropping the plan's office-visit copays with no error.
COPAY_PLAN_TYPES = ("PPO", "HMO", "EPO")
VALID_PLAN_TYPES = COPAY_PLAN_TYPES + ("HDHP",)

# Accordion parent row -> PlanElement child groupings (the dropdown hierarchy).
# A value supplied for a specific UIMedCatId_Sequence wins; otherwise every
# PlanElement in the grouping inherits the parent row's tier value. Notes:
# - PP and SP both map to grouping PH (physician); they are pinned to their
#   exact sequences (509 PCP / 511 Specialist) so the two rows can differ.
# - LP (lab) has no accordion row of its own and rolls up under O.
# - GBL elements are global/rollup rows the build deliberately never touches.
# - RX is handled by the Rx loop against RX_SEQ_MAP, not by this map.
# Check 8 uses this map to verify the display mirror; check 10 uses
# KNOWN_GROUPINGS to refuse shells containing groupings no branch handles.
MEDTYPE_TO_GROUPINGS = {"IP": ("IP",), "OP": ("OP",), "RAD": ("RAD",),
                        "O": ("O", "LP")}
MEDTYPE_TO_SEQ = {"PP": "509", "SP": "511"}
KNOWN_GROUPINGS = tuple(sorted(set(MED_GROUPINGS) | {"RX", "GBL"}))

# ACA preventive care / immunizations. Confirmed from two independent sources:
# (1) the blank Claros export ships EXACTLY these two PH sequences with all
#     three DeductibleNApplies="N" and CoInsRate="1" (no charge, deductible
#     waived) while every other PH row ships Y/Y/Y, and
# (2) a verified hand-built 2021 client export (GAP/MOU) leaves exactly these
#     two at N/N/N/CoInsRate=1 in the OON SubPlan while coinsuring the rest.
# ACA-compliant plans (including HDHPs - preventive is deductible-exempt by
# IRS safe harbor) cover these at 100% IN-NETWORK with no deductible. The
# in-network tier loops MUST write them covered-full BEFORE any grouping-level
# fill, or the tier default clobbers them - that exact overwrite shipped wrong
# preventive coverage in 10 of 10 real-client POC tests on v3.1. OON preventive
# is NOT ACA-mandated: Tier 3 keeps the tier default unless the SBC says
# otherwise (see field_mapping.md). Escape hatch: a plan dict may set
# "preventive_in_net_covered": False for the rare non-ACA design whose SBC
# literally prints cost sharing on preventive care - validation check 11
# enforces covered-full on every in-network preventive element otherwise.
PREVENTIVE_SEQS = ("504", "510")

# FACILITY READ (v4.0 - consistency finding D1). The inpatient subtypes and
# the OP surgery / OP Other rows are "Client fill" confidence in the crosswalk
# (field_mapping.md): placed only by resolve_facility_read(), from the SBC's
# facility rows in the plan dict's facility_lines, under a fixed rule.
# Three v3 sessions on identical SBCs placed, half-placed, and skipped these
# seven sequences - 162 of 222 output differences. The rule now lives here.
IP_SEQS = ("101", "102", "103", "104", "105")
OP_OTHER_SEQ = "308"
OP_SURGERY_HOSPITAL_SEQ = "309"
OP_SURGERY_ASC_SEQ = "311"
FACILITY_SEQS = IP_SEQS + (OP_OTHER_SEQ, OP_SURGERY_HOSPITAL_SEQ, OP_SURGERY_ASC_SEQ)
IP_LINE_KEYS = ("ip_hospital_stay", "ip_mental_health", "ip_childbirth")
OP_LINE_KEYS = ("op_surgery", "op_surgery_hospital", "op_surgery_asc", "op_other")
FACILITY_LINE_KEYS = IP_LINE_KEYS + OP_LINE_KEYS + ("ip_discrepancy_choice",)
UNLABELED_SPLIT = "UNLABELED_SPLIT"
IP_DISCREPANCY_CHOICES = ("hospital_stay_all", "tier_default")

# INPATIENT COPAY BASIS (v4.0 - consistency finding D2). The ONE place the
# Claros code for the Per Admit / Per Day radio is defined; both the
# PlanElement writer and the accordion writer read it from here, so the
# display and the calculation layers can no longer disagree (v3 hard-coded
# "D" twice; a session patched one copy and shipped D-vs-A). Evidence:
# the blank Claros shells ship every IP row as "A" and the UI defaults to
# Per Admit; the CRMC client fill of a per-day design exported "D".
# Confirm with the BASIS TEST import (SKILL.md, known_issues.md #19) and
# update this note when it lands; flip here only, never at a call site.
IP_BASIS_CODES = {"per_admit": "A", "per_day": "D"}
# What a facility_lines IP copay records: the SBC's own wording, transcribed.
# "not_stated" builds Per Admit (Claros default) and is flagged for review.
IP_BASIS_VALUES = ("per_admit", "per_day", "not_stated")
IP_BASIS_DEFAULT = "per_admit"

# ---------------------------------------------------------------------------
# Reference loading (declaration quirk)
# ---------------------------------------------------------------------------
def coins(val):
    """Format a coinsurance/plan-pays value the way the Claros reference does:
    whole numbers as bare integers ('1', not '1.0'; '0', not '0.0') and
    fractions without trailing zeros ('0.8', not '0.80'). Different models were
    emitting '1' vs '1.0' and '0.8' vs '0.80' for the same plan, producing
    spurious diffs and inconsistent files. The untouched reference XML that
    Claros itself generates uses the integer form, so we match it. Always route
    CoInsRate / CoInsPlanPaysPerc / CoinsurancePlanPct through this. See
    known_issues.md #9."""
    f = float(val)
    if f == int(f):
        return str(int(f))
    return ("%g" % f)


def load_reference(ref_path):
    """Load and PREFLIGHT the reference. A user-supplied REF_FILE that is not a
    valid Claros plan export used to die deep in the build with a parse error
    or a baffling structural-check failure that looked like a build bug; this
    turns that into a one-line answer at the door."""
    if str(ref_path).startswith("<<MISSING:"):
        raise SystemExit(
            "*** BUNDLED REFERENCE NOT FOUND - BUILD NOT STARTED ***\n"
            + str(ref_path).split(">> ", 1)[-1])
    with open(ref_path, "r", encoding="utf-8", errors="replace") as f:
        content = f.read()
    content = re.sub(r'<\?xml[^?]*\?>\s*', '', content, count=1)
    try:
        root = ET.fromstring(content)
    except ET.ParseError as e:
        raise SystemExit(
            f"*** REF_FILE is not valid XML ***\n  {ref_path}\n  parse error: {e}\n"
            "A reference must be a Claros plan export (.xml). Use the bundled "
            "2-tier blank (the default) or a client's Claros export - for "
            "multi-tier plans, a 3-tier export (see references/multi_tier.md).")
    problems = []
    if root.tag != "Scenarios":
        problems.append(f"root element is <{root.tag}>, expected <Scenarios>")
    sc = root.find("Scenario")
    if sc is None:
        problems.append("no <Scenario> element")
    else:
        plan = sc.find("Plan")
        if plan is None or not plan.findall("SubPlan"):
            problems.append("no Scenario/Plan/SubPlan structure (calc layer missing)")
        if not list(sc.iter("PlanDesignAccordion1")):
            problems.append("no PlanDesignAccordion1 rows (UI layer missing)")
        if sc.find("MedCategoryParentNode") is None:
            problems.append("no MedCategoryParentNode")
    if problems:
        raise SystemExit(
            "*** REF_FILE does not look like a Claros plan export ***\n"
            f"  {ref_path}\n  - " + "\n  - ".join(problems) +
            "\nUse a Claros plan export: the bundled 2-tier blank (the default) "
            "or a client's Claros export (3-tier for multi-tier plans). Do not "
            "point REF_FILE at an SBC, a Snapshot/Insights file, or any non-Claros XML.")
    return ET.ElementTree(root)


def find_subplan(plan, subplan_id):
    for sp in plan.findall("SubPlan"):
        if sp.get("SubPlanId") == subplan_id:
            return sp
    return None


def get_rx_planelements(plan):
    sp4 = find_subplan(plan, "4")
    if sp4 is not None and any(
        pe.get("UIMedCatId_Sequence", "").startswith("7")
        for pe in sp4.findall("PlanElement")
    ):
        return sp4
    return find_subplan(plan, "1")


# ---------------------------------------------------------------------------
# Deductible / MOOP
# ---------------------------------------------------------------------------
def set_deductible(subplan, ind, fam, moop_ind, moop_fam, embedded, plan_pays):
    d = subplan.find("Deductible/Deduct")
    if d is None:
        return
    d.set("EmbededDeductible", embedded)
    d.set("Deductible2Individual", str(ind))
    d.set("Deductible2Family", str(fam))
    d.set("Deductible2EESP", str(fam))
    d.set("Deductible2EE1C", str(fam))
    d.set("Deductible2EECC", str(fam))
    d.set("MOOPIndividual", str(moop_ind))
    d.set("MOOPFamily", str(moop_fam))
    d.set("MOOPEESP", str(moop_fam))
    d.set("MOOPEE1C", str(moop_fam))
    d.set("MOOPEECC", str(moop_fam))
    d.set("CoInsPlanPaysPerc", coins(plan_pays))


# ---------------------------------------------------------------------------
# Accordion (UI mirror) helpers
# ---------------------------------------------------------------------------
def set_accordion_fam(scenario, tier, fam_ded, fam_moop):
    for n in scenario.findall(".//PlanDesignAccordion2ParentNode"):
        if n.get("TierType") == tier:
            n.set("FamDed", str(fam_ded))
            n.set("FamMoop", str(fam_moop))


def set_accordion_med(scenario, tier, med_type, copay_min, ded_applies, coins_pct,
                      participant_pct=None):
    """ded_applies: a bool writes the same value to all three Deductd
    checkboxes (correct for the two homogeneous cases: coinsurance rows mirror
    PlanElements at Y/Y/Y, waived copays at N/N/N). Pass a 3-tuple of bools to
    mirror mixed PlanElement flags, e.g. (False, True, False) for a
    copay-after-deductible design - check 8 verifies the mirror either way.
    participant_pct mirrors a copay+coinsurance PlanElement's Copay1Perc
    (member share, 0-100 scale); leave None for plain copay or coinsurance
    rows, which clears the field so no shell leftover survives."""
    tier_n = tier[1]
    if isinstance(ded_applies, tuple):
        d1, d2, d3 = ded_applies
    else:
        d1 = d2 = d3 = ded_applies
    for n in scenario.findall(".//PlanDesignAccordion1"):
        if n.get("TierType") == tier and n.get("MedType") == med_type:
            n.set(f"CopayMinT{tier_n}", str(copay_min) if copay_min is not None else "")
            n.set(f"Deductd1T{tier_n}", "True" if d1 else "False")
            n.set(f"Deductd2T{tier_n}", "True" if d2 else "False")
            n.set(f"Deductd3T{tier_n}", "True" if d3 else "False")
            n.set(f"CoinsurancePlanPctT{tier_n}", str(coins_pct))
            if f"ParticipantPctT{tier_n}" in n.attrib:
                n.set(f"ParticipantPctT{tier_n}",
                      "" if participant_pct is None else str(participant_pct))


def set_accordion_rx(scenario, rx_type, rx_subtype, copay, ded1, ded2, ded3):
    """Mirror an Rx PlanElement's copay and ALL THREE deductible checkboxes.
    ded1/ded2/ded3 must equal the matching PlanElement's Deductible1/2/3Applies
    exactly. Setting only Deductd2 and leaving Deductd1/Deductd3 at the shell's
    defaults (True in the blank export) made the UI show deductibles the calc
    layer never applied - the exact drift known_issues.md #5 warns about."""
    for n in scenario.findall(".//PlanDesignAccordion1"):
        if (n.get("TierType") == "T4" and n.get("RxType") == rx_type
                and n.get("RxSubType") == rx_subtype):
            n.set("CopayMinT4", str(copay))
            n.set("Deductd1T4", "True" if ded1 else "False")
            n.set("Deductd2T4", "True" if ded2 else "False")
            n.set("Deductd3T4", "True" if ded3 else "False")
            if "ParticipantPctT4" in n.attrib:
                n.set("ParticipantPctT4", "")
            if "CopayMaxT4" in n.attrib:
                n.set("CopayMaxT4", "")


def set_accordion_rx_coins(scenario, rx_type, rx_subtype, coins_rate, max_copay,
                           ded1, ded2, ded3):
    """Coins-with-max Rx mirror. ded1/ded2/ded3 must equal the matching
    PlanElement's Deductible1/2/3Applies exactly (see set_accordion_rx)."""
    for n in scenario.findall(".//PlanDesignAccordion1"):
        if (n.get("TierType") == "T4" and n.get("RxType") == rx_type
                and n.get("RxSubType") == rx_subtype):
            n.set("CopayMinT4", "")
            n.set("ParticipantPctT4", str(coins_rate * 100))
            n.set("CopayMaxT4", "" if max_copay is None else str(max_copay))
            n.set("Deductd1T4", "True" if ded1 else "False")
            n.set("Deductd2T4", "True" if ded2 else "False")
            n.set("Deductd3T4", "True" if ded3 else "False")


# ---------------------------------------------------------------------------
# PlanElement (calc layer) helpers
# ---------------------------------------------------------------------------
def set_planelement_copay(pe, copay, after_ded):
    pe.set("Copay1Flag", "Y")
    pe.set("Copay1Min", str(copay))
    for k in ("Copay1Perc", "Copay1Max"):
        if k in pe.attrib:
            del pe.attrib[k]
    pe.set("Deductible1Applies", "N")
    pe.set("Deductible2Applies", "Y" if after_ded else "N")
    pe.set("Deductible3Applies", "N")
    pe.set("CoInsRate", "1")


def set_planelement_coins_with_max(pe, coins_rate, max_copay, after_ded):
    pe.set("Copay1Flag", "Y")
    pe.set("Copay1Perc", str(coins_rate))
    if max_copay is None:
        if "Copay1Max" in pe.attrib:
            del pe.attrib["Copay1Max"]
    else:
        pe.set("Copay1Max", str(max_copay))
    if "Copay1Min" in pe.attrib:
        del pe.attrib["Copay1Min"]
    pe.set("Deductible1Applies", "N")
    pe.set("Deductible2Applies", "Y" if after_ded else "N")
    pe.set("Deductible3Applies", "N")
    pe.set("CoInsRate", "1")


def set_planelement_coins(pe, plan_pays, after_ded=True):
    """Coinsurance-typed service. after_ded=True (the default and by far the
    common case) applies all three deductibles then the coinsurance band.
    after_ded=False models "X% Cost Sharing, Deductible does not apply" - a
    real and recurring SBC line (e.g. diagnostic x-ray/blood work on Cigna OAP
    plans) that v3.1 could not express and silently built as
    deductible-then-coinsurance. Composition of verified mechanics: the
    verified exports use DeductibleNApplies="N" + CoInsRate on the preventive
    rows; this is the same shape at a non-100% rate."""
    pe.set("Copay1Flag", "N")
    if "Copay1Min" in pe.attrib:
        pe.set("Copay1Min", "0")
    flag = "Y" if after_ded else "N"
    pe.set("Deductible1Applies", flag)
    pe.set("Deductible2Applies", flag)
    pe.set("Deductible3Applies", flag)
    pe.set("CoInsRate", coins(plan_pays))


def set_planelement_covered_full(pe):
    """ACA preventive in-network: no charge, deductible waived, plan pays 100%.
    This is EXACTLY how the blank Claros export ships sequences 504/510
    (Copay1Flag="N", all three DeductibleNApplies="N", CoInsRate="1") - the
    helper restores that state after a tier loop would otherwise overwrite it,
    rather than inventing a representation."""
    pe.set("Copay1Flag", "N")
    if "Copay1Min" in pe.attrib:
        pe.set("Copay1Min", "0")
    for k in ("Copay1Perc", "Copay1Max"):
        if k in pe.attrib:
            del pe.attrib[k]
    pe.set("Deductible1Applies", "N")
    pe.set("Deductible2Applies", "N")
    pe.set("Deductible3Applies", "N")
    pe.set("CoInsRate", "1")


def service_coins_spec(v):
    """Normalize a t1_service_coins value: scalar plan-pays (deductible
    applies, the common case) or {"plan_pays": 0.5, "after_ded": False}.
    Returns (plan_pays, after_ded)."""
    if isinstance(v, dict):
        return v["plan_pays"], bool(v.get("after_ded", True))
    return v, True


def copay_spec(v):
    """Normalize a copay value from the plan dict.
    Scalar (e.g. 20) -> copay only: deductible waived, plan pays 100% after
    the copay. This is the common SBC pattern and the historical behavior.
    Dict -> the combined design the schema supports but SBCs state less often:
        {"copay": 100, "plan_pays": 0.7, "after_ded": False}
    means: member pays the copay PLUS (1 - plan_pays) coinsurance (e.g. the
    SBC line "$100 Copay/visit, 30% Cost Sharing"). Any dict key may be
    omitted (after_ded defaults False, plan_pays defaults to 100%-after-copay).
    Returns (copay, after_ded, plan_pays_or_None).

    ENCODING (verified against a real hand-built Claros export, 2021 GAP/MOU,
    ER sequence 307 = Copay1Min="100" Copay1Perc="0.3" DeductibleNApplies=all
    "N" CoInsRate="1"): the coinsurance component of a copay+coins combination
    lives in Copay1Perc as the MEMBER share, with CoInsRate staying "1".
    A prior version wrote plan_pays into CoInsRate and forced Deductible2
    on, which feeds the actuarial engine a different (wrong) design. Note the
    verified export left the deductible OFF even though its SBC printed
    "after Deductible" on the ER line; after_ded=True (Deductible2Applies="Y"
    alongside Copay1Perc) is therefore a composition of individually verified
    mechanics, not itself export-verified - when used, list it in the output
    presentation as a verify-in-Claros item."""
    if isinstance(v, dict):
        return v["copay"], bool(v.get("after_ded", False)), v.get("plan_pays")
    return v, False, None


def set_planelement_ip_copay(pe, copay_val, basis_code):
    """Inpatient facility copay. Accepts a scalar (deductible waived, plan pays
    100% after - the Platinum-tier pattern) or a copay_spec dict for copay +
    coinsurance (+ optional deductible) combined. basis_code is a value of
    IP_BASIS_CODES (Per Admit / Per Day) and is REQUIRED: the same code must
    reach set_accordion_ip_copay, which validation check 8 enforces."""
    assert basis_code in IP_BASIS_CODES.values(), f"bad IP basis code {basis_code!r}"
    copay, after_ded, plan_pays = copay_spec(copay_val)
    pe.set("Copay1Flag", "Y")
    pe.set("Copay1Min", str(copay))
    pe.set("IPCopayPerDay", basis_code)
    pe.set("IPDayAdmit", "A")      # meaning unverified; shells and v3 both carry "A" - unchanged
    if "Copay1Max" in pe.attrib:
        del pe.attrib["Copay1Max"]
    if plan_pays is not None:
        pe.set("Copay1Perc", coins(1 - plan_pays))   # MEMBER share - see copay_spec
    elif "Copay1Perc" in pe.attrib:
        del pe.attrib["Copay1Perc"]
    pe.set("Deductible1Applies", "N")
    pe.set("Deductible2Applies", "Y" if after_ded else "N")
    pe.set("Deductible3Applies", "N")
    pe.set("CoInsRate", "1")


def set_planelement_flat_copay(pe, copay_val):
    """Per-visit copay (ER / OP surgery / urgent care / office). Accepts a
    scalar (deductible waived, plan pays 100% after) or a copay_spec dict for
    copay + coinsurance (+ optional deductible) combined."""
    copay, after_ded, plan_pays = copay_spec(copay_val)
    pe.set("Copay1Flag", "Y")
    pe.set("Copay1Min", str(copay))
    pe.set("IPCopayPerDay", "N")
    if "Copay1Max" in pe.attrib:
        del pe.attrib["Copay1Max"]
    if plan_pays is not None:
        pe.set("Copay1Perc", coins(1 - plan_pays))   # MEMBER share - see copay_spec
    elif "Copay1Perc" in pe.attrib:
        del pe.attrib["Copay1Perc"]
    pe.set("Deductible1Applies", "N")
    pe.set("Deductible2Applies", "Y" if after_ded else "N")
    pe.set("Deductible3Applies", "N")
    pe.set("CoInsRate", "1")


def set_accordion_ip_copay(scenario, tier, copay_val, basis_code):
    """IP accordion rollup mirroring set_planelement_ip_copay: scalar = per-day
    copay, deductible waived, plan pays 100% after; dict = copay + coinsurance
    (+ optional deductible) combined, with the member share in ParticipantPct
    and the plan percentage staying 100 (mirrors Copay1Perc / CoInsRate=1)."""
    assert basis_code in IP_BASIS_CODES.values(), f"bad IP basis code {basis_code!r}"
    copay, after_ded, plan_pays = copay_spec(copay_val)
    tier_n = tier[1]
    for n in scenario.findall(".//PlanDesignAccordion1"):
        if n.get("TierType") == tier and n.get("MedType") == "IP":
            n.set(f"CopayMinT{tier_n}", str(copay))
            n.set("IPCopayPerDay", basis_code)   # SAME code as the PlanElements
            n.set(f"Deductd1T{tier_n}", "False")
            n.set(f"Deductd2T{tier_n}", "True" if after_ded else "False")
            n.set(f"Deductd3T{tier_n}", "False")
            n.set(f"ParticipantPctT{tier_n}",
                  "" if plan_pays is None else coins((1 - plan_pays) * 100))
            n.set(f"CoinsurancePlanPctT{tier_n}", "100")


# ---------------------------------------------------------------------------
# Scenario cloning
# ---------------------------------------------------------------------------
def ensure_scenario_count(root, count):
    scenarios = root.findall("Scenario")
    if len(scenarios) >= count:
        return
    template = scenarios[-1]
    base_id = int(template.get("ScenarioId", "1000000"))
    for i in range(count - len(scenarios)):
        clone = copy.deepcopy(template)
        clone.set("ScenarioId", str(base_id + i + 1))
        root.append(clone)


# ---------------------------------------------------------------------------
# Minimal non-zero placeholders (divide-by-zero protection)
# ---------------------------------------------------------------------------
def apply_minimal_placeholders(scenario):
    pop = scenario.find("Population")
    if pop is not None:
        ct1 = pop.find("CensusType1")
        if ct1 is not None:
            ct1.set("TotalEE", "1")
            ct1.set("EEOnlyPct", "1")
            if "Totmem" in ct1.attrib:
                ct1.set("Totmem", "0")
        cts = pop.find("CensusTypes")
        if cts is not None:
            target_set = False
            for c in cts.findall("CensusType23"):
                if (not target_set and c.get("Gender") == "M"
                        and c.get("Age") == "40" and c.get("EnrollTier") == "1"):
                    c.set("EE", "1")
                    target_set = True
                else:
                    c.set("EE", "0")
        eec = pop.find("EECount")
        if eec is not None:
            eec.set("EECount1", "1")
            for k in ("EECount2", "EECount3", "EECount4", "EECount5"):
                if k in eec.attrib:
                    eec.set(k, "")
        zips = pop.find("EmployeesInZip")
        if zips is not None:
            for i, z in enumerate(zips.findall("Zip")):
                z.set("EnrolledEmp", "1" if i == 0 else "0")

    pte = scenario.find("PlanTierEnrollment")
    if pte is not None:
        pte.set("EE", "1")
        for k in ("EESP", "EECH", "EECHs", "EEFam"):
            if k in pte.attrib:
                pte.set(k, "0")
        pte.set("Total", "1")

    enr = scenario.find("Enrollment")
    if enr is not None:
        enr.set("EEs", "1")
        enr.set("SPs", "0")
        enr.set("CHs", "0")
        enr.set("TotalMembers", "1")
        enr.set("DependentPerEmployee", "1")
        enr.set("AdultEquivalent", "1")

    rs = scenario.find("ReimbursementAccounts")
    if rs is not None:
        for ra in rs.findall("ReimbursementAccount"):
            ra.set("EmployeeContribution", "1")
            ra.set("MaxAnnualContribution", "")
            ra.set("MemberPaidFirstDollar", "0.00")
            ra.set("PlanPaidPerc", "1")

    sl = scenario.find("StopLoss")
    if sl is not None:
        for k in ("SF_FixedExpense", "SF_SLPremium", "sf_ClaimPaymentPEPM",
                  "sf_EstPPAFeesPEPM", "sf_NetworkAccessFeePEPM",
                  "sf_MedicalManagementPEPM", "sf_DiseaseManagementPEPM",
                  "sf_PharmacyPEPM"):
            if sl.get(k, "") in ("", "0"):
                sl.set(k, "1")
        for k in ("fi_Administration", "fi_Profit", "fi_PremiumTaxTotal",
                  "fi_MandatedBenifits", "fi_Commission", "sl_Administration",
                  "sl_Profit", "sl_PremiumTaxTotal", "sl_Commission"):
            if sl.get(k, "") in ("", "0"):
                sl.set(k, "0.01")
        if sl.get("AgMargin", "") in ("", "0"):
            sl.set("AgMargin", "1")
        if "AgSpecCorridor" not in sl.attrib:
            sl.set("AgSpecCorridor", "0")

    norm = scenario.find(".//Normalization")
    if norm is not None:
        norm.set("TotalEE", "1")
        norm.set("NormalizeDate", NORMALIZE_DATE)
        norm.set("ManualForDataYear", DATA_YEAR)


# ---------------------------------------------------------------------------
# Plan design application (3-tier)
# ---------------------------------------------------------------------------
def apply_plan_design(scenario, pd):
    scenario.set("ScenarioName", pd["scenario_name"])
    scenario.set("TrendDate", TREND_DATE)

    # Embedded vs aggregate deductible is a PLAN FACT read off the SBC, not a
    # function of plan type. v3.1 hardcoded N for every non-HDHP, which built
    # two embedded PPOs as aggregate in POC test 6 (and could do the reverse
    # on an aggregate HDHP). Every plan type must state it, HDHPs included (v5.0:
    # HDHPs used to default to embedded, which would silently mis-build an
    # aggregate HDHP like Augustana's; the ACA embedding rule covers the MOOP,
    # not the deductible) - the KeyError makes the question unskippable rather
    # than silently defaulted. SBC read rule: "each family member must meet their own
    # individual Deductible" / "per person" = embedded (True); family
    # deductible met only in aggregate = aggregate (False). See SKILL.md
    # clarifying question 7.
    # CLAROS ENCODING (verified against a Claros export, Nov 2026, Augustana
    # University): EmbededDeductible="N" = EMBEDDED, "Y" = AGGREGATE. The
    # attribute name reads the opposite way; do not "fix" this back.
    if "embedded_deductible" not in pd:
        raise KeyError(
            f"{pd['scenario_name']}: embedded_deductible is required for "
            "every plan type, HDHPs included - read the SBC's family-deductible language "
            "(\"each family member must meet their own individual "
            "Deductible\" = True; aggregate-only = False) and set it "
            "explicitly. Do not guess.")
    embedded = "N" if pd["embedded_deductible"] else "Y"   # N = embedded (see above)

    # Derive coinsurance from structure: when a tier's deductible equals its
    # MOOP, there is no coinsurance band left — once the deductible is paid the
    # MOOP is met, so the plan pays 100% of subsequent claims. This is true
    # regardless of any "80% coinsurance" line the SBC prints, because that band
    # has zero width. Force plan_pays to 1.0 in that case so the value is derived
    # from the numbers rather than mis-read off the SBC. Applies per tier.
    # (See known_issues.md #11.)
    if pd.get("in_net_ded_ind") is not None and pd.get("in_net_moop_ind") is not None \
            and pd["in_net_ded_ind"] == pd["in_net_moop_ind"]:
        pd["in_net_plan_pays"] = 1.0
    if pd.get("t2_ded_ind") is not None and pd.get("t2_moop_ind") is not None \
            and pd["t2_ded_ind"] == pd["t2_moop_ind"]:
        pd["t2_plan_pays"] = 1.0
    if pd.get("oon_ded_ind") is not None and pd.get("oon_moop_ind") is not None \
            and pd["oon_ded_ind"] == pd["oon_moop_ind"]:
        pd["oon_plan_pays"] = 1.0

    # Per-scenario network-tier activation (OPTIONAL plan-dict fields).
    # <Load InTier1/InTier2/OONTier3="Y|N"> switches tiers on/off in Claros;
    # <PlanDesignAccordion1GlobalNode InTier1/InTier2/OONTier3="pct"> carries
    # the claims-utilization split across the tiers. Together these let a
    # single-in-network-level plan (e.g. an HDHP with no sub-network split)
    # live inside a 3-tier file as an effective 2-tier scenario: set
    # tier2_active=False, mirror Tier 1 into the t2_* fields so the structural
    # check passes, and give Tier 2 zero weight in tier_split. The tier_split
    # values are NOT in any SBC - they are a modeling input the user must set
    # in Claros; always list them as a placeholder in the output presentation.
    load = scenario.find("Load")
    if load is not None and "tier2_active" in pd:
        load.set("InTier2", "Y" if pd["tier2_active"] else "N")
    gnode = scenario.find(".//PlanDesignAccordion1GlobalNode")
    if gnode is not None and "tier_split" in pd:
        t1, t2, oon = pd["tier_split"]
        gnode.set("InTier1", str(t1))
        gnode.set("InTier2", str(t2))
        gnode.set("OONTier3", str(oon))

    plan = scenario.find("Plan")
    sp1 = find_subplan(plan, "1")
    sp2 = find_subplan(plan, "2")
    sp3 = find_subplan(plan, "3")
    rx_sp = get_rx_planelements(plan)

    # -- Deductibles / MOOPs --
    set_deductible(sp1, pd["in_net_ded_ind"], pd["in_net_ded_fam"],
                   pd["in_net_moop_ind"], pd["in_net_moop_fam"],
                   embedded, pd["in_net_plan_pays"])
    if sp2 is not None:
        if "t2_ded_ind" not in pd:
            raise KeyError(
                "Reference has a Tier 2 (SubPlan 2) but this plan dict has no t2_* "
                "fields. Multi-tier plans must supply t2_ded_ind/t2_ded_fam/"
                "t2_moop_ind/t2_moop_fam/t2_plan_pays (see references/multi_tier.md)."
            )
        set_deductible(sp2, pd["t2_ded_ind"], pd["t2_ded_fam"],
                       pd["t2_moop_ind"], pd["t2_moop_fam"],
                       embedded, pd["t2_plan_pays"])
    if sp3 is not None:
        set_deductible(sp3, pd["oon_ded_ind"], pd["oon_ded_fam"],
                       pd["oon_moop_ind"], pd["oon_moop_fam"],
                       embedded, pd["oon_plan_pays"])

    # Rx subplan: no separate Rx deductible. The "N" below is the
    # unchanged v4 value; with Deductible2Individual=0 the embedded/aggregate
    # flag has no effect on Rx (Claros "N" = embedded). Left as-is pending a
    # Claros export that shows SubPlan 4 with a nonzero Rx deductible.
    sp4 = find_subplan(plan, "4")
    if sp4 is not None:
        d = sp4.find("Deductible/Deduct")
        if d is not None:
            d.set("EmbededDeductible", "N")
            d.set("Deductible2Individual", "0")

    # -- Family ded/MOOP accordion mirrors --
    set_accordion_fam(scenario, "T1", pd["in_net_ded_fam"], pd["in_net_moop_fam"])
    if "t2_ded_fam" in pd:
        set_accordion_fam(scenario, "T2", pd["t2_ded_fam"], pd["t2_moop_fam"])
    set_accordion_fam(scenario, "T3", pd["oon_ded_fam"], pd["oon_moop_fam"])

    # ===================== TIER 1 (in-network / Platinum) =====================
    # PPO Platinum uses copays (facility per-day + per-visit) mapped by Claros
    # service ID; coinsurance services get the tier plan-pays rate. HDHP Platinum
    # is all coinsurance. Copay ID crosswalk confirmed from the client's Claros fill:
    #   IP 101-105 = inpatient subtypes (per-day facility copay)
    #   307=ER 308=OP Other 309=OP Surgery Hospital 311=OP Surgery ASC 312=Urgent Care
    #   509=PCP 511=Specialist ; 601/602/604=Radiology 201=Lab 401-408=Other
    prev_covered = pd.get("preventive_in_net_covered", True)
    svc_coins = pd.get("t1_service_coins", {})  # seq -> plan_pays scalar or {"plan_pays":, "after_ded":}
    if pd["plan_type"] in COPAY_PLAN_TYPES:
        # t1_ip_copay is written ONLY by resolve_facility_read() from the plan
        # dict's facility_lines (v4.0), on 2-tier and multi-tier plans alike.
        # t1_flat_copays is a FIRST-CLASS 2-tier field, not multi-tier-only:
        # ordinary PPO/HMO/EPO SBCs almost always carry per-visit copays beyond
        # the office visits - ER (307) and urgent care (312) at minimum. Leaving
        # it empty on a plan whose SBC prints those copays ships them as
        # coinsurance (the exact miss in 6 of 10 v3.1 POC failures); the copay
        # inventory question in SKILL.md exists to populate it.
        ip_copay = pd.get("t1_ip_copay")           # inpatient facility copay (or None)
        ip_basis = pd.get("_ip_basis_code")         # ONE code for elements AND accordion (D2)
        flat = dict(pd.get("t1_flat_copays", {}))  # seq -> per-visit copay (scalar or copay_spec dict)
        # Office copays. 509/511 take pcp_copay/specialist_copay unless t1_flat_copays
        # supplies them explicitly. A scalar 0 means NO copay (coinsurance-only
        # office visits, rare but real) and falls through to the tier default -
        # a $0 copay with the deductible waived would model the service as FREE.
        # (Explicit t1_flat_copays/cross_tier_copays zeros are honored as real
        # $0 copays: rich-tier steerage plans genuinely use them.)
        def _office(v):
            return None if (not isinstance(v, dict) and not v) else v
        if "509" not in flat and _office(pd["pcp_copay"]) is not None:
            flat["509"] = pd["pcp_copay"]
        if "511" not in flat and _office(pd["specialist_copay"]) is not None:
            flat["511"] = pd["specialist_copay"]
        # PH GROUPING FILL: the physician grouping holds 14 service rows (office
        # surgery, therapy visits, MH office, allergy, etc.), and the verified
        # hand-built export fills the office copay across ALL of them, not just
        # PCP (509) and Specialist (511). v3.1 coinsured the other 12, which is
        # what testers reported as "the Specialist copays did not fill in the
        # drop down / Physician other". Default fill = the specialist copay
        # (the other rows are specialist-like services); override wholesale
        # with "ph_other_copay" or per-sequence via t1_flat_copays. Preventive
        # (504/510) is carved out FIRST below.
        ph_fill = _office(pd.get("ph_other_copay", pd["specialist_copay"]))
        for pe in sp1.findall("PlanElement"):
            seq = pe.get("UIMedCatId_Sequence")
            grp = pe.get("MedicalGrouping")
            if seq in PREVENTIVE_SEQS and prev_covered:
                set_planelement_covered_full(pe)   # ACA: BEFORE any fill, see PREVENTIVE_SEQS
            elif grp == "IP" and ip_copay is not None:
                set_planelement_ip_copay(pe, ip_copay, ip_basis)
            elif seq in flat:
                set_planelement_flat_copay(pe, flat[seq])
            elif seq in svc_coins:
                pp_, ad_ = service_coins_spec(svc_coins[seq])
                set_planelement_coins(pe, pp_, ad_)
            elif grp == "PH" and ph_fill is not None:
                set_planelement_flat_copay(pe, ph_fill)
            elif grp in MED_GROUPINGS:
                set_planelement_coins(pe, pd["in_net_plan_pays"])
        # Accordion rollup: IP (if per-day copay) + PCP + Specialist show copays;
        # OP/RAD/O stay coinsurance (individual OP copays live on the PlanElements).
        if ip_copay is not None:
            set_accordion_ip_copay(scenario, "T1", ip_copay, ip_basis)
        elif pd.get("_ip_coins_override") is not None:
            # facility_lines resolved the whole IP group to one coinsurance-only
            # override; the rollup row shows that rate, not the tier default.
            pp_, ad_ = pd["_ip_coins_override"]
            set_accordion_med(scenario, "T1", "IP", None, ad_, int(round(pp_ * 100)))
        else:
            set_accordion_med(scenario, "T1", "IP", None, True, int(pd["in_net_plan_pays"] * 100))
        # PP/SP rows mirror the RESOLVED element value (flat wins over the
        # pcp_copay/specialist_copay keys, exactly as the element loop above),
        # so an explicit t1_flat_copays["509"/"511"] override can't desync the
        # display from the math. Copay rows keep CoinsurancePlanPct=100 with
        # any coinsurance component in ParticipantPct (member share), mirroring
        # the Copay1Perc / CoInsRate="1" element encoding.
        for med in ("PP", "SP"):
            v = flat.get(MEDTYPE_TO_SEQ[med])
            if v is None:   # no office copay -> the row is plain coinsurance
                set_accordion_med(scenario, "T1", med, None, True,
                                  int(pd["in_net_plan_pays"] * 100))
            else:
                c, ad, pp = copay_spec(v)
                set_accordion_med(scenario, "T1", med, c, (False, ad, False), "100",
                                  None if pp is None else coins((1 - pp) * 100))
        for med in ("OP", "RAD", "O"):
            set_accordion_med(scenario, "T1", med, None, True, int(pd["in_net_plan_pays"] * 100))
    else:  # HDHP: coinsurance, no copays (no pre-deductible copays on an HSA-qualified
           # plan) - EXCEPT preventive, which the IRS safe harbor exempts from the
           # deductible and ACA requires at 100% in-network. v3.1 coinsured it.
        set_accordion_med(scenario, "T1", "PP", None, True, int(pd["in_net_plan_pays"] * 100))
        set_accordion_med(scenario, "T1", "SP", None, True, int(pd["in_net_plan_pays"] * 100))
        for med in ("IP", "OP", "RAD", "O"):
            set_accordion_med(scenario, "T1", med, None, True, int(pd["in_net_plan_pays"] * 100))
        for pe in sp1.findall("PlanElement"):
            seq = pe.get("UIMedCatId_Sequence")
            if seq in PREVENTIVE_SEQS and prev_covered:
                set_planelement_covered_full(pe)
            elif seq in svc_coins:
                pp_, ad_ = service_coins_spec(svc_coins[seq])
                set_planelement_coins(pe, pp_, ad_)
            elif pe.get("MedicalGrouping") in MED_GROUPINGS:
                set_planelement_coins(pe, pd["in_net_plan_pays"])

    # ===================== TIER 2 (middle / broad) ============================
    # Mostly coinsurance, but on the PPOs ER/urgent care carry a flat copay in
    # every tier (deductible waived, plan pays 100% after copay).
    if sp2 is not None:
        cross = pd.get("cross_tier_copays", {}) if pd["plan_type"] in COPAY_PLAN_TYPES else {}
        # t2_flat_copays / t2_pcp_copay / t2_specialist_copay (all OPTIONAL):
        # copays that exist in BOTH in-network levels but NOT out-of-network.
        # Needed for SUB-NETWORK COPAY SPLIT plans (e.g. BCI ChoiceDocs), where
        # Tier 1 and Tier 2 are the SAME network with the same ded/MOOP/coins
        # and only the office-visit / urgent-care copays differ. On steerage
        # plans (CRMC pattern) Tier 2 has no office copays, so omit these and
        # the tier stays coinsurance-only exactly as before.
        t2flat = dict(pd.get("t2_flat_copays", {})) if pd["plan_type"] in COPAY_PLAN_TYPES else {}
        t2_office = (pd["plan_type"] in COPAY_PLAN_TYPES
                     and ("t2_pcp_copay" in pd or "t2_specialist_copay" in pd))
        t2_ph_fill = None
        if t2_office:
            t2flat.setdefault("509", pd.get("t2_pcp_copay", pd["pcp_copay"]))
            t2flat.setdefault("511", pd.get("t2_specialist_copay", pd["specialist_copay"]))
            # Tier 2 is an in-network level, so the PH grouping fills with its
            # office copay too (see the Tier 1 PH GROUPING FILL note). Override
            # with "t2_ph_other_copay"; falls back to the Tier 2 specialist copay.
            t2_ph_fill = pd.get("t2_ph_other_copay",
                                pd.get("t2_specialist_copay", pd["specialist_copay"]))
        for med in MED_ALL:
            set_accordion_med(scenario, "T2", med, None, True, int(pd["t2_plan_pays"] * 100))
        if t2_office:
            # Mirror the RESOLVED Tier 2 element values (t2flat wins), same
            # convention as Tier 1: plan pct 100, member share in ParticipantPct.
            for med in ("PP", "SP"):
                c, ad, pp = copay_spec(t2flat[MEDTYPE_TO_SEQ[med]])
                set_accordion_med(scenario, "T2", med, c, (False, ad, False), "100",
                                  None if pp is None else coins((1 - pp) * 100))
        for pe in sp2.findall("PlanElement"):
            seq = pe.get("UIMedCatId_Sequence")
            grp = pe.get("MedicalGrouping")
            if seq in PREVENTIVE_SEQS and prev_covered:
                set_planelement_covered_full(pe)   # Tier 2 is in-network: ACA applies
            elif seq in t2flat:
                set_planelement_flat_copay(pe, t2flat[seq])
            elif seq in cross:
                set_planelement_flat_copay(pe, cross[seq])
            elif grp == "PH" and t2_ph_fill is not None:
                set_planelement_flat_copay(pe, t2_ph_fill)
            elif grp in MED_GROUPINGS:
                set_planelement_coins(pe, pd["t2_plan_pays"])

    # ===================== TIER 3 (out-of-network) ============================
    cross = pd.get("cross_tier_copays", {}) if pd["plan_type"] in COPAY_PLAN_TYPES else {}
    for med in MED_ALL:
        set_accordion_med(scenario, "T3", med, None, True, int(pd["oon_plan_pays"] * 100))
    if sp3 is not None:
        for pe in sp3.findall("PlanElement"):
            seq = pe.get("UIMedCatId_Sequence")
            grp = pe.get("MedicalGrouping")
            if seq in cross:
                set_planelement_flat_copay(pe, cross[seq])
            elif grp in MED_GROUPINGS:
                set_planelement_coins(pe, pd["oon_plan_pays"])

    # ===================== Rx (SubPlan 4) =====================================
    for tier_key, seq_list in RX_SEQ_MAP.items():
        copay_value = pd[tier_key]
        after_ded = pd[tier_key + "_after_ded"]
        rx_type, rx_subtype = RX_ACCORDION_MAP[tier_key]

        if isinstance(copay_value, dict):
            coins_rate = copay_value["coins"]   # renamed: shadowing the coins() formatter broke calls to it inside this function
            max_copay = copay_value.get("max")
            applies_ded = True if pd["plan_type"] == "HDHP" else after_ded
            for pe in rx_sp.findall("PlanElement"):
                if pe.get("UIMedCatId_Sequence") in seq_list:
                    set_planelement_coins_with_max(pe, coins_rate, max_copay, applies_ded)
            # Mirror set_planelement_coins_with_max exactly:
            # Deductible1Applies=N, Deductible2Applies=applies_ded, Deductible3Applies=N.
            set_accordion_rx_coins(scenario, rx_type, rx_subtype, coins_rate, max_copay,
                                   False, applies_ded, False)
            continue

        copay = copay_value
        # HSA HDHP: no separate Rx copay (full cost until deductible met). Zero
        # the copay so BOTH the calc-layer PlanElement and the UI accordion agree.
        if pd["plan_type"] == "HDHP":
            copay = 0
        for pe in rx_sp.findall("PlanElement"):
            if pe.get("UIMedCatId_Sequence") in seq_list:
                if pd["plan_type"] == "HDHP":
                    # HSA HDHP: member pays full cost of Rx until the medical
                    # deductible is met, so there is NO separate Rx copay. Force
                    # Copay1Min to 0 regardless of any post-deductible copay the
                    # SBC lists or the plan dict carries. Writing the SBC's
                    # post-deductible copay here is a known error that produced
                    # inconsistent outputs (see known_issues.md #7). Rx is then
                    # governed by the medical deductible + coinsurance only.
                    pe.set("Copay1Flag", "Y")
                    pe.set("Copay1Min", "0")
                    for k in ("Copay1Perc", "Copay1Max"):
                        if k in pe.attrib:
                            del pe.attrib[k]
                    pe.set("Deductible1Applies", "Y")
                    pe.set("Deductible2Applies", "Y")
                    pe.set("Deductible3Applies", "Y")
                    pe.set("CoInsRate", "1")
                else:
                    set_planelement_copay(pe, copay, after_ded)
        if pd["plan_type"] == "HDHP":
            # Mirror the HDHP PlanElement block above: all three deductible
            # flags are Y on the calc layer, so all three are True in the UI.
            set_accordion_rx(scenario, rx_type, rx_subtype, copay, True, True, True)
        else:
            # Mirror set_planelement_copay: Ded1=N, Ded2=after_ded, Ded3=N.
            set_accordion_rx(scenario, rx_type, rx_subtype, copay,
                             False, after_ded, False)


# ---------------------------------------------------------------------------
# Validation
# ---------------------------------------------------------------------------
# Attributes the build legitimately ADDS to elements the blank shell ships
# without them. Discovered empirically by diffing full builds (both scalar and
# combined-design dicts) against the untouched reference - the build only ever
# ADDS attributes, never removes, so the structural check treats any REMOVAL
# as a hard fail and tolerates additions only from this table:
#   CensusType1 - apply_minimal_placeholders writes the 1-EE census
#   Deduct      - set_deductible writes ded/MOOP values the blank shell omits
#   PlanElement - the copay setters (Copay1 family; shrink once the
#                 Copay1-vs-Copay2 question is verified against a real
#                 copay-bearing Claros export - the blank shell carries a
#                 populated Copay2 group but no Copay1Min/Perc/Max anywhere)
#   StopLoss    - placeholder self-funded fees
SIG_ATTR_ADD_EXEMPT = {
    "CensusType1": {"TotalEE", "EEOnlyPct"},
    "Deduct": {"Deductible2Individual", "Deductible2Family", "Deductible2EESP",
               "Deductible2EE1C", "Deductible2EECC", "MOOPIndividual",
               "MOOPFamily", "MOOPEESP", "MOOPEE1C", "MOOPEECC"},
    "PlanElement": {"Copay1Min", "Copay1Perc", "Copay1Max"},
    "StopLoss": {"SF_FixedExpense", "SF_SLPremium"},
}


def validate(out_path, ref_root, plan_dicts):
    out = ET.parse(out_path).getroot()
    n = len(out.findall("Scenario"))
    print(f"Output scenarios: {n}")

    # 1. Structural match, ATTRIBUTE-AWARE: each output scenario must carry the
    # reference's exact element tree AND, element by element, its attribute
    # keys (values are checked by the design-specific checks below). A key
    # PRESENT in the reference but MISSING from the output is always a hard
    # fail (a setter deleted or forgot one). A key ADDED by the build is
    # allowed only from SIG_ATTR_ADD_EXEMPT above - anything else added is a
    # typo'd n.set() or an unreviewed behavior change.
    ref_sc = ref_root.findall("Scenario")[0]
    ref_elems = list(ref_sc.iter())
    struct_problems = []
    for i, sc in enumerate(out.findall("Scenario")):
        out_elems = list(sc.iter())
        if len(out_elems) != len(ref_elems):
            struct_problems.append(
                f"scenario {i}: element count {len(out_elems)} vs reference {len(ref_elems)}")
            continue
        for j, (re_, oe) in enumerate(zip(ref_elems, out_elems)):
            if re_.tag != oe.tag:
                struct_problems.append(
                    f"scenario {i} element #{j}: tag <{oe.tag}> vs reference <{re_.tag}>")
                break
            missing = set(re_.attrib) - set(oe.attrib)
            added = (set(oe.attrib) - set(re_.attrib)
                     - SIG_ATTR_ADD_EXEMPT.get(oe.tag, set()))
            if missing or added:
                struct_problems.append(
                    f"scenario {i} element #{j} <{oe.tag}"
                    + (f" seq={oe.get('UIMedCatId_Sequence')}" if oe.get('UIMedCatId_Sequence') else "")
                    + ">: "
                    + (f"missing attrs {sorted(missing)} " if missing else "")
                    + (f"unexpected attrs {sorted(added)}" if added else ""))
                if len(struct_problems) >= 10:
                    break
        if len(struct_problems) >= 10:
            break
    assert not struct_problems, (
        "Structural / attribute-key mismatch vs reference:\n  " +
        "\n  ".join(struct_problems) +
        "\nThe build must never DROP an attribute the reference carries, and "
        "may only ADD the attributes named in SIG_ATTR_ADD_EXEMPT. A missing "
        "attribute means a setter deleted or forgot one; an unexpected "
        "addition is usually a typo'd n.set().")
    print(f"  Element structure + attribute keys match reference across all "
          f"{n} scenarios ({len(ref_elems)} elements each)")

    # 2. Namespaces + declaration
    with open(out_path, "rb") as f:
        head = f.read(300)
    assert b"xmlns:xsi" in head, "Missing xmlns:xsi"
    assert b"xmlns:xsd" in head, "Missing xmlns:xsd"
    assert not head.lstrip().startswith(b"<?xml"), "Output should have no XML declaration"
    print("  xmlns:xsi / xmlns:xsd present, no XML declaration")

    # 3. CRLF
    with open(out_path, "rb") as f:
        data = f.read()
    assert b"\r\n" in data, "Not CRLF-terminated"
    print("  CRLF line endings confirmed")

    # 4. Divide-by-zero guard: no zero/empty in denominator-risk fields
    risky = []
    for sc in out.findall("Scenario"):
        checks = {
            "CensusType1.TotalEE": sc.findtext(".//CensusType1[@TotalEE]") or
                                   (sc.find(".//CensusType1").get("TotalEE") if sc.find(".//CensusType1") is not None else None),
        }
        c1 = sc.find(".//CensusType1")
        enr = sc.find("Enrollment")
        if c1 is not None and c1.get("TotalEE") in ("", "0"):
            risky.append("CensusType1.TotalEE")
        if enr is not None and enr.get("AdultEquivalent") in ("", "0"):
            risky.append("Enrollment.AdultEquivalent")
    assert not risky, f"Zero in denominator-risk fields: {risky}"
    print("  Census/enrollment denominators non-zero")

    # 5. HDHP deductible-vs-MOOP sanity check.
    # A common extraction slip is populating the deductible field with the MOOP
    # value (they sit next to each other on the SBC). On an HDHP the individual
    # deductible should be LESS than the individual MOOP; equal values almost
    # always mean the MOOP leaked into the deductible. This catches the error
    # that once shipped an OON deductible of $8,300 (the OON MOOP) on a plan
    # whose real deductible was $6,800 (see known_issues.md #8).
    #
    # NOTE: deductible == MOOP is NOT itself an error. On a 100%-after-deductible
    # HDHP the in-network deductible legitimately equals the in-network MOOP.
    # The real error signature is a deductible that has drifted UP to match a
    # DIFFERENT tier's (usually higher) MOOP. We catch it two ways:
    #   (a) A tier's deductible must never EXCEED its own MOOP (that's impossible
    #       and always an extraction error).
    #   (b) For HDHPs, the OON deductible should not be larger than the in-net
    #       deductible unless the SBC genuinely splits them; when it equals the
    #       OON MOOP while the in-net deductible is lower, the OON MOOP leaked in.
    # Both are warnings that HARD-FAIL the build so the values get re-checked.
    ded_moop_problems = []
    tier_fields = [
        ("in-net",  "in_net_ded_ind", "in_net_moop_ind", "in_net_ded_fam", "in_net_moop_fam"),
        ("middle",  "t2_ded_ind",     "t2_moop_ind",     "t2_ded_fam",     "t2_moop_fam"),
        ("OON",     "oon_ded_ind",    "oon_moop_ind",    "oon_ded_fam",    "oon_moop_fam"),
    ]
    for pd in plan_dicts:
        # (a) deductible strictly greater than its own MOOP — impossible, any plan type
        for label, di_k, mi_k, df_k, mf_k in tier_fields:
            di = pd.get(di_k); mi = pd.get(mi_k)
            df = pd.get(df_k); mf = pd.get(mf_k)
            if di is not None and mi is not None and di > mi:
                ded_moop_problems.append(
                    f"{pd['scenario_name']}: {label} individual deductible {di} "
                    f"EXCEEDS its MOOP {mi} (impossible - MOOP leaked into deductible?)")
            if df is not None and mf is not None and df > mf:
                ded_moop_problems.append(
                    f"{pd['scenario_name']}: {label} family deductible {df} "
                    f"EXCEEDS its MOOP {mf} (impossible - MOOP leaked into deductible?)")
        # (b) HDHP OON deductible equals the OON MOOP while the in-net deductible
        #     is lower -> the OON MOOP almost certainly leaked into the OON ded.
        if pd.get("plan_type") == "HDHP":
            in_di = pd.get("in_net_ded_ind"); oon_di = pd.get("oon_ded_ind")
            oon_mi = pd.get("oon_moop_ind")
            if (oon_di is not None and oon_mi is not None and in_di is not None
                    and oon_di == oon_mi and oon_di > in_di):
                ded_moop_problems.append(
                    f"{pd['scenario_name']}: OON individual deductible {oon_di} "
                    f"equals the OON MOOP {oon_mi} but is higher than the in-net "
                    f"deductible {in_di} - did the OON MOOP leak into the OON deductible? "
                    f"(Usually OON ded should match the in-net ded.)")
    assert not ded_moop_problems, (
        "Deductible/MOOP inconsistency detected:\n  " + "\n  ".join(ded_moop_problems) +
        "\nConfirm against the SBC. If the split is genuinely correct, override "
        "this check deliberately and note it to the user.")
    print("  Deductible/MOOP relationships consistent")

    # 6. Rx deductible-application count check (expected counts derived from
    # the plan dicts). On an HDHP, ALL 8 Rx tiers are subject to the medical
    # deductible (IRS rule, forced by the build): 16 PlanElements + 8 T4
    # accordion rows per HDHP scenario. On a PPO, a tier is subject to the
    # deductible ONLY when its *_after_ded flag says so - PPOs legitimately
    # apply deductibles to brand/specialty Rx tiers (a separate Rx deductible
    # or the medical deductible; see the SBC's "after the deductible" wording),
    # so the expectation is computed per dict rather than assumed zero.
    # A count ABOVE expected means a deductible rule leaked into a tier whose
    # dict says False (the classic HDHP-rule-bleeds-into-PPO error, see
    # known_issues.md #10); BELOW expected means a flagged tier never got
    # written. Parse the OUTPUT file we just wrote.
    expected_rx = 0
    expected_t4 = 0
    for pd in plan_dicts:
        if pd.get("plan_type") == "HDHP":
            expected_rx += 16   # 8 tiers x 2 PlanElements, forced regardless of flags
            expected_t4 += 8
        else:
            after = sum(1 for k in RX_SEQ_MAP if pd.get(k + "_after_ded"))
            expected_rx += 2 * after
            expected_t4 += after
    out_root = ET.parse(out_path).getroot()
    rx_ded_y = 0
    t4_ded_true = 0
    for pe in out_root.iter("PlanElement"):
        if pe.get("MedicalGrouping") == "RX" and pe.get("Deductible2Applies") == "Y":
            rx_ded_y += 1
    for n in out_root.iter("PlanDesignAccordion1"):
        if n.get("TierType") == "T4" and n.get("Deductd2T4") == "True":
            t4_ded_true += 1
    rx_problems = []
    if rx_ded_y != expected_rx:
        rx_problems.append(
            f"RX PlanElements with Deductible2Applies=Y: got {rx_ded_y}, "
            f"expected {expected_rx} (16 per HDHP + 2 per non-HDHP tier flagged "
            f"after_ded). Higher = a deductible rule leaked into a tier whose "
            f"dict says False; lower = a flagged tier was never written.")
    if t4_ded_true != expected_t4:
        rx_problems.append(
            f"T4 accordion rows with Deductd2=True: got {t4_ded_true}, "
            f"expected {expected_t4} (8 per HDHP + 1 per non-HDHP tier flagged "
            f"after_ded).")
    assert not rx_problems, (
        "Rx deductible application does not match the plan dicts:\n  " +
        "\n  ".join(rx_problems) +
        "\nThe *_after_ded flags in each plan dict are the source of truth - "
        "re-read the SBC and fix the flags or find the leak. See "
        "known_issues.md #10.")
    print(f"  Rx deductible application matches plan dicts ({rx_ded_y} RX / {t4_ded_true} T4)")

    # 7. HDHP coinsurance sanity checks (WARN + PROMPT, do not auto-fail).
    # The OON coinsurance on an HDHP is NOT derivable from the deductible/MOOP
    # numbers (unlike the in-net 100%-after-deductible case, which IS derived).
    # It must be read off the HDHP's own SBC. A recurring error was writing the
    # HDHP OON band at the PPO's OON rate (e.g. 60%) instead of the HDHP's real
    # rate (e.g. 80%) - the model carried the PPO number across. We cannot force
    # the correct value, but we can flag two suspicious patterns and make the
    # user confirm before the file is trusted. These are WARNINGS: the file is
    # still written, but the build prints a CONFIRM block the user must resolve.
    # See known_issues.md #12.
    coins_warnings = []
    hdhp_pds = [pd for pd in plan_dicts if pd.get("plan_type") == "HDHP"]
    for pd in hdhp_pds:
        in_pays = pd.get("in_net_plan_pays")
        oon_pays = pd.get("oon_plan_pays")
        oon_di = pd.get("oon_ded_ind"); oon_mi = pd.get("oon_moop_ind")
        in_di = pd.get("in_net_ded_ind"); in_mi = pd.get("in_net_moop_ind")
        # Guard (a): HDHP OON band coinsurance differs from the HDHP in-net band.
        # Only meaningful where BOTH tiers actually have a band (ded < MOOP);
        # if in-net ded==MOOP there is no in-net band to compare, so we compare
        # the OON band against a plausible expectation instead of the 100%.
        oon_has_band = (oon_di is not None and oon_mi is not None and oon_di < oon_mi)
        in_has_band = (in_di is not None and in_mi is not None and in_di < in_mi)
        if oon_has_band and in_has_band and oon_pays != in_pays:
            coins_warnings.append(
                f"{pd['scenario_name']}: HDHP OON band pays {int(oon_pays*100)}% but the "
                f"in-net band pays {int(in_pays*100)}%. HDHP OON and in-net coinsurance "
                f"often match - confirm the {int(oon_pays*100)}% OON figure against the SBC.")
        # A former guard (b) also flagged an HDHP OON rate MATCHING a PPO's OON
        # rate in the same file. Removed in v3.2: with a single carrier and one
        # OON schedule across all plans - the normal case - that match is
        # expected, and the guard fired on correct files in POC testing. The
        # real error it targeted (a PPO rate carried onto the HDHP) still trips
        # guard (a) above whenever the carried rate differs from the HDHP's own
        # in-net band, which is the actual error signature.
    if coins_warnings:
        print("\n  *** CONFIRM HDHP COINSURANCE (read off the SBC, not derivable) ***")
        for w in coins_warnings:
            print("   - " + w)
        print("   If these values are correct per the SBC, proceed. If not, fix "
              "the plan dict's oon_plan_pays / in_net_plan_pays and re-run.")
        print("   *** ASK THE USER to confirm the HDHP OON coinsurance before "
              "delivering this file. ***\n")
    else:
        print("  HDHP coinsurance checks passed")

    # 8. UI accordion mirror check (hard fail).
    # The PlanDesignAccordion1 rows (what Claros DISPLAYS) must agree with the
    # design PlanElements (what Claros CALCULATES). This drift is the one
    # documented failure mode that ships silently: the file loads and models
    # correctly but shows the user wrong values (see known_issues.md #5). It is
    # not hypothetical - the Rx accordion rows once kept the shell's leftover
    # Deductd1/Deductd3 "True" defaults while the PlanElements said "N".
    # Only design elements under Plan/SubPlan are compared; the copies under
    # Plan/Utilization/SubPlan are utilization shadows with no design
    # attributes and are ignored (find_subplan reads direct children only).
    def _flag(v):                      # PlanElement "Y"/"N" -> accordion "True"/"False"
        return "True" if v == "Y" else "False"

    def _num_eq(a, b):                 # tolerant numeric compare on attr strings
        if a in (None, "") or b in (None, ""):
            return a in (None, "") and b in (None, "")
        try:
            return abs(float(a) - float(b)) < 1e-6
        except ValueError:
            return a == b

    ACC_GROUP_ROWS = MEDTYPE_TO_GROUPINGS       # GBL elements are never written; skip
    ACC_SEQ_ROWS = MEDTYPE_TO_SEQ                # PCP / Specialist rows mirror one element

    # Per-scenario service-level coinsurance overrides (t1_service_coins):
    # these elements legitimately differ from their rollup row, but must match
    # the override the plan dict declared - verified here instead of blanket-
    # exempted, so a typo'd override still fails loudly. (Tier 1 only; the
    # loops only apply svc_coins in SubPlan 1.)
    svc_overrides = []
    for pd in plan_dicts:
        ov = {}
        for seq, v in pd.get("t1_service_coins", {}).items():
            pp_, ad_ = (v["plan_pays"], bool(v.get("after_ded", True))) \
                if isinstance(v, dict) else (v, True)
            ov[seq] = (pp_, ad_)
        svc_overrides.append(ov)

    mirror_problems = []
    rx_row_to_key = {v: k for k, v in RX_ACCORDION_MAP.items()}
    for si, sc in enumerate(out.findall("Scenario")):
        plan = sc.find("Plan")
        rx_pes = {pe.get("UIMedCatId_Sequence"): pe
                  for pe in get_rx_planelements(plan).findall("PlanElement")}
        for acc in sc.iter("PlanDesignAccordion1"):
            tier = acc.get("TierType")
            where = f"scenario {si} {tier}"
            if tier == "T4":
                key = rx_row_to_key.get((acc.get("RxType"), acc.get("RxSubType")))
                if key is None:
                    continue
                for seq in RX_SEQ_MAP[key]:
                    pe = rx_pes.get(seq)
                    if pe is None:
                        continue
                    for i in ("1", "2", "3"):
                        if acc.get(f"Deductd{i}T4") != _flag(pe.get(f"Deductible{i}Applies")):
                            mirror_problems.append(
                                f"{where} {key}: Deductd{i}T4={acc.get(f'Deductd{i}T4')} "
                                f"but PlanElement {seq} Deductible{i}Applies="
                                f"{pe.get(f'Deductible{i}Applies')}")
                    if acc.get("CopayMinT4") not in (None, ""):
                        if not _num_eq(acc.get("CopayMinT4"), pe.get("Copay1Min")):
                            mirror_problems.append(
                                f"{where} {key}: CopayMinT4={acc.get('CopayMinT4')} "
                                f"but PlanElement {seq} Copay1Min={pe.get('Copay1Min')}")
                    if acc.get("ParticipantPctT4") not in (None, ""):
                        pct = None if pe.get("Copay1Perc") in (None, "") \
                            else str(float(pe.get("Copay1Perc")) * 100)
                        if not _num_eq(acc.get("ParticipantPctT4"), pct):
                            mirror_problems.append(
                                f"{where} {key}: ParticipantPctT4="
                                f"{acc.get('ParticipantPctT4')} but PlanElement {seq} "
                                f"Copay1Perc={pe.get('Copay1Perc')}")
                        if not _num_eq(acc.get("CopayMaxT4"), pe.get("Copay1Max")):
                            mirror_problems.append(
                                f"{where} {key}: CopayMaxT4={acc.get('CopayMaxT4')} "
                                f"but PlanElement {seq} Copay1Max={pe.get('Copay1Max')}")
                continue
            n = tier[1]
            sp = find_subplan(plan, n)
            if sp is None:
                continue
            med = acc.get("MedType")
            acc_ded = tuple(acc.get(f"Deductd{i}T{n}") for i in ("1", "2", "3"))
            acc_coins = acc.get(f"CoinsurancePlanPctT{n}")
            acc_copay = acc.get(f"CopayMinT{n}")
            if med in ACC_SEQ_ROWS:
                pe = next((p for p in sp.findall("PlanElement")
                           if p.get("UIMedCatId_Sequence") == ACC_SEQ_ROWS[med]), None)
                if pe is None:
                    continue
                pe_ded = tuple(_flag(pe.get(f"Deductible{i}Applies")) for i in ("1", "2", "3"))
                if acc_ded != pe_ded:
                    mirror_problems.append(
                        f"{where} {med}: Deductd flags {acc_ded} but PlanElement "
                        f"{ACC_SEQ_ROWS[med]} says {pe_ded}")
                if pe.get("Copay1Flag") == "Y":
                    if not _num_eq(acc_copay, pe.get("Copay1Min")):
                        mirror_problems.append(
                            f"{where} {med}: CopayMinT{n}={acc_copay} but PlanElement "
                            f"{ACC_SEQ_ROWS[med]} Copay1Min={pe.get('Copay1Min')}")
                    # Copay+coinsurance combination: the member share lives in
                    # Copay1Perc (verified encoding) and must mirror to
                    # ParticipantPctT{n}; a plain copay must leave it empty.
                    acc_part = acc.get(f"ParticipantPctT{n}")
                    pe_part = pe.get("Copay1Perc")
                    want = None if pe_part in (None, "") else str(float(pe_part) * 100)
                    if not _num_eq(acc_part, want):
                        mirror_problems.append(
                            f"{where} {med}: ParticipantPctT{n}={acc_part} but "
                            f"PlanElement {ACC_SEQ_ROWS[med]} Copay1Perc={pe_part}")
                elif acc_copay not in (None, ""):
                    mirror_problems.append(
                        f"{where} {med}: accordion shows copay {acc_copay} but "
                        f"PlanElement {ACC_SEQ_ROWS[med]} is coinsurance-typed")
                if not _num_eq(acc_coins,
                               None if pe.get("CoInsRate") in (None, "")
                               else str(float(pe.get("CoInsRate")) * 100)):
                    mirror_problems.append(
                        f"{where} {med}: CoinsurancePlanPctT{n}={acc_coins} but "
                        f"PlanElement {ACC_SEQ_ROWS[med]} CoInsRate={pe.get('CoInsRate')}")
            elif med == "IP":
                # v4.0 (finding D2): NO copay exemption for the inpatient group.
                # facility_lines writes all five IP rows with one value, so the
                # IP display row must show exactly what they carry: same copay,
                # same member share, plan pct 100, same deductible checkboxes and
                # the same Per Admit / Per Day code - or, for coinsurance-typed
                # rows, no copay and the same coinsurance. v3 skipped copay-typed
                # IP elements under a coinsurance rollup, which passed a file
                # whose display showed 90% coinsurance over $150 copays.
                ip_pes = [p for p in sp.findall("PlanElement")
                          if p.get("MedicalGrouping") in ACC_GROUP_ROWS["IP"]]
                kinds = {p.get("Copay1Flag") == "Y" for p in ip_pes}
                if len(kinds) > 1:
                    mirror_problems.append(f"{where} IP: inpatient rows mix copay and "
                                           "coinsurance - one value must cover all five")
                for pe in ip_pes:
                    seq_ = pe.get("UIMedCatId_Sequence")
                    pe_ded = tuple(_flag(pe.get(f"Deductible{i}Applies")) for i in ("1", "2", "3"))
                    if acc_ded != pe_ded:
                        mirror_problems.append(f"{where} IP: Deductd flags {acc_ded} but "
                                               f"PlanElement {seq_} says {pe_ded}")
                    if pe.get("Copay1Flag") == "Y":
                        perc = pe.get("Copay1Perc")
                        want_part = None if perc in (None, "") else str(float(perc) * 100)
                        if not _num_eq(acc_copay, pe.get("Copay1Min")):
                            mirror_problems.append(f"{where} IP: CopayMinT{n}={acc_copay!r} but "
                                                   f"PlanElement {seq_} Copay1Min={pe.get('Copay1Min')}")
                        if not _num_eq(acc.get(f"ParticipantPctT{n}"), want_part):
                            mirror_problems.append(f"{where} IP: ParticipantPctT{n}="
                                                   f"{acc.get(f'ParticipantPctT{n}')!r} but PlanElement "
                                                   f"{seq_} Copay1Perc={perc}")
                        if not _num_eq(acc_coins, "100"):
                            mirror_problems.append(f"{where} IP: CoinsurancePlanPctT{n}={acc_coins} "
                                                   "but a copay row must display plan pct 100")
                        if pe.get("IPCopayPerDay") not in IP_BASIS_CODES.values():
                            mirror_problems.append(f"{where} IP: PlanElement {seq_} IPCopayPerDay="
                                                   f"{pe.get('IPCopayPerDay')!r} is not a Per Admit / "
                                                   "Per Day code")
                        if acc.get("IPCopayPerDay") != pe.get("IPCopayPerDay"):
                            mirror_problems.append(f"{where} IP: accordion IPCopayPerDay="
                                                   f"{acc.get('IPCopayPerDay')!r} but PlanElement {seq_} "
                                                   f"IPCopayPerDay={pe.get('IPCopayPerDay')!r}")
                    else:
                        if acc_copay not in (None, ""):
                            mirror_problems.append(f"{where} IP: accordion shows copay {acc_copay} but "
                                                   f"PlanElement {seq_} is coinsurance-typed")
                        if not _num_eq(acc_coins, str(float(pe.get("CoInsRate")) * 100)):
                            mirror_problems.append(f"{where} IP: CoinsurancePlanPctT{n}={acc_coins} but "
                                                   f"PlanElement {seq_} CoInsRate={pe.get('CoInsRate')}")
            elif med in ACC_GROUP_ROWS:
                # Rollup row. Copay-typed elements (flat/per-day copay exceptions,
                # e.g. ER or urgent care from t1_flat_copays / cross_tier_copays)
                # legitimately differ from the rollup; every COINSURANCE-typed
                # element in the grouping must match the row exactly. If the row
                # itself shows a copay (per-day IP), copay-typed elements must
                # match that copay instead.
                for pe in sp.findall("PlanElement"):
                    if pe.get("MedicalGrouping") not in ACC_GROUP_ROWS[med]:
                        continue
                    if pe.get("Copay1Flag") == "Y":
                        if acc_copay not in (None, "") and not _num_eq(acc_copay, pe.get("Copay1Min")):
                            mirror_problems.append(
                                f"{where} {med}: rollup copay {acc_copay} but PlanElement "
                                f"{pe.get('UIMedCatId_Sequence')} Copay1Min={pe.get('Copay1Min')}")
                        continue   # copay-typed exceptions under a coins rollup are by design
                    seq_ = pe.get("UIMedCatId_Sequence")
                    ov = svc_overrides[si].get(seq_) if tier == "T1" and si < len(svc_overrides) else None
                    if ov is not None:
                        # Declared per-service override: must match the plan dict,
                        # not the rollup row.
                        pp_, ad_ = ov
                        want_flags = ("True",) * 3 if ad_ else ("False",) * 3
                        pe_ded = tuple(_flag(pe.get(f"Deductible{i}Applies")) for i in ("1", "2", "3"))
                        if pe_ded != want_flags:
                            mirror_problems.append(
                                f"{where} {med}: service override seq {seq_} deductible "
                                f"flags {pe_ded} but t1_service_coins declares "
                                f"after_ded={ad_}")
                        if not _num_eq(pe.get("CoInsRate"), str(pp_)):
                            mirror_problems.append(
                                f"{where} {med}: service override seq {seq_} "
                                f"CoInsRate={pe.get('CoInsRate')} but t1_service_coins "
                                f"declares plan_pays={pp_}")
                        continue
                    pe_ded = tuple(_flag(pe.get(f"Deductible{i}Applies")) for i in ("1", "2", "3"))
                    if acc_ded != pe_ded:
                        mirror_problems.append(
                            f"{where} {med}: Deductd flags {acc_ded} but PlanElement "
                            f"{pe.get('UIMedCatId_Sequence')} says {pe_ded}")
                    if not _num_eq(acc_coins,
                                   None if pe.get("CoInsRate") in (None, "")
                                   else str(float(pe.get("CoInsRate")) * 100)):
                        mirror_problems.append(
                            f"{where} {med}: CoinsurancePlanPctT{n}={acc_coins} but "
                            f"PlanElement {pe.get('UIMedCatId_Sequence')} "
                            f"CoInsRate={pe.get('CoInsRate')}")
    assert not mirror_problems, (
        "UI accordion does not mirror the PlanElements:\n  " +
        "\n  ".join(mirror_problems[:20]) +
        ("\n  ... plus %d more" % (len(mirror_problems) - 20) if len(mirror_problems) > 20 else "") +
        "\nThe accordion drives the Claros DISPLAY and the PlanElements drive the "
        "MATH; when they disagree the file loads and models correctly but shows "
        "the user wrong values. Fix whichever layer is wrong and re-run. See "
        "known_issues.md #5.")
    print(f"  UI accordion mirrors PlanElements ({sum(1 for sc in out.findall('Scenario') for _ in sc.iter('PlanDesignAccordion1'))} rows checked)")

    # 9. Stray $1 placeholder copay check (hard fail).
    # Claros shells frequently pre-flag Tier 1 services (grouping OP/RAD/LP)
    # as copay-type with a $1 placeholder (Copay1Flag="Y", Copay1Min="1"),
    # anticipating a copay design. Logic that writes coinsurance only to
    # elements NOT already flagged as copays skips these and ships the $1.
    # The Tier 1 loop handles every element explicitly, so nothing should
    # survive - this is the assertion from known_issues.md #13, verifying it.
    # Scoped to design SubPlans (Plan direct children); the bundled blank
    # reference has no placeholders, but client-supplied Claros exports do.
    # The build's own placeholder strategy ($1 admin fees, 1 EE) touches fee
    # and census nodes, never PlanElement copays, so no collision here.
    stray = [f"scenario {si} SubPlan {sp.get('SubPlanId')} seq {pe.get('UIMedCatId_Sequence')}"
             for si, sc in enumerate(out.findall("Scenario"))
             for sp in sc.find("Plan").findall("SubPlan")
             for pe in sp.findall("PlanElement")
             if pe.get("Copay1Flag") == "Y" and pe.get("Copay1Min") == "1"]
    assert not stray, (
        "Stray $1 placeholder copays survived the build:\n  " +
        "\n  ".join(stray) +
        "\nThe reference shell pre-flagged these as copay-type and the build "
        "never rewrote them. Every element in the tier must be populated with "
        "the real copay or converted to coinsurance. See known_issues.md #13.")
    print("  No stray $1 placeholder copays")

    # 10. Coverage gate: nothing in the shell may be invisible to the build.
    # The tier loops write every element whose MedicalGrouping they know; an
    # element with an UNKNOWN grouping (a client-supplied shell with e.g. "MH"
    # or "TRANS" rows), a design SubPlan the tiers never visit, or an Rx
    # element outside RX_SEQ_MAP would pass through silently at the shell's
    # template defaults - values that belong to no plan design. Refuse all
    # three. GBL is exempt by design (global rollup rows, deliberately
    # untouched); Utilization shadows are excluded (design SubPlans only).
    coverage_problems = []
    gbl_count = 0
    for si, sc in enumerate(out.findall("Scenario")):
        plan = sc.find("Plan")
        for sp in plan.findall("SubPlan"):
            spid = sp.get("SubPlanId")
            if spid not in ("1", "2", "3", "4"):
                coverage_problems.append(
                    f"scenario {si}: design SubPlanId={spid} - the tier loops "
                    f"only write SubPlans 1/2/3 (+Rx); this one was never touched")
                continue
            for pe in sp.findall("PlanElement"):
                grp = pe.get("MedicalGrouping")
                seq = pe.get("UIMedCatId_Sequence")
                if grp == "GBL":
                    gbl_count += 1
                    continue
                if grp not in KNOWN_GROUPINGS:
                    coverage_problems.append(
                        f"scenario {si} SubPlan {spid} seq {seq}: unknown "
                        f"MedicalGrouping={grp!r} - no branch writes this "
                        f"grouping, so it kept the shell's template values")
                elif grp == "RX" and not any(seq in seqs for seqs in RX_SEQ_MAP.values()):
                    coverage_problems.append(
                        f"scenario {si} SubPlan {spid} seq {seq}: RX element "
                        f"outside RX_SEQ_MAP - the Rx loop never wrote it")
    assert not coverage_problems, (
        "Shell contains elements the build never writes:\n  " +
        "\n  ".join(coverage_problems[:20]) +
        ("\n  ... plus %d more" % (len(coverage_problems) - 20) if len(coverage_problems) > 20 else "") +
        "\nA reference export with unexpected groupings/SubPlans/Rx rows needs "
        "an explicit decision: extend MEDTYPE_TO_GROUPINGS + the tier loops to "
        "handle it, or use a matching 2-tier/3-tier blank export instead. "
        "Never ship shell template values as if they were the client's design.")
    print(f"  Coverage gate passed (all design elements written; {gbl_count} GBL rollups exempt by design)")

    # 11. In-network preventive covered in full (hard fail).
    # ACA-compliant plans cover preventive care/immunizations (PH sequences
    # 504/510) at 100% in-network with no deductible - on EVERY plan type,
    # HDHPs included (IRS safe harbor). The blank Claros export even SHIPS them
    # that way; v3.1's tier loops overwrote them with the tier coinsurance, and
    # because every other check validates structure rather than benefit
    # content, the wrong file passed all ten checks in 10 of 10 real-client
    # POC tests. This check makes that failure impossible to ship: every
    # preventive element in an in-network SubPlan (1 and 2) must be
    # Copay1Flag="N", all three DeductibleNApplies="N", CoInsRate="1", unless
    # that scenario's plan dict explicitly opts out with
    # preventive_in_net_covered=False (rare non-ACA designs only). OON
    # (SubPlan 3) is deliberately not checked - preventive is not mandated
    # out-of-network and keeps the tier default.
    prev_problems = []
    for si, sc in enumerate(out.findall("Scenario")):
        pd = plan_dicts[si] if si < len(plan_dicts) else {}
        if not pd.get("preventive_in_net_covered", True):
            continue
        plan = sc.find("Plan")
        for spid in ("1", "2"):
            sp = find_subplan(plan, spid)
            if sp is None:
                continue
            for pe in sp.findall("PlanElement"):
                if pe.get("UIMedCatId_Sequence") not in PREVENTIVE_SEQS:
                    continue
                bad = []
                if pe.get("Copay1Flag") != "N":
                    bad.append(f"Copay1Flag={pe.get('Copay1Flag')}")
                for i in ("1", "2", "3"):
                    if pe.get(f"Deductible{i}Applies") != "N":
                        bad.append(f"Deductible{i}Applies={pe.get(f'Deductible{i}Applies')}")
                if pe.get("CoInsRate") not in ("1", "1.0"):
                    bad.append(f"CoInsRate={pe.get('CoInsRate')}")
                if bad:
                    prev_problems.append(
                        f"scenario {si} SubPlan {spid} seq "
                        f"{pe.get('UIMedCatId_Sequence')}: {', '.join(bad)}")
    assert not prev_problems, (
        "In-network preventive care is not covered in full:\n  " +
        "\n  ".join(prev_problems) +
        "\nACA plans cover preventive/immunizations (sequences 504/510) at 100% "
        "in-network with no deductible, on HDHPs too. A tier loop overwrote "
        "them - the preventive branch must run BEFORE any grouping fill or "
        "coinsurance default. If this plan genuinely cost-shares preventive "
        "(the SBC literally prints it), set preventive_in_net_covered=False "
        "in that plan dict and flag the judgment call to the user.")
    print(f"  In-network preventive covered in full (sequences {'/'.join(PREVENTIVE_SEQS)})")

    # Check 12 (v4.0): FACILITY READ APPLIED. Every facility sequence in the
    # Tier 1 SubPlan of a copay-style scenario must carry exactly what
    # resolve_facility_read() decided from the plan dict's facility_lines -
    # the declared copay / copay+coins / coinsurance override, or the plain
    # tier default when the line was None or flagged. All five IP rows are
    # therefore also identical to each other. Catches a facility row that was
    # dropped, half-written, or overwritten by a later branch of the tier loop.
    fac_problems = []
    for si, sc in enumerate(out.findall("Scenario")):
        pd = plan_dicts[si] if si < len(plan_dicts) else {}
        if pd.get("plan_type") not in COPAY_PLAN_TYPES:
            continue
        expected = pd.get("_facility_expected", {})
        sp1 = find_subplan(sc.find("Plan"), "1")
        for pe in sp1.findall("PlanElement"):
            seq = pe.get("UIMedCatId_Sequence")
            if seq not in FACILITY_SEQS:
                continue
            want = expected.get(seq, ("coins", round(float(pd["in_net_plan_pays"]), 4), True))
            got = _element_facility_norm(pe)
            if got != want:
                fac_problems.append(f"scenario {si} seq {seq}: element is {got}, "
                                    f"facility read declared {want}")
    assert not fac_problems, (
        "Facility rows do not match the facility read:\n  " +
        "\n  ".join(fac_problems[:20]) +
        "\nThe facility sequences (IP 101-105, OP 308/309/311) must come ONLY from "
        "facility_lines via resolve_facility_read(). Fix the plan dict, never the check.")
    print("  Facility read applied (IP 101-105, OP 308/309/311)")

    print("All checks passed\n")


def print_coverage_table(out_path):
    """READ-SIDE CONFIRMATION (prints only after a passing build). Every
    numbered check above validates the BUILD against the reference and the
    plan dicts; none of them can know whether the plan dicts match the SBC.
    This table is the countermeasure: it renders what each service sequence
    actually received so the builder can walk the SBC's benefit table line by
    line against it BEFORE presenting the file. In v3.1 POC testing, six of
    ten failures were copays the SBC printed and the build never placed -
    every one of them would have been visible on this table. Do not present
    the output file to the user until this table has been compared against
    the SBC and any mismatch resolved."""
    root = ET.parse(out_path).getroot()
    tier_names = {"1": "Tier 1 (in-network)", "2": "Tier 2 (middle)", "3": "OON"}
    for si, sc in enumerate(root.findall("Scenario")):
        print(f"\n=== SERVICE COVERAGE - scenario {si}: {sc.get('ScenarioName')} ===")
        plan = sc.find("Plan")
        for spid in ("1", "2", "3"):
            sp = find_subplan(plan, spid)
            if sp is None:
                continue
            rows = {}
            for pe in sp.findall("PlanElement"):
                grp = pe.get("MedicalGrouping")
                if grp in ("GBL", "RX"):
                    continue
                seq = pe.get("UIMedCatId_Sequence")
                ded = "after ded" if pe.get("Deductible2Applies") == "Y" else "ded waived"
                if pe.get("Copay1Flag") == "Y":
                    desc = f"${pe.get('Copay1Min')} copay"
                    perc = pe.get("Copay1Perc")
                    if perc not in (None, ""):
                        desc += f" + {round(float(perc) * 100)}% member coins"
                    desc += f", {ded}"
                else:
                    rate = float(pe.get("CoInsRate") or 1)
                    if rate == 1 and pe.get("Deductible2Applies") == "N":
                        desc = "covered 100%, no ded"
                    else:
                        desc = f"{round(rate * 100)}% plan pays, {ded}"
                rows.setdefault((grp, desc), []).append(seq)
            print(f"  {tier_names[spid]}:")
            for (grp, desc), seqs in sorted(rows.items()):
                print(f"    {grp:<4} {','.join(sorted(seqs))}: {desc}")
    print("\n*** Compare this table against the SBC's benefit table line by line "
          "before presenting the file. Checks validate the BUILD, not the "
          "READING - a copay the SBC prints but no row above shows was never "
          "extracted. ***\n")


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------
def _norm_facility_line(v, where, ip=False):
    """Normalize one facility_lines value to a comparable tuple:
        None                                   -> ("default",)
        scalar / {"copay": ...}                -> ("copay", copay, after_ded, plan_pays|None)
        {"plan_pays": ..., "after_ded": ...}   -> ("coins", plan_pays, after_ded)
    Two lines are 'the same value' exactly when their tuples are equal: same
    copay, same coinsurance component, same deductible treatment."""
    if v is None:
        return ("default",)
    if isinstance(v, bool) or not isinstance(v, (int, float, dict)):
        raise SystemExit(f"*** BAD facility_lines value at {where}: {v!r} - use None, "
                         "a copay, a copay_spec dict, or {'plan_pays':, 'after_ded':}.")
    if isinstance(v, dict) and "copay" not in v:
        if "plan_pays" not in v or set(v) - {"plan_pays", "after_ded"}:
            raise SystemExit(f"*** BAD facility_lines value at {where}: {v!r}")
        return ("coins", round(float(v["plan_pays"]), 4), bool(v.get("after_ded", True)))
    allowed = {"copay", "plan_pays", "after_ded"} | ({"basis"} if ip else set())
    if isinstance(v, dict) and set(v) - allowed:
        extra = " ('basis' belongs only on inpatient rows)" if "basis" in v else ""
        raise SystemExit(f"*** BAD facility_lines value at {where}: {v!r}{extra}")
    c, ad, pp = copay_spec(v)
    base = ("copay", float(c), ad, None if pp is None else round(float(pp), 4))
    if not ip:
        return base
    # Inpatient copay: the Per Admit / Per Day basis is part of the value, so
    # three rows that differ only in basis are a discrepancy (pinned F1).
    basis = v.get("basis") if isinstance(v, dict) else None
    if basis not in IP_BASIS_VALUES:
        raise SystemExit(
            f"*** {where}: an inpatient copay must state its basis as a dict, e.g. "
            '{"copay": 150, "basis": "per_admit"} - "per_admit" when the SBC says per '
            'admission / per stay, "per_day" when it says per day, "not_stated" when it '
            "says neither (built Per Admit and flagged). Got " + repr(v))
    return base + (basis,)


def _element_facility_norm(pe):
    """The same tuple shape as _norm_facility_line, read back off a built
    PlanElement (check 12). A plain tier-default row reads as ("coins", ...).
    An inpatient copay row also reads back its Per Admit / Per Day code."""
    if pe.get("Copay1Flag") == "Y":
        perc = pe.get("Copay1Perc")
        pp = None if perc in (None, "") else round(1 - float(perc), 4)
        base = ("copay", float(pe.get("Copay1Min", "0")),
                pe.get("Deductible2Applies") == "Y", pp)
        if pe.get("MedicalGrouping") == "IP":
            return base + (pe.get("IPCopayPerDay"),)
        return base
    return ("coins", round(float(pe.get("CoInsRate", "1")), 4),
            pe.get("Deductible2Applies") == "Y")


def _describe_line(n):
    if n[0] == "default":
        return "tier default coinsurance"
    if n[0] == "coins":
        return f"{int(round(n[1] * 100))}% plan pays, " + ("after ded" if n[2] else "ded waived")
    s = f"${n[1]:g} copay"
    if len(n) == 5:
        s += {"per_admit": " per admit", "per_day": " per day",
              "not_stated": " (basis not stated: built per admit)"}[n[4]]
    if n[3] is not None:
        s += f" + {int(round((1 - n[3]) * 100))}% member coins"
    return s + (", after ded" if n[2] else ", ded waived")


def resolve_facility_read(pd):
    """FACILITY READ (v4.0, finding D1). Turns the SBC's facility rows
    (pd["facility_lines"]) into sequence placements under ONE fixed rule, so
    identical SBCs always produce identical facility rows:
      IP 101-105: placed when the hospital-stay, mental-health-inpatient and
        childbirth-facility rows print the SAME value. When they differ the
        build STOPS until the operator answers the pinned discrepancy question
        (ip_discrepancy_choice); subtype order is not guaranteed, so a
        per-subtype placement is never attempted.
      OP 309/311: one row -> both; a labeled hospital/ASC split -> each;
        an UNLABELED split -> neither, flagged.
      OP 308: an explicit op_other line; otherwise the same value as 309
        (builder decision), and not placed when 309 is not.
    Returns (resolved_copy, read_lines, manual_items). The caller's dict is
    not modified."""
    name = pd.get("scenario_name", "?")
    if pd["plan_type"] not in COPAY_PLAN_TYPES:
        if "facility_lines" in pd:
            raise SystemExit(
                f"*** {name}: facility_lines is not used on an HDHP (no pre-deductible "
                "facility copays on an HSA-qualified plan). Remove it; list any "
                "off-default HDHP facility line as a set-manually-in-Claros item.")
        return pd, [], []

    problems = []
    if "t1_ip_copay" in pd:
        problems.append("t1_ip_copay is set directly - inpatient copays come from "
                        "facility_lines (ip_hospital_stay / ip_mental_health / ip_childbirth)")
    for field, banned in (("t1_flat_copays", FACILITY_SEQS),
                          ("t1_service_coins", FACILITY_SEQS),
                          ("t2_flat_copays", IP_SEQS),
                          ("cross_tier_copays", IP_SEQS)):
        hits = sorted(set(pd.get(field, {})) & set(banned))
        if hits:
            problems.append(f"{field} carries facility sequence(s) {hits} - facility rows "
                            "come only from facility_lines")
    fl = pd.get("facility_lines")
    if not isinstance(fl, dict):
        problems.append("facility_lines is required for PPO/HMO/EPO - transcribe the SBC's "
                        "hospital-stay, mental-health-inpatient, childbirth-facility and "
                        "outpatient-surgery facility rows (None = tier default coinsurance)")
    else:
        unknown = sorted(set(fl) - set(FACILITY_LINE_KEYS))
        if unknown:
            problems.append(f"facility_lines has unknown key(s) {unknown}")
        missing = [k for k in IP_LINE_KEYS if k not in fl]
        if missing:
            problems.append(f"facility_lines is missing {missing} (use None for a row that "
                            "prints the tier default)")
        single = "op_surgery" in fl
        split = ("op_surgery_hospital" in fl, "op_surgery_asc" in fl)
        if single == any(split) or (any(split) and not all(split)):
            problems.append("facility_lines needs EXACTLY ONE outpatient-surgery form: "
                            "op_surgery, or BOTH op_surgery_hospital and op_surgery_asc")
    if problems:
        raise SystemExit(f"*** FACILITY READ REFUSED - {name} - BUILD NOT STARTED ***\n  "
                         + "\n  ".join(problems))

    out = copy.deepcopy(pd)
    flat = out.setdefault("t1_flat_copays", {})
    svc = out.setdefault("t1_service_coins", {})
    expected, read, manual = {}, [], []

    def place(seqs, raw, label):
        n = _norm_facility_line(raw, f"{name}.{label}")
        for seq in seqs:
            if n[0] == "copay":
                flat[seq] = raw
                expected[seq] = n
            elif n[0] == "coins":
                svc[seq] = {"plan_pays": n[1], "after_ded": n[2]}
                expected[seq] = n
        return n

    # ---- Inpatient group ----
    ip_n = {k: _norm_facility_line(fl[k], f"{name}.{k}", ip=True) for k in IP_LINE_KEYS}
    choice = fl.get("ip_discrepancy_choice")
    if len(set(ip_n.values())) == 1:
        if choice is not None:
            raise SystemExit(f"*** {name}: ip_discrepancy_choice is set but the three IP rows "
                             "agree - remove the stale answer (it is valid only when they differ).")
        ip_raw, ip_how = fl["ip_hospital_stay"], "all three IP rows agree"
    else:
        rows = "\n  ".join(f"{k}: {_describe_line(v)}" for k, v in ip_n.items())
        if choice is None:
            raise SystemExit(
                f"*** STOP - ASK THE OPERATOR (pinned question F1) - {name} ***\n"
                f"The SBC's inpatient facility rows do not agree:\n  {rows}\n"
                "Claros cannot tell the five IP subtypes apart reliably, so the build "
                "will not pick one. Ask SKILL.md's pinned question F1 VERBATIM, record "
                "the answer as facility_lines['ip_discrepancy_choice'], and re-run.")
        if choice == "hospital_stay_all":
            ip_raw, ip_how = fl["ip_hospital_stay"], "operator chose the hospital-stay value for all five"
        elif choice == "tier_default":
            ip_raw, ip_how = None, "operator chose tier default coinsurance for all five"
        elif isinstance(choice, dict) and set(choice) == {"custom"}:
            ip_raw, ip_how = choice["custom"], "operator supplied one value for all five"
        else:
            raise SystemExit(f"*** {name}: ip_discrepancy_choice {choice!r} must be one of "
                             f"{IP_DISCREPANCY_CHOICES} or {{'custom': <line>}}.")
        manual.append(f"Inpatient facility rows differ on the SBC ({rows.replace(chr(10) + '  ', '; ')}); "
                      f"{ip_how} - set the differing inpatient subtype manually in Claros.")
    ip_final = _norm_facility_line(ip_raw, f"{name}.ip", ip=True)
    if ip_final[0] == "copay":
        basis = ip_final[4]
        code = IP_BASIS_CODES[IP_BASIS_DEFAULT if basis == "not_stated" else basis]
        out["t1_ip_copay"] = {k: v for k, v in ip_raw.items() if k != "basis"}
        out["_ip_basis_code"] = code
        for seq in IP_SEQS:
            expected[seq] = ip_final[:4] + (code,)
        if basis == "not_stated":
            manual.append("The SBC prints an inpatient facility copay without saying per "
                          "admission or per day; built Per Admit (the Claros default) - "
                          "confirm the Per Admit / Per Day setting in Claros.")
    elif ip_final[0] == "coins":
        for seq in IP_SEQS:
            svc[seq] = {"plan_pays": ip_final[1], "after_ded": ip_final[2]}
            expected[seq] = ip_final
        out["_ip_coins_override"] = (ip_final[1], ip_final[2])
    read.append(f"IP 101-105  <- {_describe_line(ip_final)} ({ip_how})")

    # ---- Outpatient surgery facility ----
    hosp_raw, hosp_placed = None, True   # what 309 receives; 308 follows it (below)
    if "op_surgery" in fl:
        if fl["op_surgery"] == UNLABELED_SPLIT:
            hosp_placed = False
            read.append("OP 309/311  <- NOT PLACED: the source prints differing outpatient "
                        "surgery facility values without labeling hospital vs ASC")
            manual.append("Outpatient surgery facility values differ without a hospital/ASC "
                          "label - set OP Surgery Hospital and OP Surgery ASC manually in Claros.")
        else:
            hosp_raw = fl["op_surgery"]
            n = place((OP_SURGERY_HOSPITAL_SEQ, OP_SURGERY_ASC_SEQ), fl["op_surgery"], "op_surgery")
            read.append(f"OP 309+311  <- {_describe_line(n)} (one facility row covers both settings)")
    else:
        hosp_raw = fl["op_surgery_hospital"]
        n = place((OP_SURGERY_HOSPITAL_SEQ,), fl["op_surgery_hospital"], "op_surgery_hospital")
        read.append(f"OP 309      <- {_describe_line(n)} (source-labeled hospital)")
        n = place((OP_SURGERY_ASC_SEQ,), fl["op_surgery_asc"], "op_surgery_asc")
        read.append(f"OP 311      <- {_describe_line(n)} (source-labeled ASC)")

    # ---- OP Other ----
    # Builder decision (October 2026): with no explicit op_other line, OP Other
    # (308) carries the SAME value as OP Surgery - Hospital (309). An explicit
    # op_other line always wins. When 309 is not placed (unlabeled split), 308
    # is not placed either and is flagged with it.
    if "op_other" in fl:
        n = place((OP_OTHER_SEQ,), fl["op_other"], "op_other")
        read.append(f"OP 308      <- {_describe_line(n)} (explicit op_other line)")
    elif hosp_placed:
        n = place((OP_OTHER_SEQ,), hosp_raw, "op_other(follows 309)")
        read.append(f"OP 308      <- {_describe_line(n)} (no op_other line: follows OP Surgery Hospital 309)")
    else:
        read.append("OP 308      <- NOT PLACED: follows OP Surgery Hospital 309, which is not placed")
        manual.append("OP Other follows OP Surgery Hospital, which was not placed (unlabeled "
                      "split) - set OP Other manually in Claros.")

    out["_facility_expected"] = expected
    return out, read, manual


def main():
    global PLAN_DICTS   # replaced below by the facility-resolved copies (v4.0)
    # Reject unknown plan types BEFORE building anything. The branching below
    # is copay-style vs HDHP; a typo'd or unsupported plan_type ("HSA", "Pos",
    # "hdhp") would otherwise fall into one branch silently and produce a
    # structurally valid file with the wrong plan design.
    bad = [f"{pd.get('scenario_name', '?')}: plan_type={pd.get('plan_type')!r}"
           for pd in PLAN_DICTS if pd.get("plan_type") not in VALID_PLAN_TYPES]
    if bad:
        raise SystemExit(
            "*** UNSUPPORTED plan_type - BUILD NOT STARTED ***\n  "
            + "\n  ".join(bad) +
            f"\nValid values: {', '.join(VALID_PLAN_TYPES)} (exact casing). "
            "PPO/HMO/EPO are copay-style designs; HDHP is coinsurance-only "
            "with the IRS Rx rule. Fix the plan dict and re-run.")

    # FACILITY READ (v4.0): resolve every copay-style plan's facility rows
    # BEFORE building. Any refusal or STOP raised here leaves no file.
    resolved, facility_reports = [], []
    for pd_ in PLAN_DICTS:
        r, read_, manual_ = resolve_facility_read(pd_)
        resolved.append(r)
        facility_reports.append((pd_["scenario_name"], read_, manual_))
    PLAN_DICTS = resolved

    ET.register_namespace("xsi", "http://www.w3.org/2001/XMLSchema-instance")
    ET.register_namespace("xsd", "http://www.w3.org/2001/XMLSchema")

    tree = load_reference(REF_FILE)
    root = tree.getroot()
    ref_root_copy = copy.deepcopy(root)   # pristine reference for structural check

    ensure_scenario_count(root, len(PLAN_DICTS))
    root.set("TotalPlans", str(len(PLAN_DICTS)))
    root.set("DataYear", DATA_YEAR)

    for sc, pd in zip(root.findall("Scenario"), PLAN_DICTS):
        apply_minimal_placeholders(sc)
        apply_plan_design(sc, pd)

    while len(root.findall("Scenario")) > len(PLAN_DICTS):
        root.remove(root.findall("Scenario")[-1])

    xml_str = ET.tostring(root, encoding="unicode")
    xml_str = xml_str.replace(
        '<Scenarios ',
        '<Scenarios xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" '
        'xmlns:xsd="http://www.w3.org/2001/XMLSchema" ',
        1
    )
    xml_str = xml_str.replace("\r\n", "\n").replace("\n", "\r\n")

    # -----------------------------------------------------------------------
    # ATOMIC WRITE + NON-SKIPPABLE VALIDATION.
    # The file is written to a TEMP path, validated there, and only promoted to
    # OUT_FILE if EVERY check passes. If validation fails, the temp file is
    # deleted and NO file exists at OUT_FILE, so there is nothing to hand to the
    # user. This makes validate() impossible to skip or work around: a failing
    # build produces no deliverable, period. Do NOT edit this to write OUT_FILE
    # directly, and do NOT wrap validate() in a try/except that swallows the
    # error and ships the file anyway. If a check fires, fix the INPUT (re-read
    # the SBC), not the check. (See known_issues.md #8 — a build once shipped an
    # $8,300 deductible because validation was bypassed instead of honored.)
    os.makedirs(os.path.dirname(OUT_FILE), exist_ok=True)
    tmp_file = OUT_FILE + ".tmp"
    if os.path.exists(OUT_FILE):
        os.remove(OUT_FILE)   # clear any stale prior output before we start
    with open(tmp_file, "w", encoding="utf-8", newline="") as f:
        f.write(xml_str)

    try:
        validate(tmp_file, ref_root_copy, PLAN_DICTS)
    except AssertionError as e:
        os.remove(tmp_file)   # destroy the bad output so it can't be delivered
        raise SystemExit(
            "\n*** BUILD FAILED VALIDATION - NO FILE WRITTEN ***\n"
            f"{e}\n"
            "The output was deleted on purpose. Fix the plan values above "
            "(re-read the SBC — this is almost always an extraction error, not "
            "a check that needs relaxing), then re-run. Do not deliver any XML "
            "until this build prints 'All checks passed'."
        )

    os.replace(tmp_file, OUT_FILE)   # promote temp -> final only on success

    print(f"Wrote {OUT_FILE}  ({os.path.getsize(OUT_FILE)} bytes)\n")
    for pd in PLAN_DICTS:
        is_emb = pd["embedded_deductible"]
        emb = ("embedded (EmbededDeductible=N)" if is_emb
               else "aggregate (EmbededDeductible=Y)")
        print(f"{pd['scenario_name']} [{pd['plan_type']}, {emb}]")
        print(f"   Tier1:  ${pd['in_net_ded_ind']}/${pd['in_net_ded_fam']} ded, "
              f"${pd['in_net_moop_ind']}/${pd['in_net_moop_fam']} MOOP, plan pays {int(pd['in_net_plan_pays']*100)}%")
        if "t2_ded_ind" in pd:
            print(f"   Tier2:  ${pd['t2_ded_ind']}/${pd['t2_ded_fam']} ded, "
                  f"${pd['t2_moop_ind']}/${pd['t2_moop_fam']} MOOP, plan pays {int(pd['t2_plan_pays']*100)}%")
        print(f"   OON:    ${pd['oon_ded_ind']}/${pd['oon_ded_fam']} ded, "
              f"${pd['oon_moop_ind']}/${pd['oon_moop_fam']} MOOP, plan pays {int(pd['oon_plan_pays']*100)}%")

    print_coverage_table(OUT_FILE)

    # TREATED AS A READ - facility rows (v4.0). Carried into the output
    # presentation verbatim, so the operator sees what was decided without
    # having been asked; MANUAL items go in the set-manually-in-Claros list.
    print("\n=== TREATED AS A READ: FACILITY ROWS ===")
    for nm, read_, manual_ in facility_reports:
        if not read_:
            continue
        print(f"{nm}:")
        for line in read_:
            print(f"   {line}")
        for line in manual_:
            print(f"   MANUAL IN CLAROS: {line}")

    # DISPLAY NOTE - OP group (v4.0, builder decision). Claros's own convention:
    # the OP dropdown row shows the tier coinsurance while individual OP copays
    # live on their service rows. Printed whenever any copay-style scenario
    # carries an OP-group copay, so the reviewer who checks the display is told
    # what the display does not show. Fixed wording - copy it verbatim.
    op_seqs = {s_ for s_ in ("307", "308", "309", "311", "312")}
    op_copay_plans = [pd_["scenario_name"] for pd_ in PLAN_DICTS
                      if pd_["plan_type"] in COPAY_PLAN_TYPES and (
                          op_seqs & set(pd_.get("t1_flat_copays", {}))
                          or op_seqs & set(pd_.get("cross_tier_copays", {}))
                          or op_seqs & set(pd_.get("t2_flat_copays", {})))]
    if op_copay_plans:
        print("\n=== DISPLAY NOTE: OUTPATIENT (OP) ROW ===")
        print("In Claros, the Outpatient (OP) dropdown row shows the tier coinsurance. "
              "The OP copays (ER, urgent care, outpatient surgery, outpatient other) are "
              "loaded on their individual service rows inside that dropdown, which is "
              "where Claros calculates them. A coinsurance value on the OP row is "
              "expected and is not an error.")
        print("Applies to: " + ", ".join(op_copay_plans))


if __name__ == "__main__":
    main()
```
~~~~~
<!-- END EMBEDDED PYTHON -->
