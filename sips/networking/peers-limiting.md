# SIPs - Networking

| Author      | Title                          | Category | Status |
|-------------|--------------------------------|----------|--------|
| [@amir-blox](https://github.com/amir-blox) | Peers Limiting and Balancing | Networking | DRAFT |

## Summary

This document describes the way peers count is limited in SSV, 
and how connected peers are selected according to their subnets.

### Rational & Design Goals

The main idea behind limiting peers count is to reduce the network throughput.
When connecting to a smaller amount of peers, we get less duplicated messages, 
while still receiving all messages directly or after some hops.

The main requirement from each node is to have enough peers 
in all the subnets of its interest. 
Peers balancing is a procedure where we tag the best peers we are connected to, 
then disconnecting from the peers that were not tagged. 
This solution ensures (over time) that: 
- our connections are replaced 
- our connections are distributed across all the subnets of our interest.

## Peers Balancing

Peers balancing is taking place every `1m` and includes the following steps:

**NOTE:** in this process we rely on information that was sent by the peer,
and on the internal state of connected peers (provided by libp2p's `Host` and `Pubsub`).

1. continue if we reached peers limit in the node level, or stop otherwise.
   (**TBD:** check topic peers limit) 
2. tag best `n` peers where `n = maxPeers - 1`
   1. calculate scores for subnets:
      1. subnet w/o peers - `2` 
      2. subnet w/ less than the minimum (2 peers) - `1`
      3. subnet w/ overflow of peers - `-1`
   2. calculate peers scores according to their subnets, 
   by a counting the subnets scores and giving bonus score for peers with multiple shared subnets.
   3. **TBD** pubsub scoring will be taken into account (once added)
3. trim untagged peers

### Discovery

Discovery system keeps on working by a random walk on the routing table, 
and therefore we'll keep finding new peers so our connected peers list changes over time.

### Tagging and Trimming

Tagging is done using `go-libp2p-core/connmgr.ConnManager` interface to protect or unprotect peers.

Trimming is currently done by checking the protection tag, and close connections of unprotected peers. \
**NOTE:** once resolved, the trimming wil be done by `go-libp2p-core/connmgr.ConnManager`
