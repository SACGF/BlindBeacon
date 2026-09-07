# Disclosure budget notes

Design stage, nothing computed yet. Written up from a discussion on 2026-09-04 in response to
"how are you computing query budget / information disclosure?".

## Principle

The clinician sees everything, unmetered. They already have access, and the budget is about
what leaves for Anthropic, not what the clinician sees. The budget applies only to the agent's
view.

Governance assumption: everything sent to Anthropic is stored forever, and models only get more
capable. The attacker is therefore an all-knowing agent with access to everything ever sent,
across every session. Anything that relies on forgetting (time-limited budgets, retention
periods) contradicts this and is out.

## Three parts

### 1. Record exclusion floor

- Predicates matching fewer than k records get a refusal, not a bucketed count.
- Those records are marked excluded for the rest of the session, and should persist across that
  clinician's later sessions. This is the cheap, categorical defence and it composes trivially.
- Blocks the classic Beacon differencing attack (ask A, ask A minus one condition, subtract).
- Cost: the excluded set only grows, so it needs an expiry or governance review eventually.

### 2. Epsilon-style accounting on counts

- The agent gets buckets instead of exact counts. Each answer draws down a session budget, and
  the session ends when it is spent.
- Use differential-privacy-style composition rather than ad hoc rules because the composition
  bound holds regardless of how the attacker chose the next query. That is the property that
  matters against an adaptive agent, which picks its next query from what it just learned.

### 3. Predicate narrowness (re-identification bit cost)

Same count, different leakage. Each predicate gets a specificity score: how many bits it takes
to single someone out, roughly

    specificity = log2( database_size / expected_matches_under_population_prior )

Narrower predicates cost more bits and hit the exclusion floor sooner.

Illustrative numbers against 67k samples, session budget 40 bits:

| Predicate | Prior | Expected matches | Bits | Agent sees | Budget after |
|---|---|---|---|---|---|
| `HP:0001249` intellectual disability | ~1:100 general, nearer 30% of a diagnostic exome cohort | ~20,000 | ~2 | "10,000 to 50,000" | 38 |
| ID AND `HP:0001250` seizures AND `HP:0000252` microcephaly AND female AND age 2-4 | 0.3 × 0.1 × 0.02 × 0.5 × 0.05, roughly 1:100k | <1 | ~16 | refusal (below k floor) | 22 |

Two more like the second and the session is over.

Key rule: **the query costs its bits whether it is answered or refused.** Asking a 16-bit
question is informative even when the answer is no. Otherwise the floor is a free oracle.

The 1:100k figure only means something relative to database size. 1:100k against 67k samples
is an expected match below 1, which is why it is a fingerprint.

Ontology walking feeds into this. Relaxing `HP:0002664` neoplasm up to a parent term, or down to
`HP:0004377` hematological neoplasm, changes expected matches and therefore the price. The agent
gets a cost signal for how specific its term choice is before the query runs.

## Multiple sessions

This is the open problem, and per-session budgets do not survive it on their own. A session
cap alone just makes the attack take N sessions.

- **Budget the principal, not the session.** The durable budget is per clinician per dataset.
  Sessions are sub-accounts drawing down a parent.
- **Refill, don't reset - but a refill alone is a rate limit, not a budget.** The per-clinician
  budget recovers slowly, say a fixed number of bits per week, so honest work rarely hits it and
  grinding does. As written above this contradicted the Principle section, which rules out
  anything that relies on forgetting, and Jordi Rambla's feedback (below) made the contradiction
  impossible to leave. Any refill makes cumulative disclosure unbounded in the limit; only the
  schedule changes. So the refill needs a **lifetime ceiling above it**, and the effective limit
  is the tighter of the two, never their sum. State the guarantee honestly as "at most X bits per
  week and at most Y bits ever", and let governance choose Y.
