# Attacks on GA4GH Beacons: a literature review

Compiled 2026-09-04 for BlindBeacon. Full citations with DOIs are in
[`bibliography.md`](bibliography.md).

**On sourcing.** Around 40 papers were read in full text, mostly through the Europe PMC REST API,
PoPETs' own open PDFs, bioRxiv preprints, ar5iv renderings of arXiv preprints, and eleven paywalled
PDFs supplied by the team. A further set were blocked by publisher paywalls, cookie walls or PDF
encodings our tooling could not decode, and are used at abstract level only. Every entry in the
bibliography carries a confidence tag saying which of the two it is, so the bibliography is also
the list of what has not been read. Nothing marked `[U]` should be cited.

The supplied PDFs fall in two groups. Five were Beacon attack papers, and sections 2.1, 3.3, 3.7
and 5 are written against those source texts. Six were the statistical-database theory underneath,
and sections 4.3, 4.4, 5.4 and 5.5 are written against those. The PDFs themselves are not kept in
this repository, since most are under publisher copyright and the repository is public.

Reading the sources changed the review both times, and the second round changed it more. Two claims
that had come from secondary summaries turned out to be wrong rather than merely imprecise: what
simulatable auditing actually requires of a refusal decision, and what the census reconstruction
paper actually reports. Both were load-bearing, one for the budget design and one for governance
material. The failure was the same both times, and it is worth stating as a rule for anyone
extending this review: **a summary preserves the headline and drops the condition attached to it.**
Read the theorem, not the abstract, and quote definitions rather than paraphrasing them.

This review covers three bodies of work that do not usually cite each other:

1. **Beacon-specific attacks and defences**, 2015 to 2025. Small, self-contained, and mostly
   produced by four groups (Bustamante at Stanford, Ayday and Cicek, Malin and Vorobeychik at
   Vanderbilt, Ohno-Machado and Hubaux).
2. **Statistical database theory**, 1979 to now. The tracker, the Dinur-Nissim reconstruction
   bound, query auditing, differential privacy and its composition theorems. The Beacon attacks
   are special cases of results proved decades earlier.
3. **Agents and language models as adversaries**, 2023 to now. Almost entirely disjoint from the
   first two.

Two headline findings. **Every published Beacon defence that has been tested against an adaptive
attacker has failed**, and the one that survives is the one that adapts back. The canonical budget
defence, which is the direct ancestor of BlindBeacon's, has been broken twice by two different
groups using two different mechanisms, described in sections 3.3 and 5.1. And **nobody has yet run
a language model against a Beacon**, in either direction.

---

## 1. What a Beacon discloses

A Beacon answers "is there a genome in your dataset with allele A at position P?". Beacon v1
returned a boolean. Beacon v2 (Rambla et al. 2022) added a data model and three response
granularities, which the Portuguese Beacon paper states cleanly:

- **Boolean** - yes if any record satisfies the query.
- **Count** - the number of records that satisfy it.
- **Record** - additional phenotypic or medical information about matching records.

Access is separately tiered public / registered / controlled, and the two dials are independent:
a dataset can be boolean-public, count-registered, record-controlled. This is the tiering
BlindBeacon hangs its two views off, so it is worth being precise that the spec offers
*granularity of response* and *level of access* as orthogonal controls, and says nothing about
metering either.

Two things make a Beacon leakier than it looks.

- Beacons are usually built around a phenotype. Shringarpure and Bustamante make this the whole
  point of the attack: of the nine beacons then indexing non-public data, four were single
  phenotype. Their example is the SFARI beacon, where membership means "belongs to a family where
  someone has autism spectrum disorder". Membership *is* the sensitive attribute.
- Beacon v2 added metadata and filtering-term queries. The Portuguese Beacon authors flag that
  "the risk of re-identification is aggravated by access to metadata queries, which were
  introduced in Beacon v2 and are not contemplated in the aforementioned studies." Every attack
  paper below predates or ignores the richer v2 query surface. **The published attack literature
  is against a weaker API than the one BlindBeacon will expose.**

---

## 2. Prehistory: aggregate genomic statistics were never safe

Homer et al. (2008) showed that an individual's presence in a DNA pool can be detected from
aggregate allele frequencies alone, even at trace concentration. NIH withdrew aggregate allele
frequency data from open access in dbGaP as a direct result. Sankararaman et al. (2009) supplied
the power analysis: how many SNPs, at what allele frequencies, against what pool size.

Wang et al. (CCS 2009) sharpened it twice over. First, using the coefficient of determination r²
rather than raw frequencies, membership can be detected from "a couple of hundred SNPs" instead of
Homer's ten thousand or more. Second, integer programming over published pairwise correlation
statistics recovers hundreds of participants' actual genotypes. Their own summary is that the
Homer threat "was found in our research to be significantly understated". Cai et al. (2015) later
gave a *deterministic* re-identification from published GWAS results, pinpointing at least one
patient from statistics over 25 randomly chosen loci on real Wellcome Trust data.

Zhou et al. (ESORICS 2011) is the most useful of these for BlindBeacon's budget design. They frame
aggregate-release risk as a search problem: the attack succeeds when the **solution space is
small**, meaning few candidate genotype assignments are consistent with the published statistics.
That is the same quantity BlindBeacon's specificity measure prices. It is prior art for the idea
that predicate narrowness, not answer magnitude, is the thing to charge for.

### 2.1 The calibration warning

Two 2009 papers are worth reading before anyone on this project reports an attack success rate.

**Sankararaman, Obozinski, Jordan and Halperin** did something more useful than another attack.
They derived an **upper bound on the power achievable by any detection method**, via the
Neyman-Pearson likelihood ratio test, which attains maximal power by construction. Their motivation
is worth quoting because it is the epistemic posture BlindBeacon's evaluation should aim at:

> although an analysis of any specific detection method provides an estimate of its power of
> detection, it remains possible that another method could provide increased power and that
> therefore no guarantee could be provided that its power of detection was below some acceptable
> level. What is needed is an upper bound on the power achievable by any method.

For m common SNPs in linkage equilibrium exposed from a pool of n individuals, the relation between
m, n, the false positive rate alpha and the power 1-beta is

```
z_alpha + z_(1-beta)  ~  sqrt(m/n)
```

valid for pools above 100 and common SNPs (MAF above 0.05). Two consequences they draw out. The
number of SNPs that can be safely exposed is **linear in pool size**. And the power does not depend
on the specific allele frequencies at all, provided the MAFs are large enough, so the bound is
population-independent. Their recommended protocol is to drop everything with MAF at or below 0.05,
keep a subset in linkage equilibrium, and then use the LR test to choose what to expose. They
shipped it as a tool, SecureGenome.

**Visscher and Hill** published the same optimal likelihood ratio result independently and almost
simultaneously, expressed as power scaling with m/N, markers over reference sample size. For 80%
power at alpha=0.05 you need m/N around 6; for 99% power at alpha=1e-6, around 50.

The transferable idea for us is the *shape* of the guarantee, not the formula, which applies to
allele frequency release rather than to Beacon responses. A committee asking "how safe is this"
wants a bound over all attackers, not a score against the attacks we happened to run. Arm 2
produces the latter. Where a bound of this kind can be derived for the fuzzed count tier, it is
worth more than another attack transcript.

**Braun et al., "Needles in the haystack"**, audited Homer's statistic rather than extending it,
and found its false positive rate is far above nominal on real data. A mean allele frequency
difference between reference and mixture, or average pairwise SNP correlation as low as
rho = 0.002 across 50,000 SNPs, is enough to shift the statistic so that true non-members are
routinely "detected" at nominal p below 1e-6. Their conclusion is that the test is reliable only
under near-ideal conditions and is better at confirming absence than presence.

This is a direct methodological requirement for Arm 2 of `overall_plan.md`. A re-identification
rate measured against a synthetic cohort is meaningless without a matched true-negative arm, and
the attacker's own claimed confidence is not evidence. The synthetic instance must contain
individuals the attacker does not hold, and the reported metric must be the false positive rate at
a fixed threshold, not the count of confident-looking hits.

### 2.2 The taxonomy worth adopting

Erlich and Narayanan (2014) give the standard three-route framing that the rest of this review
implicitly uses:

- **Identity tracing** - exploiting quasi-identifiers to unmask an anonymous record. Gymrek et al.
  (2013) is the canonical example: Y-STR haplotypes recovered from a genome, matched against
  recreational genealogy databases, yield the correct surname for about 12% of US Caucasian males,
  and surname plus age plus state narrows to a median of **12 candidate individuals**.
- **Attribute disclosure** - linking a known identity to a sensitive phenotype. This is what Beacon
  membership inference actually is, since beacons are organised around phenotypes.
- **Completion** - inferring the masked parts of an already-identified individual's genome. The
  Beacon reconstruction attacks in section 3.4 and 3.5 sit here.

Using these three labels in BlindBeacon's own write-up will make it legible to the genomic privacy
community without further explanation. Mittos, Malin and De Cristofaro's systematisation adopts the
same three-way split rather than inventing its own, which is good evidence it is the field's
working vocabulary.

Naveed et al.'s survey was read as the alternative candidate and it is not a better frame for this
review. Its attack section splits into re-identification, phenotype inference and other, and it
files Homer-style membership inference under phenotype inference, on the reasoning that being in
the case group of an HIV study *is* a phenotype. That holds for a case-control study and fails for
a Beacon, where membership in the queried dataset may carry no phenotype at all. It also uses
Erlich and Narayanan's own vocabulary in passing, calling linkage-disequilibrium inference
"completion attacks", so the two are the same three mechanisms under different labels. Naveed's own
organising contribution is orthogonal and it is a dataflow framework: ten pipeline stages from
biospecimen to recreational genomics, each with a threat model. A Beacon does not appear on that
pipeline, and the nearest stage assumes a one-shot aggregate publication rather than an interactive
service. The survey predates Shringarpure and Bustamante and contains no Beacon or GA4GH content
at all.

