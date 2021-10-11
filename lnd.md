# Table of Contents

- [Table of Contents](#table-of-contents)
- [site](#site)
- [update application](#update-application)
- [設定](#設定)
    - [~/.lnd/lnd.conf](#lndlndconf)
        - [bitcoind用の設定例](#bitcoind用の設定例)
        - [neutrino用の設定例](#neutrino用の設定例)
    - [lncli.conf](#lncliconf)
- [起動](#起動)
    - [lnd](#lnd)
        - [基本](#基本)
        - [ログレベル指定](#ログレベル指定)
        - [announcementするIPアドレス指定](#announcementするipアドレス指定)
    - [lncli](#lncli)
        - [基本](#基本-1)
        - [create / unlock](#create--unlock)
        - [状態確認](#状態確認)
        - [入金](#入金)
        - [相手ノードへの接続](#相手ノードへの接続)
        - [チャネル開設](#チャネル開設)
        - [チャネル開設状態](#チャネル開設状態)
        - [invoiceの作成](#invoiceの作成)
        - [invoiceのデコード](#invoiceのデコード)
        - [送金](#送金)
        - [チャネル閉鎖](#チャネル閉鎖)
- [lncli --help](#lncli---help)
    - [v0.12.99-beta(commit-id: 6d661334599ffa2a409ad6b0942328f9fd213d09)](#v01299-betacommit-id-6d661334599ffa2a409ad6b0942328f9fd213d09)
    - [v0.3](#v03)
- [デバッグ](#デバッグ)
    - [log](#log)

# site

- [lightningnetwork/lnd](https://github.com/lightningnetwork/lnd)

# update application

```bash
mkdir -p $GOPATH/src/github.com/lightningnetwork
cd $GOPATH/src/github.com/lightningnetwork
git clone https://github.com/lightningnetwork/lnd.git
cd lnd
make && make install
```

# 設定

## ~/.lnd/lnd.conf

- [サンプル](https://github.com/lightningnetwork/lnd/blob/master/sample-lnd.conf)

### bitcoind用の設定例

~/.lnd/lnd.conf

```text
[Bitcoin]
bitcoin.active=true
bitcoin.node=bitcoind
bitcoin.testnet=true

[Bitcoind]
bitcoind.rpcuser=bitcoinusertestnet
bitcoind.rpcpass=bitcoinpasswordtestnet
bitcoind.zmqpubrawblock=tcp://127.0.0.1:28332
bitcoind.zmqpubrawtx=tcp://127.0.0.1:28333
```

~/.bitcoin/bitcoin.conf
```text
server=1
txindex=1

[testnet]
rpcuser=bitcoinusertestnet
rpcpassword=bitcoinpasswordtestnet
rpcport=18332
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
```

### neutrino用の設定例

`neutrino.connect`にはどこか指定しないといけないのか？

~/.lnd/lnd.conf
```text
[Bitcoin]
bitcoin.active=true
bitcoin.node=neutrino
bitcoin.testnet=true

[neutrino]
neutrino.connect=faucet.lightning.community
```

## lncli.conf

- そういう設定ファイルは無い

# 起動

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

### 基本

networkのデフォルトはmainnetなので、それ以外は引数で指定する。

```bash
lncli getinfo
```

```bash
lncli -n testnet getinfo
```

### create / unlock

- 新規の場合は `create`
  - `~/.lnd/data` がデフォルトのデータディレクトリなので、新規にしたい場合はこれを削除するとよい

```bash
lncli create
```

作成後に[24単語のパスフレーズ](https://github.com/lightningnetwork/lnd/blob/master/docs/recovery.md#24-word-cipher-seeds)が出力される。  
これはazseed形式で、BIP-39とは互換性がないので注意が必要である。  
復元したい場合は、`lncli create`で"Do you have an existing cipher seed mnemonic you want to use?"という質問が出たときにYESとこたえるとよい。

- 新規でない場合は `unlock`

```bash
lncli unlock
```

### 状態確認

- `getinfo`

```bash
lncli getinfo
```

- `listpeers`

```bash
lncli listpeers
```

- ` listchannels`

```bash
lncli listchannels
```

### 入金

- walletにbalanceがない場合は、`lncli`でアドレスを取得し、faucetなどから入金してもらう
  - 私は[このfaucet](https://testnet.manu.backend.hamburg/faucet)をよく使っている

```bash
lncli newaddress np2wkh
```

- balanceの確認
  `confirmed_balance` に値が入るまで待つ。

```bash
lncli walletbalance
```

### 相手ノードへの接続

- [参考](http://dev.lightning.community/tutorial/01-lncli/index.html)
- `<port>`は9735以外の場合に指定する
- `lnd`は自動的にpeerを見つけて接続しに行くようだから、`listpeers`で接続されるのを待った方が良いのかもしれない。

```bash
lncli connect <node_id>@<IP address>[:<port>]
```

### チャネル開設

- `--local_amt`の単位はsatoshis

```bash
lncli openchannel --node_key=<node_id> --local_amt <amt_satoshi>
```

### チャネル開設状態

- 状態の調べ方がよくわからないが、開設できれば`listchannels`に出てくる。

```bash
lncli listchannels
```

### invoiceの作成

- Lightning Networkで送金してもらうために、まずinvoiceを作る
  - `--amt`の単位はsatoshi
  - 成功すると `"pay_req"`に`ln`で始まる文字列が返ってくる。

```bash
lncli addinvoice --amt 100
```

### invoiceのデコード

```bash
lncli decodepayreq <invoice>
```

### 送金

```bash
lncli payinvoice <BOLT11 invoice>
```

### チャネル閉鎖

* <funding_txindex>を指定しないと `[lncli] rpc error: code = Unknown desc = unable to find channel` となることがある
  * txindexが0の場合は省略可能なだけか？

* mutual close

```bash
lncli closechannel --funding_txid <funding_txid> [--output_index <funding_txindex>]
```

* unilateral close

```bash
lncli closechannel --force --funding_txid <funding_txid> [--output_index <funding_txindex>]
```

# lncli --help

## 0.13.3-beta

```
$ lncli
NAME:
   lncli - control plane for your Lightning Network Daemon (lnd)

USAGE:
   lncli [global options] command [command options] [arguments...]

VERSION:
   0.13.3-beta commit=v0.13.3-beta

COMMANDS:
     getinfo          Returns basic information related to the active daemon.
     getrecoveryinfo  Display information about an ongoing recovery attempt.
     debuglevel       Set the debug level.
     stop             Stop and shutdown the daemon.
     version          Display lncli and lnd version info.
     getmccfg         Display mission control's config.
     setmccfg         Set mission control's config.
     help, h          Shows a list of commands or help for one command
   Autopilot:
     autopilot  Interact with a running autopilot.
   Channels:
     openchannel        Open a channel to a node or an existing peer.
     closechannel       Close an existing channel.
     closeallchannels   Close all existing channels.
     abandonchannel     Abandons an existing channel.
     channelbalance     Returns the sum of the total available channel balance across all open channels.
     pendingchannels    Display information pertaining to pending channels.
     listchannels       List all open channels.
     closedchannels     List all closed channels.
     getnetworkinfo     Get statistical information about the current state of the network.
     feereport          Display the current fee policies of all active channels.
     updatechanpolicy   Update the channel policy for all channels, or a single channel.
     exportchanbackup   Obtain a static channel back up for a selected channels, or all known channels
     verifychanbackup   Verify an existing channel backup
     restorechanbackup  Restore an existing single or multi-channel static channel backup
     updatechanstatus   Set the status of an existing channel on the network.
   Graph:
     describegraph   Describe the network graph.
     getnodemetrics  Get node metrics.
     getchaninfo     Get the state of a channel.
     getnodeinfo     Get information on a specific node.
   Invoices:
     addinvoice      Add a new invoice.
     lookupinvoice   Lookup an existing invoice by its payment hash.
     listinvoices    List all invoices currently stored within the database. Any active debug invoices are ignored.
     decodepayreq    Decode a payment request.
     cancelinvoice   Cancels a (hold) invoice
     addholdinvoice  Add a new hold invoice.
     settleinvoice   Reveal a preimage and use it to settle the corresponding invoice.
   Macaroons:
     bakemacaroon      Bakes a new macaroon with the provided list of permissions and restrictions.
     listmacaroonids   List all macaroons root key IDs in use.
     deletemacaroonid  Delete a specific macaroon ID.
     listpermissions   Lists all RPC method URIs and the macaroon permissions they require to be invoked.
     printmacaroon     Print the content of a macaroon in a human readable format.
   On-chain:
     estimatefee    Get fee estimates for sending bitcoin on-chain to multiple addresses.
     sendmany       Send bitcoin on-chain to multiple addresses.
     sendcoins      Send bitcoin on-chain to an address.
     listunspent    List utxos available for spending.
     listchaintxns  List transactions from the wallet.
   Payments:
     sendpayment    Send a payment over lightning.
     payinvoice     Pay an invoice over lightning.
     sendtoroute    Send a payment over a predefined route.
     listpayments   List all outgoing payments.
     queryroutes    Query a route to a destination.
     fwdinghistory  Query the history of all forwarded HTLCs.
     trackpayment   Track progress of an existing payment.
     querymc        Query the internal mission control state.
     importmc       Import a result to the internal mission control state.
     queryprob      Estimate a success probability.
     resetmc        Reset internal mission control state.
     buildroute     Build a route from a list of hop pubkeys.
   Peers:
     connect     Connect to a remote lnd peer.
     disconnect  Disconnect a remote lnd peer identified by public key.
     listpeers   List all active, currently connected peers.
   Profiles:
     profile  Create and manage lncli profiles
   Startup:
     create          Initialize a wallet when starting lnd for the first time.
     unlock          Unlock an encrypted wallet at startup.
     changepassword  Change an encrypted wallet's password at startup.
     state           Get the current state of the wallet and RPC
   Wallet:
     newaddress     Generates a new address.
     walletbalance  Compute and display the wallet's current balance.
     signmessage    Sign a message with the node's private key.
     verifymessage  Verify a message signed with the signature.
     wallet         Interact with the wallet.
   Watchtower:
     tower     Interact with the watchtower.
     wtclient  Interact with the watchtower client.

GLOBAL OPTIONS:
   --rpcserver value          The host:port of LN daemon. (default: "localhost:10009")
   --lnddir value             The path to lnd's base directory. (default: "/home/ubuntu/.lnd")
   --tlscertpath value        The path to lnd's TLS certificate. (default: "/home/ubuntu/.lnd/tls.cert")
   --chain value, -c value    The chain lnd is running on, e.g. bitcoin. (default: "bitcoin")
   --network value, -n value  The network lnd is running on, e.g. mainnet, testnet, etc. (default: "mainnet")
   --no-macaroons             Disable macaroon authentication.
   --macaroonpath value       The path to macaroon file.
   --macaroontimeout value    Anti-replay macaroon validity time in seconds. (default: 60)
   --macaroonip value         If set, lock macaroon to specific IP address.
   --profile value, -p value  Instead of reading settings from command line parameters or using the default profile, use a specific profile. If a default profile is set, this flag can be set to an empty string to disable reading values from the profiles file.
   --macfromjar value         Use this macaroon from the profile's macaroon jar instead of the default one. Can only be used if profiles are defined.
   --help, -h                 show help
   --version, -v              print the version
```

## v0.3

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
             disable macaroon authentication
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
