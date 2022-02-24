# Episode 2: Joins

## Intro

Here we'll explore the inner workings of a `joinPool()` call in Balancer V2. We'll follow the process all the way from the top level call at the Vault to the internal pool hooks in a WeightedPool.

{% hint style="info" %}
Not all pools use `joinPool()` and `exitPool()` for transactions involving their Balancer Pool Tokens (BPTs). A LinearPool, for example, mints all of its BPT and the time of creation and registers them as a pool token so that users can join/exit the pool only via `swap()` or `batchSwap()`.
{% endhint %}

## Scenario

* A user has quantities `N,M` of `Token A` and `Token B` and wants to join a Token A/Token B Balancer Pool.
* The user finds the `poolId` of the desired pool, and crafts a join (and exit) call.
* The user executes a `joinPool()` in the Vault.

## The Code

Like `swap()` calls, `joinPool()` and `exitPool()` calls happen through the Vault. The Vault receives and sends all tokens, while asking the pool contract how many tokens or Balancer Pool Tokens to exchange. We're going to start out with a `joinPool()`.

If you'd like to follow along with the source code, file names will be relative to [contracts folder on the weighted-deployment tag of the Balancer V2 Monorepo](https://github.com/balancer-labs/balancer-v2-monorepo/tree/weighted-deployment/contracts).

### `vault/Vault.sol`

```
* Roughly speaking, these are the contents of each sub-contract:
*
*  - `AssetManagers`: Pool token Asset Manager registry, and Asset Manager interactions.
*  - `Fees`: set and compute protocol fees.
*  - `FlashLoans`: flash loan transfers and fees.
*  - `PoolBalances`: Pool joins and exits.
*  - `PoolRegistry`: Pool registration, ID management, and basic queries.
*  - `PoolTokens`: Pool token registration and registration, and balance queries.
*  - `Swaps`: Pool swaps.
*  - `UserBalance`: manage user balances (Internal Balance operations and external balance transfers)
*  - `VaultAuthorization`: access control, relayers and signature validation.
```

