# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

Named **BlindBeacon** on 2026-09-04 (was "Fuzzed Beacon agent harness"). The repo lives at
`SACGF/BlindBeacon` and is public.

No code yet beyond an empty package skeleton and `pyproject.toml` (package `blindbeacon`, Apache-2.0,
src layout, not yet published to PyPI). There is no build, test, or lint tooling to run. When the
first code lands, replace this section with the actual commands.

Planning material, roughly in reading order:

- `doc/project.md` - background, approach, evaluation design and timeline. The authoritative spec
  for what gets built.
- `claude/overall_plan.md` - milestones and sub-projects (A to F) with dependencies.
- `claude/architecture.md` - working design decisions and their reasons.
- `claude/budget_notes.md` - disclosure budget design. The authority on the budget.
- `notes.md` - prior-art reviews (GA4GH Agentic Harness, Noisegate, AskBeacon) and the licensing
  decision.
- `literature/beacon_attacks/` - literature review of published attacks on GA4GH Beacons and the
  defences proposed against them, plus the statistical-database theory underneath. `README.md` is
  the review, `bibliography.md` has the DOIs and a confidence tag per entry saying whether the
  paper was read in full, read at abstract level, or is an unverified lead not to be cited. Source
  PDFs are not kept in the repository, since most are under publisher copyright and this repo is
  public.

The original funding application has been removed: it held budget figures and negotiation detail
that must not go public. `doc/project.md` carries everything from it that governs the build.
Funding is not yet finalised, so no funder is named anywhere in this repo.

## What is being built

A harness that lets Claude mine clinical variant databases for rare disease cohort assembly and
trial recruitment, without record-level patient data ever reaching the model. Exposed over MCP,
built against the **GA4GH Beacon v2** API rather than against VariantGrid directly, so it applies to
any Beacon implementation.

The core idea is **view splitting**: the same query returns two different answers.

- **The clinician** (trusted) sees exact counts, phenotypes, ages.
- **Claude** (untrusted) sees fuzzed values and ranges only.

They then work the problem together — Claude contributes query construction and ontology skill
(MONDO/HPO term walking, paralogs, pathogenicity thresholds), the clinician contributes disease
knowledge and judgement from patient notes, and decides what to relax next.

Layered on top is a **disclosure budget** metered per session. It must hold under *composition* and
against an *adaptive* attacker — the known Beacon attacks (membership inference, genome
reconstruction from snapshots) are brute-force loops, but an agent adapts its next query based on
what it just learned. Budget design has three parts: categorical record-exclusion, epsilon-style
accounting on counts, and a specificity measure on predicate narrowness.

## Design constraints

These come from clinical governance and are not negotiable defaults to be optimised away:

- Assume anything sent to Anthropic is stored forever, and that models only get more capable — what
  cannot be mined from a response today may be minable in five years. Fuzz accordingly.
- No record-level data reaches the model. Audit logs must be able to *prove* this, per session.
- Agentic discovery and agentic re-identification are the same capability. The defence is only as
  good as the best attacker fielded against it, so red-teaming uses frontier models.
- All attack work runs against a **synthetic** instance, never patient records.

## Evaluation shape

Three arms, and the deliverable is the curve between them:

1. **Control** — clinicians on a plain, unfuzzed Beacon MCP with no budget. What an honest user and
   an ordinary agent retrieve.
2. **Attack** — Claude adversarially attacking the synthetic instance, measuring what proportion of
   synthetic patients it re-identifies at each restriction level.
3. **Live** — clinicians on real data for actual trial recruitment.

Restriction levels are then iterated to find the capability/privacy balance. Success is measured as
patients actually recruited and time-to-answer against the current manual baseline, which runs to
months of elapsed time across a substantial group of people, not as a benchmark score. Keep the
specific figures out of this repo.

## Context you will need

- **VariantGrid** — https://github.com/SACGF/variantgrid — SA Pathology's open-source variant
  database and ACMG curation system (67,741 samples). Its REST API
  (https://variantgrid.com/api/docs#/) already includes Beacon v2, whose 3-tiered access model
  returns presence/absence, counts, or counts + phenotypes. Extending that implementation is in
  scope for this project.
- **RUNX1db** — https://runx1db.runx1-fpd.org/ — rare disease database on the same stack, hosted for
  the RUNX1 familial platelet disorder patient organisation. First intended beneficiary: it would
  let international consortium collaborators query without records leaving SA Pathology systems.
- **cdot** — https://github.com/SACGF/cdot — same team's HGVS transcript resolution service,
  published with an llms.txt. Prior art for agent-accessible genomics infrastructure; this project
  is the controlled-access version of that idea.
- **AskBeacon** (Denis Bauer's team) — a chat interface over Beacon. Flagged in `notes.md` as worth
  reviewing; not yet evaluated.
- Findings are intended to go back to Monarch, GA4GH, and ClinGen.
