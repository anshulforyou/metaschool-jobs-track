Hey Lads, today we will learn about delegation, a skill necessary in today's world. Still, it can also cause tremendous damage if done wrong.

We will learn about delegate calls and how they make things easier for solidity developers. When implemented wrong, they can cause changes that can hack the smart contract in every way possible. 

# Description of the Attack
## What is a Delegate call?
While programming our applications, we make many calls to the different functions. A delegate call can also be thought of as a call to the function but that particular function exists in some other contract.

So basically, we call another function of a different contract from one function. In the delegate call, the code executed at the target address is run in the context of the calling contract. Thus, the msg.sender and msg.value remains changed.

Let me also talk about the syntax of delegate calls and how it is used since it is not something trivial.
```
<address of contract1>.delegatecall(abi.encodeWithSignature("functionName(type1, type2)", variableValue1, variableValue2));
```
Here contract1 is the contract to which delegate call is made, functionName is the name of the function that needs to be executed. As seen in the syntax above, complete function description needs to be provided which means that we need to mention the type of variables that exist in the function. VariableValue is the value that we need to pass to the function.

Let's take an example to understand delegate calls and how they work.
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract LogicContract {
    address returnedAddress;
    event contractAddress(address returnedAddress );
    
    function print_address() public  returns(address){
         returnedAddress = address(this); 
         emit contractAddress(returnedAddress);
    }
}
```
LogicContract is a smart contract that is doing the heavy lifting. We will call this smart contract from other contract using it's contract address.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract CallingContract { 
    address returnedAddress; 
    address logic_pointer = address(new LogicContract());
  
    function print_my_delegate_address() public returns(address){
       logic_pointer.delegatecall(abi.encodeWithSignature("print_address()"));
    }
}
```
Here in the CallingContract we can see the delegate call being made to the address of LogicContract. Through the delegate call we are executing the print_address function of the contract CallingContract. Feel free to copy paste the code on remix and play with it. We will learn more about it in our task section below.

The point that needs to be noted here is that the storage layout of both the contracts is same. We will see why this is important and how it is helping the delegate call to work properly in a while.

Now if you are thinking more about the delegate calls, you see that it has a lot of use cases. Libraries exist in the smart contract because of delegate calls. Developers can easily reuse their code deployed on a smart contract.

You must be thinking now how interesting delegate calls are and how they are making work easier for many of us, but let me tell you that these same calls can lead to unexpected code execution.

As I mentioned above, delegate calls are run in the context of caller contract. This unique functionality of delegate calls introduces some extra vulnerabilities due to which it's become challenging to implement them securely, and I will tell you why.

## Vulnerability
Before knowing the vulnerabilities of delegate calls, we need to understand how solidity stores variables. 
### How solidity stores variables?
The variables of a contract in solidity are stored in slots. So the first variable of a contract will be stored in slot[0] while the second one will be stored in slot[1]. Now every contract stores it's variable in the same way. We know that delegate calls maintains the context of the calling contract.

Now since you know how variables are stored, let's look at an example to understand how vulneralibility occurs in delegate calls.
```
pragma solidity 0.6.6;

contract LogicContract {
    uint public a;
 
    function set(uint256 val) public {
      a = val;
    }
}

contract CallingContract {
   uint256 public b = 5; 
   uint256 public a = 5;
   address logic_pointer = address(new LogicContract());

   function setA(uint val) public {
        logic_pointer.delegatecall(abi.encodeWithSignature("set(uint256)", val));
}
```
Here we see that the function set of the LogicContract is setting the value of variable "a" of LogicContract function. We also see that in the CallingContract's setA function, a delegate call is made to the set function of LogicContract.

So what exactly this delegate call should do? When called, it should change the value of variable "a" of the LogicContract, right? Wrong, remember I mentioned that delegate call maintains the context of the calling contract. So it should change the value of variable "a" of the CallingContract, right? Well, I am sorry to disappoint😔 but it's wrong again.

Now let's see the storage layout of both the contracts, keeping in mind what I told about how solidity stores variables. We see that variable "b" of CallingContract will be stored in slot[0], variable "a" will be stored in slot[1]. Similarly variable "a" of LogicContract will be stored in slot[0]. Now when set function of LogicContract is called from CallingContract through a delegatecall. It would be changing the value of variable stored in slot[0] so it will change the value of variable "b" because the context of calling contract is maintained in delegate call and in CallingContract, variable "b" is stored in slot[0].

Isn't it blows your mind🤯 that while writing the code you don't just need to care about syntax and errors but also how the compiler is storing the variables of a contract.

# Contract for Case Study ( Analysis )
Let's see a real-world hack that took the world into pieces, and it occurred due to delegate attack.

## Parity Multisig Wallet (Second Hack)
The second parity multisig wallet hack told the world that how well written contracts can also be hacked due to delegate calls. A lot of people and companies used parity-generated multisig wallets and when the attack took place, around 300M$ were frozen as no one was allowed to withdraw money from the parity multisig wallet.