Where it is worth citing is elsewhere, and section 2.3 uses it there.

### 2.3 What the field says about itself

Mittos et al. (PoPETs 2019) systematised 197 genome privacy papers. It is worth knowing what it is
and is not: it deliberately **excludes attack papers**, covering them only as background, and
systematises privacy-enhancing technologies instead. So it is not a source on Beacon attack
mechanics, but it is the field's own audit of its methods, and four of its findings bear on this
project.

**Long-term security is the field's biggest unsolved problem, and it is our design constraint.**
Their P1 challenge scored highest for importance among the self-identified experts they surveyed,
six of seven scoring it 10 out of 10. Their reasoning is the same as `CLAUDE.md`'s:

> The longevity of security and privacy threats stemming from the disclosure of genomic data is
> substantial for several notable reasons. First, access to an individual's genome allows an
> adversary to deduce a range of genomic features that may also be relevant for her descendants,
> possibly several generations down the line. Thus, the sensitivity of the data does not
> necessarily degrade quickly, even after its owner has deceased. Moreover, the full extent of the
> inferences one can make from genomic data is still not clear.

And the consent point, which governance will care about:

> the threat model under which a volunteer decides to donate their genome to science, or have it
> tested by a DTC company, is likely to change in the future. As a consequence, the need or desire
> to conceal one's genetic data might evolve.

They make the point through cryptography, noting that key sizes are chosen assuming thirty to fifty
years of security, and that encrypted traffic can be harvested now and decrypted later. Our version
of the same argument is simpler and, if anything, stronger: nothing needs to be broken, because the
data goes to the model provider in the clear and the only question is what a future model can infer
from it.

**Noise alone does not stop membership inference.** They cite Dwork et al. (2015) for the result
that "membership attacks can be successful even if aggregate statistics are released with
significant noise". Fuzzing is not a substitute for metering.

**Differential privacy in the clinic is contested, not merely costly.** Their headline cautionary
case is Fredrikson et al.'s study of a differentially private model for personalised warfarin
dosing, where "to effectively preserve privacy, the resulting utility of the model would be so low
that patients would be at risk of strokes, bleeding events, and even death". In their expert
survey, four respondents who identified as knowledgeable "rejected the use of differentially
private mechanisms in clinical settings" outright. One asked for exactly the artefact this project
is proposing to produce: "A comprehensive, experiment-driven analysis for feasible epsilon and
delta values can pave the way towards more formally founded utility results for DP." They also
flag the cultural problem, that it is "challenging to convince biomedical researchers, who are
striving to get the best possible results, to accept a non-negligible toll on utility".

The route out they credit is Tramèr et al.'s relaxed differential privacy, which buys utility by
**bounding the adversary's prior belief** rather than assuming it is unlimited. Note that Tramèr is
a co-author on Raisaro's Beacon budget paper, so the two ideas are already connected in that group's
work. Mittos et al. present this as the one move the field regards as having genuinely improved the
tradeoff, and it sits against our own constraint that the adversary is an all-knowing agent with
access to everything ever sent. That made it the review's one open design question. It has now been
read and it is settled in section 5.4: the lever is real, it is smaller than its reputation, and in
our regime it is worth close to nothing.

**The gap they name is the one this project is in.** Their P9:

> Current genomics initiatives, e.g., All of Us or Genomics England, primarily address privacy by
> relying on access control mechanisms and informed consent. In the meantime, we lack technical
> genome privacy solutions that can be applied to such initiatives.

On the two Beacon defences specifically, they are unimpressed: both Raisaro's and Wan's "make
important trade-offs with respect to utility", and Wan's approach, "acknowledging that utility loss
is unavoidable, provides ways to calculate (but not solve) this trade-off".

**Naveed et al. corroborate two of these independently, and one of them is the argument for view
splitting.** They polled 61 biomedical scientists and found that "the respondents are willing to
trade money and test time (duration) to protect privacy, but they usually do not accept trading
accuracy or utility". Their own verdict on differentially private GWAS release is that it "makes
data more noisy, which makes adoption of these approaches difficult" and that this "is particularly
problematic because biomedical researchers and physicians want more (not less) accurate data than
is available today". That is a measured version of the objection BlindBeacon's design answers by
not asking the clinician to accept any noise at all. It is worth citing when explaining why the
exact view exists.

They are also the better source for the longevity constraint in `CLAUDE.md`. They classify value as
a defining property of genomic data and note that this value "does not decline with time ... In
fact, this importance will likely increase with time", that "we do not know what will be inferred
from one's genomic data in the future", and, in the sharpest form of the argument, that "the
lifetime of genomic data is much longer than the lifetime of a cryptographic algorithm". Their
adversary is computationally bounded and unbounded in background knowledge, which is our assumption
minus the adaptivity. One caveat for anyone citing them: they cite Fredrikson et al. twice and
never report the warfarin mortality finding, so that argument needs the Fredrikson paper directly
and Naveed will not support it.

The lesson the Beacon designers took from this era was "release presence, not frequencies". The
next section is the discovery that presence is enough.

---

## 3. The Beacon attack lineage

### 3.1 Shringarpure and Bustamante 2015: membership from boolean answers

The founding result. A likelihood ratio test on the beacon's yes/no answers over many SNPs, under
Hardy-Weinberg equilibrium, a beta-distributed site frequency spectrum, and a sequencing error
rate. Deliberately *not* dependent on allele frequencies, which they present as the strength of
the test: it needs only the victim's genome.

| Setting | Cost of re-identification |
|---|---|
| Simulated beacon, 1,000 individuals | 5,000 queries |
| 65 CEU individuals from 1000 Genomes | 250 SNPs |
| A real beacon, Personal Genome Project genome | 1,000 SNP queries |

Relatives are detectable too. The authors are explicit that these bounds are conservative and
should not be used to size a beacon: "A re-identification test that uses only rare SNPs and/or
incorporates the allele frequencies at SNPs will be more powerful than our method and will require
fewer queries than our estimates." Raisaro et al. proved them right within two years.

Their mitigation discussion contains the sentence most relevant to us:

> Limiting the number and/or rate of queries per IP address can only slow down attackers and is
> therefore ineffective.

They also note that publishing beacon metadata such as sample ethnicity, beacon size or the names
of constituent datasets *reduces* security, and that multi-population beacons are less secure than
single-population beacons of the same size. Their preferred non-technical mitigations are a
minimum beacon size and no anonymous pings.

### 3.2 Raisaro et al. 2017: the optimal attack, and the first budget

Raisaro et al. added the missing ingredient, public minor allele frequencies, and reordered the
queries to hit rare variants first. The collapse in attacker cost is the single most important
number in this literature.

| Attack | Queries for confident re-identification |
|---|---|
| Shringarpure-Bustamante on a 1,000-person beacon | ~5,000 |
| Optimal attack, correct ancestry assumed | **3 SNPs** |
| Optimal attack, wrong ancestry (East Asian frequencies against Europeans) | 36 SNPs |

Three orders of magnitude, from one piece of public background knowledge. This is the concrete
form of the design constraint in `CLAUDE.md` that says to assume the attacker's capability only
grows: the same beacon, unchanged, went from "5,000 queries" to "3 queries" because the attacker
got smarter, not because the beacon got weaker.

They proposed three mitigations, which set the template everything since has worked within:

- **S1, minimum allele threshold.** Answer yes only if at least k individuals carry the allele.
  At k=2 re-identification power drops to near zero, but only 60% of alleles are shared by two or
  more individuals, so roughly 40% of the variants become unqueryable, and they are exactly the
  rare ones research wants.
- **S2, random flipping.** Flip responses on unique alleles with probability epsilon. At
  epsilon=0.15, attack power stays below 35% for about 6% utility loss, which dominates S1.
- **S3, per-individual query budget.** Each individual gets budget `-log(p)` for an acceptable
  type I error p. Each time an individual's genome contributes to a yes, their budget is reduced
  by `-log(1 - D_Ni)`, larger for rarer alleles. When an individual's budget is spent, they are
  dropped from that user's answers.

S3 is the direct ancestor of BlindBeacon's record-exclusion floor, and the shape matches: the
budget is per individual per user, it is spent in proportion to how rare the queried allele is,
and exhaustion means silent removal rather than refusal. Against real ExAC query logs, utility was
"completely preserved for the first 2,000 queries". Note the dependency S3 introduces: it requires
authenticated, non-colluding, single-account users, which contradicts the anonymous-ping model
Beacon was designed around.

### 3.3 von Thenen et al. 2019: correlation defeats the budget

The most important negative result for BlindBeacon. Two attacks exploiting linkage disequilibrium:

- **QI (query inference).** Build a network of SNPs in strong LD (r² above 0.7) and *infer* the
  beacon's answer to queries never asked. The attacker gets answers without spending budget.
- **GI (genome inference).** A fourth-order Markov chain trained on public reference panels
  reconstructs the victim's genotype at loci the operator deliberately censored for being rare.

Their Table 1, on a simulated 65-member CEU beacon where t is the MAF threshold below which the
victim's SNPs are hidden. The figure is the average number of queries needed to receive the first
negative response.

| t (SNPs below this MAF hidden) | SB attack | Optimal attack | QI attack | GI attack |
|---|---|---|---|---|
| 0 (nothing hidden) | 1,418 | 3 | 3 | n/a |
| 0.03 | 19,525 | 270 | 160 | **2** |
| 0.05 | 56,759 | 1,495 | 1,031 | **2** |

The often-quoted "5 queries" from the abstract is a different metric, the number needed to reach
95% confidence, and the paper reports both. Two things stand out. Censoring rare variants makes the
beacon *dramatically* harder for the attacks that respect it, from 3 queries to 1,495, and makes no
difference at all to GI, which infers the censored loci from public panels rather than querying
them. And the QI attack needs about 30% fewer queries than the optimal attack across the board.

