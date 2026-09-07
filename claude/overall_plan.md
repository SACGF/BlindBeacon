# Overall plan

High-level milestones and sub-projects. Timeline is 6 months; the four deliverables are
(1) open source harness, (2) evaluation, (3) live deployment, (4) curves and best practice to
Monarch / GA4GH / ClinGen. Spec is `doc/project.md`; design decisions in
`claude/architecture.md`; budget design in `claude/budget_notes.md`.

```
M1 ──── M2 ──── M3 ──── M4 ──── M5 ──── M6
[ A. Harness + synthetic ]
        [ B. Attack + calibrate      ]
                        [ C. Live use  ]
[ D. Control arm (workshops) ...........]
                                [ E. Report ]
[ F. Governance / ethics (lodged at start, runs throughout) ]
```

---

## A. Harness and synthetic instance (M1–M2)

Goal: a working two-view Beacon MCP, and a fake Beacon to point it at.

- **A1. Synthetic Beacon instance.** Generate a synthetic cohort (variants, phenotypes, ages,
  genes) with known ground truth, served over Beacon v2. This is the target for all attack work
  and the sandbox for workshops. Must be realistic enough that attacks against it mean something.
  Candidate: extend VariantGrid's existing Beacon on a synthetic database; fallback is a
  standalone Beacon v2 reference implementation with generated data.
- **A2. Plain Beacon MCP.** *Adopt, do not build.* Fiume's `ga4gh-agentic-harness-mcp` (Apache-2.0)
  is the control-arm tool. Use the profile's canonical `ga4gh.beacon.variant.query` name for our
  tools too, so control and treatment arms present the same surface and the comparison is not
  confounded. Note the harness does **not** wrap this: it speaks Beacon v2 REST upstream, because we
  cannot fuzz what we cannot parse. See `claude/architecture.md`.
- **A3. View splitting.** Same query, two answers: exact to the clinician's UI, fuzzed/ranges to
  the model. Define the fuzzing functions (count bucketing, age ranges, phenotype coarsening) as
  configurable restriction levels so B can iterate them.
- **A4. Disclosure budget.** Per-session metering: categorical record-exclusion, epsilon-style
  accounting on counts, specificity measure on predicate narrowness. Must compose across
  queries. Start simple and let B break it. Four constraints added since, from the literature
  review and from Jordi Rambla's feedback; `claude/budget_notes.md` is the authority.
  - A refill is a rate limit, not a budget. Any refilling ceiling needs a lifetime ceiling above
    it, and the effective limit is the tighter of the two, never the sum.
  - Memoisation of repeated queries is permanent and keyed to the principal, and must not depend
    on budget state. Otherwise refunds hand the attacker the repetitions needed to average out the
    fuzzing.
  - Composition with any budget the upstream Beacon already runs. The gateway authenticates
    upstream as its own service principal, so agent queries never land on the clinician's account
    and silently occlude their view. Two meters on one pipe compose as a minimum, not a sum.
  - Refusals must be simulatable: computable from the query history and already-released answers,
    never from the answer being refused. A non-simulatable refusal is an oracle for the count that
    triggered it, and charging bits for it does not fix that.
- **A5. Audit log.** Per-session log that can *prove* no record-level data reached the model.
  Every response to the model is logged post-fuzz; log format is designed for governance review.
- **A6. Clinician webapp.** Hosts the conversation server-side rather than pointing Claude Desktop
  at us. This is not cosmetic: MCP has no channel that shows a human something without it entering
  model context, so the trusted view must live outside the transport, and the audit claim is
  unprovable if the clinician's own client talks to the model provider. Includes the explicit
  "tell Claude" declassification control. See `claude/architecture.md`.
- **A7. Ontology tooling.** MONDO/HPO term walking, gene paralogs, pathogenicity thresholds, as
  tools the model calls. Reuse AskBeacon's ontology retrieval approach where it fits.
- **A8. VariantGrid Beacon extension.** Whatever VariantGrid's Beacon v2 needs to support the
  above (filters, tiered responses). In scope per `doc/project.md`.
- **A9. Operator surface: presets and steward declarations.** The admin-facing configuration is a
  *named restriction level* picked off the E2 curve, plus a handful of per-dataset declarations set
  by the data steward: what membership in this dataset implies (nothing, a phenotype, a diagnosis,
  a stigmatised one), whether the cohort contains families, the k floor, and the granularity the
  model may see. Nothing infers risk at query time. **Test to apply to every budget component: if
  it cannot be reduced to a preset an admin picks without understanding it, it does not ship.** The
  three-part budget as currently written does not pass this yet, which is a finding for A4 to
  resolve, not a caveat to carry.
- **A10. Per-subject exposure ledger.** The exclusion floor is per individual but currently priced
  with a population average, and the worst-affected participant in Simmons and Berger's real-data
  measurement was about five times as exposed as the average one. Needs a per-subject accumulator
  of that individual's own contribution to what has been released. Also the place where the
  release-decision leak gets corrected for, by tightening the internal threshold until the
  posterior conditioned on the fact of release still meets the advertised bound.

## B. Attack and calibrate (M2–M4)

Goal: know how much an adaptive agent can re-identify at each restriction level, and pick one.

- **B1. Attacker harness.** Claude driving the plain MCP (A2) and the fuzzed MCP (A3/A4)
  against the synthetic instance (A1), with a scoring loop against ground truth. Frontier
  models, bulk of the credit spend.
- **B2. Attack library.** Reproduce known Beacon attacks (membership inference, genome
  reconstruction from snapshots) as baselines, then let the agent adapt: query-splitting,
  budget-probing, cross-session composition, prompt-level exfil of the clinician view. Add two
  from the feedback: **repeat-to-average**, where a refilling or refunding budget supplies the
  repetitions needed to average the fuzzing away, and **price-model probing**, hunting the
  discount where the specificity prior does not fit a diagnostic cohort, which is how Cho et al.
  broke Raisaro's budget in under 40 queries.
- **B2a. Pin the gallery in CI.** Noisegate's pattern, and a better shape than a one-off
  evaluation: each attack demonstrated succeeding with the defence off and failing with it on, on
  every build, so the defence cannot quietly rot. Cheap, and it is what makes B4 safe to iterate.
- **B3. Restriction-level sweep.** Run B1 across the A3/A4 configurations. Output is the
  capability-vs-privacy curve: proportion of synthetic patients re-identified vs proportion of
  honest mining tasks still solvable. Report re-identification rate at a fixed false positive
  rate, not hit counts. **Validity precondition:** any budget the upstream Beacon runs must be off
  for these runs. Raisaro-style occlusion is silent, so an upstream budget shrinks the cohort
  underneath the measurement and we cannot detect that it happened. We control the synthetic
  instance and RUNX1db; sBeacon and DNAstack's have to be arranged before the sweep, not during.
- **B4. Fix-and-rerun.** Feed breaks back into A3/A4. Iterate until the curve has a usable knee.
- **B5. Safeguards liaison.** Share attack transcripts with Anthropic safeguards; coordinate
  disclosure before publishing attacker-cost curves.

## C. Live deployment (M4–M5)

Goal: clinicians recruit real patients through the harness.

- **C1. Pre-registered request list.** Every mining/recruitment request logged before the
  harness touches it, so coverage (proportion solved via harness) is honest.
- **C2. RUNX1db first.** Lower governance bar, international consortium collaborators querying
  without records leaving SA Pathology. Also the fallback if the ethics amendment is late.
- **C3. SA Pathology VariantGrid.** Real trial recruitment against 67k samples. Depends on F.
- **C4. Measurement.** Time-to-answer against the current manual baseline (figures held
  internally, not in this repo);
  patients recruited; audit logs per session.

## D. Control arm (runs alongside A–C)

Goal: what an honest user and an ordinary agent retrieve with no fuzzing and no budget.

- **D1. Claude Science workshops.** Clinicians and researchers drive the plain MCP (A2) against
  the synthetic instance (A1). Trains users, and the session logs become the control arm.
- **D2. Task set.** A fixed set of realistic mining tasks (drawn from C1 and past manual
  requests) that both control and harness arms attempt, so the two are comparable.

## E. Report and release (M6)

- **E1. Open source release.** Harness, synthetic generator, attack harness, restriction-level
  configs, with the audit format documented.
- **E2. Curves and best practice.** Attacker-cost curves, recommended restriction levels, and a
  short best-practice document. To Monarch, offered to GA4GH and ClinGen. The curve is the
  evidence; the **shipped artefact is the named presets read off it**, because an operator should
  pick a point rather than tune a mechanism (A9). Report alongside it the prior-to-posterior
  statement that makes an epsilon legible - "for an adversary with prior p, this moves the
  posterior to at most q" - which costs nothing since accounting stays at the unbounded rate.
- **E2a. Spec feedback to GA4GH.** Distinct from the best-practice document and cheaper to land.
  Concrete asks accumulated so far: enough structure on the query object to meter predicate
  specificity, granularity as a first-class enforceable field (both raised with Fiume), and
  guidance separating a refunding rate limiter from a disclosure budget, with repeated-query
  memoisation specified independently of budget state (from Rambla). All three are small,
  and the profile and spec are early enough to influence.
- **E3. Write-up.** Paper / preprint. AskBeacon is the natural "no budget" comparator.

## F. Governance and ethics (throughout)

- **F1. Ethics amendment** lodged at project start. Gates C3.
- **F2. Governance stats pack.** The concrete numbers (budget, re-identification rate, audit)
  that turn "can we point an agent at patient data" from "no" into a qualified "yes". Feeds SA
  Health TRE forum discussions.
- **F3. Bedrock contingency.** The harness must not assume the first-party API. Note the
  server-side MCP connector is unavailable on Bedrock and Vertex, so we run our own tool loop.

---

## Dependencies, in one line each

- A1 and A2 first; everything else depends on them.
- B needs A3/A4 to exist but not to be good. Start attacking early.
- D can start as soon as A1/A2 are up, and should, because workshop scheduling is slow.
- C2 can start before F1 lands. C3 cannot.
- E depends on B3 having a curve and C4 having at least one real recruitment.
- A9's preset test gates A4: a budget component that cannot be reduced to a preset is not finished.
- B3 is gated on upstream budgets being off wherever we do not own the Beacon.
