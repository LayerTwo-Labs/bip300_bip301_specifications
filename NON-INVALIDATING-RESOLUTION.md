# Non-Invalidating Resolution for BIP-300/301

How far block rejection can be eliminated, what the irreducible floor is, and
why that floor exists.

Companion to [`ENFORCER_INVALIDATEBLOCK.md`](./ENFORCER_INVALIDATEBLOCK.md),
which enumerates every condition under which the enforcer currently causes a
block to be invalidated.

---

## The proposal in one paragraph

Today, a block that violates almost any BIP-300/301 rule is invalidated
wholesale: the enforcer returns `ConnectBlockAction::Reject` and the driver
issues `invalidateblock` to bitcoind. For most of these conditions the block is
perfectly good apart from one ambiguous or redundant message. This document
sorts every rejection condition into three tiers — **convertible now**,
**convertible with a design change**, and **irreducible** — and gives the test
that decides which is which.

**Result: of 29 rejection conditions, 13 can be converted immediately, 6 more
can be eliminated by two specific design changes, and 9 are irreducible. The
irreducible 9 are all one rule.**

---

## 1. Why rejection is used at all

Two distinct jobs are being done by the same mechanism, and separating them is
the whole point of this analysis.

**Job A — preserving consensus.** Every enforcing node must derive the same
state from the same block. If a block is ambiguous and nodes resolve it
differently, their databases diverge silently and permanently. Rejecting the
block is a blunt but certain way to prevent that.

**Job B — making an attack unprofitable.** Some rules exist because a miner
would otherwise gain by breaking them. Rejection reaches the miner's entire
block revenue, which is the largest lever available.

Job A does **not** require rejection. It requires _determinism_. If the
resolution rule is a pure function of `(block, prior state)`, every node
computes the same answer and consensus holds exactly as well as under rejection.

Job B **does** require rejection, or something equally costly. No amount of
determinism removes an attacker's profit.

Most convertible cases turn out to be Job A only.

---

## 2. The floor: why zero rejections is impossible

It is worth being blunt about this, because it bounds everything below.

`OP_DRIVECHAIN` is `OP_NOP5 <push S> OP_TRUE`. To a node that does not enforce
BIP-300, that script evaluates as: no-op, push one byte, push true — it
**succeeds with an empty scriptSig**. A treasury UTXO is _anyone-can-spend_
under stock Bitcoin rules. This is not incidental; BIP-300 says so directly, in
explaining why the trailing `OP_TRUE` is there:

> The final OP_TRUE is to ensure this change remains a softfork.

That is the soft-fork trick, and it has a consequence that is easy to miss:
**nothing in Bitcoin protects sidechain funds.** The only thing standing between
a treasury UTXO and anyone with a wallet is that enforcing nodes reject the
block that spends it improperly.

So the treasury rejection rules are not _enforcement of_ the peg. They **are**
the peg. Remove them and BIP-300 stops being a soft fork and becomes a
suggestion — the treasury is spendable by anyone, and the sidechain's entire
balance is taken in the first block after the rule is dropped.

Every other rejection in the protocol is negotiable. These are not, and no
resolution rule, however clever, can make them so.

---

## 3. The safety test

A rejection rule `R` may be replaced by a deterministic resolution `D` **only if
all four conditions hold**:

**(1) Determinism.** `D` is a pure function of the block and the prior state.
Not arrival order, not mempool contents, not wall-clock, not peer state. In
practice: tie-breaks must use in-block ordering (output index, transaction
index) or prior-state ordering (proposal height), never anything observed
locally.

**(2) Incentive neutrality.** No party's payoff from triggering `D` exceeds
their payoff from complying.

**(3) Invariant preservation.** `D` cannot leave the state violating a
fund-safety invariant:

- exactly one treasury UTXO per active slot, at all times;
- a treasury's value decreases only via an M6 whose `M6ID` cleared the vote
  threshold;
- a bundle's vote count changes by at most 1 per block.

**(4) Bounded state.** `D` must not require unbounded memory.

A rule failing any of these keeps its rejection — _unless the protocol itself is
changed so the rule is no longer needed_, which is Tier 2.

---

## 4. Tier 1 — convertible now

Thirteen conditions. All satisfy the four conditions above; none requires any
change to the protocol beyond the resolution rule itself. Four (marked ★) are
open questions the LayerTwo-Labs draft raises with `???`.

### 4.1 Duplicate coinbase messages — take the first

