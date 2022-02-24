# Detour: The INIT Join

{% hint style="warning" %}
If you made your way here randomly or after finishing [Episode 2: Joins](./), this may seem a bit discontinuous. This page is intended as a detour [during the `onJoinPool()` call](./#pools-basepool.sol-onjoinpool) in `BasePool`.

If you're not interested in `INIT` joins, hop on over to [Episode 3: Deploying a Pool](../episode-3-deploying-a-pool.md)
{% endhint %}

## The Code

### `pools/BasePool.sol`

```
if (totalSupply() == 0) {
    (uint256 bptAmountOut, uint256[] memory amountsIn) = _onInitializePool(poolId, sender, recipient, userData);
    ...
} else {
    ...
}
```

Now that we're interested in the first join on a pool, we enter the `totalSupply() == 0` if block. The first call we make it to `_onInitializePool()`. This is a [virtual function in BasePool](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BasePool.sol#L374-L379), but it is [implemented in WeightedPool](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L201-L228).

### `pools/weighted/WeightedPool.sol`

```
WeightedPool.JoinKind kind = userData.joinKind();
_require(kind == WeightedPool.JoinKind.INIT, Errors.UNINITIALIZED);
```

First in `_onInitializePool()`, we make sure that the user has sent a join of `JoinKind` `INIT`. This `JoinKind` is different from others since users do _not_ provide a BPT output amount/limit; the pool calculates the precise amount of BPT out since no one can front-run this operation.

```
uint256[] memory amountsIn = userData.initialAmountsIn();
```

Next we decode the `amountsIn` from the `userData` in [`initialAmountsIn()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPoolUserDataHelpers.sol#L32-L34).

### `pools/weighted/WeightedPoolUserDataHelpers.sol`

```
function initialAmountsIn(bytes memory self) internal pure returns (uint256[] memory amountsIn) {
    (, amountsIn) = abi.decode(self, (WeightedPool.JoinKind, uint256[]));
}
```

This function is simply decoding the `amountsIn` from `bytes` into a `uint256[]` array.

### `pools/weighted/WeightedPool.sol`

#### `_onInitializePool()`

```
uint256[] memory amountsIn = userData.initialAmountsIn();
InputHelpers.ensureInputLengthMatch(_getTotalTokens(), amountsIn.length);
_upscaleArray(amountsIn, _scalingFactors());

uint256[] memory normalizedWeights = _normalizedWeights();
```

After getting our amountsIn, we verify their lengths and upscale them according to the decimals we stored during pool creation. Next we move to get `normalizedWeights` in [`_normalizedWeights()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L120-L137).

#### `_normalizedWeights()`

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

First, we call `_getTotalTokens()` in [`BasePool.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BasePool.sol#L163-L165), which simply returns `_totalTokens`. From there, we take our immutable variables `normalizedWeights<i>` and pack them into the dynamic array `normalizedWeights[]`, returning the array as soon as it's been filled with all the corresponding weights for `totalTokens`.

#### `_onInitializePool()`

```
uint256 invariantAfterJoin = WeightedMath._calculateInvariant(normalizedWeights, amountsIn);

// Set the initial BPT to the value of the invariant times the number of tokens. This makes BPT supply more
// consistent in Pools with similar compositions but different number of tokens.
uint256 bptAmountOut = Math.mul(invariantAfterJoin, _getTotalTokens());

_lastInvariant = invariantAfterJoin;

return (bptAmountOut, amountsIn);
```

Now that we have our `normalizedWeights` and we've decoded our `amountsIn`, we have everything we need to calculate the starting invariant. This is calculated in [WeightedMath.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedMath.sol#L47-L65), where it puts these amount and weight values into the generalized constant product formula, $$\prod_{i=0}^{n}{{B_i}^{w_i}}$$. Next, the starting BPT amount, `bptAmountOut`, is calculated by multiplying the just-determined invariant with the number of tokens in the pool (from `_getTotalTokens()` in [`BasePool.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BasePool.sol#L163-L165), again).

Now we set `_lastInvariant` to the invariant we just calculated. This value is saved for later in order to determine due protocol fees. Finally, we return our `bptAmountOut` and `amountsIn` and hop back up to [`onJoinPool()` in `BasePool.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/BasePool.sol#L197-L245).

### `pools/BasePool.sol`

```
if (totalSupply() == 0) {
    (uint256 bptAmountOut, uint256[] memory amountsIn) = _onInitializePool(poolId, sender, recipient, userData);

    // On initialization, we lock _MINIMUM_BPT by minting it for the zero address. This BPT acts as a minimum
    // as it will never be burned, which reduces potential issues with rounding, and also prevents the Pool from
    // ever being fully drained.
    _require(bptAmountOut >= _MINIMUM_BPT, Errors.MINIMUM_BPT);
    _mintPoolTokens(address(0), _MINIMUM_BPT);
    _mintPoolTokens(recipient, bptAmountOut - _MINIMUM_BPT);

    // amountsIn are amounts entering the Pool, so we round up.
    _downscaleUpArray(amountsIn, scalingFactors);

    return (amountsIn, new uint256[](_getTotalTokens()));
} else {
    ...
}
```

We now mint BPT to two addresses: the user performing the `INIT` join, and the zero address. The zero address gets `_MINIMUM_BPT` (=1e6 wei) to prevent rounding issues and avoiding total pool drain. This is taken from the recipient's share, and is considered to be negligible.

We now downscale the amounts according to their decimals, and return the `amountsIn` alongside a newly crafted array of zeros, which signify that the INIT join is incurring no protocol fees.

## Fin

Now that we've completed our detour, feel free to hop back over to the [`onJoinPool()` call in `BasePool`](./#pools-basepool.sol-onjoinpool).
