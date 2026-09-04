Denis Bauer and her team have developed AskBeacon, which is a chat interface on top of Beacon. Worth to look at it?


## AskBeacon (reviewed 2026-09-04)

- Paper: https://pmc.ncbi.nlm.nih.gov/articles/PMC11889448/ (Bioinformatics 2025), preprint https://arxiv.org/abs/2410.16700
- Repo: https://github.com/aehrc/AskBeacon – Jupyter notebooks + Python utils, not a standalone app. 16 commits.
  Sits on top of sBeacon (CSIRO's serverless Beacon on AWS).
- What it does: NL question -> LLM extracts scope/granularity/variants/filters -> confirms with user
  -> Beacon query -> LLM generates analysis/visualisation code, run sandboxed after static analysis.
- LLMs: GPT-4 via Azure, plus Gemma 2 / Mistral / Llama / Qwen via Ollama. Gemma 2 best of open models.
- Privacy model: "sBeacon is the conduit, data never exposed to the LLM directly", extractors run at
  the user's own access level, code sanitised. That is it.
- Not addressed: privacy budget, composition, membership inference, adaptive attacker, fuzzing of
  what the LLM sees (LLM sees the real Beacon response, just not raw records).
- Relevance to us: overlaps on the NL->Beacon query construction and ontology term lookup. Nothing
  on view splitting or disclosure budgets, so it is complementary rather than competing. Worth
  reusing their ontology retrieval approach; worth citing as the "no budget" baseline. Possible
  collaboration/eval target since sBeacon is another Beacon implementation to test against.

## GA4GH Agentic Harness + existing Beacon MCPs (reviewed 2026-09-04)

Prior art found while checking whether a plain Beacon MCP needs building. It does not.

- **GA4GH Agentic Harness Profile** - https://github.com/mfiume/ga4gh-agentic-harness-profile
  Draft v0.1, Apache-2.0, "Copyright 2026 Global Alliance for Genomics and Health". Marc Fiume
  (DNAstack) personal repo, pushed 2026-08-12, pre-release. A protocol-neutral operation model for
  exposing GA4GH standards to agents: normative profile, conformance classes, JSON Schema contracts,
  an explicit MCP binding (`bindings/mcp.md`) and a 13-operation catalog. One of the 13 is
  `ga4gh.beacon.variant.query`.
  Reference impls: `mfiume/ga4gh-agentic-harness-python` (SDK), `mfiume/ga4gh-agentic-harness-mcp`
  (MCP projection over the SDK, stdio + streamable-http).
- **DNAstack/ga4gh-mcp-server** - 57 tools over the GA4GH ecosystem incl. 7 Beacon v2 tools. Live
  hosted endpoint, API key on request. No license file. Python.
- `vsmalladi/ga4gh-mcp` - smaller, MIT, less active.

Status: pitched to Marc Fiume on the GA4GH Slack (2026-09-04). Response was positive and he sounds
keen to collaborate. Treat the profile as a partner track, not competing prior art.

Implications:

1. **Do not build a plain Beacon MCP.** Build on the profile and Fiume's reference implementation,
   and use the canonical operation name `ga4gh.beacon.variant.query` so our control and treatment
   arms share a tool surface. If the two arms differ in tool naming as well as in fuzzing, the
   comparison is confounded.
2. **Licensing.** Fiume's profile and MCP server are Apache-2.0, so we can build on them.
   DNAstack/ga4gh-mcp-server has no license file, which means all rights reserved and we cannot
   legally build on it as it stands. Worth asking - likely an oversight and a one-line fix.
3. **"Harness" is fine, earlier collision concern withdrawn.** Agent harness is established generic
   vocabulary (Google Cloud has an explainer page, there is an arXiv paper on necessary and
   sufficient conditions for an agent harness, plus awesome-harness-engineering). Within GA4GH the
   definite-article usage now points at Fiume's profile, but with collaboration live, being read as
   part of that family is accurate and useful. The naming question that actually remains is product
   versus layer: clinicians log into a product, not a harness.
4. **Positioning.** The profile states it "does not ... make authorization decisions for data
   holders". That gap is exactly our project. We are the disclosure-control layer over the profile.
   Our "best practice to GA4GH" deliverable now has a concrete home: a privacy and budget extension
   to a draft that is early enough to influence.

Two concrete gaps to raise with Fiume while the profile is at v0.1. Both block the budget layer:

- **The `query` field is an opaque passthrough** (`additionalProperties: true`, described only as "a
  Beacon request entity matching the advertised Beacon version"). A disclosure-control layer cannot
  compute predicate specificity - one of our three budget components - from an opaque blob. We would
  have to parse the Beacon request ourselves, defeating the point of a protocol-neutral operation
  model. Ask for enough structure to meter query narrowness.
- **Granularity is not first-class.** Beacon's three-tier access model (boolean / count / record) is
  the main privacy dial in the spec, and in the current schema it is buried inside that opaque query
  object. For any privacy profile it should be a top-level, enforceable field on the operation.

Also minor: the operation is named `ga4gh.beacon.variant.query` but `entry_type` accepts
`individuals`, `biosamples` and `analyses`. Our cohort work is phenotype-centric, so the
variant-centric name undersells what the operation already supports.

## Noisegate (reviewed 2026-09-04)

The closest prior art found for the budget layer. Much closer than AskBeacon.

- Repo: https://github.com/yashmahajan10/llm-differential-privacy-gateway - "Noisegate: a
  differential privacy gateway that lets an untrusted LLM agent query sensitive data over MCP".
  Apache-2.0, Python, ~28 stars, created 2026-06-30, active. Has DESIGN.md, CITATION.cff, 250+ tests.
- What it does: MCP stdio server for Claude Desktop. Agent gets structured tools (`count`, `sum`,
  `average`, `histogram`, `get_budget`) whose schemas are generated from a dataset policy. A DP
  engine executes under a tracked budget and returns noisy answers with stated confidence intervals.
  Too-narrow queries are rejected at the trust boundary. Budget exhaustion returns a refusal rather
  than a quieter answer.
- Design stance, which is ours almost word for word: "the privacy guarantee does not depend on the
  LLM being trustworthy ... the model proposes a query, it enforces nothing. Every privacy property
  is enforced downstream." Enforcement lives in trusted code the agent cannot reach.
- Rigour we should match or beat: noise mechanism verified against OpenDP to 1e-9 across 35
  noise-scale checks. Hybrid zCDP composition admits 308 queries where advanced composition gives
  268 and naive sum-of-epsilon gives 100. Guarantee is (epsilon, delta)-DP, not pure epsilon.
- **Attack gallery pinned in CI** - differencing, membership inference, singling out by
  re-identification. Each shown succeeding with privacy off and defeated with privacy on, checked on
  every build "so the defense cannot quietly rot". This is exactly the shape our Arm 2 deliverable
  should take, and it is a better pattern than a one-off evaluation.

What it does NOT do, which is where our contribution sits:

- No view splitting. Single view, single user. Nobody sees the exact numbers, so there is no trusted
  clinician channel and no human-in-the-loop collaboration. This is our main novelty.
- No Beacon, no genomics. Generic tabular data (census, synthetic patients). No genomic semantics,
  no ontology walking, no allele frequency, and none of the Beacon-specific attack literature.
- Attacks are the classic brute-force loops. No adaptive agent that adapts its next query on what it
  just learned, which is the threat model this project exists to study.
- No categorical record-exclusion. It has budget and something like our specificity measure (the
  too-narrow rejection), so two of our three budget components have a reference implementation here.

Action: read DESIGN.md properly before designing our budget. Strong candidate for reuse of the DP
engine and composition accounting rather than building our own, which would free the project to
spend its time on view splitting, Beacon semantics and adaptive-attacker curves. Apache-2.0 makes
that legally straightforward. Worth citing regardless, and worth contacting the author.

## Licensing decision: Apache-2.0 (2026-09-04)

BlindBeacon is Apache-2.0, deliberately breaking from SACGF house style (cdot and hgvs are both MIT).
Copyright holder is `SACGF`, matching cdot's convention. Three reasons:

1. **Relicensing is a one-way door.** MIT code can be absorbed into an Apache-2.0 project, but
   Apache-2.0 code cannot be relicensed as MIT. Noisegate, our leading candidate for reusing a DP
   engine and composition accounting, is Apache-2.0. Under MIT we could depend on it freely but
   could not vendor or fork it without a mixed-license tree carrying its notices. Apache keeps both
   paths open, and we have not yet decided which we will take.
2. **The patent grant is what clears institutional review.** MIT grants copyright permission and
   says nothing about patents. Apache-2.0 carries an express patent grant from every contributor
   plus a retaliation clause. Our entire adoption thesis is other institutions deploying this, and
   their legal teams will be reviewing it. The express grant is unglamorous but it is the thing that
   gets a dependency through a hospital's review.
3. **Ecosystem match.** The GA4GH Agentic Harness Profile we are extending is Apache-2.0 with GA4GH
   copyright, and Noisegate is Apache-2.0. Matching the community we are contributing into removes
   friction we would otherwise have to explain.

Costs accepted: inconsistency with the other SACGF repos, a much longer licence, and downstream
obligations to mark modified files and propagate any NOTICE. Apache-2.0 is also incompatible with
GPLv2, which is irrelevant here since nothing in the stack is GPLv2.

No NOTICE file yet, and none is required. If we ever vendor Noisegate or any other Apache-2.0 code,
its attribution goes there.

**Open:** SA Pathology may have an intellectual property policy that overrides this. Worth a check
with whoever handles IP before the first PyPI publish, not after.