Against the nine live beacons on the Beacon Network, using a Personal Genome Project individual as
the victim, most needed **1 to 4 queries**. AMPLab and 1000 Genomes Phase 3 were the exceptions at
20 to 250. For three beacons no negative response was found within 1,000 queries.

**The budget failure is quantified precisely, and the margin is one query.** Their worked example
uses HapMap individual NA12272 in a 65-member CEU beacon with nothing hidden:

- The optimal attack needs **7 queries** to re-identify.
- Raisaro's budget depletes after **6**, at which point the beacon starts returning false responses.
  The attack fails.
- The QI attack needs **5**. It succeeds, under budget.

Their conclusion: "a query budget that is merely based on the SNPs' MAFs and that does not consider
SNP correlations would fail to protect an individual's privacy. An attacker using the QI-attack
would not exhaust the budget but still be able to determine the victim's beacon membership."

That margin matters for how we read this. Raisaro's budget was not wildly miscalibrated. It was
correctly sized against the attack it was designed for and then beaten by a hair, because
correlation shaved two queries off the cost. A budget priced on the wrong quantity does not fail
loudly. It fails by a query or two, silently, and only against the attacker who thought of it.

Their own suggested fix is the one `budget_notes.md` already reaches independently:

> One possible countermeasure could be assigning budgets to users rather than beacon participants.
> This requires having users sign up for the beacon with institutional accounts and agree to the
> terms. This would let the data owner monitor and restrict user activity without removing people
> from beacons' answers, and hence without decreasing utility.

They also report that beacon size and population diversity are "effective in increasing privacy
while fully preserving the utility", though attacks remain applicable: mixing CEU, Mexican and
Yoruba populations raised the QI attack cost eight-fold, to 40 queries. And they dismiss noise
outright, arguing it "makes the system useless due to the significant decrease in utility and
should not be applied".

**Read this as a specification for BlindBeacon's budget.** A budget that prices individual queries
prices the wrong object. The attacker's information gain per query is not bounded by what the
query returns, because correlated facts arrive free. Any accounting that charges per query rather
than per *marginal information gained relative to what is already inferable* is defeated the same
way. This is a stronger argument for the novelty-pricing idea in `budget_notes.md` than that
document currently makes: novelty pricing is not just a utility optimisation to avoid charging
twice for the same fact, it is a necessary defence, and "novelty" has to mean novel given the
attacker's inferential closure, not novel given the literal query string.

### 3.4 Ayoz et al. 2021: reconstruction from snapshots, and the differencing attack

The first move past membership inference. The attacker knows the victim was among m individuals
added to the beacon in a recent update, holds a snapshot of the beacon's answers before the update,
knows public MAF and LD, and knows a few visible traits of the victim such as eye or hair colour.

The core mechanism is a differencing attack across time: if the answer at position i was no before
the update and yes after it, the probability that a newcomer carries the minor allele there goes
up. LD-weighted spectral clustering then partitions those newly-flipped alleles among the m
newcomers, and a phenotype classifier picks out which cluster is the victim.

| Newcomers added (m) | Reconstruction precision / recall |
|---|---|
| 3 | ~0.9 / ~0.9 |
| 10 (a 20% increase in beacon size) | 0.7 / 0.8 |

A reconstructed genome then feeds membership inference against a *different*, sensitive beacon. So
a phenotype-neutral beacon becomes an oracle for attacking a phenotype-specific one. The authors
note that "existing countermeasures proposed for membership inference are not directly applicable
to mitigate genome reconstruction attack", and their mitigations are about update policy: batch
larger updates, risk-score donors before admission, mix ancestries.

For BlindBeacon this is the load-bearing precedent for treating **cohort change as a channel**. A
clinical variant database is not static. VariantGrid gains samples continuously, and a live trial
recruitment session is precisely a period when the cohort is being updated. Two identical queries
a week apart are a differencing attack on the intervening additions, and neither query is
individually narrow.

### 3.5 Saleem et al. 2025: reconstruction from summary statistics alone

The most recent, and the one that most directly threatens a counts-based design. No snapshot, no
known join time, no victim identity. An alternating optimisation matches (a) the reconstructed
cohort's SNP correlations against public correlations and (b) its allele frequencies against those
the beacon reports.

| Condition | Reconstruction F1 |
|---|---|
| 50 individuals, 1,000 SNPs, allele frequencies reported | 70% (baseline 45%) |
| Doubling to 2,000 SNPs | 67.8% |
| Frequencies withheld, estimated instead | 53.4% |

Downstream, reconstructed genomes re-identified 41 of 50 individuals with 21 or fewer queries. The
authors' first recommendation is to stop reporting allele frequencies, because "beacons releasing
allele frequencies substantially increase the reconstruction risk", and their overall conclusion
is that "the protocol leaks much more information than previously thought".

The 70%-versus-53.4% gap is the price of the count tier. It is the closest thing in the literature
to a measurement of what BlindBeacon buys by bucketing counts rather than returning them exactly,
and it says bucketing helps materially but does not close the channel.

### 3.6 Attacks that do not need the victim's genome

Every attack above assumes the attacker holds the target's genome. Bu, Wang and Tang (2021) remove
that assumption. Haplotypes, being sequences of strongly linked variants, can be inferred from
summary data alone, and novel haplotypes not present in the database can be reconstructed from
allele frequencies. Haplotype-based membership inference then identifies targets with greater
power than SNV-based methods.

This matters for the BlindBeacon threat model because "the model would need the patient's VCF" is
not a defence. The 2025 reconstruction attack makes the same point from the other side: the
attacker manufactures the genomes it needs from the beacon's own answers.

### 3.7 Kinship, in both directions

Deznabi et al. (2018) showed kinship and phenotype correlations sharpen genomic inference.
Ayoz, Aysen, Ayday and Cicek (2020) turned it around into a defence: having a victim's parent in
the beacon garbles the attack signal, because a substantial fraction of variants are shared.

- One parent present: attack power stops reaching 1, peaking around 0.55 to 0.70.
- Both parents present: "attacker's power never reaches to 1, and it is always low."
- Utility loss, measured as the fraction of additional spurious "yes" responses caused by the added
  relatives, is 7.96% for the control group and 5.75% for the case group when both parents are
  added. Under 0.05% for SNPs with MAF below 0.01.

Two caveats from the paper. It assumes **the attacker does not hold the relatives' VCFs**, which is
the assumption Deznabi et al. attack from the other side. And the authors make the point that for
this project's setting the defence is unusually cheap: "in beacons of heritable diseases, a
relative is more likely to be related to the trait than a random control individual. Thus, the
utility loss caused by the proposed approach will be less compared with adding random controls." A
rare disease registry gets this protection at a lower utility price than a general cohort would.

The direction of the kinship effect flips depending on where the relative's data sits. Deznabi et
al. model a family as a factor graph, with Mendelian inheritance, higher-order SNP correlation via
a hidden Markov model, and phenotype-to-SNP association as factor nodes, then run loopy belief
propagation to infer a target's hidden variants. Inference error falls monotonically as more
relatives' partial genomes become public. So a relative whose genome is **public** leaks the
target, while a relative who is merely **co-enrolled in the same controlled beacon** protects them.

For a rare disease cohort this is unusually relevant, because rare disease databases are
disproportionately *full of families*. RUNX1db is a familial platelet disorder registry. Trios are
the norm in diagnostic exome cohorts. So SA Pathology's data may already carry a meaningful amount
of this protection for free, and it is worth measuring on the synthetic instance rather than
assuming. The flip side, from Deznabi and from Shringarpure and Bustamante's own relative-detection
result, is that a hit on one family member is partial information about all of them, which is a
argument for BlindBeacon's record-exclusion set being family-aware.

### 3.8 Beyond genotypes

Hagestedt et al. (NDSS 2019) built MBeacon for DNA methylation data and showed membership
inference against an unprotected methylation beacon reaching AUC above 0.9 in **100 queries**.
Methylation is more identifying per site than genotype and it changes over time. If VariantGrid
ever exposes anything besides variants through the same interface, the attack cost changes and
the budget calibration does not transfer.

---

## 4. The theory underneath: none of this is new

The Beacon attacks are rediscoveries of results from the statistical database literature. Reading
that literature is the cheapest way to avoid re-deriving its failures.

### 4.1 The tracker, 1979

Denning, Denning and Schwartz formalised the differencing attack. Given a query-set-size
restriction (answer only if the matching set has between k and n-k records), an attacker who wants
a statistic about an individual identified by formula C decomposes C into an answerable A and an
answerable tracker T = A AND NOT B, then computes the forbidden statistic by subtraction. Three
answerable queries suffice.

Their general result is the one to internalise:

> General trackers always exist if there are enough distinguishable classes of individuals in the
> database... Almost all databases have a general tracker, and general trackers are almost always
> easy to find. Security is not guaranteed by the lack of a general tracker.

BlindBeacon's record-exclusion floor is exactly a query-set-size restriction. Denning et al. proved
in 1979 that this alone does not secure a statistical database. The floor is necessary and it is
cheap, but it must not be presented to a governance committee as the primary defence, because a
47-year-old paper already says it is not one.

### 4.2 Dinur and Nissim 2003: the reconstruction bound

The hard limit under everything. For a database of n bits answering subset-sum queries, any
mechanism whose answers are within o(sqrt(n)) of the truth is non-private against a polynomial-time
adversary: an LP-based reconstruction over about n(log n)² random queries recovers almost the whole
database. The bound is tight, and there is a matching positive result at perturbation magnitude
around sqrt(n).

Two consequences for BlindBeacon.

- Noise magnitude must scale with cohort size, not be a fixed bucket width. For 67,741 samples,
  sqrt(n) is about 260. A design that returns counts accurate to a few units, over enough queries,
  is inside the reconstructible regime regardless of how clever the bucketing is.
