```
bLIP: 29
Title: Taproot Asset Channels
Status: Active
Author: Olaoluwa Osuntokun <laolu32@gmail.com>
Created: 2023-07-14
License: CC0
```

# Table of Contents

  * [Abstract](#abstract)
  * [Copyright](#copyright)
  * [Motivation](#motivation)
  * [Preliminaries](#preliminaries)
    * [Merkle-Sum Sparse Merkle Trees](#merkle-sum-sparse-merkle-trees)
    * [Taproot Assets Protocol](#taproot-assets-protocol)
      * [Asset Creation & Asset IDs](#asset-creation-&-asset-ids)
      * [Taproot Assets VM](#taproot-assets-vm)
      * [Asset Splits](#asset-splits)
  * [Design Overview](#design-overview)
    * [Anchoring TAP Assets in the Funding Transaction](#anchoring-tap-assets-in-the-funding-transaction)
    * [Modified Taproot Asset Scripts](#modified-taproot-asset-scripts)
    * [Last Mile RFQ Rate Negotiation](#last-mile-rfq-rate-negotiation)
    * [Decoupled Multi-Hop Multi-Asset Payments](#decoupled-multi-hop-multi-asset-payments)
  * [Specification](#specification)
    * [Feature Bits](#feature-bits)
    * [New TLV Types](#new-tlv-types)
      * [tap_partial_signature_with_nonce](#tap_partial_signature_with_nonce)
      * [tap_partial_signature](#tap_partial_signature)
      * [tap_htlc_info](#tap_htlc_info)
      * [tap_htlc_sigs](#tap_htlc_sigs)
      * [tap_onion_rfq_id](#tap_onion_rfq_id)
    * [Channel Funding](#channel-funding)
    * [`tx_asset_proof` Message](#`tx_asset_proof`-message)
      * [`open_channel` Extensions](#`open_channel`-extensions)
      * [`accept_channel` Extensions](#`accept_channel`-extensions)
      * [`funding_created` Extensions](#`funding_created`-extensions)
      * [`funding_accepted` Extensions](#`funding_accepted`-extensions)
      * [Funding Output Construction](#funding-output-construction)
    * [Cooperative Closure](#cooperative-closure)
      * [`closing_signed` Extensions](#`closing_signed`-extensions)
      * [Requirements](#requirements)
    * [Channel Operation](#channel-operation)
      * [`update_add_htlc` Extensions](#`update_add_htlc`-extensions)
      * [`commitment_signed` Extensions](#`commitment_signed`-extensions)
    * [Commitment Transaction Construction](#commitment-transaction-construction)
      * [To Local Outputs](#to-local-outputs)
      * [To Remote Outputs](#to-remote-outputs)
      * [Anchor Outputs](#anchor-outputs)
    * [HTLC Scripts & Transactions](#htlc-scripts-&-transactions)
      * [Offered HTLCs](#offered-htlcs)
      * [Accepted HTLCs](#accepted-htlcs)
        * [HTLC-Success Transactions](#htlc-success-transactions)
        * [HTLC-Timeout Transactions](#htlc-timeout-transactions)
    * [Last Mile Routing](#last-mile-routing)
      * [RFQ Negotiation](#rfq-negotiation)
        * [Request For Quote (`tap_rfq`)](#request-for-quote-(`tap_rfq`))
          * [Requirements](#requirements)
      * [Request For Quote Response (`tap_rfq_accept`) + (`tap_rfq_reject`)](#request-for-quote-response-(`tap_rfq_accept`)-+-(`tap_rfq_reject`))
      * [Accepting Quotes  (`tap_rfq_accept`)](#accepting-quotes--(`tap_rfq_accept`))
        * [Requirements](#requirements)
      * [Rejecting Quotes  (`tap_rfq_reject`)](#rejecting-quotes--(`tap_rfq_reject`))
        * [Requirements](#requirements)
      * [First Hop TAP HTLC Onion Processing](#first-hop-tap-htlc-onion-processing)
      * [Last Hop TAP HTLC Onion Processing](#last-hop-tap-htlc-onion-processing)
      * [Invoice Format](#invoice-format)
  * [Universality](#universality)
  * [Backwards Compatibility](#backwards-compatibility)
  * [Reference Implementation](#reference-implementation)



## Abstract

This document describes a variant on the modern "simple taproot channels" that
also supports holding an transferring assets created by the Taproot Assets
Protocol. As Taproot Assets are built on top of the taproot itself, from the
PoV of the taproot channel format, Taproot Assets manifests entirely as an
extra tapscript sibling placed in the tapscript tee of relevant outputs. A set
of asset-specific balances (in the form of taproot asset tree commitments) are
maintained as an overlay layer on top of the normal initiator+responder
balances of Lightning channels. For channel state transitions and eventual
on-chain contract claims, in addition to normal taproot witnesses, a set of
taproot asset level witnesses are also exchanged, encumbered by a nested
iteration of the current Tapscript VM, the Taproot Assets VM.

In order to facilitate multi-hop payments of the existing LN using Taproot
Assets edge liquidity, an RFQ (Request For Quote) last-mile negotiation scheme
is used to lock in an exchange rate for both incoming and outgoing payments by
liquidity providers. Tendered quotes `(asset_id, volume, price)`are identified
by a cryptographic hash and scid-like sequence number, and ephemerally expire
in order to reduce exchange rate risk. The existing BOLT 11 invoice format is
used verbatim, in a manner that allows a receiver to accept an taproot asset
without burdening the sender with up to date knowledge of exchange rates.

## Copyright

This bLIP is licensed under the CC0 license.

## Motivation

The Lightning Network is the world's first open, fully-collateralized
decentralized payment system.  The Lightning system is globally adopted around
the world and is used for: machine to machine payments, remittances,
e-commerce, donations, tipping and more. As the LN is built on top of the
Bitcoin blockchain, it uses bitcoin as its primary unit of account to send and
receive payments. By enabling assets created by the Taproot Assets protocol to
be sent and received over Lightning utilizing the _existing_ bitcoin backbone
liquidity of the network, we enable a wave of additional use case and activity
to leverage the network, which may have been previously hindered by the lack of
another option for a unit for the edge of the Lightning Network.

Enabling assets created by the Taproot Assets protocol to be sent over the
Lightning Network introduces yet another demand fly wheel, which may serve to
invigorate the Lightning Network by increasing the demand for transactional
volume, thereby increasing investment in the system, and providing active
network routers with a new revenue source. One such potential asset includes
stablecoins, which at the time of writing have a market cap of nearly $100
billion. By enabling stablecoins to be sent and received at the edge of the
network, the utility of the Lightning Network increase, as LN effectively
becomes the monetary backbone of the digital age with participants transacting
in arbitrary asset at the edges, transported through the network by the
bitcoin liquidity backbone.

## Preliminaries

In this section we lay out some preliminary concepts, protocol flows, and
notation that'll be used later in the core channel specification.


### Merkle-Sum Sparse Merkle Trees

A sparse merkle tree is an merkalized authenticated data structure that maps a
`256-bit` value to an arbitrary set of bytes. A merkle-sum tree creates a
merkle-set by simulating a merkle tree of depth `N` (256 in this case). As such
a tree has 256 leaves, the entire range of the `sha256` hash function can be
uniquely located amongst the set of leaves. The tree initially starts as an
empty tree with a known "empty hash" value for the lowest level. Given this
value, the empty hash value for each level of the tree can be pre-computed and
known. Levering this feature, succinct proof of exclusion for a certain key can
be produced by showing that the designate leaf is actually the empty hash.
Compared to a normal merkle tree, an SMT is able to achieve succinct proofs of
exclusion, as the set of leaves is already pre-sorted, and the tree requires
no rebalancing.

A merkle-sum sparse merkle tree builds on the SMT data structure with the
addition of augmented leaves and branches. In addition to a key-location, and a
value, each leaf also contains a _sum value_. When creating the parent of two
leaf nodes, the _sum_ of the accumulator values for both leaf node is also
included in the hash digest. This enables a prover to prove to a verify
attributes related to the value of a leaf, and also the sum of all the leaves
in the entire tree. Such attributes are useful when proving invariants such as
supply constraints if one models a set of assets as leaves within the tree.

### Taproot Assets Protocol

The Taproot Assets Protocol is an overlay protocol for representing arbitrary
assets on Bitcoin which leverages the existing commitment space in the
Tapscript tree of Pay-To-Taproot outputs to store structured asset data.
Structured asset data is stored as a TLV encoded data structure within the
leaves of an MS-SMT tree. The sum value for a leaf is the total amount of
assets held, with the root value of the tree committing to the sum of all
assets held by an output. A single taproot output may hold up to `2^256-1`
individual assets. 

The Taproot Assets commitment is stored in _unique_ location within the
tapscript tree for a given output in a special tapscript leaf with the
following value:
```
tagged_hash("TapLeaf", leaf_version || taproot_asset_marker || taproot_asset_version || asset_tree_root)
```

#### Asset Creation & Asset IDs

Asset IDs in the protocol are uniquely generated by leveraging the chain
invariant enforced by BIP 34 wherein a `txid` account be repeated multiple
times in a valid chain. For a given asset, an `asset_id` is generated as: 
```
sha256(genesis_outpoint || sha256(asset_tag) || asset_meta_hash || output_index || asset_type)
```

With the latter four values arbitrarily determined by the creator of a given
asset. As a result of this derivation, all assets within the protocol gain a
globally unique identifier that can be used to easily identify a given asset.

#### Taproot Assets VM

The asset TLV of a taproot asset includes a special `script_key` field. This
`script_key` is derived according to the rules defined in BIP-341 and 342. In
other woods, the initial version of the Taproot Assets VM is actually a
_nested_ instantiation of the _Tapscript_ VM. In order to spend an asset, a
holder generates a valid witness first mapping the set of input+output asset
UTXOs into a special virtual transaction format, with the witness being
generated as one would for a normal tapscript state transition.

Due to the above design, at a base level, any script that can be expressed in
tapscript can also be expressed by the Taproot Assets VM. This enables a 1:1
mapping between scripts at the Bitcoin level (chain anchor) and the Taproot
Assets level. We'll leverage this feature to create enforced asset funding
outputs, commitment outputs, and also HTLCs. Future versions of the VM will
also permit additional expressibility in the form of more advanced contracting
primitives such as covenants, and STARKs.

When combined with the Taproot Assets Tree format, the system allows an
affectively unbounded amount of assets to be transferred in a single transaction.

#### Asset Splits

Similar to normal Bitcoin UTXOs, UTXOs for Taproot Assets can also be split and
merged. When splitting an asset, in order to ensure that the supply of an asset
is conserved (no inflation occurred) any resulting UTXOs created from the
source UTXO are inserted into a special MS-SMT tree dubbed a
`split_commitment`. A "root" asset is designated which holds the root hash of
the `split_commitment`. Any splits that were created from the root asset carry
a special merkle proof that proves that no assets were inflated.

This design enables a verifier to easily verify that an asset's supply wasn't
inflated by verifying that the `(asset_id, amt)` of the input asset is exactly
equal to the `(asset_id, amt)` of the `split_commitment` held by the root
asset.

## Design Overview

### Anchoring TAP Assets in the Funding Transaction

As the Taproot Assets protocol is a overlay system on top of the Taproot script
template, the impact of Taproot Assets on the Simple Taproot Assets channel
type is minimal. At a high level, one can bind Taproot Assets to a given P2TR
output simply by including the asset tree root commitment within the committed
tapscript tree. From here, the channel type simply needs to ensure that any
subsequent spends from that output includes a new valid commitment (including
valid witness data) in the outputs.

For the funding transaction, the musig2 mapping is inherited onto the TAP
layer: the same set of musig2 keys are re-used, with the sighash being signed
one that's derived from the TAP virtual transaction for the commitment
transaction.

### Modified Taproot Asset Scripts

For HTLCs, a new tapscript sibling is introduced into the tapscript tree for
all outputs. This tapscript sibling commits to a corresponding TAP level HTLC
that replicates the same the HTLC script onto the TAP layer. By replicating the
HTLC script onto the TAP layer, we ensure that the TAP assets can only be
unilaterally moved under the same conditions as normal HTLCs. This contract
mirroring also applies to the second level HTLC construct as well.

Similarly, the set of revocation scripts and semantic are also lifted up to the
TAP layer. In the case of a breach transaction, then TAP assets into addition
to BTC are also forfeited to the defender.

Anchor outputs are the only output type that doesn't inherit any semantics from
the TAP layer. They operate as normal and allow CPFP fee bumping, just like
with the normal zero fee anchors channel type.

Given that the TAP layer is anchored on the Bitcoin layer, BTC is used as
normal to pay for all on-chain fees. For channels that are intended to be used
primarily with TAP assets, then a sufficient amount of BTC must be funded in
order to allow for the commitment transaction to enter the mempool, respecting
the current min mempool fee. For pure asset HTLC outputs, a static HTLC size of
the negotiated dust limit is used for asset amounts that manifest above the
current dust limit.

// TODO(roasbeef): inherit distinct dust limit here?

### Last Mile RFQ Rate Negotiation 

For the multi-hop layer, the existing LN invoicing scheme is mostly unchanged.
In order to cross pairs for a route at the outgoing link, incoming link, or
both an RFQ-based negotiation scheme is used between the initiating/resewing
node and their direct liquidity proving peer. For the outgoing link, the
accepted RFQ quote is identified by the hash of the signed RFQ quote. For the
final receiver hop, the utilized quota in the quote is represented by a special
scid, that resembles the existing scid alias feature. 

### Decoupled Multi-Hop Multi-Asset Payments

By piggy backing on the existing onion hop payload space, we enable upgraded
senders to send BTC with the receiver receiving their asset of choice without
burdening them with exchange rate information. The invoice only requests a BTC
amount, which is derived via the latest tendered quote between the resewing
node and their liquidity provider. If the sender is sending with a TAP asset of
their own, then the payload the construct must respect the latest accepted
quote by their outgoing liquidity provider to cross into the BTC backbone of the
network.

The end result is a new optional overlay layer that's anchored to the greater
Lightning Network by edge node liquidity. This new system enables a receiver to
accept any asset of their choosing, with the sender either sending BTC or their
TAP asset of choice. From the PoV of the sender the additional exchange fees
simply net out as an aggregate transaction fee. Most modern wallets will
implement a strict fee limit (say of 0.5%) that can be used to allow a user to
express their fee tolerance, and thus what an acceptable end to end exchange
rate will be. Existing routers in the Bitcoin backbone of the network do not
need to update, and instead will benefit due to the increased routing volume
and routing fee revenue.


## Specification

### Feature Bits 

* new feature bit and channel tyep
  * both even
  * new chan type TLV and node ann level feature bit as well (can seek out
    those to open asset chans with)

Inheriting the structure put forth in BOLT 9, we define a new feature bit to be
placed in the `init` message, `node_announcement` and also
`channel_announcement`. We opt to also place the bit in the
`channel_announcement` so taproot channels can be identified on the routing
level, which will eventually matter once PTLCs are fully rolled out.


| Bits  | Name             | Description                           | Context | Dependencies                             | Link                 |
|-------|------------------|---------------------------------------|---------|------------------------------------------|----------------------|
| ??/?? | `option_taproot_assets` | Node supports taproot asset channels | IN      | `option_channel_type` + `option_anchors` + `option_simple_taproot` | TODO(roasbeef): link |

The Context column decodes as follows:

 * `I`: presented in the `init` message.
 * `N`: presented in the `node_announcement` messages
 * `C`: presented in the `channel_announcement` message.
 * `C-`: presented in the `channel_announcement` message, but always odd (optional).
 * `C+`: presented in the `channel_announcement` message, but always even (required).
 * `9`: presented in [BOLT 11](11-payment-encoding.md) invoices.

The `option_taproot_assets` feature bit also becomes a defined channel type
feature bit for explicit channel negotiation.

Throughout this document, we assume that `option_taproot` was negotiated, and
also the `option_taproot` channel type is used.

### New TLV Types

Note that these TLV types exist across different messages, but their type IDs
are always the same.

The starting TLV type for these non-official values is `75249`. 

#### tap_partial_signature_with_nonce
- type: 75249
- data:
   * [`32*byte`: `partial_signature`]
   * [`66*byte`: `public_nonce`]

#### tap_partial_signature
- type: 75250
- data:
   * [`32*byte`: `partial_signature`]

#### tap_htlc_info

- type: 75251
- data:
   * [`32*byte`:`asset_id`]
   * [`BigSize`:`asset_amt`]

#### tap_htlc_sigs

- type: 75252
- data:
   * [`BigSize`:`num_sigs`]
   * [`num_sigs*64`:`htlc_sigs`]

#### tap_onion_rfq_id
- type: 75253
- data:
   * [`32*byte`:`tap_rfq_scid`]

### Channel Funding

In order to fund a new channel with a set of Taproot Assets, before sending the
normal `open_channel` message, the initiator first sends a series of
`tx_asset_proof` messages that allow the responder to verify the authenticity
of the asset level inputs.

// TODO(roasbeef): add more on the mirro'd musig2 state

// TODO(roasbeef): better spell out the prev_id signed for all the TAP musig2
signatures

### `tx_asset_proof` Message

The `tx_asset_proof` message links a prospective channel funding attempt to a
set of proposed assets via the `temporary_channel_id`. For each asset UTXO that
is to be anchored in the final `musig2` funding output, a new `tx_asset_proof`
message is sent.

Each `tap_asset_proof` declares an `asset_id`, an `amt`, and finally a
serialized existence proof for the Taproot Asset.

1. type: ?? (`tap_asset_proof`)
2. data:
  * [`32*byte`:`temporary_channel_id`]
  * [`32*byte`:`asset_id`]
  * [`u64`:`amt`]
  * [`...*byte`:asset_proof`]

// TODO(roasbeef): TLV errwhere?

The `asset_proof` includes a serialized `asset_proof`, which is a _fully
signed_ state transition the asset leaf in question. This serves to prove to
the responder that the asset exists, and that the initiator is capable of
moving them. The `asset_proof` itself is fully singed, which then allows the
responder to independently arrive at the final TAP asset root to be committed
into the funding output.

**Requirements**

The sending node:

* MUST send a new `tap_asset_proof` for each asset UTXO they wish to anchor in
  the final funding output
* MUST set `asset_id` and `amt` to match the `asset_proof` serialized 
* MUST include a valid `asset_proof` as specified in `bip-tap-proof.mediawiki`
* MUST ensure that each `asset_proof` includes a finalized witness

The receiving node:

* MUST verify each incoming `asset_proof` based on the rules of
  `bip-tap-vm.mediawiki` and `bip-tap-proof.mediawiki`
* MUST fail (TODO(roasbeef): maybe finally error messages here?) the channel
  funding early if the `asset_proof` is invalid
* SHOULD incrementally construct a final TAP asset tree in order to be able to
  quickly verify the TAP root which will be sent in the `open_channel` message

#### `open_channel` Extensions

Once all relevant `tap_asset_proof` messages have been sent, then the receiver
MUST send an `open_channel` message with the same `temporary_channel_id`, which
includes the new TLV extensions defined below:

1. `tlv_stream`: `open_channel_tlvs`
2. types:
   1. type: ? (`tap_asset_root`)
   2. data:
      * [`40*byte`:`asset_root_hash || asset_root_sum`]

The new TLV extension includes the final `tap_asset_root` which commits to the
asset leaves revealed in the prior `tx_asset_proof` messages.

**Requirements**
The sending node:

* MUST set the `tap_asset_root` to the final TAp asset root hash+sum created by
  inserting each of the specified asset UTXOs into a TAP tree according to
  `bip-tap.mediawiki`
* MUST NOT send any further `tap_asset_proof` messages after `open_channel` has
  been sent.

The receiving node:

* MUST verify that the `asset_root_hash` matches the root hash they arrived at
  by inserting each of the TLV leaves into a TAP asset tree

#### `accept_channel` Extensions

After verifying that the sent `tap_asset_root` matches what they independently
constructed, the responder MUST then send over a `accept_channel` which
includes the TAP asset root they arrive at. This allows the initiator to verify
that the responder has constructed the same asset root.

1. `tlv_stream`: `accept_channel_tlvs`
2. types:
   1. type: ? (`tap_asset_root`)
   2. data:
      * [`40*byte`:`asset_root_hash || asset_root_sum`]

#### `funding_created` Extensions

Once the responder has sent `accept_channel`, the responder will send a
`funding_created` message as normal. For TAP channels, the `funding_created`
message also inherits a new TLV that carries the `musig2` signature for the
committed TAP funding output.

1. `tlv_stream`: `funding_created_tlvs`
2. types: 
   1. type: 75250 (`tap_partial_signature_with_nonce`)
   2. data:
       * [`32*byte`: `partial_signature`]
       * [`66*byte`: `public_nonce`]

**Requirements**

The sending node:

* MUST set the above `funding_created_tlvs` when sending the `funding_created`
  message
* MUST set the `tap_partial_signature_with_nonce` TLV type to be the `musig`
  partial signature of the TAP funding output as specified in the section on
  Funding Output Creation
  * This signature will cover the TAP virtual tx that sending the TAP funding
    output for the TAP commitment transaction

The receiving node:

* MUST verify the incoming partial signature against the constructed TAP
  commitment transaction which spends the TAP funding output
* MUST store the received partial signature to later be able to broadcast a
  force close transaction with the commitment transaction

#### `funding_accepted` Extensions

To collude with the initial channel funding flow, the responder will then send
a `funding_accepted` message with a musig2 TAP partial signature for the TAP
virtual commitment transaction.

1. `tlv_stream`: `funding_created_tlvs`
2. types: 
   1. type: 75250 (`tap_partial_signature_with_nonce`)
   2. data:
       * [`32*byte`: `partial_signature`]
       * [`66*byte`: `public_nonce`]

**Requirements**

The sending node:

* MUST set the above `funding_created_tlvs` when sending the `funding_created`
  message
* MUST set the `tap_partial_signature_with_nonce` TLV type to be the `musig`
  partial signature of the TAP funding output as specified in the section on
  Funding Output Creation
  * This signature will cover the TAP virtual tx that sending the TAP funding
    output for the TAP commitment transaction

The receiving node:

* MUST verify the incoming partial signature against the constructed TAP
  commitment transaction which spends the TAP funding output
* MUST store the received partial signature to later be able to broadcast a
  force close transaction with the commitment transaction

#### Funding Output Construction

The funding output for the TAP layer is effectively a direct mirror of the
funding output on the Bitcoin layer, with a normal tapscript tweak used rather
than a BIP 86 tweak.

The `tap_asset_proof` message can omit the normal witness field for the TLVs.
Instead, only a challenge TLV needs to be sent to prove that the initiator can
actually move those assets. Unless otherwise specified, all asset TLVs as part
of this protocol will use encoding `v1`, also known as the segwit encoding.

// TODO(roasbeef): partial reveal here? needed in any case to accept funding,
but has similar issues to dual funding reveal, but constrained

Given a set of fully signed TAP asset leaf TLVs (each dubbed `tap_input_leaf`),
the funding output is constructed as follows:

  1. Initialize an empty TAP asset tree dubbed `tap_asset_tree`

  2. For each `tap_input_leaf`:

    1a. If a TAP asset commitment for specified `asset_id` does not exist, then
    initialize an empty instance as a sub-tree within `tap_asset_tree`.

    1b. Insert the `tap_input_leaf` into the TAP asset commitment retrived or
    initialized above.

  3. Generate the serialized `tagged_hash` TAP root commitment as specified in
     the Taproot Assets Protocol section in the top-level Preliminary section,
     dubbed the `tap_tapscript_leaf`.

  4. Generate a new tapscript root commitment, with the `tap_tapscript_leaf` as
     the sole leaf in the tapscript commitment, this value is hence forth
     referred to as the `tap_tapscript_root`.

  5. Give the funding multi-sig keys for each party (`local_funding_key`, and
     `remote_funding_key`) assemble the combined `musig2` funding output by
     combining the _sorted_ funding keys using the `musig2.KeyAgg` routine with
     a _tapscript_ tweak value of `tap_tapscript_root.

The resulting P2TR output mirrors the `musig2` set up of the normal funding
output with an embedded TAP commitment that holds the specified set of assets.

### Cooperative Closure

The co-op close structure is identical to that of regular Taproot channels. 

The `shutdown` message is unchanged, as the same nonce can be used on the TAP
layer as the message (sighash signed) is different, as each time a
`tap_virtual_tx` is signed.

#### `closing_signed` Extensions

We add a new TLV to the `closing_signed` message's existing `tlv_stream` to
carry the partial signature on the TAP layer needed to properly spend TAP
funding output:

1. `tlv_stream`: `closing_signed_tlvs`
2. types:
   1. type: ??? (`tap_partial_signature`)
   2. data:
      * [`32*byte`: `tap_partial_signature`

Both sides **MUST** provide this new TLV field.

#### Requirements

The sender:

  - MUST use the set of nonces exchanged in the `shutdown` message to generate
    a `musig2` session for the closing transaction.

  - MUST sign the `tap_virtual_tx` constructed by spending the input TAP
    funding output, with the asset root anchor of the initiator committing to a
    valid split for the balance of the responder.

The recipient:
  - MUST verify the `tap+partial_signature` field using the
    `PartialSigVerifyInternal` algorithm of `bip-musig2` over the specified
    `tap_virtual_tx` :
  
    - if the partial signature is invalid, MUST fail the channel

### Channel Operation 

#### `update_add_htlc` Extensions

The `update_add_htlc` message is reused to send HTLC which can optionally also
transmit committed TAP assets. 

The existing `tlv_stream` for the `update_add_htlc` message now gains a new set
of TLV types:

1. `tlv_stream`: `update_add_htlc`
2. types:
    1. type: ??? (`tap_htlc_info`)
    2. data:
        * [`32*byte`:`asset_id`]
        * [`BigSize`:`asset_amt`]
        
When the new TLVs are specified, each HTLC value MUST be above the dust limit
(TODO(roasbeef): revisit).

The TAP HTLCs inherit the same `htlc_id` as the normal Bitcoin level HTLCs.

#### `commitment_signed` Extensions

The `commitment_signed` message gains a new set of TLVs to carry both the new
`tap_commit_sig` as well as the set of `tap_htlc_sigs` for each second level
HTLC transactions:

1. `tlv_stream`: `commitment_signed_tlvs`
2. types:
       1. type: ? (`tap_partial_signature_with_nonce`)
       2. data:
          * [`98*byte`: `partial_signature || public_nonce`]
       1. type: ? 
       2. data:
          * [`BigSize`:`num_sigs`]
          * [`num_sigs*64`:`htlc_sigs`]

### Commitment Transaction Construction 

The commitment transaction for TAP channels uses the base Simple Taproot
Channels commitment transaction. Similar to the funding transaction, a TAP
overlay layer is inserted in each active settled commitment transaction output.
The output of the initiator is used to serve as the root split for all created
commitment and HTLC outputs.

#### To Local Outputs

The `to_local` output as identical to the existing construction with the
addition of a new `tap_to_local_script_root:
```
to_delay_script_root = tapscript_root(tap_to_local_script_root, [to_delay_script, revoke_script])
to_local_output_key = taproot_nums_point + tagged_hash("TapTweak", taproot_nums_point || to_delay_script_root)
```

Note how the existing scripts (`to_delay_script`, and `revoke_script`) are used
as a nested sub-list to create the `to_delay_script_root`. This is due to the
fact that for TAP asset, the tapscript commitment needs to be the only element,
or the highest element in the tree. The spec of `bip-tap.mediawiki` elaborates
on this.

The `tap_to_local_script_root` is itself, a nested instance of the _existing_
`to_delay_script_root`. 

The important derivation is the `script_key` of the TLV leaf in the asset tree,
which is derived as:
```
tap_to_delay_script_root = tapscript_root([to_delay_script, revoke_script])
tap_to_local_output_key = taproot_nums_point + tagged_hash("TapTweak", taproot_nums_point || tap_to_delay_script_root)
```

To derive the `tap_to_local_script_root`:

1. For each `asset_id`, `a_i` committed to in the `tap_funding_output`:
     1. Create a new TAP TLV leaf template for `asset_id`, `a_i` based on the input
    multi-sig root.
     2. Modify the `amt` field to match the current settled balance of the initiator.
     3. If the responder has an active balance or active HTLCs exist:
    *  3a. Initialize a new empty split commitment. For each active `asset_id` in
         flight, or held in settled balance:
           * 3a_i. Create a new TLV leaf clone, with the amount set to the
             active balance of the responder, then insert this into the split
             commitment tree.

           * 3a_ii. For each active HTLC, create a new TLV leaf clone and insert
             that into the split commitment tree for the initiator's.

           * 3a_iii. Compile the split commitment to obtain the
             `split_commitment_root` for the asset, populating the
             corresponding asset TLV field in the root split asset.

     4. Using the `tap_partial_signature` exchanged in the prior round, create the
    final combined signature, then set that as the witness. 
     5. For each created split, create the final split commitment root hash,
    replace the value in the asset TLV, then update the inclusion proofs for
    all the commitment + HTLC splits.
     6. Collect the root commitment for `a_i` in a temporary variable `c_a_i`
2. Given each taproot asset inner commitment `c_a_i`, assemble a final TAP asset
   tree, dubbed `asset_tree_root`.
3. With the final `asset_tree_root` root commitment output TLV, construct a
   `tap_to_local_script_root` based on `bip-tap.mediawiki`.


#### To Remote Outputs

The `to_remote` output mirrors the structure of the `to_local` output. This
output also holds a TAP commitment, with the commitment holding the split from
the root asset in the `to_local` output. 

The construction of the `to_remote` output is identical, with the modification
of the `to_remote_script_root`, which now gains the
`tap_to_remote_script_root`:
```
to_remote_script_root = tapscript_root([to_remote_script, tap_to_remote_script_root])`
to_remote_output_key = taproot_nums_point + tagged_hash("TapTweak", taproot_nums_point || to_remote_script_root)
```

The script key used for the asset TLV in the `tap_to_remote_script_root` then
mirrors the inner structure of the `to_remote_script`:
```
tap_to_remote_script_root = tapscript_root([to_remote_script])
```

To derive the `tap_to_remote_script_root`:

1. For each `asset_id`, `a_i` that the responder has a non-zero balance of:
    1. Create a new TAP TLV leaf for asset ID `a_i` that spends no inputs (it's
       a split commitment leaf).
    2. Modify the `amt` field to match the settled balance of the responder.
    3. Referring to the anchor asset's split commitment root as
       `split_root_a_i`, generate a valid split commitment witness proof for
       the asset.
    4. Construct a valid split commitment witness for the asset
       `split_witness_i`, populating the asset's witness accordingly.
    5. Create a new asset commitment tree with the leaf subbed `c_i`.
2. Assemble each asset commitment root `c_i` into a final TAP asset tree,
   dubbed `asset_tree_root`.
3. With the final `asset_tree_root` root commitment output TLV, construct a
   `tap_to_remote_script_root` based on `bip-tap.mediawiki`.

#### Anchor Outputs

Anchor outputs for TAP channels are unchanged. The committed BTC amount of the
channel is used to pay for the general transaction fees, as well as the value
of the anchor outputs.

### HTLC Scripts & Transactions

#### Offered HTLCs

Similar to the settled commitment outputs, each HTLC also mirrors a new
tapscript commitment that commits to the TAP root that holds the target HTLC
amount and asset ID.

The `htlc_script_root` is now crafted as:
```
htlc_script_root = tapscript_root(tap_htlc_script_root, [htlc_timeout, htlc_success])
```

The `asset_script_key` for the sole asset committed to within the
`tap_htlc_script_root` is derived as:
```
tap_htlc_asset_script_key = tapscript_root([htlc_timeout, htlc_success])`
```

To derive the `tap_htlc_script_root`:

1. Create a new TAP TLV leaf for the target `asset_id`, and `amt`.
2. Referring to the anchor asset's split commitment root as
   `split_root_htlc_i`, generate a valid split commitment witness proof for the
   asset.
3. Construct a valid split commitment witness for the asset
   `split_witness_htlc_i`, populating the asset's witness accordingly.
5. Create a new asset commitment tree with the leaf subbed `c_htlc`.
6. Create a new TAP commitment tree containing only `c_htlc` dubbed
   `asset_tree_root`.
3. With the final `asset_tree_root` root commitment output TLV, construct a
   `tap_htlc_script_root` based on `bip-tap.mediawiki`.


#### Accepted HTLCs

Accepted HTLCs mirror the structure of the Offered HTLCs with the TAP layer
mirroring the script at the taproot layer.

The `htlc_script_root` is now crafted as:
```
htlc_script_root = tapscript_root(tap_htlc_script_root, [htlc_timeout, htlc_success])
```

The `asset_script_key` for the sole asset committed to within the
`tap_htlc_script_root` is derived as:
```
tap_htlc_asset_script_key = tapscript_root([htlc_timeout, htlc_success])`
```

To derive the `tap_htlc_script_root`:

1. Create a new TAP TLV leaf for the target `asset_id`, and `amt`.
2. Referring to the anchor asset's split commitment root as
   `split_root_htlc_i`, generate a valid split commitment witness proof for the
   asset.
3. Construct a valid split commitment witness for the asset
   `split_witness_htlc_i`, populating the asset's witness accordingly.
5. Create a new asset commitment tree with the leaf subbed `c_htlc`.
6. Create a new TAP commitment tree containing only `c_htlc` dubbed
   `asset_tree_root`.
3. With the final `asset_tree_root` root commitment output TLV, construct a

##### HTLC-Success Transactions

For the HTLC success transaction, the `htlc_script_root` is replicated to the
TAP level output for the incoming HTLC (for the acceptor).

The `htlc_script_root` is modified to also include a TAP level inner script
root:
```
htlc_script_root = tapscript_root([tap_htlc_script_root, htlc_success])
```

The key used for the asset TLV for the HTLC-success output is derived as:
```
asset_script_key_htlc = tapscript_root([htlc_success])
```

This `asset_script_key_htlc` is then used as the `asset_script_key` for the
asset TLV leaf anchored in the designated output.

To create the `tap_htlc_script_root`:

1. Create a new TAP TLV leaf template with `asset_id` and `amt` based on the
   input asset. The `prev_asset_id` MUST be set to spend the input TAP
   commitment within the spent HTLC output.
2. Assign the `asset_script_key_htlc` to the `script_key` field of the TLV>
3. Create a new TAP commitment tree containing only `c_htlc` dubbed
   `asset_tree_root`.
4. With the final `asset_tree_root` root commitment output TLV, construct a

During the `commitment_signed` phase, this new TAP level output will be covered
by the sent `tap_htlc_signatures`.


##### HTLC-Timeout Transactions

For the HTLC-Timeout transaction, the `htlc_script_root` is replicated to the
TAP level output for the outgoing HTLC (for the acceptor).

The `htlc_script_root` is modified to also include a TAP level inner script
root:
```
htlc_script_root = tapscript_root([tap_htlc_script_root, htlc_timeout])
```

The key used for the asset TLV for the HTLC-Timeout output is derived as:
```
asset_script_key_htlc = tapscript_root([htlc_timeout])
```

This `asset_script_key_htlc` is then used as the `asset_script_key` for the
asset TLV leaf anchored in the designated output.

To create the `tap_htlc_script_root`:

1. Create a new TAP TLV leaf template with `asset_id` and `amt` based on the
   input asset. The `prev_asset_id` MUST be set to spend the input TAP
   commitment within the spent HTLC output.
2. Assign the `asset_script_key_htlc` to the `script_key` field of the TLV>
3. Create a new TAP commitment tree containing only `c_htlc` dubbed
   `asset_tree_root`.
4. With the final `asset_tree_root` root commitment output TLV, construct a

During the `commitment_signed` phase, this new TAP level output will be covered
by the sent `tap_htlc_signatures`.

* all reserve requirements still expressed in BTC
  * BTC still needed for channel as still tool for fees
  * given fee rate can compute amt of BTC needed for 483 HTLCs in each direction
  * not that fdiff from channels today re min channlel size

### Last Mile Routing

In order to bridge channels with TAP assets to the Lightning Network, a
specialized set of last-mile liquidity providers are required. These providers
have incoming/outgoing channels of TAP assets, then utilize the _existing_ HTLC
mechanism to complete the route by bridging the TAP edge liquidity with the
Bitcoin Backbone portion of the Lightning Network.

For each logical payment to be accepted, the Edge Node and the Liquidity
Provider engage in Request-For-Quote (RFQ) swap negotiation. Once a price has
been accepted, a final response is returned that uniquely identifies that
specified price. The final quote is then identified by either a `tap_rfq_scid`
or a `tap_rfq_hash`. The former is laced in the first hop for an outgoing
payment with a TAP origin, and the latter placed within the normal onion
payload for a last-mile hop into a TAP asset.

All quotes are ephemeral and will expire along side the created invoice.
Epidermal quotes ensure that all sides are able to limit their exposure to
price fluctuations. It's recommended that invoices expire within minutes in
order to clamp down on price volatility exposure.

TODO(roasbeef): explain AMP/keyspend, basically unsolicited quote req, can
accept if favorable


#### RFQ Negotiation

##### Request For Quote (`tap_rfq`)

When a receiver wishes to receive `N` units of TAP asset ID `asset_id`, a new
p2p message `tap_rfq` is sent with the following structure:

1. type: ?? (`tap_rfq`)
2. data:
     * [`32*byte`:`rfq_id`]
     * [`32*byte`:`asset_id`]
     * [`BigSize`:`asset_amt`]
     * [`BigSize`:`suggested_rate_tick`]

where:

* `rfq_id` is a randomly generate 32-byte value to uniquely identify this RFQ
  request
* `asset_id` is the asset ID of the asset the receiver intends to receive
* `asset_amt` is the amount of units of said asset
* `suggested_rate_tick` is the internal unit used for asset conversions. A tick
  is 1/10000th of a currency unit. It gives us up to 4 decimal places of
  precision (0.0001 or 0.01% or 1 bps). As an example, if the BTC/USD rate was
  $61,234.95, then we multiply that by 10,000 to arrive at the `usd_rate_tick`:
  `$61,234.95 * 10000 = 612,349,500`.  To convert back to our normal rate, we
  decide by `10,000` to arrive back at `$61,234.95`.

Given valid `rfq_id`, we then define an `tap_rfq_scid` by taking the last `8`
bytes of the `rfq_id` and interpreting them as a 64-bit integer. 

###### Requirements

The sender:

  - MUST ensure that the `tap_rfq_scid` mapping of an `rfq_id` doesn't collide
    with the `scid` of any active channels.

  - MUST ensure that an `rfq_id` is never repeated for the lifetime of a
    connection
    - It is recommended that the value be generated using a CSPRNG. Otherwise,
      a simple counter system from a starting value can be used, with the nonce
      offer incrementing by one each time.

  - MUST set `asset_id` to the ID of an asset contained in the backing channel.

  - SHOULD specify reasonable values for `suggested_rate_tick` 

The recipient:

  - SHOULD send a `tap_rfq_reject` message if `rfq_id` has been used before

  - MUST send an `tap_rfq_reject` message if `asset_id` is not committed to in
    the open channel

  - MUST send an `tap_rfq_reject` message if the requested `asset_amt` is
    greater than the settled remote balance of that asset

  - SHOULD take the `suggested_rate_tick` values into account when deciding
    whether to accept or reject the quote

#### Request For Quote Response (`tap_rfq_accept`) + (`tap_rfq_reject`)

Once the edge node has received, the `tap_rfq` message, then it should decide
if it's able to accommodate the quote or not. 

#### Accepting Quotes  (`tap_rfq_accept`) 

If it can, then it should send `tap_rfq_accept` that returns the quote amount
the edge node is willing to observe to move `N` units of asset `asset_id`:

1. type: ?? (`tap_req_accept`)
2. data:
     * [`32*byte`:`rfq_id`]
     * [`BigSize`:`accepted_rate_tick]
     * [`BigSize`:`expiry_seconds`]
     * [`64*byte`:`rfq_sig`]

TODO(roasbeef): tlv err where?

where:

* `rfq_id` matches the existing `rfq_id` of a set `tap_rfq`

* `accepted_rate_tick` is the proposed rate for the volume unit expressed in
  the internal unit of a `tick`.

* `expiry_seconds` is the amount of seconds to use for the expiry of both the
  quote and the invoice

* `rfq_sig` is a signature over the serialized contents of the message 

##### Requirements

The sender:

  - MUST set `rfq_id` to the matching `rfq_id` sent in a prior `tap_rfq` message

  - MUST set `accepted_rate_tick` to a value they deem to be an acceptable
    exchange rate

  - MUST set `expiry_seconds` to the relative expiry time in the future that
    the quote will expire after which

  - MUST set `rfq_sig` to be a BIP-340 schnorr signature over the serialized
    contents of the message without the `rfq_sig` serialized, using their node
    public key

The recipient:

  - MUST abandon the attempt if `rfq_sig` is invalid

  - MUST abandon the attempt if they deem that `accepted_rate_tick` is
    unreasonable

  - SHOULD no longer attempt to utilize the cleared quote after
    `expiry_seconds` has elapsed

#### Rejecting Quotes  (`tap_rfq_reject`) 

In the event that an edge node is unable to satisfy a quote request, then they
should send `tap_rfq_reject`, identifying the rejected quote ID. A quote might
be rejected if the channel cannot accommodate the proposed volume, or if the
edge node is unwilling to carry any HTLCs for that `asset_id`.

1. type: ?? (`tap_req_accept`)
2. data:
     * [`32*byte`:`rfq_id`]

where:

* `rfq_id` is the quote ID that they wish to reject

##### Requirements

The sender:

  - MUST set `rfq_id` to the matching `rfq_id` sent in a prior `tap_rfq` message

The recipient:

  - MUST not attempt to send/receive using the rejected `rfq_id`


#### First Hop TAP HTLC Onion Processing

When sending a payment that uses an outbound TAP asset HTLC as it's origin, in
order for the payment succeed, the sender must place the negotiated `rfq_id` in
the payload for the first hop (the liquidity provider):

- type: 75253
- data:
     * [`32*byte`:`tap_rfq_id`]

The `tap_onion_rfq_id` MUST be present in the onion to the outgoing peer if the
HTLC has a TAP asset origin.

In addition to the normal hop payload checks, the forwarding node MUST also
verify that the conversion rate of the outgoing HTLC as specified in the onion
matches the `accepted_rate_tick` of the corresponding `tap_rfq_id`.

When receiving an incoming onion TLV payload sourced from a TAP channel, the
receiving node:

- MUST reject the payment if the incoming HLTC is a TAP asset, and the outgoing
  payload doesn't include a `tap_rfq_id` value.

- MUST reject the payment if `tap_rfq_id` is unknown.

- MUST reject the payment if the `tap_rfq_id` has expired based on the
  posted `expiry_seconds` value.

- MUST reject the payment the `amt_to_forward != (amt_asset_incoming * tick *
  msat_multiplier) / accepted_rate_tick

where:

  * `amt_asset_incoming` is the HTLC value of the `asset_id` quote accepted in
    the earlier RFQ round

  * `msat_multiplier` is a factor used to scale from `BTC/asset` to
    `msat/asset_id`, this value is constant within the protocol and is derived
    by multiplying the number of sats per Bitcoin, by the amount of sats in an
    msat: `msat_multiplier = 100_000_000 * 1000 = 100_000_000_000` (100
    billion)

    * As an example: 
      * If the incoming channel is a USD backed asset, and wishes to send $1000
        outbound, with a rate of $61234.95 (`accepted_rate_tick = 61234.95 *
        tick = 612_349_500` then the forwarding node expects to be instructed
        to send `1,633,054,326 mSAT` over the outgoing link:
          * `amt_to_forward = ($1000 * tick * msat_multiplier) // 612_349_500`
          * `amt_to_forward = (1_000 * 10_000 * 100_000_000_000) // 612_349_500`
          * `amt_to_forward = (1000000000000000000) // 612_349_500`
          * `amt_to_forward = 1_633_054_326`

#### Last Hop TAP HTLC Onion Processing

In the event that the last hop in a route receive a payload that has an
`short_channel_id` value that matches a prior accepted `rfq_scid` value, then
this indicates that the sender is attempting to pay a receiver in the asset
bound by the `rfq_id` and `rfq_scid`.

Note that we don't require that the sender use any special values other than
what is already known in the existing protocol. This enables _unupgraded_
senders to send BTC, with the receiver obtaining their asset of choice, without
the sender needing to worry about exchange rates at all.

When receieving an incoming onion payload with a known `rfq_scid` value, the
receiver:

- MUST reject the HTLC is `tap_scid` is expired based on the posted
  `expiry_seconds` value

- MUST reject the entire HTLC set if at anytime, the sum of HTLCs (the
  `amt_to_forward` field) targetting `tap_rfq_scid` eceeds the negotiated
  `asset_amt` field (volume for quote exhausted)

- MUST extend a TAP HTLC with an `asset_id` corresponding to the accepted
  `tap_rfq_scid` with an asset value of: `((amt_asset_incoming *
  accepted_rate_tick) // msat_multiplier) / tick`

  * As an example:
    * If the outgoing channel (last hop receiver) is a USD backed asset, and
      requested an invoice `1_633_054_326 mSAT`, but wants USD with an accepted
      tick rate of `612_349_500` ($61,234.95), then the penultimate node should
      send $1000 to the last hop:
        * `amt_to_forward = ((incoming_amt_msat * accepted_rate_tick) // msat_multiplier) / tick`
        * `amt_to_forward = ((1_633_054_326 * 612_349_500) // 100_000_000_000) / 10_000`
        * `amt_to_forward = ((999999999998937000) // 100_000_000_000) / 10_000`
        * `amt_to_forward = (9999999 / 10_000)`
        * `amt_to_forward = (9999999 / 10_000)`
        * `amt_to_forward = 999.9`
    * Note that all assets internally are accounted in a unit of a `tick`
      (1/1000th) of an asset. When convering back to the main asset, the value
      should be rounded up, giving us the original value of `$1000`.



#### Invoice Format

TAP invoices look just like normal invoices with the exception that the routing
hop hint included for the last mile includes a `tap_rfq_scid` rather than a
_normal_ scid value. This is similar to the existing `scid-alias-feature`
widely deployed across the Lightning Network. During the negotiation process,
after a `tap_rfq_scid` is accepted, the border node then will forward received
HTLCs to that final hop.

From the PoV of the sender, the route construction when starting from a
BTC-only link is no different. The final rate negotiated is then transparently
passed on as an additional last-hop fee.

When creating an invoice, the creator MUST ensure that the invoice expiry value
is set exactly to the `expiry_seconds` value of the accepted RFQ.

When creating an invoice from an `accepted_rate_tick`, with a base currency of
`asset_id`, a conversion MUST be carried out to express the desired amount in
terms of _BTC_ rather than the asset tick. As an example:

  * A user wants to receive $100 over their USD backed channel. The
    `accepted_rate_tick` that can satisfy that volume, and hasn't expired yet
    is `612_349_500` or `$61,234.95`. To arrive at the `mSAT` value they should
    put into the invoice:
      * `invoice_amt = (asset_amt * tick * msat_multiplier) / accepted_rate_tick`
      * `invoice_amt = (100 * 10_000 * 100_000_000_000) / 612_349_500`
      * `invoice_amt = (100000000000000000) / 612_349_500`
      * `invoice_amt = (100000000000000000) / 612_349_500`
      * `invoice_amt = 163305432 mSAT`
      * `invoice_amt = 163305 SAT`
    
Always expressing the invoice amount in BTC/mSAT ensures that unpugraded
senders will be able to send over these asset channels.


## Universality

## Backwards Compatibility

## Reference Implementation
