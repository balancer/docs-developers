# Using Native ETH

{% hint style="info" %}
This article discusses how the Vault can wrap ETH to WETH. While this concept applies to the native assets on other EVM compatible chains (ex. MATIC on Polygon), this article will refer to native assets as ETH and their wrapped counterparts as WETH.
{% endhint %}

## Overview

While native ETH is not _directly_ compatible with Balancer, the Vault can automatically wrap/unwrap ETH to/from WETH when performing typical operations.

## Sentinel Value

In the documentation and smart contract comments, you may find references to the _Sentinel Value_ when dealing with Native ETH. This is a placeholder address to denote that the asset you're dealing with is not an ERC20 token, but is instead Native ETH.

When dealing with Native ETH, the Sentinel Value you'll provide is the zero address`0x0000000000000000000000000000000000000000`

## Swaps

### Single Swaps

If you wish to send or receive native ETH in a single swap, simply provide the sentinel address for `assetIn` or `assetOut` respectively.

### Batch Swaps

If you wish to send or receive native ETH in a batch swap, provide the sentinel address in the `assets` array and refer to it by its index in your swap steps as you would any other token. For Batch Swaps, you must sort the sentinel address numerically; do not sort it in the array as if it is the WETH address.\
\
Note: it is possible to send ETH and WETH in the same swap by providing both of the corresponding asset indices as inputs.

## Join/Exit

When joining or exiting a pool, you have to construct a `JoinPoolRequest` or `ExitPoolRequest` struct. Within this struct, you have the `assets` array:

```cpp
address[] assets
```

As you'll find in the documentation for [Joins and Exits](../resources/joins-and-exits/), this array must be sorted numerically; there is a caveat here though. If you wish to join with or exit to Native ETH, you need to **order the array** **as if you're dealing with WETH**. **** Note that it is not possible to combine ETH and WETH in the same join/exit; any excess ETH will be sent back to the caller (not the sender, which is important for relayers).

#### Correctly Ordered Example with WETH

| Token | Address                                    |
| ----- | ------------------------------------------ |
| DAI   | 0x6b175474e89094c44da98b954eedeac495271d0f |
| BAL   | 0xba100000625a3754423978a60c9317c58a424e3D |
| WETH  | 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 |

**Correctly Ordered Example with Native ETH**

| Token | Address                                    |
| ----- | ------------------------------------------ |
| DAI   | 0x6b175474e89094c44da98b954eedeac495271d0f |
| BAL   | 0xba100000625a3754423978a60c9317c58a424e3D |
| ETH   | 0x0000000000000000000000000000000000000000 |
