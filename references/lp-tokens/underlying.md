# Underlying

## Overview

It's common to want to know what a Balancer LP token has as its underlying tokens.

## Directly Query The Vault and LP Tokens

The Vault can tell you exactly how many tokens are in the pool. You can query that, and then divide by the amount of the pool that is yours

### Pseudocode

```
(tokens, balances, lastChangeBlock) = vault.getPoolTokens(poolId);
yourPoolShare = bpt.balanceOf(yourAddress)/bpt.totalSupply();
uint256 yourUnderlyingBalances = new uint256[](balances.length);
for(i=0, i<balances.length, i++){
    yourUnderlyingBalances[i] = balances[i]*yourPoolShare;
}
return(tokens,yourUnderlyingBalances); 
```

{% hint style="warning" %}
The above assumes you have your BPT in your wallet. If you have staked your BPT in a gauge, you'll need to calculate your BPT holdings as:\
`myBpt = bpt.balanceOf(yourAddress) + bptGaugeDeposit.balanceOf(yourAddress);`
{% endhint %}
