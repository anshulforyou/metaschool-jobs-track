# Description of the Attack
Hey Lads, today we are going to learn about one of the most basic, yet highly catastrophic attack in the smart contract world - The underflow/overflow attack.  
The main cause of this attack are some very basic errors, errors that have cost a lot of people, a lot of money.  

Overflow and underflow errors are something which we encounter in mathematics while dealing with integers  with a specific limit. 

That is why they are also called "integer overflow".  

In some programming languages you can think of a number line, like a number circle.
## Illustration add number circle here
So you start from 0 , 1 ,2 3, .... max value.  
As soon as you reach the max value, the next step you take will bring you back to 0.  
Why?  
Because it is a circle duh.  

# Overflow and underflow error attack
## What precisely are overflow and underflow errors?   

An overflow error occurs when a number gets incremented above its maximum defined value. Let's say we define a variable of type uint8 in solidity.   

Now we know that this variable can handle a maximum of eight bits, allowing the variable to reach the maximum value of 255 (2^8-1). Imagine if the value of this variable is at its highest value, which is 255, and we increment the value by one, then what will happen?
## Gif - falling ball and all digits getting changed
```
uint8 balance = 255;
balance++;
```
Ideally, the balance value here should be 256. Still, since the variable cannot store the value physically, it will reset to its initial value of 0.

So the final value of the balance would become 0 after the increment. 

This type of error can create havoc for the user, let's see how - 

Imagine you are a solidity developer writing a smart contract which implements a transfer function. In your transfer function, you update the balance of the receiver and sender by changing their account balance in the smart contract record as mentioned below.

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;

contract TokenFlow{
    mapping(address => uint8) balances;

    function Transfer(address sender, address receiver, uint8 amount) public {
        balances[sender] -= amount;
        balances[receiver] += amount;
    }
}
```
Now let's assume that the mapping "balances" contains a variable of type uint8 integers, so if the receiver's balance was 254. We add 2 in it, then instead of its final balance being 256, it becomes 0. Imagine seeing your wallet empty when you were expecting 256 coins or tokens in it😱  

### What if you had added 3 instead of 2?  
Then the balance should have become 257, but instead it will become 1.  
On adding 4 it would have become 2, and so on...  


Similarly, the underflow error is an exact mirror to the overflow error. Instead of exceeding the maximum value, the error occurs when the value goes below the minimum allowed number.   

Consider a variable of unsigned type integer which can contain a maximum og 8 bits (uint8). Now since it's an unsigned integer, it's value can't be negative, so if the current value of this variable is zero and we decrement it any further, the variable will end up attaining it's maximum value which is 255.  

Let's go back to above "writing our own smart contract" example to understand it better. If you have 2 apples and somehow your evil brain tells you to send 3 apples to your friend.

```
function Transfer(address sender, address receiver, uint256 amount) public {
        balances[sender] -= amount;
        balances[receiver] += amount;
    }
