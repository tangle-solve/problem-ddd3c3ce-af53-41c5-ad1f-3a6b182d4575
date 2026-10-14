<!-- solve-problem:ddd3c3ce-af53-41c5-ad1f-3a6b182d4575 -->
# Validation work versus propagation priority

A [Tangle Solve problem](https://solve.tangle.tools/problems/ddd3c3ce-af53-41c5-ad1f-3a6b182d4575).

## What is the problem?

In proof-of-stake Ethereum (and similar gossip-based blockchains), a node that receives a block header signed by the (currently or recently) legitimate proposer must decide how much computation to speculatively spend validating and executing that block *before* it is fully authenticated (e.g., before all transactions or state witnesses arrive, and before the node knows whether the block will end up canonical). Yaish et al. (2024) showed adversaries can exploit this: a validator with proposal rights (or a recently-slashed/rotated one) can issue blocks whose *cheap-to-verify signature* triggers *expensive* speculative work — huge transaction lists, gas-heavy execution, witness reconstruction — on every honest node simultaneously. The result is computation exhaustion: honest nodes fall behind and miss consensus deadlines (attestation slots, fork-choice updates), causing liveness failures.

The open problem is to design and analyze a **scheduler / resource-allocation policy** for block validation with the following informal specification:

- **State:** each node has limited CPU (and memory/IO), with a stream of incoming tasks: consensus-critical traffic (attestations, fork-choice, block headers), peer gossip, and speculative block execution of varying cost.
- **Adversary:** a bounded-stake adversary can inject adversarial blocks whose validation cost is arbitrarily high relative to honest blocks, at times of its choosing; client hardware is heterogeneous.
- **Goal:** a policy that (a) provably bounds the delay adversarial speculative work can impose on consensus traffic, and (b) degrades gracefully — honest blocks still validate within their slot whenever the adversary is below some resource/stake threshold.

Ambiguity note: the census calls this "engineering," so both a deployed-software solution (e.g., priority scheduling with admission control in Ethereum clients) and a formal model with proven bounds would count; the precise threat model (which keys count as "untrusted" after proposer rotation) must be pinned down.

## Why does it matter?

A solution directly protects consensus liveness of Ethereum-class systems against a demonstrated, cheap attack using only legitimate protocol messages; Yaish et al. showed real client behavior is exploitable today. It would also give client teams (Geth, Lighthouse, Prysm, etc.) a principled replacement for ad-hoc timeouts. More broadly, it connects blockchain networking to the classical theory of adversarial scheduling and resource isolation (fair scheduling, DoS-resistant queuing), potentially yielding reusable results for any gossip protocol where cheap authentication gates expensive work.

## What's already known?

- **Yaish et al. (USENIX Security 2024)** demonstrated speculative DoS: adversaries exploit that validation is triggered by an authenticated header; they measured client resource consumption and showed consensus messages can be starved. They discuss mitigations but (to our knowledge) do not give a proven scheduling policy — verify what mitigations they evaluate, as this is your baseline.
- **BIP 152 (Corallo, 2016)** — compact block relay — reduces *bandwidth* for block propagation (transaction-by-short-ID announcement), showing the precedent of restructuring what a receiver must do per block, but it does not address CPU scheduling of execution.
- Ethereum clients use various ad-hoc mechanisms: block timeouts, execution parallelization, fee caps on calls during sync, and (post-Deneb) blob-carrying sidecars. None of these is a principled adversarially-analyzed scheduler.
- Adjacent literature: DoS resistance in gossip (e.g., Bitcoin's "netting" / peer banning), OS fair-share scheduling (lottery/stride scheduling, CFG scheduling), and network DoS queuing theory. No published result (verify) gives a bound like "adversary spending X block proposals can delay attestations by at most f(X, hardware)."

## What would count as solving it?

Any of the following, ideally all three:

1. **Formal model + bound:** A queueing/scheduling model where an adversary controls arrival times and validation costs of speculative tasks, with a proven bound on worst-case delay of consensus tasks, and a matching lower bound showing any policy without resource isolation must suffer unbounded delay.
2. **Policy/construction:** A concrete scheduler (priority classes, preemption granularity, admission control, cost estimation, budget-per-proposer) implementable in heterogeneous clients, with the analysis above applying to it.
3. **Empirical validation:** Implement in a major client (or test harness with heterogeneous hardware, e.g., Whiteblock/Madow-style testnets), replay the Yaish attacks, and demonstrate consensus traffic deadlines are met while honest-block latency degrades only mildly.

## How to start

1. **Read Yaish et al. (2024) first** — the attack section defines the exact attack surface (which messages trigger which work, which clients are vulnerable) and their mitigation discussion is the natural jumping-off point.
2. **Model the node as a queuing system:** two or three priority classes (consensus-critical, honest-appearing block work, unknown work) with adversarially chosen service demands. Try weighted fair queuing / stride scheduling and see where the bound breaks (cost is unknown before execution — the "speculative" part means you can't know a task's cost a priori; this is the core obstacle).
3. **Handle heterogeneity:** any budget in wall-clock terms must scale with hardware; consider cost units measured in local gas/executed-instructions instead.
4. **Check for prior art** in Ethereum client repos (issue trackers for "block validation timeout," "priority executor") — much mitigation discussion may exist informally but unpublished.

## Key references

- Aviv Yaish, Kaihua Qin, Liyi Zhou, Aviv Zohar, Arthur Gervais. *Speculative Denial-of-Service Attacks in Ethereum.* USENIX Security 2024. https://www.usenix.org/conference/usenixsecurity24/presentation/yaish — Demonstrates that adversaries with (temporary) proposal rights can force expensive speculative validation on all honest nodes, exhausting computation and delaying consensus.
- Matt Corallo. *BIP 152: Compact Block Relay.* 2016. https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki — Bitcoin's protocol for relaying blocks by short transaction IDs to cut bandwidth; precedent for restructuring per-block receiver work, not a scheduling solution.
- (verify) Waldspurger & Weihl, *Lottery/Stride Scheduling* (1994–95) — classical fair-share CPU scheduling; likely the right formal toolkit for the priority/isolation analysis.

## Warnings

- **Cost is unknown before work is done:** the essence of "speculative" — you cannot admit-control on true validation cost, only on estimates (gas limit, tx count, witness size). Any model assuming known service demand is solving the wrong problem.
- **Don't break honest performance:** Ethereum slots are ~12s with tight attestation deadlines; a policy that delays honest blocks by even a second matters. Report honest-case latency, not just adversarial-case bounds.
- **Threat-model subtlety:** the attacker may be a *legitimate current or recently-rotated proposer*; peer-banning and slashing-style punishments are hard to apply. Distinguish "untrusted" carefully.
- The census record is sparse; treat the precise scope (Ethereum-specific vs. general gossip blockchains) as a decision you must make and state explicitly.

---
*`BCGT-000273` · flagged for review*

## Acceptance guidance

Any of the following, ideally all three:

1. **Formal model + bound:** A queueing/scheduling model where an adversary controls arrival times and validation costs of speculative tasks, with a proven bound on worst-case delay of consensus tasks, and a matching lower bound showing any policy without resource isolation must suffer unbounded delay.
2. **Policy/construction:** A concrete scheduler (priority classes, preemption granularity, admission control, cost estimation, budget-per-proposer) implementable in heterogeneous clients, with the analysis above applying to it.
3. **Empirical validation:** Implement in a major client (or test harness with heterogeneous hardware, e.g., Whiteblock/Madow-style testnets), replay the Yaish attacks, and demonstrate consensus traffic deadlines are met while honest-block latency degrades only mildly.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/ddd3c3ce-af53-41c5-ad1f-3a6b182d4575?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000273.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000273)
