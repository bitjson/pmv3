# CHIP-2021-01-RSN: Ranged Script Number Format

        Title: Ranged Script Number Format
        Parent CHIP: Version 3 Transaction Format (PMv3)
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Withdrawn
        Initial Publication Date: 2021-01-15
        Latest Revision Date: 2021-06-23

This proposal has been [withdrawn](https://bitcoincashresearch.org/t/chip-2021-01-pmv3-version-3-transaction-format/265/55?u=bitjson) in favor of [`CHIP-2022-02-CashTokens: Token Primitives for Bitcoin Cash`](https://github.com/bitjson/cashtokens).

## Summary

This proposal describes the `Ranged Script Number` (RSN) format. The RSN format is derived from the existing `Script Number` format used within the BCH VM, but it includes a prefix which allows RSN values to be safely parsed from serialized transaction messages.

## Deployment

This proposal is designed for deployment with [`CHIP: Version 3 Transaction Format (PMv3)`](./readme.md).

## Motivation

Many advanced BCH contracts inspect and validate the contents of transactions, and even if [native introspection operations](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md) are introduced, contracts which [inspect the contents of parent transactions](https://blog.bitjson.com/pmv3-build-decentralized-applications-on-bitcoin-cash/) will still need the ability to safely parse serialized transactions.

Bitcoin Cash transactions use 4 different formats for representing integers – `Uint32`, `Uint64`, `VarInt`, and `Script Number` (used within `Unlocking Bytecode` and `Locking Bytecode`). The `OP_BIN2NUM` and `OP_NUM2BIN` operations allow conversion between VM integers (Script Numbers) and unsigned integers, but the `VarInt` format remains unaddressed. Because `VarInt` is variable-width, a simple `OP_VARINT2NUM` and `OP_NUM2VARINT` solution wouldn't fully address the issue either: many contracts need to be able to safely parse a number from within a longer item on the stack.

Additionally, existing transaction integer formats are inefficient: the specified `Ranged Script Number` format saves ~12 bytes for small transactions and ~3 additional bytes per input/output.

## Benefits

By offering compatibility with the existing BCH VM `Script Number` format, the `Ranged Script Number` format simplifies contracts, reduces transaction sizes, and enables future upgrades.

### Simplifies In-Contract Parsing

Because RSN is directly compatible with Script Numbers for values below `128`, many use cases with fixed or capped integer requirements will require no parsing at all. Input counts, output counts, outpoint indexes, and bytecode lengths below `128` can be treated as standard Script Numbers.

For variable number ranges, all parsing can safely be accomplished using a [single 11-byte macro](#rsn-macro), rather than the [minimum 29-byte macro](#varint-macro-2-byte-limit) required of `VarInt` in most cases. (Note, the safe, [fully-equivalent `VarInt` macro is 53-bytes](#varint-macro-rsn-equivalent).) This simplification is notable enough to enable use cases which would otherwise be inhibited by parsing constructions causing contracts to exceed VM limits.

### Allows for Future Expansion

RSN is a positive integer format which is compatible with a signed integer format (Script Numbers), so it can make use of a range of unused values (the negative range of Script Numbers). This unused range is valuable both for implementing non-disruptive future upgrades (like [fractional satoshis](#upgrade-path-for-fractional-satoshis)) and for [off-chain signing and proof-of-ownership applications](#use-of-negative-script-number-range).

### Reduces Transaction Sizes

By eliminating waste from fixed-size encoding formats, the RSN format could save [~3.6% of blockchain space](./readme.md#reduced-transaction-sizes).

## Costs & Risk Mitigation

This proposal is a dependency of [`CHIP: Version 3 Transaction Format (PMv3)`](./readme.md) for which [costs and risks have been assessed](./readme.md#costs--risk-mitigation).

## Technical Specification

`Ranged Script Number` (RSN) is a new variable-length integer format based on the encoding already used to represent integers in Bitcoin Cash VM bytecode ("Script Numbers" or `CScriptNum` in the Satoshi implementation). Because standard `Script Numbers` do not indicate their length, they cannot be safely parsed from a serialized format without modification.

`Ranged Script Numbers` borrow from the space-saving strategy of the existing `VarInt` format: values from `0` (`0x00`) to `127` (`0x7f`) can be encoded in a single byte. To encode larger values, a prefix is added to indicate the length of the subsequent Script Number to read. Valid prefix values are `0x82` (`2`) through `0x87` (`7`). These are `-2` through `-7` in the VM respectively, so contracts can read the expected length using `<1> OP_SPLIT OP_SWAP OP_NEGATE`.

> In the VM, `0x80` represents negative zero, and `0x81` represents `-1`. `Script Numbers` require 2 bytes to represent values larger than `127`, so the `0x80` and `0x81` prefixes are never used in RSN.

The prefix `0x87` allows for the reading of `2.1×10^15` (`0x870040075af07507`), the number of satoshis in 21 million bitcoin cash. This is the maximum value required for parsing RSN-encoded numbers in v3 transactions, but future transaction versions may allow for larger prefix values in conjunction with a move to fractional satoshis. (E.g. `0x89` (9 bytes) would enable encoding `2.1×10^15` in 1/1000ths of a satoshi.)

Though the existing `Script Number` format supports negative integers, `Ranged Script Numbers` must always be positive integers.

### Test Vectors

Note, the RSN encoding for `0` (`0x00`) differs from that of standard Script Numbers (`0x`, an empty stack item). Other values from `1` through `127` are equivalent, then numbers greater than `127` are encoded with the proper length prefix.

| Value           | Script Number      | RSN Encoded          |
| --------------- | ------------------ | -------------------- |
| 0               | `0x`               | `0x00`               |
| 1               | `0x01`             | `0x01`               |
| 2               | `0x02`             | `0x02`               |
| 3               | `0x03`             | `0x03`               |
| 126             | `0x7e`             | `0x7e`               |
| 127             | `0x7f`             | `0x7f`               |
| 128             | `0x8000`           | `0x828000`           |
| 129             | `0x8100`           | `0x828100`           |
| 254             | `0xfe00`           | `0x82fe00`           |
| 255             | `0xff00`           | `0x82ff00`           |
| 256             | `0x0001`           | `0x820001`           |
| 32766           | `0xfe7f`           | `0x82fe7f`           |
| 32767           | `0xff7f`           | `0x82ff7f`           |
| 32768           | `0x008000`         | `0x83008000`         |
| 32769           | `0x018000`         | `0x83018000`         |
| 65534           | `0xfeff00`         | `0x83feff00`         |
| 65535           | `0xffff00`         | `0x83ffff00`         |
| 65536           | `0x000001`         | `0x83000001`         |
| 8388607         | `0xffff7f`         | `0x83ffff7f`         |
| 16777214        | `0xfeffff00`       | `0x84feffff00`       |
| 16777215        | `0xffffff00`       | `0x84ffffff00`       |
| 16777216        | `0x00000001`       | `0x8400000001`       |
| 822083584       | `0x00000031`       | `0x8400000031`       |
| 2147483646      | `0xfeffff7f`       | `0x84feffff7f`       |
| 2147483647      | `0xffffff7f`       | `0x84ffffff7f`       |
| 2147483648      | `0x0000008000`     | `0x850000008000`     |
| 4294967294      | `0xfeffffff00`     | `0x85feffffff00`     |
| 4294967295      | `0xffffffff00`     | `0x85ffffffff00`     |
| 4294967296      | `0x0000000001`     | `0x850000000001`     |
| 549755813887    | `0xffffffff7f`     | `0x85ffffffff7f`     |
| 549755813888    | `0x000000008000`   | `0x86000000008000`   |
| 140737488355327 | `0xffffffffff7f`   | `0x86ffffffffff7f`   |
| 140737488355328 | `0x00000000008000` | `0x8700000000008000` |
| 2.1×10^15       | `0x0040075af07507` | `0x870040075af07507` |

## Rationale

This section documents design decisions made in this specification.

### Use of Negative Script Number Range

The existing `Script Number` format (A.K.A. `CScriptNum`) has unusual properties: values from `0x00` to `0x7f` represent `0` through `127`, respectively, while `0x80` through `0xff` represent `-0` (negative zero) through `-127`. Because `Script Numbers` cannot represent values larger than `127` in a single byte, `128` is the ideal number for `Ranged Script Numbers` to also expand to multiple bytes. Because we are still left with the single-byte negative range, this range is ideal for indicating the `Ranged Script Numbers`' byte length.

One alternative is to use `0x00` or `0x80` as a special case to indicate that the following byte is a byte length indicator, e.g. `0x80020001` rather than `0x820001` (`256`). This would make parsing in VM contracts slightly simpler (no need for `OP_NEGATE`) at the cost of 1-byte per number larger than `127`. However, because only a small fraction of transactions on the network will require introspecting these numbers, this inconvenience is worth the additional byte saved per integer (~4 bytes for small transactions, with an additional `byte * (inputs + outputs)`).

Another notable benefit of the RSN format for satoshi values is the availability of a known-invalid range. Because prefixes larger than `0x87` (for 7 byte Script Numbers) are invalid for v3 transactions, higher prefixes can safely be used by off-chain proof protocols, e.g. a transaction with a 1-byte output value of `0xff` is guaranteed to be rejected by the chain and can therefore be used to safely prove ownership of inputs (e.g. [Bitauth](https://github.com/bitauth/bitauth-cli), Proof-of-Reserves, etc.).

### Usage Comparison vs. VarInt

With the RSN format, it's possible to safely parse Ranged Script Numbers of any size with the following 11-byte macro (parses RSN of any size from the top item on the stack, leaving the remaining portion of the parsed item and placing the parsed number above on the stack):

#### RSN Macro

```
OP_1 OP_SPLIT OP_SWAP OP_DUP
OP_1NEGATE OP_LESSTHAN
OP_IF
  OP_NEGATE OP_SPLIT OP_SWAP
OP_ENDIF
```

A mostly-equivalent (up to `65535`) implementation for the VarInt format is the following 29-byte macro:

#### VarInt Macro (2-byte limit)

```
OP_1 OP_SPLIT OP_SWAP
OP_DUP <0xfd> OP_EQUAL
OP_IF
  OP_DROP <2> OP_SPLIT OP_SWAP OP_BIN2NUM
OP_ELSE
  <0x80> OP_XOR OP_DUP
  OP_1NEGATE OP_GREATERTHAN
  OP_IF
    <128> OP_ADD
  OP_ELSE
    OP_NEGATE
  OP_ENDIF
OP_ENDIF
```

However, if a v3 transaction format were to compress transactions by using the VarInt format for output values ([saving 2-7 bytes per output](./readme.md#use-of-rsn-for-satoshi-values)), parsing would require the following 53-byte macro (which supports VarInt values up to 8-bytes, fully equivalent to the 11-byte RSN macro above):

#### VarInt Macro (RSN equivalent)

```
OP_1 OP_SPLIT OP_SWAP
OP_DUP <0xfd> OP_EQUAL
OP_IF
  OP_DROP <2> OP_SPLIT OP_SWAP
  OP_BIN2NUM
OP_ELSE
  OP_DUP <0xfe> OP_EQUAL
  OP_IF
    OP_DROP <4> OP_SPLIT OP_SWAP
    OP_BIN2NUM
  OP_ELSE
    OP_DUP <0xff> OP_EQUAL
    OP_IF
      OP_DROP <8> OP_SPLIT OP_SWAP
      OP_BIN2NUM
    OP_ELSE
      <0x80> OP_XOR OP_DUP
      OP_1NEGATE OP_GREATERTHAN
      OP_IF
        <128> OP_ADD
      OP_ELSE
        OP_NEGATE
      OP_ENDIF
    OP_ENDIF
  OP_ENDIF
OP_ENDIF
```

For details and interactive example evaluations, see [the full analysis in Bitauth IDE](https://ide.bitauth.com/import-template/eJztVmtP2zAU_StWxAeQ-khSChlCSIEGqNSWri3sVRSZxGkzmodil3VC_Pdd50HiJNUqJDZpWj-0ju-95xzb9zR-lvaotSQelk6kJWMhPWm3H1yG12zZsgKvnQRpm08Qn7kWZm7gNxnxwhVmpPkkt5KU1nca-FJDsgm1IjfkWQCpI5t4gU9ZFNehwEEhjqjrLxBM-RRb8bTrM7IgEUVOFHgII8qw9YhcYEHYigJKESVPJMIrFEZBGFC8oi3g8rFHgGScIs4KiP0M0fURKEd3Q8jnC2AuodLJ80tDSnTyB4lrIiaUL4htJvOmv_YeSMSjBRqCJnESmsZJaJQkZWCQdTM2FQRf0_GgP4sHn_Qx_-3djtHc5-GRcaXPDD43MKbT2bU-iuf7l3Mf8dk8LoDEScao178EPkYol_7tWYLdtx6B-FQ545nGx1t9cGdM-pdf5v6pvJHhUw7kW6dw6YStQxi324hv44qgh5-MIAtTkiAoKIOBFEV6aRRJOxX0XWg7skjsrVfMbb7y0hhDUzVVlgVyqBPo1e7RG-ihaid6ABHWzusE-uM6nh0ExHUFCad7-wWsaf-rUeqDtIcu9NlBJqh0DmrnsHt0rH2Q39QGeXVJVRl3V233YLDEVU84Anubary9poc3FUvd4QjsivbV5jk_gSHeHPzWUbEZuKVgbY6dr0x0Um9yAxnqWa0hz_sjdXQ7TGw1mBq8hh97sk-fbyapa1Osgm2vJgaMJolz42DCCPWKqsXleq-XhjLogrWzCPdy7uq5_-_5ukjsudTDzFrmCjT1T5ib_Qia4sodu9bZlZ7VTF5HtzashnjD0nfv1jRe17AFXFLEFbryFfuwHltEr3ZtxuCIDAJHgUXbxlLmKTFtt1-aut2CFTFVK9bQlSyZZSRmLD9VDfvfrn_Vru_xIgZsJ1hHZQHkWFYUWREvI0mtIEPedLqKZVu2I_PLw9s01b-MgZC4iyUrS3PKjMIZ5VDwdoE_OH7tXYdhEDFiQ9tK5xfXpgqVptyV7hsS3LFpfHGXX34BZNCDfA==).

### Upgrade Path for Fractional Satoshis

At a purchasing power of ~$100,000 (USD, 2020), Bitcoin Cash transaction fees will begin rising in real terms due to the indivisibility of satoshis; a 1 satoshi fee will be more expensive than today's median fees. By defining a variable-length integer format for satoshi values, this specification offers several upgrade paths for implementing fractional satoshis. A future transaction format could either migrate output values to a smaller base unit or implement a fractional satoshi scheme within the unused prefix byte range (`0x88` through `0xFE`) of output value RSNs.

### Use in PMv3

Additional rationale is available for the [design and use of the RSN format in the context of PMv3](./readme.md#use-of-rsn-format).

## Implementations

This proposal is a dependency of [`CHIP: Version 3 Transaction Format (PMv3)`](./readme.md) which includes [a list of implementations](./readme.md#implementations).

## Changelog

This section summarizes the evolution of this document.

- **v1.1 - 2021-6-23** (current)
  - Extracted from `CHIP-2021-01-RSN` (to simplify external references)
  - Added `Usage Comparison vs. VarInt`
- **v1.0 – 2021-1-15** ([`a0771cef`](https://github.com/bitjson/pmv3/blob/a0771cef0f515c8e9187dc987edaf500348a93e4/readme.md))
  - Initial publication

## Feedback & Reviews

- [`CHIP 2021-01 Ranged Script Numbers` – June 23, 2021 | bitcoincashresearch.org](https://bitcoincashresearch.org/t/chip-2021-01-ranged-script-numbers/498)

## Copyright

This document is placed in the public domain.
