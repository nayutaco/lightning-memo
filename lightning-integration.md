## URL

* [lightning-integration](https://github.com/cdecker/lightning-integration)

## build

* libsecp256k1のビルドに失敗していたが、[こちら](https://github.com/JoinMarket-Org/joinmarket/wiki/Installing-the-libsecp256k1-binding)を参考にしてapt installを追加したら成功した。

```bash
sudo apt install bitcoind python3 python3-pip
sudo apt install build-essential automake pkg-config libtool libffi-dev libgmp-dev python-dev libsecp256k1-dev
python3 -m venv penv
source penv/bin/activate
pip install -r requirements.txt
```

* pkg-config、secp256k1のビルドに失敗したためインストール。（2019/4/10追記）

```bash
sudo apt install pkg-config
pip3 install secp256k1
```

* pythonにmakoというライブラリがいる

c-lightningのビルドに失敗する

```bash
pip install mako
```

* pythonにattrsというライブラリがいる

`TypeError: attrib() got an unexpected keyword argument 'convert'`というエラーが出る

```bash
pip install attrs==19.1.0
```

## usage

* `make clients`で全部ビルド
  * だけどgoのパッケージとか入れとかないといけない。
  * GOPATHの設定とかも
  * mavenは3.5.4以上だが、ubuntuの`apt`だと3.5.2しかインストールされなかったので手動でインストールした(2019/01/08)
  * eclairのguiのモジュールのビルドがエラーになるけど気にしない。

```bash
# (2019/01/29)

#c-lightning
sudo apt-get install -y autoconf automake build-essential git libtool libgmp-dev libsqlite3-dev python python3 net-tools zlib1g-dev clang

#eclair
wget http://ftp.yz.yamagata-u.ac.jp/pub/network/apache/maven/maven-3/3.6.0/binaries/apache-maven-3.6.0-bin.tar.gz
tar xzvf apache-maven-3.6.0-bin.tar.gz
sudo mv apache-maven-3.6.0 /usr/local/apache-maven
rm apache-maven-3.6.0-bin.tar.gz

#lnd
sudo snap install --classic go

#environment
export PATH=$PATH:/usr/local/apache-maven/bin
export GOPATH=~/gocode
export PATH=$PATH:$GOPATH/bin
```

## execute

* clientを最新にして、makeしなおして、テストを全部実行

```bash
make clean
make update
make clients
make test
```

* 全部

```bash
py.test -v test.py
```

* 一部

```bash
# ptarmiganの入ったテストのみ実施
pytest -v test.py -k 'ptarmigan or PtarmNode'

# ptarmiganとlndの入ったテストのみ実施
pytest -v test.py -k '(ptarmigan or PtarmNode) and lnd'

# 接続(eclair--eclair)のみ実施
pytest -v test.py -k 'test_connect[eclair_eclair]'

# ptarmiganの転送関係テストのみ実施
pytest -v test.py -k 'test_forwarded_payment and ptarmigan'

# ptarmiganのgossip以外
pytest -v test.py -k '(ptarmigan or PtarmNode) and not test_gossip'
```

## result

* ログやDBは /tmp/lightning-???? 下に保存される。/tmp/ltests-????にも何か保存される
  * ????は
  * 全部消したければ`rm -rf /tmp/lightning-* /tmp/ltests-*`
* 最後に失敗したテストは `<PYTEST DIR>/.pytest_cache/v/cache/lastfailed` に名前が残っている

## trouble

* `bitcoind`が既に起動している
* 何かのスクリプトが起動している
* ディスクに空きがない

##

### `$ pytest -v test.py -k 'test_gossip[FIRST_SECOND]'`

```
+-----+                                +-----+
|node1| (FIRST)                        |node2| (SECOND)
+--+--+                                +--+--+
   |                                      |
   |                                      |
+--+---+     +------+     +------+     +--+---+     +------+
|nodes0+-----+nodes1+-----+nodes2+-----+nodes3+-----+nodes4|
+------+     +------+     +------+     +------+     +------+
```

1. nodes0～node4はc-lightning、node1はパラメータの前者、node2はパラメータの後者を使う
2. nodes0～node4は事前にopen_channelする
3. generate 30
4. それぞれのnodesが、nodeを5つ、channelを8つ認識するまで待つ(これを書いている時点では最長120秒)
5. node1はnode0と、node2はnode3(正確には最後から2番目のnode)とconnectする
6. node1とnode2でそれぞれnodeを5つ認識できればOK

* 6の確認がテストの本体なのだが、それより前でうまく行かないことがしばしばある
* nodesはchannelの認識を待つけど、node1,2は待たない。これはeclairがlocalのchannelをannounceしないための措置
* `/tmp/lightning-????/test_gossip[FIRST_SECOND]/`以下に`node-1`～`node-7`までのディレクトリがある
  * `node-1`がFIRST、`node-2`がSECOND、`node-3`以降はnodes0～4に対応している