Let's look at two contracts to understand what happened.
```
contract WalletLibrary is WalletEvents {
    ...
    // throw unless the contract is not yet initialized.
    modifier only_uninitialized { if (m_numOwners > 0) throw; _; }
    // constructor - just pass on the owner array to multiowned and
    // the limit to daylimit
    function initWallet(address[] _owners, uint _required, uint _daylimit)
        only_uninitialized {
        initDaylimit(_daylimit);
        initMultiowned(_owners, _required);
    }
    // kills the contract sending everything to `_to`.
    function kill(address _to) onlymanyowners(sha3(msg.data)) external {
        suicide(_to);
    }
    ...
}
```
```
contract Wallet is WalletEvents {
    ...
    // METHODS
    // gets called when no other function matches
    function() payable {
        // just being sent some cash?
        if (msg.value > 0)
            Deposit(msg.sender, msg.value);
        else if (msg.data.length > 0)
            _walletLibrary.delegatecall(msg.data);
    }
    ...
    // FIELDS
    address constant _walletLibrary =
    0xcafecafecafecafecafecafecafecafecafecafe;
}
```
Here are some part of the wallet and wallet library contract which were used for the implementation of multisig wallets. Here we can see that the Wallet contract is making a lot of delegate calls to WalletLibrary contract. So WalletLibrary is doing most of heavy lifting.

Now if we closely read the contract and think more about it, we will realise that anyone can deploy a contract with WalletLibrary address and can call the initWallet function through our infamous delegate call. There is no check to ensure that the call is coming from the right contract. And Boom, anyone can become owner of the contract by making the proper delegate call and thus can drain the wallet of any other user.

In the real world, at the time of attack. A user named @devops199 hacked the contract and by mistake(as mentioned by him/her) called the initWallet function to get the ownership of the wallet and then again by mistake it called the function kill which self destructed the whole wallet.

Here the attack would have been prevented if the constructor logic was not implemented in the library but It's worth noting that the concept of abstracting logic into a shared library can be really valuable. It aids in code reuse and lowers the cost of gas deployment. 

However, this hack demonstrates that the Ethereum ecosystem need some more set of standards and best practices to ensure that these code patterns are executed properly and securely. Otherwise, even the most innocent-looking issue can cause havoc.

Here are the two calls that were made by the hacker to hack the contract.
https://etherscan.io/tx/0x9dbf0326a03a2a3719c27be4fa69aacc9857fd231a8d9dcaede4bb083def75ec
https://etherscan.io/tx/0xeef10fc5170f669b86c4cd0444882a96087221325f8bf2f55d6188633aa7be7c

# Preventive Methods 
Here we will figure out how we can prevent delegate calls attack.
## Stateless Libraries
Solidity provides a keyword library that can be used to implement library contracts. This guarantees that the library contract is stateless and unbreakable.

The complications of storage context are mitigated by forcing libraries to be stateless.
Stateless libraries also protect against attacks in which attackers change the state of the library directly to affect contracts that rely on its code.

When using DELEGATECALL, pay close attention to the possible calling context of both the library contract and the calling contract, and construct stateless libraries whenever possible. 

# Description of Task  
Now let's get our hands dirty. Through this task we will be deploying our contracts on remix and would see how the delegate call attacks take place.

```
pragma solidity 0.8.10;

contract RunSomeLogic{
    uint public money;
    address public owner;

    constructor(){
        money = 5;
    }

    function setOwner() public{
        owner = msg.sender;
    }

    function withdraw(uint amount) public{
        money = money - amount;
    }
}

contract doCall{
    uint public money;
    address public owner;
    
    RunSomeLogic _RunSomeLogic = new RunSomeLogic();

    constructor(){
        money = 10;
    }

    modifier onlyOwner {
        require(msg.sender == owner);
        _;
    }

    function withdrawMoney(uint amount) onlyOwner public{
        address(_RunSomeLogic).delegatecall(abi.encodeWithSignature("withdraw(uint)", amount));
    }

    
    fallback() external {
        (bool result,) = address(_RunSomeLogic).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```
Copy and paste this code on your <b>Remix IDE</b> and let's get started. Here we have two contracts, one named RunSomeLogic and the other named doCall.

Money matters a lot in this world and in our contracts, it can be withdrawn only using the withdrawMoney function. Now withdraw money is an onlyOwner function. So we need to get the ownership of the contract to withdraw the money. So Avengers, Today our task is to get the ownership of doCall contract.

We can see that the doCall contract makes some delegate calls to the RunSomeLogic contract. Let's see if we can use some of our learning to get the ownership of the contract.

If we closely review the code of doCall contract, we see that it's fallback function contains an exciting delegate call. Well you must be thinking what's so interesting about it? The exciting part is that in this particular delegate call, msg.data is passed as function name. 

So we can specify the delegate call function by ourselves, it's a jackpot for us😎. Will see how.

In the RunSomeLogic contract, we have the setOwner function which change the owner to msg.sender, now if we make the delegate call to this function from any other external contract then we can simply get the ownership of that external contract(if the storage layout is same as that of RunSomeLogic contract)😱. 

As mentioned above, delegate call maintains the context of calling contract. So actually the owner which is being set in setOwner function will actually set the owner of the calling contract.

We get the deployed contract instance when we deploy our doCall contract on remix. On the bottom of this instance, we get the "Transact" option.

<!-- Image of remix's transact button -->

Now we just need to send the keccak256 hash of the string "setOwner()" in the calldata and then we are done. You can use any online tool to calculate this hash, once you get it, pass it on remix and press the transact button. 

Now just click the owner button above and boom💥, we will see our account address as the owner of the contract.

<!-- Image showing the address of the owner -->

Now since you are the contract owner, feel free to withdraw the money and drain the wallet.