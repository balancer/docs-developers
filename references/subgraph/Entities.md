---
sidebar_position: 2
title: Subgraph Entities
---

# Entities

- [`Balancer`](#balancer)
- [`Pool`](#pool)
- [`PoolToken`](#pooltoken)
- [`PriceRateProvider`](#pricerateprovider)
- [`PoolShare`](#poolshare)
- [`User`](#user)
- [`UserInternalBalance`](#userinternalbalance)
- [`GradualWeightUpdate`](#gradualweightupdate)
- [`AmpUpdate`](#ampupdate)
- [`Swap`](#swap)
- [`JoinExit`](#joinexit)
- [`LatestPrice`](#latestprice)
- [`PoolHistoricalLiquidity`](#poolhistoricalliquidity)
- [`TokenPrice`](#tokenprice)
- [`Investment`](#investment)
- [`PoolSnapShot`](#poolsnapshot)
- [`Token`](#token)
- [`TokenSnapShot`](#tokensnapshot)
- [`TradePair`](#tradepair)
- [`TradePairSnapshot`](#tradepairsnapshot)
- [`BalancerSnapshot`](#balancersnapshot)

# Balancer

Description: 

| Field           | Type             | Description |
| --------------- | ---------------- | ----------- |
| id              | ID!              |             |
| poolCount       | Int!             |             |
| pools           | [`Pool!`](#pool) |             |
| totalLiquidity  | BigDecimal!      |             |
| totalSwapCount  | BigInt!          |             |
| totalSwapVolume | BigDecimal!      |             |
| totalSwapFee    | BigDecimal!      |             |

# Pool

Description: 

| Field                   | Type                                                                             | Description |
| ----------------------- | -------------------------------------------------------------------------------- | ----------- |
| id                      | ID!                                                                              |             |
| address                 | Bytes!                                                                           |             |
| poolType                | String                                                                           |             |
| factory                 | Bytes                                                                            |             |
| strategyType            | Int!                                                                             |             |
| oracleEnabled           | Boolean!                                                                         |             |
| symbol                  | String                                                                           |             |
| name                    | String                                                                           |             |
| swapEnabled             | Boolean!                                                                         |             |
| swapFee                 | BigDecimal!                                                                      |             |
| owner                   | Bytes                                                                            |             |
| totalWeight             | BigDecimal                                                                       |             |
| totalSwapVolume         | BigDecimal!                                                                      |             |
| totalSwapFee            | BigDecimal!                                                                      |             |
| totalLiquidity          | BigDecimal!                                                                      |             |
| totalShares             | BigDecimal!                                                                      |             |
| createTime              | Int!                                                                             |             |
| swapCount               | BigInt!                                                                          |             |
| holdersCount            | BigInt!                                                                          |             |
| vaultID                 | Balancer!                                                                        |             |
| tx                      | Bytes                                                                            |             |
| tokensList              | [Bytes!]!                                                                        |             |
| tokens                  | [`PoolToken!`](#pooltoken)                                                       |             |
| swaps                   | [`Swap!`](#swap)                                                                 |             |
| shares                  | [`PoolShare!`](#poolshare)                                                       |             |
| historicalValues        | [`PoolHistoricalLiquidity!`](#poolhistoricalliquidity) |             |
| weightUpdates [^1]      | [`GradualWeightUpdate!`](#gradualweightupdate)                                   |             |
| amp [^2]                | BigInt                                                                           |
| priceRateProviders [^3] | [`PriceRateProvider!`](#pricerateprovider)                  |             |
| principalToken [^4]     | Bytes                                                                            |             |
| baseToken [^4]          | Bytes                                                                            |             |
| expirtyTime [^4]        | BigInt                                                                           |             |
| unitSeconds [^4]        | BigInt                                                                           |             |
| managementFee [^5]      | BigDecimal                                                                       |             |
| mainIndex [^6]          | Int                                                                              |             |
| wrappedIndex [^6]       | Int                                                                              |             |
| lowerTarget [^6]        | BigDecimal                                                                       |             |
| upperTarget [^6]        | BigDecimal                                                                       |             |
| sqrtAlpha [^7]          | BigDecimal                                                                       |             |
| sqrtBeta [^7]           | BigDecimal                                                                       |             |
| root3Alpha [^8]         | BigDecimal                                                                       |             |

[^1]: Liquiditybootstrappingpoolonly
[^2]: Stablepoolonly
[^3]: MetastablepoolandLinearPoolOnly
[^4]: ConvergentCurvePoolElementOnly
[^5]: InvestmentPoolOnly
[^6]: LinearPoolONly
[^7]: Gyro2PoolOnly
[^8]: Gyro3PoolOnly

# PoolToken

Description:

| Field         | Type                   | Description |
| ------------- | ---------------------- | ----------- |
| id            | ID!                    |             |
| poolId        | Pool                   |             |
| token         | Token!                 |             |
| assetManager  | Bytes                  |             |
| symbol        | String!                |             |
| decimals      | Int!                   |             |
| address       | String!                |             |
| priceRate     | BigDecimal!            |             |
| balance       | BigDecimal!            |             |
| cashBalance   | BigDecimal!            |             |
| manageBalance | BigDecimal!            |             |
| management    | [ManagementOperation!] |             |
| weight [^9]   | BigDecimal             |             |

[^9]: weightedpoolonly

# PriceRateProvider

Description: 

| Field          | Type        | Description |
| -------------- | ----------- | ----------- |
| id             | ID!         |             |
| poolID         | Pool!       |             |
| token          | PoolToken!  |             |
| address        | Bytes!      |             |
| rate           | BigDecimal! |             |
| lastCached     | Int!        |             |
| cachedDuration | Int!        |             |
| cashExpiry     | Int!        |             |

# PoolShare

Description:

| Field       | Type        | Description |
| ----------- | ----------- | ----------- |
| id          | ID!         |             |
| userAddress | User!       |             |
| poolId      | Pool!       |             |
| balance     | BigDecimal! |             |

# User

Description: 

| Field                | Type                                           | Description |
| -------------------- | ---------------------------------------------- | ----------- |
| id                   | ID!                                            |             |
| sharesOwned          | [`PoolShare!`](#poolshare)                     |             |
| swaps                | [`Swap!`](#swap)                               |             |
| userInternalBalances | [`UserInternalBalance!`](#userinternalbalance) |             |

# UserInternalBalance

Description: 

| Field       | Type        | Description |
| ----------- | ----------- | ----------- |
| id          | ID!         |             |
| userAddress | User        |             |
| token       | Bytes!      |             |
| balance     | BigDecimal! |             |

# GradualWeightUpdate

Description: 

| Field              | Type    | Description |
| ------------------ | ------- | ----------- |
| id                 | ID!     |             |
| poolId             | Pool!   |             |
| scheduledTimestamp | Int!    |             |
| startTimestamp     | BigInt! |             |
| endTimestamp       | BigInt! |             |
| startWeights       | BigInt! |             |
| endWeights         | BigInt! |             |

# AmpUpdate

Description: 

| Field              | Type    | Description |
| ------------------ | ------- | ----------- |
| id                 | ID!     |             |
| poolId             | Pool!   |             |
| scheduledTimestamp | Int!    |             |
| startTimestamp     | BigInt! |             |
| endTimestamp       | BigInt! |             |
| endAmp             | BigInt! |             |

# Swap

Description: 

| Field          | Type        | Description |
| -------------- | ----------- | ----------- |
| id             | ID!         |             |
| caller         | Bytes!      |             |
| tokenIn        | Bytes!      |             |
| tokenIn        | Bytes!      |             |
| tokenInSym     | String!     |             |
| tokenOut       | Bytes!      |             |
| tokenOutSym    | String!     |             |
| tokenAmountIn  | BigDecimal! |             |
| tokenAmountOut | BigDecimal! |             |
| valueUSD       | BigDecimal! |             |
| poolId         | Pool!       |             |
| UserAddress    | User!       |             |
| timestamp      | Int!        |             |
| tx             | Bytes!      |             |

# JoinExit

Description:

| Field     | Type           | Description |
| --------- | -------------- | ----------- |
| id        | ID!            |             |
| type      | InvestType!    |             |
| sender    | Bytes!         |             |
| amounts   | [BigDecimal!]! |             |
| pool      | Pool!          |             |
| user      | User!          |             |
| timestamp | Int!           |             |
| tx        | Bytes!         |             |

# LatestPrice

Description: 

| Field        | Type        | Description                         |
| ------------ | ----------- | ----------------------------------- |
| id           | ID!         |                                     |
| asset        | Bytes!      |                                     |
| pricingAsset | Bytes!      | address of stable asset             |
| poolID       | Pool!       | last pool which set price           |
| price        | BigDecimal! | all the latest prices               |
| block        | BigInt!     | last block that prices were updated |

# PoolHistoricalLiquidity

Description: 

| Field           | Type        | Description                                      |
| --------------- | ----------- | ------------------------------------------------ |
| id              | ID!         |                                                  |
| poolId          | Pool!       |                                                  |
| poolTotalShares | BigDecimal! |                                                  |
| poolLiquidity   | BigDecimal! | total value, priced in the stable asset - ie USD |
| poolShareValue  | BigDecimal! |                                                  |
| pricingAsset    | Bytes!      | address of stable asset                          |
| block           | BigInt!     |                                                  |

# TokenPrice

Description: 

| Field        | Type        | Description                                     |
| ------------ | ----------- | ----------------------------------------------- |
| id           | ID!         | address of token + address of stablecoin-poolId |
| poolId       | Pool!       |                                                 |
| assets       | Bytes!      |                                                 |
| amount       | BigDecimal! |                                                 |
| pricingAsset | Bytes!      | address of stable asset                         |
| price        | BigDecimal! |                                                 |
| block        | BigInt!     |                                                 |
| timestamp    | Int!        |                                                 |

# ManagementOperation

Description: 

| Field       | Type           | Description |
| ----------- | -------------- | ----------- |
| id          | ID!            |             |
| type        | OperationType! |             |
| cashDelta   | BigDecimal!    |             |
| manageDelta | BigDecimal!    |             |
| poolTokenID | PoolToken!     |             |
| timestamp   | Int!           |             |

# PoolSnapshot

Description: 

| Field       | Type           | Description |
| ----------- | -------------- | ----------- |
| id          | ID!            |             |
| pool        | Pool!          |             |
| amounts     | [BigDecimal!]! |             |
| totalShares | BigDecimal!    |             |
| swapVolume  | BigDecimal!    |             |
| swapFees    | BigDecimal!    |             |
| liquidity   | BigDecimal!    |             |
| timestamp   | Int!           |             |

# Token

Description: 

| Field                | Type        | Description                                                       |
| -------------------- | ----------- | ----------------------------------------------------------------- |
| id                   | ID!         |                                                                   |
| symbol               | String      |                                                                   |
| name                 | String      |                                                                   |
| decimals             | Int!        |                                                                   |
| address              | String!     |                                                                   |
| totalBalanceUSD      | BigDecimal! | total balance of tokens across balancer                           |
| totalBalanceNotional | BigDecimal! |                                                                   |
| totalVolumeUSD       | BigDecimal! | total volume in fiat (usd)                                        |
| totalVlumeNotional   | BigDecimal! |                                                                   |
| totalSwapCount       | BigInt!     |                                                                   |
| latestPrice          | LatestPrice | latest price of token, updated when pool liquidity changes        |
| latestUSDPrice       | BigDecimal! | latest price of token in USD, updated when pool liquidity changes |
| pool                 | Pool        | pool entity associated with the token, if it is a Balancer pool   |

# TokenSnapshot

Description: 

| Field                | Type        | Description                                      |
| -------------------- | ----------- | ------------------------------------------------ |
| id                   | ID!         | token address + dayId                            |
| token                | Token!      |                                                  |
| totalBalanceUSD      | BigDecimal! | total balance of tokens across balancer          |
| totalBalanceNotional | BigDecimal! | underlying asset balance                         |
| totalVolumeUSD       | BigDecimal! | amount of volume the token has moved on this day |
| totalVolumeNotional  | BigDecimal! | underyling asset volume                          |
| totalSwapCount       | BigInt!     |                                                  |

# TradePair

Description: 

| Field           | Type        | Description                   |
| --------------- | ----------- | ----------------------------- |
| id              | ID!         | Token Address - Token Address |
| token0          | Token!      |                               |
| token1          | Token!      |                               |
| totalSwapVolume | BigDecimal! |                               |
| totalSwapFee    | BigDecimal! |                               |

# TradePairSnapshot

Description:

| Field           | Type        | Description |
| --------------- | ----------- | ----------- |
| id              | ID!         |             |
| pair            | TradePair!  |             |
| timestamp       | Int!        |             |
| totalSwapVolume | BigDecimal! |             |
| totalSwapFee    | BigDecimal! |             |

# BalancerSnapshot

Description: 

| Field           | Type        | Description |
| --------------- | ----------- | ----------- |
| id              | ID!         |             |
| vault           | Balancer!   |             |
| timestamp       | Int!        |             |
| poolCount       | Int!        |             |
| totalLiquidity  | BigDecimal! |             |
| totalSwapCount  | BigInt!     |             |
| totalSwapVolume | BigDecimal! |             |
| totalSwapFee    | BigDecimal! |             |
