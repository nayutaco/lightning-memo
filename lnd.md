# Table of Contents

  * [site](#site)
  * [update application](#update-application)
  * [設定](#設定)
    * [~/.lnd/lnd.conf](#lndlndconf)
    * [~/.btcctl/btcctl.conf](#btcctlbtcctlconf)
    * [~/.lncli/lncli.conf](#lnclilncliconf)
  * [起動](#起動)
    * [btcd](#btcd)
    * [lnd](#lnd)
      * [基本](#基本)
      * [ログレベル指定](#ログレベル指定)
      * [announcementするIPアドレス指定](#announcementするipアドレス指定)
    * [lncli](#lncli)
      * [create / unlock](#create--unlock)
      * [状態確認](#状態確認)
      * [入金](#入金)
      * [相手ノードへの接続](#相手ノードへの接続)
      * [チャネル開設](#チャネル開設)
      * [チャネル開設状態](#チャネル開設状態)
      * [invoiceの作成](#invoiceの作成)
      * [invoiceのデコード](#invoiceのデコード)
      * [送金](#送金)
      * [チャネル閉鎖](#チャネル閉鎖)
  * [lncli --help](#lncli---help)
  * [デバッグ](#デバッグ)
    * [log](#log)
      * [見方](#見方)

# site

- [lightningnetwork/lnd](https://github.com/lightningnetwork/lnd)

# update application

```bash
cd $GOPATH/src/github.com/lightningnetwork/lnd
git pull
make && make install
```

## update golang

golangのバージョンは[docs](https://github.com/lightningnetwork/lnd/blob/master/docs/INSTALL.md#preliminaries)を参照。

```bash
# ubuntu 18.04

# install
sudo snap install --channel=1.13/stable --classic go

# update
sudo snap refresh --channel=1.13/stable --classic go

```

# 設定

## ~/.lnd/lnd.conf

- [サンプル](https://github.com/lightningnetwork/lnd/blob/master/sample-lnd.conf)
- [`nobootstrap=1`](https://github.com/lightningnetwork/lnd/blob/3036fc01817eedd1821c2945e04c0e79b69a8453/sample-lnd.conf#L96-L99) すると、自動的に他のノードを検索しにいかなくなる
  - `lncli listpeers` で何も出てこないので、そうなのだろう
  - `lnd`がどうやってノードを見つけているのかはわかっていない。mDNSか？
- `no-macaroons`をtrueにした場合、`lncli`にも`-no-macaroons`がいる。
  - macaroonsを使う場合、mainnet以外なら`lncli`には`--network=xxxx`で種類を指定する必要があるようだ。

### 私が使っている設定(bitcoind版)

```text
[Application Options]
debuglevel=trace
debughtlc=true
no-macaroons=true
nobootstrap=1

[Bitcoin]
bitcoin.node=bitcoind
bitcoin.testnet=1
; bitcoin.regtest=1
bitcoin.active=1
```

~/.bitcoin/bitcoin.conf
```text
rpcuser=bitcoinuser
rpcpassword=bitcoinpassword
server=1
txindex=1
testnet=1
#regtest=1
#rpcport=18443
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
```

### 私が使っている設定(btcd版)

```text:~/.lnd/lnd.conf
[Application Options]
debughtlc=true
maxpendingchannels=10
no-macaroons=true
debuglevel=trace

[Bitcoin]
bitcoin.active=1
bitcoin.testnet=1
bitcoin.node=btcd
```

~/.btcd/btcd.conf

```text:~/.btcd/btcd.conf
testnet=1
txindex=1
rpcuser=nayuta
rpcpass=nayuta
```

## ~/.btcctl/btcctl.conf

- 私が使っている設定
  - rpcuserなどはbtcd.confにあわせる

```text
rpcuser=nayuta
rpcpass=nayuta
```

## ~/.lncli/lncli.conf

- そういう設定ファイルは、ありそうで、無い

# 起動

## btcd

- 起動直後はブロックの読み込みに時間がかかっているので、`btcctl`は応答できないかもしれない。待つ。

```bash
btcd > /dev/null 2>&1 &
btcctl --testnet getblockcount
```

## lnd

### 基本

- `lnd.conf`の内容が使われるので、よく使うものはそちらに書いた方が楽。

```bash
lnd
```

### ログレベル指定
  - `trace, debug, info, warn, error, critical`

```bash
lnd --debuglevel=warn
```

### announcementするIPアドレス指定

- 指定しない場合に`node_announcement`のIPアドレスが未指定になるかどうかは未確認

```bash
lnd --externalip=<IP address>
```

## lncli

- 面倒であれば、`.bashrc` に `alias lncli='lncli --no-macaroons'`などとしておくのがよいだろう

### create / unlock

- 新規の場合は `create`
  - `~/.lnd/data` がデフォルトのデータディレクトリなので、新規にしたい場合はこれを削除するとよい

```bash
lncli --no-macaroons create
```

- 新規でない場合は `unlock`

```bash
lncli --no-macaroons unlock
```

### 状態確認

- `getinfo`

```bash
lncli --no-macaroons getinfo
```

- `listpeers`

```bash
lncli --no-macaroons listpeers
```

- ` listchannels`

```bash
lncli --no-macaroons listchannels
```

### 入金

- walletにbalanceがない場合は、`lncli`でアドレスを取得し、faucetなどから入金してもらう
  - 私は[このfaucet](https://testnet.manu.backend.hamburg/faucet)をよく使っている

```bash
lncli --no-macaroons newaddress np2wkh
```

- balanceの確認
  `confirmed_balance` に値が入るまで待つ。

```bash
lncli --no-macaroons walletbalance
```

### 相手ノードへの接続

- [参考](http://dev.lightning.community/tutorial/01-lncli/index.html)
- `<port>`は9735以外の場合に指定する
- `lnd`は自動的にpeerを見つけて接続しに行くようだから、`listpeers`で接続されるのを待った方が良いのかもしれない。

```bash
lncli --no-macaroons connect <node_id>@<IP address>[:<port>]
```

### チャネル開設

- `--local_amt`の単位はsatoshis

```bash
lncli --no-macaroons openchannel --node_key=<node_id> --local_amt <amt_satoshi>
```

### チャネル開設状態

- 状態の調べ方がよくわからないが、開設できれば`listchannels`に出てくる。

```bash
lncli --no-macaroons listchannels
```

### invoiceの作成

- Lightning Networkで送金してもらうために、まずinvoiceを作る
  - `--amt`の単位はsatoshi
  - 成功すると `"pay_req"`に`ln`で始まる文字列が返ってくる。

```bash
lncli --no-macaroons addinvoice --amt 100
```

### invoiceのデコード

```bash
lncli --no-macaroons decodepayreq <invoice>
```

デコード例

```text
$ lncli --no-macaroons decodepayreq lntb17u1pd8dmetpp5hjewt9hzeywgzj3q6373ncsv
zc8reqckl76zfzku7tmu7w5d4wzsdpzxysy2umswfjhxum0yppk76twypgxzmnwvypfqvr3cegc6qkck
9h58yrt56yrfynexpaqhtwujc6s3v5el3g7yrgsy9anxldskmqwwetw32teql70u8tvgc47rkelvtjgz
mal7x3xcpfwhnf3
{
    "destination": "035b55e3e08538afeef6ff9804e3830293eec1c4a6a9570f1e96a478dad1c86fed",
    "payment_hash": "bcb2e596e2c91c814a20d47d19e20c160e3c8316ffb4248adcf2f7cf3a8dab85",
    "num_satoshis": "1700",
    "timestamp": "1517743915",
    "expiry": "3600",
    "description": "1 Espresso Coin Panna",
    "description_hash": "",
    "fallback_addr": "",
    "cltv_expiry": "9"
}
```

### 送金

```bash
lncli --no-macaroons payinvoice <BOLT11 invoice>
```

### チャネル閉鎖

* <funding_txindex>を指定しないと `[lncli] rpc error: code = Unknown desc = unable to find channel` となることがある
  * txindexが0の場合は省略可能なだけか？

* mutual close

```bash
lncli --no-macaroons closechannel --funding_txid <funding_txid> [--output_index <funding_txindex>]
```

* unilateral close

```bash
lncli --no-macaroons closechannel --force --funding_txid <funding_txid> [--output_index <funding_txindex>]
```

# lncli --help

```
$ lncli --help
NAME:
   lncli - control plane for your Lightning Network Daemon (lnd)

USAGE:
   lncli [global options] command [command options] [arguments...]

VERSION:
   0.3

COMMANDS:
     create            used to set the wallet password at lnd startup
     unlock            unlock encrypted wallet at lnd startup
     newaddress        generates a new address.
     sendmany          send bitcoin on-chain to multiple addresses.
     sendcoins         send bitcoin on-chain to an address
     connect           connect to a remote lnd peer
     disconnect        disconnect a remote lnd peer identified by public key
     openchannel       Open a channel to an existing peer.
     closechannel      Close an existing channel.
     listpeers         List all active, currently connected peers.
     walletbalance     compute and display the wallet's current balance
     channelbalance    returns the sum of the total available channel balance across all open channels
     getinfo           returns basic information related to the active daemon
     pendingchannels   display information pertaining to pending channels
     sendpayment       send a payment over lightning
     payinvoice        pay an invoice over lightning
     addinvoice        add a new invoice.
     lookupinvoice     Lookup an existing invoice by its payment hash.
     listinvoices      List all invoices currently stored.
     listchannels      list all open channels
     listpayments      list all outgoing payments
     describegraph     describe the network graph
     getchaninfo       get the state of a channel
     getnodeinfo       Get information on a specific node.
     queryroutes       Query a route to a destination.
     getnetworkinfo    getnetworkinfo
     debuglevel        Set the debug level.
     decodepayreq      Decode a payment request.
     listchaintxns     List transactions from the wallet.
     stop              Stop and shutdown the daemon.
     signmessage       sign a message with the node's private key
     verifymessage     verify a message signed with the signature
     feereport         display the current fee policies of all active channels
     updatechanpolicy  update the channel policy for all channels, or a single channel
     help, h           Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --rpcserver value        host:port of ln daemon (default: "localhost:10009")
   --tlscertpath value      path to TLS certificate (default: "/home/nayuta/.lnd/tls.cert")
   --no-macaroons           disable macaroon authentication
   --macaroonpath value     path to macaroon file (default: "/home/nayuta/.lnd/admin.macaroon")
   --macaroontimeout value  anti-replay macaroon validity time in seconds (default: 60)
   --macaroonip value       if set, lock macaroon to specific IP address
   --help, -h               show help
   --version, -v            print the version
```

# デバッグ

## log

- `~/.lnd/logs/` にある
- ある程度のサイズになるとgzで固められるようだが、3ファイルくらいしか残っていないようである

### 見方

- `lnd`が切断したときの例

```
2018-02-05 14:12:59.604 [TRC] PEER: Disconnecting 52.243.46.154:55248, reason: didn't receive ChannelReestablish before deadline
2018-02-05 14:12:59.604 [INF] PEER: unable to read message from 52.243.46.154:55248: read tcp 10.0.0.5:9735->52.243.46.154:55248: use of closed network connection
2018-02-05 14:12:59.604 [TRC] PEER: writeHandler for peer 52.243.46.154:55248 done
2018-02-05 14:12:59.604 [TRC] PEER: readHandler for peer 52.243.46.154:55248 done
2018-02-05 14:12:59.604 [DBG] SRVR: Peer 52.243.46.154:55248 has been disconnected
2018-02-05 14:12:59.604 [TRC] PEER: Update stream for gossiper exited
2018-02-05 14:12:59.604 [DBG] FNDG: Cancelling all reservations for peer 0361949de95991cc68bb82bd968a1bac411830590478e1a00d1907a351211f93b2
2018-02-05 14:12:59.604 [DBG] FNDG: No active reservations for node: 0361949de95991cc68bb82bd968a1bac411830590478e1a00d1907a351211f93b2
2018-02-05 14:12:59.604 [INF] HSWC: Removing channel link with ChannelID(fc48ddb367b541a1358506f76dd3c2478f3ced335ee5653c0e64af909fcacb2c)
2018-02-05 14:12:59.604 [INF] HSWC: ChannelLink(2ccbca9f90af640e3c65e55e33ed3c8f47c2d36df7068535a141b567b3dd48fc:0) is stopping
2018-02-05 14:12:59.604 [DBG] SRVR: removing peer 52.243.46.154:55248
```
