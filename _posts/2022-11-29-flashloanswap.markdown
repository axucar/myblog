---
layout: post
title:  "Flash Loans for Arbitrage"
date:   2022-11-29
categories: 
---
* TOC 
{:toc}
Since multiple state changes can be executed in one atomic transaction (where all or none of the changes are executed), one feature in blockchain contracts is the ability to conduct flash loans, where users can borrow unlimited amounts without collateral but must return funds within the same transaction. Predictably, flash loans have been applied towards arbitrage opportunities in automated market makers (AMMs, ie. protocols that facilitate trading through liquidity pools, instead of traditional order books). Below we demonstrate a minimal prototype of such technique.

## UNI-WETH LP

On-chain arbitrage can generally be found during periods of high price volatility, and this month saw the market-moving events of FTX’s insolvency on Nov. 11, 2022. Within hours of filing for bankruptcy, FTX had also mysteriously been hacked. The FTX drainer account had swapped UNI tokens, among other stolen assets, for WETH across various AMMs including Uniswap V2, V3, and CoW protocols (see [etherscan](https://etherscan.io/tx/0xc5475d0026b74e567ba9fe6b2301e8c8760bcc27670a3fe3e5c984fb3568d153)) at block [15951518](https://etherscan.io/block/15951518). Subsequently, we ought to observe imbalances in the reserves across liquidity pools for UNI-WETH.

For simplicity, we focus only on Uniswap and Sushiswap liquidity pools (LPs) to spot an arbitrage, since the contract of the latter is just a fork of the former. 

- Uniswap UNI-WETH LP `0xd3d2E2692501A5c9Ca623199D38826e513033a17`
- Sushiswap UNI-WETH LP `0xDafd66636E2561b0284EDdE37e42d192F2844D40`

***

**UNI Reserves**

| Block | Uniswap | Sushiswap |
| ---: | ---: | ---: |
| 15951517 | 1,482,000 | 19,050 |
| 15951518 | 1,863,000 | 25,090 |

***

**WETH Reserves**

| Block | Uniswap | Sushiswap |
| ---: | ---: | ---: |
| 15951517 | 6,683.0 | 85.98 |
| 15951518 | 5,324.0 | 65.33 |

***

**UNI/WETH Reserves**

| Block | Uniswap | Sushiswap |
| ---: | ---: | ---: |
| 15951517 | 221.7 | 221.5 |
| 15951518 | 350.0 | 384.0 |

***

The ratio of UNI tokens vs. WETH (wrapped ether) significantly increased across both Uniswap and Sushiswap pools (221.7, 221.5 → 350, 384) respectively on block 15951518 vs. the previous block. While the ratio in the previous block was nearly inline, the reserve ratios at the block in which the FTX drainer dumped UNI tokens have diverged between the two LPs. Note that 350 vs. 384 is a significant deviation relative to the 30bps of swap fees that we have to pay to each LP to trade this, whereas 221.7 vs. 221.5 is not. 

Sushiswap has an abundance of UNI tokens relative to Uniswap, so the arbitrage outline would be:
1. Borrow WETH from Uniswap LP
2. Swap WETH for UNI on Sushiswap LP (where UNI is cheap)
3. Repay the loan from Uniswap LP using the UNI we receive from Sushiswap. Keep the difference (profit) in UNI tokens

## AMM Math

We derive two formulas needed to calculate the arbitrage profit: 1) compute the output of a swap given an input amount, and 2) compute the required input of a swap, given a desired output. They should match Uniswap's reference implemention of these functions in Solidity [here](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol#L43-L59).

Uniswap follows the constant product formula for pricing, where the product of the two reserve quantities remains constant across swaps. Mathematically, this means $(r_{out}-x_{out})(r_{in}+x_{in}) = r_{out}*r_{in} = k$, where $r$, $x$ represent the reserves and the amounts being swapped, respectively, and $k$ is a constant.