**Conditions:** `DuplicateM1`, `DuplicateM2`, `DuplicateM4`

Process the message at the lowest output index; ignore the rest.

_Determinism:_ output index is fixed by the block. ✔ _Incentives:_ the attacker
wants more than one message applied — a second ack, a second vote. Taking the
first caps the effect at exactly what a compliant block achieves. ✔

For `DuplicateM1` the state already works this way: a re-proposal of an existing
`(slot, description_hash)` is ignored so vote counts cannot be reset. Extending
that to same-block duplicates is consistent, not novel.

### 4.2 M3 for an inactive slot — ignore ★

**Condition:** `InactiveSidechain`

The original BIP-300 already says: _"M3 is ignored if it does not parse, or if
it is for a sidechain that doesn't exist."_ The block-invalidity rule
contradicts the same document's own prose. Proposing a bundle for a dead slot
accomplishes nothing under either rule. ✔

### 4.3 M3 re-proposing a pending bundle — ignore

**Condition:** `BundleAlreadyPending`

The attack is resetting a bundle's accumulated votes. Ignoring defeats it as
well as rejecting does, without punishing an honest miner who included a stale
proposal. ✔

### 4.4 M4 vote array longer than the active-slot vector — truncate ★

**Condition:** `InvalidVotes`

Apply votes at indices mapping to active slots; ignore the excess. The
active-slot vector is sorted ascending from prior state, so all nodes compute
the same prefix. The excess entries address slots that do not exist and could
not move any count under either rule. ✔

### 4.5 M4 bundle index out of range — abstain ★

**Condition:** `UpvoteFailed`

Treat that slot's vote as `ABSTAIN`.

The most elegant of the conversions: `ABSTAIN` is already a first-class vote
value with defined semantics, so the malformed vote is mapped onto an existing
transition rather than introducing a new one. Compare the alternative of
snapping to the nearest valid index, which would be arbitrary and exploitable.
The miner wanted to upvote a bundle that does not exist; abstain gives them
nothing, which is the correct payoff. ✔

### 4.6 `VOTES_TWO_BYTE` where one byte suffices — accept ★

**Condition:** `TwoBytesWithinByteRange`

This rule invalidates a block over **encoding efficiency**. A miner loses an
entire block subsidy for wasting roughly one byte per active sidechain in an
OP_RETURN they already paid for. The waste is self-penalising; consensus
enforcement adds nothing. Vote semantics are identical between the two encodings
by definition. ✔

### 4.7 M5 with no address OP_RETURN — accept as an uncredited deposit

**Condition:** `MissingDepositAddress`

Accept the treasury movement, update the CTIP, credit no depositor.

The subtlest of the group, and it turns on **which direction the error runs**.
The sats land in the treasury, so the sidechain's L1 backing increases while its
L2 issuance does not: the result is _over_-collateralisation. No sidechain user
can lose funds; the depositor loses their own money through their own malformed
transaction.

Under-collateralisation would fail condition (3) outright.
Over-collateralisation does not. ✔

Enforcing nodes SHOULD log and expose these so operators can make the depositor
whole out of band.

### 4.8 Zero-value deposit or withdrawal — accept as a CTIP move

**Condition:** `ZeroDiff`

`T_new == T_old` means no value moved. A no-op costing the sender a fee. ✔

### 4.9 M5/M6 ambiguity — classify by value delta

**Condition:** `Ambiguous`

A transaction cannot both increase and decrease the treasury. Classify strictly:
`T_new > T_old` → M5, `T_new < T_old` → M6, equality → §4.8. This is a
clarification rather than a relaxation — once classified, the normal M5 or M6
rules apply unchanged, and a transaction failing the M6 rules still fails them.

### 4.10 Parent-related conditions — defer rather than reject

**Conditions:** `MissingParentHeight`, `BlockParent`

These are not rule violations at all. They mean _"this node does not yet have
the context to validate this block"_ — the parent's height is unknown, or the
block does not build on the current tip.

Invalidating here is actively harmful: a valid block arriving during a reorg, or
before its parent is processed, gets pushed through `invalidateblock` at
bitcoind. The correct behaviour is to **defer** — buffer until the parent
connects, or handle it as normal reorg processing.

This is a bug fix rather than a protocol change, and the only item that improves
liveness rather than merely reducing disruption.

---

## 5. Tier 2 — convertible with a design change