- The reconstruction in the theorem uses **non-adaptive** queries chosen in advance. An adaptive
  attacker can only do better. So the bound holds against an agent a fortiori, and nothing about
  agentic adaptivity is needed to make it bite.

Dinur and Nissim also give the bounded-adversary version: against an adversary limited to T(n)
work, perturbation of about sqrt(T(n)) suffices. That is the formal ancestor of budgeting: the
required noise is a function of how many queries you let the adversary ask.

### 4.3 Auditing is intractable, and refusals leak

Two results that bear directly on BlindBeacon's audit deliverable and its refusal behaviour. Both
were previously summarised here from secondary sources and both changed on reading the papers.

**Kleinberg, Papadimitriou and Raghavan (2000).** Their problem: given a log of answered sum
queries over Boolean attributes, is there any individual whose value is the same in every 0-1
solution consistent with the answers? Their results, by query class:

| Query class | Deciding whether any value is determined |
|---|---|
| Arbitrary subsets | coNP-complete |
| Range queries over two or more public attributes | coNP-complete |
| Range queries over one public attribute (intervals) | polynomial |

Three corrections to what this review previously said. The complexity class is **coNP-complete**,
which is the paper's own term in both theorems, not "NP-hard", which appears only in its abstract.
The headline result is for arbitrary query sets, and the two-dimensional result is the
strengthening that says the hardness is not an artefact of pathological query families. And the
hardness is robust to approximation: their reduction produces instances that are either fully
determined or have no determined variable at all, so "even distinguishing 'all values determined'
from 'none determined' is coNP-hard".

The consequence for us is unchanged and now properly sourced. Genomic cohort queries range over
variant, gene, phenotype, age and sex, so we are in the hard case by several dimensions, and an
exact after-the-fact audit of "did this session compromise anyone" is not computable in general.
BlindBeacon's audit log should claim what it can prove mechanically, which is that every response
was fuzzed and every disclosure was metered and logged, and should not promise an exact disclosure
verdict computed after the fact.

Two findings the summary had lost. First, **bounded data force refusals on their own**, before any
budget is involved:

> In the case of Boolean-valued variables - or, indeed, variables over any bounded domain - this
> kind of generic auditing is impossible: For any set S of Boolean variables, there is a set of
> values in which all these variables are 1. ... Therefore, an auditing system for Boolean
> attributes should, with small probability, refuse to answer any query submitted.

Any count over a finite cohort is a bounded sum. A count that comes back equal to the cohort size,
or to zero, discloses every member's value at that locus, so no query class is safe for all
databases and a correct auditor must sometimes refuse queries that happen to be harmless. Second,
their own practical proposal is a conservative trace-pairing test that "may refuse to answer a query
even when the answer would not in fact compromise safety", and it works by reading the actual secret
values. That makes it exactly the kind of auditor the next paper rules out.

**Kenthapadi, Mishra and Nissim (2005), "simulatable auditing".** Their headline holds: "solutions
to the offline auditing problem cannot be directly used to solve the online auditing problem since
query denials may leak information". Their worked example is three lines long and worth carrying
verbatim, because it makes the mechanism obvious. The attacker asks `sum(x1,x2,x3)` and is told 15.
The attacker then asks `max(x1,x2,x3)` and is denied. The max cannot be below 5 or the sum would not
reach 15, and if it were above 5 no single value would be determined and the query would have been
answered. So the max is exactly 5, and all three values are 5. Nothing was answered and everything
leaked.

They quantify it too. Used online, an offline max auditor leaks one entry in eight across the
database, and against Boolean data an adjacent-pair attack recovers **the entire database** from
n-1 queries. That second attack also defeats the tractable approximate auditor from Kleinberg et
al., and it uses only one-dimensional range queries, so the polynomial case above is no protection.

**The definition, which this review previously got wrong.** We said the refusal decision must be
computable "from the predicate, the population prior, and the budget state, never from the actual
count". That is stronger than the paper and it describes a different, weaker design. Their
Definition 4.1:

> An auditor is simulatable if the decision to deny or give an answer to the query qt is made based
> exclusively on q1 , . . . , qt , and a1 , . . . , at-1 (and not at and not the dataset
> X = {x1 , . . . , xn }) and possibly also the underlying probability distribution D from which the
> data was drawn.

The decision may depend on every answer already released, which is data-dependent but already in
the attacker's hands. What it may not depend on is the answer to the query being decided, or on any
data not yet released. Auditors that use the queries alone and ignore the answers are the older
**monitoring** designs, which are trivially simulatable and which the paper's whole contribution is
to improve on, because they have to shut off access after a constant number of queries. Our
rendering had accidentally specified the weak one.

The correction matters in the practical direction. A budget accountant that reads its own release
history, which is what novelty pricing in `budget_notes.md` requires, is permitted. What is
forbidden is a refusal rule that reads the true count of the query being refused.

Two corollaries follow directly, and they pull in opposite directions.

- **If refusals are simulatable, a denied query need not be charged at all.** "The auditor's response
  to denied queries does not convey any new information to the attacker ... Hence denied queries need
  not be taken into account in the auditor's decision." The attacker could have computed the refusal
  itself.
- **If refusals are not simulatable, charging bits does not fix them.** `budget_notes.md` says "the
  query costs its bits whether it is answered or refused", which blocks the free-oracle version and
  is worth keeping, but a refusal boundary that reads the true count is an oracle for that count and
  a price does not stop it being one.

**Raisaro's silent occlusion is not the free pass this review previously called it.** We said S3
"gets this right almost by accident, by dropping individuals silently rather than refusing". It does
avoid the explicit refusal channel, which is real. But whether a subject's budget depletes depends on
whether that subject carries the queried allele, so which subjects have been dropped is a function of
the data, and every later answer is conditioned on it. Silence changes the shape of the channel and
does not close it. Simmons and Berger hit the same problem from the other side and had to correct for
it explicitly, which is in section 5.5.

**How tractable is it, at our scale?** This was the second question we wanted the paper for, and the
answer is discouraging for anyone hoping to lift a construction.

| Setting | What they can do |
|---|---|
| Sum queries, real-valued data, exact-disclosure criterion | Existing auditors are already simulatable, but on a bounded domain they refuse everything |
| Max queries, exact-disclosure criterion | Simulatable auditor, roughly quadratic in the query log |
| Fractional sum queries, probabilistic criterion, uniform prior | Polynomial via randomised volume estimation, and they say the volume algorithms "are still not practical" |
| **Sum queries over Boolean data** | **Open problem, named as future work** |

The last row is our case. Beacon presence and allele counts are Boolean sums, and simulatable
auditing for them is unsolved, by the authors who defined the notion. Two further constraints. The
exact-disclosure criterion is unusable on bounded domains, so a workable version needs a
probabilistic prior-to-posterior criterion, which needs a prior, and they can only handle the uniform
one. And their guarantee is spent as delta over T, with the horizon T fixed in advance and appearing
in the sampling cost, which is precisely the rigidity that privacy filters in section 6 exist to
remove.

So the honest position is that simulatability is a design rule we should adopt and can state
plainly, not a construction we can import. One consolation, and it is a real one for the audit
deliverable: a simulatable refusal can be recomputed by anyone holding the released transcript,
"a user need not even communicate with the database to check which queries would be allowed". That
makes it a property an auditor can verify from the log alone, without access to the data.

### 4.4 The census: reconstruction is not theoretical

This section was written from secondary sources and it attributed to Garfinkel, Abowd and
Martindale a result that is not in their paper. Since it was earmarked for governance material,
the correction matters more than most.

**What Garfinkel et al. (CACM 2019) actually contains** is a worked toy attack and an argument
from scale. The toy attack is a fictional block of seven people over two households, for which the
agency publishes 15 statistics and suppresses 8 of them under a rule-of-three plus complementary
suppression. Translated to a satisfiability problem it becomes 6,755 variables in 252,575 clauses,
and a solver returns a **unique** satisfying assignment, so all seven residents are reconstructed
exactly, in about two seconds on a laptop. Suppression barely helps: removing one more statistic
takes the solution count from one to two, and removing two others leaves eight solutions of which
four records are still common to every one.

The argument from scale is that the 2010 census published about 7.7 billion linearly independent
statistics over 308,745,538 people, which is roughly 25 statistics per person, against a
Dinur-Nissim requirement that noise scale as the square root of the database's bit length. Their
own conclusion is careful, and it is a possibility claim rather than a result: there is "a
theoretical possibility the national-level census could be reconstructed", though the solvers they
used are "probably not powerful enough to do so".

There is no national reconstruction in this paper, no match rate, no commercial-database linkage
and no re-identification count. **The figures "46% of the US population" and "138 million people"
appear in neither this paper nor the one that did run the experiment**, at least not paired that
way, and they should not be attributed to Garfinkel et al.

**What did run is Abowd et al. (2025)**, a fifteen-author Census Bureau paper that carries out the
attack on the real 2010 data and reports it in the detail a committee will ask for. Using 34
published tables it reconstructs block, sex, age, race and ethnicity. For 70% of populated blocks,
97,238,000 people, the reconstruction has zero solution variability, meaning the published tables
admit exactly one consistent microdata set and the attacker knows without any access to
confidential data that these records are exact. Agreement with the confidential file is 95.2% on
binned age and 46.6% on exact age.

Re-identification then depends on the quality of the attacker's own external data:

| Attacker's external data | Putative re-identifications | Confirmed | Precision |
|---|---|---|---|
| Commercial name-and-address database | 166,100,000 (57.9%) | 68,480,000 (23.9%) | 41.2% |
| High-quality quasi-identifiers | 267,800,000 (97.0%) | 208,500,000 (75.5%) | 77.9% |

Their sharpest single example is a Montana resident who gave a rare four-race non-Hispanic
response, is a population unique in a block of 15 people, and can be re-identified with 95.2%
precision. Had that person not answered the census, the best statistical guess would have been
wrong essentially every time. That is the cleanest available demonstration that the disclosure is
caused by the individual's own presence in the aggregate, which is exactly the property a
membership-inference defence has to deny.

