# Table of Contents

  * [site](#site)
  * [update application](#update-application)
  * [設定](#設定)
    * [lightningd](#lightningd)
    * [~/.bitcoin/bitcoin.conf](#bitcoinbitcoinconf)
  * [起動](#起動)
    * [bitcoind](#bitcoind)
    * [lightningd](#lightningd-1)
      * [基本](#基本)
      * [ログレベル指定](#ログレベル指定)
      * [announcementするIPアドレス指定](#announcementするipアドレス指定)
    * [lightning-cli](#lightning-cli)
      * [状態確認](#状態確認)
      * [入金](#入金)
      * [相手ノードへの接続](#相手ノードへの接続)
      * [チャネル開設](#チャネル開設)
      * [invoice作成](#invoice作成)
      * [送金](#送金)
      * [チャネル閉鎖](#チャネル閉鎖)
  * [lightning-cli help](#lightning-cli-help)
  * [devel](#devel)

# site

- [ElementsProject/lightning](https://github.com/ElementsProject/lightning)

# update application

```bash
git pull
make
```

# 設定

## lightningd

- 特になし
- `~/.lightningd/` に自動生成される
  - やり直したいときは `~/./lightningd`自体を削除すればよい
    - node_idを維持したい場合は、`~/./lightningd/lightningd.sqlite3`ファイルだけ削除する

## ~/.bitcoin/bitcoin.conf

- 私が使っている設定
  - rpcuser/rpcpasswordはこの値でなくてもよいが、設定は必要

```text
testnet=1
server=1
txindex=1
rpcuser=bitcoinuser
rpcpassword=bitcoinpassword
```

# 起動

## bitcoind

- 普通に起動すればよい
- `lightningd`が`~/.bitcoin/bitcoin.conf`を見に行くので、起動しているbitcoindと不一致がないようにする
  - 動いているbitcoindがregtestで、bitcoin.confがtestnetになっている、など

## lightningd

### 基本

```bash
./lightningd/lightningd --network=testnet
```

### ログレベル指定

- `io, debug, info, unusual, broken`

```bash
./lightningd/lightningd --network=testnet --log-level=debug
```

### announcementするIPアドレス指定

- 指定しない場合に`node_announcement`のIPアドレスが未指定になるかどうかは未確認
  - `getinfo`に出てこないので、たぶん未指定になるだろう
  - IPADDRは0.0.0.0でもよい

```bash
./lightningd/lightningd --network=testnet --ipaddr <IPADDR>:<PORT>
```

## lightning-cli

- 出力を見やすくするために`jq`しているが、なくてもよい

### 状態確認

- `getinfo`

```bash
./cli/lightning-cli getinfo | jq
```

- `listpeers`

```bash
./cli/lightning-cli listpeers | jq
```

- `listchannels`

```bash
./cli/lightning-cli listchannels | jq
```

- `listnodes`

```bash
./cli/lightning-cli listnodes | jq
```

### 入金

- walletにbalanceがない場合は、`lightning-cli`でアドレスを取得し、どこかから入金してもらう
  - `bitcoin-cli sendtoaddress <address> <amount BTC>`かfaucetになるだろう
    - 状態がおかしくなったらDBを削除することもあるので、`bitcoin-cli getnewaddress`で作ったアドレスにfaucetで入金して、`bitcoin-cli sendtoaddress`するのがおすすめ
    - 私は[このfaucet](https://testnet.manu.backend.hamburg/faucet)をよく使っている

```bash
./cli/lightning-cli newaddr
```

- 送金したら、`listfunds`の`outputs`に現れるまでしばらく待つ
  - コマンドを何度も実行するのが面倒だったら、`watch -n 30 ./cli/lightning-cli listfunds`などとしておくとよいだろう

```
./cli/lightning-cli listfunds
```

### 相手ノードへの接続

```
./cli/lightning-cli connect <node_id>@<ip>:<port>
```

- connect要求したら、相手がpeerリストに現れるのを待つ
  - 表示されるまでに時間がかかることがある(announcementが多いため？)

```
./cli/lightning-cli listpeers | jq
```

### チャネル開設

- connectが成功したら、`fundchannel`でチャネル開設開始
  - `amount`の単位はsatoshi

```
./cli/lightning-cli fundchannel <node_id> <amount>
```

- fundchannel要求したら、stateが`CHANNELD_AWAITING_LOCKIN`から`CHANNELD_NORMAL`に変化するまで待つ

```
./cli/lightning-cli listpeers | jq
```

### invoice作成

- amount_msatの単位はmsatoshi
- labelとdescriptionは目印程度なので、あまり考えなくて良い
  - descriptionは、BOLT11形式のinvoiceに文字列(purpose of payment(ASCII))として載る

```
./cli/lightning-cli invoice <amount_msat> <label> <description>
```

### 送金

```
./cli/lightning-cli pay <BOLT11 invoice>
```

### チャネル閉鎖

```
# mutual close
./cli/lightning-cli close <peer node_id>
```

```
# unilateral close
./cli/lightning-cli close <peer node_id> true
```

# lightning-cli help

```
./cli/lightning-cli help
dev-blockheight
    Show current block height

dev-setfees
    Set feerate in satoshi-per-kw for {immediate}, {normal} and {slow} (each is optional, when set, separate by spaces) and show the value of those three feerates

connect
    Connect to {id} at {host} (which can end in ':port' if not default). {id} can also be of the form id@host

listnodes
    Show node {id} (or all, if no {id}), in our local network view

getroute
    Show route to {id} for {msatoshi}, using {riskfactor} and optional {cltv} (default 9). If specified search from {fromid} otherwise use this node as source. Randomize the route with up to {fuzzpercent} (0.0 -> 100.0, default 5.0) using {seed} as an arbitrary-size string seed.

listchannels
    Show channel {short_channel_id} (or all known channels, if no {short_channel_id})

invoice
    Create an invoice for {msatoshi} with {label} and {description} with optional {expiry} seconds (default 1 hour)

listinvoice
    (DEPRECATED) Show invoice {label} (or all, if no {label}))

listinvoices
    Show invoice {label} (or all, if no {label})

delinvoice
    Delete unpaid invoice {label} with {status}

delexpiredinvoice
    Delete all expired invoices that expired as of given {maxexpirytime} (a UNIX epoch time), or all expired invoices if not specified

autocleaninvoice
    Set up autoclean of expired invoices. Perform cleanup every {cycle_seconds} (default 3600), or disable autoclean if 0. Clean up expired invoices that have expired for {expired_by} seconds (default 86400).

waitanyinvoice
    Wait for the next invoice to be paid, after {lastpay_index} (if supplied)

waitinvoice
    Wait for an incoming payment matching the invoice with {label}, or if the invoice expires

decodepay
    Decode {bolt11}, using {description} if necessary

help
    List available commands, or give verbose help on one command.

stop
    Shut down the lightningd process

dev-rhash
    Show SHA256 of {secret}

dev-crash
    Crash lightningd by calling fatal()

getinfo
    Show information about this node

getlog
    Show logs, with optional log {level} (info|unusual|debug|io)

fundchannel
    Fund channel with {id} using {satoshi} satoshis

listconfigs
    List all configuration options, or with [config], just that one.

sendpay
    Send along {route} in return for preimage of {payment_hash}

waitsendpay
    Wait for payment attempt on {payment_hash} to succeed or fail, but only up to {timeout} seconds.

listpayments
    Show outgoing payments

pay
    Send payment specified by {bolt11} with optional {msatoshi} (if and only if {bolt11} does not have amount), {description} (required if {bolt11} uses description hash), {riskfactor} (default 1.0), {maxfeepercent} (default 0.5) the maximum acceptable fee as a percentage (e.g. 0.5 => 0.5%), and {retry_for} (default 60) the integer number of seconds before we stop retrying

listpeers
    Show current peers, if {level} is set, include logs for {id}

close
    Close the channel with peer {id}

disconnect
    Disconnect from {id} that has previously been connected to using connect

dev-sign-last-tx
    Sign and show the last commitment transaction with peer {id}

dev-fail
    Fail with peer {id}

dev-reenable-commit
    Re-enable the commit timer on peer {id}

dev-forget-channel
    Forget the channel with peer {id}, ignore UTXO check with {force}='true'.

dev-ping
    Send {peerid} a ping of length {len} asking for {pongbytes}

dev-memdump
    Show memory objects currently in use

dev-memleak
    Show unreferenced memory objects

withdraw
    Send to {destination} address {satoshi} (or 'all') amount via Bitcoin transaction

newaddr
    Get a new {bech32, p2sh-segwit} address to fund a channel

dev-listaddrs
    Show addresses list up to derivation {index} (default is the last bip32 index)

listfunds
    Show available funds from the internal wallet

dev-rescan-outputs
    Synchronize the state of our funds with bitcoind
```

# devel

* `update_fulfill_htlc`を送信させない
  * [issue 366](https://github.com/ElementsProject/lightning/issues/366#issuecomment-346249070)
