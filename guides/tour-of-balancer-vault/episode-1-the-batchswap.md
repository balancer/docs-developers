# Episode 1: The batchSwap

## Intro

Here we'll dive into how a trade gets executed on Balancer V2. We'll look at a `batchSwap` that hops through two Weighted Pools.

## Scenario

* A user has quantity `N` of `Token A` and wants to trade them all to `Token B`
* Either the user or a route finder determines the trade route with the best price. The best route in this case goes through an intermediate step in which it is converted to `Token C`
* The user executes a `batchSwap()` in the Vault instead of multiple calls to `swap()`, which saves gas on multi-hop trades

## The Code

All trades on Balancer go through the Vault. This architecture abstracts asset bookkeeping and security from the pools themselves, which empowers pool developers to focus purely on pricing equations. We'll start our journey at the Vault contract.

If you'd like to follow along with the source code, file names will be relative to [contracts folder on the weighted-deployment tag of the Balancer V2 Monorepo](https://github.com/balancer-labs/balancer-v2-monorepo/tree/weighted-deployment/contracts).

### `vault/Vault.sol`

* Looking at [Vault.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Vault.sol#L60), we quickly see that the **swap** functionality we're looking for lives in its own sub-contract

```
contract Vault is VaultAuthorization, FlashLoans, Swaps {
    ...
}
```

### `vault/Swaps.sol`

* Diving into [Swaps.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol), we make our way to [`function batchSwap()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L107)
* After some basic checks for a swap deadline and matching argument lengths, we come to a **`_swapWithPools()`** call

```
_require(block.timestamp <= deadline, Errors.SWAP_DEADLINE);

InputHelpers.ensureInputLengthMatch(assets.length, limits.length);

// Perform the swaps, updating the Pool token balances and computing the net Vault asset deltas.
assetDeltas = _swapWithPools(swaps, assets, funds, kind);
```

* As the [dev comments](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L200-L204) explain, `_swapWithPools()` doesn't actually perform token transfers, rather it calculates the token input/output `assetDeltas` to tell the Vault how many of each token should be transferred.
* As we [iterate through each swap step](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L222) in our `batchSwap`, we get the input/output tokens from our `batchSwap` data. To reduce size of arguments on arbitrarily long `batchSwap` data, swap steps use asset indices, so we [convert from indices to addresses to ERC20 assets](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L229-L230) at each step.

```
IERC20 tokenIn = _translateToIERC20(assets[batchSwapStep.assetInIndex]);
IERC20 tokenOut = _translateToIERC20(assets[batchSwapStep.assetOutIndex]);
```

* We're going to ignore [the block ](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L234)for if`batchSwapStep.amount == 0` for now, but we'll come back to it later.
* We now [populate and send the request](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L244-L257) to the pool

```
// Initializing each struct field one-by-one uses less gas than setting all at once
poolRequest.poolId = batchSwapStep.poolId;
poolRequest.kind = kind;
poolRequest.tokenIn = tokenIn;
poolRequest.tokenOut = tokenOut;
poolRequest.amount = batchSwapStep.amount;
poolRequest.userData = batchSwapStep.userData;
poolRequest.from = funds.sender;
poolRequest.to = funds.recipient;

...

(previousAmountCalculated, amountIn, amountOut) = _swapWithPool(poolRequest);
```

* In order to know how to handle the arguments for a swap, the Vault needs to know the specialization type for the pool with which we'll be trading. There are three specializations:
  * `TWO_TOKEN`: the pool has only two tokens
  * `MINIMAL_SWAP_INFO`: the pool has >= 2 tokens, but the balances for only input/output tokens are needed to calculate swaps
  * `GENERAL`: the pool has >= 2 tokens and all token balances are needed to calculate swaps

```
// Get the calculated amount from the Pool and update its balances
address pool = _getPoolAddress(request.poolId);
PoolSpecialization specialization = _getPoolSpecialization(request.poolId);

if (specialization == PoolSpecialization.TWO_TOKEN) {
    amountCalculated = _processTwoTokenPoolSwapRequest(request, IMinimalSwapInfoPool(pool));
} else if (specialization == PoolSpecialization.MINIMAL_SWAP_INFO) {
    amountCalculated = _processMinimalSwapInfoPoolSwapRequest(request, IMinimalSwapInfoPool(pool));
} else {
    // PoolSpecialization.GENERAL
    amountCalculated = _processGeneralPoolSwapRequest(request, IGeneralPool(pool));
}
```

In our case of a Weighted Pool, `_getPoolSpecialization()` will return **`MINIMAL_SWAP_INFO`**. This is because only the input and output token balances are needed to determine prices.

* Now we move to [`_processMinimalSwapInfoPoolSwapRequest()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L342)\`\`

```
bytes32 tokenInBalance = _getMinimalSwapInfoPoolBalance(request.poolId, request.tokenIn);
bytes32 tokenOutBalance = _getMinimalSwapInfoPoolBalance(request.poolId, request.tokenOut);

// Perform the swap request and compute the new balances for 'token in' and 'token out' after the swap
(tokenInBalance, tokenOutBalance, amountCalculated) = _callMinimalSwapInfoPoolOnSwapHook(
    request,
    pool,
    tokenInBalance,
    tokenOutBalance
);
```

### `vault/balances/MinimalSwapInfoPoolsBalance.sol`

* First, we need to get input/output tokens balances; this requires calling [`_getMinimalSwapInfoPoolBalance()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/balances/MinimalSwapInfoPoolsBalance.sol#L204) which is defined in `balances/MinimalSwapInfoPoolsBalance.sol`.
  * If you're curious how this gets imported, `Swaps.sol` imports `PoolBalances.sol` which imports `PoolTokens.sol` which imports `AssetManagers.sol` which imports `balances/MinimalSwapInfoPoolsBalance.sol`

\`\`

### `vault/Swaps.sol`

* Now that we have our token balances, we continue to [`_callMinimalSwapInfoPoolOnSwapHook()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L349-L355)\`\`

```
uint256 tokenInTotal = tokenInBalance.total();
uint256 tokenOutTotal = tokenOutBalance.total();
request.lastChangeBlock = Math.max(tokenInBalance.lastChangeBlock(), tokenOutBalance.lastChangeBlock());

