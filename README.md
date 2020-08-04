# *Reddsight API*

*Reddsight API* is an open-source Reddcoin blockchain REST
and websocket API. Reddsight API runs in NodeJS and uses LevelDB for storage. 

This is a backend-only service. If you're looking for a web frontend application,
take a look at our official blockchain explorer [Reddsight](https://github.com/reddcoin-project/reddsight).

*Reddsight API* allows everyone to develop Reddcoin-related applications (such as wallets) that 
require certain information from the blockchain that reddcoind does not provide.


## Prerequisites

* **reddcoind** - Download and install [Reddcoin](https://github.com/reddcoin-project/reddcoin).

*Reddsight API* needs a *trusted* reddcoind node to run. *Reddsight API* will connect to the node
through the RPC API, Reddcoin peer-to-peer protocol, and will even read its raw block .dat files for syncing.

Configure reddcoind to listen to RPC calls and set `txindex` to true. reddcoind must be running and must have
finished downloading the blockchain **before** running *Reddsight API*.

* **Node.js v0.10.x** - Download and Install [Node.js](http://www.nodejs.org/download/).

* **NPM** - Node.js package manager, should be automatically installed when you get Node.js.


## Quick Install
  Check the Prerequisites section above before installing.

  To install *Reddsight API*, clone the main repository:

    $ git clone https://github.com/reddcoin-project/reddsight-api.git && cd reddsight-api

  Install dependencies:

    $ npm install

  Run the main application:

    $ node insight.js

  Then open a browser and go to:

    http://localhost:3000

  Please note that the app will need to sync its internal database
  with the blockchain state, which may take some time. You can check
  sync progress at http://localhost:3000/api/sync.


## Configuration

All configuration is specified in the [config](config/) folder, particularly the [config.js](config/config.js) file.
There you can specify your application name and database name. Certain configuration values are pulled from environment
variables if they are defined:

```
BITCOIND_HOST         # RPC reddcoind host
BITCOIND_PORT         # RPC reddcoind Port
BITCOIND_P2P_HOST     # P2P reddcoind Host (will default to BITCOIND_HOST, if specified)
BITCOIND_P2P_PORT     # P2P reddcoind Port
BITCOIND_USER         # RPC username
BITCOIND_PASS         # RPC password
BITCOIND_DATADIR      # reddcoind datadir. 'testnet' will be appended automatically if testnet is used. NEED to finish with '/'. e.g: `/vol/data/`
INSIGHT_NETWORK [= 'livenet' | 'testnet']
INSIGHT_PORT          # insight api port
INSIGHT_DB            # Path where to store the internal DB. (defaults to $HOME/.reddsight)
INSIGHT_SAFE_CONFIRMATIONS=6  # Nr. of confirmation needed to start caching transaction information   
INSIGHT_IGNORE_CACHE  # True to ignore cache of spents in transaction, with more than INSIGHT_SAFE_CONFIRMATIONS confirmations. This is useful for tracking double spents for old transactions.
ENABLE_MAILBOX # if "true" will enable mailbox plugin
ENABLE_CLEANER # if "true" will enable message db cleaner plugin
ENABLE_MONITOR # if "true" will enable message db monitor plugin
ENABLE_EMAILSTORE # if "true" will enable a plugin to store data with a validated email address
ENABLE_RATELIMITER # if "true" will enable the ratelimiter plugin
LOGGER_LEVEL # defaults to 'info', can be 'debug','verbose','error', etc.
ENABLE_HTTPS # if "true" it will server using SSL/HTTPS

```

Make sure that reddcoind is configured to [accept incoming connections using 'rpcallowip'](https://en.bitcoin.it/wiki/Running_Bitcoin).

In case the network is changed (testnet to livenet or vice versa) levelDB database needs to be deleted. This can be performed running:
```util/sync.js -D``` and waiting for *Reddsight API* to synchronize again.  Once the database is deleted,
the sync.js process can be safely interrupted (CTRL+C) and continued from the synchronization process embedded in main app.


## Synchronization

The initial synchronization process scans the blockchain from the paired reddcoind server to update addresses and balances.
*reddsight-api* needs exactly one trusted reddcoind node to run. This node must have finished downloading the blockchain
before running *reddsight-api*.

While *reddsight-api* is synchronizing the website can be accessed (the sync process is embedded in the webserver),
but there may be missing data or incorrect balances for addresses. The 'sync' status is shown at the `/api/sync` endpoint.

The blockchain can be read from reddcoind's raw `.dat` files or RPC interface. 
Reading the information from the `.dat` files is much faster so it's the
recommended (and default) alternative. `.dat` files are scanned in the default
location for each platform (for example, `~/.reddcoin` on Linux). In case a
non-standard location is used, it needs to be defined (see the Configuration section).

While synchronizing the blockchain, *reddsight-api* listens for new blocks and
transactions relayed by the reddcoind node. Those are also stored on *reddsight-api*'s database.
In case *reddsight-api* is shutdown for a period of time, restarting it will trigger
a partial (historic) synchronization of the blockchain. Depending on the size of
that synchronization task, a reverse RPC or forward `.dat` syncing strategy will be used.

If reddcoind is shutdown, *reddsight-api* needs to be stopped and restarted
once reddcoind is restarted.


### Syncing old blockchain data manually

  Old blockchain data can be manually synced issuing:

    $ util/sync.js

  Check util/sync.js --help for options, particularly -D to erase the current DB.

  *NOTE*: there is no need to run this manually since the historic synchronization
  is built in into the web application. Running *reddsight-api* normally will trigger
  the historic sync automatically.


### DB storage requirement

To store the blockchain and address related information, *reddsight-api* uses LevelDB.
Two DBs are created: txs and blocks. By default these are stored on

  ``~/.reddsight/``

This can be changed at config/config.js.


## Development

To run *reddsight-api* locally for development with grunt:

```$ NODE_ENV=development grunt```

To run the tests

```$ grunt test```


Contributions and suggestions are welcome at [reddsight-api github repository](https://github.com/reddcoin-project/reddsight-api).

## Caching schema

Since v0.2 a new cache schema has been introduced. Only information from transactions with
INSIGHT_SAFE_CONFIRMATIONS settings will be cached (by default SAFE_CONFIRMATIONS=6). There 
are 3 different caches:
 * Number of confirmations 
 * Transaction output spent/unspent status
 * scriptPubKey for unspent transactions

Cache data is only populated on request, i.e., only after accessing the required data for
the first time, the information is cached, there is not pre-caching procedure.  To ignore 
cache by default, use INSIGHT_IGNORE_CACHE. Also, address related calls support `?noCache=1`
to ignore the cache in a particular API request.

## API

By default, *reddsight-api* provides a REST API at `/api`, but this prefix is configurable from the var `apiPrefix` in the `config.js` file.

The end-points are:


### Block
```
  /api/block/[:hash]
  /api/block/00000000a967199a2fad0877433c93df785a8d8ce062e5f9b451cd1397bdbf62
```
### Transaction
```
  /api/tx/[:txid]
  /api/tx/525de308971eabd941b139f46c7198b5af9479325c2395db7f2fb5ae8562556c
```
### Address
```
  /api/addr/[:addr][?noTxList=1&noCache=1]
  /api/addr/mmvP3mTe53qxHdPqXEvdu8WdC7GfQ2vmx5?noTxList=1
```
### Address Properties
```
  /api/addr/[:addr]/balance
  /api/addr/[:addr]/totalReceived
  /api/addr/[:addr]/totalSent
  /api/addr/[:addr]/unconfirmedBalance
```
The response contains the value in Satoshis.
### Unspent Outputs
```
  /api/addr/[:addr]/utxo[?noCache=1]
```
Sample return:
``` json
[
    {
      address: "n2PuaAguxZqLddRbTnAoAuwKYgN2w2hZk7",
      txid: "dbfdc2a0d22a8282c4e7be0452d595695f3a39173bed4f48e590877382b112fc",
      vout: 0,
      ts: 1401276201,
      scriptPubKey: "76a914e50575162795cd77366fb80d728e3216bd52deac88ac",
      amount: 0.001,
      confirmations: 3
    },
    {
      address: "n2PuaAguxZqLddRbTnAoAuwKYgN2w2hZk7",
      txid: "e2b82af55d64f12fd0dd075d0922ee7d6a300f58fe60a23cbb5831b31d1d58b4",
      vout: 0,
      ts: 1401226410,
      scriptPubKey: "76a914e50575162795cd77366fb80d728e3216bd52deac88ac",
      amount: 0.001,
      confirmation: 6    
      confirmationsFromCache: true,
    }
]
```
Please note that in case confirmations are cached (which happens by default when the number of confirmations is bigger
that INSIGHT_SAFE_CONFIRMATIONS) the response will include the pair confirmationsFromCache:true, and confirmations will
equal INSIGHT_SAFE_CONFIRMATIONS. See noCache and INSIGHT_IGNORE_CACHE options for details.


### Unspent Outputs for multiple addresses
GET method:
```
  /api/addrs/[:addrs]/utxo
  /api/addrs/2NF2baYuJAkCKo5onjUKEPdARQkZ6SYyKd5,2NAre8sX2povnjy4aeiHKeEh97Qhn97tB1f/utxo
```

POST method:
```
  /api/addrs/utxo
```

POST params:
```
addrs: 2NF2baYuJAkCKo5onjUKEPdARQkZ6SYyKd5,2NAre8sX2povnjy4aeiHKeEh97Qhn97tB1f
```

### Transactions by Block
```
  /api/txs/?block=HASH
  /api/txs/?block=00000000fa6cf7367e50ad14eb0ca4737131f256fc4c5841fd3c3f140140e6b6
```
### Transactions by Address
```
  /api/txs/?address=ADDR
  /api/txs/?address=mmhmMNfBiZZ37g1tgg2t8DDbNoEdqKVxAL
```
### Transaction broadcasting
POST method:
```
  /api/tx/send
```
POST params:
```
  rawtx: "signed transaction as hex string"

  eg

  rawtx: 01000000017b1eabe0209b1fe794124575ef807057c77ada2138ae4fa8d6c4de0398a14f3f00000000494830450221008949f0cb400094ad2b5eb399d59d01c14d73d8fe6e96df1a7150deb388ab8935022079656090d7f6bac4c9a94e0aad311a4268e082a725f8aeae0573fb12ff866a5f01ffffffff01f0ca052a010000001976a914cbc20a7664f2f69e5355aa427045bc15e7c6c77288ac00000000

```
POST response:
```
  {
      txid: [:txid]
  }

  eg

  {
      txid: "c7736a0a0046d5a8cc61c8c3c2821d4d7517f5de2bc66a966011aaa79965ffba"
  }
```

### Historic blockchain data sync status
```
  /api/sync
```

### Live network p2p data sync status
```
  /api/peer
```

### Status of the Reddcoin network
```
  /api/status?q=xxx
```

Where "xxx" can be:

 * getInfo
 * getStakingInfo
 * getDifficulty
 * getTxOutSetInfo
 * getBestBlockHash
 * getLastBlockHash

## Web Socket API
The web socket API is served using [socket.io](http://socket.io).

The following are the events published by *Reddsight API*:

'tx': new transaction received from network. This event is published in the 'inv' room. Data will be a app/models/Transaction object.
Sample output:
```
{
  "txid":"00c1b1acb310b87085c7deaaeba478cef5dc9519fab87a4d943ecbb39bd5b053",
  "processed":false
  ...
}
```


'block': new block received from network. This event is published in the 'inv' room. Data will be a app/models/Block object.
Sample output:
```
{
  "hash":"000000004a3d187c430cd6a5e988aca3b19e1f1d1727a50dead6c8ac26899b96",
  "time":1389789343,
  ...
}
```

'<reddcoinAddress>': new transaction concerning <reddcoinAddress> received from network. This event is published in the '<reddcoinAddress>' room.

'status': every 1% increment on the sync task, this event will be triggered. This event is published in the 'sync' room.

Sample output:
```
{
  blocksToSync: 164141,
  syncedBlocks: 475,
  upToExisting: true,
  scanningBackward: true,
  isEndGenesis: true,
  end: "000000000933ea01ad0ee984209779baaec3ced90fa3f408719526f8d77f4943",
  isStartGenesis: false,
  start: "000000009f929800556a8f3cfdbe57c187f2f679e351b12f7011bfc276c41b6d"
}
```

### Example Usage

The following html page connects to the socket.io Reddsight API and listens for new transactions.

html
```
<html>
<body>
  <script src="http://live.reddcoin.com/socket.io/socket.io.js"></script>
  <script>
    eventToListenTo = 'tx'
    room = 'inv'

    var socket = io("http://live.reddcoin.com/");
    socket.on('connect', function() {
      // Join the room.
      socket.emit('subscribe', room);
    })
    socket.on(eventToListenTo, function(data) {
      console.log("New transaction received: " + data.txid)
    })
  </script>
</body>
</html>
```

## License
(The MIT License)

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
