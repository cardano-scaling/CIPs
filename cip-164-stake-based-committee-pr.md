# CIP-164: Replace wFA^LS committee with stake-based committee selection

> **Target:** `cardano-scaling/CIPs`, branch `leios`, file `CIP-0164/README.md`
> **Status:** Draft for community discourse
> **Authors:** _to be filled in by submitter — likely Sebastian Nagel et al._

---

## PR title

```
CIP-164: Replace wFA^LS committee with stake-based committee selection
```

## PR body

### Summary

This PR proposes replacing the **wFA^LS (weighted Fait Accompli with Local Sortition)** voting-committee scheme currently specified in CIP-164 with a simpler **stake-based committee** scheme. The committee for an epoch is fixed at the epoch boundary by ordering pools by stake (descending) and selecting them, in order, until a configured cumulative-stake target is met. There are no per-EB elections, no non-persistent voters, and no per-vote sortition proofs. Every committee member is eligible to vote on every EB within the epoch.

The motivation is **certificate size and verification cost on the critical path**. Under wFA^LS, the bulk of certificate size and verification cost comes from the non-persistent voters' eligibility proofs. Removing local sortition eliminates that overhead. The certificate collapses to a bitfield over a known committee plus a single aggregated BLS signature. The following comparison uses the [`leios-wfa-ls-demo`](https://github.com/cardano-scaling/leios-wfa-ls-demo) benchmarks against mainnet stake data, with the wFA^LS configuration of 481 PVs + 94 NPVs as the reference point:

|                                       | wFA^LS (current spec)   | Stake-based, equivalent coverage (this PR) |
| ------------------------------------- | ----------------------- | ------------------------------------------ |
| Committee size                        | 481 PV + 94 NPV         | 916 voters (top-stake at 99% cumulative, mainnet epoch 612) |
| Certificate size                      | ~6.8 kB                 | ~203 bytes (bitfield ⌈916/8⌉ + agg. sig. + overhead) |
| Certificate verification (worst case) | ~10 ms                  | ~2 ms                                      |
| Vote size                             | 94 B (PV) / 171 B (NPV) | 94 B (uniform)                             |
| Vote verification                     | 427 μs / 1750 μs        | 427 μs (uniform)                           |

