---
layout: post
title:  "Uniswap, Geth, SSH"
date:   2022-08-03
categories: 
---
* TOC 
{:toc}
Behind major Ethereum applications, such as OpenSea for NFTs and dYdX for trading, there is usually a “blockchain developer platform” (e.g. Alchemy, QuickNode, Infura) that acts as an API layer to interact with any given blockchain network. For instance, to pull Uniswap on-chain price for WETH-USDC, I can get an API key from Alchemy and use Uniswap’s SDK (example [below](#uniswap-on-chain-pricing)).

However, in the spirit of learning, I was curious how to run a local Ethereum node instead. I used a NUC (Intel  i7 9700, 32GB RAM, 2TB SSD, Ubuntu). In particular, the Go Ethereum (Geth) client seemed like the easiest one to get started with. Note that for the sake of client diversity for the Ethereum network, it is recommended to use other client implementations since Geth is by far the most popular at this point (for instance, OpenEthereum or Nethermind). 

Since I was running dual boot with Windows with the storage split half-way, I first expanded the Ubuntu partition to make more room. In Ubuntu, a mounted drive that is in use can’t be expanded, so I boot from a live Ubuntu USB in trial mode, and used `gparted` which is pre-installed to do the disk resizing.

## Get started with Geth

I installed Geth using builtin-in [launchpad PPA](https://geth.ethereum.org/docs/install-and-build/installing-geth#ubuntu-via-ppas). 

```bash
sudo add-apt-repository -y ppa:ethereum/ethereum`
sudo apt-get update
sudo apt-get install ethereum
```

### Syncing 
To sync to the current state of the network, just run `geth` in the command line. By default, it will run a full node (using “snap” sync), which is what I did. 
- A full node stores all blocks since genesis, hence all state can be derived from it. 
- Alternatively, one can try syncing a light node `geth --syncmode light`, which only stores header blocks (trust that majority are validating transactions using full nodes)
- An archival node `geth --syncmode full --gcmode archive --txlookuplimit=0 --cache.preimages` (which is a full node but also keeps a snapshot of all intermediate, historical states at every block). In contrast to an archival node, a full node only stores recent state for faster initial sync. Once fully synced, it will store all state moving forward as in archival nodes. 

At this point, the data directory `~/.ethereum` should be filling up quickly, since it is where all the historical data is saved. Restarting the node will simply pick up where it left off. Using the default settings, syncing mainnet took about 3 hours using up ~500GB of the SSD. 
- In a separate terminal to the one running `geth` , running `geth attach` opens a Javascript console (by default it attaches using IPC, found in `~/.ethereum/geth.ipc` which exists only when the geth node is running). 
- To confirm when the syncing is done, you can compute `eth.syncing` within the Javascript console. If it returns `false`, syncing is finished.

### JSON-RPC Server
Geth supports the standard JSON-RPC API methods via multiple so-called transport protocols: IPC, HTTP, and Websocket. I only tried the first two. More details about Geth transport options [here](https://geth.ethereum.org/docs/rpc/server). 

- IPC provides unrestricted access to all method namespaces (eth, web3, net, etc.). Crucially, this only works when program is run locally on the same host as the Geth node
    - As a general rule, IPC is most secure because it is limited to interactions on the local machine and cannot be exposed to external traffic such as in HTTP
    - The listening socket for the IPC server is placed in the data directory at `~/.ethereum/geth.ipc`
- HTTP (RPC) is the most widely used transport for interacting with Geth. By default, it only provides access to `eth, web3, net` method namespaces, because enabling other APIs like `personal` for account management over HTTP increases the attack surface for external traffic and is therefore not recommended.
    - To open an HTTP server, run `geth --http` . To check that the HTTP port is open, in a separate terminal run`curl http://localhost:8545` If nothing returned then HTTP connection is open, otherwise if the connection is refused then HTTP is not running.
    - By default, the HTTP connection refers to `localhost` with listening port 8545. Only use default `http.addr=localhost` (otherwise opens access to external traffic)
    
## Uniswap Prices
Now that we have an Ethereum node, we can try pulling the most up-to-date data on the blockchain, such as prices on Uniswap. I followed the [Uniswap V2 SDK](https://docs.uniswap.org/sdk/2.0.0/guides/quick-start).
My Uniswap boilerplate code below can also be found [on Github](https://github.com/axucar/uniswap-boilerplate)

Uniswap's `fetchPairData` method allows us to specify an HTTP API using a URL, via the `ethers.js` module (see [ethers's JSON RPC documentation](https://docs.ethers.io/v5/api/providers/jsonrpc-provider/)). Below I tried 3 different methods (Alchemy API vs. IPC vs. HTTP), and noted a 10x speed difference between running on local node vs. using third-party HTTP API (Alchemy).
    
```js    
import { ChainId, Fetcher, WETH, Route, Trade, TokenAmount, TradeType, Token} from '@uniswap/sdk';
const chainId = ChainId.MAINNET
console.log(`The chainId of Mainnet is ${chainId}.`)
const dai_tokenAddress = '0x6B175474E89094C44Da98b954EedeAC495271d0F' // must be checksummed
const usdc_tokenAddress = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48' // must be checksummed

import ethers from 'ethers';

//METHOD 1: Alchemy, ~500-1000ms
// const url = 'https://eth-mainnet.g.alchemy.com/v2/<YOUR-API-KEY>';
// const customProvider = new ethers.providers.JsonRpcProvider(url);

//METHOD 2: IPC (LOCAL node machine only)
// const url = '/home/<user>/.ethereum/geth.ipc';
// const customProvider = new ethers.providers.IpcProvider(url);

//METHOD 3: HTTP, ~50-100ms
const url = 'http://localhost:8545'; 
const customProvider = new ethers.providers.JsonRpcProvider(url);

var startTime = performance.now();

const init = async () => {	
	const dai = new Token(ChainId.MAINNET, dai_tokenAddress, 18, 'DAI', 'Dai stablecoin');	
	const weth = WETH[dai.chainId];
	const weth_dai_pair = await Fetcher.fetchPairData(dai, WETH[dai.chainId], customProvider); 		
	const weth_dai_route = new Route([weth_dai_pair], weth); //input is weth		
	const weth_dai_trade = new Trade(weth_dai_route, new TokenAmount(weth, String(0.1*1e18)), TradeType.EXACT_INPUT);	

	console.log("Mid Price WETH --> DAI:", weth_dai_route.midPrice.toSignificant(6));	
	console.log("Execution Price WETH --> DAI:", weth_dai_trade.executionPrice.toSignificant(6));
	console.log("Mid Price after trade WETH --> DAI:", weth_dai_trade.nextMidPrice.toSignificant(6));	
	var endTime = performance.now();
	console.log("-".repeat(45));
	console.log(`Call to Uniswap took ${endTime - startTime} milliseconds`);
}
init();
```

To run the above, I installed Javascript via `nvm`. Then installed node.js version 16`nvm install 16` then `nvm use 16`. So ultimately `which node` for me points to `/home/<username>/.nvm/versions/node/v16.13.1/bin/node` . Also, the Uniswap script required `npm install @uniswap/sdk`, `npm install ethers`. Now, running `node test-uniswap-sdk.js` should give something like this:

```
The chainId of Mainnet is 1.
Mid Price WETH --> DAI: 1650.63
Execution Price WETH --> DAI: 1645.65
Mid Price after trade WETH --> DAI: 1650.56
---------------------------------------------
Call to Uniswap took 69.21298998594284 milliseconds
```

## Setting up SSH
I also have a laptop (running Ubuntu) from which I want to access the NUC’s files and ultimately the Geth node. In terms of accessing the NUC’s files, one can use `ssh` , see [OpenSSH on Ubuntu documentation](https://ubuntu.com/server/docs/service-openssh) which walks through installation of the OpenSSH client and server application. 

As first step, we need to get the IP address of the remote server (the NUC’s IP in this case). 

```bash
ip addr show|grep "inet "
```

For instance, an IP address may have the form 192.168.x.xxx. On client (laptop) I would enter `ssh nuc_username@192.168.x.xxx`, where the username is that of the server (NUC). After being prompted for the remote host's password, you can now access those remote files! You will notice terminal now says `nuc_username@server_name`. 

To do SSH without password: 
- generate ssh keys on my client (laptop) via `ssh-keygen -t rsa -b 4096` , which saves `~/.ssh/id_rsa.pub` (public key) and `~/.ssh/id_rsa` (private key) respectively. Then, we copy the public key to the remote host (in this case my NUC) via `ssh-copy-id nuc_username@192.168.x.xxx` (ie. nuc_username@remotehostIP)

- this will copy all the public keys in ~/.ssh/ into the remote host’s `~/.ssh/authorized_keys` file. Of course, this prompts for a password. When it’s done, no password will be required anymore. `exit` will log you out of the ssh’ed terminal

For other basic ssh commands on Ubuntu, see [this page](https://www.ssh.com/academy/ssh/command).

### SSH Tunneling (Local Port Forwarding)
While the SSH has been setup, this still requires me to have my Uniswap script on the NUC and run against the Geth node once SSH'ed into the remote host. However, say I want to run the Uniswap boilerplate [above](#uniswap-on-chain-pricing), but using the local environment on my laptop (e.g. boilerplate script only exists on my laptop). Notice that the HTTP JSON-RPC provider parameter `const url = 'http://localhost:8545'` when running the Uniswap script on the NUC. I can tunnel some port on the laptop (say port 3333) to the default Geth port on my NUC (port 8545), and query the Geth node as if the node was running on my laptop.

Local forwarding allows forwarding a port from a client machine (my laptop) to the server machine (NUC). 
See this [page](https://www.ssh.com/academy/ssh/tunneling/example) for an explanation.

```bash
ssh -L [LOCAL_IP:] LOCAL_PORT:DESTINATION:DESTINATION_PORT USER@SSH_SERVER
```

In my case, running the following command on my laptop opens a connection to the remote host (NUC) and forwards the connection from `localhost:8545` on the NUC to `localhost:3333` on my laptop.

```bash
ssh -L localhost:3333:localhost:8545 nuc_username@192.168.x.xxx`
```

Now, even though my laptop is not running Geth, I can connect to the NUC’s geth node via `localhost:3333`

```js
//Now I can connect to the NUC's geth node from my laptop!
const url = 'http://localhost:3333';
const customProvider = new ethers.providers.JsonRpcProvider(url);
```
