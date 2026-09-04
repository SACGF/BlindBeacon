# Name suggestions

Working title: "Fuzzed Beacon Agent Harness". Decision deferred.

Availability was checked on 2026-09-04 for ~330 candidates against PyPI (exact package), GitHub
(exact-name repos and stars, plus user/org name), PubMed (title hits, as a proxy for how badly the
word is overloaded in the clinical literature), `.com`/`.io` DNS, and a web search for clashes with
genomics, privacy, or LLM-agent tools. "Free" below means no PyPI package, no meaningful GitHub
repo of that name, and no product or paper using it. `.com` is taken for almost every real English
word, so it is only noted when free.

Side finding while searching: `yashmahajan10/llm-differential-privacy-gateway` ("Noisegate") is a
differential-privacy gateway that lets an untrusted LLM agent query sensitive data over MCP under a
tracked budget, with an attack gallery. Close prior art for the budget layer; worth reading before
the design doc is written.

## Verdicts on the original shortlist

| Name | Verdict | Why |
|---|---|---|
| **Squint** | Avoid | SQUINT is a published Bioinformatics-journal alignment editor (2007). `squint-cljs/squint` has ~900 stars, PyPI `squint` is taken. The joke survives as **Squintbeacon** (free everywhere, including `.com`). |
| **Penumbra** | Avoid | PyPI taken, `nealmckee/penumbra` 1.4k stars, Penumbra is also a privacy-focused blockchain, and "penumbra" appears in ~900 PubMed titles (stroke). Ungoogleable. |
| **Parallax** | Avoid | 34k GitHub repos, 16k-star library. |
| **Chaperone** | Avoid | PyPI taken, `uber-archive/chaperone` 635 stars, CHAPERONg is a bioinformatics tool, and "chaperone" is a protein class. |
| **Foglight** | Avoid | Quest Software's database-monitoring product. |
| **Dimmer** | Avoid | DiMmeR is a Bioinformatics-journal methylation tool; PyPI taken. |
| **Fogbeacon** | Usable | Free on PyPI, GitHub and `.io`; `.com` taken. But Foglight, Fogsight (2.5k stars) and Foglamp (341 stars) crowd the "fog" space. |

## Recommendations

Grouped by theme. Within each group, roughly in order of preference.

### Blinded agent (the "secret agent" reading)

| Name | Idea | Availability | Notes |
|---|---|---|---|
| **BlindBeacon** | A Beacon the agent queries blind. | PyPI free, `.io` free, one dead 2016 Java repo with 0 stars, `.com` taken. | Front-runner. Says "Beacon" to the GA4GH audience and "blinded" to the trial audience in one word. Googleable as one word. |
| **Blindcount** | The agent only ever sees counts, and sees those blind. | Free everywhere, including `.com`. | Most literal. Slightly plain. |
| **Blindtally** | Same idea, warmer word. | Free everywhere, including `.com`. | |
| **Blindlantern** | A dark lantern (shuttered so the bearer sees without being seen) with the sides reversed. | Free everywhere, including `.com`. | `Darklantern` itself is taken by an FHE deep-learning library. |
| **Shutterlamp** | A lamp with the shutter mostly closed. | Free everywhere, including `.com`. | |
| **Askance** | "To look askance": a sidelong, distrustful glance. | PyPI free, `.io` free, one 4-star repo. | The squint joke plus the untrusted-agent framing. Reads well in a paper. |
| **Halfblind** / **Nightblind** | Partially sighted agent. | Halfblind: 0-star repos only. Nightblind: 0-star repo, `.io` free. | |
| **Blindfold** | The image on the whiteboard. | Avoid: `blindfold-dev/Blindfold` is a PII-before-LLM redaction product with an MCP server. Too close. | |

### Spy tradecraft

