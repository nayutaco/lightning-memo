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

`basedon-v0.10.1`

```
=== bitcoin ===

feerates style
    Return feerate estimates, either satoshi-per-kw ({style} perkw) or satoshi-per-kb ({style} perkb).

fundpsbt satoshi feerate startweight [minconf] [reserve] [locktime] [min_witness_weight] [excess_as_change]
    Create PSBT using enough utxos to allow an output of {satoshi} at {feerate}

newaddr [addresstype]
    Get a new {bech32, p2sh-segwit} (or all) address to fund a channel (default is bech32)

parsefeerate feerate
    Return current feerate in perkw + perkb for given feerate string.

reserveinputs psbt [exclusive] [reserve]
    Reserve utxos (or increase their reservation)

sendpsbt psbt [reserve]
    Finalize, extract and send a PSBT.

signpsbt psbt [signonly]
    Sign this wallet's inputs on a provided PSBT.

unreserveinputs psbt [reserve]
    Unreserve utxos (or at least, reduce their reservation)

utxopsbt satoshi feerate startweight utxos [reserve] [reservedok] [locktime] [min_witness_weight] [excess_as_change]
    Create PSBT using these utxos

=== channels ===

close id [unilateraltimeout] [destination] [fee_negotiation_step] [wrong_funding] [force_lease_closed]
    Close the channel with {id} (either peer ID, channel ID, or short channel ID). Force a unilateral close after {unilateraltimeout} seconds (default 48h). If {destination} address is provided, will be used as output address.

fundchannel_cancel id
    Cancel inflight channel establishment with peer {id}.

fundchannel_complete id psbt
    Complete channel establishment with peer {id} for funding transactionwith {psbt}. Returns true on success, false otherwise.

fundchannel_start id amount [feerate] [announce] [close_to] [push_msat]
    Start fund channel with {id} using {amount} satoshis. Returns a bech32 address to use as an output for a funding transaction.

listforwards [in_channel] [out_channel] [status]
    List all forwarded payments and their information optionally filtering by [in_channel] [out_channel] and [state]

openchannel_abort channel_id
    Abort {channel_id}'s open. Usable while `commitment_signed=false`.

openchannel_bump channel_id amount initialpsbt [funding_feerate]
    Attempt to bump the fee on {channel_id}'s funding transaction.

openchannel_init id amount initialpsbt [commitment_feerate] [funding_feerate] [announce] [close_to] [request_amt] [compact_lease]
    Init an open channel to {id} with {initialpsbt} for {amount} satoshis. Returns updated {psbt} with (partial) contributions from peer

openchannel_signed channel_id signed_psbt
    Send our {signed_psbt}'s tx sigs for {channel_id}.

openchannel_update channel_id psbt
    Update {channel_id} with {psbt}. Returns updated {psbt} with (partial) contributions from peer. If {commitments_secured} is true, next call should be to openchannel_signed

setchannelfee id [base] [ppm]
    Sets specific routing fees for channel with {id} (either peer ID, channel ID, short channel ID or 'all'). Routing fees are defined by a fixed {base} (msat) and a {ppm} (proportional per millionth) value. If values for {base} or {ppm} are left out, defaults will be used. {base} can also be defined in other units, for example '1sat'. If {id} is 'all', the fees will be applied for all channels.

setleaserates lease_fee_base_msat lease_fee_basis funding_weight channel_fee_max_base_msat channel_fee_max_proportional_thousandths
    Called by plugin to set the node's present channel lease rates. Not to be set without having a plugin which can handle `openchannel2` hooks.

=== network ===

connect id [host] [port]
    Connect to {id} at {host} (which can end in ':port' if not default). {id} can also be of the form id@host

disconnect id [force]
    Disconnect from {id} that has previously been connected to using connect; with {force} set, even if it has a current channel

listpeers [id] [level]
    Show current peers, if {level} is set, include logs for {id}

ping id [len] [pongbytes]
    Send peer {id} a ping of length {len} (default 128) asking for {pongbytes} (default 128)

=== payment ===

createinvoice invstring label preimage
    Lowlevel command to sign and create invoice {invstring}, resolved with {preimage}, using unique {label}.

createinvoicerequest bolt12 [recurrence_label]
    Create and sign an invoice_request {bolt12}, with {recurrence_label} if recurring, filling in payer_info and payer_key.

createoffer bolt12 [label] [single_use]
    Create and sign an offer {bolt12} with and optional {label}.

createonion hops assocdata [session_key] [onion_size]
    Create an onion going through the provided nodes, each with its own payload

decodepay bolt11 [description]
    Decode {bolt11}, using {description} if necessary

delexpiredinvoice [maxexpirytime]
    Delete all expired invoices that expired as of given {maxexpirytime} (a UNIX epoch time), or all expired invoices if not specified

delinvoice label status
    Delete unpaid invoice {label} with {status}

delpay payment_hash status
    Delete payment with {payment_hash} and {status}

disableoffer offer_id
    Disable offer {offer_id}

invoice msatoshi label description [expiry] [fallbacks] [preimage] [exposeprivatechannels] [cltv]
    Create an invoice for {msatoshi} with {label} and {description} with optional {expiry} seconds (default 1 week), optional {fallbacks} address list(default empty list) and optional {preimage} (default autogenerated)

listinvoices [label] [invstring] [payment_hash] [offer_id]
    Show invoice matching {label}, {invstring}, {payment_hash} or {offerid} (or all, if no query parameter specified)

listoffers [offer_id] [active_only]
    If {offer_id} is set, show that. Otherwise, if {showdisabled} is true, list all, otherwise just non-disabled ones.

listsendpays [bolt11] [payment_hash]
    Show sendpay, old and current, optionally limiting to {bolt11} or {payment_hash}.

listtransactions
    List transactions that we stored in the wallet

payersign messagename fieldname merkle tweak
    Sign {messagename} {fieldname} {merkle} (a 32-byte hex string) using public {tweak}

sendonion onion first_hop payment_hash [label] [shared_secrets] [partid] [bolt11] [msatoshi] [destination] [localofferid]
    Send a payment with a pre-computed onion.

sendpay route payment_hash [label] [msatoshi] [bolt11] [payment_secret] [partid] [localofferid]
    Send along {route} in return for preimage of {payment_hash}

waitanyinvoice [lastpay_index] [timeout]
    Wait for the next invoice to be paid, after {lastpay_index} (if supplied).  If {timeout} seconds is reached while waiting, fail with an error.

waitinvoice label
    Wait for an incoming payment matching the invoice with {label}, or if the invoice expires

waitsendpay payment_hash [timeout] [partid]
    Wait for payment attempt on {payment_hash} to succeed or fail, but only up to {timeout} seconds.

=== plugin ===

autocleaninvoice [cycle_seconds] [expired_by]
    Set up autoclean of expired invoices.

decode string
    Decode {string} message, returning {type} and information.

estimatefees
    Get the urgent, normal and slow Bitcoin feerates as sat/kVB.

fundchannel id amount [feerate] [announce] [minconf] [utxos] [push_msat] [close_to] [request_amt] [compact_lease]
    Fund channel with {id} using {amount} (or 'all'), at optional {feerate}. Only use outputs that have {minconf} confirmations.

funderupdate [policy] [policy_mod] [leases_only] [min_their_funding_msat] [max_their_funding_msat] [per_channel_min_msat] [per_channel_max_msat] [reserve_tank_msat] [fuzz_percent] [fund_probability] [lease_fee_base_msat] [lease_fee_basis] [funding_weight] [channel_fee_max_base_msat] [channel_fee_max_proportional_thousandths]
    Configuration for dual-funding settings.

getchaininfo
    Get the chain id, the header count, the block count, and whether this is IBD.

getrawblockbyheight height
    Get the bitcoin block at a given height

getroute id msatoshi riskfactor [cltv] [fromid] [fuzzpercent] [exclude] [maxhops]
    Primitive route command

getutxout txid vout
    Get information about an output, identified by a {txid} an a {vout}

keysend destination msatoshi [label] [maxfeepercent] [retry_for] [maxdelay] [exemptfee] [routehints]
    Send a payment without an invoice to a node

legacypay bolt11 [msatoshi] [label] [riskfactor] [maxfeepercent] [retry_for] [maxdelay] [exemptfee]
    Send payment specified by {bolt11} with {amount}

listchannels [short_channel_id] [source] [destination]
    List all known channels in the network

listincoming
    List the channels incoming from our direct peers

listnodes [id]
    List all known nodes in the network

listpays [bolt11] [payment_hash]
    List result of payment {bolt11} or {payment_hash}, or all

multifundchannel destinations [feerate] [minconf] [utxos] [minchannels] [commitment_feerate]
    Fund channels to {destinations}, which is an array of objects containing peer {id}, {amount}, and optional {announce} and {push_msat}.  A single transaction will be used to fund all the channels.  Use {feerate} for the transaction, select outputs that are buried {minconf} blocks deep, or specify a set of {utxos}.

multiwithdraw outputs [feerate] [minconf] [utxos]
    Send to multiple {outputs} via a single Bitcoin transaction.

offer amount description [vendor] [label] [quantity_min] [quantity_max] [absolute_expiry] [recurrence] [recurrence_base] [recurrence_paywindow] [recurrence_limit] [single_use]
    Create an offer to accept money

offerout amount description [vendor] [label] [absolute_expiry] [refund_for]
    Create an offer to send money

pay bolt11 [msatoshi] [label] [riskfactor] [maxfeepercent] [retry_for] [maxdelay] [exemptfee] [localofferid]
    Send payment specified by {bolt11}

paystatus [bolt11]
    Detail status of attempts to pay {bolt11}, or all

plugin subcommand=start|stop|startdir|rescan|list
    Control plugins (start, stop, startdir, rescan, list)

sendrawtransaction tx allowhighfees
    Send a raw transaction to the Bitcoin network.

txdiscard txid
    Discard a transaction created by txprepare

txprepare outputs [feerate] [minconf] [utxos]
    Create a transaction, with option to spend in future (either txsend and txdiscard)

txsend txid
    Send a transaction created by txprepare

withdraw destination satoshi [feerate] [minconf] [utxos]
    Send funds to {destination} address

=== utility ===

addgossip message
    Inject gossip {message} into gossipd

check command_to_check
    Don't run {command_to_check}, just verify parameters.

checkmessage message zbase [pubkey]
    Verify a digital signature {zbase} of {message} signed with {pubkey}

dev-sendcustommsg node_id msg
    Send a custom message to the peer with the given {node_id}

getinfo
    Show information about this node

getlog [level]
    Show logs, with optional log {level} (info|unusual|debug|io)

getsharedsecret point
    Compute the hash of the Elliptic Curve Diffie Hellman shared secret point from this node private key and an input {point}.

help [command]
    List available commands, or give verbose help on one {command}.

listconfigs [config]
    List all configuration options, or with [config], just that one.

listfunds [spent]
    Show available funds from the internal wallet

notifications enable
    Enable notifications for {level} (or 'false' to disable)

sendcustommsg node_id msg
    Send a custom message to the peer with the given {node_id}

sendonionmessage hops [reply_path]
    Send message over {hops} (id, [short_channel_id], [blinding], [enctlv], [invoice], [invoice_request], [invoice_error], [rawtlv]) with optional {reply_path} (blinding, path[id, enctlv])

signmessage message
    Create a digital signature of {message}

stop
    Shut down the lightningd process

waitblockheight blockheight [timeout]
    Wait for the blockchain to reach {blockheight}, up to {timeout} seconds.

=== developer ===

dev-listaddrs [bip32_max_index]
    Show addresses list up to derivation {index} (default is the last bip32 index)

dev-rescan-outputs
    Synchronize the state of our funds with bitcoind

---
run `lightning-cli help <command>` for more information on a specific command
```

# devel

* `update_fulfill_htlc`を送信させない
  * [issue 366](https://github.com/ElementsProject/lightning/issues/366#issuecomment-346249070)
