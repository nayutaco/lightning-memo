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
  * [lightningd help](#lightningd-help)
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

### lightningd help

`v23.08.1`

```text
Usage: lightningd 
A bitcoin lightning daemon (default values shown for network: regtest).
--list-features-only                              List the features configured, and exit immediately
--conf=<file>                                     Specify configuration file
--lightning-dir=<dir>                             Set base directory: network-specific subdirectory is under here (default: "/home/ueno/.lightning")
--network <arg>                                   Select the network parameters (bitcoin, testnet, signet, regtest, litecoin or litecoin-testnet) (default: regtest)
--testnet                                         Alias for --network=testnet
--signet                                          Alias for --network=signet
--mainnet                                         Alias for --network=bitcoin
--regtest                                         Alias for --network=regtest
--version|-V                                      Print version and exit
--rpc-file <arg>                                  Set JSON-RPC socket (or /dev/tty) (default: "lightning-rpc")
--test-daemons-only                               Test that subdaemons can be run, then exit immediately
--allow-deprecated-apis <arg>                     Enable deprecated options, JSONRPC commands, fields, etc. (default: true)
--plugin <arg>                                    Add a plugin to be run (can be used multiple times)
--plugin-dir <arg>                                Add a directory to load plugins from (can be used multiple times)
--clear-plugins                                   Remove all plugins added before this option
--disable-plugin <arg>                            Disable a particular plugin by filename/name
--important-plugin <arg>                          Add an important plugin to be run (can be used multiple times). Die if the plugin dies.
--always-use-proxy <arg>                          Use the proxy always (default: false)
--daemon                                          Run in the background, suppress stdout/stderr
--wallet <arg>                                    Location of the wallet database.
--recover <arg>                                   Populate hsm_secret with the given codex32 secret and starts the node in `offline` mode.
--large-channels|--wumbo                          Allow channels larger than 0.16777215 BTC
--experimental-dual-fund                          experimental: Advertise dual-funding and allow peers to establish channels via v2 channel open protocol.
--experimental-splicing                           experimental: Enables the ability to resize channels using splicing
--experimental-onion-messages                     EXPERIMENTAL: enable send, receive and relay of onion messages and blinded payments
--experimental-offers                             EXPERIMENTAL: enable send and receive of offers (also sets experimental-onion-messages)
--experimental-shutdown-wrong-funding             EXPERIMENTAL: allow shutdown with alternate txids
--experimental-peer-storage                       EXPERIMENTAL: enable peer backup storage and restore
--experimental-quiesce                            experimental: Advertise ability to quiesce channels.
--experimental-anchors                            EXPERIMENTAL: enable option_anchors_zero_fee_htlc_tx to open zero-fee-anchor channels
--help|-h                                         Print this message.
--rgb <arg>                                       RRGGBB hex color for node
--alias <arg>                                     Up to 32-byte alias for node
--pid-file=<file>                                 Specify pid file (default: "/home/ueno/.lightning/lightningd-regtest.pid")
--ignore-fee-limits <arg>                         (DANGEROUS) allow peer to set any feerate (default: true)
--watchtime-blocks <arg>                          Blocks before peer can unilaterally spend funds (default: 6)
--max-locktime-blocks <arg>                       Maximum blocks funds may be locked for (default: 2016)
--funding-confirms <arg>                          Confirmations required for funding transaction (default: 1)
--require-confirmed-inputs <arg>                  Confirmations required for inputs to funding transaction (v2 opens only) (default: false)
--cltv-delta <arg>                                Number of blocks for cltv_expiry_delta (default: 6)
--cltv-final <arg>                                Number of blocks for final cltv_expiry (default: 10)
--commit-time=<millseconds>                       Time after changes before sending out COMMIT (default: 10)
--fee-base <arg>                                  Millisatoshi minimum to charge for HTLC (default: 1)
--rescan <arg>                                    Number of blocks to rescan from the current head, or absolute blockheight if negative (default: 30)
--fee-per-satoshi <arg>                           Microsatoshi fee for every satoshi in HTLC (default: 10)
--htlc-minimum-msat <arg>                         The default minimal value an HTLC must carry in order to be forwardable for new channels (default: 0)
--htlc-maximum-msat <arg>                         The default maximal value an HTLC must carry in order to be forwardable for new channel (default: 18446744073709551615)
--max-concurrent-htlcs <arg>                      Number of HTLCs one channel can handle concurrently. Should be between 1 and 483 (default: 483)
--max-dust-htlc-exposure-msat <arg>               Max HTLC amount that can be trimmed (default: 50000000)
--min-capacity-sat <arg>                          Minimum capacity in satoshis for accepting channels (default: 10000)
--addr <arg>                                      Set an IP address (v4 or v6) to listen on and announce to the network for incoming connections
--bind-addr <arg>                                 Set an IP address (v4 or v6) to listen on, but not announce
--announce-addr <arg>                             Set an IP address (v4 or v6) or .onion v3 to announce, but not listen on
--announce-addr-discovered <arg>                  Explicitly turns IP discovery 'on' or 'off'. (default: auto)
--announce-addr-discovered-port <arg>             Sets the public TCP port to use for announcing discovered IPs. (default: 19846)
--offline                                         Start in offline-mode (do not automatically reconnect and do not accept incoming connections)
--autolisten <arg>                                If true, listen on default port and announce if it seems to be a public interface (default: true)
--proxy <arg>                                     Set a socks v5 proxy IP address and port
--tor-service-password <arg>                      Set a Tor hidden service password
--accept-htlc-tlv-type <arg>                      HTLC TLV type to accept (can be used multiple times)
--disable-dns                                     Disable DNS lookups of peers
--encrypted-hsm                                   Set the password to encrypt hsm_secret with. If no password is passed through command line, you will be prompted to enter it.
--rpc-file-mode <arg>                             Set the file mode (permissions) for the JSON-RPC socket (default: 0600)
--force-feerates <arg>                            Set testnet/regtest feerates in sats perkw, opening/mutual_close/unlateral_close/delayed_to_us/htlc_resolution/penalty: if fewer specified, last
                                                  number applies to remainder
--commit-fee <arg>                                Percentage of fee to request for their commitment (default: 100)
--min-emergency-msat <arg>                        Amount to leave in wallet for spending anchor closes (default: 25000000)
--subdaemon <arg>                                 Arg specified as SUBDAEMON:PATH. Specifies an alternate subdaemon binary. If the supplied path is relative the subdaemon binary is found in the
                                                  working directory. This option may be specified multiple times. For example, --subdaemon=hsmd:remote_signer would use a hypothetical remote
                                                  signing subdaemon.
--experimental-upgrade-protocol                   experimental: allow channel types to be upgraded on reconnect
--database-upgrade <arg>                          Set to true to allow database upgrades even on non-final releases (WARNING: you won't be able to downgrade!)
--log-level <arg>                                 log level (io, debug, info, unusual, broken) [:prefix] (default: debug)
--log-timestamps <arg>                            prefix log messages with timestamp (default: true)
--log-prefix <arg>                                log prefix
--log-file=<file>                                 Also log to file (- for stdout)
--renepay-debug-mcf                               Enable renepay MCF debug info.
--renepay-debug-payflow                           Enable renepay payment flows debug info.
--bitcoin-datadir <arg>                           -datadir arg for bitcoin-cli
--bitcoin-cli <arg>                               bitcoin-cli pathname
--bitcoin-rpcuser <arg>                           bitcoind RPC username
--bitcoin-rpcpassword <arg>                       bitcoind RPC password
--bitcoin-rpcconnect <arg>                        bitcoind RPC host to connect to
--bitcoin-rpcport <arg>                           bitcoind RPC host's port
--bitcoin-retry-timeout <arg>                     how long to keep retrying to contact bitcoind before fatally exiting
--funder-policy <arg>                             Policy to use for dual-funding requests. [match, available, fixed]
--funder-policy-mod <arg>                         Percent to apply policy at (match/available); or amount to fund (fixed)
--funder-min-their-funding <arg>                  Minimum funding peer must open with to activate our policy
--funder-max-their-funding <arg>                  Maximum funding peer may open with to activate our policy
--funder-per-channel-min <arg>                    Minimum funding we'll add to a channel. If we can't meet this, we don't fund
--funder-per-channel-max <arg>                    Maximum funding we'll add to a channel. We cap all contributions to this
--funder-reserve-tank <arg>                       Amount of funds we'll always leave available.
--funder-fuzz-percent <arg>                       Percent to fuzz the policy contribution by. Defaults to 0%. Max is 100%
--funder-fund-probability <arg>                   Percent of requests to consider. Defaults to 100%. Setting to 0% will disable dual-funding
--funder-lease-requests-only <arg>                Only fund lease requests. Defaults to true if channel lease rates are being advertised
--lease-fee-base-sat <arg>                        Channel lease rates, base fee for leased funds, in satoshi.
--lease-fee-basis <arg>                           Channel lease rates, basis charged for leased funds (per 10,000 satoshi.)
--lease-funding-weight <arg>                      Channel lease rates, weight we'll ask opening peer to pay for in funding transaction
--channel-fee-max-base-msat <arg>                 Channel lease rates, maximum channel fee base we'll charge for funds routed through a leased channel.
--channel-fee-max-proportional-thousandths <arg>  Channel lease rates, maximum proportional fee (in thousandths, or ppt) we'll charge for funds routed through a leased channel. Note: 1ppt =
                                                  1,000ppm
--disable-mpp                                     Disable multi-part payments.
--dev-sqlfilename <arg>                           Use on-disk sqlite3 file instead of in memory (e.g. debugging)
--fetchinvoice-noconnect                          Don't try to connect directly to fetch an invoice.
--grpc-port <arg>                                 Which port should the grpc plugin listen for incoming connections? (default: -1)
--autoclean-cycle <arg>                           Perform cleanup every given seconds
--autoclean-succeededforwards-age <arg>           How old do successful forwards have to be before deletion (0 = never)
--autoclean-failedforwards-age <arg>              How old do failed forwards have to be before deletion (0 = never)
--autoclean-succeededpays-age <arg>               How old do successful pays have to be before deletion (0 = never)
--autoclean-failedpays-age <arg>                  How old do failed pays have to be before deletion (0 = never)
--autoclean-paidinvoices-age <arg>                How old do paid invoices have to be before deletion (0 = never)
--autoclean-expiredinvoices-age <arg>             How old do expired invoices have to be before deletion (0 = never)
--bookkeeper-dir <arg>                            Location for bookkeeper records.
--bookkeeper-db <arg>                             Location of the bookkeeper database
```

`configure --enable-developer`を付けると開発者モードでビルドされる。

```text
$ ./lightningd --help | grep -e "--dev"
--dev-debugger=<subprocess>                       Invoke gdb at start of <subprocess>
--dev-no-plugin-checksum                          Don't checksum plugins to detect changes
--dev-builtin-plugins-unimportant                 Make builtin plugins unimportant so you can plugin stop
--dev-no-reconnect                                Disable automatic reconnect-attempts by this node, but
--dev-fast-reconnect                              Make max default reconnect delay 3 (not 300) seconds
--dev-disconnect=<filename>                       File containing disconnection points
--dev-allow-localhost                             Announce and allow announcments for localhost address
--dev-bitcoind-poll <arg>                         Time between polling for new transactions (default: 30)
--dev-fast-gossip                                 Make gossip broadcast 1 second, etc
--dev-fast-gossip-prune                           Make gossip pruning 30 seconds
--dev-gossip-time <arg>                           UNIX time to override gossipd to use. (default: 0)
--dev-force-privkey <arg>                         Force HSM to use this as node private key
--dev-force-bip32-seed <arg>                      Force HSM to use this as bip32 seed
--dev-force-channel-secrets <arg>                 Force HSM to use these for all per-channel secrets
--dev-max-funding-unconfirmed-blocks <arg>        Maximum number of blocks we wait for a channel funding
--dev-force-tmp-channel-id <arg>                  Force the temporary channel id, instead of random
--dev-no-htlc-timeout                             Don't kill channeld if HTLCs not confirmed within 30 seconds
--dev-fail-process-onionpacket                    Force all processing of onion packets to fail
--dev-no-version-checks                           Skip calling subdaemons with --version on startup
--dev-force-features <arg>                        Force the init/globalinit/node_announce/channel/bolt11/
--dev-timeout-secs <arg>                          Seconds to timeout if we don't receive INIT from peer
--dev-no-modern-onion                             Ignore modern onion messages
--dev-disable-commit-after <arg>                  Disable commit timer after this many commits (default: -1)
--dev-no-ping-timer                               Don't hang up if we don't get a ping response
--dev-onion-reply-length <arg>                    Send onion errors of custom length (default: 256)
--dev-max-fee-multiplier <arg>                    Allow the fee proposed by the remote end to be up to
--dev-allowdustreserve <arg>                      If true, we allow the `fundchannel` RPC command and the
--dev-no-fake-fees <arg>                          Suppress fee faking for regtest
--dev-sqlfilename <arg>                           Use on-disk sqlite3 file instead of in memory (e.g.
```

### lightning-cli help

`v23.08.1`

```text
=== backup ===

staticbackup 
    Returns SCB of all the channels currently present in the DB

=== bitcoin ===

feerates style
    Return feerate estimates, either satoshi-per-kw ({style} perkw) or satoshi-per-kb ({style} perkb).

fundpsbt satoshi feerate startweight [minconf] [reserve] [locktime] [min_witness_weight] [excess_as_change] [nonwrapped] [opening_anchor_channel]
    Create PSBT using enough utxos to allow an output of {satoshi} at {feerate}

newaddr [addresstype]
    Get a new {bech32} (or all) address to fund a channel

parsefeerate feerate
    Return current feerate in perkw + perkb for given feerate string.

reserveinputs psbt [exclusive] [reserve]
    Reserve utxos (or increase their reservation)

sendpsbt psbt [reserve]
    Finalize, extract and send a PSBT.

setpsbtversion psbt version
    Convert a given PSBT to the {version} requested (v0 or v2)

signpsbt psbt [signonly]
    Sign this wallet's inputs on a provided PSBT.

unreserveinputs psbt [reserve]
    Unreserve utxos (or at least, reduce their reservation)

utxopsbt satoshi feerate startweight utxos [reserve] [reservedok] [locktime] [min_witness_weight] [excess_as_change] [opening_anchor_channel]
    Create PSBT using these utxos

=== channels ===

close id [unilateraltimeout] [destination] [fee_negotiation_step] [wrong_funding] [force_lease_closed] [feerange]
    Close the channel with {id} (either peer ID, channel ID, or short channel ID). Force a unilateral close after {unilateraltimeout} seconds (default 48h). If {destination} address is provided, will be used as output address.

delforward in_channel in_htlc_id status
    Delete a forwarded payment by [in_channel], [in_htlc_id] and [status]

fundchannel_cancel id
    Cancel inflight channel establishment with peer {id}.

fundchannel_complete id psbt
    Complete channel establishment with peer {id} for funding transactionwith {psbt}. Returns true on success, false otherwise.

fundchannel_start id amount [feerate] [announce] [close_to] [push_msat] [mindepth] [reserve]
    Start fund channel with {id} using {amount} satoshis. Returns a bech32 address to use as an output for a funding transaction.

listforwards [status] [in_channel] [out_channel]
    List all forwarded payments and their information optionally filtering by [status], [in_channel] and [out_channel]

listhtlcs [id]
    List all known HTLCS (optionally, just for [id] (scid or channel id))

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

recoverchannel scb
    Populate the DB with a channel and peerUsed for recovering the channel using DLP.This needs param in the form of an array [scb1,scb2,...]

setchannel id [feebase] [feeppm] [htlcmin] [htlcmax] [enforcedelay] [ignorefeelimits]
    Sets fees and/or htlc_max for channel with {id} (either peer ID, channel ID, short channel ID or 'all'). If {feebase}, {feeppm} or {htlcmax} is missing, it is unchanged.{base} can also be defined in other units, for example '1sat'. If {id} is 'all', the fees will be applied for all channels. 

setleaserates lease_fee_base_msat lease_fee_basis funding_weight channel_fee_max_base_msat channel_fee_max_proportional_thousandths
    Called by plugin to set the node's present channel lease rates. Not to be set without having a plugin which can handle `openchannel2` hooks.

splice_init channel_id relative_amount [initialpsbt] [feerate_per_kw] [force_feerate]
    Init a channel splice to {channel_id} for {relative_amount} satoshis with {initialpsbt}. Returns updated {psbt} with (partial) contributions from peer

splice_signed channel_id psbt [sign_first]
    Send our {signed_psbt}'s tx sigs for {channel_id}.

splice_update channel_id psbt
    Update {channel_id} currently active negotiated splice with {psbt}. Returns updated {psbt} with (partial) contributions from peer. If {commitments_secured} is true, next call may be to splicechannel_finalize, otherwise keep calling splice_update passing back in the returned PSBT until {commitments_secured} is true.

=== network ===

connect id [host] [port]
    Connect to {id} at {host} (which can end in ':port' if not default). {id} can also be of the form id@host

disconnect id [force]
    Disconnect from {id} that has previously been connected to using connect; with {force} set, even if it has a current channel

listclosedchannels [id]
    Show historical (dead) channels.

listpeerchannels [id]
    Show channels with direct peers.

listpeers [id] [level]
    Show current peers, if {level} is set, include logs for {id}

ping id [len] [pongbytes]
    Send peer {id} a ping of length {len} (default 128) asking for {pongbytes} (default 128)

=== payment ===

createinvoice invstring label preimage
    Lowlevel command to sign and create invoice {invstring}, resolved with {preimage}, using unique {label}.

createinvoicerequest bolt12 savetodb [exposeid] [recurrence_label] [single_use]
    Create and sign an invoice_request {bolt12}, with {recurrence_label} if recurring, filling in payer_info and payer_key.

createoffer bolt12 [label] [single_use]
    Create and sign an offer {bolt12} with and optional {label}.

createonion hops assocdata [session_key] [onion_size]
    Create an onion going through the provided nodes, each with its own payload

decodepay bolt11 [description]
    Decode {bolt11}, using {description} if necessary

delexpiredinvoice (DEPRECATED!) [maxexpirytime]
    Delete all expired invoices that expired as of given {maxexpirytime} (a UNIX epoch time), or all expired invoices if not specified

delinvoice label status [desconly]
    Delete unpaid invoice {label} with {status}

delpay payment_hash status [partid] [groupid]
    Delete payment with {payment_hash} and {status}

disableinvoicerequest invreq_id
    Disable invoice_request {invreq_id}

disableoffer offer_id
    Disable offer {offer_id}

invoice amount_msat label description [expiry] [fallbacks] [preimage] [exposeprivatechannels] [cltv] [deschashonly]
    Create an invoice for {msatoshi} with {label} and {description} with optional {expiry} seconds (default 1 week), optional {fallbacks} address list(default empty list) and optional {preimage} (default autogenerated)

listinvoicerequests [invreq_id] [active_only]
    If {invreq_id} is set, show that. Otherwise, if {showdisabled} is true, list all, otherwise just non-disabled ones.

listinvoices [label] [invstring] [payment_hash] [offer_id] [index] [start] [limit]
    Show invoice matching {label}, {invstring}, {payment_hash} or {offerid} (or all, if no query parameter specified)

listoffers [offer_id] [active_only]
    If {offer_id} is set, show that. Otherwise, if {showdisabled} is true, list all, otherwise just non-disabled ones.

listsendpays [bolt11] [payment_hash] [status]
    Show sendpay, old and current, optionally limiting to {bolt11} or {payment_hash}.

listtransactions 
    List transactions that we stored in the wallet

payersign messagename fieldname merkle tweak
    Sign {messagename} {fieldname} {merkle} (a 32-byte hex string) using public {tweak}

preapproveinvoice bolt11
    Ask the HSM to preapprove an invoice.

preapprovekeysend destination payment_hash amount_msat
    Ask the HSM to preapprove a keysend payment.

sendonion onion first_hop payment_hash [label] [shared_secrets] [partid] [bolt11] [amount_msat] [destination] [localinvreqid] [groupid] [description]
    Send a payment with a pre-computed onion.

sendpay route payment_hash [label] [amount_msat] [bolt11] [payment_secret] [partid] [localinvreqid] [groupid] [payment_metadata] [description]
    Send along {route} in return for preimage of {payment_hash}

signinvoice invstring
    Lowlevel command to sign invoice {invstring}.

waitanyinvoice [lastpay_index] [timeout]
    Wait for the next invoice to be paid, after {lastpay_index} (if supplied).  If {timeout} seconds is reached while waiting, fail with an error.

waitinvoice label
    Wait for an incoming payment matching the invoice with {label}, or if the invoice expires

waitsendpay payment_hash [timeout] [partid] [groupid]
    Wait for payment attempt on {payment_hash} to succeed or fail, but only up to {timeout} seconds.

=== plugin ===

autoclean-once subsystem age
    Perform a single run of autocleaning on one subsystem

autoclean-status [subsystem]
    Show status of autocleaning

autocleaninvoice (DEPRECATED!) [cycle_seconds] [expired_by]
    Set up autoclean of expired invoices. 

bkpr-channelsapy [start_time] [end_time]
    Stats on channel fund usage

bkpr-dumpincomecsv csv_format [csv_file] [consolidate_fees] [start_time] [end_time]
    Print out all the income events to a csv file in  {csv_format

bkpr-inspect account
    See the current on-chain graph of an {account}

bkpr-listaccountevents [account]
    List all events for an {account}

bkpr-listbalances 
    List current account balances

bkpr-listincome [consolidate_fees] [start_time] [end_time]
    List all income impacting events

commando peer_id method [params] [rune] [filter]
    Send a commando message to a direct peer, wait for response

commando-blacklist [start] [end]
    Blacklist a rune or range of runes by unique id

commando-listrunes [rune]
    List runes we have created earlier

commando-rune [rune] [restrictions]
    Create or restrict a rune

decode string
    Decode {string} message, returning {type} and information.

emergencyrecover 
    Populates the DB with stub channels

estimatefees 
    Get the urgent, normal and slow Bitcoin feerates as sat/kVB.

fundchannel id amount [feerate] [announce] [minconf] [utxos] [push_msat] [close_to] [request_amt] [compact_lease] [mindepth] [reserve]
    Fund channel with {id} using {amount} (or 'all'), at optional {feerate}. Only use outputs that have {minconf} confirmations.

funderupdate [policy] [policy_mod] [leases_only] [min_their_funding_msat] [max_their_funding_msat] [per_channel_min_msat] [per_channel_max_msat] [reserve_tank_msat] [fuzz_percent] [fund_probability] [lease_fee_base_msat] [lease_fee_basis] [funding_weight] [channel_fee_max_base_msat] [channel_fee_max_proportional_thousandths]
    Configuration for dual-funding settings.

getchaininfo last_height
    Get the chain id, the header count, the block count, and whether this is IBD.

getrawblockbyheight height
    Get the bitcoin block at a given height

getroute id amount_msat riskfactor [cltv] [fromid] [fuzzpercent] [exclude] [maxhops]
    Primitive route command

getutxout txid vout
    Get information about an output, identified by a {txid} an a {vout}

invoicerequest amount description [issuer] [label] [absolute_expiry] [single_use]
    Create an invoice_request to send money

keysend destination amount_msat [label] [maxfeepercent] [retry_for] [maxdelay] [exemptfee] [extratlvs] [routehints]
    Send a payment without an invoice to a node

listchannels [short_channel_id] [source] [destination]
    List all known channels in the network

listincoming 
    List the channels incoming from our direct peers

listnodes [id]
    List all known nodes in the network

listpays [bolt11] [payment_hash] [status]
    List result of payment {bolt11} or {payment_hash}, or all

listsqlschemas [table]
    Display schemas for internal sql tables, or just {table}

multifundchannel destinations [feerate] [minconf] [utxos] [minchannels] [commitment_feerate]
    Fund channels to {destinations}, which is an array of objects containing peer {id}, {amount}, and optional {announce} and {push_msat}.  A single transaction will be used to fund all the channels.  Use {feerate} for the transaction, select outputs that are buried {minconf} blocks deep, or specify a set of {utxos}.

multiwithdraw outputs [feerate] [minconf] [utxos]
    Send to multiple {outputs} via a single Bitcoin transaction.

offer amount description [issuer] [label] [quantity_max] [absolute_expiry] [recurrence] [recurrence_base] [recurrence_paywindow] [recurrence_limit] [single_use]
    Create an offer to accept money

pay bolt11 [amount_msat] [label] [riskfactor] [maxfeepercent] [retry_for] [maxdelay] [exemptfee] [localinvreqid] [exclude] [maxfee] [description]
    Send payment specified by {bolt11}

paystatus [bolt11]
    Detail status of attempts to pay {bolt11}, or all

plugin subcommand=start|stop|startdir|rescan|list
    Control plugins (start, stop, startdir, rescan, list)

renepay invstring [amount_msat] [maxfee] [maxdelay] [retry_for] [localofferid] [description] [label]
    Send payment specified by {invstring}

renepaystatus [invstring]
    Detail status of attempts to pay {bolt11}, or all

restorefrompeer 
    Checks if i have got a backup from a peer, and if so, will stub those channels in the database and if is successful, will return list of channels that have been successfully stubbed

sendrawtransaction tx allowhighfees
    Send a raw transaction to the Bitcoin network.

sql query
    Run {query} and return result

txdiscard txid
    Discard a transaction created by txprepare

txprepare outputs [feerate] [minconf] [utxos]
    Create a transaction, with option to spend in future (either txsend and txdiscard)

txsend txid
    Send a transaction created by txprepare

upgradewallet [feerate] [reservedok]
    Spend p2sh wrapped outputs into a native segwit output

withdraw destination satoshi [feerate] [minconf] [utxos]
    Send funds to {destination} address

=== utility ===

addgossip message
    Inject gossip {message} into gossipd

batching enable
    Database transaction batching {enable}

blacklistrune [start] [end]
    Blacklist a rune or range of runes by taking an optional {start} and an optional {end} and returns {blacklist} array containing {start}, {end}

blindedpath ids pathsecret
    Create blinded path to us along {ids} (pubkey array ending in our id)

check command_to_check
    Don't run {command_to_check}, just verify parameters.

checkmessage message zbase [pubkey]
    Verify a digital signature {zbase} of {message} signed with {pubkey}

checkrune rune [nodeid] [method] [params]
    Checks rune for validity with required {rune} and optional {nodeid}, {method}, {params} and returns {valid: true} or error message

createrune [rune] [restrictions]
    Create or restrict an optional {rune} with optional {restrictions} and returns {rune}

datastore key [string] [hex] [mode] [generation]
    Add a {key} and {hex}/{string} data to the data store

deldatastore key [generation]
    Remove a {key} from the data store

destroyrune [start] [end]
    Destroy a rune or range of runes by taking an optional {start} and an optional {end} and returns {blacklist} array containing {start}, {end}

dev-sendcustommsg node_id msg
    Send a custom message to the peer with the given {node_id}

getinfo 
    Show information about this node

getlog [level]
    Show logs, with optional log {level} (info|unusual|debug|io)

help [command]
    List available commands, or give verbose help on one {command}.

invokerune [rune] [restrictions]
    Invoke or restrict an optional {rune} with optional {restrictions} and returns {rune}

listconfigs [config]
    List all configuration options, or with [config], just that one.

listdatastore [key]
    List the datastore, optionally only {key}

listfunds [spent]
    Show available funds from the internal wallet

makesecret [hex] [string]
    Get a pseudorandom secret key, using some {hex} data.

notifications enable
    {enable} notifications

sendcustommsg node_id msg
    Send a custom message to the peer with the given {node_id}

sendonionmessage first_id blinding hops
    Send message to {first_id}, using {blinding}, encoded over {hops} (id, tlv)

setconfig config [val]
    Set a dynamically-adjustable config.

showrunes [rune]
    Show the list of runes or decode an optional {rune}.

signmessage message
    Create a digital signature of {message}

stop 
    Shut down the lightningd process

wait subsystem indexname nextvalue
    Wait for {subsystem} {indexname} to reach or exceed {value})

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

### devel

* `update_fulfill_htlc`を送信させない
  * [issue 366](https://github.com/ElementsProject/lightning/issues/366#issuecomment-346249070)