// Perform the swap request callback, and compute the new balances for 'token in' and 'token out' after the swap
amountCalculated = pool.onSwap(request, tokenInTotal, tokenOutTotal);
(uint256 amountIn, uint256 amountOut) = _getAmounts(request.kind, request.amount, amountCalculated);
```

* Our `tokenInBalance` and `tokenOutBalance` arguments are `bytes32`s that contain encoded data including amounts both held by the Vault and held externally via Asset Managers. We [add those two amounts together](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L378-L379) so that we can make trades based on the full amount that is under the pool's management. In the following line, we also grab the `lastChangeBlocks` for each in/out token.
* Now at long last, we finally call the pool's [`onSwap()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L383) hook.

### `pools/BaseMinimalSwapInfoPool.sol`

```
// Fees are subtracted before scaling, to reduce the complexity of the rounding direction analysis.
request.amount = _subtractSwapFeeAmount(request.amount);

// All token amounts are upscaled.
balanceTokenIn = _upscale(balanceTokenIn, scalingFactorTokenIn);
balanceTokenOut = _upscale(balanceTokenOut, scalingFactorTokenOut);
request.amount = _upscale(request.amount, scalingFactorTokenIn);

uint256 amountOut = _onSwapGivenIn(request, balanceTokenIn, balanceTokenOut);
```

