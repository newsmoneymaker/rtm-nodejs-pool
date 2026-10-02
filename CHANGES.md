# Changes

## 1.0.0

* First release: Raptoreum (RTM, a Raptoreum/Dash based node with GhostRider) pool derived from scash-nodejs-pool, veil-nodejs-pool, qwc-nodejs-pool and epic-nodejs-pool.
* The pool builds the blocks itself from `getblocktemplate` (`lib/blockBuilder.js`): a version 3 / type 5 coinbase (BIP34 height, extranonce, the pool's output, the smartnode / superblock / founder payments
  exactly as the template demands, the node's `coinbase_payload`), the merkle branch and the 80 byte header; a solved block is sent with `submitblock`.
* `lib/pool.js` speaks Bitcoin Stratum v1 as GhostRider miners parse it (prevhash with every 4 byte word reversed, version / ntime / nbits big endian, the nonce as the hex of the number, extranonce1 of
  4 bytes and extranonce2 of 4 bytes). Job ids are per miner (`<job>.<n>`) and carry the difficulty of that miner: the miner applies a new difficulty with its next job.
* `hasher/grhash` calls the node's own `HashGR`; it is linked against the built Raptoreum Core tree (`make -C hasher RAPTOREUM=...`).
* Payments and unlocker as in Bitcoin based pools (`sendmany`, `gettransaction`, coinbase maturity 100), base58 addresses (`R...`, `7...`).
* The variable difficulty of a worker is remembered across reconnects (`poolServer.diffMemoryMinutes`, default 15).
* Tests: address handling, the GhostRider hash of real mainnet blocks, and a block built by the pool from a real node's template validated by that node as a proposal.
* Not yet checked on a block found on mainnet: the payout with the real wallet (`sendmany` with `subtractfeefrom`).
