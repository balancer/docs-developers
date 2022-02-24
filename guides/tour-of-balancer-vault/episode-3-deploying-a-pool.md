# Episode 3: Deploying a Pool

## Intro

Here we'll explore the inner workings of calling `create()` on a pool factory in Balancer V2. We'll follow the process all the way from the top level call at the WeightedPoolFactory to the underlying calls that make everything work.

## Scenario

A user wants a pool that has their favorite tokens in it, but can't find an existing version. They therefore design their own WeightedPool with different tokens and token weights.

## The Code

If you'd like to follow along with the source code, file names will be relative to [contracts folder on the weighted-deployment tag of the Balancer V2 Monorepo](https://github.com/balancer-labs/balancer-v2-monorepo/tree/weighted-deployment/contracts).

### `pools/weighted/WeightedPoolFactory.sol`

```
/**
 * @dev Deploys a new `WeightedPool`.
 */
function create(
    string memory name,
    string memory symbol,
    IERC20[] memory tokens,
    uint256[] memory weights,
    uint256 swapFeePercentage,
    address owner
) external returns (address) {
    (uint256 pauseWindowDuration, uint256 bufferPeriodDuration) = getPauseConfiguration();

    address pool = address(
        new WeightedPool(
            getVault(),
            name,
            symbol,
            tokens,
            weights,
            swapFeePercentage,
            pauseWindowDuration,
            bufferPeriodDuration,
            owner
        )
    );
    _register(pool);
    return pool;
}
```

Right away, we need to check the pool factory's pause configuration. Factories have time-based pause configurations so that if a bug or vulnerability is discovered in a `poolType` soon after a pool factory has been deployed, pools can be paused. When paused, no one can join or swap with the pool, but Liquidity Providers are able to withdraw their tokens.

### `pools/factories/FactoryWidePauseWindow.sol`

```
function getPauseConfiguration() public view returns (uint256 pauseWindowDuration, uint256 bufferPeriodDuration) {
    uint256 currentTime = block.timestamp;
    if (currentTime < _poolsPauseWindowEndTime) {
        // The buffer period is always the same since its duration is related to how much time is needed to respond
        // to a potential emergency. The Pause Window duration however decreases as the end time approaches.

        pauseWindowDuration = _poolsPauseWindowEndTime - currentTime; // No need for checked arithmetic.
        bufferPeriodDuration = _BUFFER_PERIOD_DURATION;
    } else {
        // After the end time, newly created Pools have no Pause Window, nor Buffer Period (since they are not
        // pausable in the first place).

        pauseWindowDuration = 0;
        bufferPeriodDuration = 0;
    }
}
```

