# Timelock Authorizer

## Credit

Credit for the below article goes to [0xSkly](https://twitter.com/0xSkly) of the BeethovenX Team. You can find the original article on [Medium](https://medium.com/@0xSkly/inside-balancer-code-timelockauthorizer-5f5a0f7bec12).&#x20;

Disclaimer: The article has been lightly edited for formatting and uniformity with the rest of the docs, but has been kept in a first-person format. Any opinions expressed below are those of the author and should not be considered opinions of Balancer Labs, BeethovenX, BalancerDAO, or any other organization.

## Inside The Contract <a href="#534e" id="534e"></a>

Fine grained authorization mechanisms are key when protecting a complex protocol like Balancer. Another key aspect is execution transparency combined with a delay so users can react on changes to the protocol before they go live.

Currently, Balancer achieves this thorough a classic `Timelock` contract which handles execution delay combined with the `Authorizer` contract which handles authorization. They’ve now combined this into one contract with the fitting name `TimelockAuthorizer`. A main problem I have with `Timelock` contracts and other proxy contracts is that it obfuscates intent, making it harder for an average user to decode what is executed because of the hashed proxy call. Have you ever tried to follow a timelock transactions executed by a Gnosis Multisig proxy contract? Good luck.

After this rather long intro, let’s do a deep dive into how this new contract works and if it makes execution intent easier to track.

### How are Authentication and Authorization Applied? <a href="#eec2" id="eec2"></a>

Just as a quick recap, authentication is figuring out **who** you are, authorization is figuring out if you are **allowed to perform** this action. At the basis of this lies the `Authentication` contract. It provides the `authenticate` modifier.

![](https://miro.medium.com/max/1400/1\*zR5qW7RNlcMUK83lyC-MmA.png)

We see the modifier calls the `_authenticateCaller` function which resolves an `actionId` and delegates a call to the virtual function `_canPerform` with the `sender` and the `actionId` which reverts when returned `false`. Just from looking at this snippet, we can assume that the `actionId`, which is based on the `msg.sig`, resembles the function to execute whereas `_canPerform` is expected to check if the caller is allowed to do this action.

So let’s have a closer look at this `actionId` which seems to be a key concept. Why is it not just `msg.sig` which is the function signature you may ask.

![](https://miro.medium.com/max/1400/1\*9N4AdJ0SeTmJ3K66qkKOfQ.png)

We see it creates a hash together with `_actionIdDisambiguator` . The comments do a good job describing this property.

![](https://miro.medium.com/max/1400/1\*xpwaMkCNWbCQ7l27BsA5kA.png)

The nice thing about this is that it allows to share permissions, so that for example all contracts deployed from the same factory share the same permissions. Otherwise one would need to add permissions for each pool deployed from a factory which would be rather cumbersome.

So now that we’ve got this out of the way, let’s move on to the `_canPerform` function, which apparently is responsible for authorization. For this we look at the `BasePoolAuthorization` contract which serves as a base for all pools.

![](https://miro.medium.com/max/1400/1\*8c3VvzZykE\_UhzmlGcLKbw.png)

So if it’s an owner only action, only the owner can do this action (what surprise), otherwise we again delegate the call to our final destination, the `Authorizer,` finally coming back to the actual thing I wanted to talk about. What a detour, but it was kinda necessary to understand the full context.

### Inside the `TimelockAuthorizer` <a href="#bbc6" id="bbc6"></a>

From what we have seen, we expect this contract to handle function execution permissions based on the `actionId` + `msg.sender` + `targetContractAddress` and also support some form of timelocked execution. We stopped at `_getAuthorizer().canPerform(..)` so let’s continue there

![](https://miro.medium.com/max/1400/1\*OX4OxQ7EMpfWEQZfB1YHrA.png)

We see that `msg.sender` is referenced as `account` and the `targetContractAddress` is the `where` parameter. We check if this `actionId` has a delay configured (which means this action is timelocked) by checking `_delaysPerActionId[actionId] > 0`. If there is no delay configured, we just check if permissions are given.

![](https://miro.medium.com/max/1400/1\*RzhLWUQTIywHdHLHnYqQ5A.png)

where `_isPermissionGranted` is a mapping between the hash over `actionId, account, where` to a boolean.

If it has a delay, then only the `_executor` is allowed to call this function ( `account == address(_executor)`) which means the contract behind this `_executor` address is somehow executing a timelocked action. So let’s see whats up with this contract which is assigned in the constructor

```
_executor = new TimelockExecutor();
```

The contract only has 1 function `execute` which basically proxies a function call

![](https://miro.medium.com/max/1400/1\*i\_OPXh5d\_ivQYbBLPH4qOA.png)

The function can only be called by `address(authorizer)` which is assigned in the constructor

```
constructor() {    authorizer = TimelockAuthorizer(msg.sender);}
```

where `msg.sender` is the `TimelockAuthorizer` which deploys the contract in its own constructor call. So only the `TimelockAuthorizer` contract can call the `execute` function. Therefore we can already assume the execution flow for a timelocked action to be something like `TimelockAuthorizer.execute(...) => TimelockExecutor.execute(...) => targetContract.performAction()`

Now that we know what the contract behind the`_executor` is, let’s go back and figure out how this all comes together. For this, we jump into the `schedule` function which is the start of a function call protected with a delay (timelocked).

![](https://miro.medium.com/max/1400/1\*Z0TJLveOoNRsLuTwDB5Mkg.png)

Seems pretty straight forward, we decode the action the caller wants to perform, verify he or she is permitted to do so and schedule it. Note that there is an additional `executors` argument passed along which is an array of addresses. We’ll see in a bit what that is about. Following the execution chain, it pulls the configured delay for this action and we end up in `_scheduleWithDelay`.

![](https://miro.medium.com/max/1400/1\*HKgyw5xIqEFKxYuD5A0utA.png)

We define the `scheduledExecutionId` which increments on the `_scheduledExecutions` arrays length. We calculate the execution time and push the whole configuration into the array. If you pass no `executors` then it sets this `protected` flag, which I don’t know yet what it does, but I'm sure we’ll find out. Finally we see what the `executors` address array is for. All those addresses are granted the permission to execute this specific scheduled action without the need to give them permanent permissions to execute all actions with this `actionId`. This adds a nice additional layer of control.

Now we made it to the final stretch, the execution of the scheduled action.

![](https://miro.medium.com/max/1400/1\*8sCH6MVkfzMmYKNdMpOSFA.png)

It is triggered via its `scheduledExecutionId`, making sure enough time has passed via `block.timestamp > scheduledExecution.executableAt` and executes it via `_executor.execute(...)`. We also see now what happens when the `protected` flag is set. It checks the permission on the `msg.sender` , so if you pass an empty array for `executors` when scheduling an action, only addresses which have explicit permission for this `actionId` can execute it.

There we are, we have seen the whole execution flow of actions which can be executed immediately and delayed (timelocked) actions. What we have not seen yet is how we grant & revoke permissions and configure delays for specific actions. But that’s for another time.

### Conclusion <a href="#33cf" id="33cf"></a>

Merging the `Timelock` & `Authorizer` contract enables for an even more fine grained control over Balancer permissions and removes one level of indirection. Unfortunately, it still requires some sort of script to decode the scheduled actions where it would be nice for people if they could just inspect them from Etherscan. But to make this possible would come with its own drawbacks.