- **Persist record exclusion across sessions.** See part 1.
- **Price by novelty.** Keep a running tally of what has already been released to this
  clinician (or globally). A repeated predicate reveals nothing new and is nearly free; charge
  only the marginal specificity a query adds. The budget then tracks cumulative disclosure, not
  cumulative queries, which is what governance actually wants.
- **Sybil / cross-user composition.** Several clinicians' sessions combined. Same problem,
  harder, because the model is the common party. Defences are a dataset-wide budget on top of
  per-clinician ones, plus the audit log making collusion visible. A dataset-wide budget will be
  unpopular because one heavy user degrades everyone. Make it a configurable ceiling, off by
  default at first.
- **Split across providers.** Only helps if you assume providers do not collude. Weak, but
  cheap to keep as an option.

First cut: per-clinician refilling budget under a per-clinician lifetime ceiling, global record
exclusion, novelty pricing against a per-clinician release cache. Dataset-wide ceiling configurable,
off.

---

## Feedback from Jordi Rambla, 2026-09-07

Three points, from the Beacon v2 spec lead. All three are taken; two of them change the design and
one of them says a piece of the design as written cannot ship. Recorded here rather than paraphrased
away, because the third one is a constraint we will be tempted to optimise around.

### Composing with the Beacon's own budget

Their implementation already has an optional budget: per IP for anonymous users or per authenticated
user, spent on each positive answer, and **returned after some minutes have passed**. The question
raised is whether "double budgeting" happens once an agent is in the loop - one budget for the
clinician's ordinary interaction and another for the agent - and whether that is desirable.

Three different questions are bundled inside it, and they have different answers.

#### 1. A refunding budget is a rate limiter, not a disclosure budget

A budget bounds cumulative disclosure. A budget that returns after minutes bounds disclosure *per
unit time*. Against an attacker for whom waiting is free, and an agent is exactly that attacker, the
second bounds nothing at all: total disclosure is unbounded and only the schedule changes. This is
the same objection that applies to our own "refill, don't reset", which is why that bullet is now
corrected above rather than defended. Both need a lifetime ceiling sitting over the refilling one,
with the effective limit the tighter of the two.

There is a second, sharper failure specific to short refunds, and it is worth flagging to them
because it is not obvious. Al Aziz et al. note that a real-time randomised response can be averaged
out by simply repeating the query, so answers must be memoised per user. A budget that refunds after
minutes hands the attacker exactly the repetitions needed to average the noise away. **So a
refunding budget does not merely fail to bound disclosure, it actively undermines response
randomisation deployed alongside it.** Memoisation of repeated queries has to be permanent and keyed
to the principal, independent of whatever the budget does. The Portuguese Beacon already does this,
recalling repeated identical queries from stored history rather than recharging them, which is our
novelty pricing in production.

Framed positively, the refunding behaviour is doing a real job. It is abuse and load control, and it
should keep doing that. It just must not be counted as disclosure control in governance material,
because the two have different guarantees.

#### 2. Per IP is not a principal

Per-IP metering is Sybil-trivial and the literature is unusually blunt about the consequence:
Venkatesaramani et al. prove that without knowing who is asking and what they already asked, an
online defence collapses to one fixed decision made up front, and that against an adaptive attacker
in that setting every method they tested, including their own, fails. So the per-IP mode is a rate
limiter with a budget's name on it. Keep it, do not count it.

For BlindBeacon this is moot in the useful direction: we require authenticated, server-side
sessions anyway, for the audit claim. That gives us the stateful authenticated case that Wan et al.
could not evaluate and said would make budget-based methods practical.

#### 3. The actual double-budgeting question

Three cases, and only one of them is the harmful one.

| Case | Verdict |
|---|---|
| Beacon's per-user budget **and** BlindBeacon's egress budget, sitting on different pipes | Keep both. Not double counting |
| The agent's queries charged against the clinician's own Beacon budget | Avoid. Actively harmful, see below |
| Per-IP **and** per-user metering of the same pipe | Not two budgets. One budget with a Sybil hole |

