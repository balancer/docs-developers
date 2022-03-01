# StablePool

## Common Arguments

In addition to the arguments listed before, you should also consider the [common arguments](./#common-arguments) when creating a pool.&#x20;

## Pool Creation Arguments

### Amplification Parameter

StableMath relies on an amplification parameter. The amplification parameter determines how "flat" the invariant curve is, or in other words, how strongly correlated the tokens in the pool are.&#x20;

If there is a StablePool with two very reputable stablecoins, one would expect it to have a high amplification parameter. On the other hand, if there's a StablePool with one reputable stablecoin, and one new, unproven stablecoin, it may have a low amplification parameter to account for token volatility.&#x20;

For more information on the amplification parameter, read about [StableMath in the main docs](https://docs.balancer.fi/concepts/math/stable-math).&#x20;
