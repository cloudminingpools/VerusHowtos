# S-NOMP for VerusCoin Setup

## Server

A VPS with 4GB of RAM, anything above 20GB SSD storage and 2 CPU cores is the absolute minimum requirement.


## Operating System

This guide is tested on Debian 9 "Stretch". Before starting, please install the latest updates.

```
apt update
apt -y upgrade
```


## Poolwallet

The packages required in order to compile a VerusCoin wallet can be installed like this:

```
apt -y install build-essential git pkg-config libc6-dev m4 g++-multilib autoconf \
                   libtool ncurses-dev unzip git python python-zmq zlib1g-dev wget \
                   libcurl4-openssl-dev bsdmainutils automake curl
```


Create a useraccount for the wallet. Switch to that account.

```
useradd -m -d /home/veruscoin -s /bin/bash veruscoin
su - veruscoin
```

Now, clone the source tree and build the binaries:

```
git clone https://github.com/VerusCoin/VerusCoin
cd VerusCoin
./zcutil/fetch-params.sh
./zcutil/build.sh -j$(nproc)
```

After that is done, create a `~/bin` directory and copy over the binaries. Strip the debug symbols.

```
mkdir ~/bin
cp src/komodod src/komodo-cli src/komodo-tx ~/bin
strip ~/bin/komodo*
```

Start the VerusCoin daemon so we have a default configuration file:

```
komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 \
-ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 \
-ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 \
-addnode=185.64.105.111 -daemon 1>/dev/null 2>&1
```

Let it run for a few seconds and stop it again: 

```
komodo-cli -ac_name=VRSC stop
```

In the resulting `~/.komodo/VRSC/VRSC.conf`, add `listen=1`. Then start the daemon again and let it sync the blockchain: 

```
komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 \
-ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 \
-ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 \
-addnode=185.64.105.111 -daemon
```

Because this time `STDOUT` and `STDERR` are not redirected, it will produce a lot of output on the console, but will tell you at a glance wether or not the daemon has started syncing the blockchain successfully. Either open a new shell or exit the current one to regain silence. To check the status and know when the initial sync has been completed, issue

```
komodo-cli -ac_name=VRSC getinfo
```

When it has synced up to height, the `blocks` and `longestchain` values will be at par. While we wait for this to happen, lets continue.

## Redis 

Switch back to the `root` account. Install Redis using `apt -y install redis-server`. In your `/etc/redis/redis.conf` file, make sure it contains this:

```
bind 127.0.0.1
appendonly yes
```

Set `redis-server` to start at bootup and start it manually: 

```
update-rc.d redis-server enable
/etc/init.d/redis start
```

## Node.js

Still as `root`, install Node.js v8 like this: 

```
curl -sL https://deb.nodesource.com/setup_8.x | bash -
apt -y install nodejs

```

We will use [PM2](http://pm2.keymetrics.io) to manage NodeJS processes. Install it globally:

```
npm -g install pm2
```

## S-NOMP

S-NOMP and some of its dependencies will need additional packages in order to be built successfully: 

```
apt -y install libboost-all-dev libsodium-dev
```

Create a new user account to run S-NOMP from. Switch to that user and clone S-NOMP from hellcatz repository: 

```
useradd -m -d /home/s-nomp -s /bin/bash s-nomp
su - s-nomp
git clone https://github.com/hellcatz/s-nomp
```

In `package.json`, change the `stratum-pool` dependency to `git+https://github.com/hellcatz/node-stratum-pool-1.git`. Next, install all dependencies using `npm`: 

```
npm update
npm install
```

Per default, S-NOMP will display `Sol` as the hashrate measuring unit. VerusCoin uses `H`, so lets change it:

```
cd ~/s-nomp
perl -p -i -e 's/Sol\/s/H\/s/g' libs/stats.js website/pages/stats.html website/static/stats.js website/static/miner_stats.js
```

In the web dashboard of the pool there is a 'Pool Luck' display which gives a rough estimate of how much time will pass between found blocks. To improve the accuracy of this number, change the value of `_blocktime` in line `584` of `libs/stats.js` to be closer to the actual block target of VerusCoin: 

```
cd ~/s-nomp
perl -p -i -e 's/_blocktime=160/_blocktime=55/g' libs/stats.js
```

_Here be dragons, doges and ALL \o/ the code changes needed to get the pool fee working._

 
## Configuration Instructions

Shielding is required for mined VerusCoins. We will need 2 public and a z-address for this. Switch to the `veruscoin` user and generate the addresses:

```
komodo-cli -ac_name=VRSC getnewaddress
komodo-cli -ac_name=VRSC getnewaddress
komodo-cli -ac_name=VRSC z_getnewaddress
```

Next, we will dump the private keys of these addresses for safety reasons. For the public addresses, use 

```
komodo-cli -ac_name=VRSC dumpprivkey <public VerusCoin address>
```

For the z-address, use

```
komodo-cli -ac_name=VRSC z_exportkey <VerusCoin z-address>
```

**Save the data in an offline location, not on your computer!**

Now, switch to the `s-nomp` account. First, copy `~/s-nomp/config_example.json` to `~/s-nomp/config.json` and in it, change the value of `stratumHost` to the external IP or DNS name of your server.

Now create a pool config. Copy `~/s-nomp/pool_configs/examples/kmd.json` to `~/s-nomp/pool_configs/vrsc.json`. Edit it to reflect the changes listed below. 

 * Set `enabled` to `true`.
 * Set `coin` to `vrsc.json`.
 * Set `address` to one of the public addresses generated before.
 * Set `zAddress` to the z-address generated before.
 * Use the remaining public address for `tAddress`
 * Set `paymentInterval` to `180`
 * Set `minimumPayment` to `2`.
 * Set `maxBlocksPerPayment` to `8`.
 * There are 2 occurences of `user`, `password` and `port` each. Use the `rpcuser`, `rpcpassword` and `rpcport` values from `/home/veruscoin/.komodo/VRSC/VRSC.conf`.
 * Set `diff` to `131072`.
 * Set `minDiff` to `16384`.
 * Set `maxDiff` to `2147483648`

We are almost done now. Using the command mentioned at the beginning of this document, check if the blockchain has finished syncing. If not, wait for it to complete before continuing. 

Now switch to the `veruscoin` user, stop the wallet once more. 

```
komodo-cli -ac_name=VRSC stop
```

Edit `~/.komodo/VRSC/VRSC.conf` and add the blocknotify command below.

```
blocknotify=/usr/bin/node /home/s-nomp/s-nomp/scripts/cli.js blocknotify verus %s
```

Restart the wallet using the command already listed above. If you are not using `STDOUT`/`STDERR`-redirection, you will see errors about blocknotify. These are expected, because the pool is not running yet and thus the blocknotify script cannot complete successfully. 


Switch to the `s-nomp` user. Then start the pool using `pm2`: 

```
cd ~/s-nomp
pm2 start init.js --name s-nomp
```

Use `pm2 log` to check for S-NOMP startup errors. 

If you completed all steps correctly, the web dashboard on your pool can be reached via port `8080` on the external IP or the DNS name of your server.