The first case looks like double counting and is not, because the two meters answer to different
attackers. The Beacon's own budget protects data subjects from *the user*. BlindBeacon's budget
protects data subjects from *the model provider*, under the governance assumption that everything
sent there is stored forever and that future models mine it better than today's. Different attacker,
different retention, different collusion set. Two meters on two pipes is correct, and collapsing
them would be the error.

The second case is the one to design against, and it is worse than inefficient. If the gateway
queries upstream using the clinician's own credentials, every exploratory agent query spends the
clinician's Beacon budget. Put the Portuguese Beacon's measured numbers against that: 8.3 mean
queries before the first data subject is occluded at N=25,000 and 32.5 at N=100,000, so VariantGrid's
67,741 samples land on the order of twenty. An agent walking an ontology will burn a whole session's
worth of subject budget in its first few minutes of relaxation. Three things then break at once:

- The clinician's exact view, which we promised is unmetered, is silently degraded because an agent
  was working on their behalf.
- Occlusion under Raisaro-style budgets is per `{user, subject}` and **silent**, so neither view
  reports that it is now computed over a subset.
- Which subjects were dropped is a function of the data, so the fuzzed view inherits the shape of
  the exclusion set. That is the leak channel section 4.3 of the literature review is about, now
  running underneath our own measurements.

The design rule that follows: **the gateway authenticates upstream as its own service principal**,
carrying the clinician's identity in-band for the audit log rather than in the upstream credential.
The upstream then meters the gateway once, and the clinician's own direct Beacon access, if they
have any, keeps a separate account and is unaffected.

And where two meters genuinely do cover the same pipe, **take the minimum, never the sum**. This is
the whole answer to "is double budgeting desirable". Summing is how two budgets become twice the
budget and the agent route becomes a way to *buy* budget; if that were the outcome we would have
shipped an exploit rather than a control. Nesting is how it stays a control.

There is a corollary for the evaluation, and it is a threat to experimental validity rather than to
privacy. Silent upstream occlusion moves the ground truth underneath Arm 2: re-identification rates
measured against a cohort that is quietly shrinking are not measuring our restriction levels. On the
synthetic instance we control this, and RUNX1db is on our own stack. For cross-testing against
sBeacon and DNAstack's we have to either get their budgets disabled for the runs or detect occlusion,
and we cannot currently detect it, by design of the mechanism. Raise this before B3, not during.

### Who prices sensitivity: the steward, not the agent

The second point is a framing one: contextualise re-identification as *what does the attacker learn
that it did not already have?* If the attacker already holds every detail and the dataset is about a
given rare disease, a count of 1 tells them only the diagnosis. Is that high, medium or low risk?
Probably neither the agent nor the model should decide - the data steward should.

Agreed, and the framing is right. It is Bayes advantage, prior against posterior, and it is what our
specificity measure is already trying to be. Two things follow, pulling in opposite directions.

**Use it as a reporting frame.** Tramèr et al.'s bounded-prior results convert an epsilon into a
statement of the form "for an adversary whose prior on this individual's membership is p, this
release moves the posterior to at most q". That is enormously more legible to a governance committee
than an epsilon, and it costs nothing because we still account at the unbounded rate. Section 5.4 of
the literature review already lands on this: use the paper as a calibration instrument, not as a
budget mechanism.

**Do not let it become a discount inside the meter.** "The attacker already knew that" cannot reduce
a price. Bounded priors have no composition theory - the word does not appear in the paper - and an
adaptive agent manufactures the violation on purpose, because the posterior after query one is the
prior for query two. A relaxation with no composition theorem is not a budget.

On the specific example: in a single-disease registry, membership *is* the diagnosis. That is
Shringarpure and Bustamante's original point, made against the SFARI beacon, and it means the
boolean tier alone already discloses the sensitive attribute before any count is returned. Whether
that is high, medium or low risk is genuinely not derivable from the query, so the instinct here is
right, and the escalation to identifying a family is real in both directions: kinship means a
positive on one member re-prices every relative.

