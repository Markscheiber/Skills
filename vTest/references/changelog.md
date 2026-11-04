# sbc-to-clarosv5 changelog

Version history for maintainers. Nothing in this file is needed to build a Claros XML; the build rules live in SKILL.md and the other reference files. The "Review decisions (accepted, do not re-flag)" block lives in SKILL.md.

**v5.1-test port of v5.1 (crosswalk + PP display fix) — November 4, 2026**

The v5.1 changes made to `sbc-to-clarosv5` (commit d5a2977), merged into vTest by a three-way merge from v5.0. vTest's own evaluator changes (runner extraction, fixed question rounds, Treated-as-a-read questions 3 and 8, open items list) are kept; the v5.1 content is identical to the v5 folder. The open items list drops Copay1 vs Copay2 and narrows the inpatient basis item to Per Day only.


Driven by Mark Scheiber's pinned Claros export on the AZ Blue 2027 build (PPO 5000 80 scenario): a distinct copay hand-keyed on every named line of the Tier Copays screen, exported, and read back.

- **Full medical crosswalk verified.** Every medical sequence now has a confirmed Claros name (field_mapping.md, build-script `SERVICE_NAMES`): IP 102 Medical / 105 Surgical / 101 Maternity / 103 MHSA / 104 NICU; RAD 601 Complex OP Hospital / 602 Complex OP Imaging Center / 604 Routine; LP 201; PH 511 Special Visit, 503 Consultation, 506 Maternity, 514 Therapy, 507 Mental Health, 512 Substance Abuse, 502 Chiro, 501 Anesthesia, 504 Immunizations, 510 Preventive, 513 Surgery, 508 Other; O 404 DME, 407 Rx Non-PBM, 402 Ambulance, 405 Home Health, 403 Dialysis, 408 SNF, 406 Hospice, 401 All Other.
  *Reason:* the Weak and Names-unknown rows forced every imaging, rehab, chiro, DME, ambulance and hospice line into the set-manually list (on AZ Blue, a $300 CT/MRI copay could not be placed).
- **Stop rule narrowed (doc + inventory guidance).** Named rows are placed directly via `t1_flat_copays` / `t1_service_coins` / `cross_tier_copays`. Only lines the XML cannot express on one row still stop: place-of-service splits on one SBC line, day-band benefits, telehealth, OON-only exceptions on 2-tier builds. Facility rows still go only through the facility read (one value on all five IP rows, unchanged).
- **PH fill unchanged, now overridable by name (builder decision).** The PH rows still default to the specialist copay; question 9 now tells the session to walk them by name and override any the SBC prices differently (therapy, anesthesia, physician surgery).
- **PP display row fixed (code change, known_issues.md #20).** Accordion `PP` is the Physician Other rollup, not PCP. `MEDTYPE_TO_SEQ` changed to `{"PP": "508", "SP": "511"}`; new `mirror_accordion_from_element()` writes PP/SP from the elements actually written (Tier 1 and Tier 2); check 8 compares against the same sequences.
- **Coverage table prints names** beside every sequence, so the post-build SBC walk is done by service name.
- **Also verified by the export:** the Copay1 field family (`Copay1Flag`/`Copay1Min`, Copay2 off) and Per Admit = `IPCopayPerDay="A"` with `IPDayAdmit="A"`.
- **Example fix:** the `t1_service_coins` examples used `403` as DME; 403 is Dialysis, DME is `404`.
- **Regression:** the AZ Blue four-plan build (3 PPOs + HSA HDHP, 2-tier) passes all checks and differs from v5.0 only on the three T1 PP rows; named-row overrides (RAD 601/602 copay, PH 514/501/513/508 coinsurance, hospice 406 covered in full), 3-tier Pattern B with an inactive-tier HDHP, and 3-tier Pattern A steerage all pass; a PP row tampered back to the PCP copay fails check 8.
- **Possible follow-up (not done):** with IP subtypes now named, the facility read could place the SBC's mental-health-inpatient row on 103 and childbirth on 101 instead of asking F1. Left as a builder decision.

**v5.1-test (vTest) — November 4, 2026**

Test build applying the November 4, 2026 AI Quality Evaluator findings on v5.0, for side-by-side comparison. Renamed to `sbc-to-clarosvtest`, triggered only by explicit invocation so it never competes with v5. No build logic changed: the example plans build byte-identically to v5.

- **Script extraction by one command, run through `run_build.py`.** Sessions no longer read the 128 KB build_script.md or retype the 2,250-line script; they extract it with awk and set only `REF_FILE`, `OUT_FILE`, the dates and `PLAN_DICTS` from a small runner. *Reason:* "write verbatim" invited reading and retyping the whole script, the largest per-build token cost and a transcription risk.
- **"Adapt it as needed" removed.** The script is never edited; a design the plan dict cannot express stops the build and is reported. *Reason:* editing script logic was the root cause of Run A.
- **Fixed question rounds.** The tier decision and questions 1, 2, 4, 5, 6, 7 and 9 go in four fixed rounds sized to the tool's 3-question limit; questions 3 and 8 moved under "Treated as a read". *Reason:* no batching order existed (open v4 finding D5).
- **Inpatient Per Admit / Per Day flagged on every inpatient-copay build.** *Reason:* the code mapping is unverified (known_issues.md #19), the same class of risk as the embedded flag.
- **Multi-tier body section cut to a pointer; placeholder table moved to field_mapping.md.** *Reason:* the body described the multi-tier patterns three times.
- **"Known open items" list added to known_issues.md.** *Reason:* open status lived only in the changelog, which builds do not read.
- **Stale text fixed:** script line count, "all twelve checks", the v3 trigger name, the Rx SubPlan wording, and the v3.1 history sentence that contradicted N = embedded.
- **Not applied:** trimming incident narratives out of known_issues.md (optional, and rewriting rule text there risks dropping a rule).

**v5.0 — November 4, 2026**

- **Embedded deductible flag inverted to match Claros.** Claros encodes `EmbededDeductible="N"` as embedded and `"Y"` as aggregate. v4 and earlier wrote `"Y"` for embedded, so every embedded plan loaded into Claros as aggregate and vice versa.
  *Reason:* verified on the Augustana University 2027 build. The v4 file wrote `"Y"` on an embedded PPO and an aggregate HDHP; the operator corrected both in the Claros UI and the re-export showed PPO `"N"`, HDHP `"Y"`. Plan dicts are unchanged (`embedded_deductible=True` still means embedded); only the written letter flipped. The build printout now says "embedded" or "aggregate" instead of the raw letter.
- **field_mapping.md and known_issues.md #6 corrected** to state the flag is read from each SBC's family-deductible wording (never by plan type) and to use the verified letters.
- Asset search paths point at `sbc-to-clarosv5`.
- **Follow-up cleanup (same day):** removed leftover v4 wording that still described the old letters. field_mapping.md's plan-type cheat sheet (PPO `N` / HDHP `Y`) now points to the SBC read rule with the verified letters; clarifying question 7 no longer treats a hand-built `N` on an embedded plan as a practice difference (it was correct); the Rx SubPlan `"N"` in the build script is documented as having no effect at a $0 Rx deductible. No code behavior changed.
- **HDHP embedded rule corrected (doc only).** Question 7, known_issues.md #6, field_mapping.md and the script comment claimed federal law requires an embedded deductible on HDHPs over the ACA cap. The ACA embedding rule covers the out-of-pocket maximum, not the deductible, so aggregate HDHPs (like Augustana's) are compliant and must be read from the SBC. The stale $9,200 figure was removed rather than updated, so there is no annual constant to maintain.
- **`embedded_deductible` now required on HDHPs too (code change).** HDHPs used to default to embedded when the key was omitted, so a session that forgot it on an aggregate HDHP (like Augustana's) shipped it embedded with no error. The build now raises the same KeyError for every plan type. Plan dicts that already set the key build byte-identically.
- **HDHP coinsurance line corrected (doc only).** The service-level section said HDHPs get `CoInsRate="0.80"`; the build uses the HDHP's own `in_net_plan_pays`, as question 8 requires.
- **Changelog moved out of SKILL.md** into this file (`references/changelog.md`), cutting about 15 KB (roughly 3,500 to 4,000 tokens) from every build's context. The "Review decisions (accepted, do not re-flag)" block stays in SKILL.md so the AI Quality Evaluator still sees it.

**v4.0-beta — October 20, 2026**

Driven by Ben Choate's (AI CoE) three-run consistency findings, October 14, 2026: identical BCBSOK SBCs and identical intake answers produced three different XML files (222 attribute differences in five clusters). Each fix below is regression-tested against v3 on synthetic fixtures: designs v3 could already express build byte-identically in v4.

Status of this beta: findings D1 and D2 are fixed. Still open and unchanged from v3: D3 (OON ER copay carry), D4 (option text behind the same letter) and D5 (question order and composition drift). The Per Admit = "A" / Per Day = "D" mapping is pending the BASIS TEST import (known_issues.md #19).

- **Fix 1 (finding D1): facility rows come from `facility_lines` and a fixed in-build rule.** New required plan-dict key on PPO/HMO/EPO; `resolve_facility_read()` places IP `101`-`105`, OP `309`/`311` and OP `308`; pinned question F1 when the inpatient rows disagree; new hard-fail check 12; new "Treated as a read" heading; crosswalk confidence levels now state what each permits; known_issues.md #18.
  *Reason:* 162 of 222 differences were sessions deciding differently whether "Client fill" sequences were placeable, and improvising a home for inpatient copays on 2-tier plans. One session's improvisation (IP copays in `t1_flat_copays`) also produced the display/calc mismatch of finding D2, which v3 validation passed; v4 refuses that input before building. Builder decisions (October 2026): place when one value covers the group; place a source-labeled hospital/ASC split; flag an unlabeled split; the operator always decides an inpatient disagreement; OP Other (`308`) carries OP Surgery - Hospital's value unless the source names an OP Other line. That last decision is a deliberate change from v3, where `308` stayed at tier default coinsurance on plans with no explicit line.
- **Fix 2 (finding D2): the display row and the calculation rows can no longer disagree on inpatient.** One mapping (`IP_BASIS_CODES`: Per Admit = `"A"`, Per Day = `"D"`) feeds both writers, which now require the code as an argument instead of each hard-coding `"D"`; inpatient copay lines carry a transcribed `basis` (`not_stated` builds Per Admit and is flagged); check 8 drops the copay exemption for the IP group and compares copay, member share, plan pct, deductible checkboxes and `IPCopayPerDay`; new fixed OP display note in the output presentation; known_issues.md #19 with the BASIS TEST import procedure.
  *Reason:* in Run A the session patched one of two hard-coded `"D"` literals (display "D", calculation "A"); in Run B the IP copays sat under a coinsurance display row with `IPCopayPerDay="N"`; v3 check 8 never compared `IPCopayPerDay` and skipped copay-typed IP rows, so both files passed. Regression: designs v3 could express (CRMC per-day) build byte-identically; the BCBSOK per-admission design changes only `IPCopayPerDay` D to A, on both layers.

**v3.2 — October 8, 2026**

Driven by the national-rollout POC test log (11 real-client tests, 1 pass / 10 accuracy failures on v3.1 — tier detection and placeholder flagging were 11/11) and by reverse-engineering a verified hand-built 2021 client export (GAP/MOU) against its SBC. The rebuilt v3.2 output agrees with that verified export on 33 of 35 Tier 1 elements; the two divergences are the preventive rows, where v3.2 follows the SBC ("No charge") and the hand build had blanket-filled the office copay.

- **Preventive care (sequences 504/510) covered in full, in-network, every plan type — with hard-fail check 11 behind it.**
  *Reason:* the blank export ships these rows correct and the v3.1 tier loops overwrote them with tier coinsurance; every check validated structure, not benefit content, so the wrong file passed all ten checks in 10 of 10 POC tests. See known_issues.md #15.
- **Copay+coinsurance encoding corrected: member share to `Copay1Perc`, `CoInsRate` stays `1`** (PlanElement), member share to `ParticipantPct` with plan pct 100 (accordion); check 8 now verifies the ParticipantPct mirror.
  *Reason:* the verified export's ER row proves the convention; v3.1 wrote plan-pays into `CoInsRate` and forced the deductible on — a materially different actuarial input. Also resolves POC test 11: the specialty-Rx "Participant %" display testers reported is correct behavior, now explained preemptively in the output presentation. See known_issues.md #16.
- **The office copay fills the entire `PH` grouping** (12 rows beyond PCP/Specialist; preventive carved out first; `ph_other_copay` / `t2_ph_other_copay` overrides; scalar-0 office copays fall to coinsurance instead of modeling free care).
  *Reason:* confirmed from the verified export; v3.1's two-row fill is what testers reported as "Specialist copays did not fill in the drop down / Physician other."
- **`t1_flat_copays` reframed as a first-class 2-tier field** in SKILL.md, the script's field docs, and both example dicts (ER 307 / urgent care 312 shown populated); the inline comment claiming "simple 2-tier plans omit them" rewritten to say the opposite.
  *Reason:* that framing produced the missing ER/UC/chiro/rehab copays in six of ten POC failures (tester finding S4).
- **`t1_service_coins` added**: per-sequence coinsurance overrides, including the deductible-waived form (`set_planelement_coins` gained `after_ded`); check 8 verifies overrides against their declared values.
  *Reason:* "X% Cost Sharing, Deductible does not apply" and off-default rates (DME at 50%) were inexpressible — silent, actuarial, uncaught (tester findings S2/S3, POC test 6).
- **`embedded_deductible` is a required explicit input for PPO/HMO/EPO** (KeyError otherwise), HDHP defaults embedded but is overridable; clarifying question 7 rewritten for all plan types with the SBC read rule.
  *Reason:* v3.1 hardcoded non-HDHPs aggregate; POC test 6 built two embedded PPOs wrong.
- **Copay inventory added as clarifying question 9, and a post-pass SERVICE COVERAGE table added to the build output** for line-by-line SBC comparison before delivery.
  *Reason:* service-level extraction was the one decision with no prompt and no check behind it (tester findings S5/S9).
- **New clarifying-questions preamble rule: `ask_user_input_v0` ends the turn**; all questions go inside the tool call, and an unanswered gating question stops the build.
  *Reason:* tester findings E2/E3 — prose questions before the tool call were unanswerable, and a build proceeded past an unanswered gate.
- **HDHP coinsurance guard (b) removed** (HDHP OON matching a PPO's OON rate); guard (a) retained.
  *Reason:* one carrier with one OON schedule is the normal case; it fired on a correct file (tester finding S7). The error it targeted still trips guard (a).
- **Non-calendar plan years: `ManualForDataYear` = plan year START year** (builder decision), documented in question 2 and field_mapping.md.
  *Reason:* no rule existed (tester finding S8).
- **`tier_split` now ships with a proposed starting default** (e.g. 95/0/5 from the verified export) instead of bare placeholder values, still flagged for the user to set in Claros.
- **field_mapping.md crosswalk rebuilt with a confidence column**: 307/312/509/511 and the full `RX_SEQ_MAP` now verified from two independent sources; 504/510 identified as the preventive pair; `PH` others / `O` 401-408 / the `RAD` split marked unconfirmed with an unmapped-copay STOP rule (tell the user, never guess) and a 10-minute distinct-values procedure to pin the remaining names.
- **known_issues.md #15/#16/#17 added** (preventive overwrite; copay+coins encoding; separate-Rx-deductible convention unverified — flag for in-Claros confirmation), #12 updated for the guard removal.
- Fixed a latent mirror bug: an explicit `t1_flat_copays["509"/"511"]` override updated the PlanElement while the PP/SP accordion rows kept reading `pcp_copay`/`specialist_copay`; both now read the same resolved value.

**Regression suite (run before this release, all 34 assertions passing):** ASC PPO 2021 rebuilt from its SBC and diffed against the verified hand-built export; HDHP sibling scenario; missing-`embedded_deductible` refusal; simulated v3.1 preventive overwrite killed by check 11 with no deliverable; 3-tier Pattern B build with Tier 2 preventive and PH fill; `t1_service_coins` overrides passing check 8. Re-run equivalents when touching the tier loops or validate().

**v3.1 — September 1, 2026**

- **The reference choice is now a confirmed decision on every build**, promoted out of conditional clarifying question 6 into its own "The tier decision" section ahead of all other questions, with a detection guide and a script-side comment on `REF_FILE` telling you not to leave it on the default.
  *Reason:* the 2-tier/3-tier choice was only surfaced when the SBC "shows more than two network levels," which puts the trigger on the same reading step it is meant to protect. A missed sub-network split reads as an ordinary plan, gets built on the 2-tier reference, and the middle tier disappears with no check that would catch it.
- **Bundled assets now resolve through a search list** (`_SKILL_ASSET_DIRS`) covering `/mnt/skills/plugins/`, `/mnt/skills/user/`, an `assets/` folder beside the script, and a bare `assets/`, and `load_reference()` reports the full list when nothing matches.
  *Reason:* `_asset()` hardcoded a single install root. The skill has since shipped from a second one, so the default build could die at `load_reference()` before any validation ran, with an error that read like an environment problem rather than a skill problem.
- **Extraction step 4 in `build_script.md` corrected.** It named `assets/blank_reference.xml`, a file that has not existed since the two-reference split, and is now a table of both references with the constant to set for each.
  *Reason:* it was the first instruction in the file carrying the build logic, and it named the wrong file at the exact point the reader chooses a reference.
- **Dangling identifier fixed:** `known_issues.md` pointed at `SIG_ATTR_EXEMPT`; the constant is `SIG_ATTR_ADD_EXEMPT`.
- **Stale line count corrected** (~900 to ~1,400).
- **Documented check list renumbered one-to-one with `validate()`.** The old list summarized ten checks as eight, folded namespaces and CRLF into one "Format check," omitted the Rx deductible-application count check entirely, and the body's cross-references ("check 8 verifies the mirror," "hard-fails check 10") used the script's numbering against the summarized list, so they resolved to the wrong entries. The plan-design printout was also mislabeled as a check; it runs after a passing build and is now documented as the final human confirmation step.
  *Reason:* when a check fires, its number in the traceback is the reader's only handle for looking it up, and that lookup silently landed on the wrong entry.
- **Dead flag instruction removed from the script.** A comment block above `PLAN_DICTS` still told the reader to "set True" on `MAIL_SPECIALTY_NOT_COVERED`, a flag deliberately removed because no code ever read it. The comment now states the actual behavior: mirror the SBC's specialty value to both tiers and flag "mark not covered in Claros" in the placeholder summary.
- **known_issues #14 updated.** It predated both the bundled 3-tier reference (claimed the bundled reference was 2-tier only) and Pattern B. Now names `blank_reference_3tier.xml`, points Pattern B at `multi_tier.md` section 10, and references "The tier decision."
- **Standalone spot-check snippet fenced.** The 3-check snippet in `known_issues.md` now opens by stating it is not a substitute for a build's validation and must never be used to clear a file this skill produced.
- **"Separate files per plan" removed from clarifying question 1.** The output is always one file with every plan as a scenario. The old question offered an option nothing implemented; the question now confirms which plans go in the file and their order.
  *Reason:* builder decision — one file is always the output for Claros uploads.

**v3.0**

- Split from v2 with the 3-tier reference (`assets/blank_reference_3tier.xml`) bundled and Pattern B sub-network copay-split logic baked in, verified on the Zoroco Packaging build: `t2_pcp_copay` / `t2_specialist_copay` / `t2_flat_copays`, `cross_tier_copays` for copays that survive OON, and `tier2_active` / `tier_split` for single-in-network plans built inside a 3-tier file.
- Validation restructured into a hard gate: atomic temp-file write, `validate()`, promote to `OUT_FILE` only on pass, so a failing build leaves no deliverable.
- Coverage gate added so a client-supplied shell containing a grouping, SubPlan, or Rx element the tier loops never visit hard-fails instead of shipping shell template values.
- HDHP rules enforced in code rather than by instruction: HDHP embedded default set by `plan_type` (v5 corrected the letter: embedded writes `"N"`), all Rx tiers zeroed and set deductible-applicable, deductible-equals-MOOP derived to 100% plan-pays.
- `MAIL_SPECIALTY_NOT_COVERED` removed rather than left as a documented flag no code read.

**v2 and earlier**

- `sbc-to-clarosv2`, `sbc-to-xml-claros`, `testingversionsbc-to-claros`. All superseded; see the frontmatter supersession clause.