```
Your transfer function will execute the balances[sender] -= amount line and since you do not have enough balance, the variable should attain the value -1 but wait, it's an unsigned integer, it can't be negative. So magic will happen here and the variable will set itself to it's maximum value because the number in unsigned integer revolves in a circle. 0 comes after the maximum value(255 in uint8 case) and the max value comes below 0.
## gif - revolving numbers

## Real world scenarios
In the real world, underflow errors are more likely to happen than overflow errors because generally in solidity while defining a variable we keep the type to be uint256, which is a fairly big number.

But all the variables generally starts from zero so there are higher chances we try to send tokens from a nil balance account and hit the underflow error as seen in the underflow error example above. The results can be disastrous. 

Now if you are on the other side of this paradigm, you choose to become the evil doctor strange and try to hack a smart contract. Then you will look at places in a smart contract where you intentionally try to hit the underflow and overflow error to gain some extra tokens. Whenever some hackers use these errors to gain some advantage in the smart contract then it is called an attack.
<!-- These errors are called attacks when someone intentionally tries to hit these errors to affect the smart contract. -->


# Hack Analysis
Now let's look into at one very interesting hack involving these simple underflow, overflow errors.  

By now you might have heard about things like proof of work, proof of stake or proof history, but have you ever heard about the proof of weak hands?  

Proof of weak hands was a ponzi scheme that grew into over a million dollars. It was advertised as a legitimate pyramid scheme, and surprisingly its value quickly grew to over a million dollars, and over a thousand Ethereum. 

## What does this mean?
But a few hours after it's launch, 866 Ethereum vanished from the contract just because the developers were not able to secure their contract against the overflow, underflow errors.

So what exactly went wrong? How did a million dollar scheme suddenly turn to nothing?
Here's what went wrong.


<!-- Some extra explanation -->
<!-- In their implementation of the ERC20 token, they allowed a person to transfer tokens from one person's account to someone else's account on behalf of the user(the usual implementation of allowance function). However in this case, the sold coins from first account can be taken off from the second account's balance.

Let's look at the transfer function
```
function transferFrom(address _from, address _to, uint256 _value)  public {
    var _allowance = allowance[_from][msg.sender];
    if (_allowance < _value)
        revert();
    allowance[_from][msg.sender] = _allowance - _value;
    transferTokens(_from, _to, _value);
}
```
Here in the first four lines of the code there is no issue, it just makes sure the caller is authorized to spend someone else's coin but then it calls the transferTokens function to do the heavylifting.
Now let's look at the transferTokens function to see if it contains something suspicious.
```
function transferTokens(address _from, address _to, uint256 _value) internal {
    if (balanceOfOld[_from] < _value)
        revert();
    if (_to == address(this)) {
        sell(_value);
    } else {
        int256 payoutDiff = (int256) (earningsPerShare * _value);
        balanceOfOld[_from] -= _value;
        balanceOfOld[_to] += _value;
        payouts[_from] -= payoutDiff;
        payouts[_to] += payoutDiff;
    }
    Transfer(_from, _to, _value);
}
``` -->
In their implementation of the ERC20 token, they defined a sell function and they made a really trivial mistake in it.
```
function sell(uint256 amount) internal {
    var numEthers = getEtherForTokens(amount);
    // remove tokens
    totalSupply -= amount;
    balanceOfOld[msg.sender] -= amount;
    
    // fix payouts and put the ethers in payout
    var payoutDiff = (int256) (earningsPerShare * amount + (numEthers * PRECISION));
    payouts[msg.sender] -= payoutDiff;
    totalPayouts -= payoutDiff;
}
```

Look at the code very carefully because it is the place where hack took place. At first glance you might think that everything looks fine.  

But on taking a closer look we find that the function assumes that the msg.sender is the seller.  

Unfortunately, if sell is invoked from some other function like TransferFrom it’s possible that msg.sender's balance is now being drained of coins that it doesn’t own. Also here we can see that the code doesn't contain any check that ensures that the user owns the amount. Now think carefully about what we have learned above and use all your knowledge to guess what can happen here.

If someone having an empty account make a transfer and only one PoWHCoin is sent, the account's balance will underflow to 1.1579E77 or 2^256 — 1. It happened!!! the underflow error just took place and the user who had 0 tokens before got the maximum account balance.
https://ropsten.etherscan.io/address/0x2CB6ef99FbC78069364144E969a9A6e89E550359#code
This is the link to the etherscan of PonziToken contract


# Preventive Methods
By now you should have a pretty comprehensive idea about how and where this attack can be used, and around this time you must also be thinking  - how can we prevent these errors?  
Don't worry we've got you covered or should I say solidity has got you covered. 

## Require Statements
A simple but tedious way of dealing with these errors is that you put require statements whenever you are doing math in your smart contract, be it addition, subtraction, division or multiplcation.  
```
require(apple>=value, "Underflow error occured!");
```
or
```
uint8 c = apple + value;
require(c > apple, "Overflow error occured!");
```
These require statements make sure that the amount being subtracted or added doesn't make the variable go out of bounds.
In the case of underflow error prevention, we are making sure that the value being subtracted is less than or equal to apple so that it will never be in the case where it will try to attain a value less than zero and will thus hit the underflow error.

In the overflow error case, we ensure that the summation of the two variables is always greater than the value of those variables put alone.

## Openzeppelin's SafeMath Library
By now most of you must be aware of the power of openzeppelin's smart contracts. Openzeppelin provides a library called SafeMath which simplifies the work for us.

Here’s an example of how to use SafeMath’s sub() function instead of using the minus arithmetic operator in our smart contract:
```
using SafeMath for uint;
function withdraw(uint _value) external {
    require(balances[msg.sender].sub(_value) >= 0);
    balances[msg.sender] -= _value;
    msg.sender.transfer(_value);    
}
```

## Solidity version 0.8
But from solidity 0.8 onwards you don't even have to do this!
From Solidity 0.8 onwards, the check for overflow/underflow is implemented on the language level itself - it adds the validation to the bytecode during compilation.

So if you are sure that you would always be using solidity versions above 0.8 then you don't need to fret at all.
But if your code might be used for lower solidity versions as well, be safe while doing math - use open zepellin's safe math.


# Task 
Let's get our hands dirty with solidity code, and get started with our first task.
Here we will deploy our own smart contract and see the underflow error take place in real life.
```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;

