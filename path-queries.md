```
bLIP: TBD
Title: Path Queries
Author: Benjamin Hindman <bhindman@proton.me>
Status: Draft
Created: 2026-07-01
License: CC0
```

## Abstract

To route a payment on the Lightning Network, a sender must find a path to the destination using channels which contain sufficient liquidity and meet certain routing requirements (e.g fees). The current gossip scheme is insufficient to reliably determine a feasible path and inflexible for routing nodes. This bLIP proposes path queries: a small set of optional messages — `query_path`, `reply_path`, and `reject_query_path` — that reduce the informational requirements during pathfinding and allow routers to respond with dynamic policy. By selectively sharing routing information between peers, payment reliability can be scaled to a growing network while preserving channel balance privacy and payment anonymity.

## Copyright

This bLIP is licensed under the CC0 license.

## Motivation

### Graph Dependence

The Lightning Network uses source-based routing, which means the payment sender is responsible for finding a feasible path to a destination. The sender's view of the network graph typically derives from two sources: gossip messages and the responses of previous payment attempts. Both sources have severe limitations, which ultimately constrain routing policy and reduce payment performance.

#### Limitations of the gossip protocol

The gossip protocol is characterized by its ability to quickly and reliably deliver a *finite* amount of information across a large distributed system via propagation. However, because these messages are delivered to every node (potentially multiple times), node performance suffers with a growing *quantity* of data; the more data that's shared, the more network and computational resources are required by each node to process those messages. Therefore, the size and frequency of gossip messages must be constrained. 

Gossip is well-suited for the Bitcoin network because nodes ultimately seek a globally consistent ledger, which requires bounding data growth to the size of a block, at a frequency governed by mining difficulty. *Global consistency* is precisely the onchain scaling constraint that the Lightning Network is designed to break; only the nodes involved in a payment must agree, and only at the time of payment. 

The per-hop routing policy which defines a payment's terms is a set of jointly composed fields determined by dynamic inputs (e.g liquidity). Today's rate limits on gossip restrict the *currency* (i.e freshness) of those terms, which leaves a routing node to publish a coarse approximation: a static policy in a dynamic environment. Gossip's need for wide adoption also restricts policy *expressivity* because messages must be standardized, and extensions with inconsistent adoption produce inconsistent policy. Until payment senders broadly understand a gossiped message, the routing node must resolve multiple interpretations of its policy.

Even when agreed upon by the network, a pricing function is limited in what it can express because HTLCs are priced independently of each other: a bend in the function, for example, is gamed by splitting the payment into smaller parts (MPP) to minimize fees. In short, the global nature of gossip messages is a mismatch to the local nature of a payment. 

#### Finding Liquidity

For a payment to succeed, every channel along the path must have sufficient liquidity. Liquidity is state that is managed between two nodes, and the unpredictability of a channel's liquidity to the rest of the network is referred to as *liquidity uncertainty*. Without any prior knowledge (i.e high uncertainty), the probability a path is feasible declines with the size of the payment and the number of channels used. Today, feasible paths are discovered through the payment delivery mechanism itself: a trial-and-error process whereby liquidity ranges are temporarily narrowed using the results of previous payment attempts. This approach has a number of issues:

1. Pathfinding calculations compensate for uncertainty by favoring shorter paths and higher-capacity channels. Not only does this affect the payment sender who's more likely to pay extra in fees for reliable liquidity, it's also a centralizing force on the network.

2. Trial-and-error is slow:

    a. HTLCs require multiple message exchanges between peers, with each HTLC addition and removal requiring commitment/revocation cycles.

    b. Payments must be attempted serially to avoid delivering the payment more than once.

3. A sender has little ability to predict the outcome of its payment upfront. As a consequence, payment terms (e.g fees) are typically bounded rather than chosen. 

4. Failed payments are a burden to routing nodes in the failing sub-path in the form of locked liquidity, locked HTLC slots, and wasted system resources.

To improve performance for real payments, nodes may choose to 'probe' the channels of routing nodes using fake payments. However, liquidity is dynamic and is always regressing to a state of uncertainty. To be effective, nodes must actively monitor the network. 

