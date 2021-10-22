# Data

If you're trying to interface with Balancer V2, chances are you're going to want to see what's going on. This section will go over some methods for querying Balancer data.

## Balancer Subgraph

The Balancer Subgraph is a great way to query pools, balances, and more. The Subgraph tracks each pool by poolId, and can be queried using filtering and sorting techniques built into GraphQL.

The hosted version of The Graph can experience downtime, so there are some cases where it's advantageous not to rely on it. As The Graph decentralizes, it will become more resilient and reliable.

## On-chain Queries

Sometimes you might want to query the Ethereum (or EVM-compatible) blockchain directly for Balancer data. There is no on-chain list of Balancer pools, so you'll need to issue a preliminary query to the Subgraph, or maintain your own cached pool list. Depending on if you have a local node or a remote node (like Infura), query time and query volume may be concerns. We'll go over techniques for how to most efficiently query on-chain data.
