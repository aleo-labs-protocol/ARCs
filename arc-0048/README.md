---
arc: 48
title: Deployment Freeze by Validator Vote
authors: ProvableHQ
discussion:
topic: Protocol
status: Draft
created: 2026-11-17
signature_domain: aleo-gov-48-v1
pass_threshold: 66
quorum_threshold: 66
voting_start: 23855000
voting_end: 24157000
snapshot: 24157000
---

## Abstract

This ARC lets validators pause program deployments for a bounded time by an on-chain, stake-weighted vote, without a network upgrade.

- A new, non-upgradable program, `deployment_freeze.aleo`, records validator votes and holds one value: the freeze window (`start`, `end`). The vote that takes support past two thirds of committee stake sets it. Each validator votes with a single `snarkos` command.
- From consensus version `V_F`, every deployment (new program, upgrade, or amendment) that reaches a block during a freeze is **rejected without being verified, and its fee is charged**. A freeze never lets a transaction onto the chain for free.
- Nodes stop accepting deployments from the block after a vote passes. The freeze itself starts 60 blocks (~3 minutes) later, so deployments already in flight are not caught.
- A freeze lasts at most ~7 days and expires on its own unless renewed. The same quorum can thaw it early.
- A freeze only stops deployments. It cannot pause executions, move funds, or change programs.

Installing the mechanism needs one network upgrade. After that, a freeze takes as long as it takes validators holding two thirds of stake to run one command, instead of a multi-day release and upgrade cycle.

## Motivation

Today the only way to stop deployments is a consensus change: a new consensus version, a release, and a wait for validators to upgrade. That is too slow for the three cases below.

### 1. Rolling out changes to deployment verification

Deployment verification synthesizes every function of the deployed program, so it exercises a large amount of low-level code: curve arithmetic, field operations, and circuit gadgets. A correct change to that code can still change whether a deployment is accepted.

For example, snarkVM#3644 changed `Affine::mul_bits`. The new code is correct for every point on the curve, but synthesis also feeds it off-curve witnesses (a `cast` of an `x` with no group element witnesses `(x, 0)`). On those inputs the old and new code panic for different values of `x`, so for some deployments old and new nodes reach different verdicts. snarkVM#3675 reverted it before any release. Had it shipped, one such deployment during the rollout would have forked the network.

