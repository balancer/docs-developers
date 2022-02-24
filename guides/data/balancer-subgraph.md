# Balancer Subgraph

## Why should I use the Balancer Subgraph?

The Subgraph features easy-to-query data using GraphQL, and can log data in such a way that you can easily access data that's difficult to query on-chain. For example, there is no on-chain list of all Balancer pools (similar to how there's no on-chain list of all ERC20 tokens), but on the Subgraph, you can easily query all pools, even filtering by PoolType.&#x20;

Explaining what TheGraph is and how it works is outside the scope of this article, but if you'd like to know more, please refer to [their documentation](https://thegraph.com/docs/about/introduction).

## Code Walkthrough

Select your desired programming language in the tabs below for the relevant tutorial.

{% tabs %}
{% tab title="Python" %}
Let's step through [an example](https://github.com/gerrrg/balancer-tutorials/blob/master/python/data/subgraph.py) chunk by chunk of getting Balancer Pools and their tokens from the Subgraph.

#### Dependencies

```python
import json

# thegraph queries
from gql import gql, Client
from gql.transport.requests import RequestsHTTPTransport
```

This sample relies on TheGraph's [gql library](https://github.com/graphql-python/gql) to query TheGraph and on a few other libraries. These dependencies can be found in the [requirements.txt](https://github.com/gerrrg/balancer-tutorials/blob/master/python/requirements.txt) file in the Python sample repository.&#x20;

#### Initialize TheGraph Connection

```python
# Initialize subgraph
subgraph_url = "https://api.thegraph.com/subgraphs/name/balancer-labs/balancer-v2"
balancer_transport=RequestsHTTPTransport(
    url=subgraph_url,
    verify=True,
    retries=3
)
client = Client(transport=balancer_transport)
```

Here, we are creating the client for the Subgraph, pointing it to the Balancer Ethereum mainnet Subgraph. There are Subgraphs for each network on which Balancer runs. We will use this client to make all of our calls.

#### Craft a Query and Request Data (Getting Pools)

```python
query_string = '''
query {{
  pools(first: {first}, skip: {skip}) {{
    id
    address
    poolType
    strategyType
    swapFee
    amp
  }}
}}
'''
num_pools_to_query = 100
formatted_query_string = query_string.format(first=num_pools_to_query, skip=0)
response = client.execute(gql(formatted_query_string))
```

Here, we create a **query\_string** that is written in GraphQL. We request pools with a variety of attributes for each one, specifying the maximum number of pools we want in _this_ request. The **first**/**skip** notation allows us to query pools in batches. We then get a response from the client, formatted as a Python _dict_.&#x20;

#### Request Tokens for each Pool

```python
for pool in response["pools"]:
	pool_token_query = '''
	query {{
	  poolTokens(first: 8, where: {{ poolId: "{pool_id}" }}) {{
	    id
		symbol
		name
		decimals
		address
		balance
		invested
		investments
		weight
	  }}
	}}
	'''
	formatted_query_string = pool_token_query.format(pool_id=pool["id"])
	token_response = client.execute(gql(formatted_query_string))
	pool["poolTokens"] = token_response["poolTokens"]

print(json.dumps(response["pools"], indent=4))
```

Using the list of pools from the previous call, we can now request **poolTokens** data for each pool. The **where** specifier in the query header filters for the specific pool that we're interested in. We can now add the **poolTokens** response into our pool _dict_ to keep all our data in the same structure.&#x20;

Finally, we print out the data formatted nicely as a json. See below for some sample output data:

```python
...
{
        "address": "0x0b09dea16768f0799065c475be02919503cb2a35",
        "amp": null,
        "id": "0x0b09dea16768f0799065c475be02919503cb2a3500020000000000000000001a",
        "poolType": "Weighted",
        "strategyType": 2,
        "swapFee": "0.0027",
        "poolTokens": [
            {
                "address": "0x6b175474e89094c44da98b954eedeac495271d0f",
                "balance": "41417760.60370376773502331",
                "decimals": 18,
                "id": "0x0b09dea16768f0799065c475be02919503cb2a3500020000000000000000001a-0x6b175474e89094c44da98b954eedeac495271d0f",
                "invested": "0",
                "investments": [],
                "name": "Dai Stablecoin",
                "symbol": "DAI",
                "weight": "0.4"
            },
            {
                "address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "balance": "19467.723570242016061162",
                "decimals": 18,
                "id": "0x0b09dea16768f0799065c475be02919503cb2a3500020000000000000000001a-0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
                "invested": "0",
                "investments": [],
                "name": "Wrapped Ether",
                "symbol": "WETH",
                "weight": "0.6"
            }
        ]
    },
...
```
{% endtab %}
{% endtabs %}
