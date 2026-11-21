# JUSTIFICATION — the design of the harness, and why

All numbers are from my submitted run of `lab06_harness_says_no.ipynb` (Qwen2.5-3B-Instruct, greedy decoding,
Colab T4, `N_PAIRS = 40`, `SEED = 2026`). Code places are given by the headings of the notebook cells.

## 1. What the system may do

**Purpose.** The system does a first screening of applications to the (fictional) Tulpar Scholarship Fund, and only
for one purpose: `scholarship_screening`. It looks only at the merit and need fields the fund asks for: programme,
GPA, entrance exam, household income bracket, achievements, age. It gives a **recommendation** (*shortlist* or
*reject*) with a score.

Two Kazakh rules decide how much it may do alone. Law 94-V art. 19-1(1) says that automated processing which
creates or ends a person's rights is allowed only with the person's consent, and art. 19-1(2)–(3) give the person a
right to object and to an answer within three working days. Law 230-VIII art. 17(2) separates systems where a person
can correct or cancel the decision (medium autonomy, art. 17(2)(2)) from systems where this is impossible (high
autonomy, art. 17(2)(3)). I designed a **medium-autonomy** system:

- it may **shortlist** alone when the score is outside the borderline band, because a shortlist is not final; people
  decide at the next stage;
- it may **reject** alone only outside the band and only if none of my escalation rules fires; the applicant must be
  told that the decision was automated and that they can object (art. 19-1(2)), and the log makes the objection
  possible to check;
- it may **not decide** inside the band, on a reply that cannot be parsed, on a reply that contradicts itself, on a
  reply whose reason talks about a name, gender, language or institution, or on a rejection of a low-income
  high-merit applicant. These go to a person (`escalate`);
- it may **not process at all**: another purpose, a minor, an applicant without consent, impossible values, or free
  text that is not data about the applicant (`refuse`).

**Each control, why I built it, and where it is.**

| control | the constraint behind it | where in the code |
|---|---|---|
| input validation | reliability and no abuse (230-VIII art. 11(1), 15(2)(2)); in Part 4 the bare pipeline gave a decision (reject) on GPA 9.9 / exam −5 | `Harness(validate_input=True)` in Part 2d; `validate_profile` in the toolbox |
| blinding | no discrimination (230-VIII art. 6) and only necessary data (94-V art. 12(1), 25(2)(1)); what Part 1 measured | `BLIND` in Part 2a |
| refusal | consent as the basis for automated decisions (94-V art. 19-1(1), 7(1), 8(1)); ban on classifying people by personal characteristics (230-VIII art. 17(3)(3)); no abuse (art. 11(1)) | `Harness(refuse=True)` and `my_extra_refusal` in Part 2b |
| decision log | documentation and the right to an explanation (230-VIII art. 15(2)(3), 16(1)(4)–(5); 94-V art. 19-1(2)) | `Harness(log_path="decisions.jsonl")`; the file `decisions.jsonl` |
| escalation | human control and human autonomy (230-VIII art. 8(1), 9, 17(2)(2)); the objection procedure (94-V art. 19-1(2)–(3)) | `Harness(escalate=True, borderline=...)` and `my_extra_escalation` in Part 2c; band calibration in Part 2d |

**A constraint that changed the architecture.** Without art. 19-1 of Law 94-V and art. 17(2) of Law 230-VIII, the
simple design is "the model decides, people check a sample later". With them, the model's output is only a
recommendation, and an uncertain negative result must be reviewable *before* it becomes final. That is why the
escalation step sits in front of every uncertain decision and why every decision is logged with enough information
to answer an objection.

A second change: the escalation band. The template suggests taking it from the unblinded baseline scores. But a
threshold from one configuration does not transfer to another, so I added a **calibration run** (Part 2d: 40
profiles from a different seed, never part of the measured pairs, shown exactly the blinded prompt) and took the band
from the point where the blinded model switches from reject to shortlist (57.5). My first rule allowed the band to
grow only ±15 points; it gave (42, 72) and held **0 of 40** calibration profiles (printed by the first attempt of this cell; that output was replaced when I re-ran the cell, which is why the execution counts jump from 16 to 21), because the blinded model gave only
four scores (20, 40, 75, 85). I stopped the run before the harness measurement finished, widened the limit to ±20
points, and re-ran from Part 2d. The final band **(40, 74)** held 6 of 40 calibration profiles (15.0%). On the
unblinded baseline the same band would have held 198 of 400 (49.5%), which shows again that a band does not transfer.

