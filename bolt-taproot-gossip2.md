# Extension BOLT XX: Taproot Gossip

# Table of Contents

* [Aim](#aim) 
** [Overview](#overview) 
* [Terminology](#terminology) 
* [Type Definitions](#type-definitions) 
** [TLV Based Messages](#tlv-based-messages) 
* [Taproot Channel Announcement Proof and Verification](#taproot-channel-announcement-proof-and-verification)
* [Future iteration of Taproot gossip: UTXO leveraging](#future-iteration-of-taproot-gossip-utxo-leveraging)
** [Block-height fields](#block-height-fields)
** [Bootstrapping Taproot Gossip](#bootstrapping-taproot-gossip)
** [Specification](#specification)
*    * [Feature Bits](#feature-bit)
*    * [Feature Bit Contexts](#feature-bit-contexts)
*    * [`open_channel` Extra Requirements](#openchannel-extra-requirements)
*    * [`channel_ready` Extensions](#channelready-extensions)
*    * [The `announcement_signatures_2` message](#the-announcementsignatures2-message)
*    * [The `channel_announcement_2` message](#the-channelannouncement2-message)
*    * [The `node_announcement_2` message](#the-nodeannouncement2-message)
*    * [The `channel_update_2` message](#the-channelupdate2-message)
** [Appendix A: Algorithms](#appendix-a-algorithms)
*    * [Partial Signature Calculation](#partial-signature-calculation)
*    * [Partial Signature Verification](#partial-signature-verification)
** [Appendix B: Test Vectors](#appendix-b-test-vectors)
** [Acknowledgements](#acknowledgements)

## Aim

This document aims to update the gossip protocol defined in [BOLT 7][bolt-7] to
allow for advertisement and verification of taproot channels. An entirely new
set of gossip messages are defined that use [BIP-340][bip-340] signatures and
that use a mostly TLV based structure.

## Overview

TODO(convert to paragraph):

- legacy gossip included proof of entire underlying P2SH script so that it could 
  be shown that the bitcoin keys and node keys commit to each other. This is 
  updated now with taproot channels so that only proof of ownership is required
  and the underlying script no longer needs to be provided. The channel peers 
  instead need only provide a partial signature for the resulting output key. 
- 

The initial version of the Lightning Network gossip protocol as defined in
[BOLT 7][bolt-7] was designed around P2WSH funding transactions. For these
channels, the `channel_announcement` message is used to advertise the channel to
the rest of the network. Nodes in the network use the content of this message to
prove that the channel is sufficiently bound to the Lightning Network context
by being provided enough information to prove that the script is a 2-of-2
multi-sig and that it is owned by the nodes advertising the channel. This
ownership proof is done by including signatures in the `channel_announcement`
from both the node ID keys along with the bitcoin keys used in the P2WSH script.
This proof and verification protocol is, however, not compatible with SegWit V1
(P2TR) outputs and so cannot be used to advertise the channels defined in the
[Simple Taproot Channel][simple-taproot-chans] proposal. This document thus aims
to define an updated gossip protocol that will allow nodes to both advertise and
verify taproot channels. This part of the update affects the
`announcement_signatures` and `channel_announcement` messages.

The opportunity is also taken to rework the `node_announcement` and
`channel_update` messages to take advantage of [BIP-340][bip-340] signatures and
TLV fields. Timestamp fields are also updated to be block heights instead of
Unix timestamps.

## Terminology

- Collectively, the set of new gossip messages will be referred to as
  "Taproot gossip".
- `node_1` and `node_2` refer to the two parties involved in opening a channel.
- `node_ID_1` and `node_ID_2` respectively refer to the public keys that
  `node_1` and `node_2` use to identify their nodes in the network.
- `bitcoin_key_1` and `bitcoin_key_2` respectively refer to the public keys
  that `node_1` and `node_2` will use for the specific channel being opened.
  The funding transaction's output will be derived from these two keys.

## Type Definitions

* `bip340_sig`: a 64-byte bitcoin Elliptic Curve Schnorr signature as per 
  [BIP-340][bip-340].
* `boolean_tlv`: a zero length TLV record. If the TLV is present then `true` is 
   implied, otherwise `false` is implied.
* `partial_signature`: a 32-byte partial MuSig2 signature as defined
  in [BIP 327][bip-327].
* `public_nonce`: a 66-byte public nonce as defined in [BIP 327][bip-327].
* `utf8`: a byte as part of a UTF-8 string. A writer MUST ensure an array of
  these is a valid UTF-8 string, a reader MAY reject any messages containing an
  array of these which is not a valid UTF-8 string.

## TLV Based Messages

## TLV Definitions

## Taproot Channel Announcement Proof and Verification

In legacy P2WSH channels, verifiers of a channel would use the two bitcoin keys
advertised in the `channel_announcement` to reconstruct the resulting P2WSH
output and would then check that it matches the output found at the given SCID.

With advertised Taproot channels, however, we want to allow a more flexible and
private construction of the underlying script of the UTXO. The verifier
therefore only needs to check that the channel peers do in-fact own the UTXO
they are advertising. This is verified by checking that the signature advertised
in the `channel_announcement_2` message is valid for the key produced by doing
a MuSig2 key aggregation of `node_id_1`, `node_id_2` and `taproot_output_key`
which is the output key found on-chain located by the provided
`short_channel_id`. Even though the verification of such a signature is possible
today, producing this signature would require nested MuSig2 which is currently
not yet proven to be secure and is not yet specified. Therefore, in the
meantime, verifiers should also allow for the case where the signers of the
message were not using nested MuSig2 and instead used 4-of-4 MuSig2 for the
creation of the signature.

### Verification of a Taproot channel announcement

Taproot channel funding transaction outputs will be SegWit V1 (P2TR) outputs of
the following form:

```
OP_1 <taproot_output_key> 
```

The `channel_announcement_2` will then have the following mandatory and optional
fields:

- Mandatory:
    - `node_id_1`: a `point`
    - `node_id_2`: a `point`
    - `short_channel_id`: a `short_channel_id`
- Optional:
    - `bitcoin_key_1`: a `point`
    - `bitcoin_key_2`: a `point`
    - `merkle_root_hash`: 32 byte hash

The `short_channel_id` can be used to find the `taproot_output_key` on-chain.

#### 3-of-3 Verification

If any of the optional `bitcoin_key_*` fields are missing, then the public key
that the signature should be verified against should be constructed as follows:

```
P_agg = MuSig2.KeyAgg(MuSig2.KeySort(`node_id_1`, `node_id_2`, `taproot_output_key`))
```

The signature in the `channel_announcement_2` message should then be verified
against `P_agg`.

#### 4-of-4 Verification

If both `bitcoin_key_*` fields are present, however, then the signature
verification should be done as follows:

1. Calculate the MuSig2 aggregation of the bitcoin keys
   ```
   P_btc_agg = MuSig2.KeyAgg(MuSig2.KeySort(`bitcoin_key_1`, `bitcoin_key_2`))
   ```
2. Calculate the output key
   ```
   P_output = P_btc_agg + tagged_hash("TapTweak", P_btc_agg || merkle_root_hash)
   ```
3. Verify that `P_output` is equal to the `taproot_output_key` found on-chain
4. Now that it has been proven that the two bitcoin keys do materialise in the
   on-chain UTXO, the following aggregate key can be calculated:
   ```
   P_agg = MuSig2.KeyAgg(MuSig2.KeySort(`node_id_1`, `node_id_2`, `bitcoin_key_1`, `bitcoin_key_2`))
   ```
5. The signature in the `channel_announcement_2` message should then be verified
   against `P_agg`.

### Signing a Taproot channel announcement

Since verifiers of the `channel_announcement_2` message will be able to verify
the 3-of-3 MuSig2 case, channel peers can use any protocol they wish in order
to create the necessary aggregate partial signature for the
`taproot_output_key`. This document does, however, put forth a suggested
protocol for constructing a valid `channel_announcement_2`. Since
a nested MuSig2 protocol has not yet been specified as of the writing of this
document, the suggested protocol for the time being is a 4-of-4 MuSig2
construction. The details of this construction are defined in the
`announcement_signature_2` and `channel_announcement_2` definitions.

## Future iteration of Taproot gossip: UTXO leveraging

It is important to keep this protocol flexible enough to allow for future
privacy extensions and to therefore not make any changes that would block the
path to the future extension. One such extension is UTXO leveraging.

Currently, in order to advertise a channel to the network, a UTXO must be
announced. This effectively ties the identities of the node's advertising the
channel to the given UTXO. In a future iteration of Lightning Network gossip, it
would be beneficial to decouple channel announcements from UTXOs. One
possibility would be to have some `channel_announcement_2` message contain
UTXO proofs and other's that do not. If a node creates a
`channel_announcement_2` that does not have a UTXO proof, then verifiers should
look at previous channel announcements from the node and use those to decide
if the node has enough "allowance" to advertise an announcement that does not
have a UTXO proof. In such a construction the SCID advertised in the 
`channel_annoucement_2` would need to be unique for the each node and for the 
node pair. 

## Block-height fields

## Bootstrapping Taproot Gossip

## Specification

### Feature Bit

A new feature bit, `option_taproot_gossip`, is introduced. Nodes can use this
feature bit in the `init` and _legacy_ `node_announcement` messages to advertise
that they understand the new set of taproot gossip messages and that they would
therefore be able to open an announced Taproot channel.

| Bits  | Name                    | Description                                                                         | Context | Dependencies     |
|-------|-------------------------|-------------------------------------------------------------------------------------|---------|------------------|
| 32/33 | `option_taproot_gossip` | Node understands taproot gossip messages and is able to advertise Taproot channels  | IN      | `option_taproot` | 

A node can be assumed to understand the taproot gossip messages and hence might
be willing to open an announced Taproot channel if:

- It advertises `option_taproot_gossip` in the `init` or legacy
  `node_announcement` messages
  OR
- It has broadcast one of the new gossip messages defined in this document.

### Feature Bit Contexts

For all feature bits other than `option_taproot_gossip` and `option_taproot`
defined in [Bolt 9][bolt-9-features] with the `N` and `C` contexts, it can be
assumed that those contexts will now refer to the new `node_announcement_2` and
`channel_announcement_2` messages. The `option_taproot_gossip` and
`option_taproot` feature bits only make sense in the context of the legacy
messages since they can both be implied with taproot gossip messages.


### `open_channel` Extra Requirements

These extra requirements only apply if the `option_taproot` channel type is set
in the `open_channel` message.

The sender:

- if `option_taproot_gossip` was negotiated:
    - MAY set the `announce_channel` bit in `channel_flags`
- otherwise:
    - MUST NOT set the `announce_channel` bit.

The receiving node MUST fail the channel if:

- if `option_taproot_gossip` was not negotiated and the `announce_channel`
  bit in the `channel_flags` was set.

### Constructing the funding output.

Can be whatever (ie leave open for anything). But add suggestion for 4-of-4 
mention than when nested MuSig2 is available, then can do 3-of-3 but not 
recommended in the meantime.

### receiver viewing the funding tx on-chain:

Should fail if amount on-chain is _less than_ what was put in open-chan. Does not
have to be equal.

### `channel_ready` Extensions

These extensions only apply if the `option_taproot` channel type was set in the
`open_channel` message along with the `announce_channel` channel flag.

1. `tlv_stream`: `channel_ready_tlvs`
2. types:
    1. type: 0 (`announcement_node_pubnonce`)
    2. data:
        * [`66*byte`: `public_nonce`]
    3. type: 2 (`announcement_bitcoin_pubnonce`)
    4. data:
        * [`66*byte`: `public_nonce`]
       
// TODO: channel_ready re-sent on channel_reestablish with new nonces?


### The `announcement_signatures_2` Message

Like the legacy `announcement_signatures` message, this is a direct message
between the two endpoints of a channel and serves as an opt-in mechanism to
allow the announcement of the channel to the rest of the network.

1. type 260
2. data:
    * [`channel_id`:`channel_id`]
    * [`short_channel_id`:`short_channel_id`]
    * [`partial_signature`:`partial_signature`]

### The `channel_announcement_2` Message

1. type 267
2. data:
    * [`bip340_sig`:`bip340_sig`]
    * [`channel_announcement_2_tlvs`:`tlvs`]

1. type: 0 (`chain_hash`)
2. data:
   * [`chain_hash`:`chain_hash`]
1. type: 2 (`features`)
2. data:
   * [`...*byte`: `features`]
1. type: 4 (`short_channel_id`)
2. data:
   * [`short_channel_id`:`short_channel_id`]
1. type: 6 (`node_id_1`)
2. data:
   * [`point`:`point`]
1. type: 8 (`node_id_2`)
2. data:
  * [`point`:`point`]
1. type: 10 (`bitcoin_key_1`)
2. data:
  * [`point`:`point`]
1. type: 12 (`bitcoin_key_2`)
2. data:
  * [`point`:`point`]
1. type: 14 (`merkle_root_hash`)
2. data:
    * [`32*byte`:`hash`]


### The `node_announcement_2` Message

This gossip message, like the legacy `node_announcement` message, allows a node
to indicate extra data associated with it, in addition to its public key.
To avoid trivial denial of service attacks, nodes not associated with an already
known channel (legacy or taproot) are ignored.

Unlike the legacy `node_announcement` message, this message makes use of a
Schnorr signature instead of an ECDSA one. This will allow nodes to be backed
by multiple keys since MuSig2 can be used to construct the single signature.

1. type 269
2. data:
   * [`bip340_sig`:`bip340_sig`]
   * [`node_announcement_2_tlvs`:`tlvs`]

1. `tlv_stream`: `node_announcement_2_tlvs`
2. types:
   1. type: 0 (`features`)
   2. data:
       * [`...*byte`: `features`]
   1. type: 2 (`block_height`)
   2. data:
       * [`u32`: `block_height`]
   1. type: 4 (`node_id`)
   2. data:
       * [`point`:`node_id`]
   1. type: 1 (`color`)
   2. data:
       * [`rgb_color`:`rgb_color`]
   1. type: 3 (`alias`)
   2. data:
       * [`...*utf8`:`alias`]
   1. type: 5 (`ipv4_addrs`)
   2. data:
       * [`...*ipv4_addr`: `ipv4_addresses`]
   1. type: 7 (`ipv6_addrs`)
   2. data:
       * [`...*ipv6_addr`: `ipv6_addresses`]
   1. type: 9 (`tor_v3_addrs`)
   2. data:
       * [`...*tor_v3_addr`: `tor_v3_addresses`]
   1. type: 11 (`dns_hostnames`)
   2. data:
       * [`...*dns_hostname`: `dns_hostnames`]

The following subtypes are defined:

1. subtype: `rgb_color`
2. data:
    * [`byte`:`red`]
    * [`byte`:`green`]
    * [`byte`:`blue`]

1. subtype: `ipv4_addr`
2. data:
    * [`u32`:`addr`]
    * [`u16`:`port`]

1. subtype: `ipv6_addr`
2. data:
    * [`16*byte`:`addr`]
    * [`u16`:`port`]

1. subtype: `tor_v3_addr`
2. data:
    * [`35*utf8`:`onion_addr`]
    * [`u16`:`port`]

1. subtype: `dns_hostname`
2. data:
    * [`u16`:`len`]
    * [`len*utf8`:`hostname`]
    * [`u16`:`port`]

#### Message Field Descriptions

- `bip340_sig` is the [BIP340][bip-340] signature for the `node_id` key. The 
  message to be signed is `MsgHash("node_announcement_2", "bip340_sig", m)` 
  where `m` is the serialised TLV stream (see the 
  [`MsgHash`](#signature-message-construction) definition).
- `features` is a bit vector with bits set according to [BOLT #9](09-features.md#assigned-features-flags)
- `block_height` allows for ordering or messages in the case of multiple 
  announcements and also allows for natural rate limiting of 
  `node_announcement_2` messages. 
- `node_id` is the public key associated with this node. It must match a node ID
  that has previously been announced in the `node_id_1` or `node_id_2` fields of 
  a `channel_announcement` or `channel_announcement_2` message for a channel 
  that is still open.
- `rgb_color` is an optional field that a node may use to assign itself a color. 
- `alias` is an optional field that a node may use to assign itself an alias 
  that can then be used for a nicer UX on intelligence services. If this field
  is set, then it MUST be 32 utf8 characters or less.
- `ipv4_addr` is an ipv4 address and port. 
- `ipv6_addr` is an ipv6 address and port.
- `tor_v3_addr` is a Tor version 3 ([prop224]) onion service address;
  Its `onion_addr` encodes:
  `[32:32_byte_ed25519_pubkey] || [2:checksum] || [1:version]`, where
  `checksum = sha3(".onion checksum" | pubkey || version)[:2]`.
- `dns_hostname` is a DNS hostname. The `hostname` MUST be ASCII characters. 
  Non-ASCII characters MUST be encoded using [Punycode][punycode]. The length of
  the `hostname` cannot exceed 255 bytes.

#### Requirements

The sender:

- MUST set TLV fields 0, 2 and 4. 
- MUST set `block_height` to be greater than that of any previous
  `node_announcement_2` it has previously created.

### The `channel_update_2` Message

1. type 271
2. data:
   * [`bip340_sig`:`bip340_sig`]
   * [`channel_update_2_tlvs`:`tlvs`]
 
1. `tlv_stream`: `channel_update_2_tlvs`
2. types:
   1. type: 0 (`chain_hash`)
   2. data:
       * [`chain_hash`:`chain_hash`]
   1. type: 2 (`short_channel_id`)
   2. data:
       * [`short_channel_id`:`short_channel_id`]
   1. type: 4 (`block_height`)
   2. data:
       * [`u32`:`block_height`]
   1. type: 6 (`disable`)
   2. data:
       * [`boolean_tlv`:`disable`]
   1. type: 8 (`direction`)
   2. data: 
       * [`boolean_tlv`:`direction`]
   1. type: 10 (`cltv_expiry_delta`)
   2. data:
       * [`u16`:`cltv_expiry_delta`]
   1. type: 12 (`htlc_minimum_msat`)
   2. data:
       * [`u64`:`htlc_minimum_msat`]
   1. type: 14 (`htlc_maximum_msat`)
   2. data:
       * [`u64`:`htlc_maximum_msat`]
   1. type: 16 (`fee_base_msat`)
   2. data:
       * [`u32`:`fee_base_msat`]
   1. type: 18 (`fee_proportional_millionths`)
   2. data:
       * [`u32`:`fee_proportional_millionths`]


#### Message Field Descriptions

- The `chain_hash` is used to identify the blockchain containing the channel
  being referred to.
- `short_channel_id` is the unique identifier of the channel. If the channel is
  unannounced, then this may be set to an agreed upon alias. 
- `block_height` is the timestamp associated with the message. A node may not 
  send two `channel_update` messages with the same `block_height`. The
  `block_height` must also be greater than or equal to the block height 
  indicated by the `short_channel_id` used in the `channel_announcement` and 
  must not be less than current best block height minus 2016 (~2 weeks of 
  blocks).
- Setting the `disable` bit indicates that node would like to advertise to the 
  network that the channel is disabled and so should not be used for routing.
  This might be due to temporary unavailability (e.g. due to a loss of
  connectivity) OR permanent unavailability (e.g. prior to an on-chain
  settlement).
- The `direction` bit is used to indicate which node in the channel node pair
  has created and signed this message. If the node was `node_id_1` in the 
  `channel_announcment` message then the `direction` is 0 (false) otherwise
  it is 1 (true) and the node is `node_id_2` in the `channel_announcement` 
  message.
- `cltv_expiry_delta` is the number of blocks that the node will subtract from 
  an incoming HTLC's `cltv_expiry`.
- `htlc_minimum_msat` is the minimum HTLC value (in millisatoshi) that the 
  channel peer will accept. 
- `htlc_maximum_msat` is the maximum value the node will send through this 
  channel for a single HTLC. 
  - MUST be less than or equal to the channel capacity 
  - MUST be less than or equal to the `max_htlc_value_in_flight_msat` it 
    received from the peer in the `open_channel` message.
- `fee_base_msat` is the base fee (in millisatoshis) that the node will charge
  for an HTLC.
- `fee_proportional_millionths` is the amount (in millionths of a satoshi) that
  node will charge per transferred satoshi.

#### TLV Defaults

For types 0, 6, 8, 10, 12, 14, 16 and 18, the following defaults apply if the
TLV is not present in the message:

| `channel_update_2` TLV Type        | Default Value                                                      | Comment                                                          |
|------------------------------------|--------------------------------------------------------------------|------------------------------------------------------------------|
| 0  (`chain_hash`)                  | `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000` | The hash of the genesis block of the mainnet Bitcoin blockchain. | 
| 6  (`disable`)                     | false                                                              |                                                                  | 
| 8  (`direction`)                   | false                                                              |                                                                  | 
| 10 (`cltv_expiry_delta`)           | 80                                                                 |                                                                  | 
| 12 (`htlc_minimum_msat`)           | 1                                                                  |                                                                  | 
| 14 (`htlc_maximum_msat`)           | floor(channel capacity / 2)                                        |                                                                  | 
| 16 (`fee_base_msat`)               | 1000                                                               |                                                                  | 
| 18 (`fee_proportional_millionths`) | 1                                                                  |                                                                  |

#### Requirements

The origin node: 

- For all fields with defined defaults:
  - SHOULD not include the field in the TLV stream if the default value is 
    desired.
- MUST use the `channel_update_2` message to communicate channel parameters of a
  Taproot channel. 
- MAY use the `channel_update_2` message to communicate channel parameters of a
  legacy (P2SH) channel. 
- MUST NOT send a created `channel_update_2` before `channel_ready` has been 
  received.
- For an unannounced channel (i.e. one where the `announce_channel` bit was not 
  set in `open_channel`):
  - MAY create a `channel_update_2` to communicate the channel parameters to the
    channel peer.
  - MUST set the `short_channel_id` to either an `alias` it has received from 
    the peer, or the real channel `short_channel_id`.
  - MUST NOT forward such a `channel_update_2` to other peers, for privacy
    reasons.
- For announced channels:
  - MUST set `chain_hash` AND `short_channel_id` to match the 32-byte hash AND
    8-byte channel ID that uniquely identifies the channel specified in the
    `channel_announcement_2` message.
- MUST set `bip340_sig` to a valid [BIP340][bip-340] signature for its own
  `node_id` key. The message to be signed is
  `MsgHash("channel_update_2", "bip340_sig", m)` where `m` is the serialised
  TLV stream of the `channel_update` (see the 
  [`MsgHash`](#signature-message-construction) definition).
- SHOULD NOT create redundant `channel_update_2`s.
- If it creates a new `channel_update_2` with updated channel parameters:
  - SHOULD keep accepting the previous channel parameters for 2 blocks.

The receiving node: 

- If the `short_channel_id` does NOT match a previous `channel_announcement_2` 
  or `channel_announcement`, OR if the channel has been closed in the meantime:
    - MUST ignore `channel_update_2`s that do NOT correspond to one of its own
      channels.
- SHOULD accept `channel_update_2`s for its own channels (even if non-public),
  in order to learn the associated origin nodes' forwarding parameters.
- if `bip340_sig` is NOT a valid [BIP340][bip-340] signature (using `node_id`
  over the message):
    - SHOULD send a `warning` and close the connection.
    - MUST NOT process the message further.

#### Rationale

- An even type is used so that best-effort propagation can be used.

For both tap chans & legacy chans. 


# Appendix A: Algorithms

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

The message, `msg`, that the peers will sign is 
`MsgHash("channel_announcement", "announcement_sig", m)` where `m` is the
serialisation of the `channel_announcement_2` TLV stream (see the
[`MsgHash`](#signature-message-construction) definition).

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

### Signature Message Construction

The following `MsgHash` function is defined which can be used to construct a
32-byte message that can be used as a valid input to the [BIP-340][bip-340]
signing and verification algorithms.

_MsgHash:_
   - _inputs_:
     * `message_name`: UTF-8 string
     * `field_name`: UTF-8 string
     * `message`: byte array

   - Let `tag` = "lightning" || `message_name` || `field_name`
   - return SHA256(SHA256(`tag`) || SHA256(`tag`) || SHA256(`message`))

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

[bolt-9-features]: ./09-features.md#bolt-9-assigned-feature-flags
[bip-340]: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
[bip-327]: https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
[ml-rusty-2019-gossip-v2]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-July/002065.html
[ml-roasbeef-2022-tr-chan-announcement]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-March/003526.html
[prop224]: https://gitweb.torproject.org/torspec.git/tree/proposals/224-rend-spec-ng.txt
[punycode]: https://en.wikipedia.org/wiki/Punycode

