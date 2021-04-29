# CHIP-2021-01-PMv3: Version 3 Transaction Format

        Title: Version 3 Transaction Format
        Proposal: PMv3
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2021-01-15
        Latest Revision Date: 2021-04-28

## Summary

This proposal describes a version 3 transaction format for Bitcoin Cash which:

- Enables fixed-size inductive proofs and signing of transaction input contents.
- Consolidates transaction format integer encodings to be compatible with contracts.
- Reduces transaction sizes by eliminating wasted bytes.

## Deployment

Deployment of this specification is proposed for the May 2022 upgrade.

## Motivation

By addressing two issues with the transaction format, Bitcoin Cash contracts can support **cross-contract interfaces**, allowing contracts to interact with each other.

### Fixed-Size Inductive Proofs

Many token and covenant designs can be supported within the existing Bitcoin Cash VM design and opcode limits, but one unintentional limitation is resistant to workarounds: the [quadratically increasing size of inductive proofs](https://blog.bitjson.com/hashed-witnesses-build-decentralized-applications-on-bitcoin-cash/). With fixed-size inductive proofs, contract designers can create SPV-compatible (Simplified Payment Verification) and contract-validated tokens, token-issuing covenants, and covenants which interact with each other.

### Unified Integer Format

Bitcoin Cash transactions use 4 different formats for representing integers – `Uint32`, `Uint64`, `VarInt`, and `Script Numbers` (used within `Unlocking Bytecode` and `Locking Bytecode`). While Bitcoin Cash allows transaction introspection with `OP_CHECKDATASIG`, contracts which operate on integers appearing within the transaction format are forced to use complex workarounds to convert between integer formats.

This problem is partially alleviated by `OP_BIN2NUM` and `OP_NUM2BIN` – which allow conversion between VM integers (`Script Numbers`) and unsigned integers, but the `VarInt` format remains unaddressed. Additionally, existing transaction integer formats are inefficient: the specified [`Ranged Script Number`](#ranged-script-numbers-rsn) format saves ~12 bytes for small transactions and ~3 additional bytes per input/output.

## Benefits

By addressing these two issues, this v3 transaction format offers several benefits.

### Decentralized Applications & Permissionless Innovation

With the ability to inspect parent transactions, Bitcoin Cash contracts can be designed to interact with other contracts, enabling complex decentralized applications.

With version 3 transactions, [new token schemes](https://blog.bitjson.com/cashtokens-contract-validated-tokens-for-bitcoin-cash-a8de58f5b7d8) and other high-level contract protocols can be implemented **entirely at the wallet/application layer**. Deployment of these systems can happen without network-wide coordination: developers are free to innovate without permission, reducing the need for network upgrades.

For larger decentralized applications, cross-contract interfaces also enable a new scaling strategy: complex contracts can be subdivided into smaller contracts, allowing many users to simultaneously interact with large, public covenants. Activity in subcontracts can later be merged back into parent contracts, reducing contention over the order in which users interact with public covenants.

### Comprehensive Malleability Protection

As a side effect of enabling interoperability between contracts, the optional `Detached Signatures` field comprehensively solves transaction malleability.

Prior to this proposal, no available signature types can cover the contents of unlocking bytecode – contract developers must carefully design contracts to individually validate each pushed value and prevent malicious transaction malleation.

With detached signatures, it becomes straightforward to design and validate secure, non-malleable contracts. Existing contracts can be simplified, reducing both byte size and operation count.

### Reduced Transaction Sizes

As a side effect of consolidating transaction format integer encodings, all transactions save a minimum of 3 bytes plus ~3 bytes for each input and output. This waste in the existing encoding formats makes up about 3.6% of historical blockchain data (5.7 GB of 161 GB)<sup>1</sup>.

Detached signatures also enable much larger savings in transactions which consolidate multiple inputs from the same address. By reusing the same signature across inputs, these large transactions can immediately save 63 to 71 bytes per input<sup>2</sup>.

By comprehensively solving malleability, this proposal also [unblocks future upgrades](https://bitcoincashresearch.org/t/supporting-input-aggregation-musig-using-transaction-introspection/350) which can save up to 108 bytes per input for P2PKH consolidation transactions<sup>3</sup> (via deduplication of unlocking bytecode) and 64 to 72 bytes per input for all P2PKH transactions<sup>4</sup> (via signature aggregation). These strategies would enable a ~20% reduction<sup>5</sup> in average transaction size across the network.

<details>
 <summary>Calculations</summary>

1. Waste is `(764,000,000 inputs + 849,000,000 outputs) * 3 bytes + (323,000,000 transactions * 3 bytes)`. 5.8 GB of 161 GB, or approx. 3.6% of historical blockchain data as block `683273` (2021-4-14).
2. Schnorr signatures (65 bytes) and ECDSA signatures (70-73 bytes) are both reduced to 2 bytes per subsequent occurrence.
3. In P2PKH inputs contain 2 bytes of push operations, a signature (between 65 and 73 bytes), and a public key (33 bytes). Saving are between 100 and 108 bytes per duplicate P2PKH input.
4. With signature aggregation, P2PKH inputs would likely contain 3 bytes of push operations (e.g. `<detached_signature_index sighash_byte> <public_key>`) and a public key (33 bytes). Before: 100-108 bytes. After: 36 bytes. Reduction: 64 to 72 bytes.
5. See [analysis here](https://bitcoincore.org/en/2017/03/23/schnorr-signature-aggregation/#signature-aggregation).

</details>

## Costs & Risk Mitigation

The following costs and risks have been assessed.

### Node Deployment

Node implementations, wallet infrastructure, and other consensus-validating software must upgrade to support these new consensus rules.

### Wallet Deployment

While many upgrade proposals can be deployed only by coordinating upgrades among node implementations, deployment of this v3 transaction format may also require upgrades to wallet software and infrastructure, and excessive upgrades can incentivize businesses to avoid or abandon investments in Bitcoin Cash infrastructure.

**Mitigations**: Existing transaction formats remain functional, and all outputs remain spendable by all transaction formats. Wallets which do not add support for sending v3 payments will not experience disruptions beyond typical node upgrade requirements. SPV wallets and wallets with unusual UTXO lookup APIs may need to deploy changes to client-side software.

### Privacy Implications

New transaction formats divide the anonymity set, making all users easier to de-anonymize.

**Mitigations**: privacy-sensitive users may choose to avoid using v3 transactions until a notable percentage of network transactions have already switched to v3 transactions. Wallets should also implement support for comprehensive privacy solutions like [CashFusion](https://cashfusion.org/).

## Technical Specification

The v3 transaction format follows the [existing v1/v2 transaction format](https://reference.cash/protocol/blockchain/transaction), but most integers are encoded as `Ranged Script Numbers`, a format which is compatible with the existing Bitcoin Cash VM bytecode integer format (`Script Numbers`). In addition, v3 transactions add new "detached" fields after `Locktime`: [`Detached Signatures`](#detached-signature) and [`Detached Proofs`](#detached-proof).

> Early drafts of this proposal called the `Detached Proofs` field "Hashed Witnesses".

### Transaction

The following tables compare the fields of the existing format with their v3 encoding.

| Field                    | v1/v2 Format                   | PMv3 Format                                               | Description                                                                      |
| ------------------------ | ------------------------------ | --------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Version                  | Uint32 (4&nbsp;bytes)          | [RSN](#ranged-script-numbers-rsn) (1&nbsp;byte)           | The version of the transaction format (`0x03`).                                  |
| Input Count              | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The number of inputs in the transaction.                                         |
| Transaction Inputs       | [Inputs](#transaction-input)   | [Inputs](#transaction-input)                              | Each of the transaction’s inputs serialized in order.                            |
| Output Count             | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The number of outputs in the transaction.                                        |
| Transaction Outputs      | [Outputs](#transaction-output) | [Outputs](#transaction-output)                            | Each of the transaction’s outputs serialized in order.                           |
| Locktime                 | Uint32 (4&nbsp;bytes)          | Uint32 (4&nbsp;bytes)                                     | A time or block height at which the transaction is considered valid.             |
| **Detached Fields**      |
| Detached Signature Count | N/A                            | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The number of detached signatures in the transaction.                            |
| Detached Signatures      | N/A                            | [Detached Signatures](#detached-signatures)               | Each of the transaction's detached signatures serialized in order.               |
| Detached Proof Count     | N/A                            | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The number of detached proofs in the transaction.                                |
| Detached Proofs          | N/A                            | [Detached Proofs](#detached-proofs)                       | Each of the transaction's detached proofs serialized in canonical order by hash. |

When computing the preimage of detached signatures (`SIGHASH_DETACHED`), the encoded transaction is truncated after `Locktime`.

When computing the transaction's hash (`TXID`) for the block merkle tree, `Outpoint Transaction Hash`, and for all other P2P messages, the encoded transaction is truncated before `Detached Proof Count`, if present. (Detached proof integrity is guaranteed by the hash in any inputs referencing detached proofs.)

In all P2P protocol messages, the transaction is transmitted in its entirety, including any detached signatures and/or detached proofs, if present. Though the TXID preimage does not re-include the `Detached Proof Count` and `Detached Proof` fields (which are attested to within `Transaction Inputs`), a transaction is considered invalid if it is missing any referenced `Detached Proofs`.

> Note: node implementations must never blacklist TXIDs for non-presence of detached proof(s), but should ban peers which repeatedly transmit malformed transactions (such as transactions from which detached proofs have been dropped).

### Transaction Input

The transaction input format is mostly unchanged, but most integers are replaced with [Ranged Script Numbers](#ranged-script-numbers-rsn). Additionally, a 32-byte, double-SHA256 `Detached-Proof Hash` may be provided in place of `Unlocking Bytecode` if `Unlocking Bytecode Length` is set to `0x00`.

Transactions opt-in to the use of detached proofs on a per-input basis, so any or all inputs within a transaction may reference detached proofs.

| Field                                     | v1/v2 Format                   | PMv3 Format                                               | Description                                                                                                                                    |
| ----------------------------------------- | ------------------------------ | --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Outpoint Transaction Hash                 | hash (32&nbsp;bytes)           | hash (32&nbsp;bytes)                                      | The hash of the transaction containing the output being spent.                                                                                 |
| Outpoint Index                            | Uint32 (4&nbsp;bytes)          | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The zero-based index of the output being spent from the previous transaction.                                                                  |
| Unlocking Bytecode Length                 | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The size of the unlocking script in bytes. If set to `0x00`, the next item is a detached-proof hash.                                           |
| Unlocking Bytecode OR Detached-Proof Hash | bytecode                       | bytecode OR hash                                          | If `Unlocking Bytecode Length` is `0x00`, a 32-byte, double-SHA256 [detached-proof hash](#detached-proofs). Otherwise, the unlocking bytecode. |
| Sequence Number                           | Uint32 (4&nbsp;bytes)          | Uint32 (4&nbsp;bytes)                                     | As of BIP-68, the sequence number is interpreted as a relative locktime for the input.                                                         |

### Transaction Output

The transaction output format is mostly unchanged. Both integer formats are replaced with [Ranged Script Numbers](#ranged-script-numbers-rsn).

| Field                   | v1/v2 Format                   | PMv3 Format                                               | Description                               |
| ----------------------- | ------------------------------ | --------------------------------------------------------- | ----------------------------------------- |
| Value                   | Uint64 (8&nbsp;bytes)          | [RSN](#ranged-script-numbers-rsn) (1&#x2011;8&nbsp;bytes) | The number of satoshis to be transferred. |
| Locking Bytecode Length | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The byte-length of the locking bytecode.  |
| Locking Bytecode        | bytecode                       | bytecode                                                  | The locking bytecode.                     |

### Detached Signatures

The v3 format includes a new transaction field called `Detached Signatures`.

| Field                     | v1/v2 Format | PMv3 Format                                               | Description                                                                                                               |
| ------------------------- | ------------ | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Detached Signature Length | N/A          | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The byte-length of the following detached signature field.                                                                |
| Detached Signature        | N/A          | bytecode                                                  | The detached signature (either [BIP66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki) ECDSA or Schnorr). |

Detached signatures are signatures which use the [`SIGHASH_DETACHED` signing serialization algorithm](#sighash_detached-algorithm--flag) (the version 3 transaction encoding). The `Detached Signatures` field is an ordered list of these signatures.

By committing to the entire encoded transaction – including the contents of all inputs – **detached signatures prevent all third-party malleability**. Applications which use detached proofs should use detached signatures to prevent third parties from manipulating inputs to detach or re-attach proofs. Other applications should also use detached signatures to prevent malleability or reduce transaction sizes.

Detached signatures are referenced by their index from unlocking bytecode using `signature references` (described below).

Detached signatures must not be duplicated (as this would introduce a malleability vector); duplication in the `Detached Signatures` field renders a transaction invalid.

> Note: its possible for more than one input to reference the same signature.

Use of detached signatures is optional. The encoding of transactions which include no detached signatures must end with the `Locktime` field; the `Detached Signature Count` must never be `0x00` (transactions without detached signatures must not encode the field).

> Note: detached signatures are "detached" from the inputs which use them, allowing them to be excluded from their own signing serialization (signatures cannot sign themselves). However, detached signatures are **included** in the transaction's hash/`TXID` for `Outpoint Transaction Hash`, block merkle trees, and for other P2P messages.

#### `SIGHASH_DETACHED` Algorithm & Flag

A new [signing serialization algorithm and flag](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/replay-protected-sighash.md#specification), `SIGHASH_DETACHED` is defined as `0x04`. When set, the signing serialization algorithm is equivalent to the [above version 3 transaction encoding](#transaction) truncated before all "detached" fields (immediately after `Locktime`).

To reference a detached signature from bytecode, the index of the referenced signature is encoded as a `Script Number` before the `SIGHASH_DETACHED` byte. This combined `signature reference` is pushed in place of a standard signature for the `OP_CHECKSIG`, `OP_CHECKSIGVERIFY`, `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` operations.

In each operation, when the final byte of the signature reference is found to contain the `SIGHASH_DETACHED` flag, the remainder of the reference is parsed as an index (`Script Number`), and the detached signature at that index is used for the remainder of the signature checking operation. Signature references must not set either `SIGHASH_FORKID` or `SIGHASH_ANYONECANPAY`, as neither flag modifies the `SIGHASH_DETACHED` algorithm. (Violations must cause evaluation to fail immediately.)

#### `Signature Reference` Test Vectors

Notably, because `SIGHASH_DETACHED` can be pushed using `OP_4`, this scheme allows the 0th detached signature to be referenced using a single byte.

| Detached Signature Index | Script Number | Encoded        | Disassembled                |
| ------------------------ | ------------- | -------------- | --------------------------- |
| `0`                      | `0x`          | `0x54`         | `OP_4`                      |
| `1`                      | `0x01`        | `0x020104`     | `OP_PUSHBYTES_2 0x0104`     |
| `2`                      | `0x02`        | `0x020204`     | `OP_PUSHBYTES_2 0x0204`     |
| `127`                    | `0x7f`        | `0x027f04`     | `OP_PUSHBYTES_2 0x7f04`     |
| `128`                    | `0x8000`      | `0x03800004`   | `OP_PUSHBYTES_3 0x800004`   |
| `32767`                  | `0xff7f`      | `0x03ff7f04`   | `OP_PUSHBYTES_3 0xff7f04`   |
| `32768`                  | `0x008000`    | `0x0400800004` | `OP_PUSHBYTES_4 0x00800004` |

### Detached Proofs

The v3 format includes a new transaction field called `Detached Proofs`.

| Field                     | v1/v2 Format | PMv3 Format                                               | Description                                |
| ------------------------- | ------------ | --------------------------------------------------------- | ------------------------------------------ |
| Unlocking Bytecode Length | N/A          | [RSN](#ranged-script-numbers-rsn) (1&#x2011;4&nbsp;bytes) | The byte-length of the unlocking bytecode. |
| Unlocking Bytecode        | N/A          | bytecode                                                  | The unlocking bytecode.                    |

Detached proofs allow the unlocking bytecode of a particular input to be provided as a hash in the transaction hash preimage (TXID/`Outpoint Transaction Hash` calculation).

> By "compressing" the unlocking bytecode of a detached proof into this hash, child transactions can efficiently inspect this transaction by pushing this transaction's TXID preimage, comparing the TXID preimage's double-SHA256 hash to one of the child's `Outpoint Transaction Hash`es, then manipulating the TXID preimage to verify required properties. By enabling child transactions to embed and thereby inspect their parent(s), complex cross-contract interactions can be developed.

Each detached proof is identified by a `Detached-Proof Hash`, the double-SHA256 hash of its serialization (`Unlocking Bytecode Length + Unlocking Bytecode`):

1. During transaction validation, if any inputs reference a detached proof, that detached proof must be present in the `Detached Proofs` transaction field.
2. If multiple detached proofs are included, they must be ordered canonically by detached-proof hash.
3. Each detached proof must be unique, though multiple inputs may refer to the same detached proof (e.g. when spending from two UTXOs with duplicated locking bytecode).
4. Each detached proof must be referenced by at least one input.

Detached proofs increase a transaction's size and cost by 32-bytes per input, so it is recommended that wallets avoid use of detached proofs unless required by a contract's design.

For a transaction to include the `Detached Proofs` field, it must include at least one detached signature. The `Detached Proof Count` may never be `0x00` (transactions without detached proofs must not encode the field).

> Note: detached proofs are "detached" from the inputs which use them, allowing their raw contents to be excluded from the transaction hash/TXID preimage. However, because inputs include the hash of each detached proof, **detached proofs still influence the resulting TXID**. If a detached proof changes, the hash referenced in any inputs will also change.

### Ranged Script Numbers (RSN)

`Ranged Script Number` (RSN) is a new variable-length integer format based on the encoding already used to represent integers in Bitcoin Cash VM bytecode ("Script Numbers" or `CScriptNum` in the Satoshi implementation). Because standard `Script Numbers` do not indicate their length, they cannot be safely parsed from a serialized format without modification.

`Ranged Script Numbers` borrow from the space-saving strategy of the current `VarInt` format: values from `0` (`0x00`) to `127` (`0x7f`) can be encoded in a single byte. To encode larger values, a prefix is added to indicate the length of the subsequent Script Number to read. Valid prefix values are `0x82` (`2`) through `0x87` (`7`). These are `-2` through `-7` in the VM respectively, so contracts can read the expected length using `<1> OP_SPLIT OP_SWAP OP_NEGATE`. (In the VM, `0x80` represents negative zero, and `0x81` represents `-1`. `Script Numbers` require 2 bytes to represent values larger than `127`, so the `0x81` prefix is never used.)

The prefix `0x87` allows for the reading of `2.1×10^15` (`0x870040075af07507`), the number of satoshis in 21 million bitcoin cash. This is the maximum value required for parsing RSN-encoded numbers in v3 transactions, but future transaction versions may allow for larger prefix values in conjunction with a move to fractional satoshis. (E.g. `0x89` (9 bytes) would enable encoding `2.1×10^15` in 1/1000ths of a satoshi.)

Though the existing `Script Number` format supports negative integers, `Ranged Script Numbers` must always be positive integers in the `PMv3` transaction format.

#### `RSN` Test Vectors

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

### Working Title: PMv3

To disambiguate this proposal from other version 3 transaction format proposals, it has the working title `PMv3` (a reference to prediction markets, one use-case it enables). If activated on the network, the format can be referenced as `v3`.

### Optional Detaching of Input Proofs

Detached proofs are a minimal strategy for allowing descendant transactions to include their parent transactions as unlocking data without being forced to include their full grandparent transactions (and so on). If the parent transaction chose to provide a hash for any of its unlocking scripts, child transactions can reduce the byte size of their own unlocking data by referencing only the hash. This enables "token covenants" which inductively prove their lineage without the proof size increasing each time the token is moved.

Other strategies, like [Segregated Witness](https://en.wikipedia.org/wiki/SegWit) could also enable this functionality, but detached proofs have the benefit of allowing transactions to opt-in on a per-input basis, avoiding unnecessary hash computations and saving block space. Detached proofs also avoid modifying the structure of the blockchain, retaining the cryptographic link of all signatures to the transactions they authorize.

### Use of VM Integer Format

The `Ranged Script Number` (RSN) format is chosen for its ease of conversion to and from standard Script Numbers within contracts. The VM Script Number format is already consensus-critical, so it is less disruptive to migrate transaction format integer types to the Script Number format than to migrate the reverse.

One alternative solution is to implement `OP_NUM2VARINT` and `OP_VARINT2NUM` operations. This would allow for slightly better compression of `VarInt`s between `128` and `252` (2 bytes), but in practice relatively few transactions use those numbers for input count, output count, or bytecode lengths. This inefficiency is also dwarfed by the space savings of compressing `Version` (3 bytes), `Outpoint Index` (approx. 3 bytes per input), and output `Value` (2-7 bytes per output).

Note, many other P2P protocol messages use other integer formats, and these remain unchanged. This includes the P2P protocol `tx` message header, which should continue to use the standard 24-byte header format.

### Use of Negative Script Number Range in RSN

The existing `Script Number` format (A.K.A. `CScriptNum`) has unusual properties: values from `0x00` to `0x7f` represent `0` through `127`, respectively, while `0x80` through `0xff` represent `-0` (negative zero) through `-127`. Because `Script Numbers` cannot represent values larger than `127` in a single byte, `128` is the ideal number for `Ranged Script Numbers` to also expand to multiple bytes. Because we are still left with the single-byte negative range, this range is ideal for indicating the `Ranged Script Numbers`' byte length.

One alternative is to use `0x00` or `0x80` as a special case to indicate that the following byte is a byte length indicator, e.g. `0x80020001` rather than `0x820001` (`256`). This would make parsing in VM contracts slightly simpler (no need for `OP_NEGATE`) at the cost of 1-byte per number larger than `127`. However, because only a small fraction of transactions on the network will require introspecting these numbers, this inconvenience is worth the additional byte saved per integer (~4 bytes for small transactions, with an additional `byte * (inputs + outputs)`).

Another notable benefit of the RSN format for satoshi values is the availability of a known-invalid range. Because prefixes larger than `0x87` (for 7 byte Script Numbers) are invalid, higher prefixes can safely be used by off-chain proof protocols, e.g. an output value of `0xff` is guaranteed to be rejected by the chain, and can therefore be used to safely prove ownership of any address (e.g. [Bitauth](https://github.com/bitauth/bitauth-cli), Proof-of-Reserves, etc.).

### Use of RSN for Satoshi Values

The `Ranged Script Number` format offers superior compression vs. the current `Uint64` format, offering significant per-output savings:

- Empty OP_RETURN outputs (`0` satoshis) save 7 bytes.
- Outputs smaller than `32,767` satoshis save 5 bytes.
- Outputs smaller than `8,388,607` satoshis (`0.08 bitcoin cash`) save 4 bytes.
- Outputs smaller than `2,147,483,647` satoshis (`~21.47 bitcoin cash`) save 3 bytes.
- Outputs smaller than `549,755,813,888` satoshis (`~5,497 bitcoin cash`) save 2 bytes.
- Outputs smaller than `140,737,488,355,328` satoshis (`~1,407,374 bitcoin cash`) save 1 byte.
- Larger outputs use 8 bytes (no savings or loss vs. `Uint64`).

### More efficient `OP_RETURN` outputs

As a coincidence of using the `Ranged Script Number` format, 0-satoshi `OP_RETURN` outputs in PMv3 are almost perfectly-efficient as a "data carrier" format. This better efficiency eliminates the value proposition of a new, additional "data carrier" field: retaining `OP_RETURN` outputs as "data carriers" is simpler, equally data-efficient, and already supports multiple outputs, VM introspection, variable signing serialization algorithms (`"SIGHASH_SINGLE"`, etc.), and other features of the existing VM system.

A possible improvement to the PMv3 format could save one final byte from these "data carrier" outputs by defining all future 0-satoshi outputs as having an "implied" `OP_RETURN` prefix, saving that unnecessary byte from the `Locking Bytecode`. (This change is not currently part of the PMv3 proposal.)

### Use of Uint32 for Locktime and Sequence Numbers

An earlier version of this specification proposed the use of the RSN format for all integers, including `Locktime` and `Sequence Number`. However, these fields commonly have values larger than `32767` (3-byte `RSN`) – the largest value for which RSN is more compressed than `Uint32` (4 bytes). Worse, the RSN format adds 1 byte for values larger than `16777214`, and up to 2 bytes for the maximum valid value of each field (`4294967295`). Based on current network usage, `Uint32` offers better median and average compression for these fields than `RSN`.

Additionally, Because `OP_BIN2NUM` and `OP_NUM2BIN` make conversion possible between `Script Numbers` and unsigned integers, keeping the `Uint32` format does not present a barrier for contract designers.

### Encoding/Order of Detached Signatures and Proofs

`Detached Signatures` and `Detached Proofs` are encoded to both eliminate parsing ambiguity and prevent all third-party malleability vectors.

While it is possible to include a valid `Detached Signatures` field without a `Detached Proofs` field, it is (by design) not possible to provide a `Detached Proofs` field without a `Detached Signatures` field:

- For v3 transactions which use **neither `Detached Signatures` nor `Detached Proofs`**, this design prevents third-parties from malleating transactions by maliciously detaching proofs because `Detached Proofs` cannot be specified without at least one detached signature.
- For v3 transactions which use **only `Detached Signatures`** (no `Detached Proofs`), the detached signatures themselves prevent all third-party malleability.
- For v3 transactions which **require `Detached Proofs`**, at least one `Detached Signature` must be included for the `Detached Proofs` field to be parsed correctly. This design protects users from inadvertently designing malleable wallet implementations.

### Replay Protection for Detached Signatures

No cross-chain replay protection mechanism for detached signatures is defined by this specification.

The existing `SIGHASH_FORKID` mechanism has failed to be useful during any notable splits since the BTC-BCH split, likely because it places the implementing side at a disadvantage. More reliable, mutually-advantageous replay protection strategies (like [AllowReplay](https://blog.bitjson.com/rfc-allowreplay-safer-splits-for-bitcoin-cash/)) now exist, and some form of [Gradually Activated Replay Protection](https://github.com/bitjson/allow-replay-spec#motivation) (GARP) is likely superior to any strategy attempting to apply `SIGHASH_FORKID` to detached signatures.

This proposal could be modified to pre-define an opt-in replay protection strategy for detached signatures: `SIGHASH_NOREPLAY` could be defined at `0x08`, such that `SIGHASH_NOREPLAY | SIGHASH_DETACHED` (`0x0c`/`12`) could still be referenced using a single-byte `OP_12`. Signature references with `SIGHASH_NOREPLAY` set could then prepend a variable [Network Fork ID](https://github.com/bitjson/allow-replay-spec#on-rotating-network-forkids) (which is only valid on the implementing chain) to the signing serialization (before `Version`).

Because this proposal could already serve as its own opt-in replay protection against non-upgraded chains, there's little purpose in adding a replay protection standard. However, future upgrades derived from PMv3-enabled chains may choose to include the above replay protection strategy.

### VM Support for Larger Script Numbers

This proposal does not currently include a specification for increasing the supported range of integers within the VM (currently: 32-bit signed integers). Because the proposed `Ranged Script Number` (RSN) format already uses larger numbers, these numbers could not be operated upon by the VM without a larger integer upgrade. As such, it's recommended that the allowable range of VM integers be expanded (to at least 64-bit signed integers) before or at the same time as the deployment of this specification.

### Upgrade Path for Signature Aggregation

Because detached signatures can be referenced from any input, it's possible for a future upgrade to deploy [signature aggregation](https://read.cash/@cpacia/new-musig-implementation-in-bchd-4ff95e28#towards-1-signature-per-transaction) with only a `SIGHASH_AGGREGATE` flag and algorithm – requiring no further transaction format changes. This strategy even allows many-to-many signature aggregation, where multiple clusters of signatures are aggregated into more than one aggregated, detached signatures.

### Upgrade Path for Fractional Satoshis

At a purchasing power of ~$100,000 (USD, 2020), Bitcoin Cash transaction fees will begin rising in real terms due to the indivisibility of satoshis; a 1 satoshi fee will be more expensive than today's median fees. By defining a variable-length integer format for satoshi values, this specification offers several upgrade paths for implementing fractional satoshis. A future transaction format could either migrate output values to a smaller base unit or implement a fractional satoshi scheme within the unused prefix byte range (`0x88` through `0xFE`) of output value [RSNs](#ranged-script-numbers-rsn).

## Implementations

Please see the following reference implementations for additional examples and test vectors:

[TODO: after initial public feedback]

## Stakeholders & Statements

[TODO]

## Copyright

This document is placed in the public domain.