As we're creating a pool, we check to see if the pool factory's pause period has ended. If it has, we set the `pauseWindowDuration` and `bufferPeriodDuration` to `0`. If the pause period is still active, we set the `pauseWindowDuration` to be the amount of time between now and when the factory's pause period expires. For the `bufferPeriodDuration`, we set this to `_BUFFER_PERIOD_DURATION` so that if a pool factory's pause period is about to end and a pause is issued, the pool can stay paused for long enough to respond to the emergency. `_BUFFER_PERIOD_DURATION` is [defined as 30 days](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/factories/FactoryWidePauseWindow.sol#L29).

### `pools/weighted/WeightedPool.sol`

We now move to the [WeightedPool constructor](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L52-L103):

```
BaseMinimalSwapInfoPool(
    vault,
    name,
    symbol,
    tokens,
    swapFeePercentage,
    pauseWindowDuration,
    bufferPeriodDuration,
    owner
)
```

We immediately are sent to the `BaseMinimalSwapInfoPool` constructor, as that is the WeightedPool's specialization.

### `pools/BaseMinimalSwapInfoPool.sol`

```
BasePool(
    vault,
    tokens.length == 2 ? IVault.PoolSpecialization.TWO_TOKEN : IVault.PoolSpecialization.MINIMAL_SWAP_INFO,
    name,
    symbol,
    tokens,
    swapFeePercentage,
    pauseWindowDuration,
    bufferPeriodDuration,
    owner
)
```

Which of course sends us to BasePool.

### `pools/BasePool.sol`

```
Authentication(bytes32(uint256(msg.sender)))
BalancerPoolToken(name, symbol)
BasePoolAuthorization(owner)
TemporarilyPausable(pauseWindowDuration, bufferPeriodDuration)
```

In the constructor, the BasePool also calls the constructors for the four contracts from which it inherits.

### `lib/helpers/Authentication.sol`

```
constructor(bytes32 actionIdDisambiguator) {
    _actionIdDisambiguator = actionIdDisambiguator;
}
```

This simply stuffs `bytes32(uint256(msg.sender))` as the `_actionIdDisambiguator`, where msg.sender is the pool factory address.

### `pools/BalancerPoolToken.sol`

```
constructor(string memory tokenName, string memory tokenSymbol) EIP712(tokenName, "1") {
    _name = tokenName;
    _symbol = tokenSymbol;
}
```

This sets the ERC20 Balancer Pool Token (BPT) name and symbol for the pool according to the user's specifications.

### `pools/BasePoolAuthorization.sol`

```
constructor(address owner) {
    _owner = owner;
}
```

This sets the owner as defined by whoever is launching the pool. This contract grants authorization to owners as well as addresses that have been granted authorization by governance.

### `lib/helpers/TemporarilyPausable.sol`

```
constructor(uint256 pauseWindowDuration, uint256 bufferPeriodDuration) {
    _require(pauseWindowDuration <= _MAX_PAUSE_WINDOW_DURATION, Errors.MAX_PAUSE_WINDOW_DURATION);
    _require(bufferPeriodDuration <= _MAX_BUFFER_PERIOD_DURATION, Errors.MAX_BUFFER_PERIOD_DURATION);

    uint256 pauseWindowEndTime = block.timestamp + pauseWindowDuration;

    _pauseWindowEndTime = pauseWindowEndTime;
    _bufferPeriodEndTime = pauseWindowEndTime + bufferPeriodDuration;
}
```

This sets the previously determined `pauseWindowDuration` and `bufferPeriodDuration`, which can be later be checked by the pool if an emergency does arise meriting a pause.

### `pools/BasePool.sol`

```
_require(tokens.length >= _MIN_TOKENS, Errors.MIN_TOKENS);
_require(tokens.length <= _MAX_TOKENS, Errors.MAX_TOKENS);
...
InputHelpers.ensureArrayIsSorted(tokens);
```

Now that we've handled all our inherited responsibilities, we start off doing some checks to make sure we have an appropriate number of tokens and that the token addresses are sorted. Token sorting is mandatory for some pools (Two Token Pools specialization) on a technical standpoint, but is enforced for all pools for the sake of standardization.

```
_setSwapFeePercentage(swapFeePercentage);
```

We next set the swap fee as defined by the pool creator:

```
    function _setSwapFeePercentage(uint256 swapFeePercentage) private {
        _require(swapFeePercentage >= _MIN_SWAP_FEE_PERCENTAGE, Errors.MIN_SWAP_FEE_PERCENTAGE);
        _require(swapFeePercentage <= _MAX_SWAP_FEE_PERCENTAGE, Errors.MAX_SWAP_FEE_PERCENTAGE);

        _swapFeePercentage = swapFeePercentage;
        emit SwapFeePercentageChanged(swapFeePercentage);
    }
```

Which simply checks the desired swap fee against the global min/max values (0.0001% and 10% respectively), updates the value, and emits the `SwapFeePercentageChanged` event.

Back to the BasePool constructor, we now register the pool with the Vault

```
bytes32 poolId = vault.registerPool(specialization);
```

Which happens in the [PoolRegistry.sol](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolRegistry.sol#L67-L87) sub-contract of the Vault.

### `vault/PoolRegistry.sol`

```
bytes32 poolId = _toPoolId(msg.sender, specialization, uint80(_nextPoolNonce));

_require(!_isPoolRegistered[poolId], Errors.INVALID_POOL_ID); // Should never happen as Pool IDs are unique.
_isPoolRegistered[poolId] = true;

_nextPoolNonce += 1;

// Note that msg.sender is the pool's contract
emit PoolRegistered(poolId, msg.sender, specialization);
return poolId;
```

To register the pool, we must first determine its `poolId`. The `poolId` is a unique encoded identifier that refers to each pool.

```
    function _toPoolId(
        address pool,
        PoolSpecialization specialization,
        uint80 nonce
    ) internal pure returns (bytes32) {
        bytes32 serialized;

        serialized |= bytes32(uint256(nonce));
        serialized |= bytes32(uint256(specialization)) << (10 * 8);
        serialized |= bytes32(uint256(pool)) << (12 * 8);

        return serialized;
    }
```

The `poolId` encodes three elements: the pool's contract `address`, the `PoolSpecialization`, and a `nonce` (to avoid `poolId` collisions). These are bit-packed into a single `bytes32` and returned to [`registerPool()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolRegistry.sol#L67-L87).

registerPool() then verifies we have crafted a valid `poolId`, adds the pool to the `isPoolRegistered` mapping, and increments the `_nextPoolNonce`. After emitting the PoolRegistered event, we return the `poolId` and return to the `BasePool` constructor.

### `pools/BasePool.sol`

```
vault.registerTokens(poolId, tokens, new address[](tokens.length));
```

After registering our pool, we now need to register the tokens that are _in_ the pool. This happens in the [`PoolTokens.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolTokens.sol#L29-L56) sub-contract of the Vault.

### `vault/PoolTokens.sol`

```
InputHelpers.ensureInputLengthMatch(tokens.length, assetManagers.length);

// Validates token addresses and assigns Asset Managers
for (uint256 i = 0; i < tokens.length; ++i) {
    IERC20 token = tokens[i];
    _require(token != IERC20(0), Errors.INVALID_TOKEN);

    _poolAssetManagers[poolId][token] = assetManagers[i];
}

PoolSpecialization specialization = _getPoolSpecialization(poolId);
if (specialization == PoolSpecialization.TWO_TOKEN) {
    _require(tokens.length == 2, Errors.TOKENS_LENGTH_MUST_BE_2);
    _registerTwoTokenPoolTokens(poolId, tokens[0], tokens[1]);
} else if (specialization == PoolSpecialization.MINIMAL_SWAP_INFO) {
    _registerMinimalSwapInfoPoolTokens(poolId, tokens);
} else {
    // PoolSpecialization.GENERAL
    _registerGeneralPoolTokens(poolId, tokens);
}

emit TokensRegistered(poolId, tokens, assetManagers);
```

After checking that the input lengths match and the token addresses provided do indeed define ERC20 token contracts, we set all of the pool's asset managers to the zero address.

{% hint style="info" %}
**Why are all the asset managers set to the zero address?**

When `BasePool` calls `registerTokens()`, we pass `new address[](tokens.length)` for the `AssetManagers`.

We do this to disable AssetManagers for all pools derived from BasePool (as of this writing) since they have such tremendous power over assets in pools. There very well may be future applications that open up this design space, but for now the base contract disables it for safety.
{% endhint %}

Next we move on to registering our tokens based on our `PoolSpecialization`. Since a WeightedPool is of type `MINIMAL_SWAP_INFO`, we end up calling [`_registerMinimalSwapInfoPoolTokens()`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/balances/MinimalSwapInfoPoolsBalance.sol#L52-L61).

### `vault/balances/MinimalSwapInfoPoolsBalance.sol`

```
function _registerMinimalSwapInfoPoolTokens(bytes32 poolId, IERC20[] memory tokens) internal {
    EnumerableSet.AddressSet storage poolTokens = _minimalSwapInfoPoolsTokens[poolId];

    for (uint256 i = 0; i < tokens.length; ++i) {
        bool added = poolTokens.add(address(tokens[i]));
        _require(added, Errors.TOKEN_ALREADY_REGISTERED);
        // Note that we don't initialize the balance mapping: the default value of zero corresponds to an empty
        // balance.
    }
}
```

First, we define `poolTokens`, which points to our `poolId`'s entry in `_minimalSwapInfoPoolsTokens`, which is defined [here](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/balances/MinimalSwapInfoPoolsBalance.sol#L40). We then iterate through all of our new `tokens`, and verify that they aren't already registered with the pool. As the comment notes, we do not initialize any `balance` mappings since the default values are zero. We now hop back up to `registerTokens()` in [`PoolTokens.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/vault/PoolTokens.sol#L29-L56).

### `vault/PoolTokens.sol`

```
emit TokensRegistered(poolId, tokens, assetManagers);
```

Not much left to do here other than emit our `TokensRegistered` event to announce our new pool's `poolId`, newly registered `tokens`, and assetManagers. We now hop back up to the BasePool constructor.

### `pools/BasePool.sol`

```
// Set immutable state variables - these cannot be read from during construction
_vault = vault;
_poolId = poolId;
_totalTokens = tokens.length;

// Immutable variables cannot be initialized inside an if statement, so we must do conditional assignments
_token0 = tokens.length > 0 ? tokens[0] : IERC20(0);
_token1 = tokens.length > 1 ? tokens[1] : IERC20(0);
_token2 = tokens.length > 2 ? tokens[2] : IERC20(0);
_token3 = tokens.length > 3 ? tokens[3] : IERC20(0);
_token4 = tokens.length > 4 ? tokens[4] : IERC20(0);
_token5 = tokens.length > 5 ? tokens[5] : IERC20(0);
_token6 = tokens.length > 6 ? tokens[6] : IERC20(0);
_token7 = tokens.length > 7 ? tokens[7] : IERC20(0);

_scalingFactor0 = tokens.length > 0 ? _computeScalingFactor(tokens[0]) : 0;
_scalingFactor1 = tokens.length > 1 ? _computeScalingFactor(tokens[1]) : 0;
_scalingFactor2 = tokens.length > 2 ? _computeScalingFactor(tokens[2]) : 0;
_scalingFactor3 = tokens.length > 3 ? _computeScalingFactor(tokens[3]) : 0;
_scalingFactor4 = tokens.length > 4 ? _computeScalingFactor(tokens[4]) : 0;
_scalingFactor5 = tokens.length > 5 ? _computeScalingFactor(tokens[5]) : 0;
_scalingFactor6 = tokens.length > 6 ? _computeScalingFactor(tokens[6]) : 0;
_scalingFactor7 = tokens.length > 7 ? _computeScalingFactor(tokens[7]) : 0;
```

{% hint style="info" %}
This inefficient-looking code is a necessity because Solidity (as of this writing) lacks native support for dynamic `immutable` arrays. There is an [open issue in the Solidity repo](https://github.com/ethereum/solidity/issues/12587) to add this functionality and improve the appearance/syntax of code blocks like this for future Solidity compiler versions.
{% endhint %}

Now we start shoving data into our immutable variables. The `vault`, `poolId`, and `tokens.length` should be self-explanatory. The lines for `_tokens<i>` fill corresponding `tokens` from the user-defined dynamic array. If the users passes fewer than 8 tokens, the `immutable` `_token<i>` variables are filled with the zero address.

The `_scalingFactor<i>` lines follow a similar pattern, but for the scaling factors of the tokens. The Vault internally bookkeeps `balances` as if each token has 18 decimals, but that is not necessarily true. The scaling factors are kept to scale up/down tokens to hit this 18 decimal standard.

```
    function _computeScalingFactor(IERC20 token) private view returns (uint256) {
        // Tokens that don't implement the `decimals` method are not supported.
        uint256 tokenDecimals = ERC20(address(token)).decimals();

        // Tokens with more than 18 decimals are not supported.
        uint256 decimalsDifference = Math.sub(18, tokenDecimals);
        return 10**decimalsDifference;
    }
```

The scaling factor for a token with $$x \leq18$$ decimals ends up being $$10^{18-x}$$.

This brings us to the end of the `BasePool` constructor, which pops us back up to the `BaseMinimalSwapInfoPool` constructor, which is also now complete. We therefore now make our way back to the `WeightedPool` constructor in [`WeightedPool.sol`](https://github.com/balancer-labs/balancer-v2-monorepo/blob/weighted-deployment/contracts/pools/weighted/WeightedPool.sol#L52-L103).

### `pools/weighted/WeightedPool.sol`

```
uint256 numTokens = tokens.length;
InputHelpers.ensureInputLengthMatch(numTokens, normalizedWeights.length);

// Ensure  each normalized weight is above them minimum and find the token index of the maximum weight
uint256 normalizedSum = 0;
uint256 maxWeightTokenIndex = 0;
uint256 maxNormalizedWeight = 0;
for (uint8 i = 0; i < numTokens; i++) {
    uint256 normalizedWeight = normalizedWeights[i];
    _require(normalizedWeight >= _MIN_WEIGHT, Errors.MIN_WEIGHT);

    normalizedSum = normalizedSum.add(normalizedWeight);
    if (normalizedWeight > maxNormalizedWeight) {
        maxWeightTokenIndex = i;
        maxNormalizedWeight = normalizedWeight;
    }
}
// Ensure that the normalized weights sum to ONE
_require(normalizedSum == FixedPoint.ONE, Errors.NORMALIZED_WEIGHT_INVARIANT);

_maxWeightTokenIndex = maxWeightTokenIndex;
_normalizedWeight0 = normalizedWeights.length > 0 ? normalizedWeights[0] : 0;
_normalizedWeight1 = normalizedWeights.length > 1 ? normalizedWeights[1] : 0;
_normalizedWeight2 = normalizedWeights.length > 2 ? normalizedWeights[2] : 0;
_normalizedWeight3 = normalizedWeights.length > 3 ? normalizedWeights[3] : 0;
_normalizedWeight4 = normalizedWeights.length > 4 ? normalizedWeights[4] : 0;
_normalizedWeight5 = normalizedWeights.length > 5 ? normalizedWeights[5] : 0;
_normalizedWeight6 = normalizedWeights.length > 6 ? normalizedWeights[6] : 0;
_normalizedWeight7 = normalizedWeights.length > 7 ? normalizedWeights[7] : 0;
```

We now grab the numTokens for easy reference, and ensure that the number of `normalizedWeights` we're about to analyze are equal to the number of `tokens` we just registered. Before looping through all the weights, we declare the following variables that we want to accumulate/keep track of:

* `normalizedSum`
  * We want to make sure the sum of all weights = 1
* `maxWeightTokenIndex`
  * We want to keep track of the highest weighted token index (for collecting protocol fees)
* `maxNormalizedWeight`
  * We need to maintain what the current highest weight for a token is so that we can properly keep track of the `maxWeightTokenIndex`

As we loop through all of our tokens, we make sure the weights are within the allowed ranges (1%-99%), add the current weight to the `normalizedSum`, and check to see if we need to update the current `maxWeightTokenIndex` and `maxNormalizedWeight`, doing so if necessary.

The lines for `_normalizedWeight<i>` might look quite similar to those for handling `tokens` and `scalingFactor`s in the `BasePool` constructor. Just like in those cases, these variables are `immutable` and therefore can't be stored in an array. Similarly, if we have fewer than 8 tokens, the `_normalizedWeight<i>` for $$i>normalizedWeights.length$$ are set to zero.

## Fin

And that's how a pool deployment works! I invite you to explore the codebase to see how different pool specializations and pool types behave in their own ways. Some pools, like PhantomStablePools and ManagedPools have some powerful differences like minting BPT at the time of pool creation and high levels of pool-owner power.

You may be wondering why there were no token transfers to handle like in the `batchSwap` and `joinPool` guides -- this is because after deploying pools, you need to seed them with tokens in a separate step. In most pools, this is done with a `joinPool` of type `INIT`, but in some pools (ie those with pre-minted/phantom BPT) this is done via swapping.

If you'd like to see how the INIT join is handled, go through Episode 2: Joins and take the INIT Join Detour.
