# Open questions for the authors

Everything in this repository is settled except the items below. This is the
agenda: each one is a decision only the authors can make, and each is blocking
something.

The order is not arbitrary — see [Why this order](#why-this-order). Work down
it.

| #   | Question                                  | Where               | Blocks                              | Answerable         |
| --- | ----------------------------------------- | ------------------- | ----------------------------------- | ------------------ |
| 1   | Treasury UTXO when a slot is overwritten  | BIP-300 A12         | Fund safety; possibly a new rule    | Now, by discussion |
| 2   | Which NOP opcode `OP_DRIVECHAIN` uses     | BIP-300 A13         | Every treasury `scriptPubKey`       | Now, one word      |
| 3   | M2 header byte — `BF` or `DF`             | BIP-300 A1          | Byte-level correctness; publication | Now, one word      |
| 4   | One M1 / M3 per block                     | BIP-300 A9          | Enforcer change; extended B1 + B10  | Now, by discussion |
| 5   | The 90% unused-slot threshold             | BIP-300 A3          | Nothing — confirm or correct        | Now, by discussion |
| 6   | What the M8 rework must preserve          | BIP-301 A7 (A3, A5) | The BIP-301 M8 section              | Now, partially     |
| 7   | Five questions on the relaxation analysis | NON-INVALIDATING §9 | Only the extended variant           | Later              |

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
