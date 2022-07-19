# Encoding userData

There are some arguments that need to be encoded when interacting with Balancer V2 Smart Contracts. For convenience, this page provides an example of how to encode arguments.

{% tabs %}
{% tab title="Solidity" %}
```cpp
uint256 JoinKindInit = 0;
uint256[] memory initBalances = new uint256[](2);
initBalances[0] = 1e18;
initBalances[1] = 2e18;
bytes userDataEncoded = abi.encode(JoinKindInit, initBalances);
```
{% endtab %}

{% tab title="js" %}
```javascript
import { defaultAbiCoder } from '@ethersproject/abi';

const JoinKindInit = 0;
const initBalances = [1e18, 2e18];
const abi = ['uint256', 'uint256[]'];
const data = [JoinKindInit, initBalances];
const userDataEncoded = defaultAbiCoder.encode(abi,data);
```
{% endtab %}

{% tab title="Python" %}
```python
import eth_abi

JoinKindInit = 0
initBalances = [1e18, 2e18]
abi = ['uint256', 'uint256[]']
data = [JoinKindInit, initBalances]
userDataEncoded = eth_abi.encode_abi(abi, data)
```
{% endtab %}
{% endtabs %}
