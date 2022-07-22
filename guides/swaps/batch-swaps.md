# Batch Swaps

## Why should I use a batchSwap?

**Tokens that aren't in the same pool**

Let's say we want to trade TokenA for TokenC, but we only have pools with \[TokenA, TokenB] and \[TokenB, TokenC]. We can swap A -> B and B -> C.

**Routes with better prices than Single Swaps**

Let's say we still want to trade TokenA for TokenC, but now there's a \[TokenA, TokenC] pool. We could use a Single Swap there, but there might be a better price routing by swapping A -> B and B -> C.

For additional information, check out the [Batch Swaps page in Resources](../../resources/swaps/batch-swaps.md).

## Code Walkthrough

Select your desired programming language in the tabs below for the relevant tutorial.

{% tabs %}
{% tab title="js" %}
{% hint style="info" %}
This script assumes that the address you're using has already granted token spend allowances to the Vault. If you do not do that first, **your transaction will fail**.
{% endhint %}

Let's step through a [JavaScript Batch Swap example](https://github.com/gerrrg/balancer-tutorials/blob/master/js/swaps/batch\_swap.js) chunk by chunk.

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

This section connects to the Ethereum (or other EVM compatible) blockchain using the **RPC\_ENDPOINT** environment variable provided either in the shell, or in the [bash script helper](https://github.com/gerrrg/balancer-tutorials/blob/master/js/swaps/sample\_batch\_swap.sh). Similarly, it initialized an Ethereum account based on the private key provided with **KEY\_PRIVATE**.&#x20;

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
const pool_WETH_USDC =  "0x3a19030ed746bd1c3f2b0f996ff9479af04c5f0a000200000000000000000004";
const pool_BAL_WETH =   "0x61d5dc44849c9c87b0856a2a311536205c96c7fd000200000000000000000000";

// Token addresses (checksum format)
const token_BAL = "0x41286Bb1D3E870f3F750eB7E1C25d7E48c8A1Ac7".toLowerCase();
const token_USDC  = "0xc2569dd7d0fd715B054fBf16E75B001E5c0C1115".toLowerCase();
const token_WETH = "0xdFCeA9088c8A88A76FF74892C1457C17dfeef9C1".toLowerCase();

// Token data
const token_data = {};
token_data[token_BAL] = 
    {
        "symbol": "BAL",
        "decimals": "18",
        "limit": "0"
    };
token_data[token_USDC] =
    {
        "symbol":"USDC",
        "decimals":"6",
        "limit":"100"
    };
token_data[token_WETH] = 
    {
        "symbol": "WETH",
        "decimals": "18",
        "limit": "0"
    };
```

Here, we list our Pool IDs, token contract addresses, and relevant token data. When entering token contract addresses, we enforce that they must be in lowercase to avoid issues with sorting later.

It is important to get your **decimals** set correctly for each token, otherwise you may send far more or far fewer tokens than you intended.

To protect users from front-running or the market changing rapidly, they supply a list of **limit**s for each token involved in the swap, where either the maximum number of tokens to send (by passing a positive value) or the minimum amount of tokens to receive (by passing a negative value) is specified. See that in this example, we are willing to send at most 100 USDC, and receive as few as 0 BAL, WETH. Setting your receive limits to 0 is generally a very bad idea (it means we are willing to accept 100% slippage on our trade), but this is an example on the Kovan testnet, so tokens are valueless here.

#### Swap Steps

```javascript
const swap_steps = [
    {
        "poolId": pool_WETH_USDC,
        "assetIn": token_USDC,
        "assetOut": token_WETH,
        "amount": 100
    },
    {
        "poolId": pool_BAL_WETH,
        "assetIn": token_WETH,
        "assetOut": token_BAL,
        "amount": 0
    }
];

// SwapKind is an Enum. This example handles a GIVEN_IN swap.
// https://github.com/balancer-labs/balancer-v2-monorepo/blob/0328ed575c1b36fb0ad61ab8ce848083543070b9/pkg/vault/contracts/interfaces/IVault.sol#L497
// 0 = GIVEN_IN, 1 = GIVEN_OUT
const swap_kind = 0;
```

Next, we define our **swap\_steps**. Each step in this list is its own swap with a pool. The first step here is clearly a swap of **100 USDC** for **WETH** in **pool\_WETH\_USDC** (amounts will be scaled for decimals later). The second step is less obvious. You may notice that the **amount** in the second swap is **0**. Here, the **0** value is used to say "take the output of the previous swap and use it as my input." The reason for this is that the expected output of the trade could change slightly between the times that the trade is requested and when it is actually executed.&#x20;

We also must specify that this swap is created with a known amount **GIVEN\_IN**; we are _giving ****_ the pool 100 USDC for an estimated output. It is also possible to create a trade with a fixed amount **GIVEN\_OUT**.

#### Token ordering

```javascript
var token_addresses = Object.keys(token_data);
token_addresses.sort();
const token_indices = {};
for (var i = 0; i < token_addresses.length; i++) {
    token_indices[token_addresses[i]] = i;
}
```

Token ordering is very important in the Balancer Vault; each pool stores its tokens sorted numerically. Because of this, we will need to sort our own token lists when interacting with pools. When calling the contract itself, we must refer to the tokens **by their index** in this sorted list. The **token\_indicies** loop creates a dictionary that gives us each token's index in a sorted list to make the bookkeeping easier.

#### Building our structs

```javascript
const swap_steps_struct = [];
for (const step of swap_steps) {
    const swap_step_struct = {
        poolId: step["poolId"],
        assetInIndex: token_indices[step["assetIn"]],
        assetOutIndex: token_indices[step["assetOut"]],
        amount: BigNumber(step["amount"] * Math.pow(10, token_data[step["assetIn"]]["decimals"])).toString(),
        userData: '0x'
    };
    swap_steps_struct.push(swap_step_struct);
}

const fund_struct = {
    sender: web3.utils.toChecksumAddress(fund_settings["sender"]),
    fromInternalBalance: fund_settings["fromInternalBalance"],
    recipient: web3.utils.toChecksumAddress(fund_settings["recipient"]),
    toInternalBalance: fund_settings["toInternalBalance"]
};
```

When we call the Vault contract, we need to pack our data into structs, specifically the ones here referred to as **swap\_steps\_struct** and __ **fund\_struct**s.&#x20;

**swap\_steps\_struct** are of type _BatchSwapStep_, which is defined here:

```
struct BatchSwapStep {
    bytes32 poolId;
    uint256 assetInIndex;
    uint256 assetOutIndex;
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

#### Just a couple more formattings

```javascript
const token_limits = [];
const checksum_tokens = [];
for (const token of token_addresses) {
    token_limits.push(BigNumber((token_data[token]["limit"]) * Math.pow(10, token_data[token]["decimals"])).toString());
    checksum_tokens.push(web3.utils.toChecksumAddress(token));
}
```

Here, we're scaling our **token\_limits** to account for the token-specific decimals, and converting all our token addresses to checksum format instead of lowercase.

#### Building the function

```javascript
const batch_swap_function = contract_vault.methods.batchSwap(
    swap_kind,
    swap_steps_struct,
    checksum_tokens,
    fund_struct,
    token_limits,
    deadline.toString()
);
```

Here, we're packing our properly formatted structs and other values into the **batchSwap** function.&#x20;

#### Setting the remaining relevant parameters in an async method

```javascript
async function buildAndSend() {
    var gas_estimate;
    try {
        gas_estimate = await batch_swap_function.estimateGas();
    }
    catch(err) {
        gas_estimate = 200000;
        console.log("Failed to estimate gas, attempting to send with", gas_estimate, "gas limit...");
    }

    const tx_object = {
        'chainId':  chain_id,
        'gas':      web3.utils.toHex(gas_estimate),
        'gasPrice': web3.utils.toHex(web3.utils.toWei(gas_price,'gwei')),
        'nonce':    await web3.eth.getTransactionCount(address),
        'data':     batch_swap_function.encodeABI(),
        'to':       address_vault
    };
```

Here, we attempt to estimate a gas price. In the event it fails, 200k gas is a safe estimate for the gas limit on a two-swap **batchSwap**. The remaining lines set the **chainId**, **gas**, **gasPrice**, **nonce, data,** and **to** address.&#x20;

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

Let's step through a [Python Batch Swap example](https://github.com/gerrrg/balancer-tutorials/blob/master/python/swaps/batch\_swap.py) chunk by chunk.

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

This section connects to the Ethereum (or other EVM compatible) blockchain using the **RPC\_ENDPOINT** environment variable provided either in the shell, or in the [bash script helper](https://github.com/gerrrg/balancer-tutorials/blob/master/python/swaps/sample\_batch\_swap.sh). Similarly, it initialized an Ethereum account based on the private key provided with **KEY\_PRIVATE**.&#x20;

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
pool_WETH_USDC = 	"0x3a19030ed746bd1c3f2b0f996ff9479af04c5f0a000200000000000000000004"
pool_BAL_WETH = 	"0x61d5dc44849c9c87b0856a2a311536205c96c7fd000200000000000000000000"

# Token addresses
token_BAL 	= "0x41286Bb1D3E870f3F750eB7E1C25d7E48c8A1Ac7".lower()
token_USDC 	= "0xc2569dd7d0fd715B054fBf16E75B001E5c0C1115".lower()
token_WETH 	= "0xdFCeA9088c8A88A76FF74892C1457C17dfeef9C1".lower()

# Token data
token_data = {
	token_BAL:{
		"symbol":"BAL",
		"decimals":"18",
		"limit":"0"
	},
	token_USDC:{
		"symbol":"USDC",
		"decimals":"6",
		"limit":"100"
	},
	token_WETH:{
		"symbol":"WETH",
		"decimals":"18",
		"limit":"0"
	}
}
```

Here, we list our Pool IDs, token contract addresses, and relevant token data. When entering token contract addresses, we enforce that they must be in lowercase to avoid issues with sorting later.

It is important to get your **decimals** set correctly for each token, otherwise you may send far more or far fewer tokens than you intended.

To protect users from front-running or the market changing rapidly, they supply a list of **limit**s for each token involved in the swap, where either the maximum number of tokens to send (by passing a positive value) or the minimum amount of tokens to receive (by passing a negative value) is specified. See that in this example, we are willing to send at most 100 USDC, and receive as few as 0 BAL, WETH. Setting your receive limits to 0 is generally a very bad idea (it means we are willing to accept 100% slippage on our trade), but this is an example on the Kovan testnet, so tokens are valueless here.

#### Swap Steps

```python
swap_steps = [
	{
		"poolId":pool_WETH_USDC,
		"assetIn":token_USDC,
		"assetOut":token_WETH,
		"amount": "100"
	},
	{
		"poolId":pool_BAL_WETH,
		"assetIn":token_WETH,
		"assetOut":token_BAL,
		"amount":"0"
	}
]

# SwapKind is an Enum. This example handles a GIVEN_IN swap.
# https://github.com/balancer-labs/balancer-v2-monorepo/blob/0328ed575c1b36fb0ad61ab8ce848083543070b9/pkg/vault/contracts/interfaces/IVault.sol#L497
swap_kind = 0 #0 = GIVEN_IN, 1 = GIVEN_OUT
```

Next, we define our **swap\_steps**. Each step in this list is its own swap with a pool. The first step here is clearly a swap of **100 USDC** for **WETH** in **pool\_WETH\_USDC** (amounts will be scaled for decimals later). The second step is less obvious. You may notice that the **amount** in the second swap is **0**. Here, the **0** value is used to say "take the output of the previous swap and use it as my input." The reason for this is that the expected output of the trade could change slightly between the times that the trade is requested and when it is actually executed.&#x20;

We also must specify that this swap is created with a known amount **GIVEN\_IN**; we are _giving ****_ the pool 100 USDC for an estimated output. It is also possible to create a trade with a fixed amount **GIVEN\_OUT**.

#### Token ordering

```python
token_addresses = list(token_data.keys())
token_addresses.sort()
token_indices = {token_addresses[idx]:idx for idx in range(len(token_addresses))}
```

Token ordering is very important in the Balancer Vault; each pool stores its tokens sorted numerically. Because of this, we will need to sort our own token lists when interacting with pools. When calling the contract itself, we must refer to the tokens **by their index** in this sorted list. The **token\_indicies** line creates a dictionary that gives us each token's index in a sorted list to make the bookkeeping easier.

#### Building our structs

```python
user_data_encoded = eth_abi.encode_abi(['uint256'], [0])
swaps_step_structs = []
for step in swap_steps:
	swaps_step_struct = (
		step["poolId"],
		token_indices[step["assetIn"]],
		token_indices[step["assetOut"]],
		int(Decimal(step["amount"]) * 10 ** Decimal((token_data[step["assetIn"]]["decimals"]))),
		user_data_encoded
	)
	swaps_step_structs.append(swaps_step_struct)

fund_struct = (
	Web3.toChecksumAddress(fund_settings["sender"]),
	fund_settings["fromInternalBalance"],
	Web3.toChecksumAddress(fund_settings["recipient"]),
	fund_settings["toInternalBalance"]
)
```

When we call the Vault contract, we need to pack our data into structs, specifically the ones here referred to as **swap\_step\_structs** and __ **fund\_struct**s.&#x20;

**swap\_step\_structs** are of type _BatchSwapStep_, which is defined here:

```
struct BatchSwapStep {
    bytes32 poolId;
    uint256 assetInIndex;
    uint256 assetOutIndex;
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

#### Just a couple more formattings

```python
token_limits = [int(Decimal(token_data[token]["limit"]) * 10 ** Decimal(token_data[token]["decimals"])) for token in token_addresses]
checksum_tokens = [Web3.toChecksumAddress(token) for token in token_addresses]
```

Here, we're scaling our **token\_limits** to account for the token-specific decimals, and converting all our token addresses to checksum format instead of lowercase.

#### Building the function

```python
batch_swap_function = contract_vault.functions.batchSwap(	
	swap_kind,
	swaps_step_structs,
	checksum_tokens,
	fund_struct,
	token_limits,
	deadline
)
```

Here, we're packing our properly formatted structs and other values into the **batchSwap** function.&#x20;

#### Setting the remaining relevant parameters

```python
try:
	gas_estimate = batch_swap_function.estimateGas()
except:
	gas_estimate = 200000
	print("Failed to estimate gas, attempting to send with", gas_estimate, "gas limit...")

data = batch_swap_function.buildTransaction(
	{
		'chainId': chain_id,
	    'gas': gas_estimate,
	    'gasPrice': web3.toWei(gas_price, 'gwei'),
	    'nonce': web3.eth.get_transaction_count(address),
	}
)
```

Here, we attempt to estimate a gas price. In the event it fails, 200k gas is a safe estimate for the gas limit on a two-swap **batchSwap**. The remaining lines set the **chain\_id**, **gas\_estimate**, **gas\_price**, and **nonce**.&#x20;

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
