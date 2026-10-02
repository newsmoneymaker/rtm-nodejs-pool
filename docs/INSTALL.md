# Installing a Raptoreum pool

Paths below are examples: the pool in `/opt/rtm-nodejs-pool`, the node and its data in `/opt/rtm`, all run by the user `rtmpool`
(the systemd templates in `deployment/systemd/` use these paths).

## 1. Raptoreum node and wallet

Raptoreum Core is a Dash based node: `https://github.com/Raptor3um/raptoreum` (MIT), tag `2.0.3.01-mainnet`. The releases page also ships Linux binaries (Ubuntu 20/22, built against a newer glibc than
Debian 10's); the pool's own GhostRider helper needs the static libraries of a from-source build anyway, so we build from source.
The source uses its own `depends` system (boost 1.77, OpenSSL, libevent, GMP for the dashbls/relic BLS library ...) and builds fine with GCC 12 with no source patches needed.
A host with an old glibc/GCC (Debian 10 has GCC 8) builds and runs the node in a newer userland, for example a Debian 12 chroot made with `debootstrap` (this pool reused the Debian 12 chroot already
set up for the FewBit/MateableCoin/Yenten nodes, at `/root/claude/ric/chroot`).

```
git clone --branch 2.0.3.01-mainnet https://github.com/Raptor3um/raptoreum.git && cd raptoreum
# build dependencies: curl build-essential libtool autotools-dev automake pkg-config python3 bsdmainutils patch bison cmake xz-utils unzip
make -C depends NO_QT=1 NO_UPNP=1 NO_NATPMP=1 -j4
./autogen.sh
./configure --prefix=$PWD/depends/x86_64-pc-linux-gnu --without-gui --disable-tests --disable-bench --disable-man
make -j4                                                    # src/raptoreumd, src/raptoreum-cli
```

`/opt/rtm/data/raptoreum.conf` (mainnet default ports from `src/chainparamsbase.cpp`/`chainparams.cpp`: RPC 10225, P2P 10226):

```
server=1
listen=1
daemon=0
rpcuser=rtmpool
rpcpassword=<a long random password>
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
rpcport=10225
port=10226
maxconnections=40
dbcache=450
txindex=1
addnode=51.89.21.112
addnode=lbdn.raptoreum.com
```

**Use the project's bootstrap/powcache for a new node.** Validating the chain from genesis takes a long time. `https://bootstrap.raptoreum.com` carries a `Bootstrap.tar.gz` of `blocks/`, `chainstate/`,
`evodb/`, `llmq/`, `powcache.dat`; on raptoreumd 1.3.17.0+ the lighter `powcache.dat` alone (checksums at `https://checksums.raptoreum.com`) lets the node fetch the rest from peers, about 1-2 hours.
Stop the node, empty the data directory (keep `raptoreum.conf`), unpack it there and start the node: it catches up by itself (`raptoreum-cli getblockchaininfo`). Smartnode sync (`mnsync status`) finishes
a bit later; block templates are served as soon as the blockchain is synced (IsBlockchainSynced).

The default wallet of the node is the pool wallet:

```
raptoreum-cli getnewaddress "pool"          # R... (or 7... for a P2SH address): the pool address (poolServer.poolAddress)
raptoreum-cli dumpwallet /safe/place/rtm-wallet-dump.txt   # the private keys: keep it offline, chmod 600 (also: raptoreum-cli backupwallet <file>)
```

The wallet is a plain (non HD) wallet with a key pool: back it up again after new addresses were handed out.

## 2. GhostRider helper

The helper is a tiny program around the node's own `HashGR`; it is linked against the static libraries of the node build (so the hash is exactly the node's).

```
cd /opt/rtm-nodejs-pool/hasher && make RTM=/path/to/raptoreum            # the build tree from step 1
# a static binary that runs on any Linux (build it where the node is built): make RTM=... STATIC=-static
```

Check it against the chain: `node test/test-real-blocks.js`. A hash takes about 15 ms; `hasher.threads` helper processes work in parallel (each share is one hash).

## 3. Redis

Use a dedicated instance with a password and AOF (`deployment/redis-pool.conf.example`, unit `rtm-pool-redis`, port 6396).

## 4. The pool

```
cd /opt/rtm-nodejs-pool && npm install --production
cp config_examples/rtm.json config.json      # then edit it
```

Edit `config.json`: `poolHost`, `poolServer.poolAddress` (the wallet address of step 1), the ports and the certificate for TLS (`poolServer.sslCert/sslKey`), `redis`, `api.password`,
`node.password` or `node.passwordFile`, `blockUnlocker.poolFee` and `donations`, `payments`. **Keep `payments.dryRun: true` until the rehearsal below.**

```
cp deployment/systemd/*.service /etc/systemd/system/ && systemctl daemon-reload
systemctl enable --now rtm-pool-redis rtm-node
systemctl enable --now rtm-pool rtm-pool-api rtm-pool-unlocker rtm-pool-payments rtm-pool-charts
node test/test-rtm-proposal.js config.json        # the node itself validates a block built by the pool (needs the synchronised node)
```

The pool runs as separate modules (`init.js -module=pool|api|unlocker|payments|chartsDataCollector`), each in its own unit. Until payouts are proven, restrict the stratum ports with `poolServer.allowIPs`.
Point a miner at it: `poolpayminer -a gr -o your.pool:3800 -u <an R... address> -p x`.

## 5. Website

Copy `website_example/` to the web root, set `poolHost`, the contact and links in `config.js`, and proxy `/api` to the pool API on 127.0.0.1:8124 (`deployment/apache-vhost.conf.example` exposes only the
read-only methods).

## 6. Rehearse the payments

1. `payments.dryRun: true`: the log of `rtm-pool-payments` shows what would be paid.
2. Fund the pool wallet with a few coins (or wait for the first block), credit a small balance in Redis to your own test addresses (`<coin>:workers:<address>`, field `balance`), set `payments.onlyAccounts`
   to them, `dryRun: false`, and watch the payout confirm.
3. Remove the test accounts from Redis and set `onlyAccounts` to `[]`.

Good to know: block rewards can be spent after 100 blocks (COINBASE_MATURITY, about 3.3 hours at the 2 minute block time); `deployment/pause-payments.sh` stops new payouts at once; the wallet must stay
unlocked and online for the payouts (leave the pool wallet unencrypted with only small balances in it). Of a block the pool only gets what is left after the smartnode and founder (5% forever) payments
the template demands; the subsidy itself steps down in irregular increments every 21262 blocks (see `GetBlockSubsidy` in `src/validation.cpp`), not a simple halving.