Three further findings transfer directly to us.

- **Adding tables only ever helps the attacker.** Omitting two tables dropped zero-variability
  blocks from 70% to 65%, so those two tables alone moved 14.3 million people into provable
  reconstruction. Disclosure accumulates monotonically over releases, which is the composition
  problem stated in census terms.
- **Suppression was tested as the alternative and it failed on utility, not on privacy.** The 1980
  rules would have blanked up to 83.8% of cells and up to 87.7% of the block-level redistricting
  tables, which destroys the statutory use case. That is the same shape as Raisaro's S1 removing
  the rare variants rare disease research needs.
- **Refusal leaks here too.** Garfinkel et al. make the section 4.3 point in passing and in a
  governance register: restricting queries by suppressing small-cell results "is often
  insufficient to prevent access to indirectly identifying information, because the system's
  refusal to answer a 'dangerous' query itself provides the attacker with information".

For governance material, cite Garfinkel et al. for the mechanism, because the seven-person example
is the most explicable thing in this literature, and cite Abowd et al. for the numbers. The claim
that survives intact is the one we wanted: a government statistical agency reconstructed its own
published aggregates, concluded that aggregation had provided no protection, and changed method as
a result.

---

## 5. Defences that have been tried against Beacons

Ordered roughly by increasing sophistication. All of them are enforced below the querier, never by
the querier, which is the one design point everything in this literature agrees on.

**Suppression / minimum allele threshold (Raisaro S1; the iDASH track).** Effective, expensive,
and removes exactly the rare variants that make a beacon useful for rare disease.

**Response flipping.** Random flipping (Raisaro S2) and Al Aziz et al.'s biased randomised
response, the latter with an explicit epsilon-DP proof. Al Aziz et al.'s numbers show how narrow
the useful band is: at 90% answer accuracy the attacker needs over 300,000 queries where 5,000
sufficed, at 75% accuracy attack power stalls at 0.22, but at 98% accuracy the defence does
essentially nothing. They also flag the obvious break, that a real-time randomised response can be
averaged out by repeating the query, so answers must be memoised per user. Wan et al.'s strategic
flipping targets the SNVs with the highest differential discriminative power rather than flipping
at random, and won the 2016 iDASH Beacon track. Its weakness is that a fixed flip set is learnable,
and the authors say so themselves.

Their discussion contains a point aimed squarely at our setting. The iDASH task assumed a stateless
defender, and they note that if users must authenticate so the custodian can track query history,
then budget-based and "greedy accountable" methods become practical and the latter "could become
the optimal solution". BlindBeacon is by construction the stateful, authenticated case their paper
could not evaluate.

**Budgets.** Raisaro's S3, described above, and its production descendant.

The **Portuguese Beacon** (Oliveira et al. 2026) is the first deployed Beacon with a disclosure
budget, built for the European Genomic Data Infrastructure, and it is the closest thing to a
working system BlindBeacon can learn from. Its Re-Identification Prevention algorithm implements
Raisaro's budget strategy: initial budget `-log(p)` per {user, data subject} pair, each query
deducts its statistical power, and once a subject's budget is spent that subject is occluded from
that user's results. Three details worth copying.

- Repeated identical queries are recalled from a stored history and do not charge budget again.
  This is BlindBeacon's novelty pricing, already in production.
- The value of p is deliberately not published, because publishing it "would enable colluding
  users to optimize their strategy". A budget-aware attacker is an assumed part of their threat
  model, as it is in ours.
- RIP applies only where the user does *not* already have access, so it strictly extends what a
  beacon can offer rather than restricting it. That framing is worth borrowing verbatim for
  governance conversations.

Their utility measurement is the most useful number in the whole review, and it is sobering.

| Dataset size N | Mean queries before the first subject is occluded | Runs with zero useful queries |
|---|---|---|
| 2,504 | 1.7 | 30% |
| 10,000 | 5.0 | 10% |
| 25,000 | 8.3 | 2% |
| 100,000 | 32.5 | 0% |

VariantGrid's 67,741 samples sit between the last two rows, so a faithful implementation of the
standard budget gives a clinician on the order of twenty useful queries before occlusion starts.
Their own view is that "5 queries seems a reasonable number" for a researcher deciding whether to
apply for full access. That is a completely different task from the one in `doc/project.md`, where
the worked example needed repeated relaxation of phenotype and gene across a long conversation.
**A per-subject budget sized for dataset discovery is roughly an order of magnitude too small for
cohort assembly.** This is the strongest evidence yet that BlindBeacon's contribution has to come
from the view split, which spends budget only on what the model sees, rather than from a better
budget alone. The clinician's exact view is unmetered by design, and that is what makes the
interaction survivable.

Cost is modest: 553ms per query against 43ms, and about 12 bytes per subject per user of budget
state.

**Differential privacy for genomic queries.** Cho, Simmons, Kim and Berger (Cell Systems 2020) is
the most directly applicable paper in the entire review, and it deserves its own treatment below.
Earlier work by Johnson and Shmatikov (KDD 2013) and Simmons and Berger (2016) addresses the regime
where the number of variants vastly exceeds the number of patients, which is where naive DP noise
swamps the signal. Yilmaz, Ji, Ayday and Li (CODASPY 2022) address the
correlation problem head-on with dependent local differential privacy, on the argument that
independent per-record noise is defeated by an attacker exploiting SNP correlation, which is the
von Thenen result restated as a DP failure mode. Almadhoun and Ayday's dependent-tuple work makes
the same point for family structure.

### 5.1 Cho et al. 2020, and the second break of the same budget

Three separate things in this paper matter to us.

**The mechanism.** They answer exactly our three query shapes. Cohort discovery is a count over a
predicate. Variant lookup is the Beacon presence query, formally `1{count > 0}`. Association
testing is a chi-squared statistic computed as post-processing over four DP counts, which by
parallel composition costs one epsilon rather than four. The mechanism is the **truncated
alpha-geometric mechanism**: add integer noise drawn from a two-sided geometric distribution with
`P(D=d) = ((1-alpha)/(1+alpha)) * alpha^|d|`, set `alpha = exp(-epsilon)`, then clamp to the valid
count range. It is one extra random draw per query, with no dependence on cohort size.

The property most useful to BlindBeacon is structural. The noisy value is released once, and all
tailoring to a user's prior and loss function happens as **local post-processing on the released
number**, which they state can be done "an arbitrary number of times for different choices of p and
loss, without affecting privacy guarantees". So a clinician can reinterpret a given answer under a
different assumption about disease prevalence for free. Only touching the database costs budget.
That maps directly onto ontology walking: reconsidering what an answer means is free, asking again
is not.

Their optimality result extends Ghosh et al.'s theorem from symmetric loss to any quasiconvex loss
minimised at zero, and crucially allows **a different loss shape for each possible true count**.
That last generalisation is not cosmetic. It is exactly what the Beacon presence query needs, and
their Theorem 3 gives the optimal mechanism for membership queries as a threshold on the optimal
count guess.

Some of their numbers, for cohort discovery under asymmetric loss with a uniform prior:

| True count | Released, epsilon=0.2 | Released, epsilon=0.05 |
|---|---|---|
| 2,000 | 1,999 | 1,998 |
| 1,000 | 998 | 991 |
| 500 | 495 | 444 |
| 250 | 246 | 214 |
| 100 | 98 | 73 |

**The second break of Raisaro's budget.** This is the finding that changes the review. Raisaro's
risk accounting prices a query using a Hardy-Weinberg equilibrium model of genotype distribution.
Cho et al. show that an attacker who orders queries by largest deviation from Hardy-Weinberg
exceeds Raisaro's own cumulative threshold in **fewer than 40 queries, which the scheme fails to
detect**. Their verdict on the class: those techniques "are based on a simplified model of genotype
distribution, which make them vulnerable to sophisticated attacks."

So Raisaro's budget has now been broken twice, by two different mechanisms. von Thenen's attack
gets answers without paying for them. Cho's attack pays the advertised price, but the price is
wrong, because the model used to compute it does not fit the data. The second failure mode is the
more dangerous one for us, and it is not addressed anywhere in `budget_notes.md`. **A budget is a
price list, and a price list computed from a population prior is only as good as that prior.** Our
specificity measure divides database size by expected matches under a population prior. Where
SA Pathology's actual cohort departs from that prior, and a diagnostic cohort departs from it a
great deal, the meter under-charges, and an attacker who can find those departures gets a discount.
The synthetic instance should be built with known, deliberate departures from the population prior
so that Arm 2 can look for exactly this.

**Their own admission, and their proposed direction.** Cho et al. are candid that even a
provably optimal DP mechanism may not be good enough here:

> due to the stringent nature of DP, the smallest possible error probabilities for beacons achieved
> by our optimal mechanism could still be too high for real-world systems to endure without
> significantly restricting their use. This potential limitation may be attributed to the fact that
> the queries submitted to beacons are highly skewed toward rare variants, which are also the most
> sensitive data from the perspective of DP.

They say surmounting it "will require fundamental shifts in theory". Their accuracy claim holds for
counts above about 50, and rare disease cohort assembly lives below that. In their own variant
lookup case study the mechanism answered 11 of 17 correctly, with every error at a true count of 0
to 2 and every count of 5 to 7 correct.

And then they name the way out, which is BlindBeacon's architecture:

> hybrid systems that combine DP with traditional access control procedures to allow the sharing of
> highly sensitive query results based on trust, while securing and facilitating other types of
> queries that are more amenable to DP

We should cite this rather than claim the idea cold. It sharpens the novelty claim rather than
weakening it: the view split was proposed in 2020 as the way past a limit the authors had just
proved, and nobody built it. Note also that they consider each query independently and explicitly
leave structure-aware composition across a session as an open problem, which is the gap section 6
fills. Code is at `github.com/hhcho/priv-query`.