As we read the comments in [Vault.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Vault.sol#L39), we see that the **join/exit** functionality we're looking for is implemented in [PoolBalances.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol). (This gets imported by way of [Swaps.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L28))

### `vault/PoolBalances.sol`

```
function joinPool(
    bytes32 poolId,
    address sender,
    address recipient,
    JoinPoolRequest memory request
) external payable override whenNotPaused {
    // This function doesn't have the nonReentrant modifier: it is applied to `_joinOrExit` instead.

    // Note that `recipient` is not actually payable in the context of a join - we cast it because we handle both
    // joins and exits at once.
    _joinOrExit(PoolBalanceChangeKind.JOIN, poolId, sender, payable(recipient), _toPoolBalanceChange(request));
}
```

We see that the [`joinPool()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L43-L54) call simply redirects us to a generalized [`_joinOrExit()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L107) function. Within `_joinOrExit()`, we have [a few checks and data retrievals](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L118-L123) to get out of the way before we get to the exciting part.

```
InputHelpers.ensureInputLengthMatch(change.assets.length, change.limits.length);

// We first check that the caller passed the Pool's registered tokens in the correct order, and retrieve the
// current balance for each.
IERC20[] memory tokens = _translateToIERC20(change.assets);
bytes32[] memory balances = _validateTokensAndGetBalances(poolId, tokens);
```

First we verify that our inputs have the same lengths, then translate our asset addresses into `ERC20`s. Finally, we get to `_validateTokensAndGetBalances()`. [This function](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L287-L301) ultimately calls `_getPoolTokens()` for our given pool and verifies that the tokens we're attempting to join with match those registered to the pool.

### `vault/PoolTokens.sol`

```
function _getPoolTokens(bytes32 poolId) internal view returns (IERC20[] memory tokens, bytes32[] memory balances) {
    PoolSpecialization specialization = _getPoolSpecialization(poolId);
    if (specialization == PoolSpecialization.TWO_TOKEN) {
        return _getTwoTokenPoolTokens(poolId);
    } else if (specialization == PoolSpecialization.MINIMAL_SWAP_INFO) {
        return _getMinimalSwapInfoPoolTokens(poolId);
    } else {
        // PoolSpecialization.GENERAL
        return _getGeneralPoolTokens(poolId);
    }
}
```

`_getPoolTokens()` is defined in [PoolTokens.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolTokens.sol#L134-L144). For this example we'll follow the join for a WeightedPool, which is of specialization type `MINIMAL_SWAP_INFO`. Calling `_getMinimalSwapInfoPoolTokens()` brings us to `MinimalSwapInfoPoolsBalance.sol`.

### `vault/balances/MinimalSwapInfoPoolsBalance.sol`

```
function _getMinimalSwapInfoPoolTokens(bytes32 poolId)
    internal
    view
    returns (IERC20[] memory tokens, bytes32[] memory balances)
{
    EnumerableSet.AddressSet storage poolTokens = _minimalSwapInfoPoolsTokens[poolId];
    tokens = new IERC20[](poolTokens.length());
    balances = new bytes32[](tokens.length);

    for (uint256 i = 0; i < tokens.length; ++i) {
        // Because the iteration is bounded by `tokens.length`, which matches the EnumerableSet's length, we can use
        // `unchecked_at` as we know `i` is a valid token index, saving storage reads.
        IERC20 token = IERC20(poolTokens.unchecked_at(i));
        tokens[i] = token;
        balances[i] = _minimalSwapInfoPoolsBalances[poolId][token];
    }
}
```

We first grab the `poolTokens` from `_minimalSwapInfoPoolsTokens` [defined here](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/balances/MinimalSwapInfoPoolsBalance.sol#L40), and then we use those to fill the `tokens`. Similarly, we grab the balances from `_minimalSwapInfoPoolBalances` [defined here](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/balances/MinimalSwapInfoPoolsBalance.sol#L39), and shove them into `balances`. From here, we jump back up the call stack with our `balances` to [`vault/PoolBalances.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L123).

### `vault/PoolBalances.sol`

```
// The bulk of the work is done here: the corresponding Pool hook is called, its final balances are computed,
// assets are transferred, and fees are paid.
(
    bytes32[] memory finalBalances,
    uint256[] memory amountsInOrOut,
    uint256[] memory paidProtocolSwapFeeAmounts
) = _callPoolBalanceChange(kind, poolId, sender, recipient, change, balances);
```

As the comments explain, [`_callPoolBalanceChange()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L155-L204) is where the bulk of the work is done for a join/exit.

```
(uint256[] memory totalBalances, uint256 lastChangeBlock) = balances.totalsAndLastChangeBlock();

IBasePool pool = IBasePool(_getPoolAddress(poolId));
(amountsInOrOut, dueProtocolFeeAmounts) = kind == PoolBalanceChangeKind.JOIN
    ? pool.onJoinPool(
        poolId,
        sender,
        recipient,
        totalBalances,
        lastChangeBlock,
        _getProtocolSwapFeePercentage(),
        change.userData
    )
    : pool.onExitPool(
        poolId,
        sender,
        recipient,
        totalBalances,
        lastChangeBlock,
        _getProtocolSwapFeePercentage(),
        change.userData
    );
```

After unpacking balances and block times, we get the pool from the `poolId`. Next (lines 4-22 in this above code block, and lines [177-195 in the contract](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L177-L195)) we call the pool's `onJoinPool()` hook after the ternary operator indicates that our `PoolBalanceChangeKind` is of type `JOIN`.

### `pools/BasePool.sol`: `onJoinPool()`

```
if (totalSupply() == 0) {
    ...
} else {
    _upscaleArray(balances, scalingFactors);
    (uint256 bptAmountOut, uint256[] memory amountsIn, uint256[] memory dueProtocolFeeAmounts) = _onJoinPool(
        poolId,
        sender,
        recipient,
        balances,
        lastChangeBlock,
        protocolSwapFeePercentage,
        userData
    );
```

Any pool that inherits from BasePool.sol will use [this function](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BasePool.sol#L197-L245). In our example case, we're joining a pool that has already been initialized, so we're going to skip the `if (totalSupply() == 0)` case.

{% hint style="info" %}
If you're interested in digging into how a join on an uninitialized, freshly deployed pool works, [click here](detour-the-init-join.md) to choose your own adventure!

Don't worry, that page will link you back here after we finish the detour.
{% endhint %}

We now upscale the balance arrays to ensure uniform token decimals for pool math, and then call the pool's specific [`_onJoinPool()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L232-L277) function.

### `pools/weighted/WeightedPool.sol`

```
uint256[] memory normalizedWeights = _normalizedWeights();
```

First, in order to join the pool we need the `normalizedWeights`.

```
function _normalizedWeights() internal view virtual returns (uint256[] memory) {
    uint256 totalTokens = _getTotalTokens();
    uint256[] memory normalizedWeights = new uint256[](totalTokens);

    // prettier-ignore
    {
        if (totalTokens > 0) { normalizedWeights[0] = _normalizedWeight0; } else { return normalizedWeights; }
        if (totalTokens > 1) { normalizedWeights[1] = _normalizedWeight1; } else { return normalizedWeights; }
        if (totalTokens > 2) { normalizedWeights[2] = _normalizedWeight2; } else { return normalizedWeights; }
        if (totalTokens > 3) { normalizedWeights[3] = _normalizedWeight3; } else { return normalizedWeights; }
        if (totalTokens > 4) { normalizedWeights[4] = _normalizedWeight4; } else { return normalizedWeights; }
        if (totalTokens > 5) { normalizedWeights[5] = _normalizedWeight5; } else { return normalizedWeights; }
        if (totalTokens > 6) { normalizedWeights[6] = _normalizedWeight6; } else { return normalizedWeights; }
        if (totalTokens > 7) { normalizedWeights[7] = _normalizedWeight7; } else { return normalizedWeights; }
    }

    return normalizedWeights;
}
```

Calling [`_normalizedWeights()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L120-L137) returns an array packed with the `immutable`, already normalized, `_normalizedWeight<i>` and if `i >= numTokens` it will return the array.

#### Jumping back to `_onPoolJoin()`...

```
// Due protocol swap fee amounts are computed by measuring the growth of the invariant between the previous join
// or exit event and now - the invariant's growth is due exclusively to swap fees. This avoids spending gas
// computing them on each individual swap
uint256 invariantBeforeJoin = WeightedMath._calculateInvariant(normalizedWeights, balances);

uint256[] memory dueProtocolFeeAmounts = _getDueProtocolFeeAmounts(
    balances,
    normalizedWeights,
    _lastInvariant,
    invariantBeforeJoin,
    protocolSwapFeePercentage
);
```

Next we grab the invariant before joining. We calculate the invariant with the generalized constant product formula defined in [`WeightedMath.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedMath.sol#L47-L65). If you're familiar with the [Balancer Whitepaper](https://balancer.fi/whitepaper.pdf), this is the $$\prod_{i=0}^{n}{{B_i}^{w_i}}$$ where $$n$$ is the number of tokens, $$B_i$$ _and_ $$w_i$$ are the balance and weight for $$token_i$$ respectively.

Next we move to get the fees due to the Protocol with [`_getDueProtocolFeeAmounts()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L481-L507). These are collected as a percentage of swap fees.

```
// Initialize with zeros
uint256[] memory dueProtocolFeeAmounts = new uint256[](_getTotalTokens());

// Early return if the protocol swap fee percentage is zero, saving gas.
if (protocolSwapFeePercentage == 0) {
    return dueProtocolFeeAmounts;
}

// The protocol swap fees are always paid using the token with the largest weight in the Pool. As this is the
// token that is expected to have the largest balance, using it to pay fees should not unbalance the Pool.
dueProtocolFeeAmounts[_maxWeightTokenIndex] = WeightedMath._calcDueTokenProtocolSwapFeeAmount(
    balances[_maxWeightTokenIndex],
    normalizedWeights[_maxWeightTokenIndex],
    previousInvariant,
    currentInvariant,
    protocolSwapFeePercentage
);

return dueProtocolFeeAmounts;
```

If there are no protocol fees, we break early returning an array of all `0`s. When the protocol fee is activated, however, we calculate the due amounts in [`WeightedMath.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedMath.sol#L327-L362). Here, we pass `_maxWeightTokenIndex` because the WeightedPool collects fees denominated in the token with the highest weight. This is done to have the protocol fee collection create the smallest price impact while collecting an underlying token. There are techniques for creation no price impact whatsoever, but that's outside the scope of this explanation.

### `pools/weighted/WeightedMath.sol`

```
/*********************************************************************************
/*  protocolSwapFeePercentage * balanceToken * ( 1 - (previousInvariant / currentInvariant) ^ (1 / weightToken))
*********************************************************************************/

if (currentInvariant <= previousInvariant) {
    // This shouldn't happen outside of rounding errors, but have this safeguard nonetheless to prevent the Pool
    // from entering a locked state in which joins and exits revert while computing accumulated swap fees.
    return 0;
}

// We round down to prevent issues in the Pool's accounting, even if it means paying slightly less in protocol
// fees to the Vault.

// Fee percentage and balance multiplications round down, while the subtrahend (power) rounds up (as does the
// base). Because previousInvariant / currentInvariant <= 1, the exponent rounds down.

uint256 base = previousInvariant.divUp(currentInvariant);
uint256 exponent = FixedPoint.ONE.divDown(normalizedWeight);

// Because the exponent is larger than one, the base of the power function has a lower bound. We cap to this
// value to avoid numeric issues, which means in the extreme case (where the invariant growth is larger than
// 1 / min exponent) the Pool will pay less in protocol fees than it should.
base = Math.max(base, FixedPoint.MIN_POW_BASE_FREE_EXPONENT);

uint256 power = base.powUp(exponent);

uint256 tokenAccruedFees = balance.mulDown(power.complement());
return tokenAccruedFees.mulDown(protocolSwapFeePercentage);
```

We now jump into WeightedMath's `_calcDueTokenProtocolSwapFeeAmount()`. The comments at the top of the function give a math explanation of what we're calculating. By dividing the `previousInvariant` by the `currentInvariant` and scaling that value by the `balance` and `weight`, what we're really calculating is the value by which the pool has grown denominated in `_maxWeightToken` due to swap fees between the last join/exit. The protocol fee is a percentage of this growth, and rounds down (in favor of the pool and its liquidity providers).

### `pool/weighted/WeightedPool.sol`

Now that we've determined the protocol fees, we're back in the `_onJoinPool()` call in [`WeightedPool.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L232-L277).

```
// Update current balances by subtracting the protocol fee amounts
_mutateAmounts(balances, dueProtocolFeeAmounts, FixedPoint.sub);
(uint256 bptAmountOut, uint256[] memory amountsIn) = _doJoin(balances, normalizedWeights, userData);
```

With our freshly determined `dueProtocolFeeAmounts`, we do the Vault's bookkeeping to remove those from the pool. We then call the [`_doJoin()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L279-L293) hook with the `JoinKind` and either exact input amounts or exact output BPT amount encoded in `userData`.

```
function _doJoin(
    uint256[] memory balances,
    uint256[] memory normalizedWeights,
    bytes memory userData
) private view returns (uint256, uint256[] memory) {
    JoinKind kind = userData.joinKind();

    if (kind == JoinKind.EXACT_TOKENS_IN_FOR_BPT_OUT) {
        return _joinExactTokensInForBPTOut(balances, normalizedWeights, userData);
    } else if (kind == JoinKind.TOKEN_IN_FOR_EXACT_BPT_OUT) {
        return _joinTokenInForExactBPTOut(balances, normalizedWeights, userData);
    } else {
        _revert(Errors.UNHANDLED_JOIN_KIND);
    }
}
```

For this example, we'll assume that we're doing a join of type `EXACT_TOKENS_IN_FOR_BPT_OUT`, which tells you how many BPT you get for given input tokens. We therefore move to [`_joinExactTokensInForBPTOut()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L295-L316).

```
function _joinExactTokensInForBPTOut(
    uint256[] memory balances,
    uint256[] memory normalizedWeights,
    bytes memory userData
) private view returns (uint256, uint256[] memory) {
    (uint256[] memory amountsIn, uint256 minBPTAmountOut) = userData.exactTokensInForBptOut();
    InputHelpers.ensureInputLengthMatch(_getTotalTokens(), amountsIn.length);

    _upscaleArray(amountsIn, _scalingFactors());

    uint256 bptAmountOut = WeightedMath._calcBptOutGivenExactTokensIn(
        balances,
        normalizedWeights,
        amountsIn,
        totalSupply(),
        _swapFeePercentage
    );

    _require(bptAmountOut >= minBPTAmountOut, Errors.BPT_OUT_MIN_AMOUNT);

    return (bptAmountOut, amountsIn);
```

After input checks and upscaling for decimals, we query [WeightedMath.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedMath.sol#L140-L179) to find out how many BPT we should receive for our given input tokens in `_calcBptOutGivenExactTokensIn()`.

### `pools/weighted/WeightedMath.sol`

```
uint256[] memory balanceRatiosWithFee = new uint256[](amountsIn.length);

uint256 invariantRatioWithFees = 0;
for (uint256 i = 0; i < balances.length; i++) {
    balanceRatiosWithFee[i] = balances[i].add(amountsIn[i]).divDown(balances[i]);
    invariantRatioWithFees = invariantRatioWithFees.add(balanceRatiosWithFee[i].mulDown(normalizedWeights[i]));
}
```

In order to prevent users from dodging swap fees, joins on WeightedMath-based pools charge swap fees on token inputs that unbalance a pool. For example, if there's a WeightedPool with 100 USDC and 100 USDT and you come along and join it with 10 USDC and 15 USDT, you'll be charged a swap fee on the 5 USDT that you're depositing that will unbalance the pool.

In order to bookkeep for this, we calculate `balanceRatiosWithFee` for all input tokens. We also accumulate `invariantRatioWithFees` by adding the amount by which each token increases the invariant. Notice how if all inputs are proportional to the tokens already in the pool, the `balanceRatiosWithFee` will all be equivalent, and will also be equivalent with the `invariantRatioWithFees`.

```
uint256 invariantRatio = FixedPoint.ONE;
for (uint256 i = 0; i < balances.length; i++) {
    uint256 amountInWithoutFee;

    if (balanceRatiosWithFee[i] > invariantRatioWithFees) {
        uint256 nonTaxableAmount = balances[i].mulDown(invariantRatioWithFees.sub(FixedPoint.ONE));
        uint256 taxableAmount = amountsIn[i].sub(nonTaxableAmount);
        amountInWithoutFee = nonTaxableAmount.add(taxableAmount.mulDown(FixedPoint.ONE.sub(swapFee)));
    } else {
        amountInWithoutFee = amountsIn[i];
    }

    uint256 balanceRatio = balances[i].add(amountInWithoutFee).divDown(balances[i]);

    invariantRatio = invariantRatio.mulDown(balanceRatio.powDown(normalizedWeights[i]));
}
```

Here, we separate out the inputs amounts to the `nonTaxableAmount`s, which are the token input amounts that maintain the pool's balance, and the `taxableAmount`s, which unbalance the pool. We calculate each token's `amountInWithoutFee`, which is `nonTaxableAmount + taxableAmount*(1-swapFee)` (for a balanced join, this just reduces down to `nonTaxableAmount`). As we accumulate the weighted product of these `balanceRatio`s in `invariantRatio`, what we're effectively doing is determining the proportion of new BPT to mint while charging a swap fee denominated in BPT of the amounts that are unbalancing the pool.

```
if (invariantRatio >= FixedPoint.ONE) {
    return bptTotalSupply.mulDown(invariantRatio.sub(FixedPoint.ONE));
} else {
    return 0;
}
```

Finally, we return amount of BPT to mint such that the new total BPT supply increases proportionally to how the deposit increases the invariant. Now, we climb back up the call stack; we return that value (and the `amountsIn`) from [`_joinExactTokensInForBPTOut()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L315) in WeightedPool.sol, and then return those from [`_doJoin()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L287).

### `pools/weighted/WeightedPool.sol`

```
// Update the invariant with the balances the Pool will have after the join, in order to compute the
// protocol swap fee amounts due in future joins and exits.
_lastInvariant = _invariantAfterJoin(balances, amountsIn, normalizedWeights);

return (bptAmountOut, amountsIn, dueProtocolFeeAmounts);
```

We're now at the end of the WeightedPool's [`onJoinPool()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L272-L276) call. We calculate and store the current invariant so that we can determine due protocol fees during the next join/exit. [`_invariantAfterJoin()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L513-L520) calculates the invariant accounting for the tokens deposited in this join. We now return the BPT due to the depositor, the amounts of tokens they need to supply, and the amounts of tokens that will be collected by the protocol.

### `pools/BasePool.sol`

```
_mintPoolTokens(recipient, bptAmountOut);
```

We've made it back to `onJoinPool()` in BasePool.sol, where we now mint the `bptAmountOut` that we just calculated.

### `pools/BalancerPoolToken.sol`

```
function _mintPoolTokens(address recipient, uint256 amount) internal {
    _balance[recipient] = _balance[recipient].add(amount);
    _totalSupply = _totalSupply.add(amount);
    emit Transfer(address(0), recipient, amount);
}
```

This is a short, sweet, and simple mint that gives the depositor their BPT. Let's go back to BasePool.sol.

### `pools/BasePool.sol`

```
// amountsIn are amounts entering the Pool, so we round up.
_downscaleUpArray(amountsIn, scalingFactors);
// dueProtocolFeeAmounts are amounts exiting the Pool, so we round down.
_downscaleDownArray(dueProtocolFeeAmounts, scalingFactors);

return (amountsIn, dueProtocolFeeAmounts);
```

Now, we scale our tokens (both `amountsIn` and `dueProtocolFeeAmounts`) by their respective decimals. For tokens entering/exiting the pool, we round up/down respectively to ensure that the pool is not susceptible to a rounding error attack. As such, `amountsIn` rounds up and `dueProtocolFeeAmounts` rounds down.

### `vault/PoolBalances.sol`

```
InputHelpers.ensureInputLengthMatch(balances.length, amountsInOrOut.length, dueProtocolFeeAmounts.length);

// The Vault ignores the `recipient` in joins and the `sender` in exits: it is up to the Pool to keep track of
// their participation.
finalBalances = kind == PoolBalanceChangeKind.JOIN
    ? _processJoinPoolTransfers(sender, change, balances, amountsInOrOut, dueProtocolFeeAmounts)
    : _processExitPoolTransfers(recipient, change, balances, amountsInOrOut, dueProtocolFeeAmounts);
```

Coming back to [`_callPoolBalanceChange()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L197-L203), we quickly ensure the input lengths and jump right into [`_processJoinPoolTransfers()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L213-L248).

```
// We need to track how much of the received ETH was used and wrapped into WETH to return any excess.
uint256 wrappedEth = 0;

finalBalances = new bytes32[](balances.length);
for (uint256 i = 0; i < change.assets.length; ++i) {
    uint256 amountIn = amountsIn[i];
    _require(amountIn <= change.limits[i], Errors.JOIN_ABOVE_MAX);

    // Receive assets from the sender - possibly from Internal Balance.
    IAsset asset = change.assets[i];
    _receiveAsset(asset, amountIn, sender, change.useInternalBalance);
    
    ...
    
}
```

We now handle the actual transfer of tokens and/or bookkeeping of internal balances. For each asset with which we are joining the pool, we make sure it's a valid amount, and then call [`_receiveAsset()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/AssetTransfersHandler.sol#L44-L79).

### `vault/AssetTransfersHandler.sol`

```
function _receiveAsset(
    IAsset asset,
    uint256 amount,
    address sender,
    bool fromInternalBalance
) internal {
    if (amount == 0) {
        return;
    }

    if (_isETH(asset)) {
        _require(!fromInternalBalance, Errors.INVALID_ETH_INTERNAL_BALANCE);

        // The ETH amount to receive is deposited into the WETH contract, which will in turn mint WETH for
        // the Vault at a 1:1 ratio.

        // A check for this condition is also introduced by the compiler, but this one provides a revert reason.
        // Note we're checking for the Vault's total balance, *not* ETH sent in this transaction.
        _require(address(this).balance >= amount, Errors.INSUFFICIENT_ETH);
        _WETH().deposit{ value: amount }();
    } else {
        IERC20 token = _asIERC20(asset);

        if (fromInternalBalance) {
            // We take as many tokens from Internal Balance as possible: any remaining amounts will be transferred.
            uint256 deductedBalance = _decreaseInternalBalance(sender, token, amount, true);
            // Because `deductedBalance` will be always the lesser of the current internal balance
            // and the amount to decrease, it is safe to perform unchecked arithmetic.
            amount -= deductedBalance;
        }

        if (amount > 0) {
            token.safeTransferFrom(sender, address(this), amount);
        }
    }
}
```

`_receiveAsset()` is a relatively straightforward function that handles users sending tokens to the Vault; it wraps any ETH into WETH, accepts ERC20 tokens, and accepts partial and full internal balances. If the join does pull tokens from internal user balances, this will call [`_decreaseInternalBalance()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/UserBalance.sol#L181-L195).

### `vault/UserBalance.sol`

```
uint256 currentBalance = _getInternalBalance(account, token);
_require(allowPartial || (currentBalance >= amount), Errors.INSUFFICIENT_INTERNAL_BALANCE);

deducted = Math.min(currentBalance, amount);
// By construction, `deducted` is lower or equal to `currentBalance`, so we don't need to use checked
// arithmetic.
uint256 newBalance = currentBalance - deducted;
_setInternalBalance(account, token, newBalance, -(deducted.toInt256()));
```

This deducts from internal balance the `min` of the requested amount and the internal balance. This allows for joins being able to be sourced entirely from internal balance or partially and supplemented with standard ERC20 balances.

### `vault/PoolBalances.sol`

```
for (uint256 i = 0; i < change.assets.length; ++i) {

    ...
    
    _receiveAsset(asset, amountIn, sender, change.useInternalBalance);

    if (_isETH(asset)) {
        wrappedEth = wrappedEth.add(amountIn);
    }

    uint256 feeAmount = dueProtocolFeeAmounts[i];
    _payFeeAmount(_translateToIERC20(asset), feeAmount);

    ...
    
}
```

Now that the Vault has received ERC20 tokens from the pool joiner, we check to see if an asset is ETH so we can handle it later. We now briefly jump to `_payFeeAmount()` to collect any protocol fees during the join.

### `vault/Fees.sol`

```
function _payFeeAmount(IERC20 token, uint256 amount) internal {
    if (amount > 0) {
        token.safeTransfer(address(getProtocolFeesCollector()), amount);
    }
}
```

This simply transfers and positive due token to the ProtocolFeesCollector contract.

### `vault/PoolBalances.sol`

```
for (uint256 i = 0; i < change.assets.length; ++i) {

        ...

        // Compute the new Pool balances. Note that the fee amount might be larger than `amountIn`,
        // resulting in an overall decrease of the Pool's balance for a token.
        finalBalances[i] = (amountIn >= feeAmount) // This lets us skip checked arithmetic
            ? balances[i].increaseCash(amountIn - feeAmount)
            : balances[i].decreaseCash(feeAmount - amountIn);
    }

    // Handle any used and remaining ETH.
    _handleRemainingEth(wrappedEth);
```

Finally in `_processJoinPoolTransfers()`, we calculate the new balances in the pool. Even though we are adding tokens with the join, we may also be removing tokens with the protocol fee collection; therefore, it's possible that there could be a net decrease in balances. We'll record these updates in the Vault in the next step.

Now that we've completed `_processJoinPoolTransfers()`, we hop back up the call stack through `_callPoolBalanceChange()`, which we have also just completed and make our way back to [`_joinOrExit()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolBalances.sol#L133-L142).

```
// All that remains is storing the new Pool balances.
PoolSpecialization specialization = _getPoolSpecialization(poolId);
if (specialization == PoolSpecialization.TWO_TOKEN) {
    _setTwoTokenPoolCashBalances(poolId, tokens[0], finalBalances[0], tokens[1], finalBalances[1]);
} else if (specialization == PoolSpecialization.MINIMAL_SWAP_INFO) {
    _setMinimalSwapInfoPoolBalances(poolId, tokens, finalBalances);
} else {
    // PoolSpecialization.GENERAL
    _setGeneralPoolBalances(poolId, finalBalances);
}
```

Since we're assuming we're joining a WeightedPool (of specialization MINIMAL\_SWAP\_INFO), we now call [`_setMinimalSwapInfoPoolBalances()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/balances/MinimalSwapInfoPoolsBalance.sol#L95-L103) to update the Vault's pool balances.

### `vault/balances/MinimalSwapInfoPoolsBalance.sol`

```
function _setMinimalSwapInfoPoolBalances(
    bytes32 poolId,
    IERC20[] memory tokens,
    bytes32[] memory balances
) internal {
    for (uint256 i = 0; i < tokens.length; ++i) {
        _minimalSwapInfoPoolsBalances[poolId][tokens[i]] = balances[i];
    }
}
```

Here we're simply iterating through all tokens to update the mapping `_minimalSwapInfoPoolsBalances`, defined [here](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/balances/MinimalSwapInfoPoolsBalance.sol#L39).

### `vault/PoolBalances.sol`

```
bool positive = kind == PoolBalanceChangeKind.JOIN; // Amounts in are positive, out are negative
emit PoolBalanceChanged(
    poolId,
    sender,
    tokens,
    // We can unsafely cast to int256 because balances are actually stored as uint112
    _unsafeCastToInt256(amountsInOrOut, positive),
    paidProtocolSwapFeeAmounts
);
```

Now as we finish `_joinOrExit()`, we finally emit the `PoolBalanceChanged` event to announce the changes that our join has created.

After we emit this event, we jump back up the call stack to `joinPool()`, where we started. As `_joinOrExit()` is the only call in `joinPool()`, we've now finished our deposit.

## Fin

And that's how a `joinPool()` call works! I invite you to explore the codebase to see how different pool specializations and pool types behave in their own ways. If you follow the `exitPool()` call similarly through the codebase, you'll find many similarities; they also need to convert between pool tokens and BPTs, and protocol fees are collected on both exits and joins.