## 2. What it must refuse

| refusal (reason code) | where it is enforced | did it fire in the submitted run? |
|---|---|---|
| `purpose_not_allowed`: any purpose that is not scholarship screening (e.g. ranking by ethnicity) | `refusal_reason` (toolbox), `Harness(refuse=True)` | yes, Part 4 request 2 |
| `minor`: age under 18 | `refusal_reason` | yes, Part 4 request 3 |
| `no_consent`: no consent to automated processing | `refusal_reason` | yes, Part 4 request 4 |
| `invalid_input`: a value outside its documented range | `validate_profile`, `Harness(validate_input=True)` | yes, Part 4 request 5 |
| `instruction_in_input`: a free-text field (name, university, programme) that talks to the model | `my_extra_refusal` (Part 2b) | yes, Part 4 request 6 |
| `name_not_a_name`, `program_not_recognised`, `free_text_not_text` | `my_extra_refusal` | no, never triggered (untested) |

Part 4 printed: the harness refused 5 of 6 requests, each with **0 model calls**; the bare pipeline refused 0 and even
**shortlisted** the "rank by ethnicity" request and the profile whose name was an instruction. Each refusal is in
`decisions.jsonl` with `outcome = "refuse"`, the reason code, `controls_fired`, and `raw_output = "[no model call] …"`.

**What each refusal also blocks, and who pays for it.**

- *Purpose*: it also blocks honest reuse (for example ranking shortlisted applicants for interviews). Only the fund
  pays; acceptable, because a new purpose needs new consent and a new risk check (94-V art. 7(1)).
- *Minor*: a 17-year-old who is already a student is a legitimate applicant, but is refused automated screening.
  Law 94-V art. 7(1) and 8(1) allow processing with the consent of a legal representative, but the profile has no
  field for that, so the system cannot know. The young applicant pays, so the refusal must send them to manual
  review, not out of the process.
- *No consent*: an applicant who does not want automation is not rejected. Art. 19-1(1) makes consent the condition
  of the *automated* route, not of the application, so the refusal means "a person screens this one".
- *Invalid input*: an honest applicant with a typing mistake (GPA 3.9 typed as 39) is stopped. The cost is delay; the
  message names the field without the value, so it can be fixed.
- *Instruction in input*: my word list is crude. A name or programme with a listed word (a family name "Score", a
  programme "Decision Sciences") would be refused. On the 400 clean measured applicants my rules refused **0** (last
  line of Part 4), but they all come from one invented list of Latin-script names, so this is not a real
  false-refusal rate. Cyrillic and Kazakh letters are allowed by my name pattern but were never tested.