### 5.2 Venkatesaramani et al.: the online setting, and why authentication is load-bearing

Their ACM TOPS paper is the one that takes the *online* setting seriously, and it produces the
single best technical argument for BlindBeacon's decision to host the conversation server-side.

They separate three settings:

- **Batch**, where all of a party's queries arrive at once. This is what almost all prior Beacon
  defence work assumes, and they say plainly it is unrealistic for deployment.
- **Online with authenticated access**, where "the Beacon knows all prior queries" and can decide
  per query.
- **Online with unauthenticated access**, where the Beacon "does not know which queries have
  previously been made" and must assume the worst.

In the unauthenticated case they prove the defence collapses to a single decision made up front:
if every variant is eventually queried, the optimal solution flips a fixed set at time zero and
nothing thereafter. A stateless Beacon cannot adapt, by construction.

And then the result that matters most. Against an adaptive attacker in the unauthenticated online
setting, "all methods including OMIG now compromise privacy". Their own defence fails. Privacy in
the online setting is only achievable when the server knows who is asking and what they have
already asked.

`architecture.md` justifies hosting the conversation server-side on governance grounds, that the
audit claim is unprovable at an egress point we do not own. This paper supplies an independent and
purely technical reason for the same decision: **an anonymous-access Beacon cannot be defended
against an adaptive attacker at all.** That is a stronger argument than the one currently in the
design doc, and it should be added to it.

Their other results: the marginal-impact greedy algorithm flips orders of magnitude fewer responses
than strategic flipping for equal or better privacy, and against adaptive attackers on unmodified
responses clustering achieves a 100% true positive rate, which their defence pushes to over 50%
false positives. Their stated limitation is the familiar one: "Our privacy model is specific to the
Beacon service, and the LRT-based attack; it is possible that other attacks can be devised that can
defeat our approach." Zhang et al. then devised one.

### 5.3 Correlation-aware differential privacy

Yilmaz, Ji, Ayday and Li's dependent local differential privacy is the most direct published answer
to the von Thenen problem, and it is worth evaluating as a component rather than reading as
background. Their argument is that standard local differential privacy assumes independence, and
that correlated genomic data breaks the assumption: an attacker who eliminates statistically
implausible values using linkage disequilibrium cut estimation error roughly in half against plain
randomised response, from 0.8 to 0.4 at epsilon=1.

Their definition calls an element **T-dependent** when its released value depends on at most T other
elements, and requires indistinguishability only among the statistically plausible values rather
than all possible states. The mechanism eliminates states that correlate poorly with already-shared
SNPs, then redistributes randomised-response probability among the survivors while preserving the
`e^epsilon` ratio, and biases the remainder toward values useful to the querier.

Their reported numbers are relevant to us because they measured beacon accuracy specifically: about
95% accuracy even at small epsilon, against under 70% for plain randomised response, with roughly
25% better privacy after their correlation attack. If those hold up, this is a better starting
point for our count and presence tiers than an independence-assuming mechanism, precisely because
the failure mode we most need to avoid is the correlated one.

**Optimisation and game theory.** Wan et al. (AJHG 2017) model release as a Stackelberg game where
the recipient decides whether attacking is cost-rational, and show suppression-only policies
increase re-identification risk by around 300% relative to a joint policy combining suppression
with data use agreement penalties. Wan et al. (Science Advances 2021) extend to multistage attacks.
Venkatesaramani et al. (Genome Research 2023) unify suppression and flipping in one optimisation,
handle the **adaptive-threshold attacker** who recalibrates after seeing the defence, and scale to
1.3 million variants. Their own stated limitation is the important one: the method is "specific to
MI attacks that leverage an LRT score", and against LD-exploiting attacks it reaches about 88%
protection and "fails to achieve privacy for all individuals". Zhang et al. (arXiv 2024) then show
a general Bayesian attacker is strictly stronger than any LRT-based threat model, "even with
non-informative priors", which retroactively weakens every defence calibrated against LRT.

**Kinship as noise.** Ayoz et al. (2020), section 3.7 above.

**Cryptography.** MedCo, FAMHE (multiparty homomorphic encryption for federated analytics), and
MPC-based variant queries such as Akgün et al. (2022), whose two-non-colluding-proxy design hides
which node contributed a yes, something plain federated Beacon does not. These change who can see
the data, not what the answers reveal. They are orthogonal to BlindBeacon and do not substitute for
it: an untrusted agent receiving a correctly computed encrypted-then-decrypted count learns exactly
what it would have learned from a plaintext one.

### 5.4 Bounded priors: the lever we are not taking, and what it would have been worth

This was the review's one genuinely open design question. `CLAUDE.md` assumes an unbounded
adversary, Mittos et al. credit Tramèr et al.'s bounded-prior relaxation as the field's most
effective move on the privacy-utility tradeoff, and the two cannot both be right about what we
should do. Having read the paper, the question is closed: **the lever is real, it is bounded above
by roughly a factor of two, and in our regime it is worth close to nothing.** Assuming an unbounded
adversary is the right call and it is nearly free.

First, a correction of vocabulary that this review had wrong. What Tramèr et al. bound is not the
adversary's knowledge of the target's genome, which their threat model happily leaves complete.
They bound only the adversary's **prior probability that a given individual is in the dataset**.
They work inside Li et al.'s positive membership privacy, in which epsilon-bounded differential
privacy is exactly gamma-PMP against adversaries whose per-individual priors may be anything at
all, and they then restrict that family so each prior lies in `[a, b]`, with certainty at 0 or 1
still allowed for individuals the adversary already knows about.

Their own headline example gives the size of the prize. To guarantee that no adversary's posterior
belief about any individual's membership more than doubles:

| Adversary's prior on each individual | Differential privacy needed |
|---|---|
| Anything in [0, 1] | epsilon = ln 2 |
| Pinned at one half | epsilon = ln 3 |

In the GWAS evaluation that buys a 25% smaller study: a mechanism releasing the causative SNPs
needs 10,000 patients under the unbounded model and 7,500 under the bounded one, at the same
privacy level. Against genotyping costs that is a real saving and it is why the field rates the
idea.

Three reasons it does not transfer to BlindBeacon.

**The gain is largest exactly where our priors are not.** Their Lemma 3 makes the PMP bounds tight
only "for priors approaching 0 or 1", which is to say the relaxation buys the most when the
adversary is maximally uncertain and nothing when it is confident either way. Working their
Theorem 2 through for a doubling guarantee, our own arithmetic puts the best achievable saving at
just under a factor of two in epsilon, reached for priors near one half, and falling to zero as the
prior approaches either end. Their 0.5 prior is not a modelling convenience, it is derived from a
balanced case-control study in which the adversary already knows the participant list and is only
guessing case versus control. A Beacon has no such structure. An outsider's prior that a particular
person appears in a rare disease registry is close to zero, and a clinician's prior about a patient
they referred is close to one. Both are the regimes where the relaxation gives back plain
differential privacy.

**There is no composition theory.** The word does not appear in the paper. The guarantee is
single-shot, and their own section on adversaries outside the assumed bounds says such an adversary
simply falls back to the underlying epsilon-DP guarantee. An adaptive attacker manufactures that
condition on purpose: the first answer moves the posterior, the moved posterior is the prior for the
second query, and the assumption has to be re-made at every step. A relaxation with no composition
theorem is not a budget.

**Failure is graceful, which is the one genuinely reassuring finding.** Because the relaxation is
attained *through* differential privacy rather than instead of it, an adversary whose prior falls
outside the assumed band still gets the plain guarantee, just at the looser factor. So the choice
is not between safety and utility. It is between claiming a tighter posterior bound that rests on
an assumption about the attacker, and claiming a looser one that does not.

**What we should actually do with the paper.** Use it as a calibration instrument rather than a
budget mechanism. Their Theorem 2 and Lemma 3 convert an epsilon into a statement of the form "for
an adversary with prior p, this answer moves the posterior to at most q". That is a far more
legible thing to put in front of a governance committee than an epsilon, and it costs nothing to
report, because the budget is still accounted at the unbounded rate. Record the choice in
`budget_notes.md` as a decision with a measured price rather than an unexamined default: we are
declining a lever worth at most a factor of two in epsilon in the best case, and close to nothing
in ours.

### 5.5 Per-individual privacy measurement, and why the exclusion floor needs it

Simmons and Berger's PrivMAF is a measure rather than a defence, and it answers a question
`budget_notes.md` currently begs. Our specificity measure prices a predicate against a population
prior, which is a statement about the average participant. PrivMAF computes, for each named
participant, an upper bound on the posterior probability that this individual is in the study given
the released frequencies. Mechanically it is the likelihood ratio between "these counts arose from
n people including d" and "these counts arose from n people not including d", times the prior odds.
It measures how much of the deviation of the released numbers from population expectation is
explained by this particular person's genome.

Their argument for doing it per person rather than on average is the one that should govern our
floor:

> One limitation of these approaches, however, is that they ignore the fact that a given piece of
> aggregate data might reveal different amounts of information about different individual study
> participants, and instead look at an average measure of privacy over all participants. For the
> unlucky few who lose a lot of privacy in a given study, a privacy guarantee for the average
> participant is not very comforting.

On real data the gap is not subtle. Against the WTCCC 1958 British Birth Cohort, 1,000 participants
drawn from an assumed background population of 100,000, releasing 1,000 SNP frequencies:

| Measure at 1,000 SNPs released | Posterior membership probability |
|---|---|
| Worst-affected participant | ~0.9 |
| Average participant | roughly a fifth of that |

The worst participant is "almost five times as likely to be in the study as the average
participant". A population-averaged meter reports the second number and the floor is supposed to
catch the first.

