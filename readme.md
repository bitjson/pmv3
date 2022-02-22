# CHIP-2021-01-PMv3: Version 3 Transaction Format

        Title: Version 3 Transaction Format
        Proposal: PMv3
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Withdrawn
        Specification Version: 2.1.0
        Initial Publication Date: 2021-01-15
        Latest Revision Date: 2021-06-23

This proposal has been [withdrawn](https://bitcoincashresearch.org/t/chip-2021-01-pmv3-version-3-transaction-format/265/55?u=bitjson) in favor of [`CHIP-2022-02-CashTokens: Token Primitives for Bitcoin Cash`](https://github.com/bitjson/cashtokens).

## Summary

This proposal describes a version 3 transaction format for Bitcoin Cash which:

- Enables fixed-size inductive proofs and signing of transaction input contents.
- Consolidates transaction format integer encodings to be compatible with contracts.
- Reduces transaction sizes by eliminating wasted bytes.

## Deployment

Deployment of this specification is proposed for the May 2022 upgrade.

This proposal makes use of the `Ranged Script Number` (RSN) format, specified in [`CHIP: Ranged Script Numbers`](./ranged-script-numbers.md).

## Motivation

By addressing two issues with the transaction format, Bitcoin Cash contracts can support **cross-contract interfaces**, allowing contracts to interact with each other.

### Fixed-Size Inductive Proofs

Many token and covenant designs can be supported within the existing Bitcoin Cash VM design and opcode limits, but one unintentional limitation is resistant to workarounds: the [quadratically increasing size of inductive proofs](https://blog.bitjson.com/hashed-witnesses-build-decentralized-applications-on-bitcoin-cash/). With fixed-size inductive proofs, contract designers can create SPV-compatible (Simplified Payment Verification) and contract-validated tokens, token-issuing covenants, and covenants which interact with each other.

### Unified Integer Format

Bitcoin Cash transactions use 4 different formats for representing integers – `Uint32`, `Uint64`, `VarInt`, and `Script Numbers` (used within `Unlocking Bytecode` and `Locking Bytecode`). While Bitcoin Cash allows transaction introspection with `OP_CHECKDATASIG`, contracts which operate on integers appearing within the transaction format are forced to use complex workarounds to convert between integer formats.

This problem is partially alleviated by `OP_BIN2NUM` and `OP_NUM2BIN` – which allow conversion between VM integers (`Script Numbers`) and unsigned integers, but the `VarInt` format remains unaddressed. Additionally, existing transaction integer formats are inefficient: the specified [`Ranged Script Number`](./ranged-script-numbers.md) format saves ~12 bytes for small transactions and ~3 additional bytes per input/output.

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

### Protocol Complexity

New transaction formats can significantly increase protocol implementation costs in new libraries and programming languages.

**Mitigations**: this v3 format maintains the existing transaction structure, only modifying and adding fields as necessary. While all transaction parsing software will require upgrades to understand this new format, changes will typically be minimal and can reuse existing code.

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
| Version                  | Uint32 (4&nbsp;bytes)          | [RSN](./ranged-script-numbers.md) (1&nbsp;byte)           | The version of the transaction format (`0x03`).                                  |
| Input Count              | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The number of inputs in the transaction.                                         |
| Transaction Inputs       | [Inputs](#transaction-input)   | [Inputs](#transaction-input)                              | Each of the transaction’s inputs serialized in order.                            |
| Output Count             | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The number of outputs in the transaction.                                        |
| Transaction Outputs      | [Outputs](#transaction-output) | [Outputs](#transaction-output)                            | Each of the transaction’s outputs serialized in order.                           |
| Locktime                 | Uint32 (4&nbsp;bytes)          | Uint32 (4&nbsp;bytes)                                     | A time or block height at which the transaction is considered valid.             |
| **Detached Fields**      |
| Detached Signature Count | N/A                            | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The number of detached signatures in the transaction.                            |
| Detached Signatures      | N/A                            | [Detached Signatures](#detached-signatures)               | Each of the transaction's detached signatures serialized in order.               |
| Detached Proof Count     | N/A                            | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The number of detached proofs in the transaction.                                |
| Detached Proofs          | N/A                            | [Detached Proofs](#detached-proofs)                       | Each of the transaction's detached proofs serialized in canonical order by hash. |

When computing the preimage of detached signatures (`SIGHASH_DETACHED`), the encoded transaction is truncated after `Locktime`.

When computing the transaction's hash/identifier (`TXID`) for the block merkle tree, `Outpoint Transaction Hash`, and for all other P2P messages, the encoded transaction is truncated after `Detached Signatures`, if present. If the transaction includes no `Detached Signatures`, the TXID preimage is truncated after `Locktime`. (Detached proof integrity is guaranteed by the hash in any inputs referencing detached proofs.)

In all P2P protocol messages, the transaction is transmitted in its entirety, including any detached signatures and/or detached proofs, if present. Though the TXID preimage does not re-include the `Detached Signature Count`, `Detached Signatures`, `Detached Proof Count`, or `Detached Proofs` fields (which are committed to within `Transaction Inputs`), a transaction is considered invalid if it is missing any referenced `Detached Signatures` or `Detached Proofs`.

> Note: node implementations must never blacklist TXIDs for non-presence of detached signatures/proof(s), but should instead ban peers which repeatedly transmit malformed transactions (such as transactions from which detached signatures/proofs have been dropped).

### Transaction Input

The transaction input format is mostly unchanged, but most integers are replaced with [Ranged Script Numbers](./ranged-script-numbers.md). Additionally, a 32-byte, double-SHA256 `Detached-Proof Hash` may be provided in place of `Unlocking Bytecode` if `Unlocking Bytecode Length` is set to `0x00`.

Transactions opt-in to the use of detached proofs on a per-input basis, so any or all inputs within a transaction may reference detached proofs.

| Field                                     | v1/v2 Format                   | PMv3 Format                                               | Description                                                                                                                                                                               |
| ----------------------------------------- | ------------------------------ | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Outpoint Transaction Hash                 | hash (32&nbsp;bytes)           | hash (32&nbsp;bytes)                                      | The hash of the transaction containing the output being spent.                                                                                                                            |
| Outpoint Index                            | Uint32 (4&nbsp;bytes)          | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The zero-based index of the output being spent from the previous transaction.                                                                                                             |
| Unlocking Bytecode Length                 | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The size of the unlocking script in bytes. If set to `0x00`, the next item is a detached-proof hash.                                                                                      |
| Unlocking Bytecode OR Detached-Proof Hash | bytecode                       | bytecode OR hash                                          | If `Unlocking Bytecode Length` is `0x00`, a 32-byte, double-SHA256 [detached-proof hash](#detached-proofs). Otherwise, the unlocking bytecode.                                            |
| Sequence Number                           | Uint32 (4&nbsp;bytes)          | Uint32 (4&nbsp;bytes)                                     | As of [BIP68](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0068.mediawiki), the sequence number is interpreted as a relative locktime for the input. |

### Transaction Output

The transaction output format is mostly unchanged. Both integer formats are replaced with [Ranged Script Numbers](./ranged-script-numbers.md).

| Field                   | v1/v2 Format                   | PMv3 Format                                               | Description                               |
| ----------------------- | ------------------------------ | --------------------------------------------------------- | ----------------------------------------- |
| Value                   | Uint64 (8&nbsp;bytes)          | [RSN](./ranged-script-numbers.md) (1&#x2011;8&nbsp;bytes) | The number of satoshis to be transferred. |
| Locking Bytecode Length | VarInt (1&#x2011;4&nbsp;bytes) | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The byte-length of the locking bytecode.  |
| Locking Bytecode        | bytecode                       | bytecode                                                  | The locking bytecode.                     |

### Detached Signatures

The v3 format includes a new transaction field called `Detached Signatures`.

| Field                     | v1/v2 Format | PMv3 Format                                               | Description                                                                                                                                                                                     |
| ------------------------- | ------------ | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Detached Signature Length | N/A          | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The byte-length of the following detached signature field.                                                                                                                                      |
| Detached Signature        | N/A          | bytecode                                                  | The detached signature (either [BIP66](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0066.mediawiki) ECDSA or Schnorr) including the sighash byte (`0x40`). |

Detached signatures are signatures which use the [`SIGHASH_DETACHED` signing serialization algorithm](#sighash_detached-algorithm) (which is based on the version 3 transaction encoding). The `Detached Signatures` field is an ordered list of these signatures.

By committing to the entire encoded transaction – including the contents of all inputs – **detached signatures prevent all third-party malleability**. Applications which use detached proofs should use detached signatures to prevent third parties from manipulating inputs to detach or re-attach proofs. Other applications should also use detached signatures to prevent malleability or reduce transaction sizes.

Detached signatures are referenced by their index from unlocking bytecode using `signature references` (described below).

Detached signatures must not be duplicated (as this would introduce a malleability vector); duplication in the `Detached Signatures` field renders a transaction invalid.

> Note: its possible for more than one input to reference the same signature.

Use of detached signatures is optional. The encoding of transactions which include no detached signatures must end with the `Locktime` field; the `Detached Signature Count` must never be `0x00` (transactions without detached signatures must not encode the field).

> Note: detached signatures are "detached" from the inputs which use them, allowing them to be excluded from their own signing serialization (signatures cannot sign themselves). However, detached signatures are **included** in the transaction's hash/`TXID` for `Outpoint Transaction Hash`, block merkle trees, and for other P2P messages.

#### `SIGHASH_DETACHED` Algorithm

A new [signing serialization algorithm](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/replay-protected-sighash.md#specification), `SIGHASH_DETACHED`, is used for all detached signatures. The algorithm is equivalent to the [above version 3 transaction encoding](#transaction) prefixed with the Fork ID and truncated before all "detached" fields (immediately after `Locktime`):

| Field               | Format                                                    | Description                                                          |
| ------------------- | --------------------------------------------------------- | -------------------------------------------------------------------- |
| Fork ID             | Fork ID (3&nbsp;bytes)                                    | On BCH, `0x000000`.                                                  |
| Version             | [RSN](./ranged-script-numbers.md) (1&nbsp;byte)           | The version of the transaction format (`0x03`).                      |
| Input Count         | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The number of inputs in the transaction.                             |
| Transaction Inputs  | [Inputs](#transaction-input)                              | Each of the transaction’s inputs serialized in order.                |
| Output Count        | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The number of outputs in the transaction.                            |
| Transaction Outputs | [Outputs](#transaction-output)                            | Each of the transaction’s outputs serialized in order.               |
| Locktime            | Uint32 (4&nbsp;bytes)                                     | A time or block height at which the transaction is considered valid. |

To reference a detached signature from bytecode, the index of the referenced signature is encoded as a `Script Number`. This `signature reference` is pushed in place of a standard signature for the `OP_CHECKSIG`, `OP_CHECKSIGVERIFY`, `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` operations.

In each signature-checking operation, when the signature popped from the stack is found to have a length less than or equal to `2` bytes, the reference is parsed as an index (`Script Number`), and the detached signature at that index is used for the remainder of the signature checking operation. (This allows for indexes up to `32767`, far beyond existing limits to transaction size.)

All detached signatures must set `SIGHASH_FORKID` (`0x40`), and no other flags are currently permitted.

> Note, the `SIGHASH_DETACHED` algorithm is not represented by a sighash byte, as it is currently the only valid algorithm for detached signatures. Future upgrades may specify other algorithms and valid sighash byte values for detached signatures.

Each detached signature must be referenced by at least one input.

#### `Signature Reference` Test Vectors

Detached signatures are referenced by pushing a valid detached signature index (as a `Script Number`) in place of a signature. Because the VM includes single-byte operations from `OP_0` to `OP_16`, up to 16 detached signatures can be referenced using a single byte.

| Encoded      | Disassembled              | Script Number | Detached Signature Index |
| ------------ | ------------------------- | ------------- | ------------------------ |
| `0x00`       | `OP_0`                    | `0x`          | `0`                      |
| `0x51`       | `OP_1`                    | `0x01`        | `1`                      |
| `0x52`       | `OP_2`                    | `0x02`        | `2`                      |
| `0x60`       | `OP_16`                   | `0x10`        | `16`                     |
| `0x0111`     | `OP_PUSHBYTES_1 0x11`     | `0x11`        | `17`                     |
| `0x017f`     | `OP_PUSHBYTES_1 0x7f`     | `0x7f`        | `127`                    |
| `0x028000`   | `OP_PUSHBYTES_2 0x8000`   | `0x8000`      | `128`                    |
| `0x03ff7f`   | `OP_PUSHBYTES_2 0xff7f`   | `0xff7f`      | `32767`                  |
| `0x04008000` | `OP_PUSHBYTES_3 0x008000` | `0x008000`    | Invalid                  |

### Detached Proofs

The v3 format includes a new transaction field called `Detached Proofs`.

| Field                     | v1/v2 Format | PMv3 Format                                               | Description                                |
| ------------------------- | ------------ | --------------------------------------------------------- | ------------------------------------------ |
| Unlocking Bytecode Length | N/A          | [RSN](./ranged-script-numbers.md) (1&#x2011;4&nbsp;bytes) | The byte-length of the unlocking bytecode. |
| Unlocking Bytecode        | N/A          | bytecode                                                  | The unlocking bytecode.                    |

Detached proofs allow the unlocking bytecode of a particular input to be provided as a hash in the transaction hash preimage (TXID/`Outpoint Transaction Hash` calculation).

> By "compressing" the unlocking bytecode of a detached proof into this hash, child transactions can efficiently inspect this transaction by pushing this transaction's TXID preimage, comparing the TXID preimage's double-SHA256 hash to one of the child's `Outpoint Transaction Hash`es, then manipulating the TXID preimage to verify required properties. By enabling child transactions to embed and thereby inspect their parent(s), complex cross-contract interactions can be developed.

Each detached proof is identified by a `Detached-Proof Hash`, the double-SHA256 hash of its `Unlocking Bytecode`. (The `Unlocking Bytecode Length` is not included in the preimage.)

1. During transaction validation, if any inputs reference a detached proof, that detached proof must be present in the `Detached Proofs` transaction field.
2. If multiple detached proofs are included, they must be ordered canonically by detached-proof hash.
3. Each detached proof must be unique, though multiple inputs may refer to the same detached proof (e.g. when spending from two UTXOs with duplicated locking bytecode).
4. Each detached proof must be referenced by at least one input.

Detached proofs increase a transaction's size and cost by 32-bytes per input, so it is recommended that wallets avoid use of detached proofs unless required by a contract's design.

For a transaction to include the `Detached Proofs` field, it must include at least one detached signature. The `Detached Proof Count` must never be `0x00` (transactions without detached proofs must not encode the field).

> Note: detached proofs are "detached" from the inputs which use them, allowing their raw contents to be excluded from the transaction hash/TXID preimage. However, because inputs include the hash of each detached proof, **detached proofs still influence the resulting TXID**. If a detached proof changes, the hash referenced in any inputs will also change.

## Rationale

This section documents design decisions made in this specification.

### Working Title: PMv3

To disambiguate this proposal from other version 3 transaction format proposals, it has the working title `PMv3` (a reference to prediction markets, one use-case it enables). If activated on the network, the format can be referenced as `v3`.

### Optional Detaching of Input Proofs

Detached proofs are a minimal strategy for allowing descendant transactions to include their parent transactions as unlocking data without being forced to include their full grandparent transactions (and so on). If the parent transaction chose to provide a hash for any of its unlocking scripts, child transactions can reduce the byte size of their own unlocking data by referencing only the hash. This enables "token covenants" which inductively prove their lineage without the proof size increasing each time the token is moved.

Other strategies, like [Segregated Witness](https://en.wikipedia.org/wiki/SegWit) (SegWit) could also enable this functionality, but detached proofs have the benefit of allowing transactions to opt-in on a per-input basis, avoiding unnecessary hash computations and saving block space. Detached proofs also avoid modifying the structure of the blockchain, retaining the cryptographic link of all signatures to the transactions they authorize.

### One TXID Per Transaction

Past upgrades and proposals have attempted to extend or modify the transaction format without requiring outdated software to be updated; this strategy is often described as a "soft fork" (e.g. [P2SH/BIP16](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0016.mediawiki#backwards-compatibility) and SegWit). While soft forks can be expedient for forcing major protocol changes on unaware or indifferent user bases, soft forks create a risk of payment fraud against un-upgraded users, disenfranchise users unaware of the upgrade, and permanently saddle the implementing chain with technical debt (via increased protocol complexity)<sup>1</sup>.

A version 3 transaction format could also be implemented as a soft fork by designing a dual-TXID system, where the Transaction Identifier (TXID) is only used in communication with un-upgraded nodes, and a "newTXID" is only used by nodes aware of the upgrade. However, this proposal considers such a dual-TXID system to be an unacceptable long-term burden on the network. Instead, this proposal accepts the possibility of a slower deployment as ecosystem participants update their systems to support the simpler, "hard fork" implementation.

> A thorough [analysis of upgrade requirements for many popular wallets and applications](https://bitcoincashresearch.org/t/chip-2021-01-pmv3-version-3-transaction-format/265/48#should-we-deploy-a-pmv3-lite-no-integer-changes-7) is also available on Bitcoin Cash Research.

1. Hearn, M. (2015, August 12). [On consensus and forks.](https://archive.vn/C3T3v)

### Use of RSN Format

The [`Ranged Script Number` (RSN)](./ranged-script-numbers.md) format is chosen for its ease of conversion to and from standard Script Numbers within contracts. The VM Script Number format is already consensus-critical, so it is less disruptive to migrate transaction format integer types to the Script Number format than to migrate the reverse.

One alternative solution is to implement `OP_NUM2VARINT` and `OP_VARINT2NUM` operations. While this would allow for slightly better compression of `VarInt`s between `128` and `252` (2 bytes), in practice, relatively few transactions use those numbers for input count, output count, or bytecode lengths. (This inefficiency is also dwarfed by the space savings of compressing `Version` [3 bytes], `Outpoint Index` [approx. 3 bytes per input], and output `Value` [2-7 bytes per output].)

An opcode-only solution would increase the size and complexity of contracts (which would still need [strategies for splitting/parsing VarInt values from stack items](./ranged-script-numbers.md#usage-comparison-vs-varint)), complicate the VM model (by introducing a completely new number "type" requiring bitwise manipulation), and require future contract compilers and wallets to include software support for the VarInt format (which is otherwise unnecessary for new wallet applications).

Because contract compilers and wallets are already required to support the `Script Number` format when creating and interacting with complex contracts, implementation of the `Ranged Script Number` format is less burdensome for these applications. (Of course, less advanced wallets can continue to use v1/v2 transactions.)

> Note, many other P2P protocol messages use other integer formats, and these remain unchanged. This includes the P2P protocol `tx` message header, which will continue to use the standard 24-byte header format.

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

As a coincidence of using the `Ranged Script Number` format, 0-satoshi `OP_RETURN` outputs in PMv3 are almost perfectly-efficient as a "data carrier" format. This better efficiency eliminates the value proposition of a new, additional "data carrier" field: retaining `OP_RETURN` outputs as "data carriers" is simpler, equally data-efficient, and already supports multiple outputs, VM introspection, variable signing serialization algorithms ("`SIGHASH_SINGLE`", "`SIGHASH_ANYONECANPAY`", etc.), and other features of the existing VM system.

A possible improvement to the PMv3 format could save one final byte from these "data carrier" outputs by defining all future 0-satoshi outputs as having an "implied" `OP_RETURN` prefix, saving that unnecessary byte from the `Locking Bytecode`. (This change is not currently part of the PMv3 proposal.)

### Use of Uint32 for Locktime and Sequence Numbers

An earlier version of this specification proposed the use of the RSN format for all integers, including `Locktime` and `Sequence Number`. Because these fields affect transaction validity over time, modifications which eliminate the current fixed-size property of each field could negatively interact with mining incentives, [reducing security against "fee sniping" attacks](https://bitcoincashresearch.org/t/chip-2021-01-pmv3-version-3-transaction-format/265/48#how-transaction-formats-can-interact-with-mining-incentives-4).

Additionally, the `Locktime` and `Sequence Number` fields commonly have values larger than `32767` (3-byte `RSN`) – the largest value for which RSN is more compressed than `Uint32` (4 bytes). Worse, the RSN format adds 1 byte for values larger than `16777214`, and up to 2 bytes for the relatively-common, maximum valid value of each field (`4294967295`). As such, based on current network usage, `Uint32` offers better median and average compression for these fields than `RSN`.

Notably, Because `OP_BIN2NUM` and `OP_NUM2BIN` make conversion possible between `Script Numbers` and unsigned integers, keeping the `Uint32` format does not present a barrier for contract designers.

### Encoding/Order of Detached Signatures and Proofs

`Detached Signatures` and `Detached Proofs` are encoded to both eliminate parsing ambiguity and prevent all third-party malleability vectors.

While it is possible to include a valid `Detached Signatures` field without a `Detached Proofs` field, it is (by design) not possible to provide a `Detached Proofs` field without a `Detached Signatures` field:

- For v3 transactions which use **neither `Detached Signatures` nor `Detached Proofs`**, this design prevents third-parties from malleating transactions by maliciously detaching proofs because `Detached Proofs` cannot be specified without at least one detached signature.
- For v3 transactions which use **only `Detached Signatures`** (no `Detached Proofs`), the detached signatures themselves prevent all third-party malleability.
- For v3 transactions which **require `Detached Proofs`**, at least one `Detached Signature` must be included for the `Detached Proofs` field to be parsed correctly. This design protects users from inadvertently designing malleable wallet implementations. (Any transaction which uses detached proofs must use at least one detached signature to prevent third-party malleability.)

Finally, this design simplifies protocol implementation, allowing the detached fields to be serially parsed (without multivariate ordering handling).

### Replay Protection for Detached Signatures

This proposal maintains the existing "Fork ID" mechanism in the new `SIGHASH_DETACHED` algorithm. While this existing replay-protection mechanism has failed consistently (it has been disabled during every notable split since the BTC-BCH split – likely because [it places the implementing side at a disadvantage](https://blog.bitjson.com/rfc-allowreplay-safer-splits-for-bitcoin-cash/)), consistency with the existing signature types will allow future replay-protection upgrades to address detached signatures in the same way as all other existing signature types.

More reliable, mutually-advantageous replay protection strategies now exist, and future upgrades should consider some form of [Gradually Activated Replay Protection](https://github.com/bitjson/allow-replay-spec#motivation) (GARP) like [AllowReplay](https://blog.bitjson.com/rfc-allowreplay-safer-splits-for-bitcoin-cash/). However, because this proposal already serves as its own opt-in replay protection against non-upgraded chains, no other replay-protection strategy is necessary for this proposal.

### VM Support for Larger Script Numbers

This proposal does not currently include a specification for increasing the supported range of integers within the VM (currently: 32-bit signed integers). Because the proposed `Ranged Script Number` (RSN) format already uses larger numbers, these numbers could not be operated upon by the VM without a larger integer upgrade. As such, it's recommended that the allowable range of VM integers be expanded (to at least 64-bit signed integers) before or at the same time as the deployment of this specification (e.g. [`CHIP 2021-03 Bigger Script Integers`](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md)).

### Upgrade Path for Signature Aggregation

Because detached signatures can be referenced from any input, it's possible for a future upgrade to deploy [signature aggregation](https://read.cash/@cpacia/new-musig-implementation-in-bchd-4ff95e28#towards-1-signature-per-transaction) with only a `SIGHASH_AGGREGATE` flag (e.g. `0x04`) and algorithm – requiring no further transaction format changes. This strategy even allows many-to-many signature aggregation, where multiple clusters of signatures are aggregated into more than one aggregated, detached signatures.

## Implementations

Please see the following reference implementations for additional examples and test vectors:

_(in progress)_

## Stakeholders & Statements

_(in progress)_

## Feedback & Reviews

- [`CHIP 2021-01 PMv3: Version 3 Transaction Format` – January 19, 2021 | bitcoincashresearch.org](https://bitcoincashresearch.org/t/chip-2021-01-pmv3-version-3-transaction-format/265)

## Changelog

This section summarizes the evolution of this document.

- **v2.1.0 - 2021-6-23** (current)
  - Extracted `Ranged Script Number` (RSN) format into an independent CHIP: [`CHIP-2021-01-RSN`](./ranged-script-numbers.md) ([#4](https://github.com/bitjson/pmv3/issues/4))
  - Removed length from detached proof preimage ([#5](https://github.com/bitjson/pmv3/pull/5))
  - Moved detached signature sighash byte out of inputs, included Fork ID mechanism in `SIGHASH_DETACHED` algorithm ([#6](https://github.com/bitjson/pmv3/issues/6))
- **v2.0.0 - 2021-4-29** ([`e999e719`](https://github.com/bitjson/pmv3/blob/e999e7190f7fd241d1995ebd573ff4cf236c9e23/readme.md))
  - `Detached Signatures` comprehensively solve third-party malleability and enable signature aggregation ([post](https://blog.bitjson.com/pmv3-chip-revised-version-3-transactions/))
  - Renamed `Hashed Witnesses` to `Detached Proofs`
  - Migrated to CHIP format: `CHIP-2021-01-PMv3`
- **v1.1.0 - 2021-1-19** ([`93e691df`](https://github.com/bitjson/pmv3/blob/93e691df41b978e60b77ed037ad553a3d13fdf38/readme.md))
  - Highlight improved OP_RETURN efficiency
- **v1.0.0 – 2021-1-15** ([`a0771cef`](https://github.com/bitjson/pmv3/blob/a0771cef0f515c8e9187dc987edaf500348a93e4/readme.md))
  - Initial publication

## Copyright

This document is placed in the public domain.
