```
bLIP: xxxx
Title: General-purpose Lightning signatures
Status: Draft
Author: Martin Habovštiak
Created: 2022-01-04
License: CC0
```

## Abstract

This document describes general-purpose signatures tied to Lightning Network
node identifiers (pubkeys) that can be used in user-facing applications or
custom extension protocols ("3rd layer") e.g. to implement reputation or
advertising of LN nodes. They are **not** intended to be used
in the Lightning Network Protocol itself (not BOLT).

This bLIP serves to document what is already well-supported in the wild (see
[Reference Implementations](#reference-implementations)), for posterity and so
that new implementations don't have to reverse-engineer general-purpose
Lightning signatures from existing implementations.

## Copyright

This bLIP is licensed under the CC0 license.

## Specification

The signature is ECDSA secp256k1 recoverable signature using *double* sha256 as
the hasing algorithm. The message MUST be prefixed with ASCII-encoded text
`Lightning Signed Message:` before signing/verification.
The content or encoding of the message is unrestricted.

The signature is encoded as 65 bytes: one-byte recovery ID and 64 bytes of
compact signature as defined by `libsecp256k1`:

> The signature must consist of a 32-byte big endian R value, followed by a
> 32-byte big endian S value. If R or S fall outside of [0..order-1], the
> encoding is invalid. R and S with value 0 are allowed in the encoding.

The constant `0x1f` has to be added to the recovery ID before encoding.
(e.g. recovery ID 0x00 will cause the final signature begin with 0x1f)

Multiple implementations encode the signature in zbase32 when presenting it to
the user. At the time of writing Eclair uses hex instead because zbase32 is
uncommon.
This document does **not** mandate any specific user-facing encoding.

### Test Vectors

Valid signature:

Pubkey: `029ef8ee0ba895e2807ac1df1987a7888116c468e70f42e7b089e06811b0e45482` (hex-encoded)
Message: ``I am the author of \`ln-types\` Rust crate.`` (ASCII-encoded)
Signature: `1fdc7b075774fbfbf9b546535f4ff3912f507c134b9e30fa519445a549475228ec71624a6e6671c976837b6f63faf42eed2034c9645cd2b58fd65205794278b01b` (hex-encoded)

Changing any byte in pubkey, message, or signature should produce invalid signature.

## Motivation

These signatures can be useful for proving ownership of an LN node, initiating
secure communication between node operators (e.g. by signing GPG keys with node
pubkeys) or other purposes.

As mentioned earlier this bLIP describes what already exists and is being used
and supported by multiple popular LN implementations, so it's useful to document it.

## Rationale

Double hashing and prefixing the message with a constant text should minimize
the risk of these signatures being confused with invoices or other LN-related
signatures. The implementors of higher-level protocols may want to implement
additional measures to prevent confusion or replay attacks among multiple
higher-level protocols.

## Reference Implementations

LND: https://github.com/lightningnetwork/lnd/pull/192

Eclair: https://github.com/ACINQ/eclair/pull/1499

C-Lightning: https://github.com/ElementsProject/lightning/pull/3150

`ln-types` Rust library: https://docs.rs/ln-types/0.1.5/ln\_types/node\_pubkey/struct.NodePubkey.html#method.verify\_lightning\_message (verification only)
