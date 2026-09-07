# Architecture notes

Working decisions from the 2026-09-04 design discussion. Nothing is built yet, so these are
positions with reasons attached, not commitments. Open questions are marked at the end.

## The shape

Four pieces, only two of which we write.

| Piece | Build or adopt | What it is |
|---|---|---|
| Beacon v2 instance | Existing | VariantGrid, RUNX1db, plus sBeacon and DNAstack's for cross-testing |
| Plain Beacon MCP (control arm) | **Adopt** | Fiume's `ga4gh-agentic-harness-mcp`, unmodified. See notes.md |
| Disclosure gateway (treatment arm) | **Build** | Speaks Beacon REST upstream, MCP downstream. Fuzz, meter, audit |
| Clinician webapp | **Build** | Hosts the conversation, renders both views |

The gateway and the webapp share a Beacon client and a set of tool schemas. Everything else is
someone else's code.

## Why the webapp is structural, not cosmetic

MCP cannot do view splitting. Anything an MCP server returns becomes model context by construction,
and there is no channel in the protocol for "show this to the human but not the model" - even
elicitation goes through the client, which is the model's host.

So the clinician's exact-counts view has to live outside the MCP transport entirely. The two
channels are not two MCP channels. The interceptor sees the unblinded Beacon response and forks it:

- fuzzed values, ranges -> MCP tool result -> model context
- exact counts, phenotypes, ages -> server-sent events -> clinician's browser, never near the model

That fork is the whole design. The webapp is the second channel, which is why it is load-bearing
rather than a nicety on top.

## Intercept at Beacon REST, not MCP-to-MCP

Tempting to make the gateway an MCP client of some upstream Beacon MCP. Do not.

You cannot fuzz what you cannot parse, and wrapping someone else's evolving tool surface gives us a
large and unstable surface to reason about. Beacon v2 REST is itself the standard, so talking to it
directly is just as general and considerably more stable. The gateway still behaves as a proxy from
the agent's point of view; it just does not route through an intermediary MCP layer to reach data.

Downstream we expose the profile's canonical operation name (`ga4gh.beacon.variant.query`) so that
the control and treatment arms present the same tool surface to the agent. If the two arms differ in
tool naming as well as in fuzzing, the experiment is confounded and the curves mean nothing.

## The gateway is its own principal upstream

Raised by Jordi Rambla, 2026-09-07: Beacon implementations already carry an optional budget, per IP
or per authenticated user, so what happens when an agent is added and two budgets are in play?
`claude/budget_notes.md` works the question through; the architectural consequence is here.

The gateway authenticates to the upstream Beacon as its own service principal, and carries the
clinician's identity in-band for the audit log rather than in the upstream credential. It must not
present the clinician's own Beacon credentials.

The reason is not tidiness. Raisaro-style budgets, including the deployed Portuguese Beacon one,
meter per `{user, subject}` pair and enforce by **silently occluding** subjects once their budget is
spent. If the agent's exploratory queries land on the clinician's upstream account, then the
clinician's own exact view, which this design promises is unmetered, is silently degraded by an
agent working on their behalf - and both views end up computed over a data-dependent subset that
neither view reports. The measured cost is not marginal: the Portuguese Beacon sees the first
occlusion after 8.3 queries at N=25,000 and 32.5 at N=100,000, putting 67,741 samples on the order of
twenty, against an agent that walks ontologies.

Where two meters unavoidably cover the same pipe, the effective limit is the **minimum** of the two,
never the sum. Summing turns two budgets into twice the budget and makes the agent route a way to
buy budget rather than a way to control it.

## Host the conversation server-side

The webapp should call Claude itself rather than have clinicians point Claude Desktop at our MCP
server. Four reasons, the first being the one governance will care about:

1. **Otherwise the audit claim is unprovable.** If the clinician's own client talks to Anthropic,
   they can paste patient notes straight into the chat and our logs never see it. We promised logs
   that prove no record-level data reached the model. That is only provable at an egress point we
   own.
2. **Budget survives.** A clinician cannot reset a session by restarting a desktop client.
3. **One swappable egress.** If SA Health mandates Bedrock, we change a client class.
4. **Identity.** We know which clinician spent which budget.

The red team and the workshop control arm do not need the webapp. They drive the MCP server
directly, which is the cheaper path for both.

## Do not use the server-side MCP connector

Checked 2026-09-04: the Anthropic MCP connector is beta on the first-party API and **not supported
on Bedrock or Vertex**. Messages, streaming, tool use and prompt caching are available on all
platforms.

Since Bedrock is a live possibility for governance reasons, the gateway must run its own tool loop
and act as an MCP client in-process rather than handing Anthropic a URL. Then first-party and
Bedrock behave identically and the provider becomes a one-line swap.

## Fuzzing: default-deny

Whitelist the fields that may reach the model. Never blacklist.

Beacon implementations vary in what extra fields they return, and under our own "assume anything
sent to Anthropic is stored forever" constraint, a blacklist that misses one new field leaks
permanently. Default deny is the only posture that survives an upstream adding a column.

## The clinician is a leak channel

If the clinician can free-type to Claude, they can type the exact count. This directly threatens the
audit deliverable, so it needs a design answer rather than a hope.

Proposal: make disclosure an explicit act. A "tell Claude" control moves a specific fact from the
trusted pane into the conversation and logs it as a declassification, with the clinician named. The
audit claim then becomes provable and precise: *no record-level data reached the model except these
logged, clinician-authorised disclosures.* That is a stronger and more honest claim than the
unqualified version, and it is one a governance committee can actually check.

Mid-conversation system messages are the right transport for declassified facts and budget notices.
They are the prompt-injection-safe operator channel and they do not invalidate the cached prefix,
which matters because sessions are long and the system prompt will be large.

## Distribution

- **Core on PyPI** - Beacon client, fuzzing layer, budget accountant, audit log. Beacon-generic, no
  VariantGrid dependency, no Django. This is what another Beacon operator adopts, and `bycon` being
  on PyPI shows this community installs that way.
- **Gateway MCP server** - a console entry point in the same package, so `uvx` works.
- **Webapp as a container.** Clinicians will not `pip install` anything or hand-edit MCP JSON.

Django for the webapp: the team knows it, it deploys inside the hospital network the way VariantGrid
already does, and async views handle streaming fine. Keep it out of the core package.

## The budget is denominated in bits

**`claude/budget_notes.md` is the authority on the budget design.** This section records only the
naming and framing decisions that sit around it.

Not in epsilon, and not in a coined unit. The quantity being spent is information that narrows down
who is in the database, and that is measurable in bits.

This makes the three budget components share a currency, which epsilon alone does not:

- **Epsilon-style accounting on counts.** Epsilon bounds the log-odds an adversary can gain, so it
  converts to bits (nats / ln 2).
- **Specificity on predicate narrowness.** A predicate that narrows the field to one in N conveys
  log2(N) bits. This is already a bit measure; it just has not been called one.
- **Categorical record-exclusion.** Not a currency. It stays a hard gate: a query that would put any
  individual over the per-individual bit ceiling is refused rather than priced.

The governance win is that the number means something concrete to a non-specialist. VariantGrid holds
67,741 samples, and log2(67,741) is about 16.05, so roughly 16 bits singles out one person. That
makes a budget legible to a committee in a way no epsilon ever will. `budget_notes.md` works the
specificity formula and a worked 40-bit session through against real HPO terms.

It also puts the budget and the evaluation metric in the same units. The project measures success partly
as the proportion of synthetic patients re-identified at each restriction level, so denominating the
budget in re-identification bits makes the attacker-cost curves directly interpretable against it.

Precedent worth citing: EFF's Panopticlick work on browser fingerprinting measured uniqueness in bits
of identifying information. Same framing, different domain, and it makes ours legible rather than
novel, which is what we want in front of a governance committee.

**On the name "beaconage".** Beaconage is the historical toll paid for the upkeep of beacons. It was
considered as the name for this budget and rejected: a toll flows *to* infrastructure and depletes
nothing from anyone, whereas a privacy budget is depleted *from* data subjects. It was then
considered as the name for the whole project and also rejected, because it names the metering half,
which Noisegate already occupies, rather than the blinding half, which is the novel part. It remains
approximately right for one thing: a standalone disclosure-accounting library, which is purely a
metered-access layer. If the accountant is ever split out of this package - the pattern fits other
governed clinical data besides variants - `beaconage` is the name to use, and it
was free on PyPI and GitHub as of 2026-09-04. Do not register it speculatively; PyPI has no name
reservation and placeholder projects are reclaimable under PEP 541.

Two honest caveats:

- **Bits do not simply add.** Under naive composition they do, but advanced and zCDP composition are
  tighter, which is the whole reason Noisegate gets 308 queries where naive summing gets 100. The
  displayed figure must come from the composition accountant, never from summing per-query costs.
- **The two bit measures answer different questions.** Predicate specificity is about what the query
  reveals structurally; epsilon is about what the answer reveals. Adding them is defensible as a
  worst case but is not rigorous, and the paper should say so rather than imply a single clean
  quantity.

## Open questions

- Does the bit denomination survive contact with the real composition accounting, or does the
  epsilon-to-bits conversion turn out too lossy to display? If so, show the three numbers separately.
- Reuse Noisegate's DP engine and composition accounting, or build our own? Apache-2.0 makes reuse
  legal and it is verified against OpenDP. Leaning reuse. Read its DESIGN.md first. See notes.md.
- How much structure does `ga4gh.beacon.variant.query` need before we can meter predicate
  specificity? Raised with Fiume; blocks the budget layer.
- Does the clinician approve each query before it spends budget, or only review after? Approval is
  safer and slower. Probably a per-deployment setting rather than a fixed choice.
- Session pairing if we ever do support Claude Desktop alongside the webapp. Deferred; not needed
  for any of the three evaluation arms.
- Upstream budgets during cross-testing. sBeacon and DNAstack's may run their own budgets, and
  silent occlusion moves the ground truth underneath Arm 2's re-identification rates. We cannot
  currently detect occlusion, by design of the mechanism, so either it is disabled for the runs or
  B3's numbers are measured against a shrinking cohort. Settle before B3, not during.
- Whether the admin-facing surface can be reduced to a named restriction level plus a handful of
  steward declarations. `budget_notes.md` now asserts that a component which cannot be reduced to a
  preset does not ship, and the three-part budget as written does not yet pass that test.
