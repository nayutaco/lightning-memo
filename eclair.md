# Table of Contents

  * [site](#site)
  * [build](#build)
  * [install/update](#installupdate)
  * [設定](#設定)
    * [~/.bitcoin/bitcoind](#bitcoinbitcoind)
    * [~/.eclair/eclair.conf](#eclaireclairconf)
  * [起動](#起動)
    * [eclair-node-x.x-yyyy.jar](#eclair-node-xx-yyyyjar)
      * [エラー](#エラー)
    * [eclair-cli](#eclair-cli)
      * [状態確認](#状態確認)
      * [接続](#接続)
      * [チャネル開設](#チャネル開設)
      * [invoice作成](#invoice作成)
      * [送金](#送金)
  * [eclair-cli help](#eclair-cli-help)

# site

* [ACINQ/eclair](https://github.com/ACINQ/eclair)

# build

* mavenのバージョンは3.5.4以上
  * aptではインストールできなかったので、ダウンロードした

```bash
#maven >= 3.5.4
wget http://ftp.tsukuba.wide.ad.jp/software/apache/maven/maven-3/3.6.0/binaries/apache-maven-3.6.0-bin.tar.gz
tar zxf apache-maven-3.6.0-bin.tar.gz
sudo mv apache-maven-3.6.0 /usr/local/
export PATH=/usr/local/apache-maven-3.6.0/bin:$PATH
```

* integration-testのMakefileを参考にした
  * ビルドできた場合、`eclair-node/target/`の中にjarファイルができている
    * 4つjarファイルがあるが、一番サイズが大きいやつ

```
#eclair
git clone https://github.com/ACINQ/eclair.git
cd eclair
mvn package -Dmaven.test.skip=true
```

# install/update

* 定期的にリリースされているため、それを使用すると良い
  * [Release](https://github.com/ACINQ/eclair/releases)
* Windows/Mac/Linuxで使用可能
  * ここでは何も書いていない場合、Azure上のUbuntu16.04で動作させる
  * java環境はこのようにしてインストールするとよい
    * java8にしたが、java9は確認していない

```bash
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt update
sudo apt install oracle-java8-installer
```

```bash
mkdir path/to/somewhere
cd path/to/somewhere
wget https://raw.githubusercontent.com/ACINQ/eclair/master/eclair-core/eclair-cli
chmod u+x eclair-cli
```

# 設定

* [v0.2-alpha11](https://github.com/ACINQ/eclair/releases/tag/v0.2-alpha11)
  * change DB format from alpha10

## ~/.bitcoin/bitcoind

```text
testnet=1
server=1
txindex=1
rpcuser=bitcoinuser
rpcpassword=bitcoinpassword
zmqpubrawblock=tcp://127.0.0.1:29000
zmqpubrawtx=tcp://127.0.0.1:29000
```

## ~/.eclair/eclair.conf

* bitcoin.confが[デフォルト値](https://github.com/ACINQ/eclair/blob/master/eclair-core/src/main/resources/reference.conf)ならば、無くてもよいはず。
* GUI版ではない場合、APIを使えるようにしないと何もできない。
  * `eclair.api.password`の値は、ダウンロードした `eclair-cli`の8行目くらいにある `PASSWORD`も変更しておいた方がいろいろ楽だろう

```text
eclair.bitcoind.rpcuser=bitcoinuser
eclair.bitcoind.rpcpassword=bitcoinpassword
eclair.api.enabled=true
eclair.api.password=xxxxx
```

## regtest

* ~/.bitcoin/bitcoin.conf

```text
regtest=1
server=1
txindex=1

rpcuser=bitcoinuser
rpcpassword=bitcoinpassword
zmqpubrawblock=tcp://127.0.0.1:29000
zmqpubrawtx=tcp://127.0.0.1:29000
```

* ~/.eclair/eclair.conf

```text
eclair.chain=regtest
eclair.bitcoind.rpcuser=bitcoinuser
eclair.bitcoind.rpcpassword=bitcoinpassword
eclair.bitcoind.rpcport=18443
eclair.api.enabled=true
eclair.api.password=xxxxx
```

# 起動

## bitcoind

* bitcoind v0.16

```bash
bitcoind -deprecatedrpc=addwitnessaddress -daemon
```

* bitcoind v0.15

```bash
bitcoind -daemon
```

## eclair-node-x.x-yyyy.jar

```bash
java -jar eclair-node-0.2-alpha11-dbb4f5b.jar
```

* 何もエラーがない場合、起動しても何も出力されないようである。

### エラー

* `fatal error: could not connect to bitcoind using json-rpc`
  * 原因：bitcoin.confとeclair.confの、JSON-RPC設定が不一致
  * 対策：どちらかのファイルを変更して一致させる

* `fatal error: assertion failed: In testnet mode, make sure that all your UTXOs are p2sh-of-p2wpkh (check out our README for more details)`
  * 原因：(testnet) bitcoindのP2WPKH nested in BIP16 P2SHのアドレスにbalanceが全部入っていない
    * 全部のbalanceがP2WPKH(in BIP16)に割り当てられていないといけない
    * bitcoindを立ち上げたばかりであれば、P2WPKH(in BIP16)のアドレスを作って、Faucetから送金した方が楽
  * 対策：`addwitnessaddress`したアドレスにすべて送金する

```bash
$ bitcoin-cli getbalance
zzzzzzzzzzzzz
$ bitcoin-cli getnewaddress
xxxxxxxxxxxxx
$ bitcoin-cli addwitnessaddress xxxxxxxxxxxxx
yyyyyyyyyyyyy
$ bitcoin-cli sendtoaddress yyyyyyyyyyyyy zzzzzzzzzzzzz
(おそらくfeeがいくら足りないというエラーになる)
$ bitcoin-cli sendtoaddress yyyyyyyyyyyyy ZZZZZZZZZZZZZ(zzzzzzzzzzzzzからfeeを引いた額)
```

## eclair-cli

### 状態確認

- `getinfo`

```bash
./eclair-cli getinfo
```

- `peers`

```bash
./eclair-cli peers
```

- `channels`

```bash
./eclair-cli channels
```

- `channel`
  * balanceを見ることができる

```bash
./eclair-cli channel <channelId>
```

- `allnodes`
  * おそらく、announcementされたnode

```bash
./eclair-cli allnodes
```

### 接続

* hostはドメイン名でもよい
* portはデフォルトでも省略できない

```bash
./eclair-cli connect <node_id> <host> <port>

  OR

./eclair-cli connect <node_id>@<host>:<port>
```

### チャネル開設

* fund_satoshiの単位はsatoshi
* confirmation待ちになると`channel created`とログが出てくるようだ
  * 自分がopenしようとした場合は、まだconfirmationが達していなくても、`./eclair-cli peers`では`channels`にカウントされている
  * 相手がopenしようとした場合は、`./eclair-cli channels`で"WAIT_FOR_FUNDING_CONFIRMED"という状態になっているようだ

```bash
./eclair-cli open <node_id> <fund_satoshi>
```

* 相手からopenされ、feerate_per_kwがあまりにも違う場合はエラーを返してくる
  * 例： `local/remote feerates are too different: remoteFeeratePerKw=253 localFeeratePerKw=53760`
    * このときの`bitcoin-cli estimatefee 6`は、`0.00001015`だった(estimatesmartfeeも同じ)
      * 1015satoshiなので、ptarmiganはそれを4で割り、253にしている
      * どうやって53760という値が出てきたのだろう？

### invoice作成

* amount_msatの単位はmsatoshi
* descriptionは文字列

```bash
./eclair-cli receive <amount_msat> <description>
```

### 送金

```bash
./eclair-cli send <BOLT11 invoice>
```

# eclair-cli help

```text
$ ./eclair-cli help
Usage: eclair-cli [OPTION]... [COMMAND]
Client for an eclair node.

With COMMAND is one of the command listed by eclair-cli help.

  -p <password>         api's password
  -a <address>          Override the api URL with <address>
  -v                    Outputs full json returned by the API

Examples:
  eclair-cli -a localhost:1234 peers     list the peers
  eclair-cli close 006fb...              closes the channel with id 006fb...

Note: Uses the json-rpc api exposed by the node on localhost:8080. Make sure the api is enabled.
Full documentation at: <https://github.com/ACINQ/eclair>

Available commands:

[
  "connect (uri): open a secure connection to a lightning node",
  "connect (nodeId, host, port): open a secure connection to a lightning node",
  "open (nodeId, fundingSatoshi, pushMsat = 0, channelFlags = 0x01): open a channel with another lightning node",
  "peers: list existing local peers",
  "channels: list existing local channels",
  "channels (nodeId): list existing local channels to a particular nodeId",
  "channel (channelId): retrieve detailed information about a given channel",
  "allnodes: list all known nodes",
  "allchannels: list all known channels",
  "allupdates: list all channels updates",
  "allupdates (nodeId): list all channels updates for this nodeId",
  "receive (amountMsat, description): generate a payment request for a given amount",
  "send (amountMsat, paymentHash, nodeId): send a payment to a lightning node",
  "send (paymentRequest): send a payment to a lightning node using a BOLT11 payment request",
  "send (paymentRequest, amountMsat): send a payment to a lightning node using a BOLT11 payment request and a custom amount",
  "close (channelId): close a channel",
  "close (channelId, scriptPubKey): close a channel and send the funds to the given scriptPubKey",
  "checkpayment (paymentHash): returns true if the payment has been received, false otherwise",
  "checkpayment (paymentRequest): returns true if the payment has been received, false otherwise",
  "getinfo: returns info about the blockchain and this node",
  "help: display this message"
]
```

# fee

* [feerate_per_kwをbyte単位のfeerateに変換しているところ？](https://github.com/ACINQ/eclair/blob/924efeabe342192193ed386c176a4e304d0fecfb/eclair-core/src/main/scala/fr/acinq/eclair/package.scala#L69)
  * 1000で割って4倍しているから、feerate_per_kwは4分の1しているということか
    * [下の行](https://github.com/ACINQ/eclair/blob/924efeabe342192193ed386c176a4e304d0fecfb/eclair-core/src/main/scala/fr/acinq/eclair/package.scala#L98)に変換式があった
  * 最小値が253である根拠の計算も、その下のコメントに書いてあった
    * が、よくわからん。segwitになってvsize方式になったのだが、関係しているのか。
* feerateの取得は[10分ごと](https://github.com/ACINQ/eclair/blob/924efeabe342192193ed386c176a4e304d0fecfb/eclair-core/src/main/scala/fr/acinq/eclair/Setup.scala#L154-L161)か？
* [estimatesmartfeeしているところ？](https://github.com/ACINQ/eclair/blob/924efeabe342192193ed386c176a4e304d0fecfb/eclair-core/src/main/scala/fr/acinq/eclair/blockchain/fee/BitcoinCoreFeeProvider.scala)
  * 取得できればその値を、できなかったら[defaultFeerates](https://github.com/ACINQ/eclair/blob/924efeabe342192193ed386c176a4e304d0fecfb/eclair-core/src/main/scala/fr/acinq/eclair/Setup.scala#L140-L147)を使うのか。
  * [bitcoinfees.earn.com](https://bitcoinfees.earn.com/api/v1/fees/list)のサービスを使っている？？
    * [src](https://github.com/ACINQ/eclair/blob/924efeabe342192193ed386c176a4e304d0fecfb/eclair-core/src/main/scala/fr/acinq/eclair/blockchain/fee/EarnDotComFeeProvider.scala#L44)
    * mainnetとtestnetで使い分けているのだろうか？
    * Setup.scalaはbitcoindとelectrumの分岐もあるので、なんかやってるんだろう、きっと。

```
  def estimateSmartFee(nBlocks: Int): Future[Long] =
    rpcClient.invoke("estimatesmartfee", nBlocks).map(BitcoinCoreFeeProvider.parseFeeEstimate)

  override def getFeerates: Future[FeeratesPerKB] = for {
    block_1 <- estimateSmartFee(1)
    blocks_2 <- estimateSmartFee(2)
    blocks_6 <- estimateSmartFee(6)
    blocks_12 <- estimateSmartFee(12)
    blocks_36 <- estimateSmartFee(36)
    blocks_72 <- estimateSmartFee(72)
  } yield FeeratesPerKB(
    block_1 = if (block_1 > 0) block_1 else defaultFeerates.block_1,
    blocks_2 = if (blocks_2 > 0) blocks_2 else defaultFeerates.blocks_2,
    blocks_6 = if (blocks_6 > 0) blocks_6 else defaultFeerates.blocks_6,
    blocks_12 = if (blocks_12 > 0) blocks_12 else defaultFeerates.blocks_12,
    blocks_36 = if (blocks_36 > 0) blocks_36 else defaultFeerates.blocks_36,
    blocks_72 = if (blocks_72 > 0) blocks_72 else defaultFeerates.blocks_72)
```