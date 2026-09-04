# BlindBeacon: project background

Substance extracted from the original proposal. The proposal itself is not in this repository: it
contained budget figures and other material that does not belong in public.
Everything below is the part that governs what gets built, and it supersedes the proposal as the
working spec.

Six months from project start.

## The problem

Rare disease research is bottlenecked by cohort assembly. Today, bioinformaticians mine clinical
databases by hand on behalf of the people with the disease knowledge, so every question is a round
trip. Clinical governance rules out cloud-based agents, and after working at agentic speed that
round trip is painfully slow.

A recent worked example, and the baseline we measure against. A clinician wanted patients with a
given gene and phenotype for a trial. Phenotypes and ontologies are imperfect and our data is
inconsistently labelled, so hitting the target numbers meant relaxing the phenotype match (inferring
from the test ordered, reading patient notes) and relaxing the affected gene (pathogenic in others
but not in them). That is data analysis, not a form, and it needs questions going back and forth.

Answering it took months of elapsed time and a long email thread across a substantial group of
people, and it is still going. That is the current manual baseline, and it is what we measure
against. The specific figures are recorded internally rather than here.

## Approach

Build over the GA4GH Beacon v2 API rather than over VariantGrid directly, so the work applies to any
Beacon implementation rather than only our own. Beacon's three-tiered access model already limits
returns to presence/absence, counts, or counts plus phenotypes.

**View splitting.** The same query returns two answers. The clinician, who is trusted, sees exact
counts, phenotypes and ages. The agent sees fuzzed values and ranges. The agent contributes query
construction and ontology skill: MONDO and HPO term walking, paralogs, pathogenicity thresholds. The
clinician contributes disease knowledge and the judgement that comes from patient notes, and decides
what to relax next.

**Disclosure budget.** Metered per session, in three parts: categorical record-exclusion,
epsilon-style accounting on counts, and a specificity measure on predicate narrowness. It must hold
under composition.

**Audit.** Logs must be able to prove, per session, that no record-level data reached the model.

## Threat model

Agentic discovery and agentic re-identification are the same capability, so the defence is only as
good as the best attacker fielded against it. That is why red-teaming uses frontier models.

The known Beacon attacks, membership inference and genome reconstruction from snapshots, are
brute-force loops. An agent is different: it adapts its next query based on what it just learned.
The budget therefore has to hold against an adaptive adversary, not merely against a loop.

All attack work runs against a synthetic instance, never against patient records.

## Design constraints

These come from clinical governance and are not negotiable defaults to be optimised away.

- Assume anything sent to a model provider is stored forever, and that models only get more capable.
  What cannot be mined from a response today may be minable in five years. Fuzz accordingly.
- No record-level data reaches the model, and the audit log must be able to prove it per session.
- All attack work runs against synthetic data.

## Evaluation

Three arms, and the deliverable is the curve between them.

1. **Control.** Clinicians on a plain, unfuzzed Beacon MCP with no budget, including hands-on
   workshops. This establishes what an honest user and an ordinary agent retrieve.
2. **Attack.** An adversarial agent against the synthetic instance, measuring what proportion of
   synthetic patients it re-identifies at each restriction level.
3. **Live.** Clinicians on real data for actual trial recruitment.

Restriction levels are then iterated to find the capability and privacy balance.

## Deliverables

1. Open source harness: view splitting and disclosure budget, over MCP.
2. An evaluation of how well a model does a rare disease task, and where it fails.
3. Live deployment on trial recruitment.
4. Curves and best practice back to Monarch, offered to GA4GH and ClinGen.

## Timeline

| Months | Work |
|---|---|
| 1 to 2 | Harness and synthetic instance |
| 2 to 4 | Attack and calibrate |
| 4 to 5 | Clinicians on real data. Ethics amendment required; RUNX1db and synthetic work are the fallback |
| 6 | Report and release |

## Success measures

Success is patients recruited and time to answer, not a benchmark score.

- **Usage.** Clinicians and researchers running mining sessions without a bioinformatician.
- **Outcome.** Patients recruited to trials or studies, and questions answered that previously
  needed manual work.
- **Speed.** Time to answer against the current manual baseline.
- **Coverage.** Every mining request pre-registered, reporting the proportion solved via the harness.
- **Delivery.** Harness open sourced, attack results and curves published, best practice to Monarch
  and GA4GH.
- **Safety.** Audit logs confirming no record-level data reached the model, disclosure budget per
  session, and the proportion of synthetic patients re-identified at each restriction level.

## Why this matters beyond us

Today the answer to "can we point an agent at patient data" is no, because the choice is framed as
full access or nothing and there is no fuzzed, budgeted, API-restricted middle ground. We intend to
provide concrete numbers for governance approval, and to publish attacker cost curves for adaptive
agents against Beacons.

Because it is built on Beacon rather than on VariantGrid, the work applies to many other systems.
The same pattern fits other governed clinical data besides variants, which SA Health trusted research
environment forums are actively discussing. RUNX1db benefits first: the harness would let
international consortium collaborators query it without records leaving our systems.

## Systems and context

- **VariantGrid** (https://github.com/SACGF/variantgrid) is SA Pathology's open source variant
  database and ACMG curation system, holding 67,741 genetic samples. Its REST API
  (https://variantgrid.com/api/docs) already includes Beacon v2. Extending that implementation is in
  scope. Being open source, we have full code and data access to do our own mining, unlike labs on
  proprietary systems.
- **RUNX1db** (https://runx1db.runx1-fpd.org/) is a rare disease database on the same stack, hosted
  for the RUNX1 familial platelet disorder patient organisation. First intended beneficiary.
- **cdot** (https://github.com/SACGF/cdot) is the same team's HGVS transcript resolution service,
  published with an llms.txt. Prior art for agent-accessible genomics infrastructure on open data;
  this project is the controlled-access version of that idea.
- **Shariant** (https://shariant.org.au/) is Australian Genomics' national variant sharing platform,
  which selected VariantGrid as its technical platform after evaluating nine alternatives.

SA Pathology is South Australia's public pathology service. Executive approval and ethics are in
place for the manual version of this work, which happens today by hand.

## Team

| Person | Role |
|---|---|
| David Lawrence | Senior Bioinformatician, VariantGrid lead. Builds the harness and APIs |
| A/Prof Chung Hoow Kok | Head of Data and Bioinformatics Innovation, SA Pathology. AI and rare disease |
| Prof Hamish Scott | Head of Genetics and Molecular Pathology, SA Pathology. Governance, cohorts |
| Arthur Gymer | Bioinformatician, formerly Genomics England. Adversarial evaluation |
| Dr Jarrod Marks | Clinician and engineer. Defines clinician recruitment and mining needs |

## Disclosure policy

Attack work runs only against synthetic data, never patient records. Reimplementations of already
published Beacon attacks are developed in the open. Novel attacks that improve on the published state
of the art, particularly adaptive-agent attacks, are coordinated with Anthropic's safeguards team
before publication, including sharing transcripts and results. Attacker cost curves are published
after that coordination.