Six conditions. These cannot be fixed by a resolution rule — each fails test (2)
— but each becomes **unnecessary** under a specific change to the protocol.
Unlike Tier 1, these are design proposals and want adversarial review before
anyone implements them.

### 5.1 The BMM family — eliminated by off-chain requests

**Conditions:** `MultipleBmmRequests`, `MultipleBmmBlocks`, `DuplicateM7`,
`NotAcceptedByMiners`, `BmmRequestExpired`

#### Why selection fails

Deterministic selection is trivially implementable here: "take the M8 at the
lowest transaction index", or "take the M8 paying the highest fee", are both
pure functions of the block. Test (1) passes.

Test (2) fails, for a reason that lies outside BIP-301 entirely.

**An M8 is an ordinary Bitcoin transaction. Its fee is paid to the miner by
Bitcoin's own consensus rules, regardless of what the enforcer decides about
it.** The enforcer can decline to _count_ an M8 as a BMM request. It cannot
decline to let the miner keep the money.

|                                  | Compliant block | Block with 10 M8s for one slot |
| -------------------------------- | --------------- | ------------------------------ |
| Side:blocks connected            | 1               | 1                              |
| BMM fees collected               | 1               | **10**                         |
| Enforcer verdict under selection | valid           | valid (9 "ignored")            |

The miner is strictly better off breaking the rule, and nine Simons paid for a
side:block that was never connected. This is the outcome the LayerTwo-Labs draft
names as the reason for the rule:

> If a miner is allowed to accept multiple BMM requests in the same block for
> the same sidechain, then the miner can collect fees for sidechain blocks that
> were not actually connected to the sidechain, which is undesirable.

Rational Simons stop bidding and the BMM market unwinds. The rule is not
protecting against confusion; it is protecting against theft. Rejection works
because it is the only response reaching the miner's _entire_ block revenue —
subsidy, ordinary fees, and harvested BMM fees together.

The same argument covers duplicate M7s (identical harvest, different route) and
stale M8s (a miner hoarding old requests and mining them later, collecting for
side:blocks that can no longer connect).

#### The change that eliminates it

**Move BMM payment off-chain.** Under Lightning-based BMM — gestured at in the
original BIP-301 and specified nowhere — Simon pays Mary conditionally off-chain
and Mary claims by including the M7. A losing bidder pays **nothing on-chain**.

That single change removes the harvest, and with it the reason for all five
rejections:

| Condition                           | Why it exists               | Under off-chain requests     |
| ----------------------------------- | --------------------------- | ---------------------------- |
| `MultipleBmmRequests`               | miner harvests N fees       | no on-chain fee to harvest   |
| `MultipleBmmBlocks` / `DuplicateM7` | same harvest via M7         | same                         |
| `NotAcceptedByMiners`               | M8 fee taken without M7     | payment is conditional on M7 |
| `BmmRequestExpired`                 | hoarded requests mined late | offer expires off-chain      |

Note what happens to the M7 rules specifically: with no on-chain M8, the "one M7
per slot" constraint stops being a theft guard and becomes a plain
well-formedness question — which side:block did the miner endorse? That is Tier
1 territory: **take the M7 at the lowest output index.**

This is the single highest-leverage change available. It converts five
rejections at once and is the only path to zero for BIP-301.

**Status: needs a specification.** The mechanism is sketched in the original BIP
and nowhere formalised. Writing it is a larger piece of work than this document,
and the five rejections must stay until it exists and is deployed.

### 5.2 Mandatory payout — eliminated by sticky approval

**Condition:** the block in which a bundle crosses the vote threshold MUST
include the corresponding M6, or the block is invalid.

#### Why it exists

Without it, miners who approved a bundle can simply decline to include the M6
until the bundle expires at `WITHDRAWAL_BUNDLE_MAX_AGE`, killing an approved
withdrawal through passive omission at zero cost. There is nothing to "select" —
the required transaction is absent, so no resolution rule applies.

#### The change that eliminates it

Make approval **sticky** rather than instantaneous:

1. When a bundle's vote count first exceeds
   `WITHDRAWAL_BUNDLE_INCLUSION_THRESHOLD`, it enters an `APPROVED` state.
2. An `APPROVED` bundle is **exempt from the ordinary expiry clock**. It remains
   payable indefinitely.
3. At most one bundle per slot may be `APPROVED` at a time. While a slot has an
   `APPROVED` bundle, M3 messages for that slot are ignored and ordinary M4
   votes for that slot cast no votes.
4. An `APPROVED` bundle leaves that state only by being paid out, or by
   revocation (below).

Omission then costs miners only time, never the withdrawal. The rejection buys
nothing and can be dropped: a miner who omits the M6 has merely postponed a
payment that is still owed.

#### The escape hatch, and why it is needed

Step 2 as stated introduces a worse failure than the one it fixes: a permanently
`APPROVED` bundle that is never mined would freeze the slot forever — no new
bundles, no withdrawals, no way out.

So revocation must exist, but it must be **explicit and costly** rather than
passive:

5. An `APPROVED` bundle is revoked if it accumulates
   `WITHDRAWAL_BUNDLE_INCLUSION_THRESHOLD` cumulative `ALARM` votes while
   approved. On revocation it is removed and the slot unfreezes.

This preserves what BIP-300 already grants miners — the ability to veto a
withdrawal — while removing the ability to _silently stall_ one. Killing an
approved bundle now requires the same sustained, visible, majority-hashrate
commitment that approving it did. Passive omission achieves nothing.

**Status: design sketch, not a finished rule.** It wants review on at least
these points: whether alarm-based revocation should use a cumulative or
consecutive count; whether freezing M3 for a slot creates a griefing vector for
whoever proposes the approved bundle; and how `REPEAT_PREVIOUS` M4s interact
with a frozen slot.

---

## 6. Tier 3 — the irreducible treasury lock

**Conditions:** `TreasurySpentWithoutNewCtip`, `MultipleOpDrivechainOutputs`,
`OldCtipUnspent`, `M6id` mismatch, and the five `InvalidM6` variants
(`InputCount`, `InsufficientVoteCount`, `MissingPendingWithdrawal`,
`TreasuryOutputCount`, `TreasuryOutputIndex`).

Nine conditions, but really **one rule**:

> A treasury UTXO may only be spent by an M6 whose `M6ID` cleared the vote
> threshold, producing exactly one new treasury UTXO for the same slot.

Every condition above is a facet of that single check. Per §2, this rule is the
peg itself — the treasury is anyone-can-spend to every node that does not
enforce it.

The individual cases, for completeness:

**`TreasurySpentWithoutNewCtip`** — coins leave the treasury and no new treasury
is created. There is no "best non-blocking option"; the funds are gone. The most
important rejection in BIP-300.

**`MultipleOpDrivechainOutputs`, `OldCtipUnspent`** — both produce two live
treasury UTXOs for one slot. One _could_ deterministically nominate the
lowest-indexed output as the CTIP, satisfying test (1) — but the other remains a
spendable `OP_DRIVECHAIN` output for the same slot, permanently breaking _"there
MUST never be two treasury UTXOs for the same sidechain slot"_. Every subsequent
deposit and withdrawal becomes ambiguous about which treasury chain it belongs
to. Determinism is achievable; invariant preservation is not.

**The M6 vote and structure rules** are the authorisation check itself.
`InsufficientVoteCount` is not a formatting complaint — it _is_ the hashrate
approval that constitutes the security model.

**`NoCoinbase`** is retained separately as defensive and unreachable: Bitcoin
rejects a block with no transactions before the enforcer sees it.

---

## 7. Summary

| #   | Condition                             | Today  | Proposed                   | Tier |
| --- | ------------------------------------- | ------ | -------------------------- | ---- |
| 1   | `MissingParentHeight`                 | reject | defer                      | 1    |
| 2   | `BlockParent`                         | reject | defer                      | 1    |
| 3   | `DuplicateM1`                         | reject | first wins                 | 1    |
| 4   | `DuplicateM2`                         | reject | first wins                 | 1    |
| 5   | `DuplicateM4`                         | reject | first wins                 | 1    |
| 6   | `InactiveSidechain` ★                 | reject | ignore                     | 1    |
| 7   | `BundleAlreadyPending`                | reject | ignore                     | 1    |
| 8   | `InvalidVotes` ★                      | reject | truncate                   | 1    |
| 9   | `UpvoteFailed` ★                      | reject | abstain                    | 1    |
| 10  | `TwoBytesWithinByteRange` ★           | reject | accept                     | 1    |
| 11  | `MissingDepositAddress`               | reject | uncredited deposit         | 1    |
| 12  | `ZeroDiff`                            | reject | CTIP move                  | 1    |
| 13  | `Ambiguous`                           | reject | classify by delta          | 1    |
| 14  | `MultipleBmmRequests`                 | reject | off-chain BMM              | 2    |
| 15  | `MultipleBmmBlocks`                   | reject | off-chain BMM              | 2    |
| 16  | `DuplicateM7`                         | reject | off-chain BMM → first wins | 2    |
| 17  | `NotAcceptedByMiners`                 | reject | off-chain BMM              | 2    |
| 18  | `BmmRequestExpired`                   | reject | off-chain BMM              | 2    |
| 19  | Mandatory payout                      | reject | sticky approval            | 2    |
| 20  | `TreasurySpentWithoutNewCtip`         | reject | **keep**                   | 3    |
| 21  | `MultipleOpDrivechainOutputs`         | reject | **keep**                   | 3    |
| 22  | `OldCtipUnspent`                      | reject | **keep**                   | 3    |
| 23  | `M6id` mismatch                       | reject | **keep**                   | 3    |
| 24  | `InvalidM6::InputCount`               | reject | **keep**                   | 3    |
| 25  | `InvalidM6::InsufficientVoteCount`    | reject | **keep**                   | 3    |
| 26  | `InvalidM6::MissingPendingWithdrawal` | reject | **keep**                   | 3    |
| 27  | `InvalidM6::TreasuryOutputCount`      | reject | **keep**                   | 3    |
| 28  | `InvalidM6::TreasuryOutputIndex`      | reject | **keep**                   | 3    |
| 29  | `NoCoinbase`                          | reject | keep (unreachable)         | —    |

```
29 conditions today
 -13  Tier 1, resolution rules only            → 16
 - 5  Tier 2, off-chain BMM specification      → 11
 - 1  Tier 2, sticky approval                  → 10
 = 9 treasury conditions + NoCoinbase
```

**The floor is the treasury lock: one rule, nine facets.**

---

## 8. Deployment: this is not a soft fork

**The most important caveat in this document.**

Every change above **relaxes** validity. A node running the new rules accepts
blocks a node running the old rules rejects. Between two enforcer versions that
is not a soft fork — it is a **chain split**, because under BIP-300 the enforcer
drives bitcoind's `invalidateblock`. Old-version nodes follow one chain and
new-version nodes another, on any block exercising a converted rule.

(Relative to Bitcoin itself nothing changes: non-enforcing nodes already accept
all these blocks. The split is confined to the enforcer network — precisely the
set of nodes whose agreement BIP-300 depends on.)

Two viable paths:

1. **Ship before mainnet activation.** BIP-300 is not activated. Until it is,
   changing these rules costs nothing but a coordinated release. Overwhelmingly
   the cheaper path, and the reason to settle this now.

2. **Flag-day activation.** After activation, changes must land at an agreed
   height with every enforcing node upgraded beforehand. Standard, but expensive
   and slow.

What must **not** happen is incremental rollout. A node converting these rules
unilaterally forks itself off the enforcer network the first time a miner emits
a duplicate M2.

Tier 2 carries an additional ordering constraint: the five BMM rejections cannot
be dropped until off-chain BMM is specified, implemented, and in use. Dropping
them earlier re-opens the fee harvest described in §5.1.

---

## 9. Open questions for reviewers

1. **Is the harvest analysis in §5.1 complete?** It assumes the miner captures
   M8 fees through ordinary block-fee collection. If some on-chain structure
   could make a losing bidder pay nothing, the BMM cases would become Tier 1
   without needing Lightning. No such structure is obvious under current script
   rules, but the question is worth putting to reviewers who know covenant
   proposals well.

2. **Does sticky approval (§5.2) create a griefing vector?** Freezing M3 for a
   slot while a bundle is approved means whoever gets a bundle approved can
   block all other withdrawal proposals for that slot until it is paid or
   revoked. Is the revocation threshold low enough to bound that?

3. **Should `MissingDepositAddress` credit a canonical address instead?** §4.7
   credits nobody. An alternative is a per-sidechain "unattributed deposits"
   address, which is friendlier to depositors but adds a field to the sidechain
   description that this proposal otherwise treats as opaque.

4. **Does deferring on `BlockParent` (§4.10) race with the driver's reorg
   handling?** The driver already handles disconnects; deferral must not
   conflict. Needs review against
   `cusf-enforcer-mempool/lib/mempool/sync/task.rs`.

5. **Is `NoCoinbase` truly unreachable?** Worth confirming there is no path —
   e.g. a malformed block delivered over ZMQ — by which the enforcer could
   observe a block with no transactions.