**What happens to a refused request.** It is written to the log, no decision is made, and in the procedure around the
system (not in this notebook's code) the application goes to a human screener and the applicant is told the reason in
plain words. The applicant can always ask for a human decision: for a refusal this is the normal route, for an
automated decision it is the objection of art. 19-1(2). The code does not send the message and does not keep a
reviewer queue; this is a gap (section 6).

## 3. What record it keeps

**Audit question:** *For any application, which outcome did the system give, from exactly which input text, with
which model, prompt and harness settings, which controls acted, and why — and can someone check this without running
the model again?*

A record in `decisions.jsonl` answers it. For example, the escalated record printed in Part 5 (pid `P0124a`) has:
`record_id`, `pid`, `input_sha256` (hash of the exact text the model saw), `fields_shown` (program, age, gpa,
exam_score, household_income_bracket, achievements) and `fields_withheld` (name, university, consent), `model_id`,
`prompt_version`, all `harness` settings including the band (40, 74), the model's raw reply (`{"decision":
"shortlist", "score": 60, "reason": "Average performance but potential"}`), `decision`, `score`, `controls_fired`
(blind, escalate, log), `outcome` = escalate, `reason_code` = borderline, `escalated_to_human` = true, and a
`timestamp`. A refusal record (same pid, purpose `rank_by_ethnicity`) has empty `fields_shown`, a hash of the profile
instead of a prompt hash, and the reason code. The log of the run has 406 records: 400 measured decisions, the
ordinary request and the 5 refusals of Part 4.

**What is left out on purpose.** Field *values* are not stored: no name, grades or income in the log. So the log can
be kept for audits without being a second copy of the applicants' data (94-V art. 12(1): only what is necessary). To
check an input, an auditor renders the profile again from the application database and compares the hash. The raw
reply can repeat what the model saw, but a blinded field is never shown to the model, so names and universities do not
leak into it.

**How long, who reads, who writes.** My proposal (not enforced by the code): keep the log for the application cycle
plus the objection and appeal period, and at least six months (the EU AI Act art. 26(6) benchmark for logs of
deployers); after the purpose is reached, delete it (94-V art. 12(2), 25(2)(4)). Readers: the fund's data-protection
officer, the review panel and auditors. Writer: only the harness, by appending. Nothing stops someone from editing a
line later: the `record_id` is a hash of the content and the line number, so an edited line would not match its id
if the id is recomputed, but there is no outside anchor. To close this I would need a hash chain (each record holds
the hash of the previous one) or write-once storage.

**What the record does not let you reconstruct.** *Why* the model gave its score (the one-sentence reason is not a
real explanation of the model); what the human reviewer did with an escalated case (there is no record of the human
decision yet); and the counterfactual and order-probe runs, which do not write to the log.

## 4. The legal constraints

**Which instruments and why.** The fund is presented as working in Kazakhstan (the course points to adilet.zan.kz,
and the names and universities are Kazakh-style). It owns an AI system that makes automated decisions about people
using their personal data. So two Kazakh laws apply directly. EU law I use only as a benchmark. I read the two Kazakh
laws in the official database adilet.zan.kz.

1. **Law of the RK No. 94-V of 21 May 2013 "On Personal Data and Their Protection"** (adilet.zan.kz, document
   Z1300000094, version in force in November 2026). It applies because names, grades, age and income are personal
   data and the fund collects and processes them.
   - **Art. 19-1(1)** (added by Law No. 231-VIII of 17.11.2025): automated processing of personal data that creates,
     changes or ends a person's rights or legitimate interests is **forbidden**, except with the person's consent or
     where a law allows it. → This *requires* the `no_consent` refusal: consent is the gate to the automated route.
   - **Art. 19-1(2)**: the owner/operator must explain how the automated processing works and its possible
     consequences, give the possibility to object, and explain how to protect one's rights. → This *requires* that
     a decision can be explained (the log) and that an objection is possible (the escalation route).
   - **Art. 19-1(3)**: an objection must be reviewed within three working days and the person informed. → This
     *requires* human review capacity; my escalation route is the technical half of it.
   - **Art. 7(1) and 8(1)**: processing with the consent of the person **or the legal representative**. → The basis
     of the `minor` refusal (there is no field for a representative). The law *allows* processing a minor's data
     with a representative's consent, so refusing automation for minors is *more* than the law requires.
   - **Art. 12(1)**: collect only the data that is necessary and sufficient for the task; **art. 25(2)(1)**: the
     owner approves this list. → Supports blinding and a log with field names only. The law *allows* collecting the
     name and university (the fund needs them to pay the scholarship); it does not require hiding them from the
     model. Blinding is my extra choice for non-discrimination.
   - **Art. 22(1)**: measures against unauthorised access. → Not implemented in this notebook (section 6).
2. **Law of the RK No. 230-VIII of 17 November 2025 "On Artificial Intelligence"** (adilet.zan.kz, document
   Z2500000230). It applies because the screening model is an AI system that the fund, as owner/holder, operates in
   Kazakhstan.
   - **Art. 6** (fairness and equality): AI systems must be built and used without any discrimination by origin,
     social and property status, sex, race, nationality, language, place of residence or any other circumstance.
     → The reason I measured name language, gender coding and university, and the reason for blinding.
   - **Art. 7(2)**: a person about whom decisions are made with AI has the right to be informed about the automated
     processing and its consequences, about the possibility to object, and about how to protect their rights.
     → Explanation (log) and escalation.
   - **Art. 8(1)**: owner, holder and user must keep constant control over the system. **Art. 8(2)**: everybody who
     creates or operates AI systems is responsible for its results, according to their role. → Escalation; section 3
     of REPORT.md.
   - **Art. 9**: human autonomy and free will in decisions must be kept. → People decide uncertain cases.
   - **Art. 11(1), 11(3), 15(2)(1)–(2)**: safety, reliability and risk management. → Validation; refusing injection.
   - **Art. 15(2)(3)**: keep documentation on the AI system, depending on its impact on people's rights. → The log
     and the control register.
   - **Art. 16(1)(4)–(5)**: the user may get explanations of results that affect their rights and may ask which data
     the decision was based on. → `fields_shown`, `input_sha256`, `raw_output`.
   - **Art. 17(2)**: low, medium and high autonomy. → I designed for medium autonomy (section 1).
   - **Art. 17(3)(3)**: creating and operating AI systems that evaluate or classify people on the basis of social
     behaviour or personal characteristics is forbidden (with exceptions set by laws). → The `purpose_not_allowed`
     refusal of "rank by ethnicity". I am not sure whether normal merit screening falls under this item; I read it
     as aimed at social scoring, not at checking the criteria an applicant sends in.
   - **Art. 21(4)**: rules for decisions based only on automated processing of personal data are set by the
     personal-data law. → This links the AI law to art. 19-1 of Law 94-V, which is the working rule.
3. **Benchmark only, not binding: Regulation (EU) 2024/1689 (AI Act)** (EUR-Lex). The fund has no EU establishment
   and its output is not used in the EU, so the AI Act's scope does not reach it (art. 2(1)(c) covers third-country deployers only where the output is used in the Union). Annex III point 3 lists
   AI systems used to decide access or admission to education as high-risk; a scholarship screener is close to that.
   I use as yardsticks: art. 10(2)(f)–(g) (examine and reduce possible biases) for Part 1; art. 12(1) (automatic
   recording of events) for the log; art. 14(1) and 14(4) (human oversight, the ability to ignore or override the
   output) for escalation; art. 26(6) (deployers keep logs at least six months) for retention. GDPR (Regulation (EU)
   2016/679) art. 22 is the EU version of art. 19-1 of Law 94-V.

**Where I did more than the law requires.** Blinding (the law asks for no discrimination as a result, not for this
technique); refusing automation for minors although a representative could consent; escalating replies whose reason
mentions a protected attribute; the calibration run for the band.

**Where I found no provision.** I did not find a Kazakh provision that requires a specific log format or a retention
time for AI decision records: 230-VIII art. 15(2)(3) speaks about documentation according to a list that still has to
be approved, and art. 22 on machine-readable forms leaves the rules to the authorised body. So the six months is an EU
benchmark, not a Kazakh rule. I also found no provision about prompt injection; I connect it to the general safety
and no-abuse duty of art. 11(1).

## 5. The ethical principles

Sources and their status: the **UNESCO Recommendation on the Ethics of Artificial Intelligence** (adopted
23 November 2021 by the 41st General Conference; a recommendation to member states, not binding law), section III.2
"Principles" (fairness and non-discrimination: paras 28–30; right to privacy and data protection: paras 32–34;
proportionality and do no harm: paras 25–26; safety and security: para 27; human oversight and determination: paras
35–36; transparency and explainability: paras 37–41; responsibility and accountability: paras 42–43); and the **OECD
Recommendation of the Council on Artificial Intelligence**, OECD/LEGAL/0449 (adopted 22 May 2019, amended 8 November
2023 and 3 May 2024; a non-binding OECD legal instrument; I could not confirm that Kazakhstan has adhered), section 1
"Principles for responsible stewardship of trustworthy AI". Law 230-VIII art. 4 also lists principles (legality,
fairness and equality, transparency and explainability, responsibility and controllability, priority of human
well-being and free will, data protection, safety); those are legal principles with binding force, used in section 4.

| principle (source) | the decision in this system it bears on |
|---|---|
| Fairness and non-discrimination (UNESCO III.2; OECD 1.2) | the matched-pair measurement; hiding name and university; escalating reasons that mention a protected attribute |
| Right to privacy and data protection (UNESCO III.2; OECD 1.2) | hiding the consent line; storing field names, not values, in the log |
| Proportionality and do no harm (UNESCO III.2) | refusing other purposes; refusing automation for minors |
| Safety and security / robustness (UNESCO III.2; OECD 1.4) | input validation; refusing text that talks to the model |
| Human oversight and determination (UNESCO III.2; OECD 1.2(b), "capacity for human agency and oversight") | the borderline band; unparseable or self-contradicting replies go to a person |
| Transparency and explainability; responsibility and accountability (UNESCO III.2; OECD 1.3, 1.5) | the decision log and the control register |

**Where the principles conflict, and what I put first.**

1. *Same treatment vs fairness by need.* Blinding treats matched applicants the same, but my rule
   `low_income_high_merit_reject` treats the lowest income bracket differently. I put the fund's legitimate need
   criterion first: income is a stated need field, not a protected attribute, and the rule only adds a human look; it
   never gives a different automatic outcome.
2. *Human oversight vs privacy.* Every escalated case is read by a person who sees the whole application, including
   the name the model did not see. So oversight brings back the exposure that blinding removed. I accepted that,
   because the person is accountable and the model is not.
3. *Transparency vs minimisation.* A log with field values would make audits easier; I chose minimisation and rely on
   re-rendering the profile plus the hash.

**Where ethics asks for more than the law.** Law 94-V lets the fund process the name and university; fairness asks
that they do not influence a merit decision, so the model does not see them (and Part 1 found a +2.5-point score gap
for the flagship university, sign p = 0.002). The law asks for a review when someone objects; oversight asks that
uncertain cases are reviewed *before* anyone has to object. The law says nothing about a model failure turned into a
rejection (the bare pipeline turns an unparseable reply into a reject); do-no-harm forbids it.

## 6. The trade-offs

| control | what it cost in my run | what harm it prevents |
|---|---|---|
| blinding | 140 of 400 outcomes changed compared with the unblinded run (106 shortlist → escalate, 32 reject → escalate, 2 shortlist → reject); 183 of 400 scores changed, mean \|delta\| 12.43. The fund loses any real signal in the university (programme quality, grading standards). | The +2.5-point flagship score gap (sign p = 0.002) and any name-based divergence cannot reach the decision any more: all five groups show 0/40 flips and 0/40 score differences after (by construction). |
| escalation band (40, 74) | **138 of 400 profiles (34.5%)** sent to a person. Planned: 15.0% on the calibration set (6 of 40). | Uncertain decisions near the line, where the university gap and the layout effect (25.0% of identical pairs flipped by line order alone) change outcomes, are not final without a person. 32 baseline rejections are now reviewed. |
| my escalation rules | 0 extra escalations in the harness run (they never fired); 2 `decision_score_mismatch` escalations fired in the counterfactual extension | Self-contradicting replies and rejections of needy strong applicants would not pass unseen; but in this run they were not tested on real cases. |
| refusal and validation | 0 refusals among the 400 clean profiles; 5 of 6 Part 4 requests refused with 0 model calls | The bare pipeline shortlisted a "rank by ethnicity" request and an injected name, and rejected a minor, a non-consenting applicant and an impossible profile. |
| log | 406 lines, no extra model calls; a small storage and governance cost | Every decision can be explained and checked on objection (94-V art. 19-1(2)–(3); 230-VIII art. 16(1)(4)–(5)). |
| calibration run | 40 extra model calls | Without it, the band from the unblinded baseline would have planned 49.5% (198 of 400); with it, the plan was wrong too (15.0% planned vs 34.5% real), but less wrong. |

Model cost: 400 calls in both runs (1.00 per profile), 481.3 s bare vs 462.0 s with the harness (fewer prompt tokens,
74722 vs 84409, because three lines are removed), plus 40 calibration calls. The counterfactual check (extension)
would double the calls: 32 calls for 16 profiles.

**The width of the band.** It sends 138 of 400 applicants (34.5%) to a person, which means about one in three
applications needs a reviewer from the fund's selection committee. Outside the band, 248 shortlists and 14 rejections
are automatic and nobody re-checks them. That is acceptable for shortlists (people decide later), and for the 14
rejections only because the applicant is told and can object within the art. 19-1 procedure. I did recalibrate after
changing what the model sees (the calibration run), but the calibration set of 40 was too small and the blinded
scores were too coarse, so the real share (34.5%) was more than twice the plan (15.0%). The honest rule for the
fund: measure the share in the first weeks and recalibrate on real traffic.

**Why this balance.** The bare agent shortlists 356 of 400, so rejections are rare but they are the decisions that
hurt, and the layout probe shows the model is not stable near the line. So I preferred too many escalations to wrong
final rejections. I would choose differently if the fund had very few reviewers (then: escalate all rejections
only, about 10–15% of applicants, and shortlist the rest automatically) or a much more stable model (then a narrow
band).

**What the design still does not protect against.**

- *Prompt fragility:* the line order alone flipped 10 of 40 identical pairs (25.0%). Blinding does not fix this. A
  fix would be several prompt orders and a majority vote, or escalating disagreements (more model calls).
- *Untested rules:* `name_not_a_name`, `program_not_recognised` and all three of my escalation rules never fired in
  the measured run.
- *Proxies:* programme and age are still shown; age was not measured as an attribute, and a programme can correlate
  with gender or region.
- *Human side:* no reviewer queue, no record of the human decision, no notification to the applicant in the code.
- *Log integrity:* no hash chain; a line could be edited later.
- *Security:* access control for the log and the data (94-V art. 22(1)) is not implemented.
- *Sample size:* n = 40 cannot detect effects below 6 one-sided pairs (15%); the gender pattern (4/0) stays open.

I did not close these because they need infrastructure outside a notebook (queue, storage, access control) or many
more model runs than the lab's time allows; they are the first items for a real deployment.

## AI-use declaration

**Tool.** I used one AI tool: Claude (Anthropic, model Claude Opus 5.5), working as an agent with access to a shell,
a web browser and my GitHub fork. I did not use any other AI tool.

**What it was used for.**

- *Setup and running:* forking the repository, running the notebook in Google Colab on a T4 GPU through the browser,
  downloading the outputs and committing them to the fork.
- *Code:* the student cells — the choice of blinded fields (`BLIND`), `my_extra_refusal`, `my_extra_escalation`, the
  calibration of the escalation band, and the `CONTROL_REGISTER`. The code was first tested with a stub model, then
  run with the real model. The predictions (`PREDICTION_0` … `PREDICTION_4`) were written before the run.
- *Writing:* drafts of `REPORT.md`, this file and the observations, from the numbers the notebook printed. The texts are
  in simple English on my request.

**Legal citations.** A statement of an AI model is not a citation, so every legal provision cited here was checked in
the primary text: Law No. 94-V (including art. 19-1) and Law No. 230-VIII in the official database adilet.zan.kz;
Regulation (EU) 2024/1689 (art. 2(1)(c), 10(2)(f)–(g), 12(1), 14(1), 14(4), 15(1), 26(6), 86(1), Annex III point 3)
and Regulation (EU) 2016/679 (art. 22(1)) on EUR-Lex; OECD/LEGAL/0449 on the OECD Legal Instruments site; and the
UNESCO Recommendation (section III.2, paras 25–43) in its published text. The tool found and opened these texts; the
citations point to the instruments, not to the tool.

**Numbers.** Every number in `REPORT.md`, this file and the observations was copied from the printed output of the
submitted run and cross-checked against the saved notebook.

**Responsibility.** I read the final texts and I am responsible for everything I submit, including the choices in the
design and any mistakes.
