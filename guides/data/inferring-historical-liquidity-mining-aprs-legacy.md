---
description: >-
  This section describes the legacy liquidity mining incentives program and how
  APRs could be calculated
---

# Inferring Historical Liquidity Mining APRs (Legacy)

The initial system of the liquidity mining incentives program was community driven and maintained by a Liquidity Mining Committee composed of “Ballers”. You can read about the legacy system [here](https://docs.balancer.fi/getting-started/faqs/liquidity-mining).

In the following article, we describe how the legacy system was set up and maintained so that historical liquidity mining incentive data can be accessed easily.

{% hint style="info" %}
If you are looking for an in-depth guide on how to calculate Gauge APRs, consult the [veBAL and Gauges section](../../resources/vebal-and-gauges/estimating-gauge-incentive-aprs/#how-can-i-obtain-calculate-the-liquidity-mining-apr-for-a-certain-gauge)
{% endhint %}

## **How was the legacy system set up?**

The legacy system was set up and controlled by three subsystems:&#x20;

1. A Liquidity Mining .json-file holding the corresponding BAL and partner token allocations for a given liquidity mining week. This .json file was maintained my the Liquidity Mining Committee
2. A claiming and distribution system developed by Balancer Labs called [Merkle Orchard ](https://docs.balancer.fi/products/merkle-orchard)
3. [A BAL liquidity mining repository](https://github.com/balancer-labs/bal-mining-scripts) calculating the per address distributions for the Merkle Orchards [per week](https://github.com/balancer-labs/bal-mining-scripts/tree/master/reports)

### Liquidity Mining .json Data Structure

The [liquidity mining json file](https://raw.githubusercontent.com/balancer-labs/frontend-v2/89c9ee2e297425865475d91e20a9e8f8d14f59e1/src/lib/utils/liquidityMining/MultiTokenLiquidityMining.json) contains the BAL and partner token allocations per week per chain. Historically, balancer-v2 started in liquidity mining week 52. Therefore, the first entry is “week\_52”. The data structure is set up as follows:

```json
 "week_99": [
    {
      "chainId": 137,
      "pools": {
        "0x0297e37f1873d2dab4487aa67cd56b58e2f27875000100000000000000000002": [
          {
            "tokenAddress": "0x9a71012b13ca4d3d0cdc72a177df3ef03b0e76a3",
            "amount": 5400
          }
        ],
        "0x36128d5436d2d70cab39c9af9cce146c38554ff0000100000000000000000008": [
          {
            "tokenAddress": "0x9a71012b13ca4d3d0cdc72a177df3ef03b0e76a3",
            "amount": 3190
          }
        ],
        "0x03cd191f589d12b0582a99808cf19851e468e6b500010000000000000000000a": [
          {
            "tokenAddress": "0x9a71012b13ca4d3d0cdc72a177df3ef03b0e76a3",
            "amount": 2250
          }
        ],
      }
  }
]
```

1. week\_XY, where XY starts at 52 and ends with 99
2. chainID
   1. 1 - mainnet
   2. 137 - Polygon
   3. 42161 - Arbitrum
3. A set of pools per chain
4. A set of token addresses and their corresponding emission amount

## How to calculate the historical APR for a given pool

Given the above mentioned data structure from the liquidity mining .json file, one can calculate historical APRs by fetching historical pool data from the subgraph and cross-referencing it to the emission schedule.&#x20;

{% hint style="info" %}
`week_52` in the liquidity mining json corresponds the 23rd of May 2021 with a unix start timestamp at `1621720800`
{% endhint %}

**Fetch historical pool data**

```graphql
{
  poolHistoricalLiquidities (where: {poolId: "0x5c6ee304399dbdb9c8ef030ab642b10820db8f56000200000000000000000014", block: "12376326"
}) {
  block
    poolLiquidity
    pricingAsset
  }
 }
```

The total liquidity in the pool can then be inferred by multiplying the poolLiquidity with the price of the pricing asset at that time point (e.g. fetch historical BAL price at that block time).

To fetch the historical price data of the pricingAsset of the underlying pool, do the following (example of BAL/WETH following above query):

```graphql
{
        tokenSnapshots(
            first: 1000
            orderBy: timestamp
            orderDirection: asc
            where: { token: "0xba100000625a3754423978a60c9317c58a424e3d", timestamp_gte: 1620208633 }
        ) {
            id
        timestamp
        totalBalanceUSD
        totalBalanceNotional
        totalVolumeUSD
        totalVolumeNotional
        totalSwapCount
        }
}
```

The token price at a given timestamp can then be derived as

$$
tokenPriceAtTimeStamp = totalBalanceUSD / totalBalanceNotional
$$

**With this information we can calculate the APR at the queried block as**

$$
APR_{historical} = \frac{totalIncentivesAtCorrespondingLMWeek * tokenPriceAtTimeStamp * 52 * 100}{ poolLiquidity * priceOfPoolPricingAsset_{inferred}}
$$

​
