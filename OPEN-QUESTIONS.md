# Open questions for the authors

Everything in this repository is settled except the items below. This is the
agenda: each one is a decision only the authors can make, and each is blocking
something.

The order is not arbitrary — see [Why this order](#why-this-order). Work down
it.

Items 1–7 came from reconciling the three documents against each other. Items
8–12 came from something else: **an independent implementation of BIP-300/301,
written from these specifications, in Bitcoin Core.** Building it surfaced five
more places where the specification and the reference implementation say
different things, and one of them — item 8 — decides what the peg actually
guarantees.

Those five are not opinions about the Rust. The implementation is checked
against the reference implementation by 46 generated vectors covering the
message formats and `M6ID`, and agrees with it on all of them; where this
document says "the implementation does X", that is a behaviour two
implementations now agree on. A further list of **corrections needing no
decision** follows the questions.

| #   | Question                                       | Where               | Blocks                              | Answerable         |
| --- | ---------------------------------------------- | ------------------- | ----------------------------------- | ------------------ |
| 1   | Treasury UTXO when a slot is overwritten       | BIP-300 A12         | Fund safety; possibly a new rule    | Now, by discussion |
| 2   | Which NOP opcode `OP_DRIVECHAIN` uses          | BIP-300 A13         | Every treasury `scriptPubKey`       | Now, one word      |
| 3   | M2 header byte — `BF` or `DF`                  | BIP-300 A1          | Byte-level correctness; publication | Now, one word      |
| 4   | One M1 / M3 per block                          | BIP-300 A9          | Enforcer change; extended B1 + B10  | Now, by discussion |
| 5   | The 90% unused-slot threshold                  | BIP-300 A3          | Nothing — confirm or correct        | Now, by discussion |
| 6   | What the M8 rework must preserve               | BIP-301 A7 (A3, A5) | The BIP-301 M8 section              | Now, partially     |
| 7   | Five questions on the relaxation analysis      | NON-INVALIDATING §9 | Only the extended variant           | Later              |
| 8   | Is Mandatory payout a real rule?               | BIP-300 M4          | What an approved withdrawal means   | Now, by discussion |
| 9   | Must an M4 vote array match exactly?           | BIP-300 M4          | Block validity; enforcer or spec    | Now, one word      |
| 10  | Do proposals fail early when they cannot win?  | BIP-300 M2          | When an ack is ignored              | Now, by discussion |
| 11  | Must a blinded M6 pay out something?           | BIP-300 M6          | Enforcer change, or drop the rule   | Now, one word      |
| 12  | Does an M7 for an inactive slot mean anything? | BIP-301             | Nothing much — confirm or correct   | Now, one word      |

**Item 8 belongs beside item 1 by this document's own ordering rule** — it is
the second question here whose answer changes what happens to money. It sits at
8 only to keep the numbering of a list that is already under review stable.

---

## 1. What happens to a treasury UTXO when its slot is overwritten?

**BIP-300 Appendix A, item 12. UNRESOLVED.**

`bip300.md:773` asks this and nothing answers it — not the original draft, not
the LayerTwo-Labs draft, not the unified documents, not the companion proposal.

Read literally, the rules give only one answer. A treasury UTXO is identified
solely by the slot number in its `scriptPubKey`; a slot has at most one; an M1
overwrite changes the slot's description without touching the UTXO set. So the
same chain of treasury UTXOs continues and **the incoming sidechain inherits the
outgoing sidechain's balance** — which the outgoing sidechain's users can no
longer withdraw, because their pending bundles name a sidechain the slot no
longer describes.

Note the bar: overwriting a _used_ slot needs
`USED_SIDECHAIN_SLOT_ACTIVATION_THRESHOLD`, a bare majority sustained over 26300
blocks. The 90% bar of item 5 below guards _empty_ slots only.

**The ask:** is inheritance intended? If not, the spec needs a rule for what
happens to the funds — and that rule is new text in M1 and M5, plus enforcer
work. This is first on the list because it is the only open item that can change
where money goes.

## 2. Which NOP does `OP_DRIVECHAIN` use?

**BIP-300 Appendix A, item 13. OPEN DECISION.**

All three sources use `OP_NOP5` (`0xB4`). The proposal on the table is to move
to `OP_NOP8` (`0xB7`) or `OP_NOP9` (`0xB8`), because the choice is arbitrary and
`OP_NOP4`–`OP_NOP7` are the numbers a future Bitcoin Core soft fork is most
likely to reach for first.

Cost of moving today: near zero — nothing is deployed on mainnet, so no treasury
`scriptPubKey` exists that would have to change. Cost of moving later: every
treasury UTXO, the enforcer, and every sidechain implementation.

**The ask:** pick one number. Two candidates have been named and only one can be
used. This is second because it is the cheapest decision on the list and the
most expensive to defer.

## 3. Is the M2 header byte `BF` or `DF`?

**BIP-300 Appendix A, item 1. UNRESOLVED.**

| Source                   | Value         |
| ------------------------ | ------------- |
| Original BIP-300         | `D6 E1 C5 BF` |
| LayerTwo-Labs draft      | `D6 E1 C5 BF` |
| Reference implementation | `D6 E1 C5 DF` |

The implementation disagrees with both specifications in the final byte. The
unified documents follow the implementation, since that is what interoperating
software must match today, but one side has a bug and the documents cannot tell
which. If the specs are right, the enforcer has a wire-format bug. If the
enforcer is right, both specs do.

**The ask:** a one-word answer from whoever wrote the parser or the original
tag. Nothing byte-exact can be published until it is given.

## 4. Restore the one-proposal-per-block limit?

**BIP-300 Appendix A, item 9. OPEN DECISION.**

The original draft allowed one M1 per block, and one M3 per slot per block. The
enforcer rejects only exact duplicates — same `(S, proposal_id)` for M1, same
`(S, M6ID)` for M3 — so differing content in the same slot is permitted, and the
rate limit is gone.

The narrowing was not a deliberate policy change. The original rule was a rate
limit with two purposes: stopping a miner from filling a coinbase with proposals
to grief the slot table or crowd out someone else's, and keeping the demand on
miner attention small, which was part of what made BIP-300 acceptable to miners
at all. Neither purpose is served by the current rule.

Both purposes have since been restated by the authors as deliberate, and the
rapid activation of seven sidechains during testing was possible only because
the limit is absent.

**The ask:** restore it or close it. Restoring means **the enforcer changes, not
the documents** — this is the one item where the specification would lead the
implementation rather than describe it. It also withdraws items 1 and 10 of the
extended variant's Appendix B, so it must be settled before that variant is
reviewed.

## 5. Is the 90% unused-slot threshold intended?

**BIP-300 Appendix A, item 3. Not formally open — flagged for confirmation.**

The original draft set 1008 fails out of 2016 blocks: a **50%** bar for claiming
an empty slot. The LayerTwo-Labs draft and the implementation set 1815 out of
2016: a **90%** bar. The unified documents follow the implementation.

This is a substantive policy difference, not an editorial one — it moves
claiming an empty slot from a bare majority of hashrate to a supermajority.

**The ask:** confirm it was deliberate, or correct it. Lower stakes than the
items above, but it should not go to publication unexamined.

## 6. What must the M8 rework preserve?

**BIP-301 Appendix A, item 7, with items 3 and 5. OPEN.**

BMM bidding and accepting are a work in progress. The M8 encoding in both prior
drafts — and still in the deployed enforcer — descends from a superseded design
in which a BMM request was a distinct transaction type carrying a critical-data
field, not an ordinary transaction with an `OP_RETURN`.

This one cannot be closed by discussion; it closes when the new mechanism
exists. What **can** be settled now is the contract the rework has to hit:

| Must survive any redesign                                                      | May change freely                                       |
| ------------------------------------------------------------------------------ | ------------------------------------------------------- |
| An M8 is valid only with a matching M7 in the same block (same `S`, `H`)       | Whether the request is in an output, and at what index  |
| At most one M8 per slot per mainchain block                                    | Tag bytes, their length, the push encoding              |
| An M8 binds to exactly one parent block, so requests cannot be hoarded         | Whether binding is a `P` field, an `nLockTime`, or else |
| Nothing else in the transaction is interpreted; payment stays out of consensus | Whether the request is on-chain at all                  |

**The ask:** confirm the left column is complete and correct. It is the whole of
BIP-301's consensus content, and it is what a reviewer of the new mechanism
should check against.

**A loose end while you're there:** the wallet sets an `nLockTime` on M8
transactions and the RPC takes a `height`, but no validation rule reads either.
Both are residue of the old design, where the transaction carried the request's
expiry; `P` does that now. If the rework keeps `nLockTime`, it should be
specified; if not, it should come out of the wallet. Right now it is an
undocumented field that looks meaningful and is not.

## 7. The relaxation analysis

**`NON-INVALIDATING-RESOLUTION.md` §9. Five questions.**

These are a different conversation: they ask whether the _proposed relaxations_
are sound, not what the protocol is. They only matter if the extended variant is
pursued, and item 4 above should be settled first, since it withdraws two of the
resolutions they discuss.

Left where they are rather than merged into this list, because the audience is
different — a reviewer who knows covenant proposals and reorg handling, not
necessarily an original author.

---

# Found by implementing it

The five below were not visible from the documents. They surfaced while writing
a second implementation from these specifications and comparing it, rule by
rule, against the reference implementation.

## 8. Is "Mandatory payout" a real rule?

**BIP-300, M4, "Mandatory payout". The largest gap between the specification and
the implementation.**

The specification says that when a bundle's vote count exceeds
`WITHDRAWAL_BUNDLE_INCLUSION_THRESHOLD`, the block in which that happens MUST
include the corresponding M6, and that a block which does not is **invalid**.

**The enforcer has no such check.** The threshold appears in validation in
exactly one place — `handle_m6` (`lib/validator/task/mod.rs:663`), where it
gates whether an M6 that _has been submitted_ is acceptable — and once more in
`lib/block_producer/coinbase.rs:388`, where the block producer decides whether
to _build_ one. Nothing anywhere rejects a block for omitting a payout that has
been approved.

The difference is what the peg guarantees. Under the specification, crossing the
threshold compels payment in that very block. Under the implementation, crossing
it makes the bundle _payable_ — and miners who approve a withdrawal and then
never include it strand it until it expires at 26300 blocks, at which point it
pays nothing and must be proposed again from zero. Both are defensible designs.
They are not the same promise to a sidechain's users.

Note also that the specification's version is demanding: the block that crosses
the threshold is the block carrying the M4 that crosses it, so a miner voting a
bundle over the line must have the M6 constructed and included in the same
block. That is buildable — the block producer already assembles M6s — but it is
a real obligation on whoever mines that block, and it should be intended rather
than inherited.

**The ask:** is the Mandatory payout section a rule or aspiration? If a rule,
the enforcer needs the check and the specification stands as written. If not,
the section should be struck and the guarantee restated: an approved withdrawal
is payable, not compelled.

## 9. Must an M4 vote array match the active-slot count exactly?

**BIP-300, M4, Validation.**

The specification invalidates a block whose vote array `A` has **more** elements
than the active-slot vector `ASN`. The implementation requires the two to be
**equal** — `handle_m4_votes` (`lib/validator/task/mod.rs:379`) rejects on
`upvotes.len() != active_sidechains.len()`, so a _short_ array is invalid too.

A miner who votes on fewer slots than are active has a block that the
implementation rejects and the specification accepts. That is not hypothetical
once slots are being claimed: the active set grows, and an array built against a
stale view of it is short rather than long.

**The ask:** which is right? The implementation's rule is the more defensible —
a short array leaves the trailing slots with no defined vote, where the
specification's own abstain sentinel exists precisely to say "no vote" — but the
specification says something else and one of them has to move.

## 10. Do proposals fail early when they can no longer win?

**BIP-300, M2, Activation.**

The specification discards a proposal when its age exceeds the relevant
`MAX_AGE`. The implementation also discards it once it has missed enough blocks
that it can no longer reach the threshold inside the window that remains
(`handle_failed_sidechain_proposals`, `lib/validator/task/mod.rs:324`):

```
max_fails = max_age - threshold
fails     = age - vote_count
failed    = age > max_age || (age > max_fails && fails >= max_fails)
```

For an empty mainnet slot that is 201 missed blocks out of 2016. A proposal with
100 acks at block 301 is already gone under the implementation and still alive
under the specification for another 1715 blocks.

This is consensus-visible, not a tidiness optimisation. The proposal leaves D1
earlier, so a later M2 naming it is ignored rather than counted, and the same
`(S, proposal_id)` becomes proposable afresh sooner.

**The ask:** intended? If yes it is new specification text. If no it is an
enforcer change.

## 11. Must a blinded M6 pay out something?

**BIP-300, M6, "M6ID and the blinded form".**

The specification says the blinded form MUST have a non-zero total payout.
`compute_m6id` (`lib/messages.rs`) never looks at `P_total`, and nothing
downstream does either.

So a withdrawal that pays out nothing and hands the entire difference to the
miner as fee is accepted, provided the bundle was approved. It is not obviously
exploitable — the bundle still has to be voted through — but it is a stated MUST
that no implementation enforces, and it burns sidechain funds to a miner rather
than paying anyone on L1.

**The ask:** keep the rule and enforce it, or drop it from the specification?

## 12. Does an M7 for an inactive slot mean anything?

**BIP-301, "Interaction with BIP-300".**

The specification says messages naming an inactive slot have no BMM meaning. The
implementation records an M7's commitment without checking whether the slot
holds a sidechain (`lib/validator/task/mod.rs:953`), so an M8 naming an inactive
slot is accepted as long as the matching M7 is in the coinbase.

Nothing much turns on it — blind merge mining a sidechain that does not exist
buys nobody anything — but it is the difference between a sentence about
semantics and a validity rule, and an implementer has to know which it is.

**The ask:** is "no BMM meaning" descriptive, or a rule that M7 and M8 for an
inactive slot are ignored?

---

# Corrections needing no decision

These are places where the documents are simply wrong or silent, and the
implementation is right. They are listed for a reviewer's confidence rather than
for a decision.

**Treasury outputs on inactive slots are ordinary outputs.** The specification
defines a treasury UTXO purely by its `scriptPubKey`. The implementation
additionally requires the slot to hold an active sidechain and skips the output
otherwise (`lib/validator/task/mod.rs:715`). The guard is load-bearing: without
it a zero-value `OP_DRIVECHAIN` output naming a slot with no sidechain reads as
a treasury moving to zero, and a perfectly good block is rejected. The
specification needs the sentence.

**The message byte counts are stated for the wrong encoding.** BIP-300 gives M2
and M3 as "exactly 38 bytes" and M4 versions 0 and 3 as "exactly 6 bytes",
counting the tag as raw script bytes. Every message is in fact
`OP_RETURN <push>` with the tag at the start of the pushed data
(`CoinbaseMessage::parse`, `lib/messages.rs:307`), so those scripts are 39 and 7
bytes. BIP-301 already describes the encoding correctly for M7; BIP-300 does not
for the rest.

**Per-network parameters are undocumented.** The specification says non-mainnet
networks MAY substitute shorter values. The implementation carries three named
sets — mainnet, a tiny one for regtest, and a middle one for dry-run networks
(`lib/types.rs`) — and an activation height below which blocks are recorded but
never scanned for messages or deposits. Both are consensus-relevant per network
and neither appears in the documents.

**The duplicate-M3 rule is enforced indirectly.** The specification invalidates
a block whose coinbase carries two M3s with the same `(S, M6ID)`. The
implementation has no such check when reading coinbase messages —
`CoinbaseMessages::push` carries `// TODO: ensure that M3 pushes are valid`
(`lib/messages.rs:496`) — but the outcome is the same, because the first M3
makes the bundle pending and the second then breaks the already-pending rule.
The independent implementation reproduces this and has a test asserting the
equivalence. No change is needed; the TODO should not be read as a gap.

---

## Why this order

Three rules produced it.

**Money before mechanism.** Item 1 is the only question whose answer can move
funds between parties. Everything else is a wire format or a policy knob. It
goes first even though it is the least likely to have a quick answer.

**Cheap-now, expensive-later before cheap-always.** Items 2 and 3 are each a
one-word answer, but their cost curve is steep: an opcode collision or a wrong
tag byte is trivial to fix today and very expensive after deployment. They sit
above items that will cost the same to decide next month as they do now.

**Decisions before reviews.** Item 4 changes what the extended variant says, so
it precedes item 7, which reviews it. Item 6 is last of the substantive items
because it tracks an implementation rather than a decision — it cannot be closed
in a meeting, only scoped in one.

Item 5 sits low not because it is unimportant but because nothing depends on it:
whichever way it goes, no other item's answer changes.

**Items 8–12 are numbered after 7 and ranked before it.** By the rules above,
item 8 belongs beside item 1: it is a money question, and the only other one on
the list. Items 9 and 11 are one-word answers whose cost curve is the same as
items 2 and 3 — cheap now, expensive after anything is deployed against them.
Item 10 needs discussion; item 12 needs a sentence. They keep their numbers only
so that a list already under review does not renumber underneath its reviewers.
