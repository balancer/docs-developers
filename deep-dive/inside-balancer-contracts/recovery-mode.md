# Recovery Mode

## Credit

Credit for the below article goes to [0xSkly](https://twitter.com/0xSkly) of the BeethovenX Team. You can find the original article on [Medium](https://medium.com/@0xSkly/inside-balancer-code-recoverymode-9af34ce5ab72).&#x20;

Disclaimer: The article has been lightly edited for formatting and uniformity with the rest of the docs, but has been kept in a first-person format. Any opinions expressed below are those of the author and should not be considered opinions of Balancer Labs, BeethovenX, BalancerDAO, or any other organization.

## `RecoveryMode` <a href="#bfd6" id="bfd6"></a>

Balancer Labs introduces a new mechanism to secure user funds in emergency situations. This is a technical insight into the `RecoveryMode` class which enables this functionality when inherited.

> ```
> This is intended to provide a safe way to exit any pool during some 
> kind of emergency, to avoid locking funds in the event the pool 
> enters a non-functional state (i.e., some code that normally runs 
> during exits is causing them to revert).
> ```

### What's the Effect When a Pool is in RecoveryMode ? <a href="#bcf2" id="bcf2"></a>

A special clean exit function is enabled which runs the absolute minimum code necessary to exit proportionally.

In particular:

* Stable pools do not try to calculate the invariant
* No protocol fees are collected

It’s important to note that the pool is not disabled, which means regular swaps, joins, and exits are still possible.

### Who is Authorized to Put a Pool Into RecoveryMode ? <a href="#ef49" id="ef49"></a>

The external function `enableRecoveryMode` is protected by the `Authorizer` configured on the Vault. Therefore any address granted permission on the `Authorizer` can enable recovery mode.

This permission needs to be treated with care since the RecoveryMode functionality could be misused.

### What Pools Have a Recovery Exit Available ? <a href="#db81" id="db81"></a>

The `BasePool` class which is a base layer for all current pools inherits from the `RecoveryMode` class enabling the recovery exit functionality for all new pools deployed (which are based on the `BasePool` class).

{% hint style="info" %}
Note: Many pools of many poolTypes were created before this was added to BasePool, so there is a decent chance the pool you're in does not have this functionality. As more pools are created using newer factories, this scenario should change.
{% endhint %}

### Inside the `RecoveryMode` Exit <a href="#b553" id="b553"></a>

The default implementation is within the `RecoveryMode` class and needs to be overwritten for special pools like a `PhantomStablePool`

![](https://miro.medium.com/max/1400/1\*U3pP4UaMPeibjMX9n4WD4Q.png)

I’ve added some comments to the function parameters for some context. So we extract the provided BPT from the `userData` and calculate the share of the total supply by `bptAmountIn / totalSupply` . Based on this ratio we can then calculate the share of each token in the pool. See `_computeProportionalAmountsOut`

![](https://miro.medium.com/max/1400/1\*0NXuNlvuVip99N4kNBzFxw.png)

### But What About StablePhantomPools ? <a href="#ab30" id="ab30"></a>

`StablePhantomPools` use pre-minted BPT, so therefore we cannot use the `totalSupply` to calculate the share of a user. That’s why there is a custom implementation.

![](https://miro.medium.com/max/1400/1\*GrsRRz5PPVuHodDWxVmR6g.png)

We see that before calling the default implementation with `super._doRecoveryModeExit(...)`, we subtract the pre-minted BPT. This results in a ‘virtual supply’ which is the actual BPT in circulation.

But how do we know the actual BPT in circulation? Remember the `balances` parameter which includes the total balance of all tokens in the pool. This also includes the pre minted BPT. So with this, we can do `totalSupply — balances[indexOfBpt]` which results in all BPT in circulation. Additionally we also need to remove the BPT token from the `balances` array. This is what happens in `_dropBptItemFromBalances(balances)`

### How is it All Wired Up ? <a href="#0ebc" id="0ebc"></a>

The entry point to a pool exit is the `Vault` class which delegates the call to the respective pool contract. The base layer of each pool contract is the `BasePool` which also implements the `onExitPool` hook:

![](https://miro.medium.com/max/1400/1\*tmG7z8bmIeAwG0GkF9D72A.png)

## Conclusion <a href="#1708" id="1708"></a>

With this mechanism in place, we gain an additional safety layer for users to exit with their funds in case of some unexpected pool behavior.
