---
layout: post
title:  "Running local node (post-merge)"
date:   2022-10-05
categories: 
---
* TOC 
{:toc}
My original [post](http://axucar.ca/2022/08/03/running-geth-node/) on setting up a local node using Geth no longer works after the [Merge](https://geth.ethereum.org/docs/interface/merge). Node operators now need to run both a consensus client in addition to the execution client (such as Geth). This post describes how I setup Geth and Prysm (one of the 5 consensus clients) using a Docker Compose script. After experimenting with installation via Ubuntu PPA, I found this method to be more elegant.

## Approach

Most other blogs using Docker Compose scripts save the historical chain-data into a Docker volume, but I find it preferable to store the data in our user's home directory directly. This avoids the scenario where we might Docker-compose down to remove containers and images, and accidentally remove the volumes as well, and we’d have to wait many hours to re-sync to the latest chain. With bind-mounting, we ensure our data persists independently from the Docker up/down cycles.

More specifically, we bind mount `/home/user/.ethereum` (source) to `/root/.ethereum` (target) for Geth and `/home/user/.eth2` to `/data` for Prysm. This mapping matches the convention in the respective [Geth](https://geth.ethereum.org/docs/install-and-build/installing-geth#docker-container) and [Prysm](https://docs.prylabs.network/docs/install/install-with-docker) documentation.

## Docker Compose 

Assuming you’ve installed [Docker](https://docs.docker.com/engine/install/ubuntu/#installation-methods), you can verify it is running: `sudo systemctl status docker`. Next, the Geth and Prysm containers are defined in the following `docker-compose.yml`, which I keep inside `~/docker/geth`. Note that 

```python
version: "3.2"

services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    restart: unless-stopped
    ports:
      - "127.0.0.1:8545:8545"
      - "127.0.0.1:8546:8546"
      - "30303:30303"
    volumes:
      - type: bind
        source: /home/chihiro/.ethereum
        target: /root/.ethereum
    command:
      - --ws
      - --ws.addr=0.0.0.0
      - --http
      - --http.addr=0.0.0.0
      - --authrpc.addr=0.0.0.0
      - --authrpc.port=8551
      - --authrpc.vhosts=*
    stop_grace_period: 3m # to avoid unclean shutdown 

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain:stable
    container_name: prysm
    restart: unless-stopped
    ports:
      - "13000:13000/tcp"
      - "12000:12000/udp"
      - "127.0.0.1:4000:4000"
    volumes:
      - type: bind
        source: /home/chihiro/.eth2
        target: /data
      - type: bind
        source: /home/chihiro/.ethereum
        target: /root/.ethereum
        read_only: true

    command:
      - --mainnet
      - --datadir=/data
      - --accept-terms-of-use
      - --rpc-host=0.0.0.0
      - --grpc-gateway-host=0.0.0.0
      - --monitoring-host=0.0.0.0
      - --execution-endpoint=http://geth:8551
      - --jwt-secret=/root/.ethereum/geth/jwtsecret
      - --checkpoint-sync-url=https://beaconstate.info/
      - --p2p-host-ip=<public_IP_address> ##check yours using curl v4.ident.me
```

A couple “gotchas” that took me some time to sort out:

1. Use absolute paths for source and target when bind mounting in Docker compose.

	```python
		volumes:
		- type: bind
			source: /home/chihiro/.ethereum  ##NOT ~/.ethereum
			target: /root/.ethereum
	```
2. It was important to set `--http.addr` and `--authrpc.addr` to `0.0.0.0` to listen on all interfaces, instead of the default `localhost` since the two containers have different IP addresses (within Docker’s internal network) so the default settings of `localhost` would not match between Geth and Prysm. This tells each Docker container to "listen" for connections from outside of the respective containers. 
    
    Under the `ports:` section of the container definition, `13000:13000/tcp` means that it will expose (ie. forward) port `13000/tcp` of the docker container’s dedicated network (which is isolated from the host) to the host's `13000` port.
    
    Note that in the `geth` container, `"127.0.0.1:8545:8545"` is protecting us by only exposing the container’s port 8545 to the host’s `localhost`. To verify this, `curl localhost:8545` will not error, whereas `curl http://hostip:8545` from another machine will fail with a connection-refused error.
    
3. Similar to the point above, we set `-execution-endpoint=http://geth:8551` in the Prysm command, instead of the usual `http://localhost:8551` that would be expected if we were running both Geth and Prysm directly on the host machine (say via Ubuntu PPA installation). In Docker, each container has a different `localhost` , so we replace it with the `geth` container service name. Docker networking was quite difficult to understand for me, but [this explainer by NetworkGuru](https://www.youtube.com/watch?v=bKFMS5C4CG0&ab_channel=NetworkChuck) gave me enough of an understanding.

## Run Script
To create the containers: 
- `sudo docker-compose up -d` where `-d` is a silencer (Docker runs in background)

To monitor logs:
- `sudo docker-compose logs -f` , where `f` means keep following logs

To stop it:
- `sudo docker-compose stop`
    - Adding `-t 99999` as a flag allows us to specify long shutdown times when stopping. This supposedly avoids a lengthy process of data-cleaning next time you restart the node, though I did not find this to make a difference
    - I use `sudo docker-compose down` after stopping if I make changes to the yaml file, since it removes the containers
    - `sudo docker-compose start` resumes the clients

To see status of containers:
- `sudo docker-compose ps` shows what containers are running
```
chihiro@chihiro-box:~/docker/geth$ sudo docker-compose ps
Name               Command               State                                                          Ports                                                        
---------------------------------------------------------------------------------------------------------------------------------------------------------------------
geth    geth --ws --ws.addr=0.0.0. ...   Up      0.0.0.0:30303->30303/tcp,:::30303->30303/tcp, 30303/udp, 127.0.0.1:8545->8545/tcp, 127.0.0.1:8546->8546/tcp         
prysm   /app/cmd/beacon-chain/beac ...   Up      0.0.0.0:12000->12000/udp,:::12000->12000/udp, 0.0.0.0:13000->13000/tcp,:::13000->13000/tcp, 127.0.0.1:4000->4000/tcp
```

While syncing can take a while (mine took ~30 hours, with last chaindata around July). You can check the status of the sync using `geth`'s Javascript console:
`sudo docker-compose exec [container name] geth attach`
- `[container name]` is the name of the service in the Docker compose config, e.g. `prysm`, or `geth` in this case
- `eth.syncing` in the Javascript console should return `false` once synced

## Configure Firewall

I configured the firewall for the host machine by following [CoinCashew’s excellent guide](https://www.coincashew.com/coins/overview-eth/guide-or-how-to-setup-a-validator-on-eth2-mainnet/part-i-installation/guide-or-security-best-practices-for-a-eth2-validator-beaconchain-node#mandatory-configure-your-firewall)

```bash
# By default, deny all incoming and outgoing traffic
sudo ufw default deny incoming
sudo ufw default allow outgoing
# Allow ssh access
sudo ufw allow 22/tcp
# # Allow consensus client port
sudo ufw allow 13000/tcp
sudo ufw allow 12000/udp
# Allow execution client port
sudo ufw allow 30303
# Enable firewall
sudo ufw enable
```

You should now see the following for `sudo ufw status numbered`:

```
chihiro@chihiro-box:~/docker/geth$ sudo ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere                  
[ 2] 13000/tcp                  ALLOW IN    Anywhere                  
[ 3] 12000/udp                  ALLOW IN    Anywhere                  
[ 4] 30303                      ALLOW IN    Anywhere                  
[ 5] 22/tcp (v6)                ALLOW IN    Anywhere (v6)             
[ 6] 13000/tcp (v6)             ALLOW IN    Anywhere (v6)             
[ 7] 12000/udp (v6)             ALLOW IN    Anywhere (v6)             
[ 8] 30303 (v6)                 ALLOW IN    Anywhere (v6)
```

[Prysm docs](https://docs.prylabs.network/docs/prysm-usage/p2p-host-ip#configure-your-firewall) also provide a nice summary table for what the appropriate settings should look like, which you can use to double-check the firewall settings.

Separately, we also have to configure the **router** itself to do port forwarding. When an external computer pings a port (say `13000`), they only know your public IP address, so `{public_IP}:13000` has to somehow forward to `{private_IP_hostmachine}:13000`, which can only be done through the router's settings (see [these instructions](https://docs.prylabs.network/docs/prysm-usage/p2p-host-ip#configure-your-router)). It basically consists of logging into your router's browser-based admin interface (mine was something like `http://192.168.1.1`). Not forwarding ports through the ISP's firewall can affect how many peers your node is able to connect to. Take care to only forward the ports needed for p2p connectivity only: `13000, 12000 (Prysm), and 30303 (Geth).`

After configuring the router, you can check that the port forwarding has been done correctly by checking your public IP address and Port (say `13000` or `30303`) using tools like 
1. [yougetsignal.com](https://www.yougetsignal.com/tools/open-ports/)
2. [canyouseeme.org](https://canyouseeme.org/).

For debugging purposes, your public IP address can be found via `curl v4.ident.me`, while private IP address can be found via `ifconfig|grep "inet "|grep -v 127.0.0.1`.

## Autossh
Once I had my node synced on my NUC, I just needed to SSH tunnel the `localhost:8545` to my laptop. To keep an SSH tunnel (between host and my laptop) running constantly, I used `autossh`, which restarts whenever the tunnels shuts down. On my laptop, I run

`autossh -f -N -M 0 -L localhost:3333:localhost:8545 nuc_username@192.168.1.xxx`

where `192.168.1.xxx` is the private IP address of the NUC, running the Docker script above. Note that the `-N` switch was necessary for the `-f` switch to also work (which keeps the `autossh` process in the background).