Two further results are directly usable. Perturbation is cheap in the right region and useless in
the wrong one: rounding frequencies to two decimal places roughly halves the worst-case posterior
in a 10,000-person study, while three decimal places "seems to add very little", which is the same
bucketing-helps-materially-but-does-not-close-it shape as section 3.5. And their noise mechanism at
the setting they recommend is formally worthless as differential privacy, 40-differentially private
across 200 SNPs, while measurably reducing the actual posterior, which is a concrete instance of
the argument in section 2.3 that clinicians will not accept DP-grade noise and may not need it if
the measure is honest about the assumptions it makes.

**The finding that changes our design is their footnote about the release decision.** They observe
that choosing to release only when the worst-case score is acceptable is itself a data-dependent
decision that leaks, and they correct for it by releasing at a strictly stricter internal threshold
so that the posterior conditioned on both the numbers and the fact of release still meets the
stated bound. That is section 4.3's simulatability problem, discovered independently by a different
group in a different formalism, and it lands squarely on the record-exclusion floor. **A floor that
drops conspicuous records is deciding what to release based on what the records contain.** Simmons
and Berger supply the template for accounting for that, which is to tighten the internal threshold
until the conditioned posterior meets the advertised one, but not the accounting itself for our
setting.

The limits are the usual ones and they are severe: SNPs are assumed independent, individuals are
assumed unrelated, and the model is a single one-shot release with no composition and no
adaptivity. So this is not a drop-in replacement for epsilon accounting. The right reading is that
the two measures price different halves of the same event. Specificity prices the predicate before
the answer arrives, from the prior. PrivMAF prices the record's conspicuousness after it arrives,
from the likelihood. A per-subject ledger accumulating the second is the natural implementation of
the exclusion floor, and it is what `budget_notes.md` gestures at when it says the floor is
per-individual but then prices it with a population-level quantity.

---

## 6. Composition against an adaptive adversary

`budget_notes.md` gets the central point right: use DP-style composition because the bound holds
regardless of how the attacker chose the next query. The literature refines that in a way that
matters for implementation.

Standard advanced composition and the tight bound of Kairouz, Oh and Viswanath (2015) allow the
*sequence of mechanisms* to be adaptive, but require the privacy parameters of each step to be
**fixed in advance**. A BlindBeacon session does not work that way. The number of queries is
unknown, and the epsilon a given query should cost depends on how narrow the clinician decided to
go after seeing the last answer.

Two results close that gap.

- **Rogers, Roth, Ullman and Vadhan (2016)** introduced privacy **filters** and **odometers** for
  the case where both the algorithms and their privacy parameters are chosen adaptively. A filter
  outputs HALT or CONT at each round and guarantees that "with probability 1 - delta, it will output
  HALT before the privacy loss exceeds epsilon". An odometer reports realised privacy loss with no
  preset budget at all. They prove a formal separation: any valid odometer loses a factor of
  sqrt(log log n) against the filter bound, so tracking without committing to a ceiling is
  strictly weaker than enforcing one.
- **Whitehouse, Ramdas, Rogers and Wu (ICML 2023)** show fully adaptive composition costs almost
  nothing: filters matching the rates of advanced composition "including constants, despite
  allowing for adaptively chosen privacy parameters".

The mapping onto BlindBeacon is direct. The **filter is the budget** and the **odometer is the
audit log**. The filter enforces the session and per-clinician ceilings; the odometer produces the
per-session statement of how much was actually disclosed, which is what F2's governance stats pack
needs. And because the theory permits adaptively chosen per-query epsilons at almost no cost, the
budget does not need a rigid pre-allocated schedule, which is what makes ontology-driven
relaxation affordable.

One caveat carries over from section 4.3, and reading Kenthapadi et al. made it sharper rather
than smaller. A filter's stopping decision must be simulatable, or the stopping time itself leaks.
The odometer and filter papers do not address whether the HALT decision reveals anything about the
data; they bound the privacy loss of the answers, not of the halting.

The two literatures are not merely unjoined, they pull against each other on the exact point that
makes filters attractive. Kenthapadi et al.'s own simulatable construction for sum queries spends
its failure probability as delta over T, with the horizon T fixed in advance and appearing in the
cost of every decision. Privacy filters exist precisely to remove that pre-commitment and let the
number of queries and their per-query epsilons be chosen adaptively. So a simulatable filter is not
a matter of bolting one result onto the other. It needs a stopping rule whose decision is
computable from the released transcript while its parameters are chosen after seeing that
transcript. Add to that Kenthapadi et al.'s open problem, simulatable auditing over Boolean sums,
which is the Beacon case, and this is a real piece of work rather than a formality. It is still
small, and BlindBeacon needs it the moment the budget refuses anything.

---

## 7. The agent layer

### 7.1 AskBeacon: the closest prior art, and what it does not do

Wickramarachchi et al. (Bioinformatics 2025) put a natural language interface over sBeacon: the
model extracts scope, granularity, variants and filters, confirms with the user, queries the
beacon, and generates analysis code that is statically analysed and sandboxed.

Its privacy model is that "genomic data is not exposed to the LLM directly", the beacon is the sole
conduit, and extraction runs at the user's own access level. It does not state which response
granularity the model actually receives, and there is no treatment of re-identification risk from
model inference over aggregate results, no adaptive-querying threat model, no budget, and no
fuzzing of what the model sees. **AskBeacon's privacy story stops precisely where the entire
literature in sections 2 to 5 begins**, which is that aggregate answers, accumulated, re-identify
people. That is not a criticism of the paper, which is about usability, but it is the delta
BlindBeacon exists to close, and it makes AskBeacon the natural no-budget comparator that
`overall_plan.md` E3 already identifies.

### 7.2 Disclosure budgets for agents, outside genomics

**Noisegate** (grey literature, no paper found) is the closest architectural match: an untrusted
model compiles a question into a constrained query, a DP engine executes it under a tracked budget,
and enforcement lives in trusted code the model cannot reach. Its attack gallery pins differencing,
membership inference and singling out in CI.

**OCELOT** (Xie and Li, arXiv 2026) is the closest conceptual match to the framing in
`architecture.md` that the budget should be denominated in bits. It defines an inference-leakage
budget as the maximum allowed improvement in an adversary's posterior about a secret across an
agent's trajectory, measured as Bayes advantage in **min-entropy, in bits**, accumulated additively
in a Merkle-chained ledger with a hard constraint at commit time. It explicitly positions this as
paralleling a DP privacy odometer while using min-entropy bounds rather than epsilon-delta. Reported
results are 0.31 bits cumulative leakage at 91.3% task success, against a best baseline at 0.52
bits and 62.3% utility.

This is useful to BlindBeacon in two ways. It is independent confirmation that bits are a
defensible currency for an agent disclosure budget, which makes our framing legible rather than
novel. And it shows the shape of the honest caveat: OCELOT separates a deductive layer, where the
ledger arithmetic cannot be exceeded, from an empirical layer, where semantic privacy depends on
the per-operation costs actually upper-bounding true adversarial inference. That is the same
two-layer honesty `architecture.md` needs when it says predicate specificity and epsilon answer
different questions and adding them is a defensible worst case rather than a rigorous quantity.

There is also a body of work on keeping data away from the model rather than metering it, such as
MaskSQL's schema and value abstraction, and a contrasting line, such as SafeNLIDB, that trains the
model itself to reason about what is safe to disclose. The second approach is incompatible with the
governance constraint in `CLAUDE.md` that the guarantee must not depend on the model's behaviour.

### 7.3 Language models as adversaries

Nothing in the literature runs a language model against a Beacon. The adjacent evidence is:

- **Staab et al. (ICLR 2024)** show models inferring personal attributes from free text at up to
  85% top-1 accuracy, at roughly 1/100 the cost and 1/240 the time of human labellers. This is the
  evidence for the `CLAUDE.md` constraint that what cannot be mined today may be minable later:
  the leak is inferential, not memorised.
- **Tran, Kotevska and Xiong (arXiv 2026)** use LLM agents to *design* new membership inference
  attacks, improving AUC by up to 0.18 over hand-designed ones. This is the literal statement of
  the project's thesis that agentic discovery and agentic re-identification are the same
  capability, demonstrated in a different domain.
- **Zhan et al. (USENIX Security 2025)** ran a 502-participant randomised trial showing
  conversational models engineered to extract personal information succeed significantly more than
  benign ones, with rapport and reciprocity strategies working best *while lowering* participants'
  perceived risk. This is the best available evidence for the clinician-as-leak-channel problem in
  `architecture.md`, and it says the social channel is real and measurable, not hypothetical.
- **Greenblatt et al. (ICML 2024), AI Control** is the closest formal framework to BlindBeacon's
  structure: an untrusted model, a trusted overseer, a small human labour budget, and protocols
  evaluated against a model actively trying to subvert them. Notably, it *assumes* human audits are
  perfectly effective and names evaluating auditing failures as future work. The gap it declines to
  study is the one BlindBeacon's "tell Claude" declassification control has to answer.

---

## 8. What this changes for BlindBeacon

Concrete, in priority order.

1. **Price marginal information, not queries.** von Thenen's QI attack defeats a per-query budget
   by inferring answers from correlated queries it never asks: 5 queries against a budget that
   expires at 6, where the attack it was designed to stop needed 7. Novelty pricing must be
   computed against the attacker's inferential closure, which for genomics means an LD model and an
   ontology model, not against the literal query string. This is the largest single design
   implication and it is currently only half-present in `budget_notes.md`. Note also how the
   failure presented itself: not as a collapse, but as a one-query margin. Our own attack arm
   should be looking for margins that size, not for dramatic breaks.
2. **Make refusals simulatable, and understand what that permits.** Kenthapadi et al.'s definition
   allows the refusal decision to depend on the query, the whole query history and every answer
   already released. It forbids dependence on the answer being decided and on data not yet
   released. That is more permissive than this review previously stated, and it means a budget
   accountant that reads its own release cache is fine. Two consequences. A refusal that is
   genuinely simulatable need not be charged at all, because the attacker could have predicted it.
   A refusal that is not simulatable is an oracle for the count that triggered it, and charging
   bits does not make it safe. Raisaro's silent per-individual occlusion is better than an explicit
   refusal but it is not exempt: which subjects have been dropped is a function of the data.
