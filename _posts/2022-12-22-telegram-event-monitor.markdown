---
layout: post
title:  "Telegram Event Monitor"
date:   2022-12-21
categories: 
---
* TOC 
{:toc}
We implement a service that tracks well-known Ethereum wallets, and sends alerts of real-time ETH or ERC20 token transfers via Telegram, using Python's `asyncio` and `websockets` libraries. We subscribe to JSON-RPC events from a node over WebSocket connection, and parse transaction logs emitted by the events. We also give a concrete example of how a probabilistic data structure called bloom filter is used by the node to efficiently filter for relevant logs.

## Prerequisites
### `asyncio` library

Our messaging service will rely on running asynchronous code, since the Telegram bot will wait on external Ethereum events. Python's [`asyncio`](https://docs.python.org/3/library/asyncio.html) is an indispensible library that enables running concurrent code using `async def`/`await` syntax. Here we'll cover the basics of `async def`, `await`, event loops, coroutines, and tasks, which will be enough for the purpose of building our Telegram bot.

Before diving into syntax, it is conceptually important to emphasize the distinction between **concurrency** and **parallelism** (`asyncio` implements the former, but not the latter). This library does not make code multi-threaded or sidestep the global interpreter lock (GIL), a mutex which prevents race conditions from multiple threads simultaneously accessing the same Python objects.

