```
bLIP: XXXX
Title: StateChannels
Status: Draft
Author: Nicholas Gregory <nicholas@commerceblock.com> 
     Ruben Somsen <rsomsen@gmail.com>
     Tom Trevethan <tom@commerceblock.com> 
Created: 2022-XX-XX
License: GPL 3.0
```

## Abstract

This document decribes how a Poon-Dryja lightning channel (lightning network) can be constructed without reliance upon on-chain transactions, using the statechain approach as implemented in MercuryWallet.  This protocol will not require any changes to the lightning network protocol and would need only simple enhancements to the MercuryWallet statechain API.

## Motivation

The MercuryWallet statechain system facilitates the transfer of ownership of Bitcoin (or Elements based) unspent transaction outputs (UTXOs) between parties without performing on-chain transactions. This is acheieved by sharing the single key of the UTXO between the owner and a statechain entity (SE), and providing a time-locked backup transaction that returns the UTXO to the full custody of the owner.  

Combining this capability with a payment channel provides two potential benefits:
1. Capability to “unroll” a once fixed-size state coin UTXO.
2. The ability to transfer a lightning channel off-chain at any time.

## Rationale

A statechain’s security model rests on certain assumptions. Initially, there is an SE and an owner (Alice), neither of which has the full underlying UTXO private key. Before depositing funds into the statechain UTXO, Alice will cooperatively generate a backup transaction with the SE, timelocked to a future blockheight. In the event the SE goes offline, Alice can recover her funds when the timelock expires. When Alice transfers the UTXO to a new owner (Bob), Alice and the SE collaboratively generate a new backup transaction paying the coin to Bob with a timelock that expires before Alice's original one, and the SE updates its key share with Bob. With each new peg-out transaction, a timelock is decremented. While this does put a limit on the total number of times a statecoin can be transferred, this timelock ensures that Bob’s peg-out transaction can be confirmed before Alice’s. Ultimately, Bob trusts that the SE and Alice do not collude, or that the SE and Alice are not the same person.

This BLIP focuses on the model where a lightning channel is created inside a statecoin peg-out transaction.  As this channel is not visible on-chain, this will be considered to exist with a subcategory of zero-conf channels. 

A 'statechannel' will then have the following attributes:
1. The state chain peg-out (withdrawal or backup) transaction will open a lightning channel.
2. The lightning channel will be timelocked.
3. Closure of the channel without SE cooperation will require the timelock to expire (i.e. the backup tx to be confirmed).
4. Each update to the peg-out transaction will result in the timelock being decremented.
5. A peg-out transaction without timelock can be cooperatively signed with the SE but must be authorised by both parties (agreeing channel balance). The SE is trusted to enforce this. 
6. The statecoin can be transferred to a new owner off-chain (Alice or Bob) if either one has the full channel balance. 

## Specification

The following sequence specifies the process of the creation and closing of the 'state channel':

1. A statecoin is deposited by Alice (A) with the SE. A has one keyshare (`S_A`) and the SE has the other (`S_SE`) - the full private key is `S_A*S_SE`. The SE and A cooperate to co-sign (via 2 party ECDSA) Alice's backup transaction with an `nLocktime` of blockheight `b0`. 
2. A finds channel counterparty Bob (B) that supports `option_state_channel`. 
3. A and B generate secrets `A_T` and `B_T` respectively. 
4. A and B interact to generate a shared secret, `T` via Diffie-Helman key exchange (using `A_T` and `B_T`). 
5. A and B share the public values `A_T.G`, `B_T.G` and `T.G` with the SE (where `G` is the secp256k1 generator point). These are used to authenticate SE co-signing. 
6. A and the SE cooperatively generate the statechannel peg-out transaction (which is authorised by B) with an `nLocktime` of `b0 - d` (where `d` is the decrement value). This transaction creates the channel (i.e. is a 2-of-2 multisig output for `A_T.G`, `B_T.G`). 
7. A, B and the SE complete the statecoin transfer (i.e. the SE key share is updated to combine with `T`). 
8. A and B can now announce the channel open to the network. In the `open_channel` message the `temporary_channel_id` should specify the timelocked peg-out transaction and the `open_channel_tlvs` must reflect that this channel is an `option_state_channel`. 

At any time, A and B can request the SE to co-sign peg-out transaction with no timelock. The SE ensures both A and B authorise the output addresses and amounts. In the event that a non-timelocked peg-out transaction is broadcast to the network, Alice or Bob must specify the new channel outpoint in the `funding_created` message, as the `txid` has has changed.

In the event that Alice and Bob wish to virtually close their statechannel (and all funds have been pushed to one side of the channel), A and B can sign a statecoin transfer message with `A_T` and `B_T`. Once the SE has validated this message, the statecoin can be transferred from to `SE+A` or `SE+B` using the normal statecoin transfer protocol. When the receiver verified the transfer, they announce the closure of the `SE+T` SC to the public lightning network. 

State Channel (SC) steps:

```
     _______           S_A*S_SE              _____________
    |       |<----------------------------->|             |
    |       |          backup_tx (S_A)      |             |
    |   A   |<----------------------------->|     SE      |       <-  A deposit statecoin
    |       |          nLocktime = b0       |             |
    |_______|                               |_____________|
    
                                
                                
     _______                                     A_T.G                                     _______ 
    |       |---------------------------------------------------------------------------->|       |
    |       |                                    B_T.G                                    |       |
    |   A   |<----------------------------------------------------------------------------|   B   |     <- Create shared secret (T)
    |       |                                T = A_T*B_T.G                                |       |
    |       |<--------------------------------------------------------------------------->|       |
    |       |                                                                             |       |
    |       |             A_T.G              _____________             B_T.G              |       |
    |       |------------------------------>|             |<------------------------------|       |
    |       |        backup_tx (A+B)        |             |         backup_tx (A+B)       |       |     <- create state_channel (S_SE*T)
    |       |<----------------------------->|     SE      |<----------------------------->|       |
    |       |        nLocktime = b0 - d     |             |         nLocktime = b0 - d    |       |
    |_______|                               |_____________|                               |_______|
    
    
     _______                                                                               _______ 
    |       |                                                                             |       |
    |       |                                     A+B                                     |       |
    |   A   |<--------------------------------------------------------------------------->|   B   |     <- update state_channel
    |       |                                                                             |       |
    |_______|                                                                             |_______|   
    
                                
     _______          Auth Sig(A_T)          _____________           Auth Sig(B_T)         _______ 
    |       |------------------------------>|             |<------------------------------|       |
    |       |          Sig (S_SE+T)         |             |           Sig (S_SE+T)        |       |
    |   A   |<----------------------------->|     SE      |<----------------------------->|   B   |      <- close state_channel
    |       |                               |             |                               |       |
    |_______|                               |_____________|                               |_______|
```

## Universality

Most nodes are probably not interested in State Channels, but that is fine since
for the approach to be useful only two parties are needed to be interested in
establishing such a channel to support it, and they connect to each other directly.

## Backwards Compatibility

The State Channel protocol is not backwards compatible.

## Acknowledgements

Special thank you to Antoine Riard & Shinobi for reviewing and provided feedback for this proposal.

## Reference Implementation

TODO

[Original Statechains Paper]: https://github.com/RubenSomsen/rubensomsen.github.io/blob/master/img/statechains.pdf
