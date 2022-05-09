# BalancerHelpers

### `queryJoin`

```
queryJoin(
    bytes32 poolId, 
    address sender, 
    address recipient, 
    JoinPoolRequest request)
returns (uint256 bptOut, uint256[] amountsIn)
```

### `queryExit`

```
queryExit(
    bytes32 poolId, 
    address sender, 
    address recipient, 
    ExitPoolRequest request)
returns (uint256 bptIn, uint256[] amountsOut)
```

For more information on `JoinPoolRequest` and `ExitPoolRequest`, please refer to the respective pages that dive into [joins](../../../resources/joins-and-exits/pool-joins.md#api) and [exits](../../../resources/joins-and-exits/pool-exits.md#api).&#x20;
