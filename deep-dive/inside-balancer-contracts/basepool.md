# BasePool

## Credit

Credit for the below article goes to [0xSkly](https://twitter.com/0xSkly) of the BeethovenX Team. You can find the original article on [Medium](https://medium.com/@0xSkly/inside-balancer-contracts-basepool-bfe1196732ac).&#x20;

Disclaimer: The article has been lightly edited for formatting and uniformity with the rest of the docs, but has been kept in a first-person format. Any opinions expressed below are those of the author and should not be considered opinions of Balancer Labs, BeethovenX, BalancerDAO, or any other organization.

## A DeFi Building Block <a href="#35b1" id="35b1"></a>

Balancer already provides a number of different pools, from stable pools to weighted pools and many more. But remember, Balancer tech has to be seen as a DeFi building block. So you won’t be surprised so see that they provide you with all the tools to actually create your own specialized pool. And guess what, that’s what we are gonna do! But not this time, we are not quite there yet. Still gotta get some fundamentals done. And you know me (probably not really), I’m all about fundamentals! So let’s dive into our base building block when it comes to pools, the abstract `BasePool` contract.

## Inside the BasePool <a href="#471d" id="471d"></a>

The `BasePool` contract provides you with all the key mechanics to build out a pool with the high standards one would expect from a pool coming from Balancer Labs. If you see a pool based on this base contract, you know it’s no joke! To prove my claims, let’s jump right into it.

Roughly, the`BasePool` contract comes with the following base functionality.

### Authorization <a href="#4ffa" id="4ffa"></a>

Implementing pools can utilize the `authenticate` modifier to protect its functions with the sophisticated authorization mechanisms covered in [this article](https://medium.com/@0xSkly/inside-balancer-code-timelockauthorizer-5f5a0f7bec12).

### Emergency Pause <a href="#2c19" id="2c19"></a>

The `BasePool` inherits from the `TemporarilyPausable` contract which provides an emergency pause feature within the first 30 days of **factory deployment.** Once the 30 days are over and a pool is still paused, it remains paused for an **** additional maximum of 90 days until it gets automatically unpaused. A paused pool cannot take any trades, joins or exit. It’s literally halted!

### Swap Fee Management <a href="#d431" id="d431"></a>

A rather small but still important feature is management of the swap fee percentage. It makes sure that the swap fee stays between the configured minimum and maximum percentage which defaults to a range of 0.0001% to 10%.

### Vault Integration <a href="#28d5" id="28d5"></a>

We have not yet talked about the `Vault` contract, which many would rank as one of the most innovative building blocks from Balancer Labs. Unfortunately it doesn’t make sense to get into the details of it now because it’s just too amazing and deserves its own spotlight. So I’ll have to cover it in a later piece, no way around that! But for now you have to live with my short summary:

The `Vault` holds all pool tokens and performs the related bookkeeping. It serves as a single entry point for pool interactions like joins, exits and swaps and delegates to the respective pools. With that in mind, let’s see how the `BasePool` integrates with the `Vault`.

![](https://miro.medium.com/max/1400/1\*HtZdC1F9CXkOU1L0puKUuA.png)

We see that the pool is registering itself with the `Vault` with its specialization in return for a `poolId`. We’ll dig into pool specialization and `poolId` generation another time, otherwise we’ll get nowhere. In short, the pool specialization is for gas optimizations when swapping and the `poolId` encodes the BPT address with its specialization for more gas optimizations. Now that we are hooked up to the `Vault`, it will call our liquidity management hooks, `onJoinPool` and `onExitPool` which leads us to our next topic.

### Liquidity Management <a href="#ce9b" id="ce9b"></a>

The `BasePool` provides some base functionality for adding and removing liquidity. We start with the `onJoinPool` hook, which also acts as a pool initializer if it’s the first join.

![](https://miro.medium.com/max/1400/1\*QxaAcWn1OZWRxzhj0\_rOVg.png)

Oh boy, there is quite some stuff going on here. Let’s try to make some sense of it. So we claimed that everything gets routed through the `Vault` right? We see that this is enforced by the `onlyVault(poolId)` modifier.

![](https://miro.medium.com/max/1400/1\*8na8FNa0K6AEr5KNkvI1iQ.png)

It checks if the `msg.sender` is actually the `Vault` and if the provided `poolId` is actually the `poolId` of this pool. Fair enough, rather not mess that up! Knowing that this call is indeed coming from the `Vault` we can assume that the arguments require no further validation. See the comments on the arguments for some context. We also check if the pool is not paused `_ensureNotPaused()` and with that out of the way, we are ready for some business logic. Right away, we see that there is a different execution branch when `totalSupply == 0` meaning this is the first join ever and therefore the **pool initialization flow**. We should’t skip this section cause there is some interesting details to uncover!

First we call the `_onInitializePool(..)` handler which has to be implemented by the implementing pool. The handler has to return

* amount of BPT to mint
* token amounts the pool will receive

Next, we do something interesting, we actually mint the amount returned from `_getMinimumBpt()`, which defaults to `1e6` BPT, to the zero address acting as a buffer for rounding errors and preventing the pool from ever getting completely drained. Neat! The remaining BPT are minted to the recipient.

Now we come to a nifty little detail, but its actually quite an important one which could lead to severe issues when not handled correctly by the implementation — the scaling factors. You might have noticed that we just skipped over the very first statement _`uint256`_`[]`` `_`memory`_` ``scalingFactors = _scalingFactors()` where `_scalingFactors()` has to be overridden by the implementing contract. So what are those scaling factors? Well once more, its about decimals. Not all tokens have the same amount of decimals, so to make the math work, all tokens get normalized to 18 decimals. The `BasePool` provides a helper function for that

![](https://miro.medium.com/max/1400/1\*ihzxWlXCeqbcIIuX07c6Og.png)

So now that we know what those scaling factors are, let us look at the last statement of the pool initialization flow `_downscaleUpArray(amountsIn, scalingFactors)`. Like the name suggests, it downscales the `amountsIn` by the `scalingFactors` where the result is scaled up, therefore `downscaleUp`. Now this call reveals that the `amountsIn` returned from the implementing contract have to be upscaled, which is also stated in the NatSpec

```
The tokens granted to the Pool will be transferred from `sender`. These amounts are considered upscaled and will be downscaled (rounding up) before being returned to the Vault.
```

Something we have to keep in mind when we create our own pool!

At last, we return the `amountsIn` to the `Vault` together with an empty array for the due protocol fees.

Ok, so we have covered the initialization flow, now let’s move onto the `else` branch which represents the **regular joins**.

We start by `_upscaleArray(balances, scalingFactors)` The `balances` are the total token balances in the `Vault` for this pool which are normalized to 18 decimals. We then delegate to the implementation with the `_onJoinPool` call which returns us the amount of BPT to mint as well as the normalized `amountsIn` . Same as with the pool initialization flow, we mint the BPT to the recipient and return the downscaled (or denormalized)`amountsIn` together with an empty array for the due protocol fees back to the `Vault`.

But wait a second! Why is the due protocol fee hardcoded to an empty array here? I mean it does make sense for the initialization flow, since nobody traded yet, there is obviously no protocol fee to collect, but on regular joins, we should collect some at some point, right? Guess what, we found another new development from Balancer Labs. Seems they have changed the way protocol fees are collected! Bear with me, we’ll get to it, but first things first, let’s talk about **exiting the pool**.

![](https://miro.medium.com/max/1400/1\*GSbLRIHt6M-zxv63vU6PWg.png)

A little shorter than the join, but still a lot going on here. The arguments are the same as for the join, so I did not add comments this time.

First we handle our new feature, dealing with recovery mode exits, which I already dedicated a [full article](https://medium.com/@0xSkly/inside-balancer-code-recoverymode-9af34ce5ab72) to. Additionally, we make sure that the pool is not paused and start again by normalizing the token balances with `upscaleArray(balances, scalingFactors)`. Nothing new here, we are starting to feel comfy in here aren’t we? We then delegate to the implementation, receiving the BPT to burn and the `amountsOut`. We downscale the `amountsOut` so they are ready to be returned to the `Vault`. Finally, we burn the BPT and return the `amountsOut` together with an empty array for the due protocol fees to the `Vault`. Easy enough! But now, what happened to those protocol fees? Let’s dive into it!

### Protocol Fee Management <a href="#1f40" id="1f40"></a>

As you might remember, until now, protocol fees were paid in one of the underlying assets of the pool and transferred to the `protocolFeesCollector` by piggybacking on joins and exits. That is why the `Vault` accepts a second parameter in the return value of joins and exits, the due protocol fee. But now that we are returning a hardcoded empty array, how are those fees paid? Well, there is a new function available

![](https://miro.medium.com/max/1400/1\*fMjKGnkVXKWy2wLQEN7dVQ.png)

Nothing too spectacular, but it shows a paradigm shift. Instead of paying protocol fee in an underlying token, it mints BPT to the protocol fee collector. But who is calling it? Its not being called in the `BasePool`, so I looked around and its for the implementing pools to figure out a gas efficient way of keeping track of the due BPT to be paid as as protocol fees and call this handler. We’ll see more about that once we dive into specific pool implementations!

This might seem like a rather small change, but it actually opens up a whole lot of possibilities. For example, right now its a chore to figure out which pool paid how much into the protocol fee collector which makes things like kickbacks much harder to implement, especially on chain. But with this mechanism, this seems quite possible. But exploring the possibilities of this change probably deserves its own article and a bigger financial visionary than I am.

## Conclusion <a href="#ba84" id="ba84"></a>

Now that we have covered the core functionality provided by the `BasePool` contract, we can safely say that it’s an amazing building block for creating pools utilizing the Balancer ecosystem. There are some pitfalls and one still needs to understand a good chunk of the Balancer tech to safely integrate a new pool, but once you know the fundamentals, it’s almost easy!

This was quite a long journey but i hope you still enjoyed it! Soon we’ll be able to try and create our own little pool type, maybe an easy 50/50 pool, who knows. But we are not quite there yet, there are still some missing pieces. Or who wants to provide liquidity to a pool you cannot trade?

\
