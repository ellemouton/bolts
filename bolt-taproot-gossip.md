# Extension BOLT XX: Taproot Gossip

# Table of Contents

  * [Aim](#aim)
  * [Overview](#overview)
  * [Terminology](#terminology)
  * [Type Definitions](#type-definitions)
  * [TLV Based Messages](#tlv-based-messages)
  * [Taproot Channel Proof and Verification](#taproot-channel-proof-and-verification)
  * [Timestamp fields](#timestamp-fields)
  * [Bootstrapping Taproot Gossip](#bootstrapping-taproot-gossip)
  * [Specification](#specification)
    * [Feature Bits](#feature-bits)
    * [The `announcement_signatures_2` message](#the-announcement_signatures_2-message)
    * [The `channel_announcement_2` message](#the-channel_announcement_2-message)
    * [The `node_announcement_2` message](#the-node_announcement_2-message)
    * [The `channel_update_2` message](#the-channel_update_2-message)
  * [Appendix A: Algorithms](#appendix-a-algorithms)
      * [Partial Signature Calculation](#partial-signature-calculation)
      * [Partial Signature Verification](#partial-signature-verification)
  * [Appendix B: Test Vectors](#appendix-b-test-vectors)
  * [Acknowledgements](#acknowledgements)

## Aim 

This document aims to update the gossip protocol defined in [BOLT 7][bolt-7] to
allow for advertisement and verification of taproot channels. An entirely new
set of gossip messages are defined that use Schnorr signatures and that use a
purely TLV based schema.

## Overview

The initial version of the Lightning Network gossip protocol as defined in
[BOLT 7][bolt-7] was designed around P2WSH funding transactions. For these
channels, the `channel_announcement` message is used to advertise the channel to
the rest of the network. Nodes in the network use the content of this message to
prove that the channel is sufficiently bound to the Lightning Network context
and that it is owned by the nodes advertising the channel. This proof and
verification protocol is, however, not compatible with SegWit V1 (P2TR)
outputs and so cannot be used to advertise the channels defined in the
[Simple Taproot Channel][simple-taproot-chans] proposal. This document thus aims
to define an updated gossip protocol that will allow nodes to both advertise and
verify taproot channels. This part of the update affects the
`announcement_signatures` and `channel_announcement` messages.

The opportunity is also taken to rework the `node_announcement` and
`channel_update` messages to take advantage of Schnorr signatures and TLV
fields. Timestamp fields are also updated to be block heights instead of Unix
timestamps.

## Terminology

- Collectively, the set of new gossip messages will be referred to as 
  `taproot gossip`
- `node_1` and `node_2` refer to the two parties involved in opening a channel.
- `node_ID_1` and `node_ID_2` respectively refer to the public keys that
  `node_1` and `node_2` use to identify their nodes in the network.
- `bitcoin_key_1` and `bitcoin_key_2` respectively refer to the public keys
  that `node_1` and `node_2` will use for the specific channel being opened.
  The funding transactions output will be derived from these two keys.

## Type Definitions

The following convenient types are defined:

* `schnorr_signature`: a 64-byte Schnorr signature as defined
  in [BIP340][bip-340].
* `partial_signature`: a 32-byte partial Musig2 signature as defined
  in [BIP-MuSig2][bip-musig2].
* `public_nonce`: a 66-byte public nonce as defined in [BIP-MuSig2][bip-musig2].

## TLV Based Messages

The initial set of Lightning Network messages consisted of fields that had to 
be present. Later on, TLV encoding was introduced which provided a backward 
compatible way to extend messages. To ensure that the new messages defined in 
this document remain as future-proof as possible, the messages will be pure 
TLV streams. 

## Taproot Channel Proof and Verification

Taproot channel funding transaction outputs will be SegWit V1 (P2TR) outputs of
the following form:

```
OP_1 <taproot_output_key> 
```

where 
```
taproot_internal_key = MuSig2.KeyAgg(MuSig2.KeySort(bitcoin_key_1, bitcoin_key_2))

taproot_output_key = taproot_internal_key + tagged_hash("TapTweak", taproot_internal_key) * G
```

`taproot_interal_key` is the aggregate key of `bitcoin_key_1` and
`bitcoin_key_2` after first sorting the keys using the
[MuSig2 KeySort][musig-keysort] algorithm and then running the
[MuSig2 KeyAgg][musig-keyagg] algorithm.

Then, to commit to spending the taproot output via the keyspend path, a
[BIP86][bip86-tweak] tweak is added to the internal key to calculate the
`taproot_output_key`.

As with legacy channels, nodes will want to perform some checks before adding 
an announced taproot channel to their routing graph:

1. The funding output of the channel being advertised exists on-chain, is an
   unspent UTXO and has a sufficient number of confirmations. To allow this 
   check, the SCID of the channel will continue to be included in the channel
   announcement.
2. The funding transaction is sufficiently bound to the Lightning context. 
   Nodes will do this by checking that they are able to derive the output key 
   found on-chain by using the advertised `bitcoin_key_1` and `bitcoin_key_2` 
   along with a [BIP86 tweak][bip86-tweak]. This provides a slightly weaker
   binding to the LN context than legacy channels do but at least somewhat 
   limits how the output can be spent due to the script-path being disabled.
3. The owners of `bitcoin_key_1` and `bitcoin_key_2` agree to be associated with  
   a channel owned by `node_1` and `node_2`.
4. The owners of `node_ID_1` and `node_ID_2` agree to being associated with the 
   output paying to `bitcoin_key_1` and `bitcoin_key_2`. 

For legacy channels, these last two proofs are made possible by including four
separate signatures in the `channel_announcement`. However, Schnorr signatures
and the MuSig2 protocol allow us to now aggregate these four signatures into a
single one. Verifiers will then be able to aggregate the four keys
(`bitcoin_key_1`, `bitcoin_key_2`, `node_ID_1` and `node_ID_2`) using MuSig2 key
aggregation and then they can do a single signature verification check instead
of four individual checks.

## Timestamp fields

_TODO: write about the benefits of switching to block-height based timestamps_

## Bootstrapping Taproot Gossip

While the network upgrades from the legacy gossip protocol to the taproot gossip
protocol, the following scenarios may exist for any node on the network:

| scenario | has legacy channels | has taproot channels  | should send `node_announcement` | should send `node_announcement_2` |
|----------|---------------------|-----------------------|---------------------------------|-----------------------------------|
| 1        | no                  | no                    | no                              | no                                |
| 2        | yes                 | no                    | yes                             | no                                |
| 3        | yes                 | yes                   | yes                             | no                                |
| 4        | no                  | yes                   | no                              | yes                               |

### Scenario 1

These nodes have no announced channels and so should not be broadcasting legacy
or new node announcement messages.

### Scenario 2

If a node has legacy channels but no taproot channels, they should broadcast
only a legacy `node_announcement` message. Both taproot-gossip aware and unaware
nodes will be able to verify the node announcement since both groups are able to
process the `channel_announcement` for the legacy channel(s). The reason why 
upgraded nodes should continue to broadcast the legacy announcement is because 
this is the most effective way of spreading the `option_taproot_gossip` feature 
bit since un-upgraded nodes can assist in spreading the message. 

### Scenario 3

If a node has both legacy channels and taproot channels, then like Scenario 2 
nodes, they should continue broadcasting the legacy `node_announcement` 
message.

### Scenario 4

In this scenario, a node has only taproot channels and no legacy channels. This
means that they cannot broadcast a legacy `node_announcement` message since 
un-upgraded nodes will drop this message due to the fact that no legacy 
`channel_announcement` message would have been received for that node. These 
nodes will also not be able to use a legacy `node_announcement` to propagate 
their new `node_announcement_2` message.

### Consideration & Suggestions

While the network is in the upgrade phase, the following suggestions apply:

- Keeping at least a single, announced, legacy channel open during the initial 
  phase of the transition to the new gossip protocol could be very beneficial 
  since nodes can continue to broadcast their legacy `node_announcement` and 
  thus more effectively advertise their support for `option_taproot_gossip`. 
- Nodes are encouraged to actively connect to other nodes that advertise the 
  `option_taproot_gossip` feature bit as this is the only way in which they 
  will learn about taproot channel announcements and updates.
- Nodes should not use the new `node_announcement_2` message until they have
  no more announced legacy channels.

### Alternative Approaches

An alternative approach would be to add an optional TLV to the legacy
`node_annoucement` that includes the serialised `node_announcement_2`. The nice
thing about this is that it allows the new `node_announcement_2` message to
propagate quickly through the network before the upgrade is complete. The
downside of this approach is that it would more than double the size of the
legacy `node_announcement` and most of the information would be duplicated. It
also makes gossip queries tricky since `node_announcement_2` uses a block-height
based timestamp instead of the unix timestamp used by the legacy message.
There also does not seem to be a big benefit in spreading the 
`node_announcement_2` message quickly since the speed of propagation of the 
`channel_announcement_2` and `channel_update_2` messages will still depend on a
network wide upgrade.

## Specification

### Feature Bits

Unlike the original set of gossip messages, it needs to be explicitly advertised
that nodes are aware of the new gossip messages defined in this document at
least until the whole network has upgraded to only using taproot gossip. We thus
define a new feature bit, `option_taproot_gossip` to be included in the `init`
message as well as the _old_ `node_announcement` message. Note that for all
other feature bits with the `N` and `C` contexts, it can be assumed that those
contexts will switch to the new `node_announcement_2` and
`channel_announcement_2` messages for all feature bits _except_ for
`option_taproot_gossip` which only makes sense in the context of the old
`node_announcement` message and not the new `node_announcement_v2` message.
The `option_taproot` feature bit can also be implied when using the new taproot 
gossip messages. 

| Bits  | Name                    | Description                                       | Context | Dependencies     |
|-------|-------------------------|---------------------------------------------------|---------|------------------|
| 32/33 | `option_taproot_gossip` | Nodes that understand the taproot gossip messages | IN      | `option_taproot` | 

A node can be assumed to understand the new gossip v2 messages if:

- They advertise `option_taproot_gossip` in a legacy `node_announcement` message
  OR
- They have broadcast one of the new gossip messages defined in this document.

Advertisement of the `option_schnorr_gossip` feature bit should be taken to 
mean:

- The node understands and is able to forward all the taproot gossip messages.
- The node may be willing to open an announced taproot channel.

### `open_channel` Extra Requirements

These extra requirements only apply if the `option_taproot` channel type is set
in the `open_channel` message.

The sender:

  - if `option_schnorr_gossip` was negotiated: 
    - MAY set the `announce_channel` bit in `channel_flags`
  - otherwise:
    - MUST NOT set the `announce_channel` bit.

The receiving node MUST fail the channel if:

  - if `option_schnorr_gossip` was not negotiated and the `announce_channel` 
    bit in the `channel_flags` was set.

### `channel_ready` Extensions

These extensions only apply if the `option_taproot` channel type was set in the
`open_channel` message along with the `announce_channel` channel flag.

1. `tlv_stream`: `channel_ready_tlvs`
2. types:
   1. type: 0 (`announcement_node_pubnonce`)
   2. data:
      * [`66*byte`: `public_nonce`]
   3. type: 1 (`announcement_bitcoin_pubnonce`)
   4. data:
      * [`66*byte`: `public_nonce`]
   
### Requirements

The sender: 

  - MUST set the `announcement_node_pubnonce` and 
    `announcement_bitcoin_pubnonce` tlv fields in `channel_ready`.
  - SHOULD use the MuSig2 `NonceGen` algorithm to generate two unique secret 
    nonces and use these to determine the corresponding public nonces: 
    `announcement_node_pubnonce` and `announcement_bitcoin_pubnonce`.
  
The recipient: 

  - MUST fail the channel if: 
    - if the `announcement_node_pubnonce` or `announcement_bitcoin_pubnonce` tlv 
      fields are missing.
    - if the `announcement_node_pubnonce` and `announcement_bitcoin_pubnonce` 
      are equal.
    
### Rationale

The final signature included in the `channel_announcement_v2` message will be a
single Schnorr signature that needs to be valid for a public key derived by 
aggregating all the `node_ID` and `bitcoin_key` public keys. The signature for 
this key will thus be created by aggregating (via MuSig2) four partial 
signatures. One for each of the keys: `node_ID_1`, `node_ID_2`, `bitcoin_key_1`
and `bitcoin_key_2`. A nonce will need to be generated and exchanged for each
of the keys and so each node will need to send the other node two nonces. The
`announcement_node_nonce` is for `node_ID_x` and the`announcement_bitcoin_nonce`
is for `bitcoin_key_x`.

Since the channel can only be announced once the `channel_ready` messages have 
been exchanged and since it is generally preferred to keep nonce exchanges as 
Just In Time as possible, the nonces are exchanged via the `channel_ready` 
message.

## The `announcement_signatures_2` Message

Like the legacy `announcement_signatures` message, this is a direct message
between the two endpoints of a channel and serves as an opt-in mechanism to
allow the announcement of the channel to the rest of the network. 

1. type: xxx (`announcement_signatures_2`)
2. data (tlv_stream):
   1. type: 0 (`channel_id`)
   2. data:
       * [`channel_id`:`channel_id`]
   3. type: 1 (`short_channel_id`)
   4. data:
       * [`short_channel_id`:`short_channel_id`]
   5. type: 2 (`partial_signature`)
   6. data:
       * [`partial_signature`:`partial_signature`]

### Requirements

The requirements are similar to the ones defined for the legacy 
`announcement_signatures`. The below requirements assume that the 
`option_taproot` channel type was set in `open_channel`.

A node:
- if the `open_channel` message has the `announce_channel` bit set AND a 
  `shutdown` message has not been sent:
  - MUST send the `announcement_signatures_2` message.
    - MUST NOT send `announcement_signatures_2` messages until `channel_ready` 
      has been sent and received AND the funding transaction has at least six 
      confirmations.
  - MUST set the `partial_signature` field to the 32-byte `s` value of the
    partial signature calculated as described
    in [Partial Signature Calculation](#partial-signature-calculation). The
    message, `m` to be signed is the double-SHA256 hash of the serialisation of 
    the `channel_announcement_2` message excluding the `announcement_sig` field.
- otherwise:
    - MUST NOT send the `announcement_signatures_2` message.
- upon reconnection (once the above timing requirements have been met):
    - MUST respond to the first `announcement_signatures_2` message with its own
      `announcement_signatures_2` message.
    - if it has NOT received an `announcement_signatures_2` message:
        - SHOULD retransmit the `announcement_signatures_2` message.

A recipient node:
- if the `short_channel_id` is NOT correct:
    - SHOULD send a `warning` and close the connection, or send an
      `error` and fail the channel.
- if the `partial_signature` is NOT valid as
  per [Partial Signature Verification](#partial-signature-verification):
    - MAY send a `warning` and close the connection, or send an
      `error` and fail the channel.
- if it has sent AND received a valid `announcement_signatures_2` message:
    - SHOULD queue the `channel_announcement_2` message for its peers.
- if it has not sent `channel_ready`:
    - MAY send a `warning` and close the connection, or send an `error` and fail
      the channel.

### Rationale

The message contains the necessary partial signature, by the sender, that the
recipient will be able to combine with their own partial signature to construct
the signature to put in the `channel_announcement_v2` message. Unlike the legacy
`announcement_signatures` message, `announcement_signatures_v2` only has one
signature field. This field is a MuSig2 partial signature which is the
aggregation of the two signatures that the sender would have created (one for
`bitcoin_key_x` and another for `node_ID_x`).

## The `channel_announcement_2` Message

This gossip message contains ownership information regarding a taproot channel. 
It ties each on-chain Bitcoin key that makes up the taproot output key to the 
associated Lightning node key, and vice-versa. The channel is not practically 
usable until at least one side has announced its fee levels and expiry, using 
`channel_update_2`.

See [Taproot Channel Proof and Verification](#taproot-channel-proof-and-verification)
for more information regarding the requirements for proving the existence of a
channel.

1. type: xxx (`channel_announcement_2`)
2. data: (tlv_stream):
   1. type: 0 (`announcement_sig`)
   2. data:
        * [`schnorr_signature`:`schnorr_signature`]
   3. type: 1 (`features`)
   4. data:
        * [`...*byte`: `features`]
   5. type: 2 (`chain_hash`)
   6. data:
        * [`chain_hash`:`chain_hash`]
   7. type: 3 (`short_channel_id`)
   8. data:
        * [`short_channel_id`:`short_channel_id`]
   9. type: 4 (`node_id_1`)
   10. data:
        * [`point`:`point`]
   11. type: 5 (`node_id_2`)
   12. data:
       * [`point`:`point`]
   13. type: 6 (`bitcoin_key_1`)
   14. data:
       * [`point`:`point`]
   15. type: 7 (`bitcoin_key_2`)
   16. data:
       * [`point`:`point`]

### Requirements

The origin node:

- MUST set `chain_hash` to the 32-byte hash that uniquely identifies the chain
  that the channel was opened within:
    - for the _Bitcoin blockchain_:
        - MUST set `chain_hash` value (encoded in hex) equal
          to `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`.
- MUST set `short_channel_id` to refer to the confirmed funding transaction, as
  specified in [BOLT #2](02-peer-protocol.md#the-channel_ready-message).
    - Note: the corresponding output MUST be a P2TR, as described
      in [Taproot Channel Proof and Verification](#taproot-channel-proof-and-verification).
- MUST set `node_id_1` and `node_id_2` to the public keys of the two nodes
  operating the channel, such that `node_id_1` is the lexicographically-lesser
  of the two compressed keys sorted in ascending lexicographic order.
- MUST set `bitcoin_key_1` and `bitcoin_key_2` to `node_id_1` and `node_id_2`'s
  respective `funding_pubkey`s.
- MUST set `features` based on what features were negotiated for this channel,
  according to [BOLT #9](09-features.md#assigned-features-flags)
- MUST set `schnorr_signature` to a valid signatures of the message `m` (the
  double-SHA256 hash of the serialised message excluding the `announcement_sig`
  field). This should be calculated by passing the partial signatures sent and
  received during the `announcement_signatures_2` exchange to the 
  [MuSig2 PartialSigAgg][musig-partial-sig-agg] function.

The sender/rebroadcaster:
- MUST ONLY send this message on to nodes that have the `option_taproot_gossip`
  feature bit in their `init` message.

The receiving node:

- MUST verify the integrity AND authenticity of the message by verifying the
  `announcement_sig` as per [BIP 340][bip-340-verify].
- if there is an unknown even bit in the `features` field:
    - MUST NOT attempt to route messages through the channel.
- if the `short_channel_id`'s output does NOT correspond to a P2WSH (using
  `bitcoin_key_1` and `bitcoin_key_2`, as specified in
  [BOLT #3](03-transactions.md#funding-transaction-output)) OR the output is
  spent:
    - MUST ignore the message.
- if the specified `chain_hash` is unknown to the receiver:
    - MUST ignore the message.
- otherwise:
    - if `announcement_sig` is invalid OR NOT correct:
        - SHOULD send a `warning`.
        - MAY close the connection.
        - MUST ignore the message.
    - otherwise:
        - if `node_id_1` OR `node_id_2` are blacklisted:
            - SHOULD ignore the message.
        - otherwise:
            - if the transaction referred to was NOT previously announced as a
              channel:
                - SHOULD queue the message for rebroadcasting.
                - MAY choose NOT to for messages longer than the minimum
                  expected length.
        - if it has previously received a valid `channel_announcement_2`, for
          the same transaction, in the same block, but for a
          different `node_id_1` or `node_id_2`:
            - SHOULD blacklist the previous message's `node_id_1`
              and `node_id_2`, as well as this `node_id_1` and `node_id_2` AND
              forget any channels connected to them.
        - otherwise:
            - SHOULD store this `channel_announcement_2`.
- once its funding output has been spent OR reorganized out:
    - SHOULD forget a channel after a 12-block delay.

### Rationale

Both nodes are required to sign to indicate they are willing to route other
payments via this channel (i.e. be part of the public network); requiring their
aggregated MuSig2 Schnorr signature proves that they control the channel.

The blacklisting of conflicting nodes disallows multiple different
announcements. Such conflicting announcements should never be broadcast by any
node, as this implies that keys have leaked.

While channels should not be advertised before they are sufficiently deep, the
requirement against rebroadcasting only applies if the transaction has not moved
to a different block.

In order to avoid storing excessively large messages, yet still allow for
reasonable future expansion, nodes are permitted to restrict rebroadcasting
(perhaps statistically).

New channel features are possible in the future: backwards compatible (or
optional) features will have _odd_ feature bits, while incompatible features
will have _even_ feature bits
(["It's OK to be odd!"](00-introduction.md#glossary-and-terminology-guide)).

A delay of 12-blocks is used when forgetting a channel on funding output spend
as to permit a new `channel_announcement_2` to propagate which indicates this
channel was spliced.

## The `node_announcement_2` Message

TODO: complete

1. type: xxx (`node_announcement_2`)
2. data: (tlv_stream):
    1. type: 0 (`announcement_sig`)
    2. data:
        * [`schnorr_signature`:`schnorr_signature`]
    3. type: 1 (`features`)
    4. data:
        * [`...*byte`: `features`]
    5. type: 2 (`timestamp`)
    6. data:
        * [`u32`: `block_height`]
    7. type: 4 (`color`)
    8. data:
        * [`3*byte`:`rgb_color`]
    9. type: 5 (`alias`)
    10. data:
         * [`32*byte`:`alias`]
    11. type: 6 (`ipv4_addrs`)
    12. data: 
         * [`...*byte`: `ipv4_addresses`]  
    13. type: 7 (`ipv6_addrs`)
    14. data:
         * [`...*byte`: `ipv6_addresses`]
    15. type: 8 (`tor_v3_addrs`)
    16. data: 
         * [`...*byte`: `tor_v3_addrs`]
    17. type: 9 (`dns_hostname`)
    18. data:
         * [`...*byte`: `dns_hostname`]

## The `channel_update_2` Message

TODO: complete

1. type: xxx (`channel_update_2`)
2. data: (tlv_stream):
    1. type: 0 (`update_sig`)
    2. data:
        * [`schnorr_signature`:`schnorr_signature`]
    3. type: 2 (`chain_hash`)
    4. data:
        * [`chain_hash`:`chain_hash`]
    5. type: 2 (`timestamp`)
    6. data:
        * [`u32`: `block_height`]
    7. type: 3 (`sequence`)
    8. data:
        * [`u8`, `sequence`]
    9. type: 4 (`message_flags`)
    10. data:
        * [`...*byte`, `message_flags`]
    11. type: 5 (`channel_flags`)
    12. data:
        * [`...*byte`, `channel_flags`]
    13. type: 6 (`cltv_expiry_delta`)
    14. data:
        * [`...*byte`, `cltv_expiry_delta`]
    15. type: 7 (`htlc_minimum_msat`)
    16. type: 8 (`fee_base_msat`)
    17. type: 9 (`fee_prop_millionths`)
    18. type: 10 (`htlc_max_msat`)

# Appendix A: Algorithms

This section fleshes out the algorithms that the peers will need in order to
construct and verify the partial signature that will be included in the
`announcement_signatures_2` messages.

### Partial Signature Calculation

When both nodes have exchanged `channel_ready` then they will each have the
following information:

- `node_1` will know:
    - `bitcoin_priv_key_1`, `node_ID_priv_key_1`,
    - `bitcoin_key_2`,
    - `node_ID_1`,
    - `announcement_node_secnonce_1` and `announcement_node_pubnonce_1`,
    - `announcement_bitcoin_secnonce_1` and `announcement_bitcoin_pubnonce_1`,
    - `announcement_node_pubnonce_2`,
    - `announcement_bitcoin_pubnonce_2`,

- `node_2` will know:
    - `bitcoin_priv_key_2`, `node_ID_priv_key_2`,
    - `bitcoin_key_1`,
    - `node_ID_1`,
    - `announcement_node_secnonce_2` and `announcement_node_pubnonce_2`,
    - `announcement_bitcoin_secnonce_2` and `announcement_bitcoin_pubnonce_2`,
    - `announcement_node_pubnonce_1`,
    - `announcement_bitcoin_pubnonce_1`,

With the above information, both nodes can now start calculating the partial
signatures that will be exchanged in the `announcement_signatures_v2` message.

Firstly, the aggregate public key, `P_agg`, that the signature will be valid for
can be calculated as follows:

```
P_agg = Musig2.KeyAgg(Musig2.KeySort(node_ID_1, node_ID_2, bitcoin_key_1, bitcoin_key_2))
```

Next, the aggregate public nonce, `aggnonce`, can be calculated:

```
aggnonce = Musig2.NonceAgg(announcement_node_secnonce_1, announcement_bitcoin_pubnonce_1, announcement_node_secnonce_2, announcement_bitcoin_pubnonce_2)
```

The message, `msg` that the peers will sign is the serialisation of
`channel_announcement_2` _without_ the `announcement_sig` field (i.e. without
type 0)

With all the information mentioned, both peers can now construct the
[`Session Context`][musig-session-ctx] defined by the MuSig2 protocol which is
necessary for as an input to the `Musig2.Sign` algorithm. The following members
of the `Session Context`, which we will call `session_ctx`, can be defined:

- `aggnonce`: `aggnonce`
- `u`: 2
- `pk1..u`: [`node_ID_1`, `node_ID_2`, `bitcoin_key_1`, `bitcoin_key_2`]
- `v`: 0
- `m`: `msg`

Both peers, `node_1` and `node_2` will need to construct two partial signatures.
One for their `bitcoin_key` and one for their `node_ID` and aggregate those.

- `node_1`:
    - calculates a partial signature for `node_ID_1` as follows:
       ```
       partial_sig_node_1 = MuSig2.Sign(announcement_node_secnonce_1, node_ID_priv_key_1, session_ctx) 
       ```
    - calculates a partial signature for `bitcoin_ID_1` as follows:
       ```
       partial_sig_bitcoin_1 = MuSig2.Sign(announcement_bitcoin_secnonce_1, bitcoin_priv_key_1, session_ctx) 
       ```
    - calculates `partial_sig_1` as follows:
       ```
       partial_sig_1 =  MuSig2.PartialSigAgg(partial_sig_node_1, partial_sig_bitcoin_1, session_ctx)
       ```

- `node_2`:
    - calculates a partial signature for `node_ID_2` as follows:
       ```
       partial_sig_node_2 = MuSig2.Sign(announcement_node_secnonce_2, node_ID_priv_key_2, session_ctx) 
       ```
    - calculates a partial signature for `bitcoin_ID_2` as follows:
       ```
       partial_sig_bitcoin_2 = MuSig2.Sign(announcement_bitcoin_secnonce_2, bitcoin_priv_key_2, session_ctx) 
       ```
    - calculates `partial_sig_2` as follows:
       ```
       partial_sig_2 =  MuSig2.PartialSigAgg(partial_sig_node_2, partial_sig_bitcoin_2, session_ctx)
       ```     

Note that since there are no tweaks involved in this MuSig2 signing flow,
signature aggregation is simply the addition of the two signatures:

```
    partial_sig = (partial_sig_node + partial_sig_bitcoin) % n
```

Where `n` is the [secp256k1][secp256k1] curve order.

### Partial Signature Verification

Since the partial signature put in `announcement_signatures_2` is the addition
of two of four signatures required to make up the final MuSig2 signature, the
verification of the partial signature is slightly different from what is
specified in the MuSig2 spec. The slightly adjusted algorithm will be defined
here. The notation used is the same as defined in the [MuSig2][musig-notation]
spec.

The inputs are:

- `partial_sig` sent by the peer in the `announcement_signatures_2` message.
- The `node_ID` and `bitcoin_key` of the peer.
- The `announcement_node_pubnonce` and `announcement_bitcoin_pubnonce` sent by
  the peer in the `channel_ready` message.
- The `session_ctx` as shown
  in [Partial Signature Calculation](#partial-signature-calculation).

Verification steps:

Note that the `GetSessionValues` and `GetSessionKeyAggCoeff` definitions can be
found in the [MuSig2][musig-session-ctx] spec.

- Let `(Q, _, _, b, R, e) = GetSessionValues(session_ctx)`
- Let `s = int(psig); fail if s >= n`
- Let `R_n1 = cpoint(announcement_node_pubnonce[0:33])`
- Let `R_n2 = cpoint(announcement_node_pubnonce[33:66])`
- Let `R_b1 = cpoint(announcement_bitcoin_pubnonce[0:33])`
- Let `R_b2 = cpoint(announcement_bitcoin_pubnonce[33:66])`
- Let `R_1 = R_n1 + R_b1`
- Let `R_2 = b*(R_n2 + R_b2)`
- Let `R_e' = R_1 + R_2`
- Let `R_e = R_e'` if `has_even_y(R)`, otherwise let `R_e = -R_e'`
- Let `P_n = node_ID`
- Let `P_b = bitcoin_key`
- Let `a_n = GetSessionKeyAggCoeff(session_ctx, P_n)`
- Let `a_b = GetSessionKeyAggCoeff(session_ctx, P_b)`
- Let `P = a_n*P_n + a_b*P_b`
- Let `g = 1` if `has_even_y(Q)`, otherwise `let g = -1 mod n`
- Fail if `s*G != R_e + e*g*P`

# Appendix B: Test Vectors

TODO(elle)

# Acknowledgements

The ideas in this document are largely a consolidation and filtering of the
ideas mentioned in the following references:

- [Rusty's initial Gossip v2 proposal][ml-rusty-2019-gossip-v2] where the idea
  of replacing all gossip messages with new ones that use Schnorr signatures was
  first mentioned (in public writing). This post also included the idea of using
  block heights instead of Unix timestamps for timestamp fields as well as the
  idea of moving some optional fields to TLVs.
- [Roasbeef's post][ml-roasbeef-2022-tr-chan-announcement] on Taproot-aware
  Channel Announcements + Proof Verification which expands on the details of how
  taproot channel verification should work.

[bolt-7]: ./07-routing-gossip.md
[open-chan-msg]: ./02-peer-protocol.md#the-open_channel-message
[ml-rusty-2019-gossip-v2]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-July/002065.html
[ml-roasbeef-2022-tr-chan-announcement]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-March/003526.html
[bip-340]: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
[bip-340-verify]: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki#verification
[simple-taproot-chans]: https://github.com/lightning/bolts/pull/995
[musig-keysort]: https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki#key-sorting
[musig-keyagg]: https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki#key-aggregation
[musig-signing]: https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki#signing
[bip86-tweak]: https://github.com/bitcoin/bips/blob/master/bip-0086.mediawiki#address-derivation
[bip-musig2]: https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki
[musig-session-ctx]: https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki#session-context
[musig-notation]: https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki#notation
[musig-partial-sig-agg]: https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki#partial-signature-aggregation
[secp256k1]: https://www.secg.org/sec2-v2.pdf
