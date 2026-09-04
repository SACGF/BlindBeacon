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
- **Refill, don't reset.** The per-clinician budget recovers slowly, say a fixed number of bits
  per week. It never fully forgets, so it composes. Honest work rarely hits it; grinding does.
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

First cut: per-clinician refilling budget, global record exclusion, novelty pricing against a
per-clinician release cache. Dataset-wide ceiling configurable, off.

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

## Generalises

Nothing here is Beacon-specific. It fits any tiered-response API where an untrusted agent sits
between a trusted user and governed records. Beacon just already has the presence / count /
count-plus-phenotype tiering to hang the two views off.