So make it a **declared per-dataset input**, set once by the steward, not a per-query inference:

- What membership in this dataset implies: nothing, a phenotype, a diagnosis, or a stigmatised or
  legally consequential diagnosis.
- Whether the cohort contains families, which switches on pedigree-aware exclusion.
- The k floor, and the response granularity available to the model.

Three or four declarations, parameterising everything downstream. Nothing infers risk at query time,
and in particular the model never assesses its own disclosure risk - which is also the governance
constraint that the guarantee must not depend on the model's behaviour.

### Simplicity is a constraint, not a preference

The third point is the one that bites: avoid sophisticated decision trees for masking query results,
in favour of something simpler for Beacon admins to understand and manage.

Take it, and take it as a security requirement rather than a usability concession. Three reasons,
all from the literature review rather than from taste.

1. **Simulatability.** A refusal must be recomputable from the released transcript by anyone holding
   it - Kenthapadi et al.'s definition, and the reason a simulatable refusal need not be charged at
   all. A simple, transcript-computable rule can be verified by an auditor with no access to the
   data. A rule with many data-dependent branches probably is not simulatable, and a non-simulatable
   refusal is an oracle for the very count it refused.
2. **Every modelling assumption in a price is a discount waiting to be found.** Cho et al. broke
   Raisaro's budget in under 40 queries by querying variants that deviate from Hardy-Weinberg
   equilibrium, because the model used to price queries did not fit the data. Our specificity
   measure has exactly the same shape and exactly the same exposure: it prices against a population
   prior, and a diagnostic cohort is not a population sample.
3. **A dial an admin does not understand is set wrong or switched off.** The Portuguese Beacon
   deliberately does not publish its `p`, which only works if the admin knows what `p` does.

The reconciliation is **complexity in the evaluation, simplicity in the knob**. The admin-facing
surface should be a named restriction level chosen off the published curve, plus the steward
declarations above. Deliverable E2 exists precisely so an operator picks a point on a curve instead
of tuning a mechanism. Everything sophisticated is then either precomputed offline into the presets,
or lives in the audit and reporting layer where it informs but does not gate.

This cuts, and it should be applied before B rather than after. **If a budget component cannot be
reduced to a preset an admin picks without understanding it, it does not ship.** Our three-part
budget as currently written does not pass that test, and saying so is a finding rather than a
defence. There is also a direct privacy argument for the same conclusion, already listed below as an
open problem: the price the agent is quoted is itself a side channel that leaks the prior, and a
simpler pricing function leaks less through it.

---

## Other open problems for the attack phase

- The agent probing the budget itself as a side channel (the price of a query leaks the prior).
- The agent talking the clinician into reading exact numbers back to it. Social channel, not a
  query channel. No good answer yet beyond the audit log catching it.
- Budget-aware query planning: an agent that knows the pricing will spend its bits optimally.
  Assume it does.

## How we will test it

Synthetic instance with known ground truth (public genomes alone will not do, no phenotypes and
no patients to re-identify).

1. Let the agent go nuts with no restrictions. Measure what proportion of synthetic patients it
   re-identifies.
2. Lock down at various restriction levels and plot effectiveness against leakage.
3. Give the agent N fresh sessions and plot re-identification against N. Flat curve means the
   composition story holds. Linear in N means the per-session budget is theatre.
4. Clinicians on the plain unrestricted Beacon give the control arm for what honest use needs.
5. Pin the refunding-budget break in CI, Noisegate style: with budget refunds on and per-principal
   memoisation off, repeat one query across refund windows and show the fuzzing averages out. With
   memoisation on it must not. This is the concrete form of the claim made to Jordi Rambla above,
   and it should be demonstrable rather than argued.

## Generalises

Nothing here is Beacon-specific. It fits any tiered-response API where an untrusted agent sits
between a trusted user and governed records. Beacon just already has the presence / count /
count-plus-phenotype tiering to hang the two views off.
