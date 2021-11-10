# Liquidity Bootstrapping Pool

## Overview

For advantages and use cases of Liquidity Bootstrapping Pools (LBPs), please refer to [the standard documentation](https://docs.balancer.fi/products/balancer-pools/liquidity-bootstrapping-pools-lbps).

For more interfaces, such as updating pool weights, see the [Liquidity Bootstrapping Pool API](../../references/contracts/apis/pools/liquiditybootstrappingpool.md#api).

## Interfacing

Some elements to consider when interfacing with Liquidity Bootstrapping Pools:

* Using [Weighted Math](../pool-math/weighted-math.md)
* Pool weights can be dynamic
* Pool swaps may be disabled by the pool owner. Typically this is to prevent swaps before the weight shifting occurs, but this can technically happen at any time.
* Pool weights range from 1% to 99%
* Pools have between 2 and 4 tokens

## Getting Pool Data

In addition to the [common pool data](./#getting-common-pool-data), you will likely want the following data when interfacing with Liquidity Bootstrapping Pools:

### Weights

Weights are stored at the pool level. For example, calling

```
pool.getNormalizedWeights()
```

returns something resembling

```
[800000000000000000, 200000000000000000]
```

which are the weights represented with 18 decimals. A pool with 80%/20% weights corresponds to \[0.8, 0.2] after scaling for decimals.

```
getGradualWeightUpdateParams:
startTime uint256, endTime uint256, endWeights uint256[]
  startTime|uint256 :  1631523600
  endTime|uint256 :  1638781200
  endWeights|uint256[] :  180010681315327688, 820004577706569009
```
