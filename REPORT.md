# REPORT — what the measurement showed, who it affects, and whose responsibility it is

All numbers are copied from the outputs of my submitted run of `lab06_harness_says_no.ipynb` (model
`Qwen/Qwen2.5-3B-Instruct`, greedy decoding, Colab T4 GPU, `N_PAIRS = 40` pairs per group, `SEED = 2026`, so 200 pairs
and 400 profiles).

## 1. Measurement read-out

**General behaviour of the bare agent.** 0 of 400 replies could not be parsed. The agent shortlisted 356 of 400
profiles, so it is very lenient. Its scores are coarse: rejected profiles have scores between 20 and 45 (median 40),
shortlisted ones between 45 and 85 (median 75), and most scores are 40, 60 or 75. The decision line is around
45: in the 40–49 bin there are 33 rejected and 31 shortlisted profiles. This matters for the reading below, because
a change of a few points only moves a decision when the profile sits near this line.

**Flips and score differences, each attribute against its own floor** (from the "FLIPS" and "FLIP RATE vs ITS
FLOOR" tables):

| attribute (40 pairs each) | flips | flip rate [95% CI] | floor | floor flips | pairs with different scores | mean \|diff\| [95% CI] | floor mean \|diff\| [95% CI] |
|---|---|---|---|---|---|---|---|
| name_language | 2 | 5.0% [1.4%, 16.5%] | null_name | 0/40, 0.0% [0.0%, 8.8%] | 11/40 | 3.12 [1.38, 5.25] | 2.00 [0.62, 3.62] (7/40) |
| gender_coded_name | 4 | 10.0% [4.0%, 23.1%] | null_name | 0/40, 0.0% [0.0%, 8.8%] | 6/40 | 1.12 [0.25, 2.13] | 2.00 [0.62, 3.62] (7/40) |
| university | 1 | 2.5% [0.4%, 12.9%] | null_university | 1/40, 2.5% [0.4%, 12.9%] | 10/40 | 2.50 [1.12, 4.12] | 2.12 [0.88, 3.62] (9/40) |

**Direction** (each pair turned so the named group is first):

- Name language: the 2 flips both favour the Kazakh-language name (2 vs 0, McNemar p = 0.500). Score gaps: Kazakh
  +0.7 [−1.8, +3.8], Russian +0.2 [−1.7, +2.1], English −1.0 [−4.2, +1.9]. Shortlist share: English 96.2% (n = 26),
  Kazakh 96.4% (n = 28), Russian 92.3% (n = 26).
- Gender coding: all 4 flips favour the female-coded name (male favoured 0, female favoured 4, McNemar p = 0.125).
  Shortlist share: female 90.0%, male 80.0% (n = 40 each). But the score gap is −0.4 [−1.2, +0.6] (sign test
  p = 0.219), and the mean absolute score difference (1.12) is *smaller* than the floor's (2.00).
- University: flagship favoured 1, regional favoured 0 (McNemar p = 1.000) for decisions, **but for scores the
  flagship member scored higher by +2.5 [+1.1, +4.1] points, sign test p = 0.002**. Shortlist share: flagship 92.5%,
  regional 90.0%; mean score 63.2 vs 60.8.

**What I can say.**

1. *Decisions.* No attribute stands clear of its floor on decisions. The 95% intervals of every attribute overlap
   with the interval of its floor. The detectability cell says that with n = 40 an exact McNemar test needs at least
   **6 pairs (15.0%)** in one direction (with 0 the other way) to be significant at 0.05. The largest one-sided count
   I saw was 4 (gender). So the honest statement is: *no decision effect was measured; an effect that moves up to
   about 5 pairs in 40 (12.5%) in one direction could exist and this run would not see it.* The gender result
   (4/0 for female-coded names, 90.0% vs 80.0% shortlisted) is the one to watch: it is not a finding, but it is
   also not proof of no effect. A run with more pairs would be needed (for example 160 pairs, where 4/0 in 40 would
   become 16/0 if the pattern held).
2. *Scores.* The university tier has a small but consistent effect on the score: +2.5 points for the flagship
   university, sign test p = 0.002 over 40 pairs. Even if I correct for the five comparisons in the table
   (0.05 / 5 = 0.01), it stays below the threshold. On the floor, the size of the movement (mean |diff| 2.12) is
   similar, but on null_university pairs a direction cannot exist (both universities are of the same tier), while
   on the university pairs the movement goes one way. So I read it as: **the tier of the university moves the
   score by about 2.5 points toward the flagship university; it changed a decision in 1 of 40 pairs, the same as the
   floor.** It is small because most profiles are far from the decision line; for a profile at 45 it can be the
   difference between shortlist and reject.
3. *Name language.* The mean |diff| (3.12) is above the floor (2.00), but the intervals overlap a lot and the signed
   gaps are all close to zero; no claim.

**How much could be noise, and what the floors do not test.** Decoding is greedy, so there is no random noise between
runs; the floors measure how much the output moves when an irrelevant detail changes (another name of the same group
and gender coding, another university of the same tier). They keep everything else fixed: the same prompt wording,
the same order of lines, the same model. They do not test other kinds of fragility. The optional order probe shows
how big that can be: for 40 pairs of **identical** applicants where only the order of the lines differs, **10 of 40
(25.0% [14.2%, 40.2%]) flipped, all 10 in one direction (McNemar p = 0.002), score −6.6 [−9.9, −3.5]**. The layout of
the prompt moves this agent much more than any protected attribute I measured. The harness run shows the same thing
from another side: removing three lines from the prompt changed 140 of 400 outcomes and 183 of 400 scores (mean
|delta| 12.43). A stronger estimate would need more pairs (the intervals are wide), several prompt wordings and line
orders, and other models, with the floors measured under each.

**Does it generalise?** No, not from this run. I measured one model, one prompt, one line order, invented names and
fictional universities whose names contain tier words ("National", "Research" vs "Regional", "Provincial", "Open").
With real applicants, real university names (whose prestige the model may know) or another prompt, the effects can be
larger, smaller or opposite. The run supports only: *this agent, with this prompt, on these 200 pairs.*

## 2. Who is affected

**Wrong rejection vs wrong shortlist.** The measured divergences affect real groups in this order:

- *Applicants from regional universities.* They get about 2.5 points less for the same profile (sign p = 0.002). For
  a profile near the decision line (around 45) this can mean a reject instead of a shortlist; in this run it
  happened in 1 of 40 pairs. A wrongly rejected applicant loses the scholarship money and maybe the chance to study at
  all, and the cost is larger for people from low-income families (the data does not tell me if they are more often
at regional institutions; if they are, the two disadvantages add up).
  The same 2.5 points work as a small advantage for flagship-university applicants, who then take part of a limited
  number of places.
- *Applicants with male-coded names* (4 of 40 pairs: the female twin shortlisted, the male twin not; 80.0% vs 90.0%
  shortlisted). Not a finding, but if it is real, men near the line are wrongly rejected and women at the same level
  displace them.
- *Applicants with Russian- or English-language names* (the 2 name-language flips both favoured the Kazakh-language
  name). Too few pairs to claim anything.
- *Everybody near the decision line*, because of the layout effect: the same applicant can be shortlisted or rejected
  only because of the order of lines in the form (25.0% of identical pairs). This hits applicants in the 40–49 score
  range, where 33 rejected and 31 shortlisted profiles of the baseline sit.

**The two errors do not cost the same.** A wrong rejection usually ends the process for the applicant (no human sees
the application again in the bare pipeline) and can follow them, if the result is reused (for example a second fund
or next year's screening reading the old score). A wrong shortlist costs the fund some reviewer time at the next stage
and, if it is not corrected, takes a place from somebody else; but it is usually seen and corrected by a person. The
bare agent is very lenient (356 of 400 shortlisted), so in this configuration wrong shortlists are probably more
common, while wrong rejections are rarer but heavier.

**Parties the brief does not mention.**

- *Reviewers.* With my harness 138 of 400 applications (34.5%) go to a person. That is a real workload; tired
  reviewers can rubber-stamp, so an escalation that nobody reads with care is not oversight.
- *Applicants who are refused.* A 17-year-old who is already a student, or an applicant who does not consent to
  automated processing, is not screened by the model; if the fund does not have a manual route, these people are
  silently excluded.
- *Applicants from institutions the model does not know*, and applicants with names in Cyrillic or with unusual
  characters: they were never tested (all names in the data are from one invented list in Latin script).
- *People who did not apply* because they expect a regional university to count against them; a known small bias in
  scores confirms that expectation.
- *The fund*, which carries the legal responsibility (Law 230-VIII art. 8(2), 21(5)) and the reputational risk.

**If an effect is hidden below what n = 40 can see.** An effect of up to about 5 pairs in 40 (12.5%) in one direction
cannot be detected here. If the gender pattern (4/0) is real, about one male applicant in ten near the line would be
wrongly rejected, and nobody would know from this measurement.

**Compounding over time.** If the system runs every year, the 2.5-point university gap adds up: fewer regional
students get the scholarship, fewer of them later appear as successful alumni, and any future model trained or tuned
on past decisions learns that regional means weaker. If the score is passed on to another stage (an interview list, a
ranking, another fund), the small gap becomes part of a "neutral" number that nobody questions.

## 3. Professional responsibility

**My responsibilities as the developer of the harness, given what I measured:**

1. *Do not claim more than the data shows.* I must not say "the agent is fair" (n = 40 cannot detect effects below
   15% of pairs, and the layout probe shows strong fragility), and I must not say "the agent discriminates by gender"
   (4/0, p = 0.125, not clear of the floor). I must report the one real result, the +2.5-point university score gap,
   as a measured fact with its limits.
2. *Check before it runs.* In this lab I did: the matched-pair measurement with floors, the order probe, the refusal
   path tested on six requests (5 refused, 0 model calls), the false-refusal check on 400 clean profiles (0 refused),
   and the before/after comparison. Still left: a larger sample, other prompt wordings and line orders, Cyrillic names,
   unknown programmes, and a test of the escalation rules that never fired in the run.
3. *Tell the deployer the limits and the price.* The fund must know: the harness removes the name and university
   from what the model sees, so those channels are closed by construction; it does **not** make the model stable (140
   of 400 outcomes changed just by removing lines); it sends 34.5% of applicants to a person, not the 15% I planned on
   the calibration set; only 14 automatic rejections are left; the layout of the prompt alone flips 25% of identical
   pairs.
4. *Build stop conditions.* The harness refuses when it must not decide, and escalates instead of guessing. A stop
   condition that is still missing: if the escalation share in production goes far above or below the planned value,
   the system should stop and be recalibrated.

**What belongs to the institution, not to me.** Setting the purpose and the merit criteria; deciding whether the
system may reject on its own at all; staffing and training reviewers; informing applicants that the decision is
automated and how to object, and handling objections within three working days (Law 94-V art. 19-1(2)–(3)); deciding
how long the log is kept and who reads it; the yearly risk review (Law 230-VIII art. 18(1)(4)).

**Where the two conflict.** If the fund wants to narrow or remove the escalation band because 34.5% of applications
is too much reviewer work, I can show what it buys: without it, the 32 profiles that moved from reject to escalate
would have been final rejections, and the profiles near the line are exactly where the university gap and the layout
effect change decisions. I would accept a narrower band only with a documented reason, a new measurement, and a
human route for every rejection on request. If the fund wanted to remove blinding or the refusal of minors and
non-consenting applicants, I would refuse to deliver that version: the consent rule is a legal condition (Law 94-V
art. 19-1(1)), not a feature, and the developer is responsible for the system's results according to their role
(Law 230-VIII art. 8(2)).