Instead, consider the following cooking analogy. Concurrency is one restaurant chef preparing a soup, while also cutting vegetables. When waiting for the water to boil for the soup, the chef can work on cutting some the vegetables. Once the water is boiling, the chef might stop cutting vegetables and work on adding several ingredients to the soup. Then, the chef continues might cut more vegetables while waiting for the soup to boil once again. 
- Removing concurrency would mean preparing the soup from start to finish, then cutting vegetables only after the soup has been served (very inefficient). 
- Concurrency is still one chef doing one thing at a time, but allows switching back and forth between several tasks.
- Parallelism, on the other hand, is having *two* chefs in the kitchen, one working on the soup, and the other working on cutting vegetables.
- As this very helpful [BBC multi-part series on asyncio](https://bbc.github.io/cloudfit-public-docs/asyncio/asyncio-part-1.html) states, “It’s not about using multiple cores, it’s about using a single core more efficiently”

When processes are spending most of the time using CPU resources (say, doing lots of arithmetic calculations like training a neural net), `asyncio` is not useful. Instead, it should be used when processes are IO-bound, meaning most of the time is spent sending and receiving data.

Indeed, building a Telegram bot tracking Ethereum activity is heavily IO-bound, as new blocks of transactions confirm only (roughly) every 10 second-interval. While waiting to receive transaction data and event logs, our CPU could be working on executing other tasks, such as parsing received data and sending updates on Telegram.

**Event Loop** 

`asyncio` revolves around an **event loop** which consists of a list of **tasks**, which are thin wrappers around **coroutines**, which are functions amenable to concurrent execution, such as cooking soup and cutting vegetables. where execution is allowed to bounce back and forth between them. 

- Firstly, event loops can only have one coroutine *actually* executing (using CPU) at once. The coroutine executes as if in a normal (synchronous) program, until it needs to **await** (for something like I/O from a node), at which point the coroutine yields control back to event loop. The event loop pauses the current task, and looks for other tasks to execute in the meantime.
- Secondly, event loops cannot interrupt the current, executing coroutine . The event loop must wait until the coroutine yields control (due to awaiting some I/O), before switching to another task. 
> :bulb: To prematurely yield control back to the event loop, use `await asyncio.sleep(0)` inside the coroutine, which interrupts the current executing coroutine to work on other ones -- assuming there are other pending coroutines (tasks) 

**Coroutines**

To declare a coroutine function, we use `async def` instead of the usual `def`. Inside such asynchronous functions, we can make use of special keywords like `await` and `async for` (see [`websockets`](#websockets-library) section).

Unlike regular synchronous functions, coroutines do *not* execute when called: if we don’t specify `await` , a coroutine does not run at all! However, `await` can only be used on coroutines within an asynchronous code blocks. To run a coroutine in a synchronous context, we pass the coroutine instance to `asyncio.run()`. Note that `asyncio.run()` starts a new event loop, so one cannot call nest calls to `asyncio.run()` since it would mean starting a new event loop within another.


```python
async def example_coroutine(a,b):
    return a+b

##prints <coroutine object example_coroutine at 0x7f7cf7ae05c0>
print(example_coroutine(1,1)) 
##prints 2
print(asyncio.run(example_coroutine(1,1))) 
```

To show how `await` is used to run a coroutine (which belongs to a class of “awaitables”) inside asynchronous functions, we quote an example from the [official docs](https://docs.python.org/3/library/asyncio-task.html#coroutines) . The return value of an `await` statement is the same as the return value of the coroutine function’s code block.

```python
import asyncio
import time 
async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)
	
async def main():
    print(f"started at {time.strftime('%X')}")
    await say_after(1, 'hello')
    await say_after(2, 'world')
    print(f"finished at {time.strftime('%X')}")

asyncio.run(main())
```

```python
##Output:
started at 22:14:35
hello
world
finished at 22:14:38
```

Notice that the above example does *not* exhibit concurrency at all. It printed “hello” after 1 second, then printed “world” after *another* 2 seconds. Not that impressive.

**Tasks**

To properly run coroutines concurrently, we need to wrap coroutines as **tasks** to allow coroutines to run concurrently in an event loop, using `asyncio.create_task(coro)` which takes a coroutine object as the parameter and returns a `Task` object (inherits `asyncio.Future` which is a low-level awaitable object with a `done()` boolean and `result()` property) . 

```python
async def main():
    task1 = asyncio.create_task(
        say_after(1, 'hello'))

    task2 = asyncio.create_task(
        say_after(2, 'world'))

    print(f"started at {time.strftime('%X')}")
    await task1
    await task2
    print(f"finished at {time.strftime('%X')}")

# This takes 2 seconds, instead of 3!
asyncio.run(main())
```

A more elegant way to run multiple awaitable tasks concurrently is to use `asyncio.gather`, i.e. `await asyncio.gather(task1, task2)` instead of `await task1` ,`await task2`.

### `websockets` library

As for connecting to our Geth node via WebSocket to receive messages (updates of on-chain activity), we can use `brownie` and `websocket` Python libraries to send and receive JSON-RPC messages as follows:

```python
async for websocket in websockets.connect(uri=brownie.web3.provider.endpoint_uri):
    await websocket.send(...)
    message = await websocket.recv()
    ...
```

where `brownie.web3.provider` is a `web3.providers.websocket.WebsocketProvider` object and `uri=brownie.web3.provider.endpoint_uri` might be `ws://localhost:3334`, or whatever you set in `~/.brownie/network-config.yaml` as seen in the section on [bot setup](#telegram-bot). 

This usage of `websockets.connect` is an example of infinite asynchronous iterators, which allows us to reconnect automatically on errors (see [websockets docs](https://websockets.readthedocs.io/en/stable/reference/client.html#websockets.client.connect)). The iterable represents a source of data which can be looped over, in this case the source of data is an iterable object is called `websocket`. Then, to asynchronously interact with the Ethereum node, we use `await websocket.send()` to send a message and `await websocket.recv()` to receive the next message. 


## Telegram Bot

For clarity and completeness, the repo with the full project code can be found on [Github](https://github.com/axucar/telegram-eth-alerts).

**Telegram API keys**

1. Create `.env` with:

    ```text
    TELEGRAM_CHAT_ID=<CHAT_ID>
    TELEGRAM_API_KEY=<BOT_API_TOKEN>
    ALCHEMY_API_KEY=<OPTIONAL> 
    ETHERSCAN_TOKEN=
    ```
    The `TELEGRAM_API_KEY` token is a string that authenticates your bot (not your account). Each bot has a unique token, which is obtained by contacting `@BotFather`, issuing the `/newbot` command and following the steps until you're given a new token (see [docs](https://core.telegram.org/bots/tutorial#obtain-your-bot-token)).

    The `TELEGRAM_CHAT_ID`, which should be an integer, uniquely identifies your user account (not the bot). To obtain the Chat ID, DM the command `/start` to `@userinfobot` on the Telegram app.

2. Running `./install.sh` installs the Python packages needed such as `websockets, eth-brownie, python-telegram-bot`, etc.
3. Set node connections details in `~/.brownie/network-config.yaml`. For instance, I named my local node's Websocket connection `mainnet-localnode-ws` (I show Alchemy option as well), so `brownie.network.connect('mainnet-localnode-ws')` should work in Python.

```python
live:
- name: Ethereum
  networks:
  - chainid: 1
    explorer: https://api.etherscan.io/api
    host: wss://eth-mainnet.g.alchemy.com/v2/$ALCHEMY_API_KEY
    id: mainnet-alchemy-wss
    name: Mainnet (ETH alchemy)
    provider: alchemy
  - chainid: 1
    explorer: https://api.etherscan.io/api
    host: ws://localhost:3334 
    id: mainnet-localnode-ws
    name: Mainnet (ETH local node WSS)
    provider: localnode
```

**Nested event loops using `nest-asyncio`**

One tricky part of this project is combining the Telegram bot application with the activity-tracking coroutines. We want to concurrently listen to user commands on Telegram (.e.g `\start` and `\stop`), while also concurrently listening to our Ethereum node for activity. Consider the following code:

```python
def main():    
    # Create the Application and pass it your bot's token.    
    application = Application.builder().token(cfg.BOT_TOKEN).build()
    # Run the bot until the user presses Ctrl-C
    application.run_polling()
    asyncio.run(track_eth_transfers())

if __name__ == "__main__":    
    main()
```

The line `application.run_polling()` initializes the Telegram bot which listens for commands and messages we send to it, and only shuts down when we press Ctrl-C to interrupt the process. Therefore, `asyncio.run(track_eth_transfers())`, which subscribes to the node's WebSocket for activity tracking, starts only after the bot is shut-down (with Ctrl-C).

The solution is to also make initializing the bot a coroutine, so it can be run concurrently with `track_eth_transfers()` coroutine. Consider making `init_bot()` a coroutine as such:

```python
async def init_bot() -> None:
    """Start the bot."""
    # on different commands - answer in Telegram
    cfg.application.add_handler(CommandHandler("start", start))
    cfg.application.add_handler(CommandHandler("stop", stop))

    # Run the bot until the user presses Ctrl-C
    cfg.application.run_polling()

def main():    
    # Create the Application and pass it your bot's token.    
    cfg.application = Application.builder().token(cfg.BOT_TOKEN).build()    
    asyncio.run(init_bot())

if __name__ == "__main__":    
    main()
```
Unfortunately, `application.run_polling()` was meant to be started from a purely synchronous context, not in an asynchronous context as a coroutine. In fact, the above code errors with `RuntimeError: This event loop is already running`. The reason is that `application.run_polling()` calls `asyncio.get_event_loop().run_until_complete(...)`, which checks whether the event loop is already running. Indeed, `asyncio.run(...)` had already created an event loop and called `loop.run_until_complete(...)`, so inevitably the event loop was already running. This error is detailed in this [StackOverflow post](https://stackoverflow.com/questions/46827007/runtimeerror-this-event-loop-is-already-running-in-python).

In short, `asyncio` by default does not allow an event loop to be started when the event loop has already been started (event loop is "re-entered"). To override this check, and allow `run_until_complete()` to be called, while `run_until_complete()` is already on the callstack, we can use `nest_asyncio`. As the [docs](https://pypi.org/project/nest-asyncio/) state, "the `nest_asyncio` module patches asyncio to allow nested use of `asyncio.run` and `loop.run_until_complete`", just what we need. Now, the following code properly runs several tasks concurrently, including initializing the Telegram bot:

```python
import nest_asyncio
nest_asyncio.apply()

async def init_bot() -> None:
    """Start the bot."""
    # on different commands - answer in Telegram
    cfg.application.add_handler(CommandHandler("start", start))
    cfg.application.add_handler(CommandHandler("stop", stop))

    # Run the bot until the user presses Ctrl-C
    cfg.application.run_polling()

async def main():    
    # Create the Application and pass it your bot's token.    
    cfg.application = Application.builder().token(cfg.BOT_TOKEN).build()    
    ##equivalently, can do await task1, await task2, etc.
    await asyncio.gather(
        asyncio.create_task(track_eth_transfers()),         
        ...       
        asyncio.create_task(init_bot()),
    )

if __name__ == "__main__":    
    asyncio.run(main())    
```
    
With the Telegram bot set-up, in any coroutine function (such as `track_eth_transfers()`), we can send a message on the app:
```python
await application.bot.send_message(cfg.USER_ID,"Hello World",parse_mode=telegram.constants.ParseMode.MARKDOWN_V2)
```

## Track Ethereum Activity

The idea is to track ETH and ERC20 transfers from well-known informed market participants. To source wallet addresses, we often find data sleuths on Twitter doxx certain fund wallets, Nansen has a smart money labeling services, and Etherscan also provides tags for large exchange wallets. For instace, I found addresses related to Vitalik, Jump Trading, Ethereum Foundation, Three Arrows Capital, Wintermute, etc. (see my [config.py](https://github.com/axucar/telegram-eth-alerts/blob/main/config.py) for more).

Subscribing to JSON-RPC notifications for real-time events on Geth is called via `eth_subscribe` method which takes the subscription type as parameter. We will use `newHeads` for ETH transfers, and `logs` for ERC20 transfers (see [Geth's subscription docs](https://geth.ethereum.org/docs/rpc/pubsub)]).

### ETH Transfers

To subscribe to `newHeads` via WebSocket: 

```python
await websocket.send(json.dumps(
      {
          "id": 1,
          "method": "eth_subscribe",
          "params": ["newHeads"],
      }))
```

Once subscribed, we await new messages which represent new Ethereum blocks being confirmed roughly every 10s, and use `brownie.web3` to fetch the transaction data for every transaction in the new block. From there, we just parse the data for the `to_address`, `from_address` , and `eth_transfer_value`

```python
message = json.loads(await websocket.recv())                                  
block_number = int(message.get("params").get("result").get("number"),16)                                        
block_tx_data = brownie.web3.eth.get_block(block_number,full_transactions=True).get("transactions")

for tx_data in block_tx_data:
    if not tx_data.get("to"): continue #contract creation, skip                        
    
    ##filter on transfer value   
    cfg.weth.update_price()                                                
    eth_transfer_value = tx_data.get("value")/(10**18)
    usd_transfer_value = eth_transfer_value*cfg.weth.price
    if usd_transfer_value <= usd_threshold: continue
    
    ##get to, from addresses
    tx_hash = tx_data.get("hash").hex()
    to_address = brownie.convert.to_address(tx_data.get("to"))
    from_address = brownie.convert.to_address(tx_data.get("from"))
    
    ##MATCH "to" and "from"
    if (to_address in cfg.WALLETS_TRACKED):
        wallet = cfg.WALLETS_TRACKED[to_address]
        sent_or_received = "received (+)"
    elif (from_address in cfg.WALLETS_TRACKED):
        wallet = cfg.WALLETS_TRACKED[from_address]
        sent_or_received = "sent (-)"
    else: continue

    output=f'ETH Transfer {tx_hash}:{wallet} {sent_or_received} {eth_transfer_value} ETH (${usd_transfer_value:0,.2f})'
```

### ERC20 Token Transfers

Notice that while ETH transfers do not emit any logs (e.g. [tx](https://etherscan.io/tx/0xc46d4c8232a576352a1c31855264d7d3ab306233d315b0a05196eeccb9cfdb24)), all ERC20 transfers emit `Transfer(index_topic_1 address from, index_topic_2 address to, uint256 value)` logs (e.g. [tx](https://etherscan.io/tx/0x9a6da392d493ba88581ce5b066da7285c9763565a33945d7b4c4c5491659c726#eventlog)).

When subscribing to `logs` of new imported blocks via WebSocket, we can optionally filter based on topics, by providing a list of 32-bytes keccak hash of the canonical event signatures you want to track (e.g. `brownie.web3.keccak(text="Transfer(address,address,uint256)").hex()` = `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`. In addition, we can specify `“address”` param to be a list of contract addresses, such that only logs created from these contracts will be shown. For our purpose, the contract addresses will correspond to our list of ERC20 token contract addresses (WETH, WBTC, DAI, USDC, etc.) defined in [config.py](https://github.com/axucar/telegram-eth-alerts/blob/main/config.py).

```
await websocket.send(json.dumps(
      {
          "id": 1,
          "method": "eth_subscribe",
          "params": ["logs",{
              "topics": [brownie.web3.keccak(text="Transfer(address,address,uint256)").hex()],
              "address": [*cfg.token_dict], #take advantage of log bloom; only show logs of tracked ERC20 tokens
              }],
      }))
```

Once we receive a message object, `message = json.loads(await websocket.recv())`, we find a JSON data structure like this (from this [transaction](https://etherscan.io/tx/0x981eb1ce252f5cf10fc148a721384dd87ebb91f56659e60589b43550e29abcbd)).

```python
{
   "jsonrpc":"2.0",
   "method":"eth_subscription",
   "params":{
      "subscription":"0xed0150cfd7a647cf822c3a2e9821c535",
      "result":{
         "address":"0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
         "topics":[
            "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",
            "0x00000000000000000000000056178a0d5f301baf6cf3e1cd53d9863437345bf9",
            "0x000000000000000000000000a57bd00134b2850b2a1c55860c9e9ea100fdd6cf"
         ],
         "data":"0x00000000000000000000000000000000000000000000000000001e875ab7ebdc",
         "blockNumber":"0xf7b8b0",
         "transactionHash":"0x981eb1ce252f5cf10fc148a721384dd87ebb91f56659e60589b43550e29abcbd",
         "transactionIndex":"0x0",
         "blockHash":"0x24701993af8d75e0da79a1e8b8b2ec37f11b92db06883901698ce5a025f609a9",
         "logIndex":"0x0",
         "removed":false
      }
   }
}
```

The 20-byte address `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` is USDC, meaning it was a transfer of such tokens. The first element `topics[0]` corresponds to the keccak hash of canonical `Transfer(address from,address to,uint256 value)` signature we saw earlier. The second and third topics `topics[1], topics[2]` are the parameters `index_topic_1 address from, index_topic_2 address to` . There can only be up to three 32-byte topics in the array. Hence, whatever parameters remain go to the `data` param (padded to 32 bytes and concatenated) `0x00000000000000000000000000000000000000000000000000001e875ab7ebdc` , which in this case corresponds to the `uint256 value` of the transfer; the hex decodes to 33566691421148 units of tokens = 33,566,691.421148 USDC.

Similar to the tracker function for ETH transfers, we parse the data for the `to_address`, `from_address` , and `usd_transfer_value` as below:

```python
message = json.loads(await websocket.recv())   
erc20_address = brownie.convert.to_address(message.get("params").get("result").get("address"))

##filter on transfer value               
token = cfg.token_dict[erc20_address]
token.update_price()
event_data = message.get("params").get("result").get("data")
transfer_value = eth_abi.decode_single("uint256",bytes.fromhex(event_data[2:])) #Data for Transfer events is only wad                        
usd_transfer_value = (transfer_value/(10**token.decimals))*token.price
if usd_transfer_value <= usd_threshold: continue

##get to, from addresses
tx_hash = message.get("params").get("result").get("transactionHash")
_ , raw_from_address, raw_to_address = message.get("params").get("result").get("topics")
to_address = brownie.convert.to_address(eth_abi.decode_single("address",bytes.fromhex(raw_to_address[2:])))
from_address = brownie.convert.to_address(eth_abi.decode_single("address",bytes.fromhex(raw_from_address[2:])))

##MATCH "to" or "from"
if (to_address in cfg.WALLETS_TRACKED):
    wallet = cfg.WALLETS_TRACKED[to_address]
    sent_or_received = "received (+)"
elif (from_address in cfg.WALLETS_TRACKED):
    wallet = cfg.WALLETS_TRACKED[from_address]
    sent_or_received = "sent (-)"
else: continue

output=f'ERC20 Transfer {tx_hash}:{wallet} {sent_or_received} {transfer_value/10**token.decimals} {token.symbol} (${usd_transfer_value:0,.2f})'
```

## Bloom Filters for Ethereum Logs

One thing to consider is that while the `newHeads` method simply returns all the transactions in a block, the `logs` subscription method has much more data to filter through, since one transaction may contain tens of logs. Ethereum efficiently searches through all the logs across a block’s transactions by computing (per block) a fixed-size 2048-bit bloom filter (0s and 1s), usually represented in 512 hexadecimal characters (256-bytes). Bloom filters save time when checking for membership in a set (e.g. *Did any USDC transfers happen in this block?*), as we will demonstrate below. 

The main idea is to set certain indices of a 2048-bit array to 1, based on hashes of the log data, keeping the remaining bits at 0. For instance, let's say hashing the USDC contract address gives a hash that implies setting the first 3 indices to 1. If the next block's bloom filter is anything other than 111..... (say 011..... or 001.....) we can safely conclude there are no USDC transfers in the *entire* block. Formally, bloom filters reduce membership checks to $$O(1)$$ time instead of $$O(N)$$, using only $$O(1)$$ space (in fact, just 2048-bits). It may have false positives (e.g. if hash of WBTC contract address also sets the first 3 indices to 1, USDC and WBTC logs are indistinguishable), but more importantly no false negatives.

We can query for a block's bloom filter in Python as such: `bloom_filter_hex=brownie.web3.eth.get_block(16234672)["logsBloom"].hex()`

For instance, block 16234672 has the following bloom filter in hex format:

`bloom_filter_hex`=`0x76a0c119e9d0d5197813b902c2d3162d82252da45b1f18d0a085b5304ca178e5580f27c3984a0452e0685f324a8699bd022181011e1568a04e54d881176ee93a28a3f18cd1a2f9ee7f06aaadc790c068fa1db091855712e149e27e4a8800d83c17b851853ac253d3cd14819098618de383d65d7248546fc2220bd3d048dde15413b08a534f23309c2d4d0eca2146810569878081956d8489473d8cc1faba18f1bb7a894f50c96893ed1dc0d68044eeaacaff52ac8aa3c15b0def2e8a85b942f94d63ec5b98344b49e8485a40c0076b3c95597f4311ac4ddb5a06d92b4d6ffa1138bcafa8a9666f27649670f4b00619475c3d70f54bb82060636b9e11e403d962`

To construct a bloom filter, start with 2048-bits all set to 0. Then, for each address and topic in every event log in this block’s transactions, we set 3 bits to 1 (based on some hashes and modulo functions we will show below). For visualization purposes, we convert the hex representation of the above `bloom_filter_hex` to binary, made up of 2048-bits.

```python
## we subtract 2 from `len(bloom_filter_hex)` to account for the 0x prefix in the hexstring.
bloom_filter = bin(int(bloom_filter_hex, 16))[2:].zfill((len(bloom_filter_hex)-2) * 4)
```
Print block 16234672's bloom filter (in full 2048-bit binary glory):
`01110110101000001100000100011001111010011101000011010101000110010111100000010011101110010000001011000010110100110001011000101101100000100010010100101101101001000101101100011111000110001101000010100000100001011011010100110000010011001010000101111000111001010101100000001111001001111100001110011000010010100000010001010010111000000110100001011111001100100100101010000110100110011011110100000010001000011000000100000001000111100001010101101000101000000100111001010100110110001000000100010111011011101110100100111010001010001010001111110001100011001101000110100010111110011110111001111111000001101010101010101101110001111001000011000000011010001111101000011101101100001001000110000101010101110001001011100001010010011110001001111110010010101000100000000000110110000011110000010111101110000101000110000101001110101100001001010011110100111100110100010100100000011001000010011000011000011000110111100011100000111101011001011101011100100100100001010100011011111100001000100010000010111101001111010000010010001101110111100001010101000001001110110000100010100101001101001111001000110011000010011100001011010100110100001110110010100010000101000110100000010000010101101001100001111000000010000001100101010110110110000100100010010100011100111101100011001100000111111010101110100001100011110001101110110111101010001001010011110101000011001001011010001001001111101101000111011100000011010110100000000100010011101110101010101100101011111111010100101010110010001010101000111100000101011011000011011110111100101110100010101000010110111001010000101111100101001101011000111110110001011011100110000011010001001011010010011110100001001000010110100100000011000000000001110110101100111100100101010101100101111111010000110001000110101100010011011101101101011010000001101101100100101011010011010110111111111010000100010011100010111100101011111010100010101001011001100110111100100111011001001001011001110000111101001011000000000110000110010100011101011100001111010111000011110101010010111011100000100000011000000110001101101011100111100001000111100100000000111101100101100010`

Let's denote a transaction log entry as $O \equiv (O_a, O_{\textbf{t}}, O_{\textbf{d}})$, where  $O_a$ is the 20-byte address that generates the logs (USDC token contract address in the example above), and $$O_{\textbf{t}} = (O_{\text{t}_0},O_{\text{t}_1},O_{\text{t}_2})$$ as the list of three 32-byte log topics (see the JSON `message` object in the previous section). Besides the logger’s address and three indexed topics, we might have some additional non-indexed data $O_{\textbf{d}}$ which is not relevant to computing the filter. 

To be precise, we quote the Bloom filter $M(O)$ definition for a single transaction log $$O$$ from the [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf).

$$
M(O)\equiv \bigvee_{x \in \{ O_a \} \cup O_{\textbf{t}}} (M_{3:2048}(x))
$$

where $\bigvee$ can be understood as the bitwise maximum of the various 2048-bit $M_{3:2048}(x)$ objects corresponding to each piece of log data (logger's contract address and topics). For instance, let the binary representation of two 4-bit bloom filters $y_1=0101_2$, $y_2=1010_2$. Then, $\bigvee \{y_1, y_2\} = 1111_2$. The $\bigvee$ operation can also be applied across all transaction logs in a block to arrive at a bloom filter summarizing the entire block's logs (of course, increases false positive rate).

$$
\begin{align}
M_{3:2048}(\textbf{x}:\textbf{x} \in \mathbb{B}) &\equiv \textbf{y}:\textbf{y} \in \mathbb{B}_{256} &\text{ where:} \\
\textbf{y} &= (0,0,...,0) &\text{ except:}\\
\forall i \in \{0,2,4\} &: \mathcal{B}_{2047-m(\textbf{x},i)}(\textbf{y}) = 1\\
m(\textbf{x},i) &\equiv \texttt{KEC}(\text{x})[i,i+1] \text{ mod 2048}
\end{align}
$$

where $\mathbb{B}$ refers to bytes, and $\mathcal{B}$ is “the bit reference function” such that $$\mathcal{B}_{j}(\textbf{y})$$ equals the bit indexed at $j$ (indexed from 0) in the (2048-bit) 256-byte array $\textbf{y}$. To make things concrete, let’s actually calculate $y=M_{3:2048}(x)$ where $x=O_a$, the logger’s address USDC contract. Note that $x$ represents a piece of log information (address or topic), and $y$ represents its Bloom filter.

```python
usdc_contract_address='0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48' ## refers to x=O_a 
kec_x = brownie.web3.keccak(hexstr=usdc_contract_address).hex()
```
 $\texttt{KEC}(x)$ = `'0x7b5855bb92cd7f3f78137497df02f6ccb9badda93d9782e0f230c807ba728be0'`

Then, we take “each of the first three pairs of bytes” of the keccak hash $\texttt{KEC}(x)$ (ie. three 4-hex-character chunks), convert to base-10 integers, then modulo each by $$2048$$. Note that each 2-byte chunk is capable of expressing 16 bits, ie. up to $$2^{16} = 65536$$ numbers, but we only take the “low-order 11 bits of each” out of the 16 bits expressed, since we modulo by $$2048=2^{11}$$.

- $m(x,0)$ = `int('0x7b58',16) % 2048 = 856`
- $m(x,2)$ = `int('0x55bb',16) % 2048 = 1467`
- $m(x,4)$ =`int('0x92cd',16) % 2048 = 717`

Finally set bits at the indices $2047-m(x,i)=1191, 580, 1330$, respectively. Hence, we set $$\mathcal{B}_{1191}(y)=\mathcal{B}_{580}(y)=\mathcal{B}_{1330}(y)=1$$. To verify our work, we notice that indeed `bloom_filter[index]` for `index` in [1191,580,1330] are set to 1!! This indicates that there is definitely a log produced by the USDC contract.

Finally, we can repeat this exercise with the log topics $$x = O_{\textbf{t}_i}$$, and again computing $M_{3:2048}(x)$ to continue setting more bits to 1 in the bloom filter.

> :warning: For log topics $O_\textbf{t}$, they must be zero-padded to be 32-bytes, before running keccak hash on them, as specified by Yellow Paper. For this transaction, we would replace $x$ with one of the below topics

```python
"0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef", ## event signature "Transfer(address,address,uint256)"
"0x00000000000000000000000056178a0d5f301baf6cf3e1cd53d9863437345bf9", ## "from" address
"0x000000000000000000000000a57bd00134b2850b2a1c55860c9e9ea100fdd6cf", ## "to" address
```

Once we have this block's bloom filter (repeat exercise for all transactions), instead of looping through $$N$$ transaction logs in a block, we can efficiently check whether this block has any USDC transfer logs by verifying that the block’s `bloom_filter[index]==1` at the appropriate indices.