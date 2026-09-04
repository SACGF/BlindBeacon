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
  queries. Start simple and let B break it.
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

## B. Attack and calibrate (M2–M4)

Goal: know how much an adaptive agent can re-identify at each restriction level, and pick one.

- **B1. Attacker harness.** Claude driving the plain MCP (A2) and the fuzzed MCP (A3/A4)
  against the synthetic instance (A1), with a scoring loop against ground truth. Frontier
  models, bulk of the credit spend.
- **B2. Attack library.** Reproduce known Beacon attacks (membership inference, genome
  reconstruction from snapshots) as baselines, then let the agent adapt: query-splitting,
  budget-probing, cross-session composition, prompt-level exfil of the clinician view.
- **B3. Restriction-level sweep.** Run B1 across the A3/A4 configurations. Output is the
  capability-vs-privacy curve: proportion of synthetic patients re-identified vs proportion of
  honest mining tasks still solvable.
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
  short best-practice document. To Monarch, offered to GA4GH and ClinGen.
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
