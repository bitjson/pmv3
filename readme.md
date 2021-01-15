# PMv3: A Draft v3 Transaction Format

This specification describes a version 3 transaction format for Bitcoin Cash. This format:

- Enables fixed-size inductive proofs (for covenants like [CashTokens](https://blog.bitjson.com/cashtokens-contract-validated-tokens-for-bitcoin-cash-e2479c30c764)) by allowing transactions to optionally include a hash for any unlocking bytecode.
- Unifies the 4 transaction integer formats around the existing virtual machine integer format ("Script Numbers").
- Defines an upgrade path for the subdivision of output values ("fractional satoshis").
- Reduces transaction sizes by cutting wasted bytes: ~12 bytes for small transactions, and ~3 additional bytes per input/output.
- Minimizes changes to the [existing transaction format](https://reference.cash/protocol/blockchain/transaction).

This proposal uses the title `PMv3` (a reference to prediction markets, one use-case it enables) to make it easier to reference. If accepted by the network, the format can simply be called `v3`.

## Deployment

Deployment of this specification is proposed for the May 2022 upgrade.

## Motivation

This format includes two related changes to the `v2` transaction format.

### Fixed-Size Inductive Proofs

Many token and covenant designs can be supported within the existing Bitcoin Cash VM design and opcode limits, but one unintentional limitation is resistant to workarounds: the quadratically increasing size of inductive proofs. With fixed-size inductive proofs, contract designers can create SPV and contract-validatable tokens, token-issuing covenants, and covenants which interact with each other.

### Unified Integer Format

Bitcoin Cash transactions use 4 different formats for representing integers (`Uint32`, `Uint64`, `VarInt`, and `Script Numbers`). While Bitcoin Cash enables transaction introspection with `OP_CHECKDATASIG`, contracts which operate on integers appearing within the transaction format are forced to use complex workarounds to convert between integer formats.

This problem is partially alleviated by `OP_BIN2NUM` and `OP_NUM2BIN`, which allow conversion between VM integers (`Script Numbers`) and unsigned integers, but the `VarInt` format remains unaddressed. Additionally, existing transaction integer formats are inefficient: the specified [`Ranged Script Number`](#ranged-script-numbers-rsn) format saves ~12 bytes for small transactions and ~3 additional bytes per input/output.

## Format

The transaction format is mostly unchanged, but some integers are replaced with `Ranged Script Numbers`, a format which is compatible with the existing Bitcoin Cash VM integer format (`Script Numbers`). An optional [`Hashed Witnesses`](#hashed-witness) serialization may also be included after `Locktime`.

### Transaction

| Field               | v1/v2 Format                   | PMv3 Format                                               | Description                                                          |
| ------------------- | ------------------------------ | --------------------------------------------------------- | -------------------------------------------------------------------- |
| Version             | Uint32 (4&nbsp;bytes)          | [RSN](#ranged-script-numbers-rsn) (1&nbsp;byte)           | The version of the transaction format (`0x03`).                      |
| Input Count         | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The number of inputs in the transaction.                             |
| Transaction Inputs  | [Inputs](#transaction-input)   | [Inputs](#transaction-input)                              | Each of the transaction’s inputs serialized in order.                |
| Output Count        | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The number of outputs in the transaction.                            |
| Transaction Outputs | [Outputs](#transaction-output) | [Outputs](#transaction-output)                            | Each of the transaction’s outputs serialized in order.               |
| Locktime            | Uint32 (4&nbsp;bytes)          | Uint32 (4&nbsp;bytes)                                     | A time or block height at which the transaction is considered valid. |
| Hashed Witnesses    | N/A                            | [Hashed Witnesses](#hashed-witness)                       | Each of the transaction's hashed witnesses in serialized order.      |

In all P2P protocol messages, the transaction is transmitted in its entirety (including any hashed witnesses, if present).

When computing the transaction's hash for `Outpoint Transaction Hash` and for other P2P messages, any hashed witnesses are truncated from the preimage. (Hashed witness integrity is guaranteed by the witness hash in each respective input serialization.)

### Transaction Input

The transaction input format is mostly unchanged, but most integers are replaced with [Ranged Script Numbers](#ranged-script-numbers-rsn). Additionally, a 32-byte, double-sha256 `Witness Hash` may be provided in place of `Unlocking Bytecode` if `Unlocking Bytecode Length` is set to `0x00`.

Transactions opt-in to the use of hashed witnesses on a per-input basis, so any or all inputs within a transaction may use hashed witnesses.

| Field                              | v1/v2 Format                   | PMv3 Format                                               | Description                                                                                                                           |
| ---------------------------------- | ------------------------------ | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Outpoint Transaction Hash          | hash (32&nbsp;bytes)           | hash (32&nbsp;bytes)                                      | The hash of the transaction containing the output being spent.                                                                        |
| Outpoint Index                     | Uint32 (4&nbsp;bytes)          | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The zero-based index of the output being spent from the previous transaction.                                                         |
| Unlocking Bytecode Length          | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The size of the unlocking script in bytes. If set to `0x00`, the next item is a witness hash.                                         |
| Unlocking Bytecode OR Witness Hash | bytecode                       | bytecode OR hash                                          | If `Unlocking Bytecode Length` is `0x00`, a 32-byte double-sha256 [witness hash](#hashed-witness). Otherwise, the unlocking bytecode. |
| Sequence Number                    | Uint32 (4&nbsp;bytes)          | Uint32 (4&nbsp;bytes)                                     | As of BIP-68, the sequence number is interpreted as a relative locktime for the input.                                                |

### Transaction Output

The transaction output format is mostly unchanged. Both number formats are replaced with [Ranged Script Numbers](#ranged-script-numbers-rsn).

| Field                   | v1/v2 Format                   | PMv3 Format                                               | Description                               |
| ----------------------- | ------------------------------ | --------------------------------------------------------- | ----------------------------------------- |
| Value                   | Uint64 (8&nbsp;bytes)          | [RSN](#ranged-script-numbers-rsn) (1&#x2011;8&nbsp;bytes) | The number of satoshis to be transferred. |
| Locking Bytecode Length | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The byte-length of the locking bytecode.  |
| Locking Bytecode        | bytecode                       | bytecode                                                  | The locking bytecode.                     |

### Hashed Witness

Hashed Witnesses are identified by the double-sha256 hash of their serialization (`Unlocking Bytecode Length + Unlocking Bytecode`).

During transaction validation, if any inputs reference a hashed witness, that witness must be present in the `Hashed Witnesses` transaction section. If multiple hashed witnesses are used, they must appear in input order for the transaction to be valid.

| Field                     | v1/v2 Format | PMv3 Format                                               | Description                                |
| ------------------------- | ------------ | --------------------------------------------------------- | ------------------------------------------ |
| Unlocking Bytecode Length | N/A          | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The byte-length of the unlocking bytecode. |
| Unlocking Bytecode        | N/A          | bytecode                                                  | The unlocking bytecode.                    |

Hashed witnesses increases a transaction's size and cost by 32-bytes per input, so it is recommended that wallets avoid use of hashed witnesses unless required by a contract's design.

### Ranged Script Numbers (RSN)

`Ranged Script Numbers` (RSN) are a variable-length integer format which borrows from the space-saving strategy of the current `VarInt` format while relying on the encoding already used to represent integers in Bitcoin Cash VM bytecode ("Script Numbers" or `CScriptNum` in the Satoshi implementation).

Values from `0` (`0x00`) to `127` (`0x7f`) can be encoded in a single byte. To encode larger values, a prefix is added to indicate the length of the subsequent Script Number to read. Valid prefix values are `0x82` (`2`) through `0x87` (`7`). These are `-2` through `-7` in the VM respectively, so contracts can read the expected length using `<1> OP_SPLIT OP_SWAP OP_NEGATE`. (In the VM, `0x80` represents negative zero, and `0x81` represents `-1`.)

The prefix `0x87` allows for the reading of `2.1×10^15` (`0x870040075af07507`), the number of satoshis in 21 million bitcoin cash. This is the maximum value required for parsing RSN-encoded numbers in v3 transactions, but future transaction versions may allow for larger prefix values in conjunction with a move to fractional satoshis. (E.g. `0x89` (9 bytes) would enable divisibility of less than 1/1000th of a satoshi.)

Though the existing `Script Number` format supports negative integers, `Ranged Script Numbers` must always be positive integers in the `PMv3` transaction format.

#### Test Vectors

| Value           | Script Number      | RSN Encoded          |
| --------------- | ------------------ | -------------------- |
| 0               | `0x00`             | `0x00`               |
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

This section documents design decisions made in the PMv3 format.

### Optional Hashing of Witness Data

Hashed witnesses are a minimal strategy for allowing descendant transactions to include their parent transactions as unlocking data without being forced to include their full grandparent transactions (and so on). If the parent transaction chose to provide a hash for any of its unlocking scripts, child transactions can reduce the byte size of their own unlocking data by referencing only the hash. This enables "token covenants" which inductively prove their lineage without the proof size increasing each time the token is moved.

Other strategies, like [Segregated Witness](https://en.wikipedia.org/wiki/SegWit) could also enable this functionality, but hashed witnesses have the benefit of allowing transactions to opt-in on a per-input basis, avoiding unnecessary hash computations and saving block space. Hashed witnesses also avoid modifying the structure of the blockchain, retaining the cryptographic link of all witness data to the transaction it authorizes.

### Use of VM Integer Format

The `Ranged Script Number` (RSN) format is chosen for its ease of conversion to and from standard Script Numbers within contracts. The VM Script Number format is already consensus-critical, so it is less disruptive to migrate transaction format integer types to the Script Number format than to migrate the reverse.

One alternative solution is to implement `OP_NUM2VARINT` and `OP_VARINT2NUM` operations. This would allow for slightly better compression of `VarInt`s between `128` and `252` (2 bytes), but in practice relatively few transactions use those numbers for input count, output count, or bytecode lengths. This inefficiency is also dwarfed by the space savings of compressing `Version` (3 bytes), `Outpoint Index` (approx. 3 bytes per input), and output `Value` (2-7 bytes per output).

Note, many other P2P protocol messages use other integer formats, and these remain unchanged. This includes the P2P protocol `tx` message header, which should continue to use the standard 24-byte header format.

### Use of Negative Script Number Range in RSN

The existing `Script Number` format (A.K.A. `CScriptNum`) has unusual properties: values from `0x00` to `0x7f` represent `0` through `127`, respectively, while `0x80` through `0xff` represent `-0` (negative zero) through `-127`. Because `Script Numbers` cannot represent values larger than `127` in a single byte, `128` is the ideal number for `Ranged Script Numbers` to also expand to multiple bytes. Because we are still left with the single-byte negative range, this range is ideal for indicating the `Ranged Script Numbers`' byte length.

One alternative is to use `0x00` or `0x80` as a special case to indicate that the following byte is a byte length indicator, e.g. `0x80020001` rather than `0x820001` (`256`). This would make parsing in VM contracts slightly simpler (no need for `OP_NEGATE`) at the cost of 1-byte per number larger than `127`. However, because only a small fraction of transactions on the network will require introspecting these numbers, this inconvenience is worth the additional byte saved per integer (~4 bytes for small transactions, with an additional `byte * (inputs + outputs)`).

Another notable benefit of the RSN format for satoshi values is the availability of a known-invalid range. Because prefixes larger than `0x87` (for 7 byte Script Numbers) are invalid, higher prefixes can safely be used by off-chain proof protocols, e.g. an output value of `0xff` is guaranteed to be rejected by the chain, and can therefore be used to safely prove ownership of an address (e.g. [Bitauth](https://github.com/bitauth/bitauth-cli), Proof-of-Reserves, etc.).

### Use of RSN for Satoshi Values

The `Ranged Script Number` format offers superior compression vs. the current `Uint64` format, offering dramatic per-output savings:

- Empty OP_RETURN outputs (`0` satoshis) save 7 bytes.
- Outputs smaller than `32,767` satoshis save 5 bytes.
- Outputs smaller than `8,388,607` satoshis (`0.008 bitcoin cash`) save 4 bytes.
- Outputs smaller than `2,147,483,647` satoshis (`~2.1 bitcoin cash`) save 3 bytes.
- Outputs smaller than `549,755,813,888` satoshis (`~549 bitcoin cash`) save 2 bytes.
- Outputs smaller than `140,737,488,355,328` satoshis (`~140,737 bitcoin cash`) save 1 byte.
- Larger outputs use 8 bytes (no savings or loss vs. `Uint64`).

### Use of Uint32 for Locktime and Sequence Numbers

An earlier version of this specification proposed the use of the RSN format for all integers, including `Locktime` and `Sequence Number`. However, these fields commonly have values larger than `32767` (3-byte `RSN`) – the largest value for which RSN is more compressed than `Uint32` (4 bytes). Worse, the RSN format adds 1 byte for values larger than `16777214`, and up to 2 bytes for the maximum valid value of each field (`4294967295`). Based on current network usage, `Uint32` offers better median and average compression for these fields than `RSN`.

Additionally, Because `OP_BIN2NUM` and `OP_NUM2BIN` make conversion possible between `Script Numbers` and unsigned integers, keeping the `Uint32` format does not present a barrier for contract designers.

### Upgrade Path for Fractional Satoshis

At a price of ~$100,000 USD, Bitcoin Cash transaction fees will be forced to rise in real terms due to the indivisibility of satoshis (a 1 satoshi fee will be more expensive than today's median fees). By defining a variable-length integer format for satoshi values, this specification paves the way for future transaction formats which specify output values in fractional satoshis. For example, a simple v4 transaction format could be implemented which matches the v3 format but requires output values to be specified in 1/1000ths of a satoshi.

### VM Support for Larger Script Numbers

This proposal does not currently include a specification for increasing the supported range of integers within the VM (currently: 32-bit signed integers). Because the proposed `Ranged Script Number` (RSN) format already uses larger numbers, these numbers could not be operated upon by the VM without a larger integer upgrade. As such, it's recommended that the allowable range of VM integers be expanded (to at least 64-bit signed integers) before or at the same time as the deployment of this specification.

### Privacy Implications

New transaction formats necessarily splinter the anonymity set, making all users easier to de-anonymize. Because this v3 transaction format is designed to enable novel contracts on Bitcoin Cash, the set of transactions which are likely to immediately upgrade to this transaction version are already highly traceable (due to their unique locking and unlocking bytecode patterns).

If a goal is to avoid having most other users switch to v3 transactions (remaining on v1 transactions, which currently have the largest anonymity set), it may be reasonable to intentionally avoid making v3 transactions cheaper, since reduced fees may be an incentive to switch. However, avoiding space savings in transaction format upgrades seems like a poor trade-off, as much stronger privacy can be achieved with solutions like CashShuffle or CashFusion. It would seem the most valuable contribution a transaction format can make toward network privacy is to enable more use-cases and users (as this proposal would).

## Implementations

Please see the following reference implementations for examples and test vectors:

[TODO: after initial public feedback]

## Copyright

This document is placed in the public domain.