Worse yet, each of the above problems compounds as the network grows, with more nodes searching for liquidity amongst a larger set of channels.

## Specification

The feature consists of three optional messages that allow nodes to cooperatively construct paths using request/response exchanges. A `query_path` may be answered by one or more `reply_path` messages or a single `reject_query_path`.

A node advertises support using the `option_path_queries` feature bit (264/265, reserved via [bLIP 2](./blip-0002.md#feature-bits)). All message types are odd, following the "it's ok to be odd" rule, so nodes that do not understand them can safely ignore them. Each message's payload is a TLV stream, allowing optional fields to be added over time without breaking existing parsers.

An `expiry_ms` is a relative duration measured from receipt of the message that carries it. No clock synchronization between peers is assumed; each node measures elapsed time locally. Transit delay therefore extends the effective window slightly, and a node may account for this slack in the durations it sets.

### The `query_path` Message

1. type: 32769 (`query_path`)
2. data:
  * [`query_path_tlvs`:`tlvs`]

1. `tlv_stream`: `query_path_tlvs`
2. types:
    1. type: 0 (`query_id`)
        * data: [`32*byte`:`id`]
    2. type: 2 (`source_node_id`)
        * data: [`point`:`node_id`]
    3. type: 4 (`destination_node_id`)
        * data: [`point`:`node_id`]
    4. type: 6 (`amount_msat`)
        * data: [`tu64`:`amount_msat`]
    5. type: 8 (`expiry_ms`)
        * data: [`tu32`:`expiry_ms`]

The `query_id` is a sender-chosen identifier, which the responding node echoes back. This correlation enables concurrent queries and asynchronous responses.

### The `reply_path` Message

1. type: 32771 (`reply_path`)
2. data:
  * [`reply_path_tlvs`:`tlvs`]

1. `tlv_stream`: `reply_path_tlvs`
2. types:
    1. type: 0 (`query_id`)
        * data: [`32*byte`:`id`]
    2. type: 2 (`path`)
        * data: [`...*part`:`parts`]
    3. type: 4 (`expiry_ms`)
        * data: [`tu32`:`expiry_ms`]

Each `part` is structured as:
  * [`u64`:`part_amount_msat`]
  * [`u16`:`num_hops`]
  * [`num_hops*hop`:`hops`]

Each `hop` is a length-prefixed TLV stream:
  * [`u16`:`len`]
  * [`hop_tlvs`:`tlvs`]

1. `tlv_stream`: `hop_tlvs`
2. types:
    1. type: 2 (`node_id`)
        * data: [`point`:`node_id`]
    2. type: 4 (`short_channel_id`)
        * data: [`short_channel_id`:`scid`]
    3. type: 6 (`fee_base_msat`)
        * data: [`tu32`:`fee_base_msat`]
    4. type: 8 (`fee_proportional_millionths`)
        * data: [`tu32`:`fee_proportional_millionths`]
    5. type: 10 (`cltv_expiry_delta`)
        * data: [`tu16`:`cltv_expiry_delta`]

A `path` consists of one or more `part`s, where each `part` is an independent hop sequence carrying a portion of the amount. The parts of a single `reply_path` are offered as jointly feasible, and their amounts must sum to the queried `amount_msat`. A node may send multiple `reply_path` messages for the same query, each a self-contained alternative; alternatives are not claimed to be jointly feasible with each other.

The `path` value is parsed by consuming `part`s until the value is exhausted. Each `part` is self-delimiting (`part_amount_msat`, `num_hops`, then `num_hops` length-prefixed `hop`s). A `path` whose value cannot be consumed exactly (e.g trailing bytes, truncated `part`) is invalid, and the receiving node MUST ignore the message.

A `hop` expresses the standard policy triple (`fee_base_msat`, `fee_proportional_millionths`, `cltv_expiry_delta`) used by `channel_update`, route hints, and blinded paths. An offered policy is a claim only for the queried `amount_msat`, and only within the reply's `expiry_ms`; a value of 0 claims validity only at the moment of the reply.

### The `reject_query_path` Message

1. type: 32773 (`reject_query_path`)
2. data:
  * [`reject_query_path_tlvs`:`tlvs`]

1. `tlv_stream`: `reject_query_path_tlvs`
2. types:
    1. type: 0 (`query_id`)
        * data: [`32*byte`:`id`]
    2. type: 1 (`reason`)
        * data: [`...*byte`:`reason`]

### Requirements

The origin node sending `query_path`:
  - MUST NOT send `query_path` to a peer that has not signaled `option_path_queries`.
  - MUST set `query_id` to an identifier unique among its outstanding queries to that peer. It SHOULD be chosen at random.
  - MUST set `source_node_id` to the node at which the requested path starts.
  - MUST set `destination_node_id` to the node at which the requested path terminates.
  - MUST set `amount_msat` to the amount, in millisatoshis, to be delivered to `destination_node_id`.
  - MUST set `expiry_ms` to the duration, in milliseconds, that it is interested in replies.
  - if querying on behalf of a query it has itself received:
    - SHOULD reuse the received `query_id`, so downstream nodes can correlate queries belonging to the same payment.
    - if the received `query_id` collides with a different query already outstanding to the same peer, MUST substitute a fresh `query_id` toward that peer, forfeiting downstream joint-feasibility accounting for that branch.
    - SHOULD set `expiry_ms` below the remaining `expiry_ms` of the received query, i.e the received value minus the time elapsed since its receipt.
    - MAY set a fresh `query_id` where cross-query correlation is a privacy concern, forfeiting downstream joint-feasibility accounting.

A node receiving `query_path`:
  - MAY ignore the message.
  - if the query duplicates one it is already servicing or has recently answered (e.g same `source_node_id` and `destination_node_id` within the query's `expiry_ms`), SHOULD answer from local knowledge or reject it, and SHOULD NOT query others on its behalf.
  - MAY respond with a `reject_query_path`:
    - MUST set `query_id` to match the received query.
    - MAY set `reason` to a human-readable explanation for the rejection.
  - MAY respond with one or more `reply_path` messages, each offering a self-contained alternative:
    - MUST set `query_id` to match the received query.
    - MUST include a `path` containing one or more `part`s whose `part_amount_msat` values sum to the queried `amount_msat`.
    - SHOULD only offer parts it believes to be jointly feasible at the time of the reply, accounting for its replies to other outstanding queries bearing the same `query_id`.
    - MUST ensure each part forms a connected route: the first hop's `node_id` is `source_node_id`, each subsequent hop's `node_id` is the counterparty of the previous hop's channel, and the final hop's channel terminates at `destination_node_id`.
    - for each `hop`:
      - MUST set `node_id` to the node expected to forward the payment over `short_channel_id`.
      - MUST set `fee_base_msat`, `fee_proportional_millionths`, and `cltv_expiry_delta` to the policy at which that node is expected to forward.
    - MUST set `expiry_ms` to the duration, in milliseconds, the offer is claimed to remain valid. This is independent of the query's `expiry_ms`: an offer may outlive the query that produced it.
    - SHOULD NOT reply after the query's `expiry_ms` has elapsed.

A node receiving `reply_path` or `reject_query_path`:
  - MUST ignore the message if `query_id` does not correspond to one of its outstanding queries to that peer.
  - MUST NOT treat an offered path as a guarantee of payment success (see [Non-binding replies](#non-binding-replies)).
  - SHOULD use the policy fields of each hop, rather than gossip, when constructing an onion from an offered path.
  - MAY attempt payment over an offered path after its `expiry_ms` has elapsed, though the offer is no longer claimed to be valid.

## Rationale

These messages expand the existing basis of cooperation between nodes. The querying node reduces its dependence on the graph, gaining current and targeted routing information in order to make payments. The responding node gains influence over payment flows through itself; it can offer paths and set routing policy (e.g fees, cltv, expiry) at reply time, per query, and unconstrained by today's gossip limitations. Like the sender, routing nodes are invested in the success rate of payments through their channels, because only successful payments earn fees.

Without disclosing any balances, a `reply_path` resolves liquidity uncertainty at the responder's own channels, since it knows its own balances and can respond accordingly. Whereas a single query draws only on the responder's own balances and its view of the surrounding graph, recursive queries compose the first-hand knowledge from a set of nodes. This nearly eliminates uncertainty along the offered path, though some uncertainty reemerges in the time between the reply and the arrival of an onion. 

Rather than favoring high-capacity hubs as the safest statistical bets (see [Finding Liquidity](#finding-liquidity)), payment senders can shop for the best route. Smaller routing nodes can attract flow by answering accurately with competitive terms, and like senders, they shed their dependence on a synced graph. While large nodes maintain their capital advantages, queries mitigate the information asymmetry that centralizes the network. 

The messages are entirely optional. Nodes can selectively query and selectively respond. A sender who receives no replies simply falls back to the gossiped graph and trial-and-error, so the feature can be adopted incrementally by any set of peers. The protocol specifies the *mechanism* of cooperation while leaving strategy to each node. 

### Just-in-time discovery

Path queries are a low-latency discovery process designed to raise the success probability of real payment attempts. Queries spend no commitment/revocation cycles, may be issued concurrently and answered asynchronously. 

Given one or more offered paths in a time window, payment outcomes can be better understood in advance, making assessments prior to attempt more valuable. Since a sender can compare alternatives before any HTLC is offered, terms can reasonably be chosen rather than simply bounded.

### Lightweight monitoring

Path queries involve *voluntary* disclosure. To the extent its peers cooperate, a node can monitor parts of the network by simply asking. Messages only consume system resources, not channel resources (e.g liquidity, HTLC slots). This makes queries a lightweight, cooperative alternative to probing. Given such an alternative, the payment mechanism can be reserved for payments alone: a failure then reflects a genuine payment failure, rather than a successful measurement. 

Because queries are observable and attributable, abuse is self-limiting: a peer that queries excessively, or appears to be searching a channel's balance, can be rate-limited, ignored, or eventually cut off (see [Denial-of-service risks](#denial-of-service-risks)). Replies are a valuable resource, to be selectively requested. 

### Per-query policy

A query presents a concrete (source, destination, amount) flow, so the responding node can price every component of the payment jointly: the inbound and outbound channels, the amount, and the time-lock are quoted together. Additionally, a node evaluates its policy on demand, one query at a time, to the one who asks.

To illustrate, consider a node N whose channel balance to A is depleted while its balances to B and C hold excess. A payment A→N→B rebalances both channels at once and merits a discount; a payment C→N→B further imbalances the channel with C and merits a surcharge. Both payments involve the same outbound channel and amount, but should be priced differently. Furthermore, not every marginal sat should be charged at the same rate: A→N→B merits its discount only up to the liquidity N wants moved; beyond that, the fee should climb.

Gossip cannot practically express such pricing. A table enumerating all combinations is too large at scale, while a pricing function, such as the one proposed in [inbound routing fees](https://github.com/lightning/blips/pull/18), is limited to linear curves (see [Limitations of the gossip protocol](#limitations-of-the-gossip-protocol)) and cannot price the example above. Path queries avoid these problems by inverting the model. Rather than broadcasting, the pricing function stays local, is evaluated on demand, and its result is read only by the peer who asked. 

### Putting into practice

The proposal outlines a basic set of messages, and it is up to each node to choose its own request and response strategies, including who it wants to talk to (any subset of nodes), what it wants to respond to (e.g minimum amounts) and any rate limits (number of requests and replies/paths). While there are innumerable strategies that may evolve, let's walk through a simple example where all nodes adopt a PEER_ONLY strategy. Under this strategy, nodes only send relevant messages to their direct channel peers.

Payment of 10 from S → R. Channel balances are shown as `left|right` (including `C|B`), each side spendable toward its neighbor:
```
               +-------+       +-------+
    +--15|5----|   A   |--3|7--|   B   |---9|1----+
    |          +-------+       +-------+          |
    |                              |              |
+-------+                         5|5          +-----+
|   S   |          ----------------+           |  R  |
+-------+          |                           +-----+
    |              |                              |
    |          +-------+       +-------+          |
    +--15|5----|   C   |-12|2--|   D   |---6|4----+
               +-------+       +-------+
```

Before attempting the payment, the sender (S) may choose to query any subset of its channel peers. Since this is a larger payment (10), the sender decides to make concurrent queries to both Alice (A) and Carol (C):

- Alice receives a `query_path` message requesting a path from herself to the receiver (R). She sees she does not have the outbound liquidity to Bob (B) to complete the payment, so either responds with a `reject_query_path` with a reason indicating a temporary failure to find a route, or waits — within the query's expiry — for liquidity to become available to respond with a `reply_path`.
- Carol receives a `query_path` requesting a path from herself to the receiver. She has sufficient outbound liquidity through Dave, but before responding to the sender, she decides to query Dave:
    - Dave receives a query from Carol for a path from himself to the receiver. Similar to Alice, Dave has no route available, so responds with a `reject_query_path`.
- Upon discovering insufficient liquidity from D → R, Carol splits the sender's amount and concurrently queries her channel peers Bob and Dave with their respective splits. 
    - Dave receives a new query requesting a path from himself to the receiver, but of a lesser amount (5). Since Dave knows he can route the requested payment, he responds to Carol with the given path and routing details. 
    - Bob receives a new query requesting a path from himself to the receiver for his split amount. Similar to Dave, he knows he can route the payment, so responds to Carol with his routing details. 
- Upon receiving the path details from Bob and Dave, Carol can now confidently assemble an MPP from herself to the receiver. She prepends her own routing details to each and sends the sender a single `reply_path` whose path contains two jointly feasible parts (one through Bob, one through Dave). 
- Upon receiving the `reply_path` from Carol, the sender assesses the payment terms and proceeds with the payment, which succeeds on the first attempt.
  
As shown, `query_path` messages concurrently spread amongst prospective routing nodes until a feasible path is discovered. After receiving a `reply_path`, a node can prepend itself to the path and either back-propagate it to the source or attempt the payment. Each node knows its channel balances and can therefore reduce the liquidity uncertainty for its respective channels. Using their local views, nodes coordinate to find a feasible path. 

While the small example above illustrates the process, it is important to consider the rate at which liquidity uncertainty is reduced; trial-and-error may work for a small network like this, but does not scale to a growing number of nodes.

### Expanding the Protocol

#### Adding message fields 

The messages defined in this proposal are intentionally minimal to communicate the core concept while avoiding additional complexity. That said, optional message fields can be easily deployed to enhance a node's capabilities and to reduce the number of messages between peers. Some examples:

- `query_path` fields:
    - `amount_range` (min/max msat) - expands the query's range of interest. A range needs fewer queries when planning splits, and discloses less about the payment than an exact amount (see [Privacy Implications](#privacy-implications)). The responder may price any sub-range with its own `amount_range` (see below), including partitioning the range into multiple `reply_path` messages.
    - `maximum_fee`, `cltv_expiration` - reduce response messages by providing upfront filters
    - `subscribe` - a flag requesting continuous updates: while the query remains live (bounded by its `expiry_ms` and the responder's own policy), the responding node refreshes or supersedes its offered paths as liquidity and policy change.
    - `fresh_query_ids` - a flag requesting that recursive queries substitute fresh `query_id`s rather than reusing the received one, trading downstream joint-feasibility accounting for reduced cross-query correlation (see [Privacy Implications](#privacy-implications)).
- `reply_path` fields: 
    - `amount_range` (min/max msat) - extends the policy and feasibility claim over a range of amounts rather than a single `amount_msat`, just as `expiry_ms` extends it over time. The querier gains optionality: it can reallocate amounts across its parts, or retry at a different amount, within the range — without a fresh query. Offers that cover ranges also satisfy more future queries, letting routing nodes answer from local knowledge and further reducing amplification (see [Denial-of-service risks](#denial-of-service-risks)).
    - `confidence` (ranged interval) - a score to indicate the expected likelihood of payment delivery; the higher the routing node's confidence, the more a path suggestion behaves like a *quote* for delivery. This may be used by a querying node to weigh the value of a responding node's offered paths, especially if there's a cost for `reply_path`s, such as onion queries as described below. Unlike forwarding endorsements (e.g [HTLC endorsement](https://github.com/lightning/bolts/pull/1071)), this value would be back-propagated in the `reply_path` messages, so does not leak information about the origin.

#### Anonymous queries via Onions

The proposal as described above only supports messages using a direct network connection between any two nodes. This is sufficient when nodes only query their channel peers because it reveals no definitive information about the source of a payment. However, this reduces anonymity when querying remote nodes. To improve anonymity, onions could be used to carry these messages. Without knowing the query source, responding nodes are vulnerable to spam, and consequently, DoS attacks. To defend against spam, nodes can potentially require a small payment for answering anonymous queries.

### Related Work

#### Trampoline 

Path queries have much in common with the [trampoline protocol](https://github.com/lightning/bolts/pull/829); both seek to reduce the dependence of nodes on the graph by leveraging the routing information of peers. However, they differ substantially in approach; trampoline is a payment delivery protocol which outsources routing responsibilities to specialized nodes, while queries are a pathfinding mechanism for nodes to learn about their environment. Given a concrete payment, these approaches may both compete and complement each other.

*When they compete*

The two approaches compete for a concrete payment when a sender needs a full route. When using trampoline, the sender delegates routing to one or more trampoline nodes. Trampoline nodes are responsible for their sub-paths, including their per-hop policies and how to handle failures. Failures can be retried at the start of the failing sub-path, or propagated back to the sender. These decisions are made while the payment is *in-flight*.

In contrast, path queries improve a node's ability to construct regular onions, and can be used *just-in-time* to give it more control over routing decisions; through recursion, their reach extends as far as trampoline's. When a payment originator is able to construct the full path to the payment recipient, it controls the fees and the handling of a failed attempt. 

There are many factors a payment sender should consider when originating a payment, such as payment latency, fees, and privacy, and their approach should reflect their situation. For example:

- A mobile user with a single LSP may choose to exclusively use trampoline: they have little optionality, while in-flight routing avoids added network latency and lets the LSP handle downstream errors. 
- A user with multiple channels sending a larger payment may prefer path queries, which give the sender the ability to shop for the best route.

*When they complement*

Path queries can be used to discover any sub-path, whereby the source and/or destination is a hop in a larger payment, and can therefore be used in conjunction with payment delivery protocols. For example, path queries can be used to construct any of the trampoline sub-paths; the payment sender can find a remote trampoline node, and trampoline nodes can find the next hop. 

#### Probabilistic Pathfinding

Probabilistic pathfinding (e.g [Pickhardt & Richter](https://arxiv.org/abs/2107.05322)) models liquidity uncertainty as a route cost alongside fees: the lower a route's success probability, the higher its effective cost. The model is refined with the results of each attempt, and variants of this approach are deployed in most major implementations (e.g LND's mission control, LDK's scorer, CLN's renepay). Because it learns only through trial-and-error, it inherits the costs described in [Finding Liquidity](#finding-liquidity).

The two approaches are complementary rather than competing: a probabilistic model is a natural way for nodes to select which peers to query, and a subsequent `reply_path` collapses the uncertainty penalty to zero for the offered channels (though the reply is [non-binding](#non-binding-replies)). More generally, a reply is a faster, more informative signal than a failed payment attempt. Routing nodes can then compete on policy (e.g fees) rather than chance. 

#### Receiver-side path sharing

The protocol already contains a precedent for sharing paths. A [Bolt 11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) route hint tells the sender "reach me via these channels, at these fees". It is, in effect, an unsolicited `reply_path` for the final hops, delivered during the invoice exchange, and it, too, is [non-binding](#non-binding-replies). Route hints were created to support unannounced channels, which are never gossiped. A [Bolt 12](https://github.com/lightning/bolts/blob/master/12-offer-encoding.md) blinded path supplies the final sub-path including routing policy in `encrypted_recipient_data` blobs. In both cases, the receiver provides local knowledge of the final sub-path directly to the sender. 

Every route can be viewed as three segments: the first hop(s), which the sender knows from its own channels; the final hop(s), which the receiver knows and already shares via hints and blinded paths; and the middle, which is currently left to chance. The existing protocol resolves the first and last segments by the same principle: let the party who has the information supply the path. Path queries apply that same principle to the remaining segment. These mechanisms compose: the sender's channels supply the first segment, queried nodes the middle, and the receiver's blinded path the last.

### Potential Concerns

#### Privacy Implications

*Privacy of channel balances*

Channel balance privacy and liquidity uncertainty are the same value viewed differently: a sender's certainty *is* a routing node's disclosed balance. In the limit, full knowledge of the network's liquidity is zero balance privacy, while full privacy leaves pathfinding blind. Every mechanism that improves payment reliability therefore spends privacy; the question is not whether it is spent, but how.

Trial-and-error provides HTLC-based evidence to the onion constructor — the sender or, in the case of trampoline, an intermediate node building an onion for its segment — about liquidity ranges (i.e lower and upper bounds) for a set of channels *selected by the onion constructor*. Because a routing node cannot distinguish a probe from a genuine payment attempt, it cannot selectively refuse them; a channel's balance can be extracted by iterating attempts against it. This is not theoretical: probing is commonplace behavior even among honest nodes.

Similar to trial-and-error, a `reply_path` message provides a set of channels that meet the liquidity requirements of a payment. However, there are a few key differences:

1. The query is *observable*: the target sees who is asking.
2. The reply is *selective*: a node chooses who it replies to, what it reveals, and when.
3. The reply is a *claim*, rather than HTLC-based evidence along the route (see [Non-binding replies](#non-binding-replies)).

These differences put the routing node in control of the information it leaks. To the extent queries raise payment success rates, failure itself becomes a signal: a routing node can deter traditional probing simply by penalizing failed attempts, with no protocol change required. And although repeated queries with varying amounts can still be used to search a channel's balance, the target observes the behavior. Over time, bad actors can be rejected, ignored, and their channels closed.

In summary, path queries convert probing's *involuntary* extraction of balances into *voluntary* disclosures governed by each node's response policy. For example, in the PEER_ONLY walkthrough in [Putting into practice](#putting-into-practice), the sender (S) gained no information about the network's liquidity beyond what was sufficient for the final route. Generally speaking, the more channels a node has, the more difficult it is to infer liquidity based on an offered path. 

*Anonymity of payment participants*

While a single query does not tell a routing node about the source of a payment, it does reveal a prospective downstream node. This information, along with correlated queries, can help the queried node reduce the anonymity set of the *query* origin, which may in turn be correlated to a future payment (e.g an HTLC of matching amount arriving shortly after).

That said, the inference is uncertain at both ends. A queried node cannot tell whether its peer is the payment source or is itself servicing an upstream query. Nor can it tell whether the queried path is the full route or only a sub-path.

The sender can act at *query time* by partitioning its route into multiple queries: asking remote nodes for successive sub-paths ([Anonymous queries](#anonymous-queries-via-onions)) ensures that no single responder sees both the source and the destination. Depending on the nature of the payment, the sender can opt out of queries altogether.

While the receiver has no control over the sender's routing process, blinded paths give the receiver the ability to construct the final sub-path at *invoice time*. As a result, the receiver controls the introduction points that queries will eventually name. Furthermore, it can employ queries of its own to build longer, more reliable paths to itself. The longer the blinded path, the more anonymity from the sender and its routing nodes.

*Privacy of payment details*

Queries belonging to the same payment can be linked, revealing its existence and contents (e.g amount, destination) to nodes that never forward it. Reusing a `query_id` across recursive queries (see [Requirements](#requirements)) gives downstream nodes an explicit handle, much as a shared `payment_hash` already links the parts of a multi-part payment. A fresh `query_id` only removes that handle, at the cost of downstream joint-feasibility accounting, while queries can still be linked by content and timing. 

To prevent linkage via the amount, the sender can split the payment (MPP) so that no query names the total: an easy mitigation that even helps delivery, since smaller parts are independently more likely to succeed. Blurring techniques, like rounding or overshooting, can also be used, but cost response accuracy. The destination may likewise be blurred by both the sender and the receiver, as described in *Anonymity of payment participants* above. Timing is less flexible: an offered path is only claimed valid within its `expiry_ms`, so proximity between query and payment is inherent to just-in-time discovery. 

Ultimately, linkage is the price of coordination. The more links a querying node is comfortable with, the more reliable its answers will be. Fortunately, the querying node is in control of what it reveals. 

#### Non-binding replies

A `reply_path` does not contain a commitment; nothing in the protocol obligates a node to successfully forward a payment along an offered path, and even an honest node's liquidity may move between the reply and the attempt. However, while a `channel_update` is an advertisement to the network at large, a `reply_path` is a claim addressed to a specific query. The basis for this claim varies along the path: at the time a node replies, it knows its own channel balances first-hand, while the remaining channels may rest on inference or on the replies of its peers, which are themselves claims. The claim is also bounded in time: a reply's `expiry_ms` states how long the offer is claimed to hold, so a failure within that window can be attributed to the responder.

Although replies are non-binding, routing nodes are persistent, capital-bonded identities with a strong reputational incentive to provide reliable paths. The terms of a claim (e.g fees, expiry window) are parameters a routing node competes on, and it must balance the competitiveness of its terms with its ability to deliver on them. 

#### Denial-of-service risks

Because a queried node may itself query others before responding, a single inbound query can fan out into many downstream messages, and a cycle among nodes could regenerate them indefinitely. Local knowledge bounds the loops: a node asked about a flow already covered by an outstanding query or an existing `reply_path` answers from what it holds rather than querying further. A cycle therefore terminates at the first node it revisits.

Reputation and expiry limit the remaining amplification. Every query a node originates is charged against its own peer relationships: amplified junk spends the node's own query budget, which is acquired through real channels and bonded capital. And every query carries an `expiry_ms`, which recursive queries propagate at decreasing values, so any cascade dies out by the original deadline. 

Each of these defenses presupposes knowing the query's source. For anonymous queries, enforcement requires imposing a cost instead (see [Anonymous queries via Onions](#anonymous-queries-via-onions)).

#### Gossip erosion 

A `reply_path` introduces a second source of routing policy, one that may diverge from a node's gossiped policy. A routing node is bound to neither of these sources (see [Non-binding replies](#non-binding-replies)), since real policy is enforced only at forward time, when an HTLC is agreed upon.

A routing node that prefers per-query quoting has less reason to honor its potentially stale gossip, which erodes the value of the gossiped message and thereby degrades pathfinding for senders who do not query. However, routing nodes that no longer honor their `channel_update` messages forfeit fees from the payment flows that rely on them: the value of gossip is proportional to the pathfinders using it.

Consistency between the two sources is therefore a strategy concern. The simplest strategy for early adopters is to reply with already-gossiped policy and an `expiry_ms` of 0: no forward claim, no deviation from published policy, yet the reply still applies the node's first-hand knowledge of its balances to the queried flow. A more sophisticated strategy sets gossiped policy at the upper bound of desired policy and applies a slight discount to queries within a window; gossip then remains accurate as a worst case, and queries only improve on it.

Note, this concern is confined to policy. Today's channel announcements, which define the network topology, must prove control of an onchain funding output and are therefore constrained by blockspace. This makes them well-suited for gossip (see [Motivation](#motivation)).

## Universality

Path queries are an optional, pairwise feature. A node only sends or responds to these messages with peers it chooses to, under whatever request & response strategy it selects (see [Putting into practice](#putting-into-practice)); nodes that do not implement this proposal are unaffected and simply won't originate or respond to these messages. Because adoption can be partial and strategies are locally decided, this feature does not require universal adoption, and is therefore proposed as a bLIP rather than a BOLT. Someday, with widespread adoption, this feature could influence gossip messages and eventually be considered universal. Today is not that day.

## Backwards Compatibility

Path queries introduce new, optional messages using odd types in the `32768`-`65535` range reserved for bLIPs, which nodes that do not understand them can safely ignore per [BOLT 1](https://github.com/lightning/bolts/blob/master/01-messaging.md). Support is advertised and negotiated via the `option_path_queries` feature bit (264/265, reserved via [bLIP 2](./blip-0002.md)); per the [requirements](#requirements), nodes MUST NOT send these messages to peers that have not signaled support. Since the feature is entirely optional and additive, there are no backwards incompatibilities with existing gossip, onion routing, or payment flows.

## Reference Implementation

None yet. A reference implementation is required before this bLIP can progress to Final status, per the [bLIP 1](./blip-0001.md) process.
