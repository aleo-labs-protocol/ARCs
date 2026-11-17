---
arc: 48
title: Deployment Freeze by Validator Vote
authors: Aleo Labs
discussion: https://github.com/aleo-labs-protocol/ARCs/discussions/148
topic: Protocol
status: Active
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

- A new, non-upgradable program, `deployment_freeze.aleo`, records validator votes. The vote that takes support past two thirds of committee stake sets a freeze window.
- From consensus version `V_F`, every deployment (new program, upgrade, or amendment) that reaches a block during a freeze is **rejected without being verified, and its fee is charged**.
- Nodes stop accepting deployments from the block after the vote passes. The freeze starts 60 blocks (~3 minutes) later, so deployments already in flight are not caught.
- A freeze lasts at most ~7 days and expires unless renewed. The same quorum can thaw it early.
- A freeze affects only deployments: not executions, transfers, staking, or existing programs.

## Motivation

Today the only way to stop deployments is a new consensus version, a release, and a multi-day upgrade cycle. That is too slow for:

1. **Rolling out changes to deployment verification.** Verification synthesizes every function, so it exercises curve, field, and gadget code. A correct change there can still change a deployment's verdict. For example, aleo-labs-protocol/snarkVM#3644 changed `Affine::mul_bits`; on off-curve witnesses that synthesis produces, old and new code panic for different inputs, so old and new nodes would disagree on some deployments and fork. It was reverted (#3675) before release. A consensus-version guard for code this deep is very invasive. Instead, validators can freeze deployments, upgrade, and thaw: while both versions are live, nobody verifies a deployment.
2. **Denial of service through deployment verification.** Type checking and synthesis have had bugs that let crafted programs take exponential time or crash nodes (snarkVM#3357, #3372, #3447, and Immunefi reports). Frozen deployments are refused at the mempool and never checked, giving relief while a fix ships. This does not cover parser bugs (parsing precedes the freeze check) or an already-halted chain.
3. **Emergency proof-system upgrades.** A published fix reveals the flaw before validators upgrade, and deploying a helper program that calls a victim program can be an attacker's tool. A freeze voted before publication removes it. The Varuna V3 upgrade (`V21`, 2026-10-02) was such a case; later analysis by Provable engineers showed the V2 flaw was not feasibly exploitable, but that was unknown during the rollout.

## Specification

### Parameters

| Parameter | Value |
|---|---|
| Threshold (freeze and thaw) | `total * 2 / 3 + 1` of committee stake (`Committee::quorum_threshold`) |
| `MAX_FREEZE_BLOCKS` | 201,600 (~7 days) |
| `FREEZE_DELAY` | 60 blocks (~3 minutes) |
| `VOTE_LIFETIME` | 28,800 blocks (~1 day) |
| `MAX_FROZEN_DEPLOYMENTS_PER_BLOCK` | 1 |
| Committee list length | 128 |

Block counts assume ~3 seconds per block.

### The freeze program

`deployment_freeze.aleo` is an ordinary Aleo program whose constructor rejects every edition after 0.

| Mapping | Key | Value | Meaning |
|---|---|---|---|
| `freeze` | `0u8` | `Freeze { start: u32, end: u32 }` | Frozen when `start <= height < end`. Absent or `end = 0` means not frozen. **The only value consensus reads.** |
| `votes` | validator `address` | `Vote { proposal: field, cast_at: u32 }` | Each validator's current vote. |
| `nonce` | `0u8` | `u64` | Incremented each time a proposal passes. |

| Function | Effect when it passes |
|---|---|
| `vote_freeze(duration: u32)`, `duration > FREEZE_DELAY` | `end = block.height + min(duration, MAX_FREEZE_BLOCKS)`. If a passed freeze has not yet ended, `start` is kept; otherwise `start = block.height + FREEZE_DELAY`. |
| `vote_thaw()` | `start = 0`, `end = 0` |

Both functions take the full committee list (`members: [address; 128]` plus a count) and pass `self.caller` to finalize, which:

1. **Records the vote.** Requires `self.caller` in `credits.aleo/committee`, then sets `votes[self.caller] = (BHP256(action, duration, nonce), block.height)`, with `duration = 0` for a thaw.
2. **Checks the list is the whole committee.** The count equals the committee size in `credits.aleo/metadata` and is at most 128; every address is in the committee; addresses cast to `field` are strictly increasing.
3. **Tallies.** `total` is the sum of `credits.aleo/delegated[v]` over the list. `support` sums the same over validators whose vote matches this proposal and is younger than `VOTE_LIFETIME`.
4. **Applies.** If `support >= total * 2 / 3 + 1`, applies the action and increments `nonce`.

Finalize processes all 128 slots and masks those at index `>= count`, because it cannot skip registers after a `branch`.

Properties:

- The crossing vote applies the result; there is no separate tally transaction. Stake is read fresh on each vote, so any validator can re-vote to recount.
- A new freeze replaces the current one, so renewal (longer or shorter) is just another vote.
- The nonce invalidates older votes, so a thaw cannot be undone by stale freeze votes. Validators that leave the committee stop counting.
- Votes count for `self.caller`, not `self.signer`, so a program a validator happens to sign for cannot cast its vote.
- The tally uses the committee in `credits.aleo` at finalize time, which can briefly differ from the committee AleoBFT uses. This is harmless: the result is one committed value that every node reads identically.

A proof of concept measured 2,587 finalize commands and 3 mapping writes per function, a cost of about 0.08 credits (76,167 microcredits), and a program size of ~110 kB (deployable from `V14`). A committee above 128 would need a new program.

#### Pinning

The program is deployed **before** the release that activates `V_F`, so its name cannot be taken. The release pins the program ID, checksum, and edition 0. If the program on chain does not match at `V_F`, nodes treat the network as never frozen. The pin ignores the amendment count: an amendment changes only verifying keys, which must still pass the certificate check against the pinned program text, so it cannot change behavior.

From block `V_F` (gated on the height of the block being built), the program is locked like `credits.aleo`: `VM::check_transaction` refuses any deployment naming it, and `finalize_deployment` refuses any amendment to it. No vote can change the program's rules; that requires a new program and a network upgrade.

### Consensus rule

For a block at height `H` with `CONSENSUS_VERSION(H) >= V_F`, let `F = deployment_freeze.aleo/freeze[0u8]` in the **committed** state after block `H − 1`. If `F.start <= H < F.end`, every `Transaction::Deploy` in the block is **frozen**.

A frozen deployment is checked without touching its program, circuits, or keys. It is **aborted** (as an invalid transaction is today) unless all of the following hold:

- It is within `MAX_TRANSACTION_SIZE`, has a unique ID, and passes the one-deployment-per-fee-payer rule.
- Fewer than `MAX_FROZEN_DEPLOYMENTS_PER_BLOCK` frozen deployments precede it in the block.
- It is deployment version 2 or later, so its ID covers the verifying keys.
- The owner signature verifies, and any named program owner matches.
- The fee names this deployment ID and its proof verifies.
- The fee is at least the **frozen minimum**: the storage, synthesis, and namespace terms of the deployment cost, using the circuit sizes claimed in the verifying keys and checked arithmetic (overflow aborts). The constructor term is omitted because it requires building the stack.

Otherwise it is **rejected**: stored as a `RejectedDeploy`, its fee charged in full, and `finalize_deployment` not run. It counts toward the per-block cap only, not toward per-program rules or synthesis density.

#### Implementation

Against snarkVM `staging` at `7e48bc69c`:

- **`VM::should_abort_transaction`** (`synthesizer/src/vm/finalize.rs:1210`), run by builder and checker: return a new "reject frozen" outcome after the per-block rules and before the `V16` compute-spend check, which would build the stack. Runs every check above except the fee proof and ledger uniqueness.
- **`VM::prepare_for_speculate`** (`:1410`), builder only: check the fee proof and ledger uniqueness, never `check_transaction`, so `Process::verify_deployment` never runs. The checker covers these in `check_transactions`, as for any `RejectedDeploy`.
- **`VM::atomic_speculate_inner`** (`:707`): convert frozen deployments to rejected deployments ahead of `finalize_deployment`.
- **`VM::check_speculate`** (`:200`): before `check_transactions`, reject a block that lists a frozen deployment as accepted.

`F` is read once per block, at the start of `VM::speculate` and `VM::check_speculate`, with `FinalizeStore::get_value_confirmed`, and passed to every check. So a vote passed in block `h` takes effect from `h + 1`, and the order of votes and deployments within a block never matters.

#### Design rationale

- **Reject and charge, not abort.** Aborted transactions are stored as IDs only and pay nothing, and `max_aborted_transactions` equals block capacity. Aborting would let any validator ignoring node policy fill blocks with free deployments, a denial-of-service vector created by the freeze itself. Rejection is equally safe for rollouts: no node on either release verifies the deployment, and every check that runs is identical across releases. Because a frozen deployment's only compute is a fee check, the block spend limit does not bound them; the per-block cap of 1 does.
- **`FREEZE_DELAY`.** A caught honest deployment loses its whole fee, including the name premium (100,000 credits for a 5-character name). A certificate can stay in the DAG until it falls `MAX_GC_ROUNDS` (100) rounds behind the last commit, and each block advances the committed round by at least 2, so 50 blocks after the vote every earlier certificate is committed or dropped. The extra 10 blocks cover nodes that have not yet applied the vote's block.
- **A consensus rule, not a signing policy.** Deployments certified before a freeze are already in the DAG, and block validity must not depend on any node's local signing policy.

### Node policy (snarkOS, not consensus)

From the block after a freeze passes until `end`, nodes:

- refuse deployments immediately, before fee or proof checks, at the REST broadcast route, `check_transaction_basic`, and peer-receive paths, returning an error that names `end`;
- never admit or gossip them;
- drop deployments queued before the freeze, without charge.

Wallets and SDKs should check `GET /{network}/program/deployment_freeze.aleo/mapping/freeze/0u8` before deploying. This policy assumes snarkOS checks transactions before admitting or gossiping them (verify-before-gossip and fee-first admission), a prerequisite for this ARC.

### `snarkos` command

```
snarkos developer deployment-freeze status --endpoint <node>
snarkos developer deployment-freeze freeze --private-key-file <key> [--duration 28800] --endpoint <node> --broadcast
snarkos developer deployment-freeze thaw   --private-key-file <key> --endpoint <node> --broadcast
```

`status` shows the freeze, nonce, and support for each open proposal. `freeze` and `thaw` fetch and sort the committee, then build and broadcast the vote, with the same fee options as `developer execute`. A single default duration keeps validators' votes matching. Operators should keep a few credits in the validator's public balance for vote fees, since rewards are auto-bonded.

### Activation

1. Deploy `deployment_freeze.aleo` and confirm its checksum.
2. Release `V_F` with the consensus rule, the pin, and the `snarkos` command.

The release must not ship before step 1 is confirmed: from `V_F`, the program can never be deployed, and a squatted name never matches the pin. Votes cast before `V_F` take effect at `V_F` if their window is still open.

### Usage

**Emergency:** validators are alerted through the usual upgrade channels, each runs `freeze`, renews if needed, and runs `thaw` once the fix is live. Operators should practice on testnet.

**Rollout:** a consensus-version guard remains the default. A freeze is for changes where a guard is impractical, mainly curve and field code inside the gadgets. The precondition is that no deployment already on chain changes verdict under the new code (via a guard or a census of confirmed deployments). Then validators freeze, upgrade, and thaw once non-upgraded stake is negligible. Release notes requesting a freeze should explain why a guard was not used.

### Test Cases

- A frozen deployment is rejected with its fee charged and never reaches `Process::verify_deployment`. With a deployment that panics in verification, old and new code build the same block.
- A frozen deployment is aborted, uncharged, for each failed check: bad fee proof, fee below the minimum, fee for another ID, bad owner signature or program owner, version 1, a second frozen deployment in the block, or claimed circuit sizes that overflow. A fee exactly at the minimum is rejected and charged.
- Builder blocks with frozen `RejectedDeploy`s pass `check_speculate` without verifying the deployment. A block carrying a frozen deployment as accepted fails before `check_transactions`.
- A frozen deployment's stack is never built, and an execution of an existing program `P` after a frozen deployment of `P` in the same block is accepted.
- Boundaries: heights `h − 1` to `h + 2` around a vote in block `h`, with a vote and a deployment in the same block in both orders; `start` and `end`; a deployment certified before the vote and committed as late as GC allows; renewals that shorten, or that keep `start` during the freeze or its delay; activation at `V_F`, including a freeze voted earlier and a pin mismatch; and refusal of any deployment of the program from block `V_F`.
- Program: non-committee votes fail; forged lists fail (missing, duplicate, unsorted, non-member, wrong count, count above 128); support one microcredit below the threshold fails and at the threshold passes; different durations do not combine; expired, pre-nonce, and departed-validator votes do not count; and a 128-entry vote fits under `TRANSACTION_SPEND_LIMIT`.
- Node: during a freeze, clients and validators refuse deployments over REST and from peers without admitting or forwarding them, and accept them again after a thaw or expiry. `status` matches the program's tally.

## Reference Implementations

A proof of concept of `deployment_freeze.aleo` produced the measurements above. Links to it and to the snarkVM and snarkOS changes will be added.

## Dependencies

- **snarkVM:** consensus version `V_F`, the consensus rule, and the program pin and lock.
- **snarkOS:** node policy, the `deployment-freeze` command, and verify-before-gossip / fee-first admission.
- **Wallets, SDKs, explorers:** should surface the freeze state.

### Backwards Compatibility

This is a consensus change active from `V_F`; validators must upgrade before it as for any consensus version. Frozen deployments use the existing `RejectedDeploy` format, so no data format changes.

## Security & Compliance

- **Power granted.** A two-thirds quorum can stop deployments for up to 7 days at a time and renew. It cannot affect executions, transfers, or staking, or change the program. That quorum can already halt the chain or censor any transaction, so the vote adds no new power, only a faster, uniformly enforced way to use it.
- **Impact.** Deployments are rare: 102 in the 90 days to 2026-11-05 (no deployments on 54 of those days), and none on 260 of the past 365 days. During a freeze, program owners cannot ship upgrades (even incident fixes), and height-bounded upgrade windows can be missed. Validators should announce each freeze with its reason and end height.
- **Validator failures.** A compromised key casts one vote and cannot reach the threshold alone. If more than a third of stake is unreachable, no freeze can pass, and any active freeze expires on its own.
- **Proof-system flaws.** Votes are executions, so an attacker able to forge proofs could forge a thaw. Validators should watch `freeze[0u8]` during incidents and re-vote if it changes unexpectedly.
- **Scope.** A freeze does not affect already-deployed programs (including helpers deployed earlier, or dynamic dispatch), `snark.verify` with runtime verifying keys, or built-in circuits. It narrows an attacker's options; it is not a fix.

### Alternatives considered

- **Guardian set:** faster, but needs seat selection, key custody, and a capture override (itself a validator vote).
- **Off-chain signature bundle:** one transaction, but needs a collector and a large signature-verification circuit.
- **Validator-local flag:** no consensus change, but cannot make rollouts safe.
- **Abort instead of reject:** opens a free path onto the chain.
- **Built-in program:** avoids pinning, but needs new machinery and moves state out of an auditable program.
- **Exempt key-only amendments:** amendments still run synthesis and the certificate check, which are exactly what Motivations 1–3 cannot trust.
- **Tally in consensus code:** the freeze window would derive from vote history instead of one committed value.

### Open questions

1. **Lower freeze threshold.** One third of stake would be faster, but would let such a coalition censor deployments without halting the chain, a new power.
2. **Default duration.** One day by default; rollouts may want 12 hours and emergencies longer.
3. **`VOTE_LIFETIME`.** One day spans time zones; shorter limits staleness.
4. **Peer message version.** Bumping it at `V_F` would stop old nodes relaying frozen deployments, at the cost of disconnecting every non-upgraded node.

## References

- aleo-labs-protocol/snarkVM#3644 and #3675: `Affine::mul_bits` change and revert
- aleo-labs-protocol/snarkVM#3357, #3372, #3447: type-checking performance fixes
- [ARC-6: Program Upgradability](../arc-0006)
- [ARC-7: Stake-Weighted Governance on Aleo](../arc-0007)