The usual remedy is a consensus-version guard: keep both versions of the code and select by height. For curve and field code called deep inside the gadgets, far from anything that knows the consensus version, that is very invasive and discourages otherwise good fixes. With a freeze, validators instead freeze deployments for 12–24 hours, upgrade, and thaw. While old and new code are both live, nobody verifies a deployment, so they cannot disagree. See [For a rollout](#for-a-rollout) for when this is appropriate.

### 2. Relieving a denial of service through deployment verification

Deployment verification type-checks the program and synthesizes every function. Bugs in either can make a crafted program take enormous time to check or crash the node. Several have been found recently, including exponential-time type-checking paths (snarkVM#3357, #3372, #3447) and Immunefi reports.

If an attacker exploited such a bug persistently, the only remedy today is a patched release. A freeze gives faster relief: frozen deployments are rejected before type checking and synthesis, and nodes turn them away from their mempools as soon as the vote passes.

Limits: deployment bytes are parsed before the freeze is checked, so a parser bug is not covered, and votes must reach a block, so a freeze cannot help once the chain has halted.

### 3. Emergency upgrades to the proof system

During an emergency proof-system upgrade, the published fix reveals the flaw before validators have upgraded. Deploying a purpose-built helper program that calls a victim program can be one of an attacker's tools, since it reaches the victim's circuits in ways a direct call does not. A freeze voted before the release is published removes that tool for the whole window.

The Varuna V3 upgrade (`V21`, 2026-10-02) was such a case. Later analysis by Provable engineers showed the V2 flaw could not feasibly be used to prove a false statement, but that was not known during the rollout.

## Specification

The proposal has four parts:

1. **The freeze program**, `deployment_freeze.aleo`: records votes, tallies them by stake, and holds the freeze state.
2. **One consensus rule** from `V_F`: while frozen, reject every deployment and charge its fee, without verifying it.
3. **Node policy** in snarkOS (not consensus): refuse deployments at the first node, and never admit or gossip them.
4. **A `snarkos` command** for validators to vote and inspect the tally.

### Parameters

| Parameter | Value |
|---|---|
| Threshold (freeze and thaw) | `total * 2 / 3 + 1` of committee stake (same as `Committee::quorum_threshold`) |
| `MAX_FREEZE_BLOCKS` | 201,600 (~7 days) |
| `FREEZE_DELAY` | 60 blocks (~3 minutes) |
| `VOTE_LIFETIME` | 28,800 blocks (~1 day) |
| `MAX_FROZEN_DEPLOYMENTS_PER_BLOCK` | 1 |
| Committee list length | 128 |

Block counts assume ~3 seconds per block, the mainnet average from May to October 2026.

### The freeze program

#### State

| Mapping | Key | Value | Meaning |
|---|---|---|---|
| `freeze` | `0u8` | `Freeze { start: u32, end: u32 }` | Deployments are frozen in blocks with `start <= height < end`. Absent or `end = 0` means not frozen. **The only value consensus reads.** |
| `votes` | validator `address` | `Vote { proposal: field, cast_at: u32 }` | The proposal the validator supports, and when it voted. |
| `nonce` | `0u8` | `u64` | Incremented each time a proposal passes. |

#### Voting

| Function | Parameters | Effect when it passes |
|---|---|---|
| `vote_freeze` | `duration: u32`, `duration > FREEZE_DELAY` | `end = block.height + min(duration, MAX_FREEZE_BLOCKS)`. If a passed freeze has not yet ended (including one still in its delay window), `start` is kept so the freeze stays continuous; otherwise `start = block.height + FREEZE_DELAY`. |
| `vote_thaw` | none | `start = 0`, `end = 0` |

Both functions take the full committee list (`members: [address; 128]` plus a count, filled in by the `snarkos` command) and pass `self.caller` to finalize, which:

1. **Records the vote.** Checks that `self.caller` is in `credits.aleo/committee`, then sets `votes[self.caller]` to `BHP256(action, duration, nonce)` (with `duration = 0` for a thaw) and the current height. A new vote replaces the old one.
2. **Checks the list is the whole committee** (programs cannot iterate mappings):
   - the count equals the committee size in `credits.aleo/metadata` and is at most 128;
   - every listed address is in `credits.aleo/committee`;
   - the addresses, cast to `field`, are strictly increasing (no duplicates; with the above, none missing).
3. **Tallies.** `total` is the sum of `credits.aleo/delegated[v]` over the list (this includes self-bond and is what the VM builds the committee from). `support` is the same sum over validators whose vote matches this proposal and is younger than `VOTE_LIFETIME`. Delegators count through their validator.
4. **Applies.** If `support >= total * 2 / 3 + 1`, applies the action and increments `nonce`.

Finalize cannot skip unused slots (a register written after a skipped `branch` is unreadable), so every vote processes all 128 slots and masks those at index `>= count`. Vote cost is therefore independent of committee size.

Consequences:

- The vote that crosses the threshold applies the result; no separate tally transaction is needed.
- Stake is read fresh on every vote. If stake moves, any validator can re-vote for the same proposal to recount.
- Freeze votes only match if they name the same `duration`, so the `snarkos` command uses one default.
- A freeze passed during another freeze replaces it, so renewal is just another vote; the new `end` may be earlier or later.
- The nonce invalidates all older votes, so a vote for a freeze overtaken by a thaw cannot later revive it. `VOTE_LIFETIME` prevents stale votes from combining with fresh ones. Validators that leave the committee stop counting.
- The tally uses the committee in `credits.aleo` at finalize time, which can briefly differ from the committee AleoBFT uses (`COMMITTEE_LOOKBACK_RANGE` = 100 rounds). This is harmless: the vote is a governance threshold over current stake whose result is one committed value every node reads identically.

#### Voter identity

Votes count for `self.caller`, which must be the validator address in `credits.aleo/committee`; validators call the program directly, as with `credits.aleo/set_validator_state`. Using `self.signer` would let any program a validator signs for cast its vote without the wallet showing it. With `self.caller`, a vote routed through another program counts for that program, which is not in the committee.

#### Limits and cost

The VM caps the committee at `MAX_CERTIFICATES` (40 on mainnet today); the list holds 128. A committee above 128 would need a new program. A proof-of-concept implementation measured:

- **Finalize size:** 2,587 commands and 3 mapping writes per vote function (limits: 65,535 and 32), reading ~384 mapping values.
- **Finalize cost:** 76,167 microcredits (`vote_freeze`) and 75,940 (`vote_thaw`), ~0.08 credits, well under the 4-credit `TRANSACTION_SPEND_LIMIT`.
- **Program size:** ~110 kB, deployable only from `V14` (mainnet is past `V14`).

#### Deployment and pinning

The program is deployed before the release that activates `V_F`, so nobody else can take the name. Its constructor rejects every edition after 0 and initializes `nonce = 0` with no freeze. The release hard-codes the program ID, checksum, and edition 0. If the program on chain does not match at `V_F`, nodes treat the network as never frozen.

From `V_F` (including block `V_F` itself, so this check gates on the height of the block being built, not the tip), the program is locked like `credits.aleo`:

- `VM::check_transaction` refuses any deployment naming it, covering new editions and amendments.
- `finalize_deployment` refuses any amendment to it.

Before `V_F`, amendments do not run the constructor, so one could be sent in that window. An amendment changes only verifying keys, keeps the program text and checksum, and each key must pass the certificate check against circuits synthesized from that text, so it cannot change program behavior. The pin therefore ignores the amendment count, so a pre-`V_F` amendment cannot switch the rule off.

The program's source is part of this ARC and will be published and reviewed before the release. No vote can change the program's rules (for example, raise `MAX_FREEZE_BLOCKS` or widen the freeze's scope); that requires a new program and a network upgrade.

### The consensus rule

For a block at height `H` with `CONSENSUS_VERSION(H) >= V_F`:

- Let `F = deployment_freeze.aleo/freeze[0u8]` in the **committed** finalize state after block `H − 1`. If absent, nothing is frozen.
- If `F.start <= H < F.end`, every `Transaction::Deploy` in the block (new program, upgrade, or amendment) is **frozen**.

A frozen deployment is first checked without touching its program, circuits, or keys. If any check fails, it is **aborted**, as an invalid transaction is today:

- **Size:** within `MAX_TRANSACTION_SIZE`.
- **Uniqueness and per-block rules:** unique ID, and one deployment per public fee payer.
- **Per-block cap:** fewer than `MAX_FROZEN_DEPLOYMENTS_PER_BLOCK` frozen deployments precede it in the block.
- **Version:** deployment version 2 or later (version 1 IDs do not cover the verifying keys, so the fee would not bind to them).
- **Owner:** the owner signature over the deployment ID verifies, and any named program owner matches.
- **Fee:** names this deployment ID, and its proof verifies (`check_fee_internal`).
- **Fee amount:** at least the **frozen minimum**, the storage, synthesis, and namespace terms of the deployment cost (the constructor term is omitted because it requires building the stack). The synthesis term uses the circuit sizes claimed in the verifying keys, which are unchecked on this path, so the guaranteed floor is storage plus namespace. It is computed with checked arithmetic; overflow aborts.

If all checks pass, the deployment is **rejected**: stored as a `RejectedDeploy`, fee charged in full, and `finalize_deployment` does not run. A frozen deployment deploys nothing, so it does not count as a deployment of its program for the per-block rules and adds nothing to the block's synthesis density. It is counted only for the per-block cap.

The block checker rebuilds rejected deployments with `Transaction::from_deployment` (which checks the owner signature and program owner) and verifies fees in `check_transactions`, so the builder must check the same things before rejecting. The checks are split as follows:

- **`should_abort_transaction`** (builder and checker): size, per-block rules, version, owner, fee deployment ID, and fee amount.
- **`prepare_for_speculate`** (builder only): fee proof and ledger uniqueness of the transaction ID. The checker covers both in `check_transactions`, as for any `RejectedDeploy`.

#### Where the rule goes

Code locations are against snarkVM `staging` at `7e48bc69c` and snarkOS `staging` at `84ccee7a1`.

- **`VM::should_abort_transaction`** (`synthesizer/src/vm/finalize.rs:1210`): return a new "reject frozen" outcome after the per-block rules and before the `V16` compute-spend block (`:1368`), which would build the stack.
- **`VM::prepare_for_speculate`** (`:1410`): run only the fee-proof and uniqueness checks for frozen deployments, never `check_transaction`, so `Process::verify_deployment` never runs.
- **`VM::atomic_speculate_inner`** (`:707`): convert frozen deployments to rejected deployments ahead of `finalize_deployment`.
- **`VM::check_speculate`** (`:200`): before `check_transactions` (`:219`), reject a block that lists a frozen deployment as accepted. Confirmed `RejectedDeploy`s need no new code: the checker verifies only the fee and rejected ID, then re-runs `atomic_speculate`, which must produce the same rejection.

`F` is read once per block at the start of `VM::speculate` and `VM::check_speculate`, using `FinalizeStore::get_value_confirmed`, and passed to every check. This ignores writes earlier in the same block and keeps builder and checker on the same value.

#### Why reject and charge, not abort

Aborted transactions are stored as IDs only and pay nothing. If frozen deployments were aborted, a validator ignoring node policy could fill its batches with them, and every node would carry and process them for free; `Transactions::max_aborted_transactions` equals a block's whole capacity, so it would not limit this. A mechanism that opens a denial-of-service vector when switched on cannot safely be used in an emergency. Under this ARC, anything that reaches a block during a freeze pays.

Rejection keeps rollouts safe just as aborting would: no node, on either release, verifies or synthesizes the deployment, and every check that does run (size, uniqueness, fee proof, cost terms) is identical across releases.

Costs of this choice:

- **Rejected deployments are stored in full** though unverified. The fee covers storage. Since a frozen deployment's only compute is a fee check, the block spend limit does not bound them, so the per-block cap of 1 does. After `FREEZE_DELAY`, any frozen deployment in a block comes from a non-compliant validator or an un-upgraded node, so one per block is enough.
- **More consensus code:** a new outcome, a fee-only check, and a reject branch. Today every rejected deployment has first been fully verified; this adds one that was not.
- **Honest deployments in flight could be caught**, addressed below.

Invalid transactions (for example, a bad fee proof) are still aborted for free during a freeze, as at any other time; a freeze does not change this.

#### Honest deployments at the boundary

A caught honest deployment loses its whole fee, including the name premium (10^(10−n) credits for an n-character name; 100,000 credits for 5 characters). Two rules prevent this:

- **Delay.** A freeze passed in block `h` starts at `h + FREEZE_DELAY`, so deployments already certified when it passes are processed normally.
- **Node policy from block `h + 1`.** Nodes stop admitting and proposing deployments. Uncertified honest deployments are dropped from mempools, uncharged, and can be resent after the thaw.

`FREEZE_DELAY` is sized for the worst case. A certificate can wait in the DAG until dropped at `MAX_GC_ROUNDS` (100) rounds behind the last committed round (snarkOS `node/bft/src/bft.rs:887`). Leaders are elected on even rounds and each block commits a later leader, so `k` blocks advance the committed round by at least `2k`. After 50 blocks, every pre-freeze certificate has been committed or dropped; 10 more blocks cover certificates created before every node applied block `h`.

During the delay a validator ignoring node policy can still propose deployments, which are fully verified. Honest validators stop at `h + 1`, so this requires a colluding validator. The delay gives no extra warning, since votes are public from the first one.

#### Why not refuse to sign

An earlier sketch had validators refuse to sign batches containing deployments. The consensus rule is still needed because deployments certified before the freeze are already in the DAG, and block validity must not depend on any node's local signing policy. Refusing to sign also hurts liveness today: an uncertified proposer re-proposes every transmission, so one frozen deployment can stall that validator. Nodes may add a signer rule that drops refused transactions as policy, but it does not replace the consensus rule.

#### Which state is read

A freeze passed in block `h` is visible from `h + 1` and starts at `h + FREEZE_DELAY`; a thaw in block `h` admits deployments from `h + 1`. The rule never reads speculative state from inside block `H`, so the relative order of a vote and a deployment in the same block, and the order in which `prepare_for_speculate` and `atomic_speculate` see transactions, does not matter. Varuna V3 activation got this boundary wrong by one block, so the [test cases](#test-cases) cover it explicitly.

### Node policy (snarkOS, not consensus)

During a freeze a deployment should never get past the node it is first sent to. The consensus rule is the backstop, not the expected path. From the block after a freeze passes until `end`:

- **Refuse immediately.** Clients and validators check the freeze first, before fee or proof checks, mempool admission, or forwarding. REST submitters get an error naming `end`. The check is one cached read per block, applied in the REST broadcast route, `check_transaction_basic` (`node/bft/ledger-service/src/ledger.rs:501`), and on peer-receive paths.
- **No gossip.** Nodes forward only checked, admitted transactions, so frozen deployments never propagate.
- **Drop queued deployments.** Deployments admitted before the freeze passed are dropped from the mempool and not proposed. Nothing is charged.
- **Wallets and SDKs** should check `GET /{network}/program/deployment_freeze.aleo/mapping/freeze/0u8` before building a deployment. No new endpoint is needed.

This assumes nodes check transactions before admitting or gossiping them (verify-before-gossip and fee-first admission), which this ARC depends on.

### The `snarkos` command

A new subcommand alongside `snarkos developer execute`:

```
snarkos developer deployment-freeze status --endpoint <node>
snarkos developer deployment-freeze freeze --private-key-file <validator key> [--duration 28800] --endpoint <node> --broadcast
snarkos developer deployment-freeze thaw   --private-key-file <validator key> --endpoint <node> --broadcast
```

- `status` prints the freeze, the nonce, and each open proposal's supporting stake against the threshold, using existing mapping routes and `GET /{network}/committee/latest`.
- `freeze` and `thaw` fetch the committee, sort it by field cast, and build and broadcast the vote. They accept the same key, fee, and broadcast options as `developer execute` (`--priority-fee`, `--record`, `--wait`).
- If the committee changes before finalize, the list check fails and the vote aborts; the command can rebuild and resend.

A vote costs ~0.08 credits of finalize compute plus normal storage cost. Validator rewards are auto-bonded, so operators should keep a few credits in the validator's public balance, or a fee record, ready in advance.

### Activation

1. Deploy `deployment_freeze.aleo` and confirm its on-chain checksum matches what the release will pin.
2. Ship a release that adds `V_F` with the rule above, pins the program, and adds the `snarkos` command.
3. From `V_F` the rule is live; nothing is frozen until validators vote.

Order matters: from `V_F`, nodes refuse every deployment naming the program, including its first, and a squatted name never matches the pin. Either case would require another network upgrade. So the release does not ship until step 1 is confirmed, which also means the name is taken before the release signals that it matters.

Validators can vote before `V_F`. A freeze passed before `V_F` has no effect until `V_F`, and takes effect then if its window is still open. The rule stays in the code permanently so post-`V_F` blocks replay.

## Using a freeze

### In an emergency

1. Whoever finds the problem alerts validators through the usual upgrade coordination channels.
2. Each operator runs `snarkos developer deployment-freeze freeze`. The crossing vote sets the freeze; nodes stop taking deployments from the next block, and the freeze starts `FREEZE_DELAY` blocks later.
3. Validators renew before expiry if needed, and thaw once the fix is live.

Speed depends on how quickly operators holding two thirds of stake each run one command. Operators should keep fee funds ready and practice on testnet.

### For a rollout

A freeze is not the default way to ship changes to deployment verification; a consensus-version guard remains the default whenever it can be applied cleanly. A freeze is for cases where a guard is invasive, mainly curve and field code called deep inside the gadgets, as in Motivation 1. Those cases are risky because synthesis can create off-curve points. The lasting fix is to remove that dependence (for example, synthesizing verification circuits without witness values), after which rollout freezes should be rare. Release notes that request a freeze should say why a guard was not used.

Procedure for a change like snarkVM#3644:

1. **Precondition:** no deployment already on chain changes verdict under the new code, either via a guard or a census of all confirmed deployments. A freeze only covers the window when both versions are live, not later replay.
2. Validators freeze for the expected upgrade window plus margin.
3. Validators upgrade.
4. Validators thaw once non-upgraded stake is negligible. A validator still on old code after the thaw can fork on the first divergent deployment, the same risk as missing any upgrade.

`V_F` must be reached before the protected release ships, and every node running during the freeze must enforce it.

### What a freeze does not do

A freeze stops new circuits from being deployed. It does not affect:

- **Already-deployed programs**, alone or composed (including via dynamic dispatch). A helper program deployed before the freeze remains usable.
- **`snark.verify` in finalize**, whose verifying key is a runtime value, so existing programs using it accept caller-chosen circuits.
- **Built-in circuits** (`credits.aleo`, inclusion, fee).
- **The first emergency**: the mechanism must be live before it is needed.

In an incident like Motivation 3, a freeze narrows the attacker's options; it is not a fix.

### Test Cases

**Consensus rule**

- A frozen deployment is rejected with its fee charged and never reaches `Process::verify_deployment`. With a deployment that panics in verification, the block still builds, and old and new verification code produce the same block.
- A frozen deployment is aborted and uncharged if its fee proof fails, its fee is below the frozen minimum, its fee names another ID, its owner signature or program owner is wrong, or it is version 1. A fee of exactly the frozen minimum is rejected and charged.
- Every builder-produced block with a frozen `RejectedDeploy` passes `check_speculate`, and the checker does not verify the deployment.
- A block carrying a frozen deployment as confirmed fails `check_speculate` before `check_transactions`, without verifying it.
- A frozen deployment is handled before the compute-spend check, so its stack is never built.
- During a freeze, a frozen deployment naming existing program `P` followed by an execution of `P` in the same block accepts the execution.
- A second frozen deployment in a block, with a different fee payer, is aborted.
- Verifying keys claiming circuit sizes that overflow the synthesis term cause an abort, not a panic.

**Boundaries**

- `FREEZE_DELAY`: a deployment certified in the block where the freeze passes is processed normally; one in block `start` is rejected. A pre-freeze certificate held back and committed as late as GC allows is processed normally.
- A vote in block `h` takes effect at `h + 1`. Test heights `h − 1` to `h + 2`, with a vote and a deployment in the same block in both orders.
- Expiry at `end`, including a shortening renewal, a renewal during a freeze that keeps `start`, and a renewal during the delay window that keeps `start`.
- A freeze passed before `V_F` has no effect before `V_F` and takes effect at `V_F` if still open.
- The rule is inactive before `V_F`, and at `V_F` if the pinned checksum does not match.
- From block `V_F` itself, a new edition or amendment of `deployment_freeze.aleo` is refused in both `check_transaction` and `finalize_deployment`.

**Freeze program**

- Votes from non-committee addresses fail.
- Forged committee lists fail: missing member, duplicate, wrong count, unsorted, non-member, or count above 128.
- Support one microcredit below threshold does not pass; support at threshold does.
- Votes for different durations do not combine.
- Votes older than `VOTE_LIFETIME`, from before the last passed proposal, or from validators that have left the committee do not count (tested with a copy of the program using a short `VOTE_LIFETIME`).
- A full 128-entry vote fits under `TRANSACTION_SPEND_LIMIT`.

**Node and CLI**

- The `snarkos` command builds a valid list from `/committee/latest`, and `status` matches the program's tally.
- During a freeze, clients and validators return an error immediately for a REST-submitted deployment, refuse one from a peer, and neither admit nor forward it. Both accept deployments again after a thaw or expiry.

## Reference Implementations

A proof-of-concept implementation of `deployment_freeze.aleo` produced the measurements above. Links to the program source and the snarkVM and snarkOS changes will be added here.

## Dependencies

- **snarkVM:** new consensus version `V_F`, the consensus rule, and the program pin and lock.
- **snarkOS:** node policy, the `deployment-freeze` command, and verify-before-gossip / fee-first admission as a prerequisite.
- **Wallets, SDKs, and explorers:** should surface the freeze state. No format changes.

### Backwards Compatibility

This is a consensus change active from `V_F`; earlier blocks are unaffected, and validators must upgrade before `V_F` as for any consensus version. Frozen deployments that reach a block use the existing `RejectedDeploy` format, so wallets, explorers, and storage need no changes.

## Security & Compliance

### What a quorum can do

At most, validators holding two thirds of stake can stop deployments, upgrades, and amendments for up to 7 days at a time, and renew. They cannot freeze executions, transfers, or staking, or change the program. That quorum can already halt the chain or censor any transaction, so the vote grants no new power; it only lets the quorum act without a release, in a way every node enforces. Validators below the quorum can neither freeze nor thaw.

### Side effects

Deployments are rare, so few users should notice a freeze. At height 23,519,863 (2026-11-06) mainnet had 646 programs. In the 90 days to 2026-11-05 there were 102 deployments (new programs and upgrades), about one a day; 54 of those days had none, and the busiest had 10. Over the past year, 260 of 365 days had no deployments. Executions are unaffected.

Program owners should be aware that:

- They cannot ship upgrades during a freeze, even legitimate ones such as adding a pause control in response to the same incident.
- A constructor that allows upgrades only before or after a given height can miss its window.

Validators should announce each freeze with its reason and end height.

### Compromised or unavailable validators

- **A compromised validator key** can cast that validator's vote and nothing more; no single validator reaches the threshold, and such a key already permits far worse.
- **Unreachable validators:** if more than one third of stake does not vote, a freeze cannot pass. Any freeze in force expires on its own, so it can never become permanent by accident.

### Forged votes during a proof-system flaw

Votes are executions verified by the same proof system. An attacker able to forge proofs for this program could forge a thaw. A thaw is a public change to one mapping value, so validators should watch `freeze[0u8]` during an incident and re-vote if it changes unexpectedly.

### Alternatives considered

- **A separate guardian set** (loosely modeled on Aave guardians). Faster to act, but needs seat selection, key custody and rotation, and an override for capture, which was a validator vote anyway. Validators already vote on ARCs, so a direct vote is simpler and adds no new key holders.
- **An off-chain signature bundle** submitted in one transaction. Needs a collector and a circuit large enough to verify the whole committee's signatures, for a small latency gain.
- **A validator-local flag.** No consensus change, but only effective as far as adopted, and it cannot make rollouts safe.
- **Abort frozen deployments.** Simpler and free for caught honest deployments, but opens a free path onto the chain (see [Why reject and charge, not abort](#why-reject-and-charge-not-abort)).
- **A built-in program like `credits.aleo`.** Avoids name pinning, but needs new machinery to initialize built-ins at a height and moves state out of an ordinary, auditable program.
- **Exempt key-only amendments.** Amendments still run synthesis and the certificate check, so they are exposed to Motivations 1 and 2, and in Motivation 3 the certificate check is exactly what cannot be trusted. Programs can re-key after the thaw.
- **Tally in consensus code.** Avoids the committee list, but the freeze window would have to be derived from vote history instead of one committed value.

### Open questions

1. **Lower freeze threshold.** More than one third of stake would be faster, and a freeze is cheap to undo. But today such a coalition cannot exclude deployments without halting the chain; a lower threshold would give it a new censorship power.
2. **Default duration.** The `snarkos` default is one day; a rollout may want 12 hours and an emergency longer.
3. **`VOTE_LIFETIME`.** One day lets votes gather across time zones; shorter limits vote staleness.
4. **Peer message version.** Bumping it at `V_F` would disconnect peers that do not enforce the freeze, stopping old nodes from relaying frozen deployments, but would disconnect every non-upgraded node.

## References

- aleo-labs-protocol/snarkVM#3644: `Affine::mul_bits` change (reverted)
- aleo-labs-protocol/snarkVM#3675: revert of #3644
- aleo-labs-protocol/snarkVM#3357, #3372, #3447: type-checking performance fixes
- [ARC-6: Program Upgradability](../arc-0006)
- [ARC-7: Stake-Weighted Governance on Aleo](../arc-0007)
- [ARC-9: Dynamic Dispatch](../arc-0009)