| Name | Idea | Availability | Notes |
|---|---|---|---|
| **Chickenfeed** | Spy jargon for genuine but low-grade intelligence deliberately fed to the other side. | PyPI free, `.io` free, one 1-star repo. | Exactly what the harness hands Claude. Memorable and funny; a committee may find it flippant. |
| **Bariummeal** | The technique of giving each suspect a different version to trace a leak, named after the radiology test. | Free everywhere, including `.com`. | View splitting and a clinical pun in one. Two-word awkwardness as a package name. |
| **Canarytrap** | Same technique, better-known name. | PyPI taken (a text-watermarking package), 11-star repo. | Weaker availability than Bariummeal. |
| **Lamplighter** | le Carré's surveillance and support crew; also someone who lights beacons. | PyPI free, 6-star repo, `.com`/`.io` taken. Minor clash with a Philips Hue automation script. | Light theme and spy theme at once. |
| **Sandbagger** | Deliberately under-representing what you hold. | PyPI free, 6-star repo, `.io` free. | The Beacon sandbags its answers. |
| **Cutout** | The trusted intermediary who knows source and destination but nothing else. | Avoid: `uoguelph-mlrg/Cutout` is the well-known ML augmentation paper. | |
| **Deaddrop**, **Safehouse**, **Sentry**, **Watchman**, **Sleeper**, **Circus** | | All heavily taken. | |

### Beacon and lighthouse lore

| Name | Idea | Availability | Notes |
|---|---|---|---|
| **Beaconage** | A real word: the toll paid for the upkeep of beacons. | PyPI free, `.io` free, no GitHub repos, `.com` taken. | Beacon plus budget in one obscure real word. The bit-budget is the beaconage you pay per query. Strongest of the "serious" options. |
| **Pharology** | The study of lighthouses. | Free everywhere except `.com`. | Serious, distinctive, googleable. Does not say "restricted". |
| **Occulter** | The disc in a coronagraph that blocks the bright star so faint companions can be seen. | PyPI free, `.io` free, no exact repo. | Block the bright individual record to see the cohort. Also "occulting light" is a lighthouse term. |
| **Belisha** | The flashing orange beacon at UK pedestrian crossings. | PyPI free, 0-star repo, `.io` free. | Quirky, very googleable. |
| **Diaphone** | A type of foghorn. | PyPI free, 0-star repos, `.io` free. | Sound rather than light: the Beacon you can hear but not see. |
| **Squintbeacon** | Keeps the original joke, avoids the SQUINT clash. | Free everywhere, including `.com`. | |
| **Fuzzbeacon** / **Fuzzedbeacon** / **Dimbeacon** / **Halfbeacon** | Literal. | All free everywhere, including `.com`. | Safe fallbacks. Fuzzedbeacon matches the working title. |
| **Hazebeacon** / **Mistbeacon** / **Beaconhaze** | Fog family without the Fog- clashes. | All free everywhere, including `.com`. | |
| **Tipara** | Tipara Reef lighthouse, Spencer Gulf, SA. | Free everywhere, including `.com`, but there is a football-tips app of that name. | Local flavour for an SA Pathology project. |

### Seeing only a little (the "glimpse" reading)

Bare **Glimpse**, **Glimmer** and **Glint** are all blocked: GLIMPSE is the widely used low-coverage
genotype imputation tool (`odelaneau/GLIMPSE`), GLIMMER is Salzberg's gene finder, GLINT is a
methylation toolset. Compounds are fine.

| Name | Idea | Availability | Notes |
|---|---|---|---|
| **Halfglimpse** | Only half a glimpse. | Free everywhere, including `.com`. | |
| **Glimpsebeacon** / **Beaconglimpse** | A glimpse of the Beacon. | Free everywhere, including `.com`. | |
| **Glimmerbeacon** / **Glintbeacon** / **Peekbeacon** | Same pattern. | Free everywhere, including `.com`. | |
| **Sideglance** | A sidelong look. | PyPI free, 0-star repo, `.io` free. | |
| **Simmerdim** | Shetland word for the midsummer twilight that never quite goes dark. | Free everywhere except `.com`. | Obscure, lovely, hard to spell from hearing. |
| **Dimpsy** | Devon dialect for dusk. | PyPI free, 0-star repo, `.io` free. | |
| **Glance**, **Peek**, **Gleam**, **Wink**, **Blink**, **Sliver** | | All heavily taken (Glance 37k stars, Peek 10k, Sliver is a C2 framework). | |

### Blur, dimming and optics