3. **Meter per record, not per population average.** Simmons and Berger measured both on real data
   and the worst-affected participant was about five times as exposed as the average one. Our
   specificity measure prices a predicate against a population prior, which is an average, while
   the exclusion floor it feeds is supposed to protect the tail. The two do not meet. The floor
   needs a per-subject ledger accumulating that individual's own contribution to what was released,
   and their likelihood-ratio construction is the right shape for it.
4. **Budget the cohort's change over time, not just the session.** Ayoz et al. reconstruct genomes
   from before-and-after snapshots. Repeating a query after a data load is a differencing attack
   that no per-query specificity measure will flag. The record-exclusion set and the release cache
   need to be keyed to a dataset version. Abowd et al. make the same point at census scale: adding
   two tables moved 14.3 million people into provably exact reconstruction.
5. **Size noise against cohort size.** Dinur-Nissim puts the reconstructible regime at o(sqrt(n))
   perturbation. For 67,741 samples that is a scale of about 260. Fixed bucket widths chosen for
   readability will not survive enough queries.
6. **Expect the budget to be too small for the real task, and let the view split carry the load.**
   The Portuguese Beacon gets a researcher about 32 useful queries at N=100,000. `doc/project.md`
   describes a task that took months of iterative relaxation. These do not meet. What makes
   BlindBeacon different is that the clinician's view is unmetered, so the budget only has to cover
   what the model sees. That should be stated as the central claim, and the evaluation should
   measure it: honest tasks solved per bit spent, not just bits spent.
7. **Assume the attacker does not need a patient's genome.** Both the haplotype inference work and
   the 2025 reconstruction attack manufacture the genomes they need. "The model would have to
   already have the VCF" is not a defence and should not appear in governance material as one.
8. **Beacon v2's metadata and filtering-term queries are unstudied.** Every published attack
   targets v1-style allele presence. BlindBeacon's phenotype-centric queries over HPO and MONDO are
   a larger and less-explored surface, and its own attack arm may be the first evidence about it.
9. **Measure the kinship effect on the synthetic instance.** Rare disease cohorts are full of
   families. That may already provide meaningful protection, and it may also mean exclusion should
   propagate across a pedigree.
10. **Calibrate against more than an LRT.** Zhang et al. show a Bayesian attacker strictly dominates
    LRT-based threat models, and the RL work shows an attacker that adapts beats static defences.
    A restriction level tuned against a fixed attacker is tuned against the wrong thing.
11. **Attack the price model, not just the budget.** Cho et al. broke Raisaro's budget by querying
    variants that deviate from Hardy-Weinberg equilibrium, exceeding the threshold in under 40
    queries without the scheme noticing, because the model used to price queries did not fit the
    data. Our specificity measure has the same shape and the same exposure: it prices against a
    population prior, and a diagnostic cohort is not a population sample. The synthetic instance
    should contain deliberate, known departures from the prior so Arm 2 can hunt for the discount.
12. **The unbounded adversary is now a priced decision, not an open question.** Bounding the
    adversary's prior is the field's favourite utility lever and it is worth at most about a factor
    of two in epsilon, obtained only when the adversary is maximally uncertain about membership.
    Our priors sit at the ends of that range, where the relaxation returns plain differential
    privacy, and it has no composition theory at all. Record the choice in `budget_notes.md` with
    that price attached, and reuse Tramèr et al.'s conversion the other way, to report what a given
    epsilon does to a stated prior. That is a better governance artefact than an epsilon on its own.
13. **Anonymous access is indefensible, and that is a technical claim, not just a governance one.**
    Venkatesaramani et al. prove that without knowing who is asking and what they already asked, an
    online defence collapses to one fixed decision made up front, and that against an adaptive
    attacker in that setting every method they tested, including their own, fails. Add this to
    `architecture.md` alongside the audit-provability argument for hosting the conversation
    server-side. It is the stronger of the two.
14. **Expect resistance to differential privacy from clinicians, and prepare for it.** Four
    knowledgeable respondents in the Mittos survey rejected DP in clinical settings outright, and
    the warfarin dosing case is the reason. Naveed et al.'s poll of 61 biomedical scientists found
    they will trade money and time but not accuracy. The counter is not an argument, it is the
    experiment-driven epsilon feasibility analysis that one of the Mittos respondents asked for and
    that Arm 2 can produce, plus the fact that the clinician's own view is never fuzzed.
15. **Report false positive rates, not hit counts, and look for a bound.** Braun et al. showed the
    founding attack statistic is badly miscalibrated on real data, so Arm 2's synthetic instance
    needs individuals the attacker does not hold and the headline number must be re-identification
    rate at a fixed false positive rate. Beyond that, Sankararaman et al. is the model to imitate
    where we can: a bound on the power of any attacker beats a score against the attackers we
    happened to field, and it is the only form of answer that survives the design constraint that
    models keep getting better.
16. **Cite the census for governance, and cite it correctly.** The reconstruction of the 2010 US
    Census is the strongest existence proof available that publishing only aggregates is not a
    privacy property, and it lands with a committee in a way a bioinformatics paper does not. Use
    Garfinkel et al. for the seven-person worked example, which anyone can follow, and Abowd et al.
    for the outcome: 97,238,000 people whose records the published tables determine uniquely, and
    re-identification precision that rises with the quality of the attacker's own data. Do not
    repeat the "46% of the population" figure that circulated in secondary sources, which this
    review carried and which neither paper supports in that form.

---

## 9. Gaps in the literature

Where BlindBeacon's contribution actually sits, stated as things nobody has published.

- **No language model has been fielded as the attacker against a Beacon.** The nearest neighbours
  are the RL attacker in the 2025 Genome Biology paper, which is a learned policy over MAF-banded
  query groups rather than a general reasoner, and AutoMIA, which uses LLM agents to design
  membership inference attacks in a different domain. Arm 2 of this project appears to be genuinely
  first.
- **No view-splitting has been built.** Every Beacon defence in section 5 is single-view: the
  protection applies to everyone, so nobody sees the exact numbers. One qualification, and we
  should make it ourselves rather than have a reviewer make it. Cho et al. proposed the idea in
  2020, as "hybrid systems that combine DP with traditional access control procedures to allow the
  sharing of highly sensitive query results based on trust, while securing and facilitating other
  types of queries that are more amenable to DP". They proposed it in one sentence, as the way past
  a limit they had just proved, and did not build it. Nobody else has either, and nobody has posed
  it as a trusted human and an untrusted model collaborating on one problem. The honest claim is
  that we are building and evaluating an architecture the field named and left alone, which is a
  better claim than inventing it.
- **No study of a model socially engineering a trusted overseer out of privileged data.** AI
  Control explicitly assumes human audits are perfect and names this as future work. Zhan et al.
  show models extracting information from humans in a consumer setting. Nobody has connected the
  two in a data-disclosure setting. The "tell Claude" declassification control is a proposed answer
  to an open problem, not to a solved one.
- **No cross-node composition analysis for Beacon networks.** Federated aggregation is discussed as
  a mitigation, never as an attack surface, despite the Beacon Network aggregating dozens of
  endpoints. Directly relevant to the RUNX1db international consortium scenario in `doc/project.md`.
- **Beacon v2's richer query surface is untested.** See point 8 above.
- **The adaptive data analysis literature and the LLM agent privacy literature do not cite each
  other.** Fully adaptive composition gives exactly the tool an agent session needs, and nobody has
  applied it there. Doing so is a cheap, real contribution.
- **Nobody has joined simulatable auditing to privacy filters.** The filter literature bounds the
  privacy loss of the answers and is silent on what the HALT decision reveals. The auditing
  literature treats the refusal as part of the output but predates modern composition, and its own
  constructions fix the query horizon in advance, which is the thing filters were invented to
  avoid. Underneath both sits Kenthapadi et al.'s own open problem, simulatable auditing over
  Boolean sums, which is the Beacon case exactly. The join is small, unpublished, and directly
  needed by anything that refuses queries when a budget runs out.
- **Nobody has accounted for the leak in a per-subject exclusion decision.** Simmons and Berger
  show that deciding whether to release based on a record's own contents leaks, and correct for it
  by tightening the internal threshold until the conditioned posterior meets the advertised one.
  Raisaro-style silent occlusion has exactly that structure and no such correction exists for it.
  Sections 4.3 and 5.5 state the problem. It is the most concrete unclaimed piece of work this
  review has turned up, and BlindBeacon needs it because the exclusion floor is in the design.
- **No cost curves.** Nothing in this literature reports attacker cost against restriction level in
  a form a data custodian could use to choose a setting. Deliverable E2 has no competition.

## 10. Reading order

If you read five things, read these.

1. Raisaro et al. 2017 (JAMIA), for the budget mechanism we are extending and the three-query
   attack that motivates it.
2. von Thenen et al. 2019 (Bioinformatics), for why that budget does not hold.
3. Cho et al. 2020 (Cell Systems), for the optimal mechanism, the second and different break of
   the same budget, and the authors' own account of where differential privacy runs out.
4. Oliveira et al. 2026 (Database), the Portuguese Beacon, for what a deployed budget actually
   costs a user.
5. Poorghaffar Aghdam et al. 2025 (Genome Biology), for the only defence that survives an adaptive
   attacker, and for its own admission that it does not cover reconstruction or collusion.

Then Dinur and Nissim 2003 and Kenthapadi et al. 2005, which between them constrain how good any
of this can get. For governance material rather than design, Garfinkel et al. 2019 and Abowd et al.
2025, in that order: the first explains the attack in seven people, the second runs it on 308
million.
