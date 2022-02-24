# Swaps

Swaps are a cornerstone of any exchange, and Balancer has a few types for different purposes.

{% hint style="info" %}
This section will dive into code for executing swaps on Balancer V2. This functionality is also available in the Python Library **balpy**. Find it on [PyPI](https://pypi.org/project/balpy/) and the source (with samples!) on [GitHub](https://github.com/gerrrg/balpy/).
{% endhint %}

### Batch Swaps

You'll want to use **Batch Swaps** when you're making a trade that hops through multiple pools. These are useful for swapping between two tokens that aren't in the same pool, and for routes with better prices than naive single swaps.

#### Flash Swaps

There is a specific case of Batch Swap called a Flash Swap that enables trades with no input tokens. These are useful for doing arbitrage among Balancer pools. To make a Flash Swap, create a Batch Swap with all your token limits set to zero.

### Single Swaps

You'll want to use **Single Swaps** when you're making a trade between two tokens in one pool. While it's possible to do this with a one-step Batch Swap, using a **Single Swap** will save \~6,000 gas.

## What kind of swap should I be using?

In most cases, you'll want to use **Batch Swaps**. The only time you would want to use a Single Swap is when you're making a trade between just two tokens in one pool. **Executing multiple Single Swaps in a single transaction is inefficient and should instead be batched**.&#x20;

![Sample gas costs for trades executed through multiple weighted pools](<../../.gitbook/assets/Screen Shot 2021-10-05 at 8.16.12 AM (1).png>)

![Sample gas costs for trades executed through Weighted->Stable->Weighted pools](<../../.gitbook/assets/Screen Shot 2021-10-05 at 8.16.07 AM (1).png>)