| Name | Idea | Availability | Notes |
|---|---|---|---|
| **Apodize** | Optics: a filter that softens the hard edges and ringing of an image; literally "to remove the foot". | Free everywhere, including `.com`; one 8-star repo. | The technical term for exactly what fuzzing does to a count. Serious and obscure. |
| **Bedim** | To make dim. | PyPI free, `.com` and `.io` free, one 38-star repo. | Short verb: "bedim the Beacon". |
| **Nubilum** | Latin: cloudy weather, obscurity. | Free everywhere, including `.com`. | |
| **Adumbrate** | To give a faint outline; also to overshadow. | PyPI free, `.io` free, 0-star repo. | Claude gets the adumbration, the clinician gets the drawing. |
| **Dichoptic** | Presenting a different image to each eye. | PyPI free, 0-star repos, `.io` free, ~280 PubMed titles (vision science). | The precise term for view splitting. Haploscope, the instrument that does it, is a genomics tool already. |
| **Skiagram** | Old word for an X-ray shadow picture. | PyPI free, 0-star repo, `.io` free. | Clinical and shadowy. |
| **Scumble** | Painting: a thin opaque layer that softens what lies beneath. | PyPI free, `.io` free, 1-star repo; a Java ML library uses SCUMBLE. | |
| **Stenopeic** | Ophthalmology: pinhole. | PyPI free, 0-star repo, `.io` free. | |
| **Groundglass** | Frosted glass; also the radiology sign. | Free everywhere, including `.com`, but ~1,150 PubMed titles. | Ungoogleable in a clinical audience. |
| **Scotoma**, **Bokeh**, **Sfumato**, **Umbra**, **Obscura**, **Pinhole** | | Taken or overloaded (Bokeh 20k stars, Obscura 24k, Sfumato is a CSS tool). | |

### Bit budget

| Name | Idea | Availability | Notes |
|---|---|---|---|
| **Beaconage** | See above. | | Best budget name. |
| **BeaconBitBudget** | Fully descriptive. | Free everywhere, including `.com`. | Works as a subtitle or spec name, not as a brand. Bitbudget alone is taken (an embedding-compression benchmark). |
| **Fuzzbudget** / **Bitstint** / **Blindbits** | Coined. | Free everywhere, including `.com`. | |
| **Pittance** | A meagre allowance. | PyPI free, no exact repo, `.io` free. | What Claude is given. |
| **Skinflint** / **Tightfist** | A Beacon that will not give anything away. | PyPI free, low-star repos, `.io` free. | |
| **Miser**, **Nibble**, **Morsel**, **Trickle**, **Purse**, **Abacus**, **Parsimony** | | All taken; Parsimony is also a phylogenetics term. | |

## Leaning (updated 2026-09-04)

Two names are needed, not one, and they do different jobs.

**The product.** A clinician-facing webapp, so the name goes on a login page and gets said aloud in
clinics. That rules out the obscure real words no matter how much they reward looking up. Leaning
**BlindBeacon**: says "Beacon" to the GA4GH audience and "blinded" to the trial audience in one
word, and it is essentially free (PyPI and `.io` open, the only GitHub hit a dead 2016 Java repo
with no description and no stars).

**The unit.** **Beaconage** is the real word for the toll paid to maintain beacons, and it works far
better as the budget denomination than as a product name. The UI reads "you have spent 40 of your
100 beaconage on this session." That gives a coinable term of art for the paper and the GA4GH deck,
which is where the cleverness pays, while keeping the login page legible.

Still live if the above is rejected: **Chickenfeed** for the joke, **Apodize** or **Askance** for an
obscure-but-apt single word, **Fuzzedbeacon** as the literal fallback.

On "harness": an earlier concern that it collides with the GA4GH Agentic Harness is **withdrawn**.
Agent harness is established generic vocabulary, and with Fiume collaborating, being read as part of
that family is accurate rather than confusing. See notes.md. The distinction that does matter is
product versus layer - "harness" describes our architecture, but clinicians log into a product.

Whichever is chosen, register the PyPI name and GitHub org the same day; several of the free ones
are single-word English and will not stay free.
