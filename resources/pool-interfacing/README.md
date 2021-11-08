# Pool Interfacing

## Overview

Since there are many different pool types, it's important to note the differences between them when interfacing with Balancer. Some pool use different pricing equations, some have dynamic pricing, and some might have swaps disabled periodically.&#x20;

## `poolId`s

If you want to interface with a pool, you'll first need to know its `poolId`. The `poolId` is a unique identifier, the first portion of which is the pool's contract address. For example, the pool with the id `0x5c6ee304399dbdb9c8ef030ab642b10820db8f56000200000000000000000014` has a contract address of `0x5c6ee304399dbdb9c8ef030ab642b10820db8f56`.&#x20;

You can get a poolId from:

* A pool's URL: [https://app.balancer.fi/#/pool/0x5c6ee304399dbdb9c8ef030ab642b10820db8f56000200000000000000000014](https://app.balancer.fi/#/pool/0x5c6ee304399dbdb9c8ef030ab642b10820db8f56000200000000000000000014)
* The [Subgraph](https://thegraph.com/hosted-service/subgraph/balancer-labs/balancer-v2)
* Calling `getPoolId()` on the contract itself

## Getting Common Pool Data

### Pool Balances

Since all tokens are held in the Vault, you must query the Vault for pool balances. For example:

```
vault.getPoolTokens(0x5c6ee304399dbdb9c8ef030ab642b10820db8f56000200000000000000000014_
```

returns something resembling:

```
tokens:  [0xba100000625a3754423978a60c9317c58a424e3D,
                0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2]
                
balances:  [5720903090084350251216632,
                7939247003721636150710]
```

