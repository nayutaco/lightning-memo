# site

- https://github.com/nayutaco/ptarmigan

# update

* 通常時
  * アップデート後にDBバージョンが上がっている場合、現在の設定は使用できなくなるため、注意が必要

```bash
git pull
make
```

# 設定

## 設定ファイル

### ptarmiganで使用する設定ファイル
  - `ucoind`を起動したカレントディレクトリ
    - `dbucoin/` : DBディレクトリ(lmdbを使用)
  - 指定可能
    - `peer.conf` : `ucoincli`に与える、接続先ノード情報
      - ノードは、`node_id`, `ipaddr`, `port`のセットなので、それをファイルにしている
      - 再指定する場合にnode_idを調べる手間が省けるのだが、接続できるノードを探している場合にはpeer.confを作成する手間が煩わしい
    - `fund.conf` : `ucoincli`に与える、funding情報。 `pay_fundin.sh`で作成する。
      - channelに入金するoutpointと、入金amountおよび相手に与えるamountを載せている
      - `c-lightning`のようにアドレスも`ptarmigan`で管理するようにするか？

### ~/.bitcoin/bitcoin.conf

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

### ディレクトリ

* `git clone`した`install`ディレクトリにビルドした実行ファイルをコピーしている。
* DBや設定ファイルをカレントディレクトリに作成しているため、作業用ディレクトリを作成するのが良い。

```bash
cd path/to/ptarmigan/install
mkdir node
cd node
```

### 基本

```bash
../ptarmd
```

## debug

`node_announcement`数
```bash
../showdb -n | jq -c '.[] | length'
```

`channel_announcement`数
```.bash
../showdb -c | jq -c '.[] | length'
```