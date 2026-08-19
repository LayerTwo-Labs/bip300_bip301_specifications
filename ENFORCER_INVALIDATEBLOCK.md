# When does the enforcer `invalidateblock`?

A reference for `bip300301_enforcer` — every condition under which a mainchain
block gets invalidated in bitcoind as a result of BIP300/BIP301 enforcement.

---

## TL;DR

**The enforcer never calls `invalidateblock` itself.** Grep the
`bip300301_enforcer` tree for the RPC and you will find only comments and
test-harness setup calls.

The real chain is:

```
bip300301_enforcer                      cusf-enforcer-mempool                bitcoind
──────────────────                      ─────────────────────                ────────
Validator::connect_block()
  └─ returns ConnectBlockAction::Reject ──► match on Reject
                                             └─ main_client
                                                  .invalidate_block(hash) ──► invalidateblock RPC
```

So "when does the enforcer invalidate a block?" is really **"when does
`connect_block` return `Reject`?"** — and that has exactly one call site.

---

## The two halves

### 1. The enforcer decides (`bip300301_enforcer`)

| What                                           | Where                                                      |
| ---------------------------------------------- | ---------------------------------------------------------- |
| The single `Reject` return                     | `lib/validator/cusf_enforcer.rs:306`                       |
| Logs `"rejecting block: {reason}"` just before | `lib/validator/cusf_enforcer.rs:304`                       |
| Reject reasons                                 | `lib/validator/cusf_enforcer.rs:160` (`enum RejectReason`) |
| Validation errors that feed it                 | `lib/validator/task/error.rs:343` (`enum ConnectBlock`)    |

Note that the header write is **committed** even on reject
(`header_rwtxn.commit()`), while the block-connect child txn is aborted. The
enforcer remembers it saw the header; it just refuses to connect the block.

### 2. The driver acts (`cusf-enforcer-mempool`)

Declared in `bip300301_enforcer/Cargo.toml:95` as a git dependency on
`https://github.com/LayerTwo-Labs/cusf-enforcer-mempool.git`.

| Path                                                               | Where                         |
| ------------------------------------------------------------------ | ----------------------------- |
| ZMQ-driven task: `Reject` → `invalidate_block`                     | `lib/cusf_enforcer.rs:318`    |
| Mempool-sync path: `RequestItem::RejectBlock` → `invalidate_block` | `lib/mempool/sync/mod.rs:498` |

The second is the batched request pipeline; `lib/mempool/sync/mod.rs:70` maps
that request variant to the literal method name `"invalidateblock"`.

---

## The fatality model

This is the part that decides everything, and it is easy to miss.

Error enums derive `Fatality`/`Split`, and each variant carries a marker:

| Marker              | Meaning                           | Block invalidated?        |
| ------------------- | --------------------------------- | ------------------------- |
| `#[fatal(false)]`   | "Jfyi" — the block is bad         | **Yes** → `Reject`        |
| `#[fatal(true)]`    | Infrastructure failure (DB, disk) | **No** → error propagates |
| `#[fatal(forward)]` | Inherits from the wrapped error   | Depends                   |

The distinction is deliberate: a corrupted database must never be allowed to
invalidate honest blocks. Only errors that mean _"this block violates the
rules"_ reach `RejectReason`.

---

## The complete case list

### Structural (`RejectReason`, `lib/validator/cusf_enforcer.rs:160`)

| Case                  | Condition                                        | Site                   |
| --------------------- | ------------------------------------------------ | ---------------------- |
| `MissingParentHeight` | Parent block's height is unknown to the enforcer | `cusf_enforcer.rs:212` |
| `ConnectBlock(jfyi)`  | Any non-fatal validation error below             | `cusf_enforcer.rs:256` |

### Block-level (`enum ConnectBlock`, `lib/validator/task/error.rs:343`)

| Case                  | Message                                                                       |
| --------------------- | ----------------------------------------------------------------------------- |
| `BlockParent`         | ``Block parent `{parent}` does not match tip `{tip}` at height {tip_height}`` |
| `NoCoinbase`          | `Block has no transactions (missing coinbase)`                                |
| `MultipleBmmBlocks`   | `Multiple blocks BMM'd in sidechain slot {n}`                                 |
| `MultipleBmmRequests` | `Multiple BMM requests accepted in sidechain slot {n}`                        |
| `CoinbaseMessages`    | see below                                                                     |

### Duplicate coinbase messages (`lib/messages.rs:420`)

All four are non-fatal, all four reject:

- `DuplicateM1` — M1 sidechain proposal for a slot already included at an
  earlier index
- `DuplicateM2` — M2 acking a proposal for a slot already included
- `DuplicateM4` — M4 already included
- `DuplicateM7` — M7 for a slot already included

### M3 — propose bundle (`error.rs:68`)

- `BundleAlreadyPending` — BIP300: an M6ID already proposed and not yet paid out
  must not be re-proposed; re-proposing would reset its ack count
- `InactiveSidechain`

### M4 — ack bundles (`error.rs:134`)

- `TwoBytesWithinByteRange` — `M4 TwoBytes encoding with no element > 253` (the
  encoding wastes a byte per element, so it MUST be rejected)
- via `HandleM4Votes` (`error.rs:105`): `InvalidVotes` (expected N, found M),
  `UpvoteFailed`

### M5 / M6 — deposit and withdrawal (`error.rs:215`)

| Case                          | Meaning                                                 |
| ----------------------------- | ------------------------------------------------------- |
| `Ambiguous`                   | Cannot be both an M5 deposit and an M6 withdrawal       |
| `MissingDepositAddress`       | Treasury output not followed by the address `OP_RETURN` |
| `MultipleOpDrivechainOutputs` | More than one `OP_DRIVECHAIN` output for a sidechain    |
| `OldCtipUnspent`              | Old Ctip for the sidechain is still unspent             |
| `TreasurySpentWithoutNewCtip` | Treasury spent without creating a new Ctip              |
| `ZeroDiff`                    | Cannot deposit or withdraw zero sats                    |
| `M6id`                        | M6ID mismatch                                           |
| `InvalidM6`                   | see below                                               |

`InvalidM6` (`error.rs:174`) splits further:

- `InputCount` — M6 withdrawals must have exactly 1 input
- `InsufficientVoteCount` — vote count below threshold for the bundle
- `MissingPendingWithdrawal` — M6ID doesn't correspond to a pending withdrawal
- `TreasuryOutputCount` — must create exactly 1 ctip
- `TreasuryOutputIndex` — treasury output must be at index 0

### M8 — BMM request (`error.rs:279`)

- `BmmRequestExpired`
- `NotAcceptedByMiners` — cannot include a BMM request the miners did not accept

---

## What does **not** invalidate

These are the traps. Each looks like it should reject, and doesn't.

**Fatal-only error paths.** `HandleM1ProposeSidechain`, `HandleM2AckSidechain`,
`HandleFailedM6Ids` and `HandleFailedSidechainProposals` contain _only_
`#[fatal(true)]` DB variants. An M1 or M2 can never on its own invalidate a
block — only the duplicate-detection in `CoinbaseMessagesError` can.

**Batch sync.** In `lib/validator/task/mod.rs:1789`, a non-fatal error during
initial block-batch sync returns `Ok(Some(block_hash))` instead of rejecting.
The comment is explicit:

> We should not call out to `invalidateblock` in case of failures here, as that
> is handled by the cusf-enforcer-mempool crate.

**Mempool tx rejection.** `accept_tx` (`lib/validator/cusf_enforcer.rs:445`):

> A fatal error here isn't something that means we should call out to the
> `invalidateblock` RPC. It simply means the transaction will not be accepted
> into the mempool.

**Wallet sync on reject.** `lib/wallet/cusf_block_producer.rs:287` deliberately
skips wallet sync when the validator rejects, because the aborted child txn
means `get_block_infos` would fail and bubble an error that _prevents_ the
standalone driver from issuing `invalidateblock`.

---

## Test-harness calls are not enforcer behavior

Several `invalidateblock` hits in the tree are tests _driving_ bitcoind to set
up a reorg — not the enforcer reacting:

- `bip300301_enforcer/integration_tests/test_zmq_sequence_gap.rs:90`
- `bip300301_enforcer/integration_tests/test_wallet_reorg_multi_block.rs:146`
- `cusf-enforcer-mempool/integration_tests/test_disconnect_through_sync_tip.rs:51`
- `cusf-enforcer-mempool/integration_tests/test_double_insert_after_reorg.rs:40`
- `cusf-enforcer-mempool/integration_tests/test_enforcer_rejection_during_reorg.rs:49`
- `cusf-enforcer-mempool/integration_tests/test_reorg_re_inserts_tx.rs:36`

---

## Test coverage

`bip300301_enforcer/integration_tests/block_verdict.rs` defines the harness:
`Expect::Rejected { log_contains }` asserts the block is invalidated (bitcoind's
`getblock` reports `confirmations == -1`) _and_ that the log carries the reason.

Cases actually exercised end-to-end in `test_invalid_block.rs`:

| Case               | Expected log                                       |
| ------------------ | -------------------------------------------------- |
| Duplicate M1       | `rejecting block: M1 sidechain proposal for slot`  |
| Duplicate M2       | `rejecting block: M2 that acks proposal for slot`  |
| Duplicate M4       | `rejecting block: M4 already included at index`    |
| Duplicate M7       | `rejecting block: M7 for slot`                     |
| Duplicate M8       | `Multiple BMM requests accepted in sidechain slot` |
| M5 missing address | `has no address OP_RETURN output`                  |

On the driver side,
`cusf-enforcer-mempool/integration_tests/test_rejected_block_disconnect.rs`
asserts the full round trip: a rejected block is invalidated in bitcoind, which
then emits the disconnect.

Note the duplicate-M7 and duplicate-M8 pair. Both enforce BIP301's _"only one M8
per mainchain block per sidechain slot"_, but they catch different halves — M7
is the coinbase commitment, M8 the transaction. The M8 case cannot be caught the
same way: both requests are individually valid, name the current tip, and carry
the same h*, so each one legitimately corresponds to the single M7 in the
coinbase.

---

## Source map

```
bip300301_enforcer/
  lib/validator/cusf_enforcer.rs      :160  enum RejectReason
                                      :212  MissingParentHeight
                                      :256  RejectReason::ConnectBlock
                                      :306  ConnectBlockAction::Reject   ← the decision
                                      :445  why accept_tx does not invalidate
  lib/validator/task/error.rs         :343  enum ConnectBlock (fatality markers)
                                      :279  HandleM8
                                      :215  HandleM5M6
                                      :174  InvalidM6
                                      :134  HandleM4AckBundles
                                      :105  HandleM4Votes
                                      :68   HandleM3ProposeBundle
  lib/validator/task/mod.rs           :1789 batch sync does NOT invalidate
  lib/messages.rs                     :420  CoinbaseMessagesError (duplicate M1/M2/M4/M7)
  lib/wallet/cusf_block_producer.rs   :287  skip wallet sync on reject
  Cargo.toml                          :95   cusf-enforcer-mempool git dependency

cusf-enforcer-mempool/
  lib/cusf_enforcer.rs                :318  invalidate_block on Reject   ← the action
  lib/mempool/sync/mod.rs             :498  invalidate_block (batched sync path)
                                      :70   RejectBlock → "invalidateblock"
```
