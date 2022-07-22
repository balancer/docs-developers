# Single Swaps

## Why should I use a single swap?

You'll want to use **Single Swaps** when you're making a trade between two tokens in one pool. While it's possible to do this with a one-step Batch Swap, using a **Single Swap** will save \~6,000 gas.

For additional information, check out the [Single Swap page in Resources](../../resources/swaps/single-swap.md).

## Code Walkthrough

Select your desired programming language in the tabs below for the relevant tutorial.

{% tabs %}
{% tab title="js" %}
{% hint style="info" %}
This script assumes that the address you're using has already granted token spend allowances to the Vault. If you do not do that first, **your transaction will fail**.
{% endhint %}

Let's step through a [JavaScript Single Swap example](https://github.com/gerrrg/balancer-tutorials/blob/master/js/swaps/single\_swap.js) chunk by chunk.

#### Dependencies

```javascript
const Web3 = require("web3");
const fs = require("fs");
const BigNumber = require("bignumber.js");
const Tx = require('ethereumjs-tx').Transaction;
const open = require('open');
```

This sample relies on [web3.js](https://web3js.readthedocs.io) to interact with on-chain Smart Contracts and a few other libraries. These dependencies can be found in the [package.json](https://github.com/gerrrg/balancer-tutorials/blob/master/js/swaps/package.json) file in the JavaScript sample repository.&#x20;

#### Connecting to RPC and setting up your account

```javascript
// Load private key and connect to RPC endpoint
const rpc_endpoint = process.env.RPC_ENDPOINT;
const private_key = process.env.KEY_PRIVATE;
if (rpc_endpoint == undefined || private_key == undefined || private_key == "") {
    throw new Error("You must set environment variables for RPC_ENDPOINT and KEY_PRIVATE");
}
const web3 = new Web3(new Web3.providers.HttpProvider(rpc_endpoint));
const account = web3.eth.accounts.privateKeyToAccount(private_key);
const address = account.address;

// Define network settings
const network = "kovan";
const block_explorer_url = "https://kovan.etherscan.io/";
const chain_id = "42";
const gas_price = "2";
```

This section connects to the Ethereum (or other EVM compatible) blockchain using the **RPC\_ENDPOINT** environment variable provided either in the shell, or in the [bash script helper](https://github.com/gerrrg/balancer-tutorials/blob/master/js/swaps/sample\_single\_swap.sh). Similarly, it initialized an Ethereum account based on the private key provided with **KEY\_PRIVATE**.&#x20;

We then define some network-specific settings. In this example, we're using the Kovan Testnet. The **block\_explorer\_url** makes it easy to see that status of our transaction in a web browser. Each chain has a different **chain\_id**, so make sure this is set properly for the network you're using. We also manually set a **gas\_price** (denominated in gwei).&#x20;

#### Initializing the Balancer Vault

```javascript
// Load contract for Balancer Vault
const address_vault = "0xBA12222222228d8Ba445958a75a0704d566BF2C8";
const path_abi_vault = "../../abis/Vault.json";
let abi_vault = JSON.parse(fs.readFileSync(path_abi_vault));
const contract_vault = new web3.eth.Contract(abi_vault, address_vault);
```

The sample repository has a copy of the [Balancer V2 Vault ABI](https://github.com/gerrrg/balancer-tutorials/blob/master/abis/Vault.json) that you'll need to interact with the contract itself. On every network that Balancer V2 is officially deployed, the Vault address is the same, and is easily recognizable starting with "**0xBA1222222222"**.&#x20;

This chunk loads the Vault from its on-chain contract address, and its ABI makes it easy to make calls to it directly.

#### Swap Settings

```javascript
// Where are the tokens coming from/going to?
const fund_settings = {
    "sender":               address,
    "recipient":            address,
    "fromInternalBalance":  false,
    "toInternalBalance":    false
};

// When should the transaction timeout?
const deadline = BigNumber(999999999999999999);
```

Here, we're specifying that the sender/recipient for the tokens going into/out of the trade are both the account that we initialized the script with. Note that with this granularity, it is possible to make a swap that sends the tokens to a different address.&#x20;

We specify that {**to**/**from**}**InternalBalance** are both False. This will be the default use case for most users; however, you may have a use case that would benefit from Internal Balances.&#x20;

The deadline for a transaction is the time (in [Unix timestamp](https://www.unixtimestamp.com)) after which it will no longer attempt to make a trade. If a trade expires, it will still take some gas to process the failed transaction, but it will be cheaper than a transaction failing for a different reason.

#### Defining our pools and tokens

```javascript
// Pool IDs
const pool_BAL_WETH = "0x61d5dc44849c9c87b0856a2a311536205c96c7fd000200000000000000000000";

// Token addresses (checksum format)
const token_BAL = "0x41286Bb1D3E870f3F750eB7E1C25d7E48c8A1Ac7".toLowerCase();
const token_WETH = "0xdFCeA9088c8A88A76FF74892C1457C17dfeef9C1".toLowerCase();

// Token data
const token_data = {};
token_data[token_BAL] = 
    {
        "symbol": "BAL",
        "decimals": "18",
        "limit": "0"
    };
token_data[token_WETH] = 
    {
        "symbol": "WETH",
        "decimals": "18",
        "limit": "1"
    };
```

Here, we list our Pool IDs, token contract addresses, and relevant token data. When entering token contract addresses, we enforce that they must be in lowercase to avoid issues with sorting later.

It is important to get your **decimals** set correctly for each token, otherwise you may send far more or far fewer tokens than you intended.

When setting your token limits, you must know that there are two types of swaps: GIVEN\_IN and GIVEN\_OUT. In a GIVEN\_IN swap, the **amount** you provide is the number of your input token you're giving the pool, while for a GIVEN\_OUT swap, you provide the number of tokens that you expect to receive. The limits are used to control the maximum/minimum amounts for the opposite side of the trade that you are not setting exact numbers for.&#x20;

| Swap Type  | **amount**                                | **TokenIn** Limit                                | **TokenOut** Limit                       |
| ---------- | ----------------------------------------- | ------------------------------------------------ | ---------------------------------------- |
| GIVEN\_IN  | Exact amount of TokenIn you will send     | Same as TokenIn amount                           | Minimum amount of TokenOut you'll accept |
| GIVEN\_OUT | Exact amount of TokenOut you will receive | Maximum amount of TokenIn you're willing to give | Same as TokenOut amount                  |

In the above example code, we set our **token\_BAL** limit to 0, which means we are willing to accept 100% slippage on our trade. That is generally a very bad idea, but this is an example on the Kovan testnet, so tokens are valueless here.

#### Swap definition

```javascript
const swap = {
    "poolId": pool_BAL_WETH,
    "assetIn": token_WETH,
    "assetOut": token_BAL,
    "amount": 1
};

// SwapKind is an Enum. This example handles a GIVEN_IN swap.
// https://github.com/balancer-labs/balancer-v2-monorepo/blob/0328ed575c1b36fb0ad61ab8ce848083543070b9/pkg/vault/contracts/interfaces/IVault.sol#L497
// 0 = GIVEN_IN, 1 = GIVEN_OUT
const swap_kind = 0;
```

Next, we define our **swap**. We must specify that this swap is created with a known amount **GIVEN\_IN**; we are _giving ****_ the pool 1 WETH for an estimated output of BAL. It is also possible to create a trade with a fixed amount **GIVEN\_OUT**.

#### Building our structs

```javascript
const swap_struct = {
    poolId: swap["poolId"],
    kind: swap_kind,
    assetIn: web3.utils.toChecksumAddress(swap["assetIn"]),
    assetOut: web3.utils.toChecksumAddress(swap["assetOut"]),
    amount: BigNumber(swap["amount"] * Math.pow(10, token_data[swap["assetIn"]]["decimals"])).toString(),
    userData: '0x'
};

const fund_struct = {
    sender: web3.utils.toChecksumAddress(fund_settings["sender"]),
    fromInternalBalance: fund_settings["fromInternalBalance"],
    recipient: web3.utils.toChecksumAddress(fund_settings["recipient"]),
    toInternalBalance: fund_settings["toInternalBalance"]
};
```

When we call the Vault contract, we need to pack our data into structs, specifically the ones here referred to as **swap\_struct** and __ **fund\_struct**.&#x20;

**swap\_struct**s are of type _SingleSwap_, which is defined here:

```
struct SingleSwap {
   bytes32 poolId;
   SwapKind kind;
   IAsset assetIn;
   IAsset assetOut;
   uint256 amount;
   bytes userData;
}
```

The values that may cause confusion&#x20;

* **amount**: Either the amount of tokens we are sending **to** the pool or want to receive **from** the pool, depending on if the SwapKind is **GIVEN\_IN** or **GIVEN\_OUT**. As shown in the code, make sure that the amount is scaled according to the number of decimals for your token.
* **userData**: Any additional data which the pool requires to perform the swap. This allows pools to have more flexible swapping logic in future - for all current Balancer pools this can be left empty, here entered as '0x'.

**fund\_struct**s are simpler FundManagements structs as defined here:

```
struct FundManagement {
    address sender;
    bool fromInternalBalance;
    address payable recipient;
    bool toInternalBalance;
}
```

The only real "gotcha" here is to make sure your **address**es are in checksum format.

#### Just one more formatting

```javascript
const token_limit = BigNumber((token_data[swap["assetIn"]]["limit"]) * Math.pow(10, token_data[swap["assetIn"]]["decimals"])).toString();
```

Here, we're scaling our **token\_limit** to account for the token-specific decimals.

#### Building the function

```javascript
const single_swap_function = contract_vault.methods.swap(
    swap_struct,
    fund_struct,
    token_limit,
    deadline.toString()
);
```

Here, we're packing our properly formatted structs and other values into the **swap** function.&#x20;

#### Setting the remaining relevant parameters in an async function

```javascript
async function buildAndSend() {
    var gas_estimate;
    try {
        gas_estimate = await single_swap_function.estimateGas();
    }
    catch(err) {
        gas_estimate = 100000;
        console.log("Failed to estimate gas, attempting to send with", gas_estimate, "gas limit...");
    }

    const tx_object = {
        'chainId':  chain_id,
        'gas':      web3.utils.toHex(gas_estimate),
        'gasPrice': web3.utils.toHex(web3.utils.toWei(gas_price,'gwei')),
        'nonce':    await web3.eth.getTransactionCount(address),
        'data':     single_swap_function.encodeABI(),
        'to':       address_vault
    };
```

Here, we attempt to estimate a gas price. In the event it fails, 100k gas is a safe estimate for the gas limit on a **swap**. The remaining lines set the **chainId**, **gas**, **gasPrice**, **nonce, data,** and **to** address.&#x20;

#### Sending and viewing the transaction

```javascript
    const tx = new Tx(tx_object);
    const signed_tx = await web3.eth.accounts.signTransaction(tx_object, private_key)
        .then(signed_tx => web3.eth.sendSignedTransaction(signed_tx['rawTransaction']));
    console.log("Sending transaction...");
    const tx_hash = signed_tx["logs"][0]["transactionHash"];
    const url = block_explorer_url + "tx/" + tx_hash;
    open(url);
}
buildAndSend();
```

Finally, we sign the transaction with our **private\_key**, and broadcast the transaction to be added to the blockchain. As a convenience, the last two lines of the **buildAndSend()** method create a link to Etherscan and opens a tab in the user's default browser.
{% endtab %}

{% tab title="Python" %}
{% hint style="info" %}
This script assumes that the address you're using has already granted token spend allowances to the Vault. If you do not do that first, **your transaction will fail**.
{% endhint %}

Let's step through a [Python Single Swap example](https://github.com/gerrrg/balancer-tutorials/blob/master/python/swaps/single\_swap.py) chunk by chunk.

#### Dependencies

```python
from web3 import Web3
import eth_abi

import os
import json
from decimal import *
import webbrowser
```

This sample relies on [web3.py](https://web3py.readthedocs.io/en/stable/) to interact with on-chain Smart Contracts and a few other libraries. These dependencies can be found in the [requirements.txt](https://github.com/gerrrg/balancer-tutorials/blob/master/python/requirements.txt) file in the Python sample repository.&#x20;

#### Connecting to RPC and setting up your account

```python
# Load private key and connect to RPC endpoint
rpc_endpoint = 	os.environ.get("RPC_ENDPOINT")
private_key =  	os.environ.get("KEY_PRIVATE")
if rpc_endpoint is None or private_key is None or private_key == "":
	print("\n[ERROR] You must set environment variables for RPC_ENDPOINT and KEY_PRIVATE\n")
	quit()
web3 = Web3(Web3.HTTPProvider(rpc_endpoint))
account = web3.eth.account.privateKeyToAccount(private_key)
address = account.address

# Define network settings
network = "kovan"
block_explorer_url = "https://kovan.etherscan.io/"
chain_id = 42
gas_price = 2
```

This section connects to the Ethereum (or other EVM compatible) blockchain using the **RPC\_ENDPOINT** environment variable provided either in the shell, or in the [bash script helper](https://github.com/gerrrg/balancer-tutorials/blob/master/python/swaps/sample\_single\_swap.sh). Similarly, it initialized an Ethereum account based on the private key provided with **KEY\_PRIVATE**.&#x20;

We then define some network-specific settings. In this example, we're using the Kovan Testnet. The **block\_explorer\_url** makes it easy to see that status of our transaction in a web browser. Each chain has a different **chain\_id**, so make sure this is set properly for the network you're using. We also manually set a **gas\_price** (denominated in gwei).&#x20;

#### Initializing the Balancer Vault

```python
# Load contract for Balancer Vault
address_vault = "0xBA12222222228d8Ba445958a75a0704d566BF2C8"
path_abi_vault = '../abis/Vault.json'
with open(path_abi_vault) as f:
  abi_vault = json.load(f)
contract_vault = web3.eth.contract(
	address=Web3.toChecksumAddress(address_vault), 
	abi=abi_vault
)
```

The sample repository has a copy of the [Balancer V2 Vault ABI](https://github.com/gerrrg/balancer-tutorials/blob/master/abis/Vault.json) that you'll need to interact with the contract itself. On every network that Balancer V2 is officially deployed, the Vault address is the same, and is easily recognizable starting with "**0xBA1222222222"**.&#x20;

This chunk loads the Vault from its on-chain contract address, and its ABI makes it easy to make calls to it directly.

#### Swap Settings

```python
# Where are the tokens coming from/going to?
fund_settings = {
	"sender":				address,
	"recipient":			address,
	"fromInternalBalance": 	False,
	"toInternalBalance": 	False
}

# When should the transaction timeout?
deadline = 999999999999999999
```

Here, we're specifying that the sender/recipient for the tokens going into/out of the trade are both the account that we initialized the script with. Note that with this granularity, it is possible to make a swap that sends the tokens to a different address.&#x20;

We specify that {**to**/**from**}**InternalBalance** are both False. This will be the default use case for most users; however, you may have a use case that would benefit from Internal Balances.&#x20;

The deadline for a transaction is the time (in [Unix timestamp](https://www.unixtimestamp.com)) after which it will no longer attempt to make a trade. If a trade expires, it will still take some gas to process the failed transaction, but it will be cheaper than a transaction failing for a different reason.

#### Defining our pools and tokens

```python
# Pool IDs
pool_BAL_WETH = "0x61d5dc44849c9c87b0856a2a311536205c96c7fd000200000000000000000000"

# Token addresses
token_BAL 	= "0x41286Bb1D3E870f3F750eB7E1C25d7E48c8A1Ac7".lower()
token_WETH 	= "0xdFCeA9088c8A88A76FF74892C1457C17dfeef9C1".lower()

# Token data
token_data = {
	token_BAL:{
		"symbol":"BAL",
		"decimals":"18",
		"limit":"0"
	},
	token_WETH:{
		"symbol":"WETH",
		"decimals":"18",
		"limit":"1"
	}
}
```

Here, we list our Pool IDs, token contract addresses, and relevant token data. When entering token contract addresses, we enforce that they must be in lowercase to avoid issues with sorting later.

It is important to get your **decimals** set correctly for each token, otherwise you may send far more or far fewer tokens than you intended.

When setting your token limits, you must know that there are two types of swaps: GIVEN\_IN and GIVEN\_OUT. In a GIVEN\_IN swap, the **amount** you provide is the number of your input token you're giving the pool, while for a GIVEN\_OUT swap, you provide the number of tokens that you expect to receive. The limits are used to control the maximum/minimum amounts for the opposite side of the trade that you are not setting exact numbers for.&#x20;

| Swap Type  | **amount**                                | **TokenIn** Limit                                | **TokenOut** Limit                       |
| ---------- | ----------------------------------------- | ------------------------------------------------ | ---------------------------------------- |
| GIVEN\_IN  | Exact amount of TokenIn you will send     | Same as TokenIn amount                           | Minimum amount of TokenOut you'll accept |
| GIVEN\_OUT | Exact amount of TokenOut you will receive | Maximum amount of TokenIn you're willing to give | Same as TokenOut amount                  |

In the above example code, we set our **token\_BAL** limit to 0, which means we are willing to accept 100% slippage on our trade. That is generally a very bad idea, but this is an example on the Kovan testnet, so tokens are valueless here.

#### Swap definition

```python
swap = {
		"poolId":pool_BAL_WETH,
		"assetIn":token_WETH,
		"assetOut":token_BAL,
		"amount":"1"
	}

# SwapKind is an Enum. This example handles a GIVEN_IN swap.
# https://github.com/balancer-labs/balancer-v2-monorepo/blob/0328ed575c1b36fb0ad61ab8ce848083543070b9/pkg/vault/contracts/interfaces/IVault.sol#L497
swap_kind = 0 #0 = GIVEN_IN, 1 = GIVEN_OUT
```

Next, we define our **swap**. We must specify that this swap is created with a known amount **GIVEN\_IN**; we are _giving ****_ the pool 1 WETH for an estimated output of BAL. It is also possible to create a trade with a fixed amount **GIVEN\_OUT**.

#### Building our structs

```python
user_data_encoded = eth_abi.encode_abi(['uint256'], [0])

swap_struct = (
	swap["poolId"],
	swap_kind,
	Web3.toChecksumAddress(swap["assetIn"]),
	Web3.toChecksumAddress(swap["assetOut"]),
	int(Decimal(swap["amount"]) * 10 ** Decimal((token_data[swap["assetIn"]]["decimals"]))),
	user_data_encoded
)

fund_struct = (
	Web3.toChecksumAddress(fund_settings["sender"]),
	fund_settings["fromInternalBalance"],
	Web3.toChecksumAddress(fund_settings["recipient"]),
	fund_settings["toInternalBalance"]
)

```

When we call the Vault contract, we need to pack our data into structs, specifically the ones here referred to as **swap\_struct** and __ **fund\_struct**.&#x20;

**swap\_struct**s are of type _SingleSwap_, which is defined here:

```
struct SingleSwap {
   bytes32 poolId;
   SwapKind kind;
   IAsset assetIn;
   IAsset assetOut;
   uint256 amount;
   bytes userData;
}
```

The values that may cause confusion&#x20;

* **amount**: Either the amount of tokens we are sending **to** the pool or want to receive **from** the pool, depending on if the SwapKind is **GIVEN\_IN** or **GIVEN\_OUT**. As shown in the code, make sure that the amount is scaled according to the number of decimals for your token.
* **userData**: Any additional data which the pool requires to perform the swap. This allows pools to have more flexible swapping logic in future - for all current Balancer pools this can be left empty, as shown for **user\_data\_encoded.**

**fund\_struct**s are simpler FundManagements structs as defined here:

```
struct FundManagement {
    address sender;
    bool fromInternalBalance;
    address payable recipient;
    bool toInternalBalance;
}
```

The only real "gotcha" here is to make sure your **address**es are in checksum format.

#### Just one more formatting

```python
token_limit = int(Decimal(token_data[swap["assetIn"]]["limit"]) * 10 ** Decimal(token_data[swap["assetIn"]]["decimals"]))
```

Here, we're scaling our **token\_limit** to account for the token-specific decimals.

#### Building the function

```python
single_swap_function = contract_vault.functions.swap(	
	swap_struct,
	fund_struct,
	token_limit,
	deadline
)
```

Here, we're packing our properly formatted structs and other values into the **swap** function.&#x20;

#### Setting the remaining relevant parameters

```python
try:
	gas_estimate = single_swap_function.estimateGas()
except:
	gas_estimate = 100000
	print("Failed to estimate gas, attempting to send with", gas_estimate, "gas limit...")

data = single_swap_function.buildTransaction(
	{
		'chainId': chain_id,
	    'gas': gas_estimate,
	    'gasPrice': web3.toWei(gas_price, 'gwei'),
	    'nonce': web3.eth.get_transaction_count(address),
	}
)
```

Here, we attempt to estimate a gas price. In the event it fails, 100k gas is a safe estimate for the gas limit on a **swap**. The remaining lines set the **chain\_id**, **gas\_estimate**, **gas\_price**, and **nonce**.&#x20;

#### Sending and viewing the transaction

```python
signed_tx = web3.eth.account.sign_transaction(data, private_key)
tx_hash = web3.eth.send_raw_transaction(signed_tx.rawTransaction).hex()
print("Sending transaction...")
url = block_explorer_url + "tx/" + tx_hash
webbrowser.open_new_tab(url)
```

Finally, we sign the transaction with our **private\_key**, and broadcast the transaction to be added to the blockchain. As a convenience, the last two lines create a link to Etherscan and opens a tab in the user's default browser.
{% endtab %}
{% endtabs %}
