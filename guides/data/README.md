# Data

If you're trying to interface with Balancer V2, chances are you're going to want to see what's going on. This section will go over some methods for querying Balancer data.

## Balancer Subgraph

The Balancer Subgraph is a great way to query pools, balances, and more. The Subgraph tracks each pool by poolId, and can be queried using filtering and sorting techniques built into GraphQL.

The hosted version of The Graph can experience downtime, so there are some cases where it's advantageous not to rely on it. As The Graph decentralizes, it will become more resilient and reliable.

## On-chain Queries

Sometimes you might want to query the Ethereum (or EVM-compatible) blockchain directly for Balancer data. There is no on-chain list of Balancer pools, so you'll need to issue a preliminary query to the Subgraph, or maintain your own cached pool list. Depending on if you have a local node or a remote node (like Infura), query time and query volume may be concerns. We'll go over techniques for how to most efficiently query on-chain data.

## FAQs

### Is there a contract that I can query with two token addresses to get the pool address or `poolId`?

No, Balancer does not have an on-chain registry of pools and their corresponding tokens.

#### Why not?

Balancer does not have canonical pools; anyone can deploy a pool of any composition they desire, so there is no guarantee that there exists only one DAI, WETH pool, for example. Since Balancer supports multi-token pools, DAI and WETH could be two tokens among many more in certain pools. Further, there could be pools of different types that have overlapping tokens, such as WeightedPool, StablePools, and LiquidityBootstrappingPools.

#### Ok, but how do I get all pools that have DAI and WETH in them?

The subgraph, of course! To query pools with specific tokens, do something like the following:

```
{
  pools(first: 100, where:{tokensList_contains:["0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2","0x6b175474e89094c44da98b954eedeac495271d0f"]}) {
    id
    poolType
    tokens {
      address
    }
  }
}
```