**On equivalent coverage.** Mainnet has roughly 3,000 SPOs with a Gini coefficient of ~0.84. Stake distribution data across 51 recent epochs (562–612, summarized in [Paul Clark's stake-distribution analysis](https://docs.google.com/document/d/1CmSC8BRDZTH32fk_bZ4db6baYGodZMB-71b-S18B2PQ)) shows highly stable coverage thresholds:

| Cumulative stake | Pools required (epoch 612) | Range across epochs 562–612 |
| ---------------- | -------------------------- | --------------------------- |
| 90%              | 471                        | 471–503                     |
| 95%              | 601                        | 601–636                     |
| 99%              | 916                        | 911–956                     |

The trend is gently *decreasing* (consolidation toward larger pools), which tightens the argument over time rather than weakening it. wFA^LS's configuration of 481 PVs + 94 NPVs gives expected coverage near 100% per round, but only as a *sample* of the residual — NPVs are stake-weighted Poisson draws, not a census. **Stake-based truncation at 99% cumulative coverage (916 voters at epoch 612) therefore matches wFA^LS's effective security regime closely**: both schemes capture the dominant top-471 (the 90% cut, essentially the same set as the 481 PVs), and the 99% cut additionally captures most of the same tail that wFA^LS's NPV draws sample from. The remaining ~1% of stake — fragmented across ~2,000 very small pools with a long Pareto tail — is too dispersed to threaten the 75% quorum under any of the schemes. Doubling the committee from ~480 (90% cut) to ~916 (99% cut) increases certificate size only by the bitfield delta of ~55 bytes; the cert-size argument is essentially invariant across the 90%–99% coverage range. The size estimate above (~203 B) is back-of-the-envelope and will be replaced with a measured value from the wfa-ls-demo before merge.

Certificates are perpetual on-chain overhead; certificate verification is on the critical path between receiving a certifying RB and being ready to extend the chain; votes diffuse under a tight $L_{\text{vote}}$ budget. The reductions compound: at coverage parity with wFA^LS, the cert-size reduction is over an order of magnitude (~6.8 kB → ~203 B, ~33×) and certificate verification time drops ~5×.

### Scope

The change is a refinement of the certificate format and committee-selection rules, not a redesign. **Unchanged:** BLS over BLS12-381, the 75% stake quorum, certificate inclusion timing constraints, the LeiosNotify / LeiosFetch mini-protocols, the threat model. **Changed:** committee structure (dual PV/NPV → single stake-truncated set), vote structure (two shapes → one), certificate format (per-NPV eligibility proofs → bitfield + single aggregated signature), and epoch-boundary computation (Fait Accompli precomputation → stake-ordered truncation). CIP-164 Appendix A already states that *"alternative schemes satisfying these requirements could be substituted"*; this PR exercises that explicit extension point.

**Explicitly out of scope:**

- A BLS key rotation mechanism. The adaptive-security claim applies equally to *all three* schemes considered (All-vote, stake-based truncation, wFA^LS) only under the assumption of regular rotation; without rotation, the static-vs-adaptive distinction collapses regardless of committee scheme. The mechanism for registering rotation on chain (pool registration certificate path, epoch-boundary activation, prevention of within-epoch grinding) is a meaningful spec change in its own right and is left to a follow-up PR.
- VRF key rotation. Related, but Praos-layer and outside CIP-164.
- A fixed committee size. The parameter is a cumulative-stake target (or maximum-error equivalent); realized committee size tracks the stake distribution.

### Evidence

1. **[`cardano-scaling/leios-wfa-ls-demo`](https://github.com/cardano-scaling/leios-wfa-ls-demo)** — Haskell implementation of wFA^LS on mainnet stake data, with the vote/cert size and verification benchmarks cited above.
2. **Cryptographer review of voting alternatives** — internal review comparing All-vote, stake-based truncation, and wFA^LS on adaptive-security and overhead grounds. With BLS key rotation, all three schemes provide comparable adaptive security in the practically-relevant regime; the one feature wFA^LS uniquely retains is non-targetability of non-persistent voters, but the persistent voters under wFA^LS are precisely the largest pools an adaptive attacker would already target. The report is being prepared for public release; this PR will be amended to link it once published.
3. **Network simulations** — early results on a 750-node mainnet-like topology show throughput, EB certification rate, and below-threshold rates essentially identical across the three schemes at all tested loads (0.150–0.350 TxMB/s). The simulation's stake-based configuration used a different threshold setting (~213-node committee at "threshold 160") than what we propose here (916 voters at 99% cumulative coverage on epoch 612 data), so the absolute vote-count numbers from the simulation should not be read as bounds on the configuration proposed in this PR. The qualitative finding — that network capacity is not the bottleneck under any of the three schemes — does carry over. Simulations at 1,500–2,000 nodes with the proposed configuration are in progress; this PR will be updated before merge.

### To be discussed

1. **Naming.** Is "stake-based committee" the right name? Internal discussion has used "Truncate", "top-stake-fraction", and "stake-based cutoff" interchangeably. Proposal: **stake-based committee** as the formal spec name, with **stake truncation** as an acceptable synonym informally.
2. **Parameter shape.** *Cumulative stake covered* (e.g. 99%) vs *maximum truncation error* (e.g. ≤1%). The two are equivalent. Slight preference for the error formulation because the same parameter shape generalizes across all three schemes — useful for the staged-fallback story in *Alternatives & Extensions* — but cumulative-stake is more intuitive.
3. **Quorum as a stake threshold.** The current spec defines $\tau$ as a "minimum fraction of committee votes required for certification" — a head-count fraction over committee members. This PR redefines $\tau$ as a **minimum fraction of total active stake** whose votes must be present in the certificate. $\tau = 0.75$ means exactly "votes from pools holding ≥75% of total active stake have signed." The security argument (certified EBs are known to >25% of honest stake even under 50% adversarial stake) is then stated directly in terms of stake, not voter count. Under head-count quorum, a coalition of many small committee members could satisfy the threshold while representing far less stake than a smaller set of large pools — the quorum would not measure what the security property actually requires. With a stake threshold, the certificate directly proves the predicate the diffusion-period safety argument relies on. This also gives a natural constraint between the two parameters: $\sigma_c > \tau$, i.e. the committee must cover more stake than the quorum demands, or certification is impossible. At $\sigma_c = 0.99$ and $\tau = 0.75$ there is a 24-percentage-point margin between what the committee covers and what the quorum requires.
4. **Inclusivity framing.** This deserves direct acknowledgement rather than soft-pedaling. Stake-based truncation is, in Paul Clark's terms, *"exclusive — only the Big Boys get it usually."* On mainnet at epoch 612, 916 of ~3,000 registered SPOs would be eligible to vote at 99% cumulative-stake coverage; the remaining ~2,000 small pools (covering the residual ~1% of stake) would not. We argue this is acceptable for three reasons: (i) the 75% stake quorum is determined by stake, not voter count, so the excluded tail cannot threaten quorum; (ii) no incentives are tied to casting votes in the current spec, so exclusion does not translate to lost rewards; (iii) "voting eligibility" is not the same primitive as "decentralization of block production" — every SPO retains full Praos block-production rights regardless of voting committee membership. wFA^LS achieves formal inclusivity through NPV sampling, but this is *expected* inclusivity over many rounds, not per-round inclusion, and the size cost is substantial. We invite challenge on this position.
5. **Vote-traffic confirmation at full mainnet scale.** Mainnet has ~3,000 SPOs. Network simulations have so far run at 750 nodes and show all three schemes well within the $L_{\text{vote}}$ budget. Larger-scale simulations (1,500–2,000 nodes) are in progress and this PR will be updated with those results before merge. Under stake-based truncation, vote count per EB is determined by the stake distribution's tail and not by total pool count, so we do not expect qualitative change at full scale; under All-vote, vote count per EB scales linearly in pool count and full-scale confirmation is more material to that scheme's viability than to ours. wFA^IID is named in *Alternatives & Extensions* as the staged fallback should vote-diffusion pressure ever materialize against stake-based.