# AaveBoosted MetaPool

## Definition

An AaveBoosted MetaPool is a StablePhantomPool that facilitates trades between existing AaveBoosted Stable Pool tokens and those of a new AaveBoosted LinearPool.

For example, if we have a new US Dollar stablecoin called `USDX`, we could create an AaveBoosted LinearPool `bb-a-USDX` and put that in a StablePhantomPool with the existing `bb-a-USD` pool token. This would facilitate trades between:

| Base Tokens | aTokens |
| ----------- | ------- |
| `DAI`       | `aDAI`  |
| `USDC`      | `aUSDC` |
| `USDT`      | `aUSDT` |
| `USDX`      | `aUSDX` |

## Steps

To build this pool, you will need to:

1. [Acquire some amount of `bb-a-USD`](aaveboosted-metapool.md#acquire-some-amount-of-bb-a-usd)``
2. [Deploy a `bb-a-USDX` AaveLinearPool](aaveboosted-metapool.md#deploy-a-bb-a-usdx-aavelinearpool)
3. [Deploy a StablePhantomPool with both `bb-a-USD` and `bb-a-USDX`](aaveboosted-metapool.md#deploy-a-stablephantompool-with-both-bb-a-usd-and-bb-a-usdx)``

## Acquire some amount of `bb-a-USD`

The easiest way to get `bb-a-USD` is to join the `bb-a-USD` pool with your choice of `DAI`, `USDC`, `USDT`, or one of their respective aTokens. For this, you can use the [Balancer UI](https://app.balancer.fi).

## Deploy a `bb-a-USDX` AaveLinearPool

For instructions on how to launch bb-a-USDX, please refer to the [AaveLinearPool](aavelinearpool.md) instructions.

## Deploy a StablePhantomPool with both `bb-a-USD` and `bb-a-USDX`

For instructions on how to launch a StablePhantomPool, please refer to the [StablePhantom Pool ](stablephantom-pool.md)instructions.
