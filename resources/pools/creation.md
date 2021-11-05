# Creation

## Overview

{% hint style="warning" %}
Before creating a new pool, make sure there isn't already a similar pool; there's no advantage to fragmenting liquidity!
{% endhint %}

Anyone can create a pool on Balancer. A pool creation UI is in progress, but users can deploy them today programmatically. It's highly recommended that you deploy a pool on a testnet before doing so on a production network.

## What kind of pool should I deploy?

That totally depends on your use-case. You should read through the [descriptions of different Balancer PoolTypes](https://docs.balancer.fi/products/balancer-pools) to decide what you want to deploy.

## Fees and Owners

Two factors your should consider before deploying your pool are how fees and owners should be set. Pools can have static fees or dynamic fees, read [more about them in the main docs](https://docs.balancer.fi/concepts/fees#static-and-dynamic-fees).

### Static Fees

If you want static fees, you should set the fee you want the pool to have forever, and set the `owner` to the zero address `0x0000000000000000000000000000000000000000`.

### Dynamic Fees

If you want dynamic fees, you should set the fee to an initial value, and set the `owner` either as the address that you want to control the pool fee, or to the delegate address. An address that is set as the owner has permission to set the fee to anything between 0.0001% and 10% whenever they want.

If the pool owner is set to the delegate address (`0xBA1BA1ba1BA1bA1bA1Ba1BA1ba1BA1bA1ba1ba1B`) then Governance-approved fee-setters have permission to change the fee. Currently [Gauntlet has this authority](https://medium.com/gauntlet-networks/balancer-v2-pools-trading-fee-methodology-7a65df671b8c).&#x20;

### Owner Rights

Aside from setting swap fees, pool owners have other right on some pools that may play a role when deciding on an owner.&#x20;

* [Stable Pools](../../references/contracts/apis/pools/stablepools.md#permissioned-functions)
  * Changing `ampParameter`
* [MetaStable Pools](../../references/contracts/apis/pools/metastablepools.md#permissioned-functions)
  * Changing `ampParameter`
  * Changing `cacheDuration`
* [Liquidity Bootstrapping Pools](../../references/contracts/apis/pools/liquiditybootstrappingpool.md#permissioned-functions)
  * Changing `swapEnable`
  * Changing weights
* [Investment Pools](../../references/contracts/apis/pools/investmentpools.md#permissioned-functions)
  * Changing `swapEnable`
  * Changing weights
  * Collecting management fees

## Deploying a pool with balpy

The Balancer Python library [balpy](https://pypi.org/project/balpy/) supports deploying pools. Using the [samples in the balpy GitHub repository](https://github.com/balancer-labs/balpy/tree/main/samples/poolCreation), you can deploy pools from config files. There are sample config files for:

* Weighted Pools
* Oracle Pools (WeightedPool2Tokens)
* Stable Pools
* Liquidity Bootstrapping Pools
* MetaStable Pools
* Investment Pools

Once you have set up the necessary [environment variables](https://github.com/balancer-labs/balpy#environment-variables) and created your [virtual environment](https://github.com/balancer-labs/balpy#install), you can run the sample script with the command below. The script will ensure that you have sufficient token balances and **will execute token allowance approvals** if you do not have sufficient allowances.&#x20;

```
python3 poolCreationSample.py <yourModifiedPoolFile.json>
```