* For a WeightedPool, this brings us to [`BaseMinimalSwapInfoPool.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BaseMinimalSwapInfoPool.sol#L54), which [collects the swap fee](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BaseMinimalSwapInfoPool.sol#L64), [scales balances/amount](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BaseMinimalSwapInfoPool.sol#L67-L69) for token decimals, and calls the pool's [`_onSwapGivenIn()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BaseMinimalSwapInfoPool.sol#L71).

### `pools/weighted/WeightedPool.sol`

```
return
    WeightedMath._calcOutGivenIn(
        currentBalanceTokenIn,
        _normalizedWeight(swapRequest.tokenIn),
        currentBalanceTokenOut,
        _normalizedWeight(swapRequest.tokenOut),
        swapRequest.amount
    );
```

* The WeightedPool's [`_onSwapGivenIn()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L165-L180) simply calls WeightedMath's `_calcOutGivenIn()` with pool balances, weights, and the input amount.

### `pools/weighted/WeightedMath.sol`

```
/**********************************************************************************************
// outGivenIn                                                                                //
// aO = amountOut                                                                            //
// bO = balanceOut                                                                           //
// bI = balanceIn              /      /            bI             \    (wI / wO) \           //
// aI = amountIn    aO = bO * |  1 - | --------------------------  | ^            |          //
// wI = weightIn               \      \       ( bI + aI )         /              /           //
// wO = weightOut                                                                            //
**********************************************************************************************/

// Amount out, so we round down overall.

// The multiplication rounds down, and the subtrahend (power) rounds up (so the base rounds up too).
// Because bI / (bI + aI) <= 1, the exponent rounds down.

// Cannot exceed maximum in ratio
_require(amountIn <= balanceIn.mulDown(_MAX_IN_RATIO), Errors.MAX_IN_RATIO);

uint256 denominator = balanceIn.add(amountIn);
uint256 base = balanceIn.divUp(denominator);
uint256 exponent = weightIn.divDown(weightOut);
uint256 power = base.powUp(exponent);

return balanceOut.mulDown(power.complement());
```

* A detailed explanation of [WeightedMath](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedMath.sol)'s [`_calcOutGivenIn()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedMath.sol#L69-L100) is outside of the scope of this code dive, but in short it maintains the constant product invariant for the pool. The formula is explained [in the comments](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedMath.sol#L76-L84) and is derived on Page 6 of the [Balancer Whitepaper](https://balancer.fi/whitepaper.pdf).

### `pools/BaseMinimalSwapInfoPool.sol`

```
uint256 amountOut = _onSwapGivenIn(request, balanceTokenIn, balanceTokenOut);

// amountOut tokens are exiting the Pool, so we round down.
return _downscaleDown(amountOut, scalingFactorTokenOut);
```

* Digging ourselves back out up the call stack from `WeightedMath.sol` and `WeightedPool.sol`, we return the scaled down (and rounded down) amount from [`BaseMinimalInfoPool.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BaseMinimalSwapInfoPool.sol#L74).

### `vault/Swaps.sol`

#### Back in Swaps.sol!

```
amountCalculated = pool.onSwap(request, tokenInTotal, tokenOutTotal);
(uint256 amountIn, uint256 amountOut) = _getAmounts(request.kind, request.amount, amountCalculated);

newTokenInBalance = tokenInBalance.increaseCash(amountIn);
newTokenOutBalance = tokenOutBalance.decreaseCash(amountOut);
```

* After getting our amount from the pool back, we [do some bookkeeping](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L384) for which token is the input and which is the output. This step is necessary since we need to account for differences in `GIVEN_IN` and `GIVEN_OUT` trades. We [calculate what the new balances ](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L386-L387)will be in the pool once the swap is finished.

#### Updating the Vault's Pool Balances

```
_minimalSwapInfoPoolsBalances[request.poolId][request.tokenIn] = tokenInBalance;
_minimalSwapInfoPoolsBalances[request.poolId][request.tokenOut] = tokenOutBalance;
```

* With these values, we now [update the Vault's balances for each pool](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L357-L358), which live in [`MinimalSwapInfoPoolsBalance.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/balances/MinimalSwapInfoPoolsBalance.sol#L39).

#### Emitting the Swap Event

```
(amountIn, amountOut) = _getAmounts(request.kind, request.amount, amountCalculated);
emit Swap(request.poolId, request.tokenIn, request.tokenOut, amountIn, amountOut);
```

* Going back up the call stack, we're now back in `_swapWithPool()` where [we emit a `Swap` event](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L296-L297) that reports relevant swap data.

#### Accumulate the `assetDeltas`

```
(previousAmountCalculated, amountIn, amountOut) = _swapWithPool(poolRequest);

previousTokenCalculated = _tokenCalculated(kind, tokenIn, tokenOut);

// Accumulate Vault deltas across swaps
assetDeltas[batchSwapStep.assetInIndex] = assetDeltas[batchSwapStep.assetInIndex].add(amountIn.toInt256());
assetDeltas[batchSwapStep.assetOutIndex] = assetDeltas[batchSwapStep.assetOutIndex].sub(
    amountOut.toInt256()
```

* And now we're back in `_swapWithPools()`, where [we're recording the `previousAmountCalculated`, and the amounts in/out](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L257). We also [determine the previous token](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L259) based on the swap kind (`GIVEN_IN` vs `GIVEN_OUT`)
* Finally, we [accumulate the input/output token changes](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L261-L264) in the `assetDeltas` array. We accumulate them over all swap steps since a `batchSwap` can have multiple steps that use the same tokens multiple times.

#### Onto our Next Swap Step!

* We now enter our second iteration of the loop over all batchSwapSteps. Remember when we said we'd be coming back to [`if(batchSwapStep.amount == 0)`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L233-L242)?

```
// Sentinel value for multihop logic
if (batchSwapStep.amount == 0) {
    // When the amount given is zero, we use the calculated amount for the previous swap, as long as the
    // current swap's given token is the previous calculated token. This makes it possible to swap a
    // given amount of token A for token B, and then use the resulting token B amount to swap for token C.
    _require(i > 0, Errors.UNKNOWN_AMOUNT_IN_FIRST_SWAP);
    bool usingPreviousToken = previousTokenCalculated == _tokenGiven(kind, tokenIn, tokenOut);
    _require(usingPreviousToken, Errors.MALCONSTRUCTED_MULTIHOP_SWAP);
    batchSwapStep.amount = previousAmountCalculated;
}
```

* Now that we're at the second step in our `batchSwap`, we can pass `0` for the amount. Doing so tells the Vault that you want the previous `batchSwapStep` to feed directly into the current one. Therefore, after we verify that we have a previous step and that the previous step's output is our input token for a `GIVEN_IN` (or vice versa for `GIVEN_OUT`), we populate `batchSwapStep.amount = previousAmountCalculated;` From here, we dive back into the individual swap at the pool level. Since we already covered how that works, though, we'll just continue on.

#### Finished with `_swapWithPools()`

* We're now back where we started in [`batchSwap()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L130)!

```
// Process asset deltas, by either transferring assets from the sender (for positive deltas) or to the recipient
// (for negative deltas).
uint256 wrappedEth = 0;
for (uint256 i = 0; i < assets.length; ++i) {
    IAsset asset = assets[i];
    int256 delta = assetDeltas[i];
    _require(delta <= limits[i], Errors.SWAP_LIMIT);

    if (delta > 0) {
        uint256 toReceive = uint256(delta);
        _receiveAsset(asset, toReceive, funds.sender, funds.fromInternalBalance);

        if (_isETH(asset)) {
            wrappedEth = wrappedEth.add(toReceive);
        }
    } else if (delta < 0) {
        uint256 toSend = uint256(-delta);
        _sendAsset(asset, toSend, funds.recipient, funds.toInternalBalance);
    }
}

// Handle any used and remaining ETH.
_handleRemainingEth(wrappedEth);
```

* Finally, we process all of the `assetDeltas`. First, we [verify that all deltas are within their user-defined limits](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L136-L138), and then we call [receive and send assets](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/Swaps.sol#L140-L150) at `_receiveAsset()` on each token the user needs to send to the Vault, and `_sendAsset()` on each token the Vault needs to send to the user. If there is any remaining ETH left over at the end of the operation, `_handleRemainingEth()` sends it back to the calling contract (_not_ `msg.sender`, so relayers must handle this properly!). All three of these functions are handled by [AssetTransfersHandler.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/AssetTransfersHandler.sol) and are pretty self explanatory. `send` sends, `receive` receives, and `handleRemaining` deals with any excess ETH that needs to go back to the user.

## Fin

And that's how a `batchSwap` works! I invite you to explore the codebase to see how different pool specializations and pool types behave in their own ways. For example, `StablePool`s is a `GENERAL` specialization so all token balances influence the swap prices.