1. `getAmountOut(amountIn, reserveIn, reserveOut)` : $x_{out} = f(x_{in}, r_{in}, r_{out})$
    
    Given an input amount and pair reserves, this function returns the maximum we can swap out. We will call this function on Sushiswap LP, where we will input WETH, and swap out UNI. So $r_{in}, r_{out}$ would be the reserves of WETH and UNI respectively, $x_{out}$ be the maximum amount of UNI we can take out of the pool, and $x_{in}$ be the amount of WETH we supply to the Sushiswap pool. We need to solve for $x_{out}$.
    
    $$(r_{out}-x_{out})(r_{in}+x_{in}) = r_{out}*r_{in} = k$$
    
    $$=> x_{out} = r_{out} - \frac{r_{in}*r_{out}}{r_{in} + x_{in}} = \frac{r_{out}*x_{in}}{r_{in}+x_{in}}$$
    
    However, there is a 0.3%=3/1000 fee applied on each swap, which is taken directly from the input amount $x_{in}$ of the “in” token. So with the fee applied on $x_{in}$ we have:
    
    $$x_{out} = \frac{r_{out}*0.997*x_{in}}{r_{in}+0.997*x_{in}} = \frac{r_{out}*x_{in}*997}{r_{in}*1000+x_{in}*997}$$
    
    In Python (see `lp_wrapper.py` on [Github](https://github.com/axucar/flashloan-swap/blob/main/lp_wrapper.py)), with `self.fee = Fraction(3,1000)`:
```python
## lp_wrapper.py
def get_amount_out(self, token_in_address:str, amount_in: int) -> int:        
    assert token_in_address in (self.token0.address, self.token1.address), "Provided token_in does not match LP tokens"

    (reserve_in, reserve_out) = (self.reserve_token0, self.reserve_token1) \
            if (token_in_address == self.token0.address) else (self.reserve_token1, self.reserve_token0) 
    
    amount_in_with_fee = amount_in * (self.fee.denominator - self.fee.numerator)
    numerator = amount_in_with_fee * reserve_out
    denominator = reserve_in * self.fee.denominator + amount_in_with_fee
    return numerator // denominator
```
    
2. `getAmountIn(amountOut, reserveIn, reserveOut)`: $x_{in} = g(x_{out}, r_{in}, r_{out})$
    
    Given a desired output amount and the corresponding reserves, this function returns the minimum amount of input $x_{in}$ that must be paid in exchange for $x_{out}$ of the asset we swap out of the pool.
    
    We will call this function to calculate the amount of UNI to repay on Uniswap for having borrowed WETH. Using the constant product formula for AMMs as above, we solve for the required $x_{in}$ amount.
    
    $$(r_{out}-x_{out})(r_{in}+x_{in}) = r_{out}*r_{in} = k$$
    
    $$x_{in} = \frac{r_{out}r_{in}}{r_{out} - x_{out}} - r_{in} = \frac{r_{in}x_{out}}{r_{out} - x_{out}}$$
    
    Adjusting for the 0.3% fees deducted from $x_{in}$, 
    
    $$x_{in} = \frac{r_{in}x_{out}*1000}{(r_{out} - x_{out})*997}$$
    
```python
## lp_wrapper.py
def get_amount_in(self, token_in_address:str, amount_out:int) -> int:    
    assert token_in_address in (self.token0.address, self.token1.address), "Provided token_in does not match LP tokens"

    (reserve_in, reserve_out) = (self.reserve_token0, self.reserve_token1) \
            if (token_in_address == self.token0.address) else (self.reserve_token1, self.reserve_token0) 

    numerator = reserve_in * amount_out * self.fee.denominator
    denominator = (reserve_out - amount_out) * (self.fee.denominator - self.fee.numerator)
    return numerator // denominator + 1
```
    

To give context to these formulas, let’s run through them with actual reserve numbers from block 15951518. 

***

| Reserves | Uniswap | Sushiswap |
| ---: | ---: | ---: |
| UNI | 1,863,000 | 25,090 |
| WETH | 5324.0 | 65.33 |

***
If we start an arbitrage trade by borrowing 2 WETH from Uniswap LP:
1. Borrow 2 WETH from `Uniswap LP`
2. Swap 2 WETH for UNI on `Sushiswap LP` using $x_{out} = f(x_{in} = 2, r_{in} = 65.33, r_{out} = 25,090)$
    - The “in” token is WETH

    $$x_{out} = \frac{25090*2*997}{65.33*1000+2*997} = 743.11 \text{ UNI} $$
    
3. Repay the loan from `Uniswap LP`, where the repay amount in UNI tokens is $x_{in} = g(x_{out} = 2, r_{in} = 1,863,000, r_{out} = 5324)$
    - The “in” token is UNI (ie. we repay `Uniswap LP` with UNI)

    $$x_{in} = \frac{1863000*2*1000}{(5324 - 2)*997} = 702.22 \text{ UNI} $$

This shows that our arbitrage would yield 473.11 - 702.22 = 40.89 UNI tokens. The next step is to determine how much WETH borrow is optimal (instead of just choosing 2 WETH arbitrarily).

## Calculate Arb Profit

Let $f(x) = f(x_{in}=x, r_{in}=65.33, r_{out}=25090)$ be the swap output amount (in UNI) from `Sushiswap LP` given an input of $x$ WETH. 

Let $g(x) = g(x_{out}=x, r_{in}=1863000, r_{out}=5324)$ be the minimum repay amount (in UNI) for `Uniswap LP` assuming we borrowed $x$ WETH.

We can represent the profit $h(x)$ of our flashloan and swap arbitrage, given a borrow amount $x$, as

$$
h(x) = f(x) - g(x)
$$

In Python, we can use the `scipy.optimize` library to find the maximum profit $h(x)$ (see `borrow_swap_arb.py` on [Github](https://github.com/axucar/flashloan-swap/blob/main/borrow_swap_arb.py)).

```python
## borrow_swap_arb.py
opt = optimize.minimize_scalar(
    lambda x: -float(
        self.swap_pool.get_amount_out(self.borrow_token.address, x) -
            self.borrow_pool.get_amount_in(self.repay_token.address, x)),
    method="bounded",
    bounds=bounds,
    bracket=bracket,
)
optim_borrow, optim_profit = int(opt.x), -int(opt.fun)
```

For the `bounds` parameter, the minimum borrow is clearly 1 wei ($10^{-18}$ WETH), whereas the maximum borrow is the total amount of WETH in the borrow pool reserves (`Uniswap LP`). The bracket param allows us to suggest a starting interval for the optimization, which I set to (0.1%, 1%) of the maximum borrow. It does not mean that the final solution will be within this suggested bracket.

## Solidity Contract

Once we have the optimal borrow amount, and thereby the corresponding swap output amount, we execute the arbitrage on-chain using a Solidity script (see full contract [here](https://github.com/axucar/flashloan-swap/blob/main/solidity_flasharb/contracts/arb.sol)). The trigger function `flash_borrow_to_swap` that starts the arbitrage calls `swap()` on the `Uniswap LP` borrow pool: `UniswapV2Pair(borrow_pool_address).swap(uint amount0Out, uint amount1Out, address to, bytes calldata data)`. This call effectively borrows `amount1Out` WETH from `borrow_pool_address` Uniswap UNI-WETH LP, since `token1` for the contract is WETH (`amount0Out=0`). One can easily check `token0, token1` refer to UNI, WETH respectively on [etherscan](https://etherscan.io/address/0xd3d2E2692501A5c9Ca623199D38826e513033a17#readContract) under "Read Contract".

```solidity
//arb.sol
function flash_borrow_to_swap (
    address _borrow_pool_address,
    uint256[2] memory _borrow_amounts,
    address _swap_pool_address,
    uint256[2] memory _swap_out_amounts,
    uint256 _repay_amount
) external {
    require(msg.sender == OWNER, "msg.sender is !OWNER");

    borrow_pool_address = _borrow_pool_address;
    borrow_amounts = _borrow_amounts;
    swap_pool_address = _swap_pool_address;
    swap_out_amounts = _swap_out_amounts;
    repay_amount = _repay_amount;
    
    //this swap will trigger uniswapV2Call
    IUniswapV2Pair(borrow_pool_address).swap(borrow_amounts[0],borrow_amounts[1], address(this), bytes("x"));
}
```

Looking at the `IUniswapV2Pair(borrow_pool_address).swap()` function [on Github](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L159-L187) , we see that it hands flow control back to the `msg.sender` (ie. our custom arbitrage contract written in Solidity), calling our own `uniswapV2Call` function defined in `arb.sol`.

We can effectively do anything we want in this `uniswapV2Call` function. Of course, to ensure we don’t simply run off with the borrowed funds, the LP enforces our repayment at the end of the swap function as shown below.

```solidity
//Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol
if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
... // other code in the between
require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
```

Once we have the flash loan funds, our `uniswapV2Call` function will swap them at the `swap_pool_address` and transfer the `repay_amount` back to the `msg.sender=borrow_pool_address`, ie. the `Uniswap LP` where we got our WETH loan.

1. First, we check that the caller of our contract’s `uniswapV2Call` is precisely the `borrow_pool_address` (ie. Uniswap UNI-WETH LP address) set by `flash_borrow_to_swap` 
2. Transfer the borrowed WETH to `swap_pool_address` (ie. Sushiswap UNI-WETH LP address) 

```solidity
//arb.sol
function uniswapV2Call (address ,uint256 _amount0, uint256 _amount1, bytes calldata) external {
    //this is where we execute flash loan + swap

    // ensure msg.sender is borrow_pool as part of flash loan and swap, as intended 
    require(msg.sender == borrow_pool_address, "!LP");
    
    address _token0_address = IUniswapV2Pair(msg.sender).token0();
    address _token1_address = IUniswapV2Pair(msg.sender).token1();

    //transfer the borrow amounts to swap pool
    if (_amount0 == 0) {
        IERC20(_token1_address).transfer(swap_pool_address,_amount1);
    }
    
    else if (_amount1 == 0) {
        IERC20(_token0_address).transfer(swap_pool_address,_amount0);
    }
        
    IUniswapV2Pair(swap_pool_address).swap(swap_out_amounts[0],swap_out_amounts[1], address(this), bytes(""));

    //if borrowed token1, repay in token0 (and vice versa)
    if (_amount0 == 0) {
        IERC20(_token0_address).transfer(msg.sender, repay_amount);
    }    
    else if (_amount1 == 0) {
        IERC20(_token1_address).transfer(msg.sender, repay_amount);
    }        
    _cleanup();
}
```

### Deploy

I created a Brownie project (`brownie init`) with the Solidity source code in the `contracts` folder, and a `brownie-config.yaml` file that specifies the imports (such as OpenZeppelin libraries) which the source code depend on (see [Github](https://github.com/axucar/flashloan-swap/blob/main/solidity_flasharb/brownie-config.yaml)). Make sure the source code compiles by running `brownie compile` . 

To deploy the contract in a dev environment, we define a local network forked at a specific block number for testing purposes, which is easy to do with Brownie. In `~/.brownie/network-config.yaml` I have something like this:

```python
development:
- cmd: ganache-cli
  cmd_settings:
    accounts: 10
    evm_version: istanbul
    fork: https://eth-mainnet.g.alchemy.com/v2/$ALCHEMY_API_KEY@$BLOCK 
    gas_limit: 12000000
    mnemonic: brownie
    port: 8545
  host: http://127.0.0.1
  id: mainnet-fork-number
  explorer: https://api.etherscan.io/api
  name: Ganache-CLI (Mainnet Fork at number)
  timeout: 120
```

Note that `id` is whatever you want to name this local network, and ensure to source environment variables `$ALCHEMY_API_KEY`, `$BLOCK` which I defined in a `.env` file.

Inside my brownie project folder `solidity_flasharb`, running \
`brownie console --network mainnet-fork-number` \
opens a Python console with our local network forked at `$BLOCK` height. Assuming the Solidity source code compiled, you can now call deploy on the `ArbContract` object.

```python
>>> import os
>>> player=accounts.add(private_key=os.getenv("ETH_PK"))
>>> ArbContract.deploy({'from':player})
Transaction sent: 0x86f53a9b7b4a60b494290457212a10703893a5b59df61f23e382d2f7b9218bbc
  Gas price: 0.0 gwei   Gas limit: 1000000   Nonce: 9
  ArbContract.constructor confirmed   Block: 15951520   Gas used: 608253 (60.83%)
  ArbContract deployed at: 0x5c3E95B217dcfaD62C4d53d33103Bc4a8499dd3A

<ArbContract Contract '0x5c3E95B217dcfaD62C4d53d33103Bc4a8499dd3A'>
```

The process for deploying live is exactly the same but we just connect to a live network instead. For instance, my `~/.brownie/network-config.yaml` set-up for prod looks like:

```solidity
live:
- name: Ethereum
  networks:
  - chainid: 1
    explorer: https://api.etherscan.io/api
    host: ws://localhost:3334 
    id: mainnet-localnode-ws
    multicall2: '0x5BA1e12693Dc8F9c48aAD8770482f4739bEeD696'
    name: Mainnet (ETH local node WS)
    provider: localnode
```

For live deploys, we can optionally verify the contract on Etherscan by publishing the source code.

```python
##to verify contract after the fact:
contract = ArbContract.at("0x5c3E95B217dcfaD62C4d53d33103Bc4a8499dd3A")
ArbContract.publish_source(contract)
```

## Test Arb @ 15951518

Once we have the contract deployed, we run the `main.py` (see [Github](https://github.com/axucar/flashloan-swap/blob/main/main.py)), which calls `flash_borrow_to_swap` on our on-chain contract with appropriate borrow-pool and swap-pool parameters.

To determine whether the transaction is profitable, we should compare the arbitrage profit in USD to the estimated gas costs in USD (we use Chainlink to pull UNI and WETH prices, and `web3.py` to estimate gas usage of the contract call), and ensure we have enough margin of safety (for instance, only execute `if arb_profit_usd > 2*estimated_cost_usd`)

```python
## main.py
web3_f = web3.eth.contract(address = arb_contract.address, abi = arb_contract.abi)
est_gas_usage = web3_f.functions.flash_borrow_to_swap(arb.borrow_pool.address,
        borrow_pool_amounts,
        arb.swap_pool.address,
        swap_pool_amounts,
        arb.res["repay_amount"]).estimateGas({'from':player.address})
estimated_cost_usd = est_gas_usage*((last_base_fee+last_priority_fee)/10**18)*ether_token.price
```

Running our script on a forked local network at block 15951518, we get the following output:

```python
Initializing LP:Uniswap
Fetching source of 0xd3d2E2692501A5c9Ca623199D38826e513033a17 from api.etherscan.io...
Init Reserves for [Uniswap]
UNI/WETH Reserve Ratio: 350.0
WETH/UNI Reserve Ratio: 0.002857
UNI Reserves: 1863000.0
WETH Reserves: 5324.0

Initializing LP:Sushiswap
Fetching source of 0xDafd66636E2561b0284EDdE37e42d192F2844D40 from api.etherscan.io...
Init Reserves for [Sushiswap]
UNI/WETH Reserve Ratio: 384.0
WETH/UNI Reserve Ratio: 0.002604
UNI Reserves: 25090.0
WETH Reserves: 65.33

ARB STEPS:
Borrow 2.87 WETH on Uniswap,
Swap for 1051.43 UNI on Sushiswap,
Repay for 1006.82 UNI on Uniswap,
Profit 44.61 UNI ($254.66)

Estimated gas usage (web3.py): 333769 wei
Base Fee: 30.0 gwei, Priority Fee: 1.0 gwei, Estimated tx cost: $12.94
Executing arbitrage...
Transaction sent: 0xa0e5db8f210854eda6785a19dc54d68424bf6429037ee11e55bfb6994fae1262
  Gas price: 0.0 gwei   Gas limit: 500000   Nonce: 10
  Transaction confirmed   Block: 15951521   Gas used: 205596 (41.12%)

Updating Reserves for [Uniswap]
UNI/WETH Reserve Ratio: 350.4
WETH/UNI Reserve Ratio: 0.002854
UNI Reserves: 1864000.0
WETH Reserves: 5321.0

Updating Reserves for [Sushiswap]
UNI/WETH Reserve Ratio: 352.5
WETH/UNI Reserve Ratio: 0.002837
UNI Reserves: 24040.0
WETH Reserves: 68.2
```

Notice how the UNI/WETH reserve ratio gap is much narrower between Uniswap and Sushiswap after the arbitrage (350, 384) → (350.4, 352.5). 

Since the profit of UNI tokens are in the arbitrage contract address, we can withdraw the tokens to our `player`. Finally, we verify that we indeed made 44.61 UNI (\\$254.66) tokens of profit. Assuming base fee and priority fee of 30+1 = 31 gwei, the gas cost to do the arbitrage would have been = 205596*31 gwei = 0.006373476 ether = \\$7.97 at time of block 15951518.

`brownie console --network mainnet-fork-number`
```python
##Brownie console
uni = Contract.from_explorer('0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984')
ArbContract[0].withdrawTokenToOwner(uni.address, uni.balanceOf(ArbContract[0].address), {'from':player})
uni.balanceOf(player)/10**18 ##should return 44.6060927721087 
```

## Closing thoughts

We showed how a flash loan from Uniswap of 2.87 WETH, swapped into UNI tokens at Sushiswap, turned into a small profit in UNI tokens, yet significant enough to warrant the cost of executing our arbitrage contract call. I was curious how long the opportunity persisted after the large inflow of UNI token supplies into the liquidity pools. In the table below, I ran our script on a local network forked at subsequent blocks to computed the arbitrage profit. It appears the opportunity goes away (tx cost exceeds profit) around 4 blocks later, ie. 15951522 and beyond. 

| Block Number | Profit (UNI) | Profit (USD) | BaseFee (gwei) |
| ---: | ---: | ---: | ---: |
| 15951517 | 0.00 | 0.00 | 32 |
| 15951518 | 44.61 | $254.66 | 30 |
| 15951519 | 202.13 | $1153.96 | 32 |
| 15951520 | 61.55 | $351.37 | 35 |
| 15951521 | 15.47 | $88.33 | 39 |
| 15951522 | 1.08 | $6.15 | 44 |
| 15951523 | 2.28 | $12.99 | 50 |

There are several shortcomings to the current setup, which serves as a proof-of-concept but unlikely to win in real-world live scenarios. 

Firstly, it simply loops continuously every 5-second interval. An improvement would be to use a local Ethereum node to listen for the arrival of new blocks, or filter for `Sync` events which are transaction logs emitted when reserves of liquidity pools are updated (e.g. when a swap on Uniswap occurs), and then immediately update our pool reserves. Furthermore, one could subscribe to new pending transactions in the public mempool of our local Ethereum node, filter for unconfirmed transactions that swap with our relevant liquidity pools, which allows us to predict future pool reserves without waiting for the on-chain confirmation.

Secondly, we would want to use flashbots relay to submit our transaction, whose custom validator client `mev-geth` allows block producers to earn extra rewards via bribes from MEV searchers (arbitrageurs). Instead of competing on priority gas auction on-chain to be included in the block, we submit transactions off-chain to the block producers running `mev-geth` to evaluate for inclusion, and compete based on our bribe. One advantage is that transactions that fail to be included do not cost any gas (since proposing the transaction to these `mev-geth` validators is done off-chain). The other benefit is that our proposed transaction does not enter the public mempool. This privacy feature avoids front-running from others.

Lastly, it may be worth noting that while flash loans are essential when the capital requirement for an opportunity is very large, they are not as gas-efficient as straight-forward swaps (due to calling the custom `UniswapV2Call` callback function). For the example above, if you already have 2.87 WETH, you can achieve the same profit just by swapping against Sushiswap then back in Uniswap while using less gas.