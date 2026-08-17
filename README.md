# BIP-300 / BIP-301 specifications

Working specifications for **Hashrate Escrows** (BIP-300) and **Blind Merged
Mining** (BIP-301), reconciled against the reference implementation
[`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer).

Three sources exist for these protocols and they do not agree with each other:
the original BIP drafts, the LayerTwo-Labs protocol specifications in this
repository, and the enforcer's actual behaviour. The unified documents below
merge all three, resolve each disagreement in favour of the implementation — or
mark it open, where the resolution is an author's call rather than an editor's —
and record the evidence so the merge can be audited rather than trusted.

## What to read

Start with the **baseline** documents. They describe the protocol as it is
enforced today and propose no behavioural change.

| Document                                                 | Contents                                                             |
| -------------------------------------------------------- | -------------------------------------------------------------------- |
| [`bip300-unified.mediawiki`](./bip300-unified.mediawiki) | BIP-300, unified. Appendix A records 13 items, 4 of them still open. |
| [`bip301-unified.mediawiki`](./bip301-unified.mediawiki) | BIP-301, unified. Appendix A records 7 items, 2 of them still open.  |

The **extended** documents are the same reconciliation plus an Appendix B that
replaces block rejections with deterministic resolution. They **do** change
behaviour and should be read as a delta against the baseline.

| Document                                                                   | Contents                                                                                                   |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| [`bip300-unified-extended.mediawiki`](./bip300-unified-extended.mediawiki) | Baseline + Appendix B: 11 conditions resolved deterministically instead of rejecting the block.            |
| [`bip301-unified-extended.mediawiki`](./bip301-unified-extended.mediawiki) | Baseline + Appendix B: why BIP-301's rejections are _not_ convertible, and how off-chain BMM removes them. |

Every change in an extended Appendix B **relaxes** validity: a node running
those rules accepts blocks a node running the baseline rules rejects. Between
enforcer versions that is a chain-splitting change, not a soft fork, and it
requires coordinated activation.

## Analysis

| Document                                                             | Contents                                                                                                                                                              |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`ENFORCER_INVALIDATEBLOCK.md`](./ENFORCER_INVALIDATEBLOCK.md)       | Every condition under which the enforcer causes a mainchain block to be invalidated, and the call chain that gets it there.                                           |
| [`NON-INVALIDATING-RESOLUTION.md`](./NON-INVALIDATING-RESOLUTION.md) | Sorts all 29 rejection conditions into convertible now (13), convertible with a design change (6), and irreducible (9). Source of the extended documents' Appendix B. |
| [`OPEN-QUESTIONS.md`](./OPEN-QUESTIONS.md)                           | Every unsettled item across both BIPs, in the order they should be decided, with what each one blocks.                                                                |

## Prior specifications

[`bip300.md`](./bip300.md) and [`bip301.md`](./bip301.md) are the LayerTwo-Labs
specifications that predate the unification. They are kept for reference and as
one of the three inputs to it; the unified documents supersede them.

## Open questions

**[`OPEN-QUESTIONS.md`](./OPEN-QUESTIONS.md) is the agenda** — every unsettled
item in dependency order, with what each one blocks and what is being asked.
Start there rather than with the list below, which is a summary of the same
items.

Four items in BIP-300's Appendix A and two in BIP-301's are not settled. The
unified documents state the implementation's behaviour in the body and record
the question in the appendix; none of them is resolved by fiat.

- **The M2 header byte is unresolved.** Both the original BIP-300 and
  `bip300.md` give `D6 E1 C5 BF`; the implementation uses `D6 E1 C5 DF`. Nothing
  in any source reveals which is correct. The unified documents follow the
  implementation and mark this UNRESOLVED — see Appendix A, item 1. It needs
  author input.
- **The treasury UTXO of an overwritten sidechain slot is unspecified.**
  `bip300.md` asks the question and no source answers it. Read literally, the
  rules hand the incoming sidechain the outgoing sidechain's funds. This is a
  fund-safety decision — see Appendix A, item 12.
- **Whether to restore the one-proposal-per-block limit.** The original draft
  allowed one M1 per block, and one M3 per slot per block, as a rate limit
  against proposal spam and against overloading miner attention. The enforcer
  rejects only exact duplicates. Restoring the limit means changing the
  enforcer, not the documents — see Appendix A, item 9.
- **Whether `OP_DRIVECHAIN` should move off `OP_NOP5`.** `OP_NOP8` or `OP_NOP9`
  has been proposed, to stay clear of the numbers a future Bitcoin Core soft
  fork is most likely to want. The choice is arbitrary and costs nothing to
  change while nothing is deployed on mainnet; two candidates have been named
  and one must be picked — see Appendix A, item 13.
- **BIP-301's M8 wire format is provisional.** BMM bidding and accepting are
  being reworked. The M8 encoding in both prior drafts — output index, 3-byte
  tag, byte-exact script — descends from a superseded design in which a BMM
  request was a distinct transaction type, not an ordinary transaction with an
  `OP_RETURN`. BIP-301 Appendix A, items 3, 5 and 7 record what is enforced
  today and separate it from the four rules any redesign must preserve.

## Development

Markdown, JSON and YAML are formatted with
[oxfmt](https://github.com/oxc-project/oxc) (80 columns, prose wrapped). CI
rejects any diff.

```
just fmt
```

`just fmt` calls `bunx`, so it requires [bun](https://bun.sh).
`npx oxfmt@0.58.0 .` is equivalent when bun is unavailable. The `.mediawiki`
files are not touched by the formatter.