contract UnderFlowInteresting{
    uint8 apple =0;

    function doUnderFlow(uint8 value) public returns (uint8) {
        apple = apple-value;
        return apple;
    }
}
```
Copy paste the above code on remix IDE and deploy it. Please take a note that we are using solidity version 7.0 here, this will be important later.
Also notice that this very simple smart contract has a uint state variable called apple, this variable has a current value of 0. And as we already know, a uint (or unsigned int ) cannot handle values below 0.
So let's see what happens when we underflow it.

## Add image of remix IDE Deploy here, along with code
After you have deployed the code, let's call the doUnderFlow function and pass the value 3 as an argument while calling it. The doUnderFlow function returns the value of global variable "apple" after it substracts the argument value from it. So here after passing 3 we will see that the value returned is 253. 
Since apple was 0 in the starting, subtracting 3 from it, first underflowed the variable, and thus it reached it's maximum value 255., and then 2 got subtracted from it which made it's value 253.

Now let's remove this error by putting in some require statements.

```
require(apple>=value, "Oops, underflow error occured!");
```
We will add this line in the starting of our doUnderFlow function and then redeploy the contract. This time when we will try to call the function, our transaction will not go through and we will see an error which says "Oops, underflow error occured!"
Hoorah we secured our contract from hitting the underflow error but what exactly happened? Let's go through our require statement.
We just put a simple statement which checks if the value that we are planning to substract from our variable apple is less than or equal to it's value or not.
If the value you are subtracting from a number, is less than the number itself, then the variable will never underflow.
But how is this different from the require statement which we generally see in all the contracts.
```
require(apple-value>0, "UnderFlow error occured!");
```
Well I would say "you tell me", if we are already subtracting the value from our variable and then checking if it's value is less than zero or not then how will this save our contract from the error?
The moment we wrote apple-value>0, we went wrong because the deed is already done and now putting the check doesn't make any sense.

Now the require statement did it's work but let's look at something more elegant, something that does all this work for us . 
You know what I am talking about - Openzeppelin's safeMath library!
Let's implement it in our code.

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v3.3.0/contracts/math/SafeMath.sol";

contract UnderFlowInteresting{
    using SafeMath for uint8;
    uint8 apple =0;

    function doUnderFlow(uint8 value) public returns (uint8) {
        apple.sub(value);
        return apple;
    }
}
```
Here when we will call the doUnderFlow function then our transaction will not pass through and it will say "SafeMath: subtraction overflow." Isn't it easy? SafeMath did all the heavylifting for us.

Now let's do a bit of magic😉
```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

contract UnderFlowInteresting{
    uint8 apple =0;

    function doUnderFlow(uint8 value) public returns (uint8) {
        apple=apple-value;
        return apple;
    }
}
```
Just run this contract in remix and call the doUnderFlow function with argument 3, it should hit the underflow error and return value 253, right?   

But the transaction didn't pass through, what happened?  
We didn't put any require statement, I also don't see the safe math library being implemented.  
So what exactly happened? Now sit back and observe the code very carefully and compare with the initial code you deployed on remix while starting this task. The solidity compiler version is different. 

As mentioned in the preventive techniques section above, from solidity version 8 the compiler itself looks for overflow, underflow errors.

And with that we are done with overflow/underflow attacks!!!
Be careful about these trivial but deadly attacks when writing solidity code :)

### Bonus Task
Here is a small bonus task for all the over achievers out there, choose your favourite programming language, and see how overflow/underflow errors are handled in that language!
Drop whatever you have learnt in the box below.


## Suggested GIFs
1. Ball with array 
2. Buckets of water
3. Number circle and we show infinity + 1 = - infinity (0)