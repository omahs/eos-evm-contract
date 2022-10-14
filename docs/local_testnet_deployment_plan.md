# Setting up a Local Antelope testnet with EVM support

Setting up a local testing environment of Antelope with EVM support allows developers to speed up their smart contract developments without worrying about any resource, network, version or other stabliliy issues that public testnet may introduce. Developers are free to modify, debug, or reset the environment to facilite their dApps developments.

## Hardware requirments

- CPU
  A high end CPU with good single threaded performance is recommended, such as i9 or Ryzen 9 or Server grade CPU. Middle or Low end CPU would also be OK but will end up having less transactions per second.
- RAM
  32GB+ is recommended. Using 16GB is OK but it can't support much data and compilation will be significantly slow
- SSD
  A big SSD is required for storing all history (blocks + State History). Recommend 1TB+. Using very small amount of storage like 100GB would still work fine but it will only support much fewer transactions / data.
- Network
  A low latency network is recommened if you plan to run multiple nodes. A simple network (or even WiFi) would works for single node.
  
  
## software requirements
- Operating System: Ubuntu 20.04 or 22.04

Have the following binaries built from https://github.com/AntelopeIO/leap
- Nodeos: the main process of an Antelope node
- Cleos: the command line interface for queries and transaction 
- keosd: the key and wallet manager.

Have the following binaries built from https://github.com/AntelopeIO/cdt
- cdt-cpp: the Antelope smart contract compiler
- eosio-wast2wasm & eosio-wasm2wast: conversion tools for building EVM contract

List of compiled system contracts from https://github.com/eosnetworkfoundation/eos-system-contracts (compiled by cdt):
- eosio.boot.wasm
- eosio.bios.wasm
- eosio.msig.wasm (optional, if you want to test multisig)
- eosio.token.wasm (optional, if you want to test token econonmy)
- eosio.system.wasm (optional, if you want to test resources, RAM, ... etc)

Compiled EVM contracts in DEBUG mode, from this repo (see https://github.com/eosnetworkfoundation/TrustEVM/blob/main/docs/compilation_and_testing_guide.md)
- evm_runtime.wasm
- evm_runtime.abi

<b> Ensure action "setbal" exists in evm_runtime.abi </b>

Compiled binaries from this repo
- trustevm-node: (silkworm node process that receive data from the main Antelope chain and convert to the EVM chain)
- trustevm-rpc: (silkworm rpc server that provide service for view actions and other read operations)


  
## Running a local node with Trust EVM service, Overview:

  In order to run a Trust EVM service, we need to have the follow items inside one physical server / VM:
  1. run a local Antelope node (nodeos process) with SHIP plugin enabled, which is a single block producer
  2. blockchain bootstrapping and initialization
  3. deploy evm contract and initilize evm
  4. run a TrustEVM-node(silkworm node) process connecting to the local Antelope node
  5. run a TrustEVM-RPC(silkworm RPC) process locally to serve the eth RPC requests
  6. setup the transaction wrapper for write transactions
  7. setup a trustEVM proxy to route read requests to TrustEVM-RPC and write requests to Antelope public network
  - prepare a eosio account with necessary CPU/NET/RAM resources for signing EVM transactions (this account can be shared to multiple EVM service)
  

## Step by Step in details:

## 1. Run a local Antelope node 

make a data-dir directory:
```
mkdir data-dir
```
prepare the genesis file in ./data-dir/genesis.json, for example
```
{
  "initial_key": "EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV",
  "initial_timestamp": "2022-01-01T00:00:00",
  "initial_parameters": {
  },
  "initial_configuration": {
  }
}
```
In this case the initial genesis public key - private key pair is "EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV"/"5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3"
You can you any other key pair as the genesis key.

prepare the config file in ./data-dir/config.ini, for example
```
chain-state-db-size-mb = 16384

# Track only transactions whose scopes involve the listed accounts. Default is to track all transactions.
# filter_on_accounts = 

# override the initial timestamp in the Genesis State file
# genesis-timestamp = 


# Pairs of [BLOCK_NUM,BLOCK_ID] that should be enforced as checkpoints.
# checkpoint = 


# The local IP and port to listen for incoming http connections.
http-server-address = 127.0.0.1:8888

# Specify the Access-Control-Allow-Origin to be returned on each request.
# access-control-allow-origin = 

# Specify the Access-Control-Allow-Headers to be returned on each request.
# access-control-allow-headers = 

# Specify if Access-Control-Allow-Credentials: true should be returned on each request.
access-control-allow-credentials = false

# The actual host:port used to listen for incoming p2p connections.
p2p-listen-endpoint = 0.0.0.0:9876

# An externally accessible host:port for identifying this node. Defaults to p2p-listen-endpoint.
# p2p-server-address = 

p2p-max-nodes-per-host = 10

# The public endpoint of a peer node to connect to. Use multiple p2p-peer-address options as needed to compose a network.
# p2p-peer-address = 

# The name supplied to identify this node amongst the peers.
agent-name = "EOS Test Agent"

# Can be 'any' or 'producers' or 'specified' or 'none'. If 'specified', peer-key must be specified at least once. If only 'producers', peer-key is not required. 'producers' and 'specified' may be combined.
allowed-connection = any

# Optional public key of peer allowed to connect.  May be used multiple times.
#peer-key = "EOS5RCMdVJ8JuzxKSbWArbYUGTVcMBc4FVpsqT9qHGYkvnHUrKnrg"

# Tuple of [PublicKey, WIF private key] (may specify multiple times)
peer-private-key = ["EOS7ZRw8XrYEk5AJgJnL4C8d6pYJg8GH76gobfLYtixFwWh763GSC", "5JVoee8UEMgoYAbNoJqWUTonpSeThLueatQBCqC2JXU3fCUMebj"]

# Maximum number of clients from which connections are accepted, use 0 for no limit
max-clients = 25

# number of seconds to wait before cleaning up dead connections
connection-cleanup-period = 30

# True to require exact match of peer network version.
# network-version-match = 0

# number of blocks to retrieve in a chunk from any individual peer during synchronization
sync-fetch-span = 100

# Enable block production, even if the chain is stale.
enable-stale-production = true

# ID of producer controlled by this node (e.g. inita; may specify multiple times)
producer-name = eosio

# Tuple of [public key, WIF private key] for block production (may specify multiple times)
private-key = ["EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV","5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3"]
private-key = ["EOS8kE63z4NcZatvVWY4jxYdtLg6UEA123raMGwS6QDKwpQ69eGcP","5JURSKS1BrJ1TagNBw1uVSzTQL2m9eHGkjknWeZkjSt33Awtior"]
private-key = ["EOS7mGPTufZzCw1GxobnS9qkbdBkV7ajd1Apz9SmxXmPnKyxw2g6u","5KXMcXZjwC69uTHyYKF5vtFP8NLyq8ZaNrbjVc5HoMKBsH3My4D"]
private-key = ["EOS7RaMheR2Fw4VYBj9dj6Vv7jMyV54NrdmgTe7GRkosEPxCTtTPR","5Jo1cABt1KLCTmYmePetiU8A5uDKVZGM44PgvQ4SiJVo2gA9Son"]
private-key = ["EOS5sUpxhaC5V231cAVxGVH9RXtN9n4KDxZG6ZUwHRgYoEpTBUidU","5JK68f7PifEtGhN2T4xK9mMxCrtYLmPp6cNdKSSYJmTJqCFhNVX"]

# state-history
trace-history = true
chain-state-history = true

state-history-endpoint = 127.0.0.1:8999

# Plugin(s) to enable, may be specified multiple times
plugin = eosio::producer_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::http_plugin
plugin = eosio::txn_test_gen_plugin
plugin = eosio::producer_api_plugin
plugin = eosio::state_history_plugin
plugin = eosio::net_plugin
plugin = eosio::net_api_plugin

```
startup Antelope node:
```
./build/programs/nodeos/nodeos --data-dir=./data-dir  --config-dir=./data-dir --genesis-json=./data-dir/genesis.json --disable-replay-opts --contracts-console
```
you will see the node is started and blocks are produced, for example:

```
info  2022-10-14T04:03:19.911 nodeos    producer_plugin.cpp:2437      produce_block        ] Produced block 12ef38e0bcf48b35... #2 @ 2022-10-14T04:03:20.000 signed by eosio [trxs: 0, lib: 1, confirmed: 0]
info  2022-10-14T04:03:20.401 nodeos    producer_plugin.cpp:2437      produce_block        ] Produced block df3ab0d68f1d0aaf... #3 @ 2022-10-14T04:03:20.500 signed by eosio [trxs: 0, lib: 2, confirmed: 0]
```

If you want to start by discarding all previous blockchain data, add --delete-all-blocks:
```
./build/programs/nodeos/nodeos --data-dir=./data-dir  --config-dir=./data-dir --genesis-json=./data-dir/genesis.json --disable-replay-opts --contracts-console --delete-all-blocks
```

If you want to start with the previous blockchain data, but encounter the "dirty flag" error, try to restart with --hard-replay (in this case the state will be disgarded, the node will validate and apply every block from the beginning.
```
./build/programs/nodeos/nodeos --data-dir=./data-dir  --config-dir=./data-dir --genesis-json=./data-dir/genesis.json --disable-replay-opts --contracts-console --hard-replay
```


## 2. Blockchain bootstrapping and initialization

You can find the command line interface "cleos" in ./build/programs/cleos/cleos

try the following command:
```
./cleos get info
```
If you can get the similar response as follow:
```
{
  "server_version": "f36e59e5",
  "chain_id": "4a920ae9b3b9c99e79542834f2332201d9393adfca26cdcca10aa3fd4a3dc68d",
  "head_block_num": 362,
  "last_irreversible_block_num": 361,
  "last_irreversible_block_id": "0000016993d6b2d1ea8f0602cea8d94690f533f90555cddc7e66f36e08d3fd53",
  "head_block_id": "0000016a079b188ba800b37afbbfb5b0dae543008e7d2daf563f9a59e9127ca1",
  "head_block_time": "2022-10-14T06:06:53.500",
  "head_block_producer": "eosio",
  "virtual_block_cpu_limit": 286788,
  "virtual_block_net_limit": 1504517,
  "block_cpu_limit": 199900,
  "block_net_limit": 1048576,
  "server_version_string": "v3.1.0",
  "fork_db_head_block_num": 362,
  "fork_db_head_block_id": "0000016a079b188ba800b37afbbfb5b0dae543008e7d2daf563f9a59e9127ca1",
  "server_full_version_string": "v3.1.0-f36e59e554e8e5687e705e32e9f8aea4a39ed213",
  "total_cpu_weight": 0,
  "total_net_weight": 0,
  "earliest_available_block_num": 1,
  "last_irreversible_block_time": "2022-10-14T06:06:53.000"
}
```
It means your local node is running fine and cleos has successfully communicated with nodeos.

Generate some public/private key pairs for testing, command:
```
./cleos create key --to-console
```
You will get similar output as follow:
```
Private key: 5Ki7JeCMXQmxreZnL2JubQEYuqByehbPhKGUXyxqo6RYGNS2F3i
Public key: EOS8j4zHDrqRrf84QJLDTEgUbhB24VUkqVCjAxMzmeJuJSvZ8S1FU
```
Repeat the same command to generate multiple key pairs. Save your key pairs for later testing use.

create your new wallet named w123 (any other name is fine)
```
./build/programs/cleos/cleos wallet create -n w123 --file w123.key
```
your wallet password is saved into w123.key

import one or more private keys into wallet w123
```
./cleos wallet import -n w123 --private-key 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
./cleos wallet import -n w123 --private-key 5JURSKS1BrJ1TagNBw1uVSzTQL2m9eHGkjknWeZkjSt33Awtior
```

Once you have done everything with the wallet, it is fine to bootstrapping the blockchain

### activate protocol features:

First we need to use curl to schedule protocol feature activation:
```
curl --data-binary '{"protocol_features_to_activate":["0ec7e080177b2c02b278d5088611686b49d739925a92d9bfcacd7fc6b74053bd"]}' http://127.0.0.1:8888/v1/producer/schedule_protocol_feature_activations
```

You'll get the "OK" response if succeed:
```
{"result":"ok"}
```

### deploy boot contract, command:
```
./cleos set code eosio ../eos-system-contracts/build/contracts/eosio.boot/eosio.boot.wasm
```
output:
```
Reading WASM from /home/kayan-u20/workspaces/leap/../eos-system-contracts/build/contracts/eosio.boot/eosio.boot.wasm...
Setting Code...
executed transaction: acaf5ed70a7ce271627532cf76b6303ebab8d24656f57c69b03cfe8103f6f457  2120 bytes  531 us
#         eosio <= eosio::setcode               {"account":"eosio","vmtype":0,"vmversion":0,"code":"0061736d0100000001480e60000060027f7f0060017e0060...
warning: transaction executed locally, but may not be confirmed by the network yetult         ] 
```

set boot.abi, command:
```
./cleos set abi eosio ../eos-system-contracts/build/contracts/eosio.boot/eosio.boot.abi
```
output
```
Setting ABI...
executed transaction: b972e178d182c1523e9abbd1fae27efae90d7711e152261a21169372a19d9d3a  1528 bytes  171 us
#         eosio <= eosio::setabi                {"account":"eosio","abi":"0e656f73696f3a3a6162692f312e32001008616374697661746500010e666561747572655f...
warning: transaction executed locally, but may not be confirmed by the network yetult         ] 
```

activate the other protocol features:
```
./cleos push action eosio activate '["f0af56d2c5a48d60a4a5b5c903edfb7db3a736a94ed589d0b797df33ff9d3e1d"]' -p eosio 
./cleos push action eosio activate '["e0fb64b1085cc5538970158d05a009c24e276fb94e1a0bf6a528b48fbc4ff526"]' -p eosio
./cleos push action eosio activate '["d528b9f6e9693f45ed277af93474fd473ce7d831dae2180cca35d907bd10cb40"]' -p eosio
./cleos push action eosio activate '["c3a6138c5061cf291310887c0b5c71fcaffeab90d5deb50d3b9e687cead45071"]' -p eosio 
./cleos push action eosio activate '["bcd2a26394b36614fd4894241d3c451ab0f6fd110958c3423073621a70826e99"]' -p eosio
./cleos push action eosio activate '["ad9e3d8f650687709fd68f4b90b41f7d825a365b02c23a636cef88ac2ac00c43"]' -p eosio
./cleos push action eosio activate '["8ba52fe7a3956c5cd3a656a3174b931d3bb2abb45578befc59f283ecd816a405"]' -p eosio 
./cleos push action eosio activate '["6bcb40a24e49c26d0a60513b6aeb8551d264e4717f306b81a37a5afb3b47cedc"]' -p eosio
./cleos push action eosio activate '["68dcaa34c0517d19666e6b33add67351d8c5f69e999ca1e37931bc410a297428"]' -p eosio
./cleos push action eosio activate '["5443fcf88330c586bc0e5f3dee10e7f63c76c00249c87fe4fbf7f38c082006b4"]' -p eosio
./cleos push action eosio activate '["4fca8bd82bbd181e714e283f83e1b45d95ca5af40fb89ad3977b653c448f78c2"]' -p eosio
./cleos push action eosio activate '["ef43112c6543b88db2283a2e077278c315ae2c84719a8b25f25cc88565fbea99"]' -p eosio
./cleos push action eosio activate '["4a90c00d55454dc5b059055ca213579c6ea856967712a56017487886a4d4cc0f"]' -p eosio 
./cleos push action eosio activate '["35c2186cc36f7bb4aeaf4487b36e57039ccf45a9136aa856a5d569ecca55ef2b"]' -p eosio
./cleos push action eosio activate '["299dcb6af692324b899b39f16d5a530a33062804e41f09dc97e9f156b4476707"]' -p eosio
./cleos push action eosio activate '["2652f5f96006294109b3dd0bbde63693f55324af452b799ee137a81a905eed25"]' -p eosio
./cleos push action eosio activate '["1a99a59d87e06e09ec5b028a9cbb7749b4a5ad8819004365d02dc4379a8b7241"]' -p eosio
```



## 3. Deploy and initialize EVM contract

Create account evmevmevmevm (using key pair EOS8kE63z4NcZatvVWY4jxYdtLg6UEA123raMGwS6QDKwpQ69eGcP 5JURSKS1BrJ1TagNBw1uVSzTQL2m9eHGkjknWeZkjSt33Awtior)
```
./cleos create account eosio evmevmevmevm EOS8kE63z4NcZatvVWY4jxYdtLg6UEA123raMGwS6QDKwpQ69eGcP EOS8kE63z4NcZatvVWY4jxYdtLg6UEA123raMGwS6QDKwpQ69eGcP
```

deploy evm_runtime contract (wasm & abi file) to account evmevmevmevm
```
./cleos set code evmevmevmevm ../TrustEVM/contract/build/evm_runtime/evm_runtime.wasm
./cleos set abi evmevmevmevm ../TrustEVM/contract/build/evm_runtime/evm_runtime.abi
```

setting initial balance for testing eth account 0x2787b98fc4e731d0456b3941f0b3fe2e01439961, private key a3f1b69da92a0233ce29485d3049a4ace39e8d384bbc2557e3fc60940ce4e954

```
./cleos push action evmevmevmevm setbal '{"addy":"2787b98fc4e731d0456b3941f0b3fe2e01439961", "bal":"0000000000000000000000000000000100000000000000000000000000000000"}' -p evmevmevmevm
```


## 4. Start up TrustEVM-node (silkworm node) 

## 5. Start up TrustEVM-RPC (silkworm RPC)

## 6. Setup transaction wrapper

## 7. Setup proxy 



### configuration and command parameters:
- nodeos:
- TrustEVM-node:
- TrustEVM-RPC




## Bootstrapping protocol features, Deploying EVM contracts, Setup genesis/initial EVM virtual accounts & tokens

### Protocol features required to activate:
- ACTION_RETURN_VALUE
- CRYPTO_PRIMITIVES
- GET_BLOCK_NUM

### Create main EVM account & Deploy EVM contract to Antelope blockchain


### Setup EVM token


### disturbute EVM tokens according to genesis

