# API Reference C


## Overview

 The C implementation of the Incubed client is prepared and optimized to run on small embedded devices. Because each device is different, we prepare different modules that should be combined. This allows us to only generate the code needed and reduce requirements for flash and memory.

### Why C?

We have been asked a lot, why we implemented Incubed in C and not in Rust. When we started Incubed we began with a feasibility test and wrote the client in TypeScript. Once we confirmed it was working, we wanted to provide a minimal verifaction client for embedded devices. And yes, we actually wanted to do it in Rust, since Rust offers a lot of safety-features (like the memory-management at compiletime, thread-safety, ...), but after considering a lot of different aspects we made a pragmatic desicion to use C.

These are the reasons why:

#### Support for embedded devices.

As of today almost all toolchain used in the embedded world are build for C. Even though Rust may be able to still use some, there are a lot of issues. Quote from [rust-embedded.org](https://docs.rust-embedded.org/book/interoperability/#interoperability-with-rtoss):

*Integrating Rust with an RTOS such as FreeRTOS or ChibiOS is still a work in progress; especially calling RTOS functions from Rust can be tricky.*

This may change in the future, but C is so dominant, that chances of Rust taking over the embedded development completly is low.

#### Portability

C is the most portable programming language. Rust actually has a pretty admirable selection of supported targets for a new language (thanks mostly to LLVM), but it pales in comparison to C, which runs on almost everything. A new CPU architecture or operating system can barely be considered to exist until it has a C compiler. And once it does, it unlocks access to a vast repository of software written in C. Many other programming languages, such as Ruby and Python, are implemented in C and you get those for free too.

Most programing language have very good support for calling c-function in a shared library (like ctypes in python or cgo in golang) or even support integration of C code directly like [android studio](https://developer.android.com/studio/projects/add-native-code) does.

#### Integration in existing projects

Since especially embedded systems are usually written in C/C++, offering a pure C-Implementation makes it easy for these projects to use Incubed, since they do not have to change their toolchain.

Even though we may not be able to use a lot of great features Rust offers by going with C, it allows to reach the goal to easily integrate with a lot of projects. For the future we might port the incubed to Rust if we see a demand or chance for the same support as C has today.

### Modules

Incubed consists of different modules. While the core module is always required, additional functions will be prepared by different modules.

```eval_rst
.. graphviz::

    digraph "GG" {
        graph [ rankdir = "RL" ]
        node [
          fontsize = "12"
          fontname="Helvetica"
          shape="ellipse"
        ];
    
        subgraph cluster_transport {
            label="Transports"  color=lightblue  style=filled
            transport_http;
            transport_curl;
    
        }
    
    
        evm;
        tommath;
    
        subgraph cluster_verifier {
            label="Verifiers"  color=lightblue  style=filled
            eth_basic;
            eth_full;
            eth_nano;
            btc;
        }
        subgraph cluster_bindings {
            label="Bindings"  color=lightblue  style=filled
            wasm;
            java;
            python;
    
        }
        subgraph cluster_api {
            label="APIs"  color=lightblue  style=filled
            eth_api;
            usn_api;
    
        }
    
        core;
        segger_rtt;
        crypto;
        core -> segger_rtt;
        core -> crypto // core -> crypto
        eth_api -> eth_nano // eth_api -> eth_nano
        btc_api -> btc // eth_api -> eth_nano
        eth_nano -> core // eth_nano -> core
        btc -> core // eth_nano -> core
        eth_basic -> eth_nano // eth_basic -> eth_nano
        eth_full -> evm // eth_full -> evm
        evm -> eth_basic // evm -> eth_basic
        evm -> tommath // evm -> tommath
        transport_http -> core // transport_http -> core
        transport_curl -> core // transport_http -> core
        usn_api -> core // usn_api -> core
    
        java -> core // usn_api -> core
        python -> core // usn_api -> core
        wasm -> core // usn_api -> core
    }
    
```
#### Verifier

Incubed is a minimal verification client, which means that each response needs to be verifiable. Depending on the expected requests and responses, you need to carefully choose which verifier you may need to register. For Ethereum, we have developed three modules:

1. [eth_nano](#eth-nano-h): a minimal module only able to verify transaction receipts (`eth_getTransactionReceipt`).
2. [eth_basic](#eth-basic-h): module able to verify almost all other standard RPC functions (except `eth_call`).
3. [eth_full](#eth-full-h): module able to verify standard RPC functions. It also implements a full EVM to handle `eth_call`.

1. [btc](#btc-h): module able to verify bitcoin or bitcoin based chains.
2. [ipfs](#ipfs-h): module able to verify ipfs-hashes

Depending on the module, you need to register the verifier before using it. This is done by calling the `in3_register...` function like [in3_register_eth_full()](#in3-register-eth-full).

#### Transport

To verify responses, you need to be able to send requests. The way to handle them depends heavily on your hardware capabilities. For example, if your device only supports Bluetooth, you may use this connection to deliver the request to a device with an existing internet connection and get the response in the same way, but if your device is able to use a direct internet connection, you may use a curl-library to execute them. This is why the core client only defines function pointer [in3_transport_send](#in3-transport-send), which must handle the requests.

At the moment we offer these modules; other implementations are supported by different hardware modules.

1. [transport_curl](#in3-curl-h): module with a dependency on curl, which executes these requests and supports HTTPS. This module runs a standard OS with curl installed.
2. [transport_http](#in3-http-h): module with no dependency, but a very basic http-implementation (no https-support)

#### API

While Incubed operates on JSON-RPC level, as a developer, you might want to use a better-structured API to prepare these requests for you. These APIs are optional but make life easier:

1. [**eth**](#eth-api-h): This module offers all standard RPC functions as described in the [Ethereum JSON-RPC Specification](https://github.com/ethereum/wiki/wiki/JSON-RPC). In addition, it allows you to sign and encode/decode calls and transactions.
2. [**usn**](#usn-api-h): This module offers basic USN functions like renting, event handling, and message verification.
3. [**btc**](#btc-api-h): Collection of Bitcoin-functions to access blocks and transactions.
4. [**ipfs**](#ipfs-api-h): Simple Ipfs-functions to get and store ipfs-content




## Building

While we provide binaries, you can also build from source:

### requirements

- cmake
- curl : curl is used as transport for command-line tools, but you can also compile it without curl (`-DUSE_CURL=false -DCMD=false`), if you want to implement your own transport.

Incubed uses cmake for configuring:

```sh
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && make
make install
```
### CMake options

When configuring cmake, you can set a lot of different incubed specific like `cmake -DEVM_GAS=false ..`.

#### ASMJS

compiles the code as asm.js.

Default-Value: `-DASMJS=OFF`

#### ASSERTIONS

includes assertions into the code, which help track errors but may cost time during runtime

Default-Value: `-DASSERTIONS=OFF`

#### BTC

if true, the bitcoin verifiers will be build

Default-Value: `-DBTC=ON`

#### BTC_PRE_BPI34

Enable BTC-Verfification for blocks before BIP34 was activated

Default-Value: `-DBTC_PRE_BPI34=ON`

#### BUILD_DOC

generates the documenation with doxygen.

Default-Value: `-DBUILD_DOC=OFF`

#### CMD

build the comandline utils

Default-Value: `-DCMD=ON`

#### CODE_COVERAGE

Builds targets with code coverage instrumentation. (Requires GCC or Clang)

Default-Value: `-DCODE_COVERAGE=OFF`

#### COLOR

Enable color codes for debug

Default-Value: `-DCOLOR=ON`

#### CORE_API

registers a chain independend rpc-methods util-functions

Default-Value: `-DCORE_API=ON`

#### ESP_IDF

include support for ESP-IDF microcontroller framework

Default-Value: `-DESP_IDF=OFF`

#### ETH_BASIC

build basic eth verification.(all rpc-calls except eth_call)

Default-Value: `-DETH_BASIC=ON`

#### ETH_FULL

build full eth verification.(including eth_call)

Default-Value: `-DETH_FULL=ON`

#### ETH_NANO

build minimal eth verification.(eth_getTransactionReceipt)

Default-Value: `-DETH_NANO=ON`

#### EVM_GAS

if true the gas costs are verified when validating a eth_call. This is a optimization since most calls are only interessted in the result. EVM_GAS would be required if the contract uses gas-dependend op-codes.

Default-Value: `-DEVM_GAS=ON`

#### FAST_MATH

Math optimizations used in the EVM. This will also increase the filesize.

Default-Value: `-DFAST_MATH=OFF`

#### GCC_ANALYZER

GCC10 static code analyses

Default-Value: `-DGCC_ANALYZER=OFF`

#### IN3API

build the USN-API which offer better interfaces and additional functions on top of the pure verification

Default-Value: `-DIN3API=ON`

#### IN3_LIB

if true a shared anmd static library with all in3-modules will be build.

Default-Value: `-DIN3_LIB=ON`

#### IN3_SERVER

support for proxy server as part of the cmd-tool, which allows to start the cmd-tool with the -p option and listens to the given port for rpc-requests

Default-Value: `-DIN3_SERVER=OFF`

#### IN3_STAGING

if true, the client will use the staging-network instead of the live ones

Default-Value: `-DIN3_STAGING=OFF`

#### IPFS

build IPFS verification

Default-Value: `-DIPFS=ON`

#### JAVA

build the java-binding (shared-lib and jar-file)

Default-Value: `-DJAVA=OFF`

#### LEDGER_NANO

include support for nano ledger

Default-Value: `-DLEDGER_NANO=OFF`

#### LOGGING

if set logging and human readable error messages will be inculded in th executable, otherwise only the error code is used. (saves about 19kB)

Default-Value: `-DLOGGING=ON`

#### MULTISIG

add capapbility to sign with a multig. Currrently only gnosis safe is supported

Default-Value: `-DMULTISIG=ON`

#### NODESELECT_DEF

Enable default nodeselect implementation

Default-Value: `-DNODESELECT_DEF=ON`

#### NODESELECT_DEF_WL

Enable default nodeselect whitelist implementation

Default-Value: `-DNODESELECT_DEF_WL=ON`

#### PAY_ETH

support for direct Eth-Payment

Default-Value: `-DPAY_ETH=OFF`

#### PKG_CONFIG_EXECUTABLE

pkg-config executable

Default-Value: `-DPKG_CONFIG_EXECUTABLE=/opt/homebrew/bin/pkg-config`

#### PK_SIGNER

Enable Signing with private keys

Default-Value: `-DPK_SIGNER=ON`

#### PLGN_CLIENT_DATA

Enable client-data plugin

Default-Value: `-DPLGN_CLIENT_DATA=OFF`

#### POA

support POA verification including validatorlist updates

Default-Value: `-DPOA=OFF`

#### RECORDER

enable recording option for reproduce executions

Default-Value: `-DRECORDER=ON`

#### RPC_ONLY

specifies a coma-seperqted list of rpc-methods which should be supported. all other rpc-methods will be removed reducing the size of executable a lot.

Default-Value: `-DRPC_ONLY=OFF`

#### SEGGER_RTT

Use the segger real time transfer terminal as the logging mechanism

Default-Value: `-DSEGGER_RTT=OFF`

#### SENTRY

Enable Sentry

Default-Value: `-DSENTRY=OFF`

#### SWIFT

swift API for swift bindings

Default-Value: `-DSWIFT=OFF`

#### TAG_VERSION

the tagged version, which should be used

Default-Value: `-DTAG_VERSION=OFF`

#### TEST

builds the tests and also adds special memory-management, which detects memory leaks, but will cause slower performance

Default-Value: `-DTEST=OFF`

#### THREADSAFE

uses mutex to protect shared nodelist access

Default-Value: `-DTHREADSAFE=ON`

#### TRANSPORTS

builds transports, which may require extra libraries.

Default-Value: `-DTRANSPORTS=ON`

#### USE_CURL

if true the curl transport will be built (with a dependency to libcurl)

Default-Value: `-DUSE_CURL=ON`

#### USE_PRECOMPUTED_EC

if true the secp256k1 curve uses precompiled tables to boost performance. turning this off makes ecrecover slower, but saves about 37kb.

Default-Value: `-DUSE_PRECOMPUTED_EC=ON`

#### USE_SCRYPT

integrate scrypt into the build in order to allow decrypt_key for scrypt encoded keys.

Default-Value: `-DUSE_SCRYPT=ON`

#### USE_WINHTTP

if true the winhttp transport will be built (with a dependency to winhttp)

Default-Value: `-DUSE_WINHTTP=OFF`

#### WASM

Includes the WASM-Build. In order to build it you need emscripten as toolchain. Usually you also want to turn off other builds in this case.

Default-Value: `-DWASM=OFF`

#### WASM_EMBED

embedds the wasm as base64-encoded into the js-file

Default-Value: `-DWASM_EMBED=ON`

#### WASM_EMMALLOC

use ther smaller EMSCRIPTEN Malloc, which reduces the size about 10k, but may be a bit slower

Default-Value: `-DWASM_EMMALLOC=ON`

#### WASM_SYNC

intiaializes the WASM synchronisly, which allows to require and use it the same function, but this will not be supported by chrome (4k limit)

Default-Value: `-DWASM_SYNC=OFF`

#### ZKCRYPTO_LIB

Path to the static zkcrypto-lib

Default-Value: `-DZKCRYPTO_LIB=OFF`

#### ZKSYNC

add RPC-function to handle zksync-payments

Default-Value: `-DZKSYNC=ON` 




## Examples

### btc_transaction

source : [in3-c/c/examples/btc_transaction.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/btc_transaction.c)

checking a Bitcoin transaction data

```c

#include <in3/btc_api.h>  // we need the btc-api
#include <in3/client.h>   // the core client
#include <in3/in3_init.h> // this header will make sure we initialize the default verifiers and transports
#include <in3/utils.h>    // helper functions
#include <stdio.h>

int main() {
  // create new incubed client for BTC
  in3_t* in3 = in3_for_chain(CHAIN_ID_BTC);

  // the hash of transaction that we want to get
  bytes32_t tx_id;
  hex_to_bytes("c41eee1c2d97f6158ea3b3aeba0a5271a2174067a38d089ccc1eefbc796706e0", -1, tx_id, 32);

  // fetch and verify the transaction
  btc_transaction_t* tx = btc_get_transaction(in3, tx_id);

  if (!tx)
    // if the result is null there was an error an we can get the latest error message from btc_last_error()
    printf("error getting the tx : %s\n", btc_last_error());
  else {
    // we loop through the tx outputs
    for (int i = 0; i < tx->vout_len; i++)
      // and prrint the values
      printf("Transaction vout #%d : value: %llu\n", i, tx->vout[i].value);

    // don't forget the clean up!
    free(tx);
  }

  // cleanup client after usage
  in3_free(in3);
}
```
### call_a_function

source : [in3-c/c/examples/call_a_function.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/call_a_function.c)

This example shows how to call functions on a smart contract eiither directly or using the api to encode the arguments

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // functions for direct api-access
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/log.h>      // logging functions
#include <inttypes.h>
#include <stdio.h>

static in3_ret_t call_func_rpc(in3_t* c);
static in3_ret_t call_func_api(in3_t* c, address_t contract);

int main() {
  in3_ret_t ret = IN3_OK;

  // Remove log prefix for readability
  in3_log_set_prefix("");

  // create new incubed client
  in3_t* c = in3_for_chain(CHAIN_ID_MAINNET);

  // define a address (20byte)
  address_t contract;

  // copy the hexcoded string into this address
  hex_to_bytes("0x2736D225f85740f42D17987100dc8d58e9e16252", -1, contract, 20);

  // call function using RPC
  ret = call_func_rpc(c);
  if (ret != IN3_OK) goto END;

  // call function using API
  ret = call_func_api(c, contract);
  if (ret != IN3_OK) goto END;

END:
  // clean up
  in3_free(c);
  return 0;
}

in3_ret_t call_func_rpc(in3_t* c) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      c,                                                                                                //  the configured client
      "eth_call",                                                                                       // the rpc-method you want to call.
      "[{\"to\":\"0x2736d225f85740f42d17987100dc8d58e9e16252\", \"data\":\"0x15625c5e\"}, \"latest\"]", // the signed raw txn, same as the one used in the API example
      &result,                                                                                          // the reference to a pointer which will hold the result
      &error);                                                                                          // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Result: \n%s\n", result);
    free(result);
    return 0;
  } else {
    printf("Error sending tx: \n%s\n", error);
    free(error);
    return IN3_EUNKNOWN;
  }
}

in3_ret_t call_func_api(in3_t* c, address_t contract) {
  // ask for the number of servers registered
  json_ctx_t* response = eth_call_fn(c, contract, BLKNUM_LATEST(), "totalServers():uint256");
  if (!response) {
    printf("Could not get the response: %s", eth_last_error());
    return IN3_EUNKNOWN;
  }

  // convert the response to a uint32_t,
  uint32_t number_of_servers = d_int(response->result);

  // clean up resources
  json_free(response);

  // output
  printf("Found %u servers registered : \n", number_of_servers);

  // read all structs ...
  for (uint32_t i = 0; i < number_of_servers; i++) {
    response = eth_call_fn(c, contract, BLKNUM_LATEST(), "servers(uint256):(string,address,uint,uint,uint,address)", to_uint256(i));
    if (!response) {
      printf("Could not get the response: %s", eth_last_error());
      return IN3_EUNKNOWN;
    }

    char*    url     = d_get_string_at(response->result, 0); // get the first item of the result (the url)
    bytes_t* owner   = d_get_bytes_at(response->result, 1);  // get the second item of the result (the owner)
    uint64_t deposit = d_get_long_at(response->result, 2);   // get the third item of the result (the deposit)

    printf("Server %i : %s owner = %02x%02x...", i, url, owner->data[0], owner->data[1]);
    printf(", deposit = %" PRIu64 "\n", deposit);

    // free memory
    json_free(response);
  }
  return 0;
}
```
### get_balance

source : [in3-c/c/examples/get_balance.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/get_balance.c)

get the Balance with the API and also as direct RPC-call

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // functions for direct api-access
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/log.h>      // logging functions
#include <in3/utils.h>
#include <stdio.h>

static void get_balance_rpc(in3_t* in3);
static void get_balance_api(in3_t* in3);

int main() {
  // create new incubed client
  in3_t* in3 = in3_for_chain(CHAIN_ID_MAINNET);

  // get balance using raw RPC call
  get_balance_rpc(in3);

  // get balance using API
  get_balance_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_balance_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                                                            //  the configured client
      "eth_getBalance",                                               // the rpc-method you want to call.
      "[\"0xc94770007dda54cF92009BFF0dE90c06F603a09f\", \"latest\"]", // the arguments as json-string
      &result,                                                        // the reference to a pointer whill hold the result
      &error);                                                        // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Balance: \n%s\n", result);
    free(result);
  } else {
    printf("Error getting balance: \n%s\n", error);
    free(error);
  }
}

void get_balance_api(in3_t* in3) {
  // the address of account whose balance we want to get
  address_t account;
  hex_to_bytes("0xc94770007dda54cF92009BFF0dE90c06F603a09f", -1, account, 20);

  // get balance of account
  long double balance = as_double(eth_getBalance(in3, account, BLKNUM_EARLIEST()));

  // if the result is null there was an error an we can get the latest error message from eth_lat_error()
  balance ? printf("Balance: %Lf\n", balance) : printf("error getting the balance : %s\n", eth_last_error());
}
```
### get_block

source : [in3-c/c/examples/get_block.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/get_block.c)

using the basic-module to get and verify a Block with the API and also as direct RPC-call

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // functions for direct api-access
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/log.h>      // logging functions

#include <inttypes.h>
#include <stdio.h>

static void get_block_rpc(in3_t* in3);
static void get_block_api(in3_t* in3);

int main() {
  // create new incubed client
  in3_t* in3 = in3_for_chain(CHAIN_ID_MAINNET);

  // get block using raw RPC call
  get_block_rpc(in3);

  // get block using API
  get_block_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_block_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                    //  the configured client
      "eth_getBlockByNumber", // the rpc-method you want to call.
      "[\"latest\",true]",    // the arguments as json-string
      &result,                // the reference to a pointer whill hold the result
      &error);                // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Latest block : \n%s\n", result);
    free(result);
  } else {
    printf("Error verifing the Latest block : \n%s\n", error);
    free(error);
  }
}

void get_block_api(in3_t* in3) {
  // get the block without the transaction details
  eth_block_t* block = eth_getBlockByNumber(in3, BLKNUM(8432424), false);

  // if the result is null there was an error an we can get the latest error message from eth_lat_error()
  if (!block)
    printf("error getting the block : %s\n", eth_last_error());
  else {
    printf("Number of transactions in Block #%llu: %d\n", block->number, block->tx_count);
    free(block);
  }
}
```
### get_logs

source : [in3-c/c/examples/get_logs.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/get_logs.c)

fetching events and verify them with eth_getLogs

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // functions for direct api-access
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/log.h>      // logging functions
#include <inttypes.h>
#include <stdio.h>

static void get_logs_rpc(in3_t* in3);
static void get_logs_api(in3_t* in3);

int main() {
  // create new incubed client
  in3_t* in3    = in3_for_chain(CHAIN_ID_MAINNET);

  // get logs using raw RPC call
  get_logs_rpc(in3);

  // get logs using API
  get_logs_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_logs_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,           //  the configured client
      "eth_getLogs", // the rpc-method you want to call.
      "[{}]",        // the arguments as json-string
      &result,       // the reference to a pointer whill hold the result
      &error);       // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Logs : \n%s\n", result);
    free(result);
  } else {
    printf("Error getting logs : \n%s\n", error);
    free(error);
  }
}

void get_logs_api(in3_t* in3) {
  // Create filter options
  char b[30];
  sprintf(b, "{\"fromBlock\":\"0x%" PRIx64 "\"}", eth_blockNumber(in3) - 2);
  json_ctx_t* jopt = parse_json(b);

  // Create new filter with options
  size_t fid = eth_newFilter(in3, jopt);

  // Get logs
  eth_log_t* logs = NULL;
  in3_ret_t  ret  = eth_getFilterLogs(in3, fid, &logs);
  if (ret != IN3_OK) {
    printf("eth_getFilterLogs() failed [%d]\n", ret);
    return;
  }

  // print result
  while (logs) {
    eth_log_t* l = logs;
    printf("--------------------------------------------------------------------------------\n");
    printf("\tremoved: %s\n", l->removed ? "true" : "false");
    printf("\tlogId: %lu\n", l->log_index);
    printf("\tTxId: %lu\n", l->transaction_index);
    printf("\thash: ");
    ba_print(l->block_hash, 32);
    printf("\n\tnum: %" PRIu64 "\n", l->block_number);
    printf("\taddress: ");
    ba_print(l->address, 20);
    printf("\n\tdata: ");
    b_print(&l->data);
    printf("\ttopics[%lu]: ", l->topic_count);
    for (size_t i = 0; i < l->topic_count; i++) {
      printf("\n\t");
      ba_print(l->topics[i], 32);
    }
    printf("\n");
    logs = logs->next;
    free(l->data.data);
    free(l->topics);
    free(l);
  }
  eth_uninstallFilter(in3, fid);
  json_free(jopt);
}
```
### get_transaction

source : [in3-c/c/examples/get_transaction.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/get_transaction.c)

checking the transaction data

```c

#include <in3/client.h> // the core client
#include <in3/eth_api.h>
#include <in3/in3_curl.h> // transport implementation
#include <in3/in3_init.h>
#include <in3/utils.h>
#include <stdio.h>

static void get_tx_rpc(in3_t* in3);
static void get_tx_api(in3_t* in3);

int main() {
  // create new incubed client
  in3_t* in3 = in3_for_chain(CHAIN_ID_MAINNET);

  // get tx using raw RPC call
  get_tx_rpc(in3);

  // get tx using API
  get_tx_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_tx_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                                                                        //  the configured client
      "eth_getTransactionByHash",                                                 // the rpc-method you want to call.
      "[\"0xdd80249a0631cf0f1593c7a9c9f9b8545e6c88ab5252287c34bc5d12457eab0e\"]", // the arguments as json-string
      &result,                                                                    // the reference to a pointer which will hold the result
      &error);                                                                    // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Latest tx : \n%s\n", result);
    free(result);
  } else {
    printf("Error verifing the Latest tx : \n%s\n", error);
    free(error);
  }
}

void get_tx_api(in3_t* in3) {
  // the hash of transaction that we want to get
  bytes32_t tx_hash;
  hex_to_bytes("0xdd80249a0631cf0f1593c7a9c9f9b8545e6c88ab5252287c34bc5d12457eab0e", -1, tx_hash, 32);

  // get the tx by hash
  eth_tx_t* tx = eth_getTransactionByHash(in3, tx_hash);

  // if the result is null there was an error an we can get the latest error message from eth_last_error()
  if (!tx)
    printf("error getting the tx : %s\n", eth_last_error());
  else {
    printf("Transaction #%d of block #%llx", tx->transaction_index, tx->block_number);
    free(tx);
  }
}
```
### get_transaction_receipt

source : [in3-c/c/examples/get_transaction_receipt.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/get_transaction_receipt.c)

validating the result or receipt of an transaction

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // functions for direct api-access
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/log.h>      // logging functions
#include <in3/utils.h>
#include <inttypes.h>
#include <stdio.h>

static void get_tx_receipt_rpc(in3_t* in3);
static void get_tx_receipt_api(in3_t* in3);

int main() {
  // create new incubed client
  in3_t* in3 = in3_for_chain(CHAIN_ID_MAINNET);

  // get tx receipt using raw RPC call
  get_tx_receipt_rpc(in3);

  // get tx receipt using API
  get_tx_receipt_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_tx_receipt_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                                                                        //  the configured client
      "eth_getTransactionReceipt",                                                // the rpc-method you want to call.
      "[\"0xdd80249a0631cf0f1593c7a9c9f9b8545e6c88ab5252287c34bc5d12457eab0e\"]", // the arguments as json-string
      &result,                                                                    // the reference to a pointer which will hold the result
      &error);                                                                    // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Transaction receipt: \n%s\n", result);
    free(result);
  } else {
    printf("Error verifing the tx receipt: \n%s\n", error);
    free(error);
  }
}

void get_tx_receipt_api(in3_t* in3) {
  // the hash of transaction whose receipt we want to get
  bytes32_t tx_hash;
  hex_to_bytes("0xdd80249a0631cf0f1593c7a9c9f9b8545e6c88ab5252287c34bc5d12457eab0e", -1, tx_hash, 32);

  // get the tx receipt by hash
  eth_tx_receipt_t* txr = eth_getTransactionReceipt(in3, tx_hash);

  // if the result is null there was an error an we can get the latest error message from eth_last_error()
  if (!txr)
    printf("error getting the tx : %s\n", eth_last_error());
  else {
    printf("Transaction #%d of block #%llx, gas used = %" PRIu64 ", status = %s\n", txr->transaction_index, txr->block_number, txr->gas_used, txr->status ? "success" : "failed");
    eth_tx_receipt_free(txr);
  }
}
```
### ipfs_put_get

source : [in3-c/c/examples/ipfs_put_get.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/ipfs_put_get.c)

using the IPFS module

```c

#include <in3/client.h>   // the core client
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/ipfs_api.h> // access ipfs-api
#include <in3/log.h>      // logging functions
#include <stdio.h>

#define LOREM_IPSUM "Lorem ipsum dolor sit amet"
#define return_err(err)                                \
  do {                                                 \
    printf(__FILE__ ":%d::Error %s\n", __LINE__, err); \
    return;                                            \
  } while (0)

static void ipfs_rpc_example(in3_t* c) {
  char *result, *error;
  char  tmp[100];

  in3_ret_t res = in3_client_rpc(
      c,
      "ipfs_put",
      "[\"" LOREM_IPSUM "\", \"utf8\"]",
      &result,
      &error);
  if (res != IN3_OK)
    return_err(in3_errmsg(res));

  printf("IPFS hash: %s\n", result);
  sprintf(tmp, "[%s, \"utf8\"]", result);
  free(result);
  result = NULL;

  res = in3_client_rpc(
      c,
      "ipfs_get",
      tmp,
      &result,
      &error);
  if (res != IN3_OK)
    return_err(in3_errmsg(res));
  res = strcmp(result, "\"" LOREM_IPSUM "\"");
  if (res) return_err("Content mismatch");
}

static void ipfs_api_example(in3_t* c) {
  bytes_t b         = {.data = (uint8_t*) LOREM_IPSUM, .len = strlen(LOREM_IPSUM)};
  char*   multihash = ipfs_put(c, &b);
  if (multihash == NULL)
    return_err("ipfs_put API call error");
  printf("IPFS hash: %s\n", multihash);

  bytes_t* content = ipfs_get(c, multihash);
  free(multihash);
  if (content == NULL)
    return_err("ipfs_get API call error");

  int res = strncmp((char*) content->data, LOREM_IPSUM, content->len);
  b_free(content);
  if (res)
    return_err("Content mismatch");
}

int main() {
  // create new incubed client
  in3_t* c = in3_for_chain(CHAIN_ID_IPFS);

  // IPFS put/get using raw RPC calls
  ipfs_rpc_example(c);

  // IPFS put/get using API
  ipfs_api_example(c);

  // cleanup client after usage
  in3_free(c);
  return 0;
}
```
### ledger_sign

source : [in3-c/c/examples/ledger_sign.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/ledger_sign.c)

```c
#include <in3/client.h>  // the core client
#include <in3/eth_api.h> // functions for direct api-access
#include <in3/ethereum_apdu_client.h>
#include <in3/in3_init.h>      // if included the verifier will automaticly be initialized.
#include <in3/ledger_signer.h> //to invoke ledger nano device for signing
#include <in3/log.h>           // logging functions
#include <in3/utils.h>
#include <stdio.h>

static void send_tx_api(in3_t* in3);

int main() {
  // create new incubed client
  uint8_t bip_path[5] = {44, 60, 0, 0, 0};
  in3_t*  in3         = in3_for_chain(CHAIN_ID_MAINNET);
  in3_log_set_level(LOG_DEBUG);
  // setting ledger nano s to be the default signer for incubed client
  // it will cause the transaction or any msg to be sent to ledger nanos device for siging
  eth_ledger_set_signer_txn(in3, bip_path);
  // eth_ledger_set_signer(in3, bip_path);

  // send tx using API
  send_tx_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void send_tx_api(in3_t* in3) {
  // prepare parameters
  address_t to, from;
  hex_to_bytes("0xC51fBbe0a68a7cA8d33f14a660126Da2A2FAF8bf", -1, from, 20);
  hex_to_bytes("0xd46e8dd67c5d32be8058bb8eb970870f07244567", -1, to, 20);

  bytes_t* data = hex_to_new_bytes("0x00", 0);
  // send the tx
  bytes_t* tx_hash = eth_sendTransaction(in3, from, to, OPTIONAL_T_VALUE(uint64_t, 0x96c0), OPTIONAL_T_VALUE(uint64_t, 0x9184e72a000), OPTIONAL_T_VALUE(uint256_t, to_uint256(0x9184e72a)), OPTIONAL_T_VALUE(bytes_t, *data), OPTIONAL_T_UNDEFINED(uint64_t));

  // if the result is null there was an error and we can get the latest error message from eth_last_error()
  if (!tx_hash)
    printf("error sending the tx : %s\n", eth_last_error());
  else {
    printf("Transaction hash: ");
    b_print(tx_hash);
    b_free(tx_hash);
  }
  b_free(data);
}
```
### send_transaction

source : [in3-c/c/examples/send_transaction.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/send_transaction.c)

sending a transaction including signing it with a private key

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // functions for direct api-access
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/log.h>      // logging functions
#include <in3/signer.h>   // default signer implementation
#include <in3/utils.h>
#include <stdio.h>

// fixme: This is only for the sake of demo. Do NOT store private keys as plaintext.
#define ETH_PRIVATE_KEY "0x8da4ef21b864d2cc526dbdb2a120bd2874c36c9d0a1fb7f8c63d7f7a8b41de8f"

static void send_tx_rpc(in3_t* in3);
static void send_tx_api(in3_t* in3);

int main() {
  // create new incubed client
  in3_t* in3 = in3_for_chain(CHAIN_ID_MAINNET);

  // convert the hexstring to bytes
  bytes32_t pk;
  hex_to_bytes(ETH_PRIVATE_KEY, -1, pk, 32);

  // create a simple signer with this key
  eth_set_pk_signer(in3, pk);

  // send tx using raw RPC call
  send_tx_rpc(in3);

  // send tx using API
  send_tx_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void send_tx_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                      //  the configured client
      "eth_sendRawTransaction", // the rpc-method you want to call.
      "[\"0xf892808609184e72a0008296c094d46e8dd67c5d32be8058bb8eb970870f0724456"
      "7849184e72aa9d46e8dd67c5d32be8d46e8dd67c5d32be8058bb8eb970870f072445675058bb8eb9"
      "70870f07244567526a06f0103fccdcae0d6b265f8c38ee42f4a722c1cb36230fe8da40315acc3051"
      "9a8a06252a68b26a5575f76a65ac08a7f684bc37b0c98d9e715d73ddce696b58f2c72\"]", // the signed raw txn, same as the one used in the API example
      &result,                                                                    // the reference to a pointer which will hold the result
      &error);                                                                    // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Result: \n%s\n", result);
    free(result);
  } else {
    printf("Error sending tx: \n%s\n", error);
    free(error);
  }
}

void send_tx_api(in3_t* in3) {
  // prepare parameters
  address_t to, from;
  hex_to_bytes("0x63FaC9201494f0bd17B9892B9fae4d52fe3BD377", -1, from, 20);
  hex_to_bytes("0xd46e8dd67c5d32be8058bb8eb970870f07244567", -1, to, 20);

  bytes_t* data = hex_to_new_bytes("d46e8dd67c5d32be8d46e8dd67c5d32be8058bb8eb970870f072445675058bb8eb970870f072445675", 82);

  // send the tx
  bytes_t* tx_hash = eth_sendTransaction(in3, from, to, OPTIONAL_T_VALUE(uint64_t, 0x96c0), OPTIONAL_T_VALUE(uint64_t, 0x9184e72a000), OPTIONAL_T_VALUE(uint256_t, to_uint256(0x9184e72a)), OPTIONAL_T_VALUE(bytes_t, *data), OPTIONAL_T_UNDEFINED(uint64_t));

  // if the result is null there was an error and we can get the latest error message from eth_last_error()
  if (!tx_hash)
    printf("error sending the tx : %s\n", eth_last_error());
  else {
    printf("Transaction hash: ");
    b_print(tx_hash);
    b_free(tx_hash);
  }
  b_free(data);
}
```
### usn_device

source : [in3-c/c/examples/usn_device.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/usn_device.c)

a example how to watch usn events and act upon it.

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // functions for direct api-access
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/log.h>      // logging functions
#include <in3/signer.h>   // signer-api
#include <in3/usn_api.h>
#include <in3/utils.h>
#include <inttypes.h>
#include <stdio.h>
#include <time.h>
#if defined(_WIN32) || defined(WIN32)
#include <windows.h>
#else
#include <unistd.h>
#endif

static int handle_booking(usn_event_t* ev) {
  printf("\n%s Booking timestamp=%" PRIu64 "\n", ev->type == BOOKING_START ? "START" : "STOP", ev->ts);
  return 0;
}

int main(int argc, char* argv[]) {
  // create new incubed client
  in3_t* c = in3_for_chain(CHAIN_ID_GOERLI);

  // setting up a usn-device-config
  usn_device_conf_t usn;
  usn.booking_handler    = handle_booking;                                          // this is the handler, which is called for each rent/return or start/stop
  usn.c                  = c;                                                       // the incubed client
  usn.chain_id           = c->chain.chain_id;                                       // the chain_id
  usn.devices            = NULL;                                                    // this will contain the list of devices supported
  usn.len_devices        = 0;                                                       // and length of this list
  usn.now                = 0;                                                       // the current timestamp
  unsigned int wait_time = 5;                                                       // the time to wait between the internval
  hex_to_bytes("0x85Ec283a3Ed4b66dF4da23656d4BF8A507383bca", -1, usn.contract, 20); // address of the usn-contract, which we copy from hex

  // register a usn-device
  usn_register_device(&usn, "office@slockit");

  // now we run en endless loop which simply wait for events on the chain.
  printf("\n start watching...\n");
  while (true) {
    usn.now              = time(NULL);                               // update the timestamp, since this is running on embedded devices, this may be depend on the hardware.
    unsigned int timeout = usn_update_state(&usn, wait_time) * 1000; // this will now check for new events and trigger the handle_booking if so.

    // sleep
#if defined(_WIN32) || defined(WIN32)
    Sleep(timeout);
#else
    nanosleep((const struct timespec[]){{0, timeout * 1000000L}}, NULL);
#endif
  }

  // clean up
  in3_free(c);
  return 0;
}
```
### usn_rent

source : [in3-c/c/examples/usn_rent.c](https://github.com/blockchainsllc/in3/blob/master/c/examples/usn_rent.c)

how to send a rent transaction to a usn contract usinig the usn-api.

```c

#include <in3/api_utils.h>
#include <in3/eth_api.h>  // functions for direct api-access
#include <in3/in3_init.h> // if included the verifier will automaticly be initialized.
#include <in3/signer.h>   // signer-api
#include <in3/usn_api.h>  // api for renting
#include <in3/utils.h>
#include <inttypes.h>
#include <stdio.h>

void unlock_key(in3_t* c, char* json_data, char* passwd) {
  // parse the json
  json_ctx_t* key_data = parse_json(json_data);
  if (!key_data) {
    perror("key is not parseable!\n");
    exit(EXIT_FAILURE);
  }

  // decrypt the key
  uint8_t* pk = malloc(32);
  if (decrypt_key(key_data->result, passwd, pk) != IN3_OK) {
    perror("wrong password!\n");
    exit(EXIT_FAILURE);
  }

  // free json
  json_free(key_data);

  // create a signer with this key
  eth_set_pk_signer(c, pk);
}

int main(int argc, char* argv[]) {
  // create new incubed client
  in3_t* c = in3_for_chain(CHAIN_ID_GOERLI);

  // address of the usn-contract, which we copy from hex
  address_t contract;
  hex_to_bytes("0x85Ec283a3Ed4b66dF4da23656d4BF8A507383bca", -1, contract, 20);

  // read the key from args - I know this is not safe, but this is just a example.
  if (argc < 3) {
    perror("you need to provide a json-key and password to rent it");
    exit(EXIT_FAILURE);
  }
  char* key_data = argv[1];
  char* passwd   = argv[2];
  unlock_key(c, key_data, passwd);

  // rent it for one hour.
  uint32_t renting_seconds = 3600;

  // allocate 32 bytes for the resulting tx hash
  bytes32_t tx_hash;

  // start charging
  if (usn_rent(c, contract, NULL, "office@slockit", renting_seconds, tx_hash))
    printf("Could not start charging\n");
  else {
    printf("Charging tx successfully sent... tx_hash=0x");
    for (int i = 0; i < 32; i++) printf("%02x", tx_hash[i]);
    printf("\n");

    if (argc == 4) // just to include it : if you want to stop earlier, you can call
      usn_return(c, contract, "office@slockit", tx_hash);
  }

  // clean up
  in3_free(c);
  return 0;
}
```
### Building

In order to run those examples, you only need a c-compiler (gcc or clang) and curl installed.

```c
./build.sh
```
will build all examples in this directory. You can build them individually by executing:

```c
gcc -o get_block_api get_block_api.c -lin3 -lcurl
```



## How it works

The core of incubed is the processing of json-rpc requests by fetching data from the network and verifying them. This is why in the `core`-module it is all about rpc-requests and their responses.

### the statemachine

Each request is represented internally by the `in3_req_t` -struct. This context is responsible for trying to find a verifyable answer to the request and acts as a statemachine.

```eval_rst
.. graphviz::

    digraph G {
        node[fontname="Helvetica",   shape=Box, color=lightblue, style=filled ]
        edge[fontname="Helvetica",   style=solid,  fontsize=8 , color=grey]
        rankdir = TB;
    
        RPC[label="RPC-Request"]
        CTX[label="in3_req_t"]
    
        sign[label="sign",color=lightgrey, style=""]
        request[label="fetch http",color=lightgrey, style=""]
    
        exec[ label="in3_req_exec_state()",color=lightgrey, style="", shape=ellipse ]
        free[label="req_free()",color=lightgrey, style=""]
    
        waiting[label="need input"]
    
    
        RPC -> CTX [label="req_new()"]
        CTX -> exec
    
    
        exec -> error [label="REQ_ERROR"]
        exec -> response[label="REQ_SUCCESS"]
        exec -> waiting[label="REQ_WAITING_TO_SEND"]
        exec -> request[label="REQ_WAITING_FOR_RESPONSE"]
    
    
        waiting -> sign[label=RT_SIGN]
        waiting -> request[label=RT_RPC] 
    
        sign -> exec [label="in3_ctx_add_response()"]
        request -> exec[label="in3_ctx_add_response()"]
    
        response -> free
        error->free
    
    
      { rank = same; error, response }
    
      { rank = same; exec,waiting }
      { rank = same; CTX,request }
    
    
        }
    
```
In order to process a request we follow these steps.

1. `req_new` which creates a new context by parsing a JSON-RPC request.
2. `in3_req_exec_state` this will try to process the state and returns the new state, which will be one of he following:

- `REQ_SUCCESS` - we have a response
- `REQ_ERROR` - we stop because of an unrecoverable error
- `REQ_WAITING_TO_SEND` - we need input and need to send out a request. By calling `in3_create_request()` the ctx will switch to the state to `REQ_WAITING_FOR_RESPONSE` until all the needed responses are repoorted. While it is possible to fetch all responses and add them before calling `in3_req_exec_state()`, but it would be more efficient if can send all requests out, but then create a response-queue and set one response add a time so we can return as soon as we have the first verifiable response.
- `REQ_WAITING_FOR_RESPONSE` - the request has been send, but no verifieable response is available. Once the next (or more) responses have been added, we call `in3_req_exec_state()` again, which will verify all available responses. If we could verify it, we have a respoonse, if not we may either wait for more responses ( in case we send out multiple requests -> `REQ_WAITING_FOR_RESPONSE` ) or we send out new requests (`REQ_WAITING_TO_SEND`)

the `in3_send_req`-function will executly this:

```c
in3_ret_t in3_send_req(in3_req_t* ctx) {
  ctx_req_transports_t transports = {0};
  while (true) {
    switch (in3_req_exec_state(ctx)) {
      case REQ_ERROR:
      case REQ_SUCCESS:
        transport_cleanup(ctx, &transports, true);
        return ctx->verification_state;

      case REQ_WAITING_FOR_RESPONSE:
        in3_handle_rpc_next(ctx, &transports);
        break;

      case REQ_WAITING_TO_SEND: {
        in3_req_t* last = in3_req_last_waiting(ctx);
        switch (last->type) {
          case RT_SIGN:
            in3_handle_sign(last);
            break;
          case RT_RPC:
            in3_handle_rpc(last, &transports);
        }
      }
    }
  }
}
```
### sync calls with in3_send_req

This statemachine can be used to process requests synchronously or asynchronously. The `in3_send_req` function, which is used in most convinience-functions will do this synchronously. In order to get user input it relies on 2 callback-functions:

- to sign : [`in3_signer_t`](#in3-signer-t) struct including its callback function is set in the `in3_t` configuration.
- to fetch data : a [in3_transport_send](#in3-transport-send) function-pointer will be set in the `in3_t` configuration.

#### signing

For signing the client expects a [`in3_signer_t`](#in3-signer-t) struct to be set. Setting should be done by using the [`in3_set_signer()`](#in3-set-signer) function. This function expects 3 arguments (after the client config itself):

- `sign` - this is a function pointer to actual signing-function. Whenever the incubed client needs a signature it will prepare a signing context [`in3_sign_ctx_t`](#in3-sign-ctx-t), which holds all relevant data, like message and the address for signing. The result will always be a signature which you need to copy into the `signature`-field of this context. The return value must signal the success of the execution. While `IN3_OK` represents success, `IN3_WAITING`can be used to indicate that we need to execute again since there may be a sub-request that needs to finished up before being able to sign. In case of an error [`req_set_error`](#ctx-set-error) should be used to report the details of the error including returning the `IN3_E...` as error-code.
- `prepare_tx`- this function is optional and gives you a chance to change the data before signing. For example signing with a mutisig would need to do manipulate the data and also the target in order to redirect it to the multisig contract.
- `wallet` - this is a optional `void*` which will be set in the signing context. It can be used to point to any data structure you may need in order to sign.

As a example this is the implemantation of the signer-function for a simple raw private key:

```c

in3_ret_t eth_sign_pk_ctx(in3_sign_ctx_t* ctx) {
  uint8_t* pk = ctx->wallet;
  switch (ctx->type) {
    case SIGN_EC_RAW:
      return ec_sign_pk_raw(ctx->message.data, pk, ctx->signature);
    case SIGN_EC_HASH:
      return ec_sign_pk_hash(ctx->message.data, ctx->message.len, pk, hasher_sha3k, ctx->signature);
    default:
      return IN3_ENOTSUP;
  }
  return IN3_OK;
}
```
The pk-signer uses the wallet-pointer to point to the raw 32 bytes private key and will use this to sign.

#### transport

The transport function is a function-pointer set in the client configuration (`in3_t`) which will be used in the `in3_send_req()` function whenever data are required to get from the network. the function will get a [`request_t`](#request-t) object as argument.

The main responsibility of this function is to fetch the requested data and the call [`in3_ctx_add_response`](#in3-ctx-add-response) to report this to the context. if the request only sends one request to one url, this is all you have to do. But if the user uses a configuration of `request_count` >1, the `request` object will contain a list of multiples urls. In this case transport function still has 3 options to accomplish this:

1. send the payload to each url sequentially. This is **NOT** recommented, since this increases the time the user has to wait for a response. Especially if some of the request may run into a timeout.
2. send the all in parallel and wait for all the finish. This is better, but it still means, we may have to wait until the last one responses even though we may have a verifiable response already reported.
3. send them all in parallel and return as soon as we have the first response. This increases the performance since we don't have to wait if we have one. But since we don't know yet whether this response is also correct, we must be prepared to also read the other responses if needed, which means the transport would be called multiple times for the same request. In order to process multiple calls to the same resouces the request-object contains two fields:

- `cptr` - a custom `void*` which can be set in the first call pointing to recources you may need to continue in the subsequent calls.
- `action` - This value is enum ( [`#in3_req_action_t`](#in3-req-action-t) ), which indicates these current state

So only if you need to continue your call later, because you don't want to and can't set all the responses yet, you need set the `cptr` to a non NULL value. And only in this case `in3_send_req()` will follow this process with these states:

```eval_rst
.. graphviz::

    digraph G {
        node[fontname="Helvetica",   shape=Box, color=lightblue, style=filled ]
        rankdir = TB;
    
        REQ_ACTION_SEND -> REQ_ACTION_RECEIVE -> REQ_ACTION_CLEANUP
        REQ_ACTION_RECEIVE -> REQ_ACTION_RECEIVE
    
```
- `REQ_ACTION_SEND` - this will always be set in the first call.
- `REQ_ACTION_RECEIVE` - a call with this state indicates that there was a send call prior but since we do not have all responses yet, the transport should now set the next reponse. So this call may be called multiple times until either we have found a verifieable response or the number of urls is reached. Important during this call the `urls` field of the request will be NULL since this should not send a new request.
- `REQ_ACTION_CLEANUP` - this will only be used if the `cptr` was set before. Here the transport should only clean up any allocated resources. This will also be called if not all responses were used.

While there are of course existing implementations for the transport-function ( as default we use `in3_curl_c`), especially for embedded devices you may even implement your own.

### async calls

While for sync calls you can just implement a transport function, you can also take full control of the process which allows to execute it completly async. The basic process is the same layed out in the [state machine](#the-statemachine).

For the js for example the main-loop is part of a async function.

```js
async sendRequest(rpc) {

    // create the context
    const r = in3w.ccall('in3_create_request_ctx', 'number', ['number', 'string'], [this.ptr, JSON.stringify(rpc)]);

    // hold a queue for responses for the different request contexts
    let responses = {}

    try {
      // main async loop
      while (true) {

          // execute and fetch the new state ( in this case the ctx_execute-function will return the status including the created request as json)
          const state = JSON.parse(call_string('ctx_execute', r))
          switch (state.status) {
              // REQ_ERROR
              case 'error':
                  throw new Error(state.error || 'Unknown error')

              // REQ_SUCCESS
              case 'ok':
                  return state.result

              // REQ_WAITING_FOR_RESPONSE
              case 'waiting':
                  // await the promise for the next response ( the state.request contains the context-pointer to know which queue)
                  await getNextResponse(responses, state.request)
                  break

              // REQ_WAITING_TO_SEND
              case 'request': {
                  // the request already contains the type, urls and payload.
                  const req = state.request
                  switch (req.type) {
                      case 'sign':
                          try {
                              // get the message and account from the request
                              const [message, account] = Array.isArray(req.payload) ? req.payload[0].params : req.payload.params;
                              // check if can sign
                              if (!(await this.signer.canSign(account))) throw new Error('unknown account ' + account)

                              // and set the signature (65 bytes) as response. 
                              setResponse(req.ctx, toHex(await this.signer.sign(message, account, true, false)), 0, false)
                          } catch (ex) {
                              // or set the error
                              setResponse(req.ctx, ex.message || ex, 0, true)
                          }
                          break;

                      case 'rpc':
                          // here we will send a new request, which puts its responses in a queue
                          await getNextResponse(responses, req)
                  }
              }
          }
      }
    }
    finally {
        // we always need to cleanup
        in3w.ccall('in3_request_free', 'void', ['number'], [r])
    }
}
```



## Plugins

While the core is kept as small as possible, we defined actions, which can be implemented by plugins. The core alone would not be able to do any good. While the in3-c repository already provides default implementations for all actions, as a developer you can always extend or replace those. There are good reasons to do so:

- optimizing by using a smaller plugin (like replacing the nodelist handling)
- allowing custom rpc-commands
- changing behavior ...

### What is a plugin?

Each plugin needs to define those 3 things:

1. **Actions** - Which actions do I want handle. This is a bitmask with the actions set. You can use any combination.
2. **Custom data** - This optional data object may contain configurations or other data. If you don't need to hold any data, you may pass `NULL`
3. **Exec-function** - This is a function pointer to a function which will be called whenever the plugin is used.

With these 3 things you can register a plugin with the `in3_plugin_register()` -function:

```c
return in3_plugin_register("myplugin"           // the plugin name
         c,                                     // the client
         PLGN_ACT_TERM | PLGN_ACT_RPC_HANDLE,   // the actions to register for
         handle_rpc,                            // the plugin-function
         cutom_data,                            // the custom data (if needed)
         false);                                // a bool indicating whether it should always add or replace a plugin with the exact same actions.
```
#### The Plugin-function

Each Plugin must provide a PLugin-function to execute with the following signature:

```c
in3_ret_t handle(
  void*            custom_data,  // the custom data as passed in the register-function
  in3_plugin_act_t action,       // the action to execute
  void*            arguments);   // the arguments (depending on the action)
```
While the `custom_data` is just the pointer to your data-object, the `arguments` contain a pointer to a context object. This object depends on the action you are reacting.

All plugins are stored in a linked list and when we want to trigger a specific actions we will loop through all, but only execute the function if the required action is set in the bitmask. Except for `PLGN_ACT_TERM` we will loop until the first plugin handles it. The handle-function must return a return code indicating this:

- `IN3_OK` - the plugin handled it and it was succesful
- `IN3_WAITING` - the plugin handled the action, but is waiting for more data, which happens in a sub context added. As soon as this was resolved, the plugin will be called again.
- `IN3_EIGNORE` - the plugin did **NOT** handle the action and we should continue with the other plugins.
- `IN3_E...` - the plugin did handle it, but raised a error and returned the error-code. In addition you should always use the current `in3_req_t`to report a detailed error-message (using `req_set_error()`)

### Lifecycle

#### PLGN_ACT_TERM

This action will be triggered during `in3_free` and must be used to free up resources which were allocated.

`arguments` : `in3_t*` - the in3-instance will be passed as argument.

### Transport

For Transport implementations you should always register for those 3 `PLGN_ACT_TRANSPORT_SEND` | `PLGN_ACT_TRANSPORT_RECEIVE` | `PLGN_ACT_TRANSPORT_CLEAN`. This is why you can also use the macro combining those as `PLGN_ACT_TRANSPORT`

#### PLGN_ACT_TRANSPORT_SEND

Send will be triggered only if the request is executed synchron, whenever a new request needs to be send out. This request may contain multiple urls, but the same payload.

`arguments` : `in3_http_request_t*` - a request-object holding the following data:

```c
typedef struct in3_http_request {
  char*           payload;  // the payload to send 
  char**          urls;     // array of urls 
  uint_fast16_t   urls_len; // number of urls 
  in3_req_t*      ctx;      // the current context 
  void*           cptr;     // a custom ptr to hold information during 
} in3_http_request_t;
```
It is expected that a plugin will send out http-requests to each (iterating until `urls_len`) url from `urls` with the `payload`. if the payload is NULL or empty the request is a `GET`-request. Otherwise, the plugin must use send it with HTTP-Header `Content-Type: application/json` and attach the `payload`.

After the request is send out the `cptr` may be set in order to fetch the responses later. This allows us the fetch responses as they come in instead of waiting for the last response before continuing.

Example: 

```c
in3_ret_t transport_handle(void* custom_data, in3_plugin, in3_plugin_act_t action, void* arguments) {
  switch (action) {

    case PLGN_ACT_TRANSPORT_SEND: {
      in3_http_request_t* req = arguments; // cast it to in3_http_request_t* 

      // init the cptr
      in3_curl_t* c = _malloc(sizeof(in3_curl_t));
      c->cm         = curl_multi_init(); // init curl
      c->start      = current_ms();      // keep the staring time
      req->cptr     = c;                 // set the cptr

      // define headers
      curl_multi_setopt(c->cm, CURLMOPT_MAXCONNECTS, (long) CURL_MAX_PARALLEL);
      struct curl_slist* headers = curl_slist_append(NULL, "Accept: application/json");
      if (req->payload && *req->payload)
        headers = curl_slist_append(headers, "Content-Type: application/json");
      headers    = curl_slist_append(headers, "charsets: utf-8");
      c->headers = curl_slist_append(headers, "User-Agent: in3 curl " IN3_VERSION);

      // send out requests in parallel
      for (unsigned int i = 0; i < req->urls_len; i++)
        readDataNonBlocking(c->cm, req->urls[i], req->payload, c->headers, req->ctx->raw_response + i, req->ctx->client->timeout);

      return IN3_OK;
    }

    // handle other actions ...
  }
}
```
#### PLGN_ACT_TRANSPORT_RECEIVE

This will only triggered if the previously triggered `PLGN_ACT_TRANSPORT_SEND`

- was successfull (IN3_OK)
- if the responses were not all set yet.
- if a `cptr` was set

`arguments` : `in3_http_request_t*` - a request-object holding the data. ( the payload and urls may not be set!)

The plugin needs to wait until the first response was received ( or runs into a timeout). To report, please use `in3_req_add_response()``

```c
void in3_req_add_response(
    in3_http_request_t* req,      //  the the request 
    int            index,    //  the index of the url, since this request could go out to many urls 
    bool           is_error, //  if true this will be reported as error. the message should then be the error-message 
    const char*    data,     //  the data or the the string of the response
    int            data_len, //  the length of the data or the the string (use -1 if data is a null terminated string)
    uint32_t       time      //  the time (in ms) this request took in ms or 0 if not possible (it will be used to calculate the weights)    
);
```
In case of a succesful response:

```c
in3_req_add_response(request, index, false, response_data, -1, current_ms() - start);
```
in case of an error, the data is the error message itself:

```c
in3_req_add_response(request, index, true, "Timeout waiting for a response", -1, 0);
```
#### PLGN_ACT_TRANSPORT_CLEAN

If a previous `PLGN_ACT_TRANSPORT_SEND` has set a `cptr` this will be triggered in order to clean up memory.

`arguments` : `in3_http_request_t*` - a request-object holding the data. ( the payload and urls may not be set!)

### Signing

For Signing we have three different action. While `PLGN_ACT_SIGN` should alos react to `PLGN_ACT_SIGN_ACCOUNT`, `PLGN_ACT_SIGN_PREPARE` can also be completly independent.

#### PLGN_ACT_SIGN

This action is triggered as a request to sign data.

`arguments` : `in3_sign_ctx_t*` - the sign context will hold those data:

```c
typedef struct sign_ctx {
  uint8_t            signature[65]; // the resulting signature needs to be writte into these bytes 
  d_signature_type_t type;          // the type of signature
  in3_req_t*         ctx;           // the context of the request in order report errors 
  bytes_t            message;       // the message to sign
  bytes_t            account;       // the account to use for the signature  (if set)
} in3_sign_ctx_t;
```
The signature must be 65 bytes and in the format , where v must be the recovery byte and should only be 1 or 0.

```c
r[32]|s[32]|v[1]
```
Currently there are 2 types of sign-request:

- SIGN_EC_RAW : the data is already 256bits and may be used directly
- SIGN_EC_HASH : the data may be any kind of message, and need to be hashed first. As hash we will use Keccak.

Example:

```c
in3_ret_t eth_sign_pk(void* data, in3_plugin_act_t action, void* args) {
  // the data are our pk
  uint8_t* pk = data; 

  switch (action) {

    case PLGN_ACT_SIGN: {
      // cast the context
      in3_sign_ctx_t* ctx = args;

      // if there is a account set, we only sign if this matches our account
      // this way we allow multiple accounts to added as plugin
      if (ctx->account.len == 20) {
        address_t adr;
        get_address(pk, adr);
        if (memcmp(adr, ctx->account.data, 20)) 
           return IN3_EIGNORE; // does not match, let someone else handle it
      }

      // sign based on sign type
      switch (ctx->type) {
        case SIGN_EC_RAW:
          return ec_sign_pk_raw(ctx->message.data, pk, ctx->signature);
        case SIGN_EC_HASH:
          return ec_sign_pk_hash(ctx->message.data, ctx->message.len, pk, hasher_sha3k, ctx->signature);
        default:
          return IN3_ENOTSUP;
      }
    }

    case PLGN_ACT_SIGN_ACCOUNT: {
      // cast the context
      in3_sign_account_ctx_t* ctx = args;

      // generate the address from the key
      get_address(pk, ctx->account);
      return IN3_OK;
    }

    default:
      return IN3_ENOTSUP;
  }
}


in3_ret_t eth_set_pk_signer(in3_t* in3, bytes32_t pk) {
  // we register for both ACCOUNT and SIGN
  return plugin_register(in3, PLGN_ACT_SIGN_ACCOUNT | PLGN_ACT_SIGN, eth_sign_pk, pk, false);
}
```
#### PLGN_ACT_SIGN_ACCOUNT

if we are about to sign data and need to know the address of the account abnout to sign, this action will be triggered in order to find out. This is needed if you want to send a transaction without specifying the `from` address, we will still need to get the nonce for this account before signing.

`arguments` : `in3_sign_account_ctx_t*` - the account context will hold those data:

```c
typedef struct sign_account_ctx {
  in3_req_t* ctx;     // the context of the request in order report errors 
  address_t  account; // the account to use for the signature 
} in3_sign_account_ctx_t;
```
The implementation should return a status code ´IN3_OK` if it successfully wrote the address of the account into the content:

Example:

```c
in3_ret_t eth_sign_pk(void* data, in3_plugin_act_t action, void* args) {
  // the data are our pk
  uint8_t* pk = data; 

  switch (action) {

    case PLGN_ACT_SIGN_ACCOUNT: {
      // cast the context
      in3_sign_account_ctx_t* ctx = args;

      // generate the address from the key
      // and write it into account
      get_address(pk, ctx->account);
      return IN3_OK;
    }

    // handle other actions ...

    default:
      return IN3_ENOTSUP;
  }
}
```
#### PLGN_ACT_SIGN_PREPARE

The Prepare-action is triggered before signing and gives a plugin the chance to change the data. This is needed if you want to send a transaction through a multisig. Here we have to change the `data` and `to` address.

`arguments` : `in3_sign_prepare_ctx_t*` - the prepare context will hold those data:

```c
typedef struct sign_prepare_ctx {
  struct in3_req* ctx;     // the context of the request in order report errors 
  address_t       account; // the account to use for the signature 
  bytes_t         old_tx;  // the data to sign 
  bytes_t         new_tx;  // the new data to be set 

} in3_sign_prepare_ctx_t;
```
the tx-data will be in a form ready to sign, which means those are rlp-encoded data of a transaction without a signature, but the chain-id as v-value.

In order to decode the data you must use rlp.h:

```c
#define decode(data,index,dst,msg) if (rlp_decode_in_list(data, index, dst) != 1) return req_set_error(ctx, "invalid" msg "in txdata", IN3_EINVAL);

in3_ret_t decode_tx(in3_req_t* ctx, bytes_t raw, tx_data_t* result) {
  decode(&raw, 0, &result->nonce    , "nonce");
  decode(&raw, 1, &result->gas_price, "gas_price");
  decode(&raw, 2, &result->gas      , "gas");
  decode(&raw, 3, &result->to       , "to");
  decode(&raw, 4, &result->value    , "value");
  decode(&raw, 5, &result->data     , "data");
  decode(&raw, 6, &result->v        , "v");
  return IN3_OK;
}
```
and of course once the data has changes you need to encode it again and set it as `nex_tx``

### RPC Handling

#### PLGN_ACT_RPC_HANDLE

Triggered for each rpc-request in order to give plugins a chance to directly handle it. If no onoe handles it it will be send to the nodes.

`arguments` : `in3_rpc_handle_ctx_t*` - the rpc_handle context will hold those data:

```c
typedef struct {
  in3_req_t*       ctx;      // Request context. 
  d_token_t*       request;  // request 
  in3_response_t** response; // the response which a prehandle-method should set
} in3_rpc_handle_ctx_t;
```
the steps to add a new custom rpc-method will be the following.

1. get the method and params:

```c
char* method      = d_get_string(rpc->request, K_METHOD);
d_token_t* params = d_get(rpc->request, K_PARAMS);
```
1. check if you can handle it
2. handle it and set the result 

```c
in3_rpc_handle_with_int(rpc,result);
```

for setting the result you should use one of the `in3_rpc_handle_...` methods. Those will create the response and build the JSON-string with the result. While most of those expect the result as a sngle value you can also return a complex JSON-Object. In this case you have to create a string builder:

```c
sb_t* writer = in3_rpc_handle_start(rpc);
sb_add_chars(writer, "{\"raw\":\"");
sb_add_escaped_chars(writer, raw_string);
// ... more data
sb_add_chars(writer, "}");
return in3_rpc_handle_finish(rpc);
```
1. In case of an error, simply set the error in the context, with the right message and error-code:

```c
if (d_len(params)<1) return req_set_error(rpc->ctx, "Not enough parameters", IN3_EINVAL);
```
If the reequest needs additional subrequests, you need to follow the pattern of sending a request asynchron in a state machine:

```c
// we want to get the nonce.....
uint64_t  nonce =0;

// check if a request is already existing
in3_req_t* ctx = req_find_required(rpc->ctx, "eth_getTransactionCount");
if (ctx) {
  // found one - so we check if it is ready.
  switch (in3_req_state(ctx)) {
    // in case of an error, we report it back to the parent context
    case REQ_ERROR:
      return req_set_error(rpc->ctx, ctx->error, IN3_EUNKNOWN);
    // if we are still waiting, we stop here and report it.
    case REQ_WAITING_FOR_RESPONSE:
    case REQ_WAITING_TO_SEND:
      return IN3_WAITING;

    // if it is useable, we can now handle the result.
    case REQ_SUCCESS: {
      // check if the response contains a error.
      TRY(req_check_response_error(ctx, 0))

      // read the nonce
      nonce = d_get_long(ctx->responses[0], K_RESULT);
    }
  }
}
else {
  // no required context found yet, so we create one:

  // since this is a subrequest it will be freed when the parent is freed.
  // allocate memory for the request-string
  char* req = _malloc(strlen(params) + 200);
  // create it
  sprintf(req, "{\"method\":\"eth_getTransactionCount\",\"jsonrpc\":\"2.0\",\"id\":1,\"params\":[\"%s\",\"latest\"]}", account_hex_string);
  // and add the request context to the parent.
  return req_add_required(parent, req_new(parent->client, req));
}

// continue here and use the nonce....
```
Here is a simple Example how to register a plugin hashing data:

```c
static in3_ret_t handle_intern(void* pdata, in3_plugin_act_t action, void* args) {
  UNUSED_VAR(pdata);

  // cast args 
  in3_rpc_handle_ctx_t* rpc = args;

  swtch (action) {
    case PLGN_ACT_RPC_HANDLE: {
      // get method and params
      char*                 method  = d_get_string(rpc->request, K_METHOD);
      d_token_t*            params  = d_get(rpc->request, K_PARAMS);

      // do we support it?
      if (strcmp(method, "web3_sha3") == 0) {
        // check the params
        if (!params || d_len(params) != 1) return req_set_error(rpc->ctx, "invalid params", IN3_EINVAL);
        bytes32_t hash;
        // hash the first param
        keccak(d_to_bytes(d_get_at(params,0)), hash);
        // return the hash as resut.
        return in3_rpc_handle_with_bytes(ctx, bytes(hash, 32));
      }

      // we don't support this method, so we ignore it.
      return IN3_EIGNORE;
    }

    default:
      return IN3_ENOTSUP;
  }
}

in3_ret_t in3_register_rpc_handler(in3_t* c) {
  return plugin_register(c, PLGN_ACT_RPC_HANDLE, handle_intern, NULL, false);
}
```
#### PLGN_ACT_RPC_VERIFY

This plugin reprresents a verifier. It will be triggered after we have received a response from a node.

`arguments` : `in3_vctx_t*` - the verification context will hold those data:

```c
typedef struct {
  in3_req_t*   ctx;                   // Request context. 
  in3_chain_t* chain;                 // the chain definition. 
  d_token_t*   result;                // the result to verify 
  d_token_t*   request;               // the request sent. 
  d_token_t*   proof;                 // the delivered proof. 
  in3_t*       client;                // the client. 
  uint64_t     last_validator_change; // Block number of last change of the validator list 
  uint64_t     currentBlock;          // Block number of latest block 
  int          index;                 // the index of the request within the bulk 
} in3_vctx_t;
```
Example:

```c
in3_ret_t in3_verify_ipfs(void* pdata, in3_plugin_act_t action, void* args) {
  if (action!=PLGN_ACT_RPC_VERIFY) return IN3_ENOTSUP;
  UNUSED_VAR(pdata);

  // we want this verifier to handle ipfs-chains
  if (vc->chain->type != CHAIN_IPFS) return IN3_EIGNORE;


  in3_vctx_t* vc     = args;
  char*       method = d_get_string(vc->request, K_METHOD);
  d_token_t*  params = d_get(vc->request, K_PARAMS);

  // did we ask for proof?
  if (in3_req_get_proof(vc->ctx, vc->index) == PROOF_NONE) return IN3_OK;

  // do we have a result? if not it is a vaslid error-response
  if (!vc->result)
    return IN3_OK;

  if (strcmp(method, "ipfs_get") == 0)
    return ipfs_verify_hash(d_string(vc->result),
                            d_get_string_at(params, 1) ? d_get_string_at(params, 1) : "base64",
                            d_get_string_at(params, 0));

  // could not verify, so we hope some other plugin will
  return IN3_EIGNORE;
}

in3_ret_t in3_register_ipfs(in3_t* c) {
  return plugin_register(c, PLGN_ACT_RPC_VERIFY, in3_verify_ipfs, NULL, false);
}
```
### Cache/Storage

For Cache implementations you also need to register all 3 actions.

#### PLGN_ACT_CACHE_SET

This action will be triggered whenever there is something worth putting in a cache. If no plugin picks it up, it is ok, since the cache is optional.

`arguments` : `in3_cache_ctx_t*` - the cache context will hold those data:

```c
typedef struct in3_cache_ctx {
  in3_req_t* ctx;     // the request context  
  char*      key;     // the key to fetch 
  bytes_t*   content; // the content to set 
} in3_cache_ctx_t;
```
in the case of `CACHE_SET` the content will point to the bytes we need to store somewhere. If for whatever reason the item can not be stored, a `IN3_EIGNORE` should be send, since to indicate that no action took place.

```c
Example:

```c
in3_ret_t handle_storage(void* data, in3_plugin_act_t action, void* arg) {
  in3_cache_ctx_t* ctx = arg;
  switch (action) {
    case PLGN_ACT_CACHE_GET: {
       ctx->content = storage_get_item(data, ctx->key);
       return ctx->content ? IN3_OK : IN3_EIGNORE;
    }
    case PLGN_ACT_CACHE_SET: {
      storage_set_item(data, ctx->key, ctx->content);
      return IN3_OK;
    }
    case PLGN_ACT_CACHE_CLEAR: {
      storage_clear(data);
      return IN3_OK;
    }
    default: return IN3_EINVAL;
  }
}

in3_ret_t in3_register_file_storage(in3_t* c) {
  return plugin_register(c, PLGN_ACT_CACHE, handle_storage, NULL, true);
}
```
#### PLGN_ACT_CACHE_GET

This action will be triggered whenever we access the cache in order to get values.

`arguments` : `in3_cache_ctx_t*` - the cache context will hold those data:

```c
typedef struct in3_cache_ctx {
  in3_req_t* ctx;     // the request context  
  char*      key;     // the key to fetch 
  bytes_t*   content; // the content to set 
} in3_cache_ctx_t;
```
in the case of `CACHE_GET` the content will be NULL and needs to be set to point to the found values. If we did not find it in the cache, we must return `IN3_EIGNORE`.

```c
Example:

```c
ctx->content = storage_get_item(data, ctx->key);
return ctx->content ? IN3_OK : IN3_EIGNORE;
```
#### PLGN_ACT_CACHE_CLEAR

This action will clear all stored values in the cache.

`arguments` :NULL - so no argument will be passed.

### Configuration

For Configuration there are 2 actions for getting and setting. You should always implement both.

Example:

```c
static in3_ret_t handle_btc(void* custom_data, in3_plugin_act_t action, void* args) {
  btc_target_conf_t* conf = custom_data;
  switch (action) {
    // clean up
    case PLGN_ACT_TERM: {
      if (conf->data.data) _free(conf->data.data);
      _free(conf);
      return IN3_OK;
    }

    // read config
    case PLGN_ACT_CONFIG_GET: {
      in3_get_config_ctx_t* cctx = args;
      sb_add_chars(cctx->sb, ",\"maxDAP\":");
      sb_add_int(cctx->sb, conf->max_daps);
      sb_add_chars(cctx->sb, ",\"maxDiff\":");
      sb_add_int(cctx->sb, conf->max_diff);
      return IN3_OK;
    }

    // configure
    case PLGN_ACT_CONFIG_SET: {
      in3_configure_ctx_t* cctx = args;
      if (cctx->token->key == key("maxDAP"))
        conf->max_daps = d_int(cctx->token);
      else if (cctx->token->key == key("maxDiff"))
        conf->max_diff = d_int(cctx->token);
      else
        return IN3_EIGNORE;
      return IN3_OK;
    }

    case PLGN_ACT_RPC_VERIFY:
      return in3_verify_btc(conf, pctx);

    default:
      return IN3_ENOTSUP;
  }
}


in3_ret_t in3_register_btc(in3_t* c) {
  // init the config with defaults
  btc_target_conf_t* tc = _calloc(1, sizeof(btc_target_conf_t));
  tc->max_daps          = 20;
  tc->max_diff          = 10;
  tc->dap_limit         = 20;

  return plugin_register(c, PLGN_ACT_RPC_VERIFY | PLGN_ACT_TERM | PLGN_ACT_CONFIG_GET | PLGN_ACT_CONFIG_SET, handle_btc, tc, false);
}
```
#### PLGN_ACT_CONFIG_GET

This action will be triggered during `in3_get_config()`and should dump all config from all plugins.

`arguments` : `in3_get_config_ctx_t*` - the config context will hold those data:

```c
typedef struct in3_get_config_ctx {
  in3_t* client; 
  sb_t*  sb;     
} in3_get_config_ctx_t;
```
if you are using any configuration you should use the `sb` field and add your values to it. Each property must start with a comma.

```c
in3_get_config_ctx_t* cctx = args;
sb_add_chars(cctx->sb, ",\"maxDAP\":");
sb_add_int(cctx->sb, conf->max_daps);
sb_add_chars(cctx->sb, ",\"maxDiff\":");
sb_add_int(cctx->sb, conf->max_diff);
```
#### PLGN_ACT_CONFIG_SET

This action will be triggered during the configuration-process. While going through all config-properties, it will ask the plugins in case a config was not handled. So this action may be triggered multiple times. And the plugin should only return `IN3_OK` if it was handled. If no plugin handles it, a error will be thrown.

`arguments` : `in3_configure_ctx_t*` - the cache context will hold those data:

```c
typedef struct in3_configure_ctx {
  in3_t*     client; // the client to configure 
  d_token_t* token;  // the token not handled yet
} in3_configure_ctx_t;
```
In order to check if the token is relevant for you, you simply check the name of the property and handle its value:

```c
in3_configure_ctx_t* cctx = pctx;
if (cctx->token->key == key("maxDAP"))
  conf->max_daps = d_int(cctx->token);
else if (cctx->token->key == key("maxDiff"))
  conf->max_diff = d_int(cctx->token);
else
  return IN3_EIGNORE;
return IN3_OK;
```
### Payment

#### PLGN_ACT_PAY_PREPARE

#### PLGN_ACT_PAY_FOLLOWUP

#### PLGN_ACT_PAY_HANDLE

#### PLGN_ACT_PAY_SIGN_REQ

this will be triggered in order to sign a request. It will provide a request_hash and expects a signature.

`arguments` : `in3_pay_sign_req_ctx_t*` - the sign context will hold those data:

```c
typedef struct {
  in3_req_t* ctx;           
  d_token_t* request;       
  bytes32_t  request_hash;  
  uint8_t    signature[65]; 
} in3_pay_sign_req_ctx_t;
```
It is expected that the plugin will create a signature and write it into the context.

Example:

```c
in3_pay_sign_req_ctx_t* ctx = args;
return ec_sign_pk_raw(ctx->request_hash, pk->key, ctx->signature);
```
### Nodelist

#### PLGN_ACT_NL_PICK_DATA

#### PLGN_ACT_NL_PICK_SIGNER

#### PLGN_ACT_NL_PICK_FOLLOWUP




## Integration of Ledger Nano S

1. Ways to integrate Ledger Nano S
2. Build incubed source with ledger nano module
3. Start using ledger nano s device with Incubed

### Ways to integrate Ledger Nano S

Currently there are two ways to integrate Ledger Nano S with incubed for transaction and message signing:

1. Install Ethereum app from Ledger Manager
2. Setup development environment and install incubed signer app on your Ledger device

Option 1 is the convinient choice for most of the people as incubed signer app is not available to be installed from Ledger Manager and it will take efforts to configure development environment for ledger manager. The main differences in above approaches are following:

 If you are confortable with Option 1 , all you need to do is setup you Ledger device as per usual instructions and install Ethereum app form Ledger Manager store. Otherwise if you are interested in Option 2 Please follow all the instructions given in "Setup development environment for ledger nano s" section .

```c
Ethereum official Ledger app requires rlp encoded transactions for  signing and there is not much scope for customization.Currently we have support for following operations with Ethereum app:
1. Getting public key
2. Sign Transactions
3. Sign Messages

Incubed signer app required just hash , so it is better option if you are looking to integrate incubed in such a way that you would manage all data formation on your end and use just hash to get signture from Ledger Nano S and use the signature as per your wish. 
```
#### Setup development environment for ledger nano s

Setting up dev environment for Ledger nano s is one time activity and incubed signer application will be available to install directly from Ledger Manager in future. Ledger nano applications need linux System (recommended is Ubuntu) to build the binary to be installed on Ledger nano devices

##### Download Toolchains and Nanos ledger SDK (As per latest Ubuntu LTS)

Download the Nano S SDK in bolos-sdk folder 

```sh
$ git clone https://github.com/ledgerhq/nanos-secure-sdk
```
```c
Download a prebuild gcc and move it to bolos-sdk folder
        Ref: https://launchpad.net/gcc-arm-embedded/+milestone/5-2016-q1-update

Download a prebuild clang and rename the folder to clang-arm-fropi then move it to bolos-sdk folder
        Ref: http://releases.llvm.org/download.html#4.0.0 
```
##### Add environment variables:

```sh
sudo -H gedit /etc/environment
```
```c
ADD PATH TO BOLOS SDK:
BOLOS_SDK="<path>/nanos-secure-sdk"

ADD GCCPATH VARIABLE
GCCPATH="<path>/gcc-arm-none-eabi-5_3-2016q1/bin/"

ADD CLANGPATH
CLANGPATH="<path>/clang-arm-fropi/bin/"
```
##### Download and install ledger python tools

Installation prerequisites :

```sh
$ sudo apt-get install libudev-dev <
$ sudo apt-get install libusb-1.0-0-dev 
$ sudo apt-get install python-dev (python 2.7)
$ sudo apt-get install virtualenv
```
##### Installation of ledgerblue:

```sh
$ virtualenv ledger
$ source ledger/bin/activate
$ pip install ledgerblue
```
Ref: [https://github.com/LedgerHQ/blue-loader-python](https://github.com/LedgerHQ/blue-loader-python)

##### Download and install ledger udev rules

 run script from the above download 

```c

```
##### Open new terminal and check for following installations

```sh
$ sudo apt-get install gcc-multilib
$ sudo apt-get install libc6-dev:i386
```
##### Install incubed signer app

Once you complete all the steps, go to folder "c/src/signer/ledger-nano/firmware" and run following command , It will ask you to enter pin for approve installation on ledger nano device. follow all the steps and it will be done.

```sh
make load
```
### Build incubed source with ledger nano module

To build incubed source with ledger nano:-

1. Open root CMakeLists file and find LEDGER_NANO option
2. Turn LEDGER_NANO option ON which is by default OFF
3. Build incubed source 

```sh
cd build
cmake  .. && make
```

### Start using ledger nano s device with Incubed

Open the application on your ledger nano s usb device and make signing requests from incubed.

Following is the sample command to sendTransaction from command line utility:- 

```sh
bin/in3 send -to 0xd46e8dd67c5d32be8058bb8eb970870f07244567  -gas 0x96c0  -value 0x9184e72a  -path 0x2c3c000000 -debug
```
-path points to specific public/private key pair inside HD wallet derivation path . For Ethereum the default path is m/44'/60'/0'/0 , which we can pass in simplified way as hex string i.e [44,60,00,00,00] => 0x2c3c000000

If you want to use apis to integrate ledger nano support in your incubed application , feel free to explore apis given following header files:-

```c
ledger_signer.h : It contains APIs to integrate ledger nano device with incubed signer app.
ethereum_apdu_client.h : It contains APIs to integrate ledger nano device with Ethereum ledger app.
```



## Module api 




### btc_api.h

BTC API. 

This header-file defines easy to use function, which are preparing the JSON-RPC-Request, which is then executed and verified by the incubed-client. 

File: [c/src/api/btc/btc_api.h](https://github.com/slockit/in3-c/blob/master/c/src/api/btc/btc_api.h)

#### btc_last_error ()

< The current error or null if all is ok 

```c
#define btc_last_error () api_last_error()
```


#### btc_transaction_in_t

the tx in 


The stuct contains following fields:

```eval_rst
========================= ================= ==========================
``uint32_t``               **vout**         the tx index of the output
`bytes32_t <#bytes32-t>`_  **txid**         the tx id of the output
``uint32_t``               **sequence**     the sequence
`bytes_t <#bytes-t>`_      **script**       the script
`bytes_t <#bytes-t>`_      **txinwitness**  witnessdata (if used)
========================= ================= ==========================
```

#### btc_transaction_out_t

the tx out 


The stuct contains following fields:

```eval_rst
===================== =================== ================================
``uint64_t``           **value**          the value of the tx
``uint32_t``           **n**              the index
`bytes_t <#bytes-t>`_  **script_pubkey**  the script pubkey (or signature)
===================== =================== ================================
```

#### btc_transaction_t

a transaction 


The stuct contains following fields:

```eval_rst
=================================================== ===================== ======================================================================
``bool``                                             **in_active_chain**  true if it is part of the active chain
`bytes_t <#bytes-t>`_                                **data**             the serialized transaction-data
`bytes32_t <#bytes32-t>`_                            **txid**             the transaction id
`bytes32_t <#bytes32-t>`_                            **hash**             the transaction hash
``uint32_t``                                         **size**             raw size of the transaction
``uint32_t``                                         **vsize**            virtual size of the transaction
``uint32_t``                                         **weight**           weight of the tx
``uint32_t``                                         **version**          used version
``uint32_t``                                         **locktime**         locktime
`btc_transaction_in_t * <#btc-transaction-in-t>`_    **vin**              array of transaction inputs
`btc_transaction_out_t * <#btc-transaction-out-t>`_  **vout**             array of transaction outputs
``uint32_t``                                         **vin_len**          number of tx inputs
``uint32_t``                                         **vout_len**         number of tx outputs
`bytes32_t <#bytes32-t>`_                            **blockhash**        hash of block containing the tx
``uint32_t``                                         **confirmations**    number of confirmations or blocks mined on top of the containing block
``uint32_t``                                         **time**             unix timestamp in seconds since 1970
``uint32_t``                                         **blocktime**        unix timestamp in seconds since 1970
=================================================== ===================== ======================================================================
```

#### btc_blockheader_t

the blockheader 


The stuct contains following fields:

```eval_rst
========================= =================== ======================================================================
`bytes32_t <#bytes32-t>`_  **hash**           the hash of the blockheader
``uint32_t``               **confirmations**  number of confirmations or blocks mined on top of the containing block
``uint32_t``               **height**         block number
``uint32_t``               **version**        used version
`bytes32_t <#bytes32-t>`_  **merkleroot**     merkle root of the trie of all transactions in the block
``uint32_t``               **time**           unix timestamp in seconds since 1970
``uint32_t``               **nonce**          nonce-field of the block
``uint8_t``                **bits**           bits (target) for the block
`bytes32_t <#bytes32-t>`_  **chainwork**      total amount of work since genesis
``uint32_t``               **n_tx**           number of transactions in the block
`bytes32_t <#bytes32-t>`_  **previous_hash**  hash of the parent blockheader
`bytes32_t <#bytes32-t>`_  **next_hash**      hash of the next blockheader
``uint8_t``                **data**           raw serialized header-bytes
========================= =================== ======================================================================
```

#### btc_block_txdata_t

a block with all transactions including their full data 


The stuct contains following fields:

```eval_rst
=========================================== ============ ========================
`btc_blockheader_t <#btc-blockheader-t>`_    **header**  the blockheader
``uint32_t``                                 **tx_len**  number of transactions
`btc_transaction_t * <#btc-transaction-t>`_  **tx**      array of transactiondata
=========================================== ============ ========================
```

#### btc_block_txids_t

a block with all transaction ids 


The stuct contains following fields:

```eval_rst
========================================= ============ ========================
`btc_blockheader_t <#btc-blockheader-t>`_  **header**  the blockheader
``uint32_t``                               **tx_len**  number of transactions
`bytes32_t * <#bytes32-t>`_                **tx**      array of transaction ids
========================================= ============ ========================
```

#### btc_get_transaction_bytes

```c
bytes_t* btc_get_transaction_bytes(in3_t *in3, bytes32_t txid);
```

gets the transaction as raw bytes or null if it does not exist. 

You must free the result with b_free() after use! 

arguments:
```eval_rst
========================= ========== ================
`in3_t * <#in3-t>`_        **in3**   the in3-instance
`bytes32_t <#bytes32-t>`_  **txid**  the txid
========================= ========== ================
```
returns: [`bytes_t *`](#bytes-t)


#### btc_get_transaction

```c
btc_transaction_t* btc_get_transaction(in3_t *in3, bytes32_t txid);
```

gets the transaction as struct or null if it does not exist. 

You must free the result with free() after use! 

arguments:
```eval_rst
========================= ========== ================
`in3_t * <#in3-t>`_        **in3**   the in3-instance
`bytes32_t <#bytes32-t>`_  **txid**  the txid
========================= ========== ================
```
returns: [`btc_transaction_t *`](#btc-transaction-t)


#### btc_get_blockheader

```c
btc_blockheader_t* btc_get_blockheader(in3_t *in3, bytes32_t blockhash);
```

gets the blockheader as struct or null if it does not exist. 

You must free the result with free() after use! 

arguments:
```eval_rst
========================= =============== ================
`in3_t * <#in3-t>`_        **in3**        the in3-instance
`bytes32_t <#bytes32-t>`_  **blockhash**  the block hash
========================= =============== ================
```
returns: [`btc_blockheader_t *`](#btc-blockheader-t)


#### btc_get_blockheader_bytes

```c
bytes_t* btc_get_blockheader_bytes(in3_t *in3, bytes32_t blockhash);
```

gets the blockheader as raw serialized data (80 bytes) or null if it does not exist. 

You must free the result with b_free() after use! 

arguments:
```eval_rst
========================= =============== ================
`in3_t * <#in3-t>`_        **in3**        the in3-instance
`bytes32_t <#bytes32-t>`_  **blockhash**  the block hash
========================= =============== ================
```
returns: [`bytes_t *`](#bytes-t)


#### btc_get_block_txdata

```c
btc_block_txdata_t* btc_get_block_txdata(in3_t *in3, bytes32_t blockhash);
```

gets the block as struct including all transaction data or null if it does not exist. 

You must free the result with free() after use! 

arguments:
```eval_rst
========================= =============== ================
`in3_t * <#in3-t>`_        **in3**        the in3-instance
`bytes32_t <#bytes32-t>`_  **blockhash**  the block hash
========================= =============== ================
```
returns: [`btc_block_txdata_t *`](#btc-block-txdata-t)


#### btc_get_block_txids

```c
btc_block_txids_t* btc_get_block_txids(in3_t *in3, bytes32_t blockhash);
```

gets the block as struct including all transaction ids or null if it does not exist. 

You must free the result with free() after use! 

arguments:
```eval_rst
========================= =============== ================
`in3_t * <#in3-t>`_        **in3**        the in3-instance
`bytes32_t <#bytes32-t>`_  **blockhash**  the block hash
========================= =============== ================
```
returns: [`btc_block_txids_t *`](#btc-block-txids-t)


#### btc_get_block_bytes

```c
bytes_t* btc_get_block_bytes(in3_t *in3, bytes32_t blockhash);
```

gets the block as raw serialized block bytes including all transactions or null if it does not exist. 

You must free the result with b_free() after use! 

arguments:
```eval_rst
========================= =============== ================
`in3_t * <#in3-t>`_        **in3**        the in3-instance
`bytes32_t <#bytes32-t>`_  **blockhash**  the block hash
========================= =============== ================
```
returns: [`bytes_t *`](#bytes-t)


#### btc_d_to_tx

```c
btc_transaction_t* btc_d_to_tx(d_token_t *t);
```

Deserialization helpers. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **t**  
=========================== ======= 
```
returns: [`btc_transaction_t *`](#btc-transaction-t)


#### btc_d_to_blockheader

```c
btc_blockheader_t* btc_d_to_blockheader(d_token_t *t);
```

Deserializes a `btc_transaction_t` type. 

You must free the result with free() after use! 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **t**  
=========================== ======= 
```
returns: [`btc_blockheader_t *`](#btc-blockheader-t)


#### btc_d_to_block_txids

```c
btc_block_txids_t* btc_d_to_block_txids(d_token_t *t);
```

Deserializes a `btc_blockheader_t` type. 

You must free the result with free() after use! 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **t**  
=========================== ======= 
```
returns: [`btc_block_txids_t *`](#btc-block-txids-t)


#### btc_d_to_block_txdata

```c
btc_block_txdata_t* btc_d_to_block_txdata(d_token_t *t);
```

Deserializes a `btc_block_txids_t` type. 

You must free the result with free() after use! 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **t**  
=========================== ======= 
```
returns: [`btc_block_txdata_t *`](#btc-block-txdata-t)


### core_api.h

Ethereum API. 

This header-file defines easy to use function, which are preparing the JSON-RPC-Request, which is then executed and verified by the incubed-client. 

File: [c/src/api/core/core_api.h](https://github.com/slockit/in3-c/blob/master/c/src/api/core/core_api.h)

#### in3_register_core_api

```c
in3_ret_t in3_register_core_api(in3_t *c);
```

register core-api 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### eth_api.h

Ethereum API. 

This header-file defines easy to use function, which are preparing the JSON-RPC-Request, which is then executed and verified by the incubed-client. 

File: [c/src/api/eth1/eth_api.h](https://github.com/slockit/in3-c/blob/master/c/src/api/eth1/eth_api.h)

#### BLKNUM (blk)

Initializer macros for eth_blknum_t. 

```c
#define BLKNUM (blk) ((eth_blknum_t){.u64 = blk, .is_u64 = true})
```


#### BLKNUM_LATEST ()

```c
#define BLKNUM_LATEST () ((eth_blknum_t){.def = BLK_LATEST, .is_u64 = false})
```


#### BLKNUM_EARLIEST ()

```c
#define BLKNUM_EARLIEST () ((eth_blknum_t){.def = BLK_EARLIEST, .is_u64 = false})
```


#### BLKNUM_PENDING ()

The current error or null if all is ok. 

```c
#define BLKNUM_PENDING () ((eth_blknum_t){.def = BLK_PENDING, .is_u64 = false})
```


#### eth_last_error ()

```c
#define eth_last_error () api_last_error()
```


#### eth_blknum_def_t

Abstract type for holding a block number. 

The enum type contains the following values:

```eval_rst
================== = 
 **BLK_LATEST**    0 
 **BLK_EARLIEST**  1 
 **BLK_PENDING**   2 
================== = 
```

#### eth_tx_t

A transaction. 


The stuct contains following fields:

```eval_rst
========================= ======================= =========================================
`bytes32_t <#bytes32-t>`_  **hash**               the blockhash
`bytes32_t <#bytes32-t>`_  **block_hash**         hash of ther containnig block
``uint64_t``               **block_number**       number of the containing block
`address_t <#address-t>`_  **from**               sender of the tx
``uint64_t``               **gas**                gas send along
``uint64_t``               **gas_price**          gas price used
`bytes_t <#bytes-t>`_      **data**               data send along with the transaction
``uint64_t``               **nonce**              nonce of the transaction
`address_t <#address-t>`_  **to**                 receiver of the address 0x0000. 
                                                  
                                                  . -Address is used for contract creation.
`uint256_t <#uint256-t>`_  **value**              the value in wei send
``int``                    **transaction_index**  the transaction index
``uint8_t``                **signature**          signature of the transaction
========================= ======================= =========================================
```

#### eth_block_t

An Ethereum Block. 


The stuct contains following fields:

```eval_rst
=========================== ======================= ====================================================
``uint64_t``                 **number**             the blockNumber
`bytes32_t <#bytes32-t>`_    **hash**               the blockhash
``uint64_t``                 **gasUsed**            gas used by all the transactions
``uint64_t``                 **gasLimit**           gasLimit
`address_t <#address-t>`_    **author**             the author of the block.
`uint256_t <#uint256-t>`_    **difficulty**         the difficulty of the block.
`bytes_t <#bytes-t>`_        **extra_data**         the extra_data of the block.
``uint8_t``                  **logsBloom**          the logsBloom-data
`bytes32_t <#bytes32-t>`_    **parent_hash**        the hash of the parent-block
`bytes32_t <#bytes32-t>`_    **sha3_uncles**        root hash of the uncle-trie
`bytes32_t <#bytes32-t>`_    **state_root**         root hash of the state-trie
`bytes32_t <#bytes32-t>`_    **receipts_root**      root of the receipts trie
`bytes32_t <#bytes32-t>`_    **transaction_root**   root of the transaction trie
``int``                      **tx_count**           number of transactions in the block
`eth_tx_t * <#eth-tx-t>`_    **tx_data**            array of transaction data or NULL if not requested
`bytes32_t * <#bytes32-t>`_  **tx_hashes**          array of transaction hashes or NULL if not requested
``uint64_t``                 **timestamp**          the unix timestamp of the block
`bytes_t * <#bytes-t>`_      **seal_fields**        sealed fields
``int``                      **seal_fields_count**  number of seal fields
=========================== ======================= ====================================================
```

#### eth_log_t

A linked list of Ethereum Logs 


 


The stuct contains following fields:

```eval_rst
=============================== ======================= ==============================================================
``bool``                         **removed**            true when the log was removed, due to a chain reorganization. 
                                                        
                                                        false if its a valid log
``size_t``                       **log_index**          log index position in the block
``size_t``                       **transaction_index**  transactions index position log was created from
`bytes32_t <#bytes32-t>`_        **transaction_hash**   hash of the transactions this log was created from
`bytes32_t <#bytes32-t>`_        **block_hash**         hash of the block where this log was in
``uint64_t``                     **block_number**       the block number where this log was in
`address_t <#address-t>`_        **address**            address from which this log originated
`bytes_t <#bytes-t>`_            **data**               non-indexed arguments of the log
`bytes32_t * <#bytes32-t>`_      **topics**             array of 0 to 4 32 Bytes DATA of indexed log arguments
``size_t``                       **topic_count**        counter for topics
`eth_logstruct , * <#eth-log>`_  **next**               pointer to next log in list or NULL
=============================== ======================= ==============================================================
```

#### eth_tx_receipt_t

A transaction receipt. 


The stuct contains following fields:

```eval_rst
=========================== ========================= =============================================================================
`bytes32_t <#bytes32-t>`_    **transaction_hash**     the transaction hash
``int``                      **transaction_index**    the transaction index
`bytes32_t <#bytes32-t>`_    **block_hash**           hash of ther containnig block
``uint64_t``                 **block_number**         number of the containing block
``uint64_t``                 **cumulative_gas_used**  total amount of gas used by block
``uint64_t``                 **gas_used**             amount of gas used by this specific transaction
`bytes_t * <#bytes-t>`_      **contract_address**     contract address created (if the transaction was a contract creation) or NULL
``bool``                     **status**               1 if transaction succeeded, 0 otherwise.
`eth_log_t * <#eth-log-t>`_  **logs**                 array of log objects, which this transaction generated
=========================== ========================= =============================================================================
```

#### DEFINE_OPTIONAL_T

```c
DEFINE_OPTIONAL_T(uint64_t);
```

Optional types. 

arguments:
```eval_rst
============  
``uint64_t``  
============  
```
returns: ``


#### DEFINE_OPTIONAL_T

```c
DEFINE_OPTIONAL_T(bytes_t);
```

arguments:
```eval_rst
=====================  
`bytes_t <#bytes-t>`_  
=====================  
```
returns: ``


#### DEFINE_OPTIONAL_T

```c
DEFINE_OPTIONAL_T(address_t);
```

arguments:
```eval_rst
=========================  
`address_t <#address-t>`_  
=========================  
```
returns: ``


#### DEFINE_OPTIONAL_T

```c
DEFINE_OPTIONAL_T(uint256_t);
```

arguments:
```eval_rst
=========================  
`uint256_t <#uint256-t>`_  
=========================  
```
returns: ``


#### eth_getStorageAt

```c
uint256_t eth_getStorageAt(in3_t *in3, address_t account, bytes32_t key, eth_blknum_t block);
```

Returns the storage value of a given address. 

arguments:
```eval_rst
=============================== ============= 
`in3_t * <#in3-t>`_              **in3**      
`address_t <#address-t>`_        **account**  
`bytes32_t <#bytes32-t>`_        **key**      
`eth_blknum_t <#eth-blknum-t>`_  **block**    
=============================== ============= 
```
returns: [`uint256_t`](#uint256-t)


#### eth_getCode

```c
bytes_t eth_getCode(in3_t *in3, address_t account, eth_blknum_t block);
```

Returns the code of the account of given address. 

(Make sure you free the data-point of the result after use.) 

arguments:
```eval_rst
=============================== ============= 
`in3_t * <#in3-t>`_              **in3**      
`address_t <#address-t>`_        **account**  
`eth_blknum_t <#eth-blknum-t>`_  **block**    
=============================== ============= 
```
returns: [`bytes_t`](#bytes-t)


#### eth_getBalance

```c
uint256_t eth_getBalance(in3_t *in3, address_t account, eth_blknum_t block);
```

Returns the balance of the account of given address. 

arguments:
```eval_rst
=============================== ============= 
`in3_t * <#in3-t>`_              **in3**      
`address_t <#address-t>`_        **account**  
`eth_blknum_t <#eth-blknum-t>`_  **block**    
=============================== ============= 
```
returns: [`uint256_t`](#uint256-t)


#### eth_blockNumber

```c
uint64_t eth_blockNumber(in3_t *in3);
```

Returns the current blockNumber, if bn==0 an error occured and you should check eth_last_error() 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: `uint64_t`


#### eth_gasPrice

```c
uint64_t eth_gasPrice(in3_t *in3);
```

Returns the current price per gas in wei. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: `uint64_t`


#### eth_getBlockByNumber

```c
eth_block_t* eth_getBlockByNumber(in3_t *in3, eth_blknum_t number, bool include_tx);
```

Returns the block for the given number (if number==0, the latest will be returned). 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
=============================== ================ 
`in3_t * <#in3-t>`_              **in3**         
`eth_blknum_t <#eth-blknum-t>`_  **number**      
``bool``                         **include_tx**  
=============================== ================ 
```
returns: [`eth_block_t *`](#eth-block-t)


#### eth_getBlockByHash

```c
eth_block_t* eth_getBlockByHash(in3_t *in3, bytes32_t hash, bool include_tx);
```

Returns the block for the given hash. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
========================= ================ 
`in3_t * <#in3-t>`_        **in3**         
`bytes32_t <#bytes32-t>`_  **hash**        
``bool``                   **include_tx**  
========================= ================ 
```
returns: [`eth_block_t *`](#eth-block-t)


#### eth_getLogs

```c
eth_log_t* eth_getLogs(in3_t *in3, char *fopt);
```

Returns a linked list of logs. 

If result is null, check eth_last_error()! otherwise make sure to free the log, its topics and data after using it! 

arguments:
```eval_rst
=================== ========== 
`in3_t * <#in3-t>`_  **in3**   
``char *``           **fopt**  
=================== ========== 
```
returns: [`eth_log_t *`](#eth-log-t)


#### eth_newFilter

```c
in3_ret_t eth_newFilter(in3_t *in3, json_ctx_t *options);
```

Creates a new event filter with specified options and returns its id (>0) on success or 0 on failure. 

arguments:
```eval_rst
============================= ============= 
`in3_t * <#in3-t>`_            **in3**      
`json_ctx_t * <#json-ctx-t>`_  **options**  
============================= ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_newBlockFilter

```c
in3_ret_t eth_newBlockFilter(in3_t *in3);
```

Creates a new block filter with specified options and returns its id (>0) on success or 0 on failure. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_newPendingTransactionFilter

```c
in3_ret_t eth_newPendingTransactionFilter(in3_t *in3);
```

Creates a new pending txn filter with specified options and returns its id on success or 0 on failure. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_uninstallFilter

```c
bool eth_uninstallFilter(in3_t *in3, size_t id);
```

Uninstalls a filter and returns true on success or false on failure. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
``size_t``           **id**   
=================== ========= 
```
returns: `bool`


#### eth_getFilterChanges

```c
in3_ret_t eth_getFilterChanges(in3_t *in3, size_t id, bytes32_t **block_hashes, eth_log_t **logs);
```

Sets the logs (for event filter) or blockhashes (for block filter) that match a filter; returns <0 on error, otherwise no. 

of block hashes matched (for block filter) or 0 (for log filter) 

arguments:
```eval_rst
============================ ================== 
`in3_t * <#in3-t>`_           **in3**           
``size_t``                    **id**            
`bytes32_t ** <#bytes32-t>`_  **block_hashes**  
`eth_log_t ** <#eth-log-t>`_  **logs**          
============================ ================== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_getFilterLogs

```c
in3_ret_t eth_getFilterLogs(in3_t *in3, size_t id, eth_log_t **logs);
```

Sets the logs (for event filter) or blockhashes (for block filter) that match a filter; returns <0 on error, otherwise no. 

of block hashes matched (for block filter) or 0 (for log filter) 

arguments:
```eval_rst
============================ ========== 
`in3_t * <#in3-t>`_           **in3**   
``size_t``                    **id**    
`eth_log_t ** <#eth-log-t>`_  **logs**  
============================ ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_chainId

```c
uint64_t eth_chainId(in3_t *in3);
```

Returns the currently configured chain id. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: `uint64_t`


#### eth_getBlockTransactionCountByHash

```c
uint64_t eth_getBlockTransactionCountByHash(in3_t *in3, bytes32_t hash);
```

Returns the number of transactions in a block from a block matching the given block hash. 

arguments:
```eval_rst
========================= ========== 
`in3_t * <#in3-t>`_        **in3**   
`bytes32_t <#bytes32-t>`_  **hash**  
========================= ========== 
```
returns: `uint64_t`


#### eth_getBlockTransactionCountByNumber

```c
uint64_t eth_getBlockTransactionCountByNumber(in3_t *in3, eth_blknum_t block);
```

Returns the number of transactions in a block from a block matching the given block number. 

arguments:
```eval_rst
=============================== =========== 
`in3_t * <#in3-t>`_              **in3**    
`eth_blknum_t <#eth-blknum-t>`_  **block**  
=============================== =========== 
```
returns: `uint64_t`


#### eth_call_fn

```c
json_ctx_t* eth_call_fn(in3_t *in3, address_t contract, eth_blknum_t block, char *fn_sig,...);
```

Returns the result of a function_call. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it with json_free()! 

arguments:
```eval_rst
=============================== ============== 
`in3_t * <#in3-t>`_              **in3**       
`address_t <#address-t>`_        **contract**  
`eth_blknum_t <#eth-blknum-t>`_  **block**     
``char *``                       **fn_sig**    
``...``                                        
=============================== ============== 
```
returns: [`json_ctx_t *`](#json-ctx-t)


#### eth_estimate_fn

```c
uint64_t eth_estimate_fn(in3_t *in3, address_t contract, eth_blknum_t block, char *fn_sig,...);
```

Returns the result of a function_call. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it with json_free()! 

arguments:
```eval_rst
=============================== ============== 
`in3_t * <#in3-t>`_              **in3**       
`address_t <#address-t>`_        **contract**  
`eth_blknum_t <#eth-blknum-t>`_  **block**     
``char *``                       **fn_sig**    
``...``                                        
=============================== ============== 
```
returns: `uint64_t`


#### eth_getTransactionByHash

```c
eth_tx_t* eth_getTransactionByHash(in3_t *in3, bytes32_t tx_hash);
```

Returns the information about a transaction requested by transaction hash. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
========================= ============= 
`in3_t * <#in3-t>`_        **in3**      
`bytes32_t <#bytes32-t>`_  **tx_hash**  
========================= ============= 
```
returns: [`eth_tx_t *`](#eth-tx-t)


#### eth_getTransactionByBlockHashAndIndex

```c
eth_tx_t* eth_getTransactionByBlockHashAndIndex(in3_t *in3, bytes32_t block_hash, size_t index);
```

Returns the information about a transaction by block hash and transaction index position. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
========================= ================ 
`in3_t * <#in3-t>`_        **in3**         
`bytes32_t <#bytes32-t>`_  **block_hash**  
``size_t``                 **index**       
========================= ================ 
```
returns: [`eth_tx_t *`](#eth-tx-t)


#### eth_getTransactionByBlockNumberAndIndex

```c
eth_tx_t* eth_getTransactionByBlockNumberAndIndex(in3_t *in3, eth_blknum_t block, size_t index);
```

Returns the information about a transaction by block number and transaction index position. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
=============================== =========== 
`in3_t * <#in3-t>`_              **in3**    
`eth_blknum_t <#eth-blknum-t>`_  **block**  
``size_t``                       **index**  
=============================== =========== 
```
returns: [`eth_tx_t *`](#eth-tx-t)


#### eth_getTransactionCount

```c
uint64_t eth_getTransactionCount(in3_t *in3, address_t address, eth_blknum_t block);
```

Returns the number of transactions sent from an address. 

arguments:
```eval_rst
=============================== ============= 
`in3_t * <#in3-t>`_              **in3**      
`address_t <#address-t>`_        **address**  
`eth_blknum_t <#eth-blknum-t>`_  **block**    
=============================== ============= 
```
returns: `uint64_t`


#### eth_getUncleByBlockNumberAndIndex

```c
eth_block_t* eth_getUncleByBlockNumberAndIndex(in3_t *in3, eth_blknum_t block, size_t index);
```

Returns information about a uncle of a block by number and uncle index position. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
=============================== =========== 
`in3_t * <#in3-t>`_              **in3**    
`eth_blknum_t <#eth-blknum-t>`_  **block**  
``size_t``                       **index**  
=============================== =========== 
```
returns: [`eth_block_t *`](#eth-block-t)


#### eth_getUncleCountByBlockHash

```c
uint64_t eth_getUncleCountByBlockHash(in3_t *in3, bytes32_t hash);
```

Returns the number of uncles in a block from a block matching the given block hash. 

arguments:
```eval_rst
========================= ========== 
`in3_t * <#in3-t>`_        **in3**   
`bytes32_t <#bytes32-t>`_  **hash**  
========================= ========== 
```
returns: `uint64_t`


#### eth_getUncleCountByBlockNumber

```c
uint64_t eth_getUncleCountByBlockNumber(in3_t *in3, eth_blknum_t block);
```

Returns the number of uncles in a block from a block matching the given block number. 

arguments:
```eval_rst
=============================== =========== 
`in3_t * <#in3-t>`_              **in3**    
`eth_blknum_t <#eth-blknum-t>`_  **block**  
=============================== =========== 
```
returns: `uint64_t`


#### eth_sendTransaction

```c
bytes_t* eth_sendTransaction(in3_t *in3, address_t from, address_t to, OPTIONAL_T(uint64_t) gas, OPTIONAL_T(uint64_t) gas_price, OPTIONAL_T(uint256_t) value, OPTIONAL_T(bytes_t) data, OPTIONAL_T(uint64_t) nonce);
```

Creates new message call transaction or a contract creation. 

Returns (32 Bytes) - the transaction hash, or the zero hash if the transaction is not yet available. Free result after use with b_free(). 

arguments:
```eval_rst
===================================== =============== 
`in3_t * <#in3-t>`_                    **in3**        
`address_t <#address-t>`_              **from**       
`address_t <#address-t>`_              **to**         
`OPTIONAL_T(uint64_t) <#optional-t>`_  **gas**        
`OPTIONAL_T(uint64_t) <#optional-t>`_  **gas_price**  
``(,)``                                **value**      
``(,)``                                **data**       
`OPTIONAL_T(uint64_t) <#optional-t>`_  **nonce**      
===================================== =============== 
```
returns: [`bytes_t *`](#bytes-t)


#### eth_sendRawTransaction

```c
bytes_t* eth_sendRawTransaction(in3_t *in3, bytes_t data);
```

Creates new message call transaction or a contract creation for signed transactions. 

Returns (32 Bytes) - the transaction hash, or the zero hash if the transaction is not yet available. Free after use with b_free(). 

arguments:
```eval_rst
===================== ========== 
`in3_t * <#in3-t>`_    **in3**   
`bytes_t <#bytes-t>`_  **data**  
===================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### eth_getTransactionReceipt

```c
eth_tx_receipt_t* eth_getTransactionReceipt(in3_t *in3, bytes32_t tx_hash);
```

Returns the receipt of a transaction by transaction hash. 

Free result after use with eth_tx_receipt_free() 

arguments:
```eval_rst
========================= ============= 
`in3_t * <#in3-t>`_        **in3**      
`bytes32_t <#bytes32-t>`_  **tx_hash**  
========================= ============= 
```
returns: [`eth_tx_receipt_t *`](#eth-tx-receipt-t)


#### eth_wait_for_receipt

```c
char* eth_wait_for_receipt(in3_t *in3, bytes32_t tx_hash);
```

Waits for receipt of a transaction requested by transaction hash. 

arguments:
```eval_rst
========================= ============= 
`in3_t * <#in3-t>`_        **in3**      
`bytes32_t <#bytes32-t>`_  **tx_hash**  
========================= ============= 
```
returns: `char *`


#### eth_log_free

```c
void eth_log_free(eth_log_t *log);
```

Frees a eth_log_t object. 

arguments:
```eval_rst
=========================== ========= 
`eth_log_t * <#eth-log-t>`_  **log**  
=========================== ========= 
```

#### eth_tx_receipt_free

```c
void eth_tx_receipt_free(eth_tx_receipt_t *txr);
```

Frees a eth_tx_receipt_t object. 

arguments:
```eval_rst
========================================= ========= 
`eth_tx_receipt_t * <#eth-tx-receipt-t>`_  **txr**  
========================================= ========= 
```

#### string_val_to_bytes

```c
int string_val_to_bytes(char *val, char *unit, bytes32_t target);
```

reades the string as hex or decimal and converts it into bytes. 

the value may also contains a suffix as unit like '1.5eth` which will convert it into wei. the target-pointer must be at least as big as the strlen. The length of the bytes will be returned or a negative value in case of an error. 

arguments:
```eval_rst
========================= ============ 
``char *``                 **val**     
``char *``                 **unit**    
`bytes32_t <#bytes32-t>`_  **target**  
========================= ============ 
```
returns: `int`


#### bytes_to_string_val

```c
char* bytes_to_string_val(bytes_t wei, int exp, int digits);
```

converts the bytes value to a decimal string 

arguments:
```eval_rst
===================== ============ 
`bytes_t <#bytes-t>`_  **wei**     
``int``                **exp**     
``int``                **digits**  
===================== ============ 
```
returns: `char *`


#### in3_register_eth_api

```c
in3_ret_t in3_register_eth_api(in3_t *c);
```

this function should only be called once and will register the eth-API verifier. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### ipfs_api.h

IPFS API. 

This header-file defines easy to use function, which are preparing the JSON-RPC-Request, which is then executed and verified by the incubed-client. 

File: [c/src/api/ipfs/ipfs_api.h](https://github.com/slockit/in3-c/blob/master/c/src/api/ipfs/ipfs_api.h)

#### ipfs_put

```c
char* ipfs_put(in3_t *in3, const bytes_t *content);
```

Returns the IPFS multihash of stored content on success OR NULL on error (check api_last_error()). 

Result must be freed by caller. 

arguments:
```eval_rst
============================== ============= 
`in3_t * <#in3-t>`_             **in3**      
`bytes_tconst , * <#bytes-t>`_  **content**  
============================== ============= 
```
returns: `char *`


#### ipfs_get

```c
bytes_t* ipfs_get(in3_t *in3, const char *multihash);
```

Returns the content associated with specified multihash on success OR NULL on error (check api_last_error()). 

Result must be freed by caller. 

arguments:
```eval_rst
=================== =============== 
`in3_t * <#in3-t>`_  **in3**        
``const char *``     **multihash**  
=================== =============== 
```
returns: [`bytes_t *`](#bytes-t)


### usn_api.h

USN API. 

This header-file defines easy to use function, which are verifying USN-Messages. 

File: [c/src/api/usn/usn_api.h](https://github.com/slockit/in3-c/blob/master/c/src/api/usn/usn_api.h)

#### usn_msg_type_t

The enum type contains the following values:

```eval_rst
================== = 
 **USN_ACTION**    0 
 **USN_REQUEST**   1 
 **USN_RESPONSE**  2 
================== = 
```

#### usn_event_type_t

The enum type contains the following values:

```eval_rst
=================== = 
 **BOOKING_NONE**   0 
 **BOOKING_START**  1 
 **BOOKING_STOP**   2 
=================== = 
```

#### usn_booking_handler


```c
typedef int(* usn_booking_handler) (usn_event_t *)
```

returns: `int(*`


#### usn_verify_message

```c
usn_msg_result_t usn_verify_message(usn_device_conf_t *conf, char *message);
```

arguments:
```eval_rst
=========================================== ============= 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**     
``char *``                                   **message**  
=========================================== ============= 
```
returns: [`usn_msg_result_t`](#usn-msg-result-t)


#### usn_register_device

```c
in3_ret_t usn_register_device(usn_device_conf_t *conf, char *url);
```

arguments:
```eval_rst
=========================================== ========== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**  
``char *``                                   **url**   
=========================================== ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### usn_parse_url

```c
usn_url_t usn_parse_url(char *url);
```

arguments:
```eval_rst
========== ========= 
``char *``  **url**  
========== ========= 
```
returns: [`usn_url_t`](#usn-url-t)


#### usn_update_state

```c
unsigned int usn_update_state(usn_device_conf_t *conf, unsigned int wait_time);
```

arguments:
```eval_rst
=========================================== =============== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**       
``unsigned int``                             **wait_time**  
=========================================== =============== 
```
returns: `unsigned int`


#### usn_update_bookings

```c
in3_ret_t usn_update_bookings(usn_device_conf_t *conf);
```

arguments:
```eval_rst
=========================================== ========== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**  
=========================================== ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### usn_remove_old_bookings

```c
void usn_remove_old_bookings(usn_device_conf_t *conf);
```

arguments:
```eval_rst
=========================================== ========== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**  
=========================================== ========== 
```

#### usn_get_next_event

```c
usn_event_t usn_get_next_event(usn_device_conf_t *conf);
```

arguments:
```eval_rst
=========================================== ========== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**  
=========================================== ========== 
```
returns: [`usn_event_t`](#usn-event-t)


#### usn_rent

```c
in3_ret_t usn_rent(in3_t *c, address_t contract, address_t token, char *url, uint32_t seconds, bytes32_t tx_hash);
```

arguments:
```eval_rst
========================= ============== 
`in3_t * <#in3-t>`_        **c**         
`address_t <#address-t>`_  **contract**  
`address_t <#address-t>`_  **token**     
``char *``                 **url**       
``uint32_t``               **seconds**   
`bytes32_t <#bytes32-t>`_  **tx_hash**   
========================= ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### usn_return

```c
in3_ret_t usn_return(in3_t *c, address_t contract, char *url, bytes32_t tx_hash);
```

arguments:
```eval_rst
========================= ============== 
`in3_t * <#in3-t>`_        **c**         
`address_t <#address-t>`_  **contract**  
``char *``                 **url**       
`bytes32_t <#bytes32-t>`_  **tx_hash**   
========================= ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### usn_price

```c
in3_ret_t usn_price(in3_t *c, address_t contract, address_t token, char *url, uint32_t seconds, address_t controller, bytes32_t price);
```

arguments:
```eval_rst
========================= ================ 
`in3_t * <#in3-t>`_        **c**           
`address_t <#address-t>`_  **contract**    
`address_t <#address-t>`_  **token**       
``char *``                 **url**         
``uint32_t``               **seconds**     
`address_t <#address-t>`_  **controller**  
`bytes32_t <#bytes32-t>`_  **price**       
========================= ================ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### api_utils.h

Ethereum API utils. 

This header-file helper utils for use with API modules. 

File: [c/src/api/utils/api_utils.h](https://github.com/slockit/in3-c/blob/master/c/src/api/utils/api_utils.h)

#### set_error_fn

function to set error. 

Will only be called internally. default implementation is NOT MT safe! 


```c
typedef void(* set_error_fn) (int err, const char *msg)
```


#### get_error_fn

function to get last error message. 

default implementation is NOT MT safe! 


```c
typedef char*(* get_error_fn) (void)
```

returns: `char *(*`


#### as_double

```c
long double as_double(uint256_t d);
```

Converts a uint256_t in a long double. 

Important: since a long double stores max 16 byte, there is no guarantee to have the full precision.

Converts a uint256_t in a long double. 

arguments:
```eval_rst
========================= ======= 
`uint256_t <#uint256-t>`_  **d**  
========================= ======= 
```
returns: `long double`


#### as_long

```c
uint64_t as_long(uint256_t d);
```

Converts a uint256_t in a long . 

Important: since a long double stores 8 byte, this will only use the last 8 byte of the value.

Converts a uint256_t in a long . 

arguments:
```eval_rst
========================= ======= 
`uint256_t <#uint256-t>`_  **d**  
========================= ======= 
```
returns: `uint64_t`


#### to_uint256

```c
uint256_t to_uint256(uint64_t value);
```

Converts a uint64_t into its uint256_t representation. 

arguments:
```eval_rst
============ =========== 
``uint64_t``  **value**  
============ =========== 
```
returns: [`uint256_t`](#uint256-t)


#### decrypt_key

```c
in3_ret_t decrypt_key(d_token_t *key_data, char *password, bytes32_t dst);
```

Decrypts the private key from a json keystore file using PBKDF2 or SCRYPT (if enabled) 

arguments:
```eval_rst
=========================== ============== 
`d_token_t * <#d-token-t>`_  **key_data**  
``char *``                   **password**  
`bytes32_t <#bytes32-t>`_    **dst**       
=========================== ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### to_checksum

```c
in3_ret_t to_checksum(address_t adr, chain_id_t chain_id, char out[43]);
```

converts the given address to a checksum address. 

If chain_id is passed, it will use the EIP1191 to include it as well. 

arguments:
```eval_rst
=========================== ============== 
`address_t <#address-t>`_    **adr**       
`chain_id_t <#chain-id-t>`_  **chain_id**  
``char``                     **out**       
=========================== ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### api_set_error_fn

```c
void api_set_error_fn(set_error_fn fn);
```

arguments:
```eval_rst
=============================== ======== 
`set_error_fn <#set-error-fn>`_  **fn**  
=============================== ======== 
```

#### api_get_error_fn

```c
void api_get_error_fn(get_error_fn fn);
```

arguments:
```eval_rst
=============================== ======== 
`get_error_fn <#get-error-fn>`_  **fn**  
=============================== ======== 
```

#### api_last_error

```c
char* api_last_error();
```

returns current error or null if all is ok 

returns: `char *`


## Module core 




### client.h

this file defines the incubed configuration struct and it registration. 

File: [c/src/core/client/client.h](https://github.com/slockit/in3-c/blob/master/c/src/core/client/client.h)

#### IN3_PROTO_VER

the protocol version used when sending requests from the this client 

```c
#define IN3_PROTO_VER "2.1.0"
```


#### CHAIN_ID_MAINNET

chain_id for mainnet 

```c
#define CHAIN_ID_MAINNET 0x01
```


#### CHAIN_ID_GOERLI

chain_id for goerlii 

```c
#define CHAIN_ID_GOERLI 0x5
```


#### CHAIN_ID_EWC

chain_id for ewc 

```c
#define CHAIN_ID_EWC 0xf6
```


#### CHAIN_ID_IPFS

chain_id for ipfs 

```c
#define CHAIN_ID_IPFS 0x7d0
```


#### CHAIN_ID_BTC

chain_id for btc 

```c
#define CHAIN_ID_BTC 0x99
```


#### CHAIN_ID_LOCAL

chain_id for local chain 

```c
#define CHAIN_ID_LOCAL 0x11
```


#### DEF_REPL_LATEST_BLK

default replace_latest_block 

```c
#define DEF_REPL_LATEST_BLK 6
```


#### PLGN_ACT_LIFECYCLE

```c
#define PLGN_ACT_LIFECYCLE (PLGN_ACT_INIT | PLGN_ACT_TERM)
```


#### PLGN_ACT_TRANSPORT

```c
#define PLGN_ACT_TRANSPORT (PLGN_ACT_TRANSPORT_SEND | PLGN_ACT_TRANSPORT_RECEIVE | PLGN_ACT_TRANSPORT_CLEAN)
```


#### PLGN_ACT_NODELIST

```c
#define PLGN_ACT_NODELIST (PLGN_ACT_NL_PICK | PLGN_ACT_NL_PICK_FOLLOWUP | PLGN_ACT_NL_BLACKLIST | PLGN_ACT_NL_FAILABLE | PLGN_ACT_NL_OFFLINE)
```


#### PLGN_ACT_CACHE

```c
#define PLGN_ACT_CACHE (PLGN_ACT_CACHE_SET | PLGN_ACT_CACHE_GET | PLGN_ACT_CACHE_CLEAR)
```


#### PLGN_ACT_CONFIG

```c
#define PLGN_ACT_CONFIG (PLGN_ACT_CONFIG_SET | PLGN_ACT_CONFIG_GET)
```


#### in3_for_chain (chain_id)

creates a new Incubed configuration for a specified chain and returns the pointer. 

when creating the client only the one chain will be configured. (saves memory). but if you pass `CHAIN_ID_MULTICHAIN` as argument all known chains will be configured allowing you to switch between chains within the same client or configuring your own chain.

you need to free this instance with `in3_free` after use!

Before using the client you still need to set the transport and optional the storage handlers:

- example of initialization: 

```c
// register verifiers
in3_register_eth_full();

// create new client
in3_t* client = in3_for_chain(CHAIN_ID_MAINNET);

// configure transport
client->transport    = send_curl;

// configure storage
in3_set_storage_handler(c, storage_get_item, storage_set_item, storage_clear, NULL);

// ready to use ...
```

```c
#define in3_for_chain (chain_id) in3_for_chain_default(chain_id)
```


#### assert_in3 (c)

```c
#define assert_in3 (c)   assert(c);                                           \
  assert((c)->chain.chain_id);                         \
  assert((c)->plugins);                                \
  assert((c)->max_attempts > 0);                       \
  assert((c)->proof >= 0 && (c)->proof <= PROOF_FULL); \
  assert((c)->proof >= 0 && (c)->proof <= PROOF_FULL);
```


#### in3_chain_type_t

the type of the chain. 

for incubed a chain can be any distributed network or database with incubed support. Depending on this chain-type the previously registered verifier will be chosen and used. 

The enum type contains the following values:

```eval_rst
===================== = =================
 **CHAIN_ETH**        0 Ethereum chain.
 **CHAIN_SUBSTRATE**  1 substrate chain
 **CHAIN_IPFS**       2 ipfs verification
 **CHAIN_BTC**        3 Bitcoin chain.
 **CHAIN_EOS**        4 EOS chain.
 **CHAIN_IOTA**       5 IOTA chain.
 **CHAIN_GENERIC**    6 other chains
===================== = =================
```

#### in3_proof_t

the type of proof. 

Depending on the proof-type different levels of proof will be requested from the node. 

The enum type contains the following values:

```eval_rst
==================== = ==================================================
 **PROOF_NONE**      0 No Verification.
 **PROOF_STANDARD**  1 Standard Verification of the important properties.
 **PROOF_FULL**      2 All field will be validated including uncles.
==================== = ==================================================
```

#### in3_flags_type_t

a list of flags defining the behavior of the incubed client. 

They should be used as bitmask for the flags-property. 

The enum type contains the following values:

```eval_rst
============================== ===== =============================================================================================
 **FLAGS_KEEP_IN3**            0x1   the in3-section with the proof will also returned
 **FLAGS_AUTO_UPDATE_LIST**    0x2   the nodelist will be automatically updated if the last_block is newer
 **FLAGS_INCLUDE_CODE**        0x4   the code is included when sending eth_call-requests
 **FLAGS_BINARY**              0x8   the client will use binary format
 **FLAGS_HTTP**                0x10  the client will try to use http instead of https
 **FLAGS_STATS**               0x20  nodes will keep track of the stats (default=true)
 **FLAGS_NODE_LIST_NO_SIG**    0x40  nodelist update request will not automatically ask for signatures and proof
 **FLAGS_BOOT_WEIGHTS**        0x80  if true the client will initialize the first weights from the nodelist given by the nodelist.
 **FLAGS_ALLOW_EXPERIMENTAL**  0x100 if true the client will support experimental features.
============================== ===== =============================================================================================
```

#### in3_plugin_act_t

plugin action list 

The enum type contains the following values:

```eval_rst
================================ ========== ==========================================================================================================================================================
 **PLGN_ACT_INIT**               0x1        initialize plugin - use for allocating/setting-up internal resources . 
                                            
                                            Plugins will be initialized before first used. The plgn_ctx will be the first request ctx in3_req_t
 **PLGN_ACT_TERM**               0x2        terminate plugin - use for releasing internal resources and cleanup. 
                                            
                                            ( ctx: in3_t )
 **PLGN_ACT_TRANSPORT_SEND**     0x4        sends out a request - the transport plugin will receive a request_t as plgn_ctx, it may set a cptr which will be passed back when fetching more responses.
 **PLGN_ACT_TRANSPORT_RECEIVE**  0x8        fetch next response - the transport plugin will receive a request_t as plgn_ctx, which contains a cptr if set previously
 **PLGN_ACT_TRANSPORT_CLEAN**    0x10       free-up transport resources - the transport plugin will receive a request_t as plgn_ctx if the cptr was set.
 **PLGN_ACT_SIGN_ACCOUNT**       0x20       returns the accounts or addresses of the signer
 **PLGN_ACT_SIGN_PUBLICKEY**     0x40       returns the public key for a given address ( ctx: in3_sign_public_key_ctx_t )
 **PLGN_ACT_SIGN_PREPARE**       0x80       allows a wallet to manipulate the payload before signing - the plgn_ctx will be in3_sign_ctx_t. 
                                            
                                            This way a tx can be send through a multisig
 **PLGN_ACT_SIGN**               0x100      signs the payload - the plgn_ctx will be in3_sign_ctx_t.
 **PLGN_ACT_RPC_HANDLE**         0x200      a plugin may respond to a rpc-request directly (without sending it to the node).
 **PLGN_ACT_RPC_VERIFY**         0x400      verifies the response. 
                                            
                                            the plgn_ctx will be a in3_vctx_t holding all data
 **PLGN_ACT_CACHE_SET**          0x800      stores data to be reused later - the plgn_ctx will be a in3_cache_ctx_t containing the data
 **PLGN_ACT_CACHE_GET**          0x1000     reads data to be previously stored - the plgn_ctx will be a in3_cache_ctx_t containing the key. 
                                            
                                            if the data was found the data-property needs to be set.
 **PLGN_ACT_CACHE_CLEAR**        0x2000     clears all stored data - plgn_ctx will be NULL
 **PLGN_ACT_CONFIG_SET**         0x4000     gets a config-token and reads data from it
 **PLGN_ACT_CONFIG_GET**         0x8000     gets a string-builder and adds all config to it.
 **PLGN_ACT_PAY_PREPARE**        0x10000    prepares a payment
 **PLGN_ACT_PAY_FOLLOWUP**       0x20000    called after a request to update stats.
 **PLGN_ACT_PAY_HANDLE**         0x40000    handles the payment
 **PLGN_ACT_PAY_SIGN_REQ**       0x80000    signs a request
 **PLGN_ACT_LOG_ERROR**          0x100000   report an error
 **PLGN_ACT_NL_PICK**            0x200000   picks the data nodes, plgn_ctx will be a pointer to in3_req_t
 **PLGN_ACT_NL_PICK_FOLLOWUP**   0x400000   called after receiving a response in order to decide whether a update is needed, plgn_ctx will be a pointer to in3_req_t
 **PLGN_ACT_NL_BLACKLIST**       0x800000   blacklist a particular node in the nodelist, plgn_ctx will be a pointer to the node's address.
 **PLGN_ACT_NL_FAILABLE**        0x1000000  handle fail-able request, plgn_ctx will be a pointer to in3_req_t
 **PLGN_ACT_NL_OFFLINE**         0x2000000  mark a particular node in the nodelist as offline, plgn_ctx will be a pointer to in3_nl_offline_ctx_t.
 **PLGN_ACT_CHAIN_CHANGE**       0x4000000  chain id change event, called after setting new chain id
 **PLGN_ACT_GET_DATA**           0x8000000  get access to plugin data as a void ptr
 **PLGN_ACT_ADD_PAYLOAD**        0x10000000 add plugin specific metadata to payload, plgn_ctx will be a sb_t pointer, make sure to begin with a comma
================================ ========== ==========================================================================================================================================================
```

#### chain_id_t

type for a chain_id. 


```c
typedef uint32_t chain_id_t
```

#### in3_node_props_t

Node capabilities. 


```c
typedef uint64_t in3_node_props_t
```

#### in3_verified_hash_t

represents a blockhash which was previously verified 


The stuct contains following fields:

```eval_rst
========================= ================== =======================
``uint64_t``               **block_number**  the number of the block
`bytes32_t <#bytes32-t>`_  **hash**          the blockhash
========================= ================== =======================
```

#### in3_chain_t

Chain definition inside incubed. 

for incubed a chain can be any distributed network or database with incubed support. 


The stuct contains following fields:

```eval_rst
=============================================== ===================== =========================================================================
``uint8_t``                                      **version**          version of the chain
`chain_id_t <#chain-id-t>`_                      **chain_id**         chain_id, which could be a free or based on the public ethereum networkId
`in3_chain_type_t <#in3-chain-type-t>`_          **type**             chain-type
`in3_verified_hash_t * <#in3-verified-hash-t>`_  **verified_hashes**  contains the list of already verified blockhashes
=============================================== ===================== =========================================================================
```

#### in3_plugin_t

plugin interface definition 


The stuct contains following fields:

```eval_rst
========================================= =============== ===================================================
``in3_plugin_supp_acts_t``                 **acts**       bitmask of supported actions this plugin can handle
``void *``                                 **data**       opaque pointer to plugin data
`in3_plugin_act_fn <#in3-plugin-act-fn>`_  **action_fn**  plugin action handler
`in3_plugin_t * <#in3-plugin-t>`_          **next**       pointer to next plugin in list
========================================= =============== ===================================================
```

#### in3_plugin_act_fn

plugin action handler 

Implementations of this function must strictly follow the below pattern for return values -

- IN3_OK - successfully handled specified action
- IN3_WAITING - handling specified action, but waiting for more information
- IN3_EIGNORE - could handle specified action, but chose to ignore it so maybe another handler could handle it
- Other errors - handled but failed


```c
typedef in3_ret_t(* in3_plugin_act_fn) (void *plugin_data, in3_plugin_act_t action, void *plugin_ctx)
```

returns: [`in3_ret_t(*`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_plugin_supp_acts_t


```c
typedef uint32_t in3_plugin_supp_acts_t
```

#### in3_t

Incubed Configuration. 

This struct holds the configuration and also point to internal resources such as filters or chain configs. 


The stuct contains following fields:

```eval_rst
================================= =========================== ================================================================================================================
``uint8_t``                        **signature_count**        the number of signatures used to proof the blockhash.
``uint8_t``                        **replace_latest_block**   if specified, the blocknumber *latest* will be replaced by blockNumber- specified value
``uint_fast16_t``                  **flags**                  a bit mask with flags defining the behavior of the incubed client. 
                                                              
                                                              See the FLAG...-defines
``uint16_t``                       **finality**               the number of signatures in percent required for the request
``uint_fast16_t``                  **max_attempts**           the max number of attempts before giving up
``uint_fast16_t``                  **max_verified_hashes**    max number of verified hashes to cache (actual number may temporarily exceed this value due to pending requests)
``uint_fast16_t``                  **alloc_verified_hashes**  number of currently allocated verified hashes
``uint_fast16_t``                  **pending**                number of pending requests created with this instance
``uint32_t``                       **cache_timeout**          number of seconds requests can be cached.
``uint32_t``                       **timeout**                specifies the number of milliseconds before the request times out. 
                                                              
                                                              increasing may be helpful if the device uses a slow connection.
``uint32_t``                       **id_count**               counter for use as JSON RPC id - incremented for every request
``in3_plugin_supp_acts_t``         **plugin_acts**            bitmask of supported actions of all plugins registered with this client
`in3_proof_t <#in3-proof-t>`_      **proof**                  the type of proof used
`in3_chain_t <#in3-chain-t>`_      **chain**                  chain spec and nodeList definitions
`in3_plugin_t * <#in3-plugin-t>`_  **plugins**                list of registered plugins
================================= =========================== ================================================================================================================
```

#### plgn_register

a register-function for a plugin. 


```c
typedef in3_ret_t(* plgn_register) (in3_t *c)
```

returns: [`in3_ret_t(*`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_for_chain_default

```c
in3_t* in3_for_chain_default(chain_id_t chain_id);
```

arguments:
```eval_rst
=========================== ============== ==========================================
`chain_id_t <#chain-id-t>`_  **chain_id**  the chain_id (see CHAIN_ID_... constants).
=========================== ============== ==========================================
```
returns: [`in3_t *`](#in3-t)


#### in3_client_rpc

```c
NONULL in3_ret_t in3_client_rpc(in3_t *c, const char *method, const char *params, char **result, char **error);
```

sends a request and stores the result in the provided buffer 

arguments:
```eval_rst
=================== ============ ==========================================================================================================================================================
`in3_t * <#in3-t>`_  **c**       [in] the pointer to the incubed client config.
``const char *``     **method**  [in] the name of the rpc-function to call.
``const char *``     **params**  [in] docs for input parameter v.
``char **``          **result**  [in] pointer to string which will be set if the request was successful. This will hold the result as json-rpc-string. (make sure you free this after use!)
``char **``          **error**   [in] pointer to a string containing the error-message. (make sure you free it after use!)
=================== ============ ==========================================================================================================================================================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_client_rpc_raw

```c
NONULL in3_ret_t in3_client_rpc_raw(in3_t *c, const char *request, char **result, char **error);
```

sends a request and stores the result in the provided buffer, this method will always return the first, so bulk-requests are not supported. 

arguments:
```eval_rst
=================== ============= ==========================================================================================================================================================
`in3_t * <#in3-t>`_  **c**        [in] the pointer to the incubed client config.
``const char *``     **request**  [in] the rpc request including method and params.
``char **``          **result**   [in] pointer to string which will be set if the request was successful. This will hold the result as json-rpc-string. (make sure you free this after use!)
``char **``          **error**    [in] pointer to a string containing the error-message. (make sure you free it after use!)
=================== ============= ==========================================================================================================================================================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_client_exec_req

```c
NONULL char* in3_client_exec_req(in3_t *c, char *req);
```

executes a request and returns result as string. 

in case of an error, the error-property of the result will be set. This function also supports sending bulk-requests, but you can not mix internal and external calls, since bulk means all requests will be send to picked nodes. The resulting string must be free by the the caller of this function! 

arguments:
```eval_rst
=================== ========= ==============================================
`in3_t * <#in3-t>`_  **c**    [in] the pointer to the incubed client config.
``char *``           **req**  [in] the request as rpc.
=================== ========= ==============================================
```
returns: `NONULL char *`


#### in3_client_register_chain

```c
NONULL in3_ret_t in3_client_register_chain(in3_t *client, chain_id_t chain_id, in3_chain_type_t type, uint8_t version);
```

registers a new chain or replaces a existing (but keeps the nodelist) 

arguments:
```eval_rst
======================================= ============== ==============================================
`in3_t * <#in3-t>`_                      **client**    [in] the pointer to the incubed client config.
`chain_id_t <#chain-id-t>`_              **chain_id**  [in] the chain id.
`in3_chain_type_t <#in3-chain-type-t>`_  **type**      [in] the verification type of the chain.
``uint8_t``                              **version**   [in] the chain version.
======================================= ============== ==============================================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_free

```c
NONULL void in3_free(in3_t *a);
```

frees the references of the client 

arguments:
```eval_rst
=================== ======= ======================================================
`in3_t * <#in3-t>`_  **a**  [in] the pointer to the incubed client config to free.
=================== ======= ======================================================
```
returns: `NONULL void`


#### in3_configure

```c
NONULL char* in3_configure(in3_t *c, const char *config);
```

configures the client based on a json-config. 

For details about the structure of the config see [https://in3.readthedocs.io/en/develop/api-ts.html#type-in3config](https://in3.readthedocs.io/en/develop/api-ts.html#type-in3config) Returns NULL on success, and error string on failure (to be freed by caller) - in which case the client state is undefined 

arguments:
```eval_rst
=================== ============ ==========================================
`in3_t * <#in3-t>`_  **c**       the incubed client
``const char *``     **config**  JSON-string with the configuration to set.
=================== ============ ==========================================
```
returns: `NONULL char *`


#### in3_get_config

```c
NONULL char* in3_get_config(in3_t *c);
```

gets the current config as json. 

For details about the structure of the config see [https://in3.readthedocs.io/en/develop/api-ts.html#type-in3config](https://in3.readthedocs.io/en/develop/api-ts.html#type-in3config) 

arguments:
```eval_rst
=================== ======= ==================
`in3_t * <#in3-t>`_  **c**  the incubed client
=================== ======= ==================
```
returns: `NONULL char *`


### plugin.h

this file defines the plugin-contexts 

File: [c/src/core/client/plugin.h](https://github.com/slockit/in3-c/blob/master/c/src/core/client/plugin.h)

#### in3_plugin_is_registered (client,action)

checks if a plugin for specified action is registered with the client 

```c
#define in3_plugin_is_registered (client,action) (((client)->plugin_acts & (action)) == (action))
```


#### vc_err (vc,msg)

```c
#define vc_err (vc,msg) vc_set_error(vc, NULL)
```


#### CNF_ERROR (msg)

raises a error during config by setting the error-message and returning a error-code. 

```c
#define CNF_ERROR (msg)   {                                     \
    ctx->error_msg = _strdupn(msg, -1); \
    return IN3_EINVAL;                  \
  }
```


#### CNF_SET_BYTES (dst,token,property,l)

sets the bytes as taken from the given property to the target and raises an error if the len does not fit. 

```c
#define CNF_SET_BYTES (dst,token,property,l)   {                                                                 \
    const bytes_t tmp = d_to_bytes(d_get(token, key(property)));    \
    if (tmp.data) {                                                 \
      if (tmp.len != l) CNF_ERROR(property " must be " #l " bytes") \
      memcpy(dst, tmp.data, l);                                     \
    }                                                               \
  }
```


#### CNF_SET_STRING (dst,token,property)

sets the string as taken from the given property to the target and raises an error if the len does not fit. 

```c
#define CNF_SET_STRING (dst,token,property)   {                                                                                                 \
    const d_token_t* t = d_get(token, key(property));                                               \
    if (d_type(t) != T_NULL && d_type(t) != T_STRING) CNF_ERROR("Invalid config for " property "!") \
    const char* tmp = d_string(t);                                                                  \
    if (tmp) {                                                                                      \
      if (dst) _free(dst);                                                                          \
      dst = _strdupn(tmp, -1);                                                                      \
    }                                                                                               \
  }
```


#### in3_signer_type_t

defines the type of signer used 

The enum type contains the following values:

```eval_rst
==================== = 
 **SIGNER_ECDSA**    1 
 **SIGNER_EIP1271**  2 
==================== = 
```

#### d_signature_type_t

type of the requested signature 

The enum type contains the following values:

```eval_rst
==================== = ===========================================================
 **SIGN_EC_RAW**     0 sign the data directly
 **SIGN_EC_HASH**    1 hash and sign the data
 **SIGN_EC_PREFIX**  2 add Ethereum Signed Message-Proefix, hash and sign the data
 **SIGN_EC_BTC**     3 hashes the data twice with sha256 and signs it
==================== = ===========================================================
```

#### d_payload_type_t

payload type of the requested signature. 

It describes how to deserialize the payload. 

The enum type contains the following values:

```eval_rst
==================== = ======================================================
 **PL_SIGN_ANY**     0 custom data to be signed
 **PL_SIGN_ETHTX**   1 the payload is a ethereum-tx
 **PL_SIGN_BTCTX**   2 the payload is a BTC-Tx-Input
 **PL_SIGN_SAFETX**  3 The payload is a rlp-encoded data of a Gnosys Safe Tx.
==================== = ======================================================
```

#### in3_nl_pick_type_t

The enum type contains the following values:

```eval_rst
=============== = ===================
 **NL_DATA**    0 data provider node.
 **NL_SIGNER**  1 signer node.
=============== = ===================
```

#### in3_get_data_type_t

The enum type contains the following values:

```eval_rst
================================== = 
 **GET_DATA_REGISTRY_ID**          0 
 **GET_DATA_NODE_MIN_BLK_HEIGHT**  1 
 **GET_DATA_CLIENT_DATA**          2 
================================== = 
```

#### in3_req_header_t

optional request headers 


The stuct contains following fields:

```eval_rst
============================================= =========== ======================
``char *``                                     **value**  the value
`in3_req_headerstruct , * <#in3-req-header>`_  **next**   pointer to next header
============================================= =========== ======================
```

#### in3_http_request_t

request-object. 

represents a RPC-request 


The stuct contains following fields:

```eval_rst
========================================= ================= =======================================================
``char *``                                 **method**       the http-method to be used
``char *``                                 **payload**      the payload to send
``char **``                                **urls**         array of urls
``uint_fast16_t``                          **urls_len**     number of urls
``uint32_t``                               **payload_len**  length of the payload in bytes.
`in3_reqstruct , * <#in3-req>`_            **req**          the current context
``void *``                                 **cptr**         a custom ptr to hold information during
``uint32_t``                               **wait**         time in ms to wait before sending out the request
`in3_req_header_t * <#in3-req-header-t>`_  **headers**      optional additional headers to be send with the request
========================================= ================= =======================================================
```

#### in3_transport_legacy


```c
typedef in3_ret_t(* in3_transport_legacy) (in3_http_request_t *request)
```

returns: [`in3_ret_t(*`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_sign_account_ctx_t

action context when retrieving the addresses or accounts of a signer. 


The stuct contains following fields:

```eval_rst
========================================= ================== =================================================
`in3_reqstruct , * <#in3-req>`_            **req**           the context of the request in order report errors
``uint8_t *``                              **accounts**      the account to use for the signature
``int``                                    **accounts_len**  number of accounts
`in3_signer_type_t <#in3-signer-type-t>`_  **signer_type**   the type of the signer used for this account.
========================================= ================== =================================================
```

#### in3_sign_public_key_ctx_t

action context when retrieving the public key of the signer. 


The stuct contains following fields:

```eval_rst
=============================== ================ =================================================
`in3_reqstruct , * <#in3-req>`_  **req**         the context of the request in order report errors
``uint8_t *``                    **account**     the account to use for the signature
``uint8_t``                      **public_key**  the public key in case the plugin returns IN3_OK
=============================== ================ =================================================
```

#### in3_sign_prepare_ctx_t

action context when retrieving the account of a signer. 


The stuct contains following fields:

```eval_rst
=============================== ============= ========================================================================================================================================================================
`in3_reqstruct , * <#in3-req>`_  **req**      the context of the request in order report errors
`address_t <#address-t>`_        **account**  the account to use for the signature
`d_token_t * <#d-token-t>`_      **tx**       the tx-definition, which may contain additional properties
`bytes_t <#bytes-t>`_            **old_tx**   the source-transaction
`bytes_t <#bytes-t>`_            **new_tx**   the new-transaction, if the transaction should not changed, the data-ptr must be NULL otherwise a new allocated memory is expected, which will be cleaned by the caller.
`sb_t * <#sb-t>`_                **output**   if this is not NULL, the transaction will not be send, but returns only the state
=============================== ============= ========================================================================================================================================================================
```

#### in3_sign_ctx_t

signing context. 

This Context is passed to the signer-function. 


The stuct contains following fields:

```eval_rst
=========================================== ================== ================================================================================================
`bytes_t <#bytes-t>`_                        **signature**     the resulting signature
`d_signature_type_t <#d-signature-type-t>`_  **type**          the type of signature
`d_payload_type_t <#d-payload-type-t>`_      **payload_type**  the type of payload in order to deserialize the payload
`in3_reqstruct , * <#in3-req>`_              **req**           the context of the request in order report errors
`bytes_t <#bytes-t>`_                        **message**       the message to sign
`bytes_t <#bytes-t>`_                        **account**       the account to use for the signature
`d_token_t * <#d-token-t>`_                  **meta**          optional metadata to pass a long, which could include data to present to the user before signing
=========================================== ================== ================================================================================================
```

#### in3_configure_ctx_t

context used during configure 


The stuct contains following fields:

```eval_rst
============================= =============== =========================================
`in3_t * <#in3-t>`_            **client**     the client to configure
`json_ctx_t * <#json-ctx-t>`_  **json**       the json ctx corresponding to below token
`d_token_t * <#d-token-t>`_    **token**      the token not handled yet
``char *``                     **error_msg**  message in case of an incorrect config
============================= =============== =========================================
```

#### in3_get_config_ctx_t

context used during get config 


The stuct contains following fields:

```eval_rst
=================== ============ ================================
`in3_t * <#in3-t>`_  **client**  the client to configure
`sb_t * <#sb-t>`_    **sb**      stringbuilder to add json-config
=================== ============ ================================
```

#### in3_storage_get_item

storage handler function for reading from cache. 


```c
typedef bytes_t*(* in3_storage_get_item) (void *cptr, const char *key)
```

returns: [`bytes_t *(*`](#bytes-t) : the found result. if the key is found this function should return the values as bytes otherwise `NULL`. 




#### in3_storage_set_item

storage handler function for writing to the cache. 


```c
typedef void(* in3_storage_set_item) (void *cptr, const char *key, bytes_t *value)
```


#### in3_storage_clear

storage handler function for clearing the cache. 


```c
typedef void(* in3_storage_clear) (void *cptr)
```


#### in3_cache_ctx_t

context used during get config 


The stuct contains following fields:

```eval_rst
=========================== ============= ===================
`in3_req_t * <#in3-req-t>`_  **req**      the request context
``const char *``             **key**      the key to fetch
`bytes_t * <#bytes-t>`_      **content**  the content to set
=========================== ============= ===================
```

#### in3_plugin_register

```c
in3_ret_t in3_plugin_register(in3_t *c, in3_plugin_supp_acts_t acts, in3_plugin_act_fn action_fn, void *data, bool replace_ex);
```

registers a plugin with the client 

arguments:
```eval_rst
========================================= ================ ============================================================================================
`in3_t * <#in3-t>`_                        **c**           the client
``in3_plugin_supp_acts_t``                 **acts**        the actions to register for combined with OR
`in3_plugin_act_fn <#in3-plugin-act-fn>`_  **action_fn**   the plugin action function
``void *``                                 **data**        an optional data or config struct which will be passed to the action function when executed
``bool``                                   **replace_ex**  if this is true and an plugin with the same action is already registered, it will replace it
========================================= ================ ============================================================================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_default

```c
void in3_register_default(plgn_register reg_fn);
```

adds a plugin rregister function to the default. 

All defaults functions will automaticly called and registered for every new in3_t instance. 

arguments:
```eval_rst
================================= ============ 
`plgn_register <#plgn-register>`_  **reg_fn**  
================================= ============ 
```

#### in3_plugin_execute_all

```c
in3_ret_t in3_plugin_execute_all(in3_t *c, in3_plugin_act_t action, void *plugin_ctx);
```

executes all plugins irrespective of their return values, returns first error (if any) 

arguments:
```eval_rst
======================================= ================ 
`in3_t * <#in3-t>`_                      **c**           
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**      
``void *``                               **plugin_ctx**  
======================================= ================ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_plugin_execute_first

```c
in3_ret_t in3_plugin_execute_first(in3_req_t *req, in3_plugin_act_t action, void *plugin_ctx);
```

executes all plugin actions one-by-one, stops when a plugin returns anything other than IN3_EIGNORE. 

returns IN3_EPLGN_NONE if no plugin was able to handle specified action, otherwise returns IN3_OK plugin errors are reported via the in3_req_t 

arguments:
```eval_rst
======================================= ================ 
`in3_req_t * <#in3-req-t>`_              **req**         
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**      
``void *``                               **plugin_ctx**  
======================================= ================ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_plugin_execute_first_or_none

```c
in3_ret_t in3_plugin_execute_first_or_none(in3_req_t *req, in3_plugin_act_t action, void *plugin_ctx);
```

same as in3_plugin_execute_first(), but returns IN3_OK even if no plugin could handle specified action 

arguments:
```eval_rst
======================================= ================ 
`in3_req_t * <#in3-req-t>`_              **req**         
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**      
``void *``                               **plugin_ctx**  
======================================= ================ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_plugin_get_data

```c
static void* in3_plugin_get_data(in3_t *c, in3_plugin_act_fn fn);
```

get direct access to plugin data (if registered) based on action function 

arguments:
```eval_rst
========================================= ======== 
`in3_t * <#in3-t>`_                        **c**   
`in3_plugin_act_fn <#in3-plugin-act-fn>`_  **fn**  
========================================= ======== 
```
returns: `void *`


#### in3_rpc_handle_start

```c
NONULL sb_t* in3_rpc_handle_start(in3_rpc_handle_ctx_t *hctx);
```

creates a response and returns a stringbuilder to add the result-data. 

arguments:
```eval_rst
================================================= ========== 
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **hctx**  
================================================= ========== 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### in3_rpc_handle_finish

```c
NONULL in3_ret_t in3_rpc_handle_finish(in3_rpc_handle_ctx_t *hctx);
```

finish the response. 

arguments:
```eval_rst
================================================= ========== 
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **hctx**  
================================================= ========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_rpc_handle_with_bytes

```c
NONULL in3_ret_t in3_rpc_handle_with_bytes(in3_rpc_handle_ctx_t *hctx, bytes_t data);
```

creates a response with bytes. 

arguments:
```eval_rst
================================================= ========== 
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **hctx**  
`bytes_t <#bytes-t>`_                              **data**  
================================================= ========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_rpc_handle_with_json

```c
NONULL in3_ret_t in3_rpc_handle_with_json(in3_rpc_handle_ctx_t *ctx, d_token_t *result);
```

creates a response with a json token. 

arguments:
```eval_rst
================================================= ============ 
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **ctx**     
`d_token_t * <#d-token-t>`_                        **result**  
================================================= ============ 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_rpc_handle_with_string

```c
NONULL in3_ret_t in3_rpc_handle_with_string(in3_rpc_handle_ctx_t *hctx, char *data);
```

creates a response with string. 

arguments:
```eval_rst
================================================= ========== 
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **hctx**  
``char *``                                         **data**  
================================================= ========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_rpc_handle_with_int

```c
NONULL in3_ret_t in3_rpc_handle_with_int(in3_rpc_handle_ctx_t *hctx, uint64_t value);
```

creates a response with a value which is added as hex-string. 

arguments:
```eval_rst
================================================= =========== 
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **hctx**   
``uint64_t``                                       **value**  
================================================= =========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_rpc_handle_with_uint256

```c
NONULL in3_ret_t in3_rpc_handle_with_uint256(in3_rpc_handle_ctx_t *hctx, bytes_t data);
```

creates a response with bytes but without a leading 0. 

arguments:
```eval_rst
================================================= ========== 
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **hctx**  
`bytes_t <#bytes-t>`_                              **data**  
================================================= ========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_get_request_payload

```c
char* in3_get_request_payload(in3_http_request_t *request);
```

getter to retrieve the payload from a in3_http_request_t struct 

arguments:
```eval_rst
============================================= ============= ==============
`in3_http_request_t * <#in3-http-request-t>`_  **request**  request struct
============================================= ============= ==============
```
returns: `char *`


#### in3_get_request_payload_len

```c
uint32_t in3_get_request_payload_len(in3_http_request_t *request);
```

getter to retrieve the length of the payload from a in3_http_request_t struct 

arguments:
```eval_rst
============================================= ============= ==============
`in3_http_request_t * <#in3-http-request-t>`_  **request**  request struct
============================================= ============= ==============
```
returns: `uint32_t`


#### in3_get_request_headers_len

```c
int in3_get_request_headers_len(in3_http_request_t *request);
```

getter to retrieve the urls list length from a in3_http_request_t struct 

arguments:
```eval_rst
============================================= ============= ==============
`in3_http_request_t * <#in3-http-request-t>`_  **request**  request struct
============================================= ============= ==============
```
returns: `int`


#### in3_get_request_headers_at

```c
char* in3_get_request_headers_at(in3_http_request_t *request, int index);
```

getter to retrieve the urls list length from a in3_http_request_t struct 

arguments:
```eval_rst
============================================= ============= =======================
`in3_http_request_t * <#in3-http-request-t>`_  **request**  request struct
``int``                                        **index**    the inde xof the header
============================================= ============= =======================
```
returns: `char *`


#### in3_get_request_method

```c
char* in3_get_request_method(in3_http_request_t *request);
```

getter to retrieve the http-method from a in3_http_request_t struct 

arguments:
```eval_rst
============================================= ============= ==============
`in3_http_request_t * <#in3-http-request-t>`_  **request**  request struct
============================================= ============= ==============
```
returns: `char *`


#### in3_get_request_urls

```c
char** in3_get_request_urls(in3_http_request_t *request);
```

getter to retrieve the urls list from a in3_http_request_t struct 

arguments:
```eval_rst
============================================= ============= ==============
`in3_http_request_t * <#in3-http-request-t>`_  **request**  request struct
============================================= ============= ==============
```
returns: `char **`


#### in3_get_request_urls_len

```c
int in3_get_request_urls_len(in3_http_request_t *request);
```

getter to retrieve the urls list length from a in3_http_request_t struct 

arguments:
```eval_rst
============================================= ============= ==============
`in3_http_request_t * <#in3-http-request-t>`_  **request**  request struct
============================================= ============= ==============
```
returns: `int`


#### in3_get_request_timeout

```c
uint32_t in3_get_request_timeout(in3_http_request_t *request);
```

getter to retrieve the urls list length from a in3_http_request_t struct 

arguments:
```eval_rst
============================================= ============= ==============
`in3_http_request_t * <#in3-http-request-t>`_  **request**  request struct
============================================= ============= ==============
```
returns: `uint32_t`


#### in3_req_add_response

```c
NONULL void in3_req_add_response(in3_http_request_t *req, int index, int error, const char *data, int data_len, uint32_t time);
```

adds a response for a request-object. 

This function should be used in the transport-function to set the response. 

arguments:
```eval_rst
============================================= ============== ================================================================================================
`in3_http_request_t * <#in3-http-request-t>`_  **req**       [in]the the request
``int``                                        **index**     [in] the index of the url, since this request could go out to many urls
``int``                                        **error**     [in] if <0 this will be reported as error. the message should then be the error-message
``const char *``                               **data**      the data or the the string
``int``                                        **data_len**  the length of the data or the the string (use -1 if data is a null terminated string)
``uint32_t``                                   **time**      the time this request took in ms or 0 if not possible (it will be used to calculate the weights)
============================================= ============== ================================================================================================
```
returns: `NONULL void`


#### in3_ctx_add_response

```c
NONULL void in3_ctx_add_response(in3_req_t *req, int index, int error, const char *data, int data_len, uint32_t time);
```

adds a response to a context. 

This function should be used in the transport-function to set the response. 

arguments:
```eval_rst
=========================== ============== ================================================================================================
`in3_req_t * <#in3-req-t>`_  **req**       [in]the current context
``int``                      **index**     [in] the index of the url, since this request could go out to many urls
``int``                      **error**     [in] if <0 this will be reported as error. the message should then be the error-message
``const char *``             **data**      the data or the the string
``int``                      **data_len**  the length of the data or the the string (use -1 if data is a null terminated string)
``uint32_t``                 **time**      the time this request took in ms or 0 if not possible (it will be used to calculate the weights)
=========================== ============== ================================================================================================
```
returns: `NONULL void`


#### in3_set_default_legacy_transport

```c
void in3_set_default_legacy_transport(in3_transport_legacy transport);
```

defines a default transport which is used when creating a new client. 

arguments:
```eval_rst
======================== =============== ===============================
``in3_transport_legacy``  **transport**  the default transport-function.
======================== =============== ===============================
```

#### in3_sign_ctx_get_message

```c
bytes_t in3_sign_ctx_get_message(in3_sign_ctx_t *ctx);
```

helper function to retrieve and message from a in3_sign_ctx_t 

helper function to retrieve and message from a in3_sign_ctx_t 

arguments:
```eval_rst
===================================== ========= ==================
`in3_sign_ctx_t * <#in3-sign-ctx-t>`_  **ctx**  the signer context
===================================== ========= ==================
```
returns: [`bytes_t`](#bytes-t)


#### in3_sign_ctx_get_account

```c
bytes_t in3_sign_ctx_get_account(in3_sign_ctx_t *ctx);
```

helper function to retrieve and account from a in3_sign_ctx_t 

helper function to retrieve and account from a in3_sign_ctx_t 

arguments:
```eval_rst
===================================== ========= ==================
`in3_sign_ctx_t * <#in3-sign-ctx-t>`_  **ctx**  the signer context
===================================== ========= ==================
```
returns: [`bytes_t`](#bytes-t)


#### in3_sign_ctx_set_signature_hex

```c
void in3_sign_ctx_set_signature_hex(in3_sign_ctx_t *ctx, const char *signature);
```

helper function to retrieve the signature from a in3_sign_ctx_t 

arguments:
```eval_rst
===================================== =============== ====================
`in3_sign_ctx_t * <#in3-sign-ctx-t>`_  **ctx**        the signer context
``const char *``                       **signature**  the signature in hex
===================================== =============== ====================
```

#### create_sign_ctx

```c
NONULL in3_sign_ctx_t* create_sign_ctx(in3_req_t *req);
```

creates a signer ctx to be used for async signing. 

arguments:
```eval_rst
=========================== ========= ====================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the rpc context
=========================== ========= ====================
```
returns: [`in3_sign_ctx_tNONULL , *`](#in3-sign-ctx-t)


#### in3_set_storage_handler

```c
void in3_set_storage_handler(in3_t *c, in3_storage_get_item get_item, in3_storage_set_item set_item, in3_storage_clear clear, void *cptr);
```

create a new storage handler-object to be set on the client. 

the caller will need to free this pointer after usage. 

arguments:
```eval_rst
=============================================== ============== ============================================================
`in3_t * <#in3-t>`_                              **c**         the incubed client
`in3_storage_get_item <#in3-storage-get-item>`_  **get_item**  function pointer returning a stored value for the given key.
`in3_storage_set_item <#in3-storage-set-item>`_  **set_item**  function pointer setting a stored value for the given key.
`in3_storage_clear <#in3-storage-clear>`_        **clear**     function pointer clearing all contents of cache.
``void *``                                       **cptr**      custom pointer which will will be passed to functions
=============================================== ============== ============================================================
```

#### vc_set_error

```c
in3_ret_t vc_set_error(in3_vctx_t *vc, char *msg);
```

arguments:
```eval_rst
============================= ========= =========================
`in3_vctx_t * <#in3-vctx-t>`_  **vc**   the verification context.
``char *``                     **msg**  the error message.
============================= ========= =========================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### request.h

Request Context. 

This is used for each request holding request and response-pointers but also controls the execution process. 

File: [c/src/core/client/request.h](https://github.com/slockit/in3-c/blob/master/c/src/core/client/request.h)

#### TRY_SUB_REQUEST (req,name,res,fmt,...)

```c
#define TRY_SUB_REQUEST (req,name,res,fmt,...)   {                                                                          \
    sb_t sb = {0};                                                           \
    sb_printx(&sb, fmt, __VA_ARGS__);                                        \
    in3_ret_t r = req_send_sub_request(req, name, sb.data, NULL, res, NULL); \
    _free(sb.data);                                                          \
    if (r) return r;                                                         \
  }
```


#### TRY_CATCH_SUB_REQUEST (req,name,res,_catch,fmt,...)

```c
#define TRY_CATCH_SUB_REQUEST (req,name,res,_catch,fmt,...)   {                                                                          \
    sb_t sb = {0};                                                           \
    sb_printx(&sb, fmt, __VA_ARGS__);                                        \
    in3_ret_t r = req_send_sub_request(req, name, sb.data, NULL, res, NULL); \
    _free(sb.data);                                                          \
    if (r) {                                                                 \
      _catch;                                                                \
      return r;                                                              \
    }                                                                        \
  }
```


#### TRY_FINAL_SUB_REQUEST (req,name,res,_catch,fmt,...)

```c
#define TRY_FINAL_SUB_REQUEST (req,name,res,_catch,fmt,...)   {                                                                          \
    sb_t sb = {0};                                                           \
    sb_printx(&sb, fmt, __VA_ARGS__);                                        \
    in3_ret_t r = req_send_sub_request(req, name, sb.data, NULL, res, NULL); \
    _free(sb.data);                                                          \
    _catch;                                                                  \
    if (r)                                                                   \
      return r;                                                              \
  }
```


#### ctx_type

type of the request context, 

The enum type contains the following values:

```eval_rst
============= = ============================================================
 **RT_RPC**   0 a json-rpc request, which needs to be send to a incubed node
 **RT_SIGN**  1 a sign request
============= = ============================================================
```

#### state

The current state of the context. 

you can check this state after each execute-call. 

The enum type contains the following values:

```eval_rst
============================== == ==========================================================
 **REQ_SUCCESS**               0  The ctx has a verified result.
 **REQ_WAITING_TO_SEND**       1  the request has not been sent yet
 **REQ_WAITING_FOR_RESPONSE**  2  the request is sent but not all of the response are set ()
 **REQ_ERROR**                 -1 the request has a error
============================== == ==========================================================
```

#### req_type_t

type of the request context, 


The enum type contains the following values:

```eval_rst
============= = ============================================================
 **RT_RPC**   0 a json-rpc request, which needs to be send to a incubed node
 **RT_SIGN**  1 a sign request
============= = ============================================================
```

#### node_match_t

the weight of a certain node as linked list. 

This will be used when picking the nodes to send the request to. A linked list of these structs desribe the result. 


The stuct contains following fields:

```eval_rst
============================= ============= ===========================================================
``unsigned int``               **index**    index of the node in the nodelist
``uint32_t``                   **s**        The starting value.
``uint32_t``                   **w**        weight value
``char *``                     **url**      the url of the node
`address_t <#address-t>`_      **address**  address of the server
`weightstruct , * <#weight>`_  **next**     next in the linked-list or NULL if this is the last element
============================= ============= ===========================================================
```

#### in3_response_t

response-object. 

if the error has a length>0 the response will be rejected 


The stuct contains following fields:

```eval_rst
========================= =========== =================================================================
``uint32_t``               **time**   measured time (in ms) which will be used for ajusting the weights
`in3_ret_t <#in3-ret-t>`_  **state**  the state of the response
`sb_t <#sb-t>`_            **data**   a stringbuilder to add the result
========================= =========== =================================================================
```

#### in3_req_t

The Request config. 

This is generated for each request and represents the current state. it holds the state until the request is finished and must be freed afterwards. 


The stuct contains following fields:

```eval_rst
===================================== ======================== =========================================================================================================
``uint_fast8_t``                       **signers_length**      number or addresses
``uint_fast16_t``                      **len**                 the number of requests
``uint_fast16_t``                      **attempt**             the number of attempts
``uint32_t``                           **id**                  JSON RPC id of request at index 0.
`req_type_t <#req-type-t>`_            **type**                the type of the request
`in3_ret_t <#in3-ret-t>`_              **verification_state**  state of the verification
``char *``                             **error**               in case of an error this will hold the message, if not it points to `NULL`
`json_ctx_t * <#json-ctx-t>`_          **request_context**     the result of the json-parser for the request.
`json_ctx_t * <#json-ctx-t>`_          **response_context**    the result of the json-parser for the response.
`d_token_t ** <#d-token-t>`_           **requests**            references to the tokens representring the requests
`d_token_t ** <#d-token-t>`_           **responses**           references to the tokens representring the parsed responses
`in3_response_t * <#in3-response-t>`_  **raw_response**        the raw response-data, which should be verified.
``uint8_t *``                          **signers**             the addresses of servers requested to sign the blockhash
`node_match_t * <#node-match-t>`_      **nodes**               selected nodes to process the request, which are stored as linked list.
`cache_entry_t * <#cache-entry-t>`_    **cache**               optional cache-entries. 
                                                               
                                                               These entries will be freed when cleaning up the context.
`in3_reqstruct , * <#in3-req>`_        **required**            pointer to the next required context. 
                                                               
                                                               if not NULL the data from this context need get finished first, before being able to resume this context.
`in3_t * <#in3-t>`_                    **client**              reference to the client
===================================== ======================== =========================================================================================================
```

#### in3_req_state_t

The current state of the context. 

you can check this state after each execute-call. 


The enum type contains the following values:

```eval_rst
============================== == ==========================================================
 **REQ_SUCCESS**               0  The ctx has a verified result.
 **REQ_WAITING_TO_SEND**       1  the request has not been sent yet
 **REQ_WAITING_FOR_RESPONSE**  2  the request is sent but not all of the response are set ()
 **REQ_ERROR**                 -1 the request has a error
============================== == ==========================================================
```

#### req_new

```c
NONULL in3_req_t* req_new(in3_t *client, const char *req_data);
```

creates a new request. 

the request data will be parsed and represented in the context. calling this function will only parse the request data, but not send anything yet.

*Important*: the req_data will not be cloned but used during the execution. The caller of the this function is also responsible for freeing this string afterwards. 

arguments:
```eval_rst
=================== ============== ====================================
`in3_t * <#in3-t>`_  **client**    [in] the client-config.
``const char *``     **req_data**  [in] the rpc-request as json string.
=================== ============== ====================================
```
returns: [`in3_req_tNONULL , *`](#in3-req-t)


#### req_new_clone

```c
NONULL in3_req_t* req_new_clone(in3_t *client, const char *req_data);
```

creates a new request but clones the request-data. 

the request data will be parsed and represented in the context. calling this function will only parse the request data, but not send anything yet. 

arguments:
```eval_rst
=================== ============== ====================================
`in3_t * <#in3-t>`_  **client**    [in] the client-config.
``const char *``     **req_data**  [in] the rpc-request as json string.
=================== ============== ====================================
```
returns: [`in3_req_tNONULL , *`](#in3-req-t)


#### in3_send_req

```c
NONULL in3_ret_t in3_send_req(in3_req_t *req);
```

sends a previously created request to nodes and verifies it. 

The execution happens within the same thread, thich mean it will be blocked until the response ha beedn received and verified. In order to handle calls asynchronously, you need to call the `in3_req_execute` function and provide the data as needed. 

arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_req_last_waiting

```c
NONULL in3_req_t* in3_req_last_waiting(in3_req_t *req);
```

finds the last waiting request-context. 

arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: [`in3_req_tNONULL , *`](#in3-req-t)


#### in3_req_exec_state

```c
NONULL in3_req_state_t in3_req_exec_state(in3_req_t *req);
```

executes the request and returns its state. 

arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: [`in3_req_state_tNONULL `](#in3-req-state-t)


#### in3_req_execute

```c
NONULL in3_ret_t in3_req_execute(in3_req_t *req);
```

execute the context, but stops whenever data are required. 

This function should be used in order to call data in a asyncronous way, since this function will not use the transport-function to actually send it.

The caller is responsible for delivering the required responses. After calling you need to check the return-value:

- IN3_WAITING : provide the required data and then call in3_req_execute again.
- IN3_OK : success, we have a result.
- any other status = error

```eval_rst
.. graphviz::

       digraph G {
     node[fontname="Helvetica",   shape=Box, color=lightblue, style=filled ]
      edge[fontname="Helvetica",   style=solid,  fontsize=8 , color=grey]
      rankdir = LR;
    
      RPC[label="RPC-Request"]
      CTX[label="in3_req_t"]
    
      sign[label="sign data",color=lightgrey, style=""]
      request[label="fetch data",color=lightgrey, style=""]
    
      exec[ label="in3_req_execute()",color=lightgrey, style="", shape=circle ]
      free[label="req_free()",color=lightgrey, style=""]
    
    
      RPC -> CTX [label="req_new()"]
      CTX -> exec
    
    
      exec -> error [label="IN3_..."]
      exec -> response[label="IN3_OK"]
      exec -> waiting[label="IN3_WAITING"]
    
      waiting -> sign[label=RT_SIGN]
      waiting -> request[label=RT_RPC]
    
      sign -> exec [label="in3_ctx_add_response()"]
      request -> exec[label="in3_ctx_add_response()"]
    
      response -> free
      error->free
    
    
     { rank = same; exec, sign, request }
    
    
    
    }
    
```
Here is a example how to use this function:

```c
 in3_ret_t in3_send_req(in3_req_t* req) {
  in3_ret_t ret;
  // execute the context and store the return value.
  // if the return value is 0 == IN3_OK, it was successful and we return,
  // if not, we keep on executing
  while ((ret = in3_req_execute(ctx))) {
    // error we stop here, because this means we got an error
    if (ret != IN3_WAITING) return ret;

    // handle subcontexts first, if they have not been finished
    while (ctx->required && in3_req_state(ctx->required) != REQ_SUCCESS) {
      // exxecute them, and return the status if still waiting or error
      if ((ret = in3_send_req(ctx->required))) return ret;

      // recheck in order to prepare the request.
      // if it is not waiting, then it we cannot do much, becaus it will an error or successfull.
      if ((ret = in3_req_execute(ctx)) != IN3_WAITING) return ret;
    }

    // only if there is no response yet...
    if (!ctx->raw_response) {

      // what kind of request do we need to provide?
      switch (ctx->type) {

        // RPC-request to send to the nodes
        case RT_RPC: {

            // build the request
            in3_http_request_t* request = in3_create_request(ctx);

            // here we use the transport, but you can also try to fetch the data in any other way.
            ctx->client->transport(request);

            // clean up
            request_free(request);
            break;
        }

        // this is a request to sign a transaction
        case RT_SIGN: {
            // read the data to sign from the request
            d_token_t* params = d_get(ctx->requests[0], K_PARAMS);
            // the data to sign
            bytes_t    data   = d_to_bytes(d_get_at(params, 0));
            // the account to sign with
            bytes_t    from   = d_to_bytes(d_get_at(params, 1));

            // prepare the response
            ctx->raw_response = _malloc(sizeof(in3_response_t));
            sb_init(&ctx->raw_response[0].error);
            sb_init(&ctx->raw_response[0].result);

            // data for the signature
            uint8_t sig[65];
            // use the signer to create the signature
            ret = ctx->client->signer->sign(ctx, SIGN_EC_HASH, data, from, sig);
            // if it fails we report this as error
            if (ret < 0) return req_set_error(ctx, ctx->raw_response->error.data, ret);
            // otherwise we simply add the raw 65 bytes to the response.
            sb_add_range(&ctx->raw_response->result, (char*) sig, 0, 65);
        }
      }
    }
  }
  // done...
  return ret;
}
```
arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_req_state

```c
NONULL in3_req_state_t in3_req_state(in3_req_t *req);
```

returns the current state of the context. 

arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: [`in3_req_state_tNONULL `](#in3-req-state-t)


#### req_get_error_data

```c
char* req_get_error_data(in3_req_t *req);
```

returns the error of the context. 

arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: `char *`


#### req_get_response_data

```c
char* req_get_response_data(in3_req_t *req);
```

returns json response for that context 

arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: `char *`


#### req_get_result_json

```c
char* req_get_result_json(in3_req_t *ctx, int index);
```

returns the result or NULL in case of an error for that context. 

The result must be freed! 

arguments:
```eval_rst
=========================== =========== 
`in3_req_t * <#in3-req-t>`_  **ctx**    
``int``                      **index**  
=========================== =========== 
```
returns: `char *`


#### req_get_type

```c
NONULL req_type_t req_get_type(in3_req_t *req);
```

returns the type of the request 

arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: [`req_type_tNONULL `](#req-type-t)


#### req_free

```c
NONULL void req_free(in3_req_t *req);
```

frees all resources allocated during the request. 

But this will not free the request string passed when creating the context! 

arguments:
```eval_rst
=========================== ========= =========================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the request context.
=========================== ========= =========================
```
returns: `NONULL void`


#### req_add_required

```c
NONULL in3_ret_t req_add_required(in3_req_t *parent, in3_req_t *req);
```

adds a new context as a requirment. 

Whenever a verifier needs more data and wants to send a request, we should create the request and add it as dependency and stop.

If the function is called again, we need to search and see if the required status is now useable.

Here is an example of how to use it:

```c
in3_ret_t get_from_nodes(in3_req_t* parent, char* method, char* params, bytes_t* dst) {
  // check if the method is already existing
  in3_req_t* req = req_find_required(parent, method, NULL);
  if (ctx) {
    // found one - so we check if it is useable.
    switch (in3_req_state(ctx)) {
      // in case of an error, we report it back to the parent context
      case REQ_ERROR:
        return req_set_error(parent, ctx->error, IN3_EUNKNOWN);
      // if we are still waiting, we stop here and report it.
      case CTX_WAITING_FOR_REQUIRED_CTX:
      case REQ_WAITING_FOR_RESPONSE:
        return IN3_WAITING;

      // if it is useable, we can now handle the result.
      case REQ_SUCCESS: {
        d_token_t* r = d_get(ctx->responses[0], K_RESULT);
        if (r) {
          // we have a result, so write it back to the dst
          *dst = d_to_bytes(r);
          return IN3_OK;
        } else
          // or check the error and report it
          return req_check_response_error(parent, 0);
      }
    }
  }

  // no required context found yet, so we create one:

  // since this is a subrequest it will be freed when the parent is freed.
  // allocate memory for the request-string
  char* req = _malloc(strlen(method) + strlen(params) + 200);
  // create it
  sprintf(req, "{\"method\":\"%s\",\"jsonrpc\":\"2.0\",\"id\":1,\"params\":%s}", method, params);
  // and add the request context to the parent.
  return req_add_required(parent, req_new(parent->client, req));
}
```
arguments:
```eval_rst
=========================== ============ ====================================
`in3_req_t * <#in3-req-t>`_  **parent**  [in] the current request context.
`in3_req_t * <#in3-req-t>`_  **req**     [in] the new request context to add.
=========================== ============ ====================================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### req_find_required

```c
in3_req_t* req_find_required(const in3_req_t *parent, const char *method, const char *param_query);
```

searches within the required request contextes for one with the given method. 

This method is used internaly to find a previously added context. 

arguments:
```eval_rst
================================== ================= ==========================================
`in3_req_tconst , * <#in3-req-t>`_  **parent**       [in] the current request context.
``const char *``                    **method**       [in] the method of the rpc-request.
``const char *``                    **param_query**  [in] a optional string within thew params.
================================== ================= ==========================================
```
returns: [`in3_req_t *`](#in3-req-t)


#### req_remove_required

```c
NONULL in3_ret_t req_remove_required(in3_req_t *parent, in3_req_t *req, bool rec);
```

removes a required context after usage. 

removing will also call free_ctx to free resources. 

arguments:
```eval_rst
=========================== ============ ==================================================
`in3_req_t * <#in3-req-t>`_  **parent**  [in] the current request context.
`in3_req_t * <#in3-req-t>`_  **req**     [in] the request context to remove.
``bool``                     **rec**     [in] if true all sub contexts will aösp be removed
=========================== ============ ==================================================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### req_check_response_error

```c
NONULL in3_ret_t req_check_response_error(in3_req_t *c, int i);
```

check if the response contains a error-property and reports this as error in the context. 

arguments:
```eval_rst
=========================== ======= =================================================================================
`in3_req_t * <#in3-req-t>`_  **c**  [in] the current request context.
``int``                      **i**  [in] the index of the request to check (if this is a batch-request, otherwise 0).
=========================== ======= =================================================================================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### req_get_error

```c
NONULL in3_ret_t req_get_error(in3_req_t *req, int id);
```

determins the errorcode for the given request. 

arguments:
```eval_rst
=========================== ========= =================================================================================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the current request context.
``int``                      **id**   [in] the index of the request to check (if this is a batch-request, otherwise 0).
=========================== ========= =================================================================================
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_client_rpc_ctx_raw

```c
NONULL in3_req_t* in3_client_rpc_ctx_raw(in3_t *c, const char *request);
```

sends a request and returns a context used to access the result or errors. 

This context *MUST* be freed with req_free(ctx) after usage to release the resources. 

arguments:
```eval_rst
=================== ============= =======================
`in3_t * <#in3-t>`_  **c**        [in] the client config.
``const char *``     **request**  [in] rpc request.
=================== ============= =======================
```
returns: [`in3_req_tNONULL , *`](#in3-req-t)


#### in3_client_rpc_ctx

```c
NONULL in3_req_t* in3_client_rpc_ctx(in3_t *c, const char *method, const char *params);
```

sends a request and returns a context used to access the result or errors. 

This context *MUST* be freed with req_free(ctx) after usage to release the resources. 

arguments:
```eval_rst
=================== ============ ========================
`in3_t * <#in3-t>`_  **c**       [in] the clientt config.
``const char *``     **method**  [in] rpc method.
``const char *``     **params**  [in] params as string.
=================== ============ ========================
```
returns: [`in3_req_tNONULL , *`](#in3-req-t)


#### in3_req_get_proof

```c
NONULL in3_proof_t in3_req_get_proof(in3_req_t *req, int i);
```

determines the proof as set in the request. 

arguments:
```eval_rst
=========================== ========= ==================================
`in3_req_t * <#in3-req-t>`_  **req**  [in] the current request.
``int``                      **i**    [in] the index within the request.
=========================== ========= ==================================
```
returns: [`in3_proof_tNONULL `](#in3-proof-t)


### bytes.h

util helper on byte arrays. 

File: [c/src/core/util/bytes.h](https://github.com/slockit/in3-c/blob/master/c/src/core/util/bytes.h)

#### bb_new ()

creates a new bytes_builder with a initial size of 32 bytes 

```c
#define bb_new () bb_newl(32)
```


#### bb_read (_bb_,_i_,_vptr_)

```c
#define bb_read (_bb_,_i_,_vptr_) bb_readl((_bb_), (_i_), (_vptr_), sizeof(*_vptr_))
```


#### bb_read_next (_bb_,_iptr_,_vptr_)

```c
#define bb_read_next (_bb_,_iptr_,_vptr_)   do {                                          \
    size_t _l_ = sizeof(*_vptr_);               \
    bb_readl((_bb_), *(_iptr_), (_vptr_), _l_); \
    *(_iptr_) += _l_;                           \
  } while (0)
```


#### bb_readl (_bb_,_i_,_vptr_,_l_)

```c
#define bb_readl (_bb_,_i_,_vptr_,_l_) memcpy((_vptr_), (_bb_)->b.data + (_i_), _l_)
```


#### b_read (_b_,_i_,_vptr_)

```c
#define b_read (_b_,_i_,_vptr_) b_readl((_b_), (_i_), _vptr_, sizeof(*_vptr_))
```


#### b_readl (_b_,_i_,_vptr_,_l_)

```c
#define b_readl (_b_,_i_,_vptr_,_l_) memcpy(_vptr_, (_b_)->data + (_i_), (_l_))
```


#### NULL_BYTES

empty bytes as struct 

```c
#define NULL_BYTES ((bytes_t){0})
```


#### b_to_stack (d)

converts bytes from heap to stack 

```c
#define b_to_stack (d)   {                                \
    bytes_t o = d;                 \
    d.data    = alloca(d.len);     \
    memcpy(d.data, o.data, o.len); \
    _free(o.data);                 \
  }
```


#### address_t

pointer to a 20byte address 


```c
typedef uint8_t address_t[20]
```

#### bytes32_t

pointer to a 32byte word 


```c
typedef uint8_t bytes32_t[32]
```

#### wlen_t

number of bytes within a word (min 1byte but usually a uint) 


```c
typedef uint_fast8_t wlen_t
```

#### bytes_t

a byte array 


The stuct contains following fields:

```eval_rst
============= ========== =================================
``uint8_t *``  **data**  the byte-data
``uint32_t``   **len**   the length of the array ion bytes
============= ========== =================================
```

#### b_new

```c
RETURNS_NONULL bytes_t* b_new(const uint8_t *data, uint32_t len);
```

allocates a new byte array with 0 filled 

arguments:
```eval_rst
=================== ========== 
``const uint8_t *``  **data**  
``uint32_t``         **len**   
=================== ========== 
```
returns: [`bytes_tRETURNS_NONULL , *`](#bytes-t)


#### b_get_data

```c
NONULL uint8_t* b_get_data(const bytes_t *b);
```

gets the data field from an input byte array 

arguments:
```eval_rst
============================== ======= 
`bytes_tconst , * <#bytes-t>`_  **b**  
============================== ======= 
```
returns: `NONULL uint8_t *`


#### b_get_len

```c
NONULL uint32_t b_get_len(const bytes_t *b);
```

gets the len field from an input byte array 

arguments:
```eval_rst
============================== ======= 
`bytes_tconst , * <#bytes-t>`_  **b**  
============================== ======= 
```
returns: `NONULL uint32_t`


#### b_print

```c
NONULL void b_print(const bytes_t *a);
```

prints a the bytes as hex to stdout 

arguments:
```eval_rst
============================== ======= 
`bytes_tconst , * <#bytes-t>`_  **a**  
============================== ======= 
```
returns: `NONULL void`


#### ba_print

```c
NONULL void ba_print(const uint8_t *a, size_t l);
```

prints a the bytes as hex to stdout 

arguments:
```eval_rst
=================== ======= 
``const uint8_t *``  **a**  
``size_t``           **l**  
=================== ======= 
```
returns: `NONULL void`


#### b_cmp

```c
NONULL int b_cmp(const bytes_t *a, const bytes_t *b);
```

compares 2 byte arrays and returns 1 for equal and 0 for not equal 

arguments:
```eval_rst
============================== ======= 
`bytes_tconst , * <#bytes-t>`_  **a**  
`bytes_tconst , * <#bytes-t>`_  **b**  
============================== ======= 
```
returns: `NONULL int`


#### bytes_cmp

```c
int bytes_cmp(const bytes_t a, const bytes_t b);
```

compares 2 byte arrays and returns 1 for equal and 0 for not equal 

arguments:
```eval_rst
=========================== ======= 
`bytes_tconst  <#bytes-t>`_  **a**  
`bytes_tconst  <#bytes-t>`_  **b**  
=========================== ======= 
```
returns: `int`


#### b_free

```c
void b_free(bytes_t *a);
```

frees the data 

arguments:
```eval_rst
======================= ======= 
`bytes_t * <#bytes-t>`_  **a**  
======================= ======= 
```

#### b_concat

```c
bytes_t b_concat(int cnt,...);
```

duplicates the content of bytes 

arguments:
```eval_rst
======= ========= 
``int``  **cnt**  
``...``           
======= ========= 
```
returns: [`bytes_t`](#bytes-t)


#### b_dup

```c
NONULL bytes_t* b_dup(const bytes_t *a);
```

clones a byte array 

arguments:
```eval_rst
============================== ======= 
`bytes_tconst , * <#bytes-t>`_  **a**  
============================== ======= 
```
returns: [`bytes_tNONULL , *`](#bytes-t)


#### bytes_dup

```c
NONULL bytes_t bytes_dup(const bytes_t a);
```

clones a byte array 

arguments:
```eval_rst
=========================== ======= 
`bytes_tconst  <#bytes-t>`_  **a**  
=========================== ======= 
```
returns: [`bytes_tNONULL `](#bytes-t)


#### b_read_byte

```c
NONULL uint8_t b_read_byte(bytes_t *b, size_t *pos);
```

reads a byte on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
======================= ========= 
```
returns: `NONULL uint8_t`


#### b_read_int

```c
NONULL uint32_t b_read_int(bytes_t *b, size_t *pos);
```

reads a integer on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
======================= ========= 
```
returns: `NONULL uint32_t`


#### b_read_long

```c
NONULL uint64_t b_read_long(bytes_t *b, size_t *pos);
```

reads a long on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
======================= ========= 
```
returns: `NONULL uint64_t`


#### b_new_chars

```c
NONULL char* b_new_chars(bytes_t *b, size_t *pos);
```

creates a new string (needs to be freed) on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
======================= ========= 
```
returns: `NONULL char *`


#### b_new_fixed_bytes

```c
NONULL bytes_t* b_new_fixed_bytes(bytes_t *b, size_t *pos, int len);
```

reads bytes with a fixed length on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
``int``                  **len**  
======================= ========= 
```
returns: [`bytes_tNONULL , *`](#bytes-t)


#### bb_newl

```c
bytes_builder_t* bb_newl(size_t l);
```

creates a new bytes_builder 

arguments:
```eval_rst
========== ======= 
``size_t``  **l**  
========== ======= 
```
returns: [`bytes_builder_t *`](#bytes-builder-t)


#### bb_free

```c
NONULL void bb_free(bytes_builder_t *bb);
```

frees a bytebuilder and its content. 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```
returns: `NONULL void`


#### bb_check_size

```c
NONULL int bb_check_size(bytes_builder_t *bb, size_t len);
```

internal helper to increase the buffer if needed 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``size_t``                               **len**  
======================================= ========= 
```
returns: `NONULL int`


#### bb_write_chars

```c
NONULL void bb_write_chars(bytes_builder_t *bb, char *c, int len);
```

writes a string to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``char *``                               **c**    
``int``                                  **len**  
======================================= ========= 
```
returns: `NONULL void`


#### bb_write_dyn_bytes

```c
NONULL void bb_write_dyn_bytes(bytes_builder_t *bb, const bytes_t *src);
```

writes bytes to the builder with a prefixed length. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
`bytes_tconst , * <#bytes-t>`_           **src**  
======================================= ========= 
```
returns: `NONULL void`


#### bb_write_fixed_bytes

```c
NONULL void bb_write_fixed_bytes(bytes_builder_t *bb, const bytes_t *src);
```

writes fixed bytes to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
`bytes_tconst , * <#bytes-t>`_           **src**  
======================================= ========= 
```
returns: `NONULL void`


#### bb_write_int

```c
NONULL void bb_write_int(bytes_builder_t *bb, uint32_t val);
```

writes a ineteger to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``uint32_t``                             **val**  
======================================= ========= 
```
returns: `NONULL void`


#### bb_write_long

```c
NONULL void bb_write_long(bytes_builder_t *bb, uint64_t val);
```

writes s long to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``uint64_t``                             **val**  
======================================= ========= 
```
returns: `NONULL void`


#### bb_write_long_be

```c
NONULL void bb_write_long_be(bytes_builder_t *bb, uint64_t val, int len);
```

writes any integer value with the given length of bytes 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``uint64_t``                             **val**  
``int``                                  **len**  
======================================= ========= 
```
returns: `NONULL void`


#### bb_write_byte

```c
NONULL void bb_write_byte(bytes_builder_t *bb, uint8_t val);
```

writes a single byte to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``uint8_t``                              **val**  
======================================= ========= 
```
returns: `NONULL void`


#### bb_write_raw_bytes

```c
NONULL void bb_write_raw_bytes(bytes_builder_t *bb, void *ptr, size_t len);
```

writes the bytes to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``void *``                               **ptr**  
``size_t``                               **len**  
======================================= ========= 
```
returns: `NONULL void`


#### bb_clear

```c
NONULL void bb_clear(bytes_builder_t *bb);
```

resets the content of the builder. 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```
returns: `NONULL void`


#### bb_replace

```c
NONULL void bb_replace(bytes_builder_t *bb, int offset, int delete_len, uint8_t *data, int data_len);
```

replaces or deletes a part of the content. 

arguments:
```eval_rst
======================================= ================ 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**          
``int``                                  **offset**      
``int``                                  **delete_len**  
``uint8_t *``                            **data**        
``int``                                  **data_len**    
======================================= ================ 
```
returns: `NONULL void`


#### bb_move_to_bytes

```c
RETURNS_NONULL NONULL bytes_t* bb_move_to_bytes(bytes_builder_t *bb);
```

frees the builder and moves the content in a newly created bytes struct (which needs to be freed later). 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```
returns: [`bytes_tRETURNS_NONULL NONULL , *`](#bytes-t)


#### bb_read_long

```c
NONULL uint64_t bb_read_long(bytes_builder_t *bb, size_t *i);
```

reads a long from the builder 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
``size_t *``                             **i**   
======================================= ======== 
```
returns: `NONULL uint64_t`


#### bb_read_int

```c
NONULL uint32_t bb_read_int(bytes_builder_t *bb, size_t *i);
```

reads a int from the builder 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
``size_t *``                             **i**   
======================================= ======== 
```
returns: `NONULL uint32_t`


#### bytes

```c
static bytes_t bytes(uint8_t *a, uint32_t len);
```

converts the given bytes to a bytes struct 

arguments:
```eval_rst
============= ========= 
``uint8_t *``  **a**    
``uint32_t``   **len**  
============= ========= 
```
returns: [`bytes_t`](#bytes-t)


#### cloned_bytes

```c
bytes_t cloned_bytes(bytes_t data);
```

cloned the passed data 

arguments:
```eval_rst
===================== ========== 
`bytes_t <#bytes-t>`_  **data**  
===================== ========== 
```
returns: [`bytes_t`](#bytes-t)


#### b_optimize_len

```c
static NONULL void b_optimize_len(bytes_t *b);
```

< changed the data and len to remove leading 0-bytes

arguments:
```eval_rst
======================= ======= 
`bytes_t * <#bytes-t>`_  **b**  
======================= ======= 
```
returns: `NONULL void`


#### b_compare

```c
static int b_compare(bytes_t a, bytes_t b);
```

arguments:
```eval_rst
===================== ======= 
`bytes_t <#bytes-t>`_  **a**  
`bytes_t <#bytes-t>`_  **b**  
===================== ======= 
```
returns: `int`


### data.h

json-parser. 

The parser can read from :

- json
- bin

When reading from json all '0x'... values will be stored as bytes_t. If the value is lower than 0xFFFFFFF, it is converted as integer. 

File: [c/src/core/util/data.h](https://github.com/slockit/in3-c/blob/master/c/src/core/util/data.h)

#### DATA_DEPTH_MAX

the max DEPTH of the JSON-data allowed. 

It will throw an error if reached. 

```c
#define DATA_DEPTH_MAX 11
```


#### printX

```c
#define printX printf
```


#### fprintX

```c
#define fprintX fprintf
```


#### snprintX

```c
#define snprintX snprintf
```


#### vprintX

```c
#define vprintX vprintf
```


#### d_type_t

type of a token. 

The enum type contains the following values:

```eval_rst
=============== = =====================================================
 **T_BYTES**    0 content is stored as data ptr.
 **T_STRING**   1 content is stored a c-str
 **T_ARRAY**    2 the node is an array with the length stored in length
 **T_OBJECT**   3 the node is an object with properties
 **T_BOOLEAN**  4 boolean with the value stored in len
 **T_INTEGER**  5 a integer with the value stored
 **T_NULL**     6 a NULL-value
=============== = =====================================================
```

#### d_key_t


```c
typedef uint16_t d_key_t
```

#### d_token_t

a token holding any kind of value. 

use d_type, d_len or the cast-function to get the value. 


The stuct contains following fields:

```eval_rst
============= ========== =====================================================================
``uint8_t *``  **data**  the byte or string-data
``uint32_t``   **len**   the length of the content (or number of properties) depending + type.
``d_key_t``    **key**   the key of the property.
============= ========== =====================================================================
```

#### str_range_t

internal type used to represent the a range within a string. 


The stuct contains following fields:

```eval_rst
========== ========== ==================================
``char *``  **data**  pointer to the start of the string
``size_t``  **len**   len of the characters
========== ========== ==================================
```

#### json_ctx_t

parser for json or binary-data. 

it needs to freed after usage. 


The stuct contains following fields:

```eval_rst
=========================== =============== ============================================================
`d_token_t * <#d-token-t>`_  **result**     the list of all tokens. 
                                            
                                            the first token is the main-token as returned by the parser.
``char *``                   **c**          pointer to the src-data
``size_t``                   **allocated**  amount of tokens allocated result
``size_t``                   **len**        number of tokens in result
``size_t``                   **depth**      max depth of tokens in result
``uint8_t *``                **keys**       
``size_t``                   **keys_last**  
=========================== =============== ============================================================
```

#### d_iterator_t

iterator over elements of a array opf object. 

usage: 

```c
for (d_iterator_t iter = d_iter( parent ); iter.left ; d_iter_next(&iter)) {
  uint32_t val = d_int(iter.token);
}
```

The stuct contains following fields:

```eval_rst
=========================== =========== =====================
`d_token_t * <#d-token-t>`_  **token**  current token
``int``                      **left**   number of result left
=========================== =========== =====================
```

#### d_to_bytes

```c
bytes_t d_to_bytes(d_token_t *item);
```

returns the byte-representation of token. 

In case of a number it is returned as bigendian. booleans as 0x01 or 0x00 and NULL as 0x. Objects or arrays will return 0x. 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
=========================== ========== 
```
returns: [`bytes_t`](#bytes-t)


#### d_bytes_to

```c
int d_bytes_to(d_token_t *item, uint8_t *dst, const int max);
```

writes the byte-representation to the dst. 

details see d_to_bytes. 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``uint8_t *``                **dst**   
``const int``                **max**   
=========================== ========== 
```
returns: `int`


#### d_bytes

```c
bytes_t* d_bytes(const d_token_t *item);
```

returns the value as bytes (Carefully, make sure that the token is a bytes-type!) 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### d_bytesl

```c
bytes_t* d_bytesl(d_token_t *item, size_t l);
```

returns the value as bytes with length l (may reallocates) 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``size_t``                   **l**     
=========================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### d_string

```c
char* d_string(const d_token_t *item);
```

converts the value as string. 

Make sure the type is string! 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `char *`


#### d_int

```c
int32_t d_int(const d_token_t *item);
```

returns the value as integer. 

only if type is integer 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `int32_t`


#### d_intd

```c
int32_t d_intd(const d_token_t *item, const uint32_t def_val);
```

returns the value as integer or if NULL the default. 

only if type is integer 

arguments:
```eval_rst
================================== ============= 
`d_token_tconst , * <#d-token-t>`_  **item**     
``const uint32_t``                  **def_val**  
================================== ============= 
```
returns: `int32_t`


#### d_long

```c
uint64_t d_long(const d_token_t *item);
```

returns the value as long. 

only if type is integer or bytes, but short enough 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `uint64_t`


#### d_longd

```c
uint64_t d_longd(const d_token_t *item, const uint64_t def_val);
```

returns the value as long or if NULL the default. 

only if type is integer or bytes, but short enough 

arguments:
```eval_rst
================================== ============= 
`d_token_tconst , * <#d-token-t>`_  **item**     
``const uint64_t``                  **def_val**  
================================== ============= 
```
returns: `uint64_t`


#### d_create_bytes_vec

```c
bytes_t** d_create_bytes_vec(const d_token_t *arr);
```

arguments:
```eval_rst
================================== ========= 
`d_token_tconst , * <#d-token-t>`_  **arr**  
================================== ========= 
```
returns: [`bytes_t **`](#bytes-t)


#### d_token_size

```c
size_t d_token_size(const d_token_t *item);
```

creates a array of bytes from JOSN-array 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `size_t`


#### d_type

```c
static d_type_t d_type(const d_token_t *item);
```

returns the size (number of tokens ) of the token 

type of the token 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: [`d_type_t`](#d-type-t)


#### d_len

```c
static int d_len(const d_token_t *item);
```

< number of elements in the token (only for object or array, other will return 0)

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `int`


#### d_eq

```c
bool d_eq(const d_token_t *a, const d_token_t *b);
```

compares 2 token and if the value is equal 

arguments:
```eval_rst
================================== ======= 
`d_token_tconst , * <#d-token-t>`_  **a**  
`d_token_tconst , * <#d-token-t>`_  **b**  
================================== ======= 
```
returns: `bool`


#### keyn

```c
NONULL d_key_t keyn(const char *c, const size_t len);
```

generates the keyhash for the given stringrange as defined by len 

arguments:
```eval_rst
================ ========= 
``const char *``  **c**    
``const size_t``  **len**  
================ ========= 
```
returns: `NONULL d_key_t`


#### ikey

```c
d_key_t ikey(json_ctx_t *ctx, const char *name);
```

returnes the indexed key for the given name. 

arguments:
```eval_rst
============================= ========== 
`json_ctx_t * <#json-ctx-t>`_  **ctx**   
``const char *``               **name**  
============================= ========== 
```
returns: `d_key_t`


#### d_get

```c
d_token_t* d_get(d_token_t *item, const uint16_t key);
```

returns the token with the given propertyname (only if item is a object) 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``const uint16_t``           **key**   
=========================== ========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_get_or

```c
d_token_t* d_get_or(d_token_t *item, const uint16_t key1, const uint16_t key2);
```

returns the token with the given propertyname or if not found, tries the other. 

(only if item is a object) 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``const uint16_t``           **key1**  
``const uint16_t``           **key2**  
=========================== ========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_get_at

```c
d_token_t* d_get_at(d_token_t *item, const uint32_t index);
```

returns the token of an array with the given index 

arguments:
```eval_rst
=========================== =========== 
`d_token_t * <#d-token-t>`_  **item**   
``const uint32_t``           **index**  
=========================== =========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_next

```c
d_token_t* d_next(d_token_t *item);
```

returns the next sibling of an array or object 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
=========================== ========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_serialize_binary

```c
NONULL void d_serialize_binary(bytes_builder_t *bb, d_token_t *t);
```

write the token as binary data into the builder 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
`d_token_t * <#d-token-t>`_              **t**   
======================================= ======== 
```
returns: `NONULL void`


#### parse_binary

```c
NONULL json_ctx_t* parse_binary(const bytes_t *data);
```

parses the data and returns the context with the token, which needs to be freed after usage! 

arguments:
```eval_rst
============================== ========== 
`bytes_tconst , * <#bytes-t>`_  **data**  
============================== ========== 
```
returns: [`json_ctx_tNONULL , *`](#json-ctx-t)


#### parse_binary_str

```c
NONULL json_ctx_t* parse_binary_str(const char *data, int len);
```

parses the data and returns the context with the token, which needs to be freed after usage! 

arguments:
```eval_rst
================ ========== 
``const char *``  **data**  
``int``           **len**   
================ ========== 
```
returns: [`json_ctx_tNONULL , *`](#json-ctx-t)


#### parse_json_error

```c
NONULL char* parse_json_error(const char *js);
```

parses the json, but only return an error if the json is invalid. 

The returning string must be freed! 

arguments:
```eval_rst
================ ======== 
``const char *``  **js**  
================ ======== 
```
returns: `NONULL char *`


#### parse_json

```c
NONULL json_ctx_t* parse_json(const char *js);
```

parses json-data, which needs to be freed after usage! 

arguments:
```eval_rst
================ ======== 
``const char *``  **js**  
================ ======== 
```
returns: [`json_ctx_tNONULL , *`](#json-ctx-t)


#### parse_json_indexed

```c
NONULL json_ctx_t* parse_json_indexed(const char *js);
```

parses json-data, which needs to be freed after usage! 

arguments:
```eval_rst
================ ======== 
``const char *``  **js**  
================ ======== 
```
returns: [`json_ctx_tNONULL , *`](#json-ctx-t)


#### json_free

```c
NONULL void json_free(json_ctx_t *parser_ctx);
```

frees the parse-context after usage 

arguments:
```eval_rst
============================= ================ 
`json_ctx_t * <#json-ctx-t>`_  **parser_ctx**  
============================= ================ 
```
returns: `NONULL void`


#### d_to_json

```c
NONULL str_range_t d_to_json(const d_token_t *item);
```

returns the string for a object or array. 

This only works for json as string. For binary it will not work! 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: [`str_range_tNONULL `](#str-range-t)


#### d_create_json

```c
char* d_create_json(json_ctx_t *ctx, d_token_t *item);
```

creates a json-string. 

It does not work for objects if the parsed data were binary! 

arguments:
```eval_rst
============================= ========== 
`json_ctx_t * <#json-ctx-t>`_  **ctx**   
`d_token_t * <#d-token-t>`_    **item**  
============================= ========== 
```
returns: `char *`


#### json_create

```c
json_ctx_t* json_create();
```

returns: [`json_ctx_t *`](#json-ctx-t)


#### json_create_null

```c
NONULL d_token_t* json_create_null(json_ctx_t *jp);
```

arguments:
```eval_rst
============================= ======== 
`json_ctx_t * <#json-ctx-t>`_  **jp**  
============================= ======== 
```
returns: [`d_token_tNONULL , *`](#d-token-t)


#### json_create_bool

```c
NONULL d_token_t* json_create_bool(json_ctx_t *jp, bool value);
```

arguments:
```eval_rst
============================= =========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**     
``bool``                       **value**  
============================= =========== 
```
returns: [`d_token_tNONULL , *`](#d-token-t)


#### json_create_int

```c
NONULL d_token_t* json_create_int(json_ctx_t *jp, uint64_t value);
```

arguments:
```eval_rst
============================= =========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**     
``uint64_t``                   **value**  
============================= =========== 
```
returns: [`d_token_tNONULL , *`](#d-token-t)


#### json_create_string

```c
NONULL d_token_t* json_create_string(json_ctx_t *jp, char *value, int len);
```

arguments:
```eval_rst
============================= =========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**     
``char *``                     **value**  
``int``                        **len**    
============================= =========== 
```
returns: [`d_token_tNONULL , *`](#d-token-t)


#### json_create_bytes

```c
NONULL d_token_t* json_create_bytes(json_ctx_t *jp, bytes_t value);
```

arguments:
```eval_rst
============================= =========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**     
`bytes_t <#bytes-t>`_          **value**  
============================= =========== 
```
returns: [`d_token_tNONULL , *`](#d-token-t)


#### json_create_object

```c
NONULL int json_create_object(json_ctx_t *jp);
```

arguments:
```eval_rst
============================= ======== 
`json_ctx_t * <#json-ctx-t>`_  **jp**  
============================= ======== 
```
returns: `NONULL int`


#### json_create_array

```c
NONULL int json_create_array(json_ctx_t *jp);
```

arguments:
```eval_rst
============================= ======== 
`json_ctx_t * <#json-ctx-t>`_  **jp**  
============================= ======== 
```
returns: `NONULL int`


#### json_object_add_prop

```c
NONULL void json_object_add_prop(json_ctx_t *jp, int ob_index, d_key_t key, d_token_t *value);
```

arguments:
```eval_rst
============================= ============== 
`json_ctx_t * <#json-ctx-t>`_  **jp**        
``int``                        **ob_index**  
``d_key_t``                    **key**       
`d_token_t * <#d-token-t>`_    **value**     
============================= ============== 
```
returns: `NONULL void`


#### json_create_ref_item

```c
NONULL d_token_t* json_create_ref_item(json_ctx_t *jp, d_type_t type, void *data, int len);
```

arguments:
```eval_rst
============================= ========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**    
`d_type_t <#d-type-t>`_        **type**  
``void *``                     **data**  
``int``                        **len**   
============================= ========== 
```
returns: [`d_token_tNONULL , *`](#d-token-t)


#### json_array_add_value

```c
NONULL void json_array_add_value(json_ctx_t *jp, int parent_index, d_token_t *value);
```

arguments:
```eval_rst
============================= ================== 
`json_ctx_t * <#json-ctx-t>`_  **jp**            
``int``                        **parent_index**  
`d_token_t * <#d-token-t>`_    **value**         
============================= ================== 
```
returns: `NONULL void`


#### token_from_string

```c
NONULL d_token_t* token_from_string(char *val, d_token_t *d, bytes32_t buffer);
```

returns a token ptr using the val without allocating memory in the heap, which can be used to pass values as token 

arguments:
```eval_rst
=========================== ============ 
``char *``                   **val**     
`d_token_t * <#d-token-t>`_  **d**       
`bytes32_t <#bytes32-t>`_    **buffer**  
=========================== ============ 
```
returns: [`d_token_tNONULL , *`](#d-token-t)


#### token_from_bytes

```c
NONULL d_token_t* token_from_bytes(bytes_t b, d_token_t *d);
```

returns a token ptr using the val without allocating memory in the heap, which can be used to pass values as token 

arguments:
```eval_rst
=========================== ======= 
`bytes_t <#bytes-t>`_        **b**  
`d_token_t * <#d-token-t>`_  **d**  
=========================== ======= 
```
returns: [`d_token_tNONULL , *`](#d-token-t)


#### d_get_keystr

```c
char* d_get_keystr(json_ctx_t *json, d_key_t k);
```

returns the string for a key. 

This only works for index keys or known keys! 

arguments:
```eval_rst
============================= ========== 
`json_ctx_t * <#json-ctx-t>`_  **json**  
``d_key_t``                    **k**     
============================= ========== 
```
returns: `char *`


#### key

```c
static NONULL d_key_t key(const char *c);
```

arguments:
```eval_rst
================ ======= 
``const char *``  **c**  
================ ======= 
```
returns: `NONULL d_key_t`


#### d_get_string

```c
static char* d_get_string(d_token_t *r, d_key_t k);
```

reads token of a property as string. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
=========================== ======= 
```
returns: `char *`


#### d_get_string_at

```c
static char* d_get_string_at(d_token_t *r, uint32_t pos);
```

reads string at given pos of an array. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **r**    
``uint32_t``                 **pos**  
=========================== ========= 
```
returns: `char *`


#### d_get_int

```c
static int32_t d_get_int(d_token_t *r, d_key_t k);
```

reads token of a property as int. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
=========================== ======= 
```
returns: `int32_t`


#### d_get_intd

```c
static int32_t d_get_intd(d_token_t *r, d_key_t k, uint32_t d);
```

reads token of a property as int. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
``uint32_t``                 **d**  
=========================== ======= 
```
returns: `int32_t`


#### d_get_int_at

```c
static int32_t d_get_int_at(d_token_t *r, uint32_t pos);
```

reads a int at given pos of an array. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **r**    
``uint32_t``                 **pos**  
=========================== ========= 
```
returns: `int32_t`


#### d_get_long

```c
static uint64_t d_get_long(d_token_t *r, d_key_t k);
```

reads token of a property as long. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
=========================== ======= 
```
returns: `uint64_t`


#### d_get_longd

```c
static uint64_t d_get_longd(d_token_t *r, d_key_t k, uint64_t d);
```

reads token of a property as long. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
``uint64_t``                 **d**  
=========================== ======= 
```
returns: `uint64_t`


#### d_get_long_at

```c
static uint64_t d_get_long_at(d_token_t *r, uint32_t pos);
```

reads long at given pos of an array. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **r**    
``uint32_t``                 **pos**  
=========================== ========= 
```
returns: `uint64_t`


#### d_get_bytes

```c
static bytes_t* d_get_bytes(d_token_t *r, d_key_t k);
```

reads token of a property as bytes. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
=========================== ======= 
```
returns: [`bytes_t *`](#bytes-t)


#### d_get_bytes_at

```c
static bytes_t* d_get_bytes_at(d_token_t *r, uint32_t pos);
```

reads bytes at given pos of an array. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **r**    
``uint32_t``                 **pos**  
=========================== ========= 
```
returns: [`bytes_t *`](#bytes-t)


#### d_is_binary_ctx

```c
static bool d_is_binary_ctx(json_ctx_t *ctx);
```

check if the parser context was created from binary data. 

arguments:
```eval_rst
============================= ========= 
`json_ctx_t * <#json-ctx-t>`_  **ctx**  
============================= ========= 
```
returns: `bool`


#### d_get_byteskl

```c
bytes_t* d_get_byteskl(d_token_t *r, d_key_t k, uint32_t minl);
```

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **r**     
``d_key_t``                  **k**     
``uint32_t``                 **minl**  
=========================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### d_getl

```c
d_token_t* d_getl(d_token_t *item, uint16_t k, uint32_t minl);
```

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``uint16_t``                 **k**     
``uint32_t``                 **minl**  
=========================== ========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_iter

```c
d_iterator_t d_iter(d_token_t *parent);
```

creates a iterator for a object or array 

arguments:
```eval_rst
=========================== ============ 
`d_token_t * <#d-token-t>`_  **parent**  
=========================== ============ 
```
returns: [`d_iterator_t`](#d-iterator-t)


#### d_iter_next

```c
static bool d_iter_next(d_iterator_t *const iter);
```

fetched the next token an returns a boolean indicating whther there is a next or not. 

arguments:
```eval_rst
====================================== ========== 
`d_iterator_t *const <#d-iterator-t>`_  **iter**  
====================================== ========== 
```
returns: `bool`


### debug.h

logs debug data only if the DEBUG-flag is set. 

File: [c/src/core/util/debug.h](https://github.com/slockit/in3-c/blob/master/c/src/core/util/debug.h)

#### dbg_log (msg,...)

logs a debug-message including file and linenumber 


#### dbg_log_raw (msg,...)

logs a debug-message without the filename 


#### _assert (exp)


#### EXPECT (cond,exit)

```c
#define EXPECT (cond,exit)   do {                     \
    if (!(cond))           \
      exit;                \
  } while (0)
```


#### EXPECT_CFG (cond,err)

```c
#define EXPECT_CFG (cond,err)   EXPECT(cond, { \
  res = malloc(strlen(err) + 1);             \
  if (res) strcpy(res, err);                 \
  goto cleanup;                              \
})
```


#### EXPECT_CFG_NCP_ERR (cond,err)

```c
#define EXPECT_CFG_NCP_ERR (cond,err) EXPECT(cond, { res = err; goto cleanup; })
```


#### EXPECT_TOK (token,cond,err)

```c
#define EXPECT_TOK (token,cond,err) EXPECT_CFG_NCP_ERR(cond, config_err(d_get_keystr(json, (token)->key), err))
```


#### EXPECT_TOK_BOOL (token)

```c
#define EXPECT_TOK_BOOL (token) EXPECT_TOK(token, d_type(token) == T_BOOLEAN, "expected boolean value")
```


#### EXPECT_TOK_STR (token)

```c
#define EXPECT_TOK_STR (token) EXPECT_TOK(token, d_type(token) == T_STRING, "expected string value")
```


#### EXPECT_TOK_ARR (token)

```c
#define EXPECT_TOK_ARR (token) EXPECT_TOK(token, d_type(token) == T_ARRAY, "expected array")
```


#### EXPECT_TOK_OBJ (token)

```c
#define EXPECT_TOK_OBJ (token) EXPECT_TOK(token, d_type(token) == T_OBJECT, "expected object")
```


#### EXPECT_TOK_ADDR (token)

```c
#define EXPECT_TOK_ADDR (token) EXPECT_TOK(token, d_type(token) == T_BYTES && d_len(token) == 20, "expected address")
```


#### EXPECT_TOK_B256 (token)

```c
#define EXPECT_TOK_B256 (token) EXPECT_TOK(token, d_type(token) == T_BYTES && d_len(token) == 32, "expected 256 bit data")
```


#### IS_D_UINT64 (token)

```c
#define IS_D_UINT64 (token) ((d_type(token) == T_INTEGER || (d_type(token) == T_BYTES && d_len(token) <= 8)) && d_long(token) <= UINT64_MAX)
```


#### IS_D_UINT32 (token)

```c
#define IS_D_UINT32 (token) ((d_type(token) == T_INTEGER || d_type(token) == T_BYTES) && d_long(token) <= UINT32_MAX)
```


#### IS_D_UINT16 (token)

```c
#define IS_D_UINT16 (token) (d_type(token) == T_INTEGER && d_int(token) >= 0 && d_int(token) <= UINT16_MAX)
```


#### IS_D_UINT8 (token)

```c
#define IS_D_UINT8 (token) (d_type(token) == T_INTEGER && d_int(token) >= 0 && d_int(token) <= UINT8_MAX)
```


#### EXPECT_TOK_U8 (token)

```c
#define EXPECT_TOK_U8 (token) EXPECT_TOK(token, IS_D_UINT8(token), "expected uint8 value")
```


#### EXPECT_TOK_U16 (token)

```c
#define EXPECT_TOK_U16 (token) EXPECT_TOK(token, IS_D_UINT16(token), "expected uint16 value")
```


#### EXPECT_TOK_U32 (token)

```c
#define EXPECT_TOK_U32 (token) EXPECT_TOK(token, IS_D_UINT32(token), "expected uint32 value")
```


#### EXPECT_TOK_U64 (token)

```c
#define EXPECT_TOK_U64 (token) EXPECT_TOK(token, IS_D_UINT64(token), "expected uint64 value")
```


#### EXPECT_TOK_KEY_HEXSTR (token)

```c
#define EXPECT_TOK_KEY_HEXSTR (token) EXPECT_TOK(token, is_hex_str(d_get_keystr(json, (token)->key)), "expected hex str")
```


#### msg_dump

```c
void msg_dump(const char *s, const unsigned char *data, unsigned len);
```

dumps the given data as hex coded bytes to stdout 

arguments:
```eval_rst
========================= ========== 
``const char *``           **s**     
``const unsigned char *``  **data**  
``unsigned``               **len**   
========================= ========== 
```

#### config_err

```c
static char* config_err(const char *keyname, const char *err);
```

arguments:
```eval_rst
================ ============= 
``const char *``  **keyname**  
``const char *``  **err**      
================ ============= 
```
returns: `char *`


#### is_hex_str

```c
static bool is_hex_str(const char *str);
```

arguments:
```eval_rst
================ ========= 
``const char *``  **str**  
================ ========= 
```
returns: `bool`


#### add_prop

```c
static void add_prop(sb_t *sb, char prefix, const char *property);
```

arguments:
```eval_rst
================= ============== 
`sb_t * <#sb-t>`_  **sb**        
``char``           **prefix**    
``const char *``   **property**  
================= ============== 
```

#### add_bool

```c
static void add_bool(sb_t *sb, char prefix, const char *property, bool value);
```

arguments:
```eval_rst
================= ============== 
`sb_t * <#sb-t>`_  **sb**        
``char``           **prefix**    
``const char *``   **property**  
``bool``           **value**     
================= ============== 
```

#### add_string

```c
static void add_string(sb_t *sb, char prefix, const char *property, const char *value);
```

arguments:
```eval_rst
================= ============== 
`sb_t * <#sb-t>`_  **sb**        
``char``           **prefix**    
``const char *``   **property**  
``const char *``   **value**     
================= ============== 
```

#### add_uint

```c
static void add_uint(sb_t *sb, char prefix, const char *property, uint64_t value);
```

arguments:
```eval_rst
================= ============== 
`sb_t * <#sb-t>`_  **sb**        
``char``           **prefix**    
``const char *``   **property**  
``uint64_t``       **value**     
================= ============== 
```

#### add_hex

```c
static void add_hex(sb_t *sb, char prefix, const char *property, bytes_t value);
```

arguments:
```eval_rst
===================== ============== 
`sb_t * <#sb-t>`_      **sb**        
``char``               **prefix**    
``const char *``       **property**  
`bytes_t <#bytes-t>`_  **value**     
===================== ============== 
```

### error.h

defines the return-values of a function call. 

File: [c/src/core/util/error.h](https://github.com/slockit/in3-c/blob/master/c/src/core/util/error.h)

#### DEPRECATED

depreacted-attribute 

```c
#define DEPRECATED __attribute__((deprecated))
```


#### JSON_RPC_ERR_INTERNAL

JSON-RPC errors. 

```c
#define JSON_RPC_ERR_INTERNAL (-32603) /** Internal error (probably offline signer node) */
```


#### JSON_RPC_ERR_FINALITY

```c
#define JSON_RPC_ERR_FINALITY (-16001) /** Block is not final so node refused to sign */
```


#### OPTIONAL_T (t)

Optional type similar to C++ std::optional Optional types must be defined prior to usage (e.g. 

DEFINE_OPTIONAL_T(int)) Use OPTIONAL_T_UNDEFINED(t) & OPTIONAL_T_VALUE(t, v) for easy initialization (rvalues) Note: Defining optional types for pointers is ill-formed by definition. This is because redundant 

```c
#define OPTIONAL_T (t) opt_##t
```


#### DEFINE_OPTIONAL_T (t)

Optional types must be defined prior to usage (e.g. 

DEFINE_OPTIONAL_T(int)) Use OPTIONAL_T_UNDEFINED(t) & OPTIONAL_T_VALUE(t, v) for easy initialization (rvalues) 

```c
#define DEFINE_OPTIONAL_T (t)   typedef struct {           \
    t    value;              \
    bool defined;            \
  } OPTIONAL_T(t)
```


#### OPTIONAL_T_UNDEFINED (t)

marks a used value as undefined. 

```c
#define OPTIONAL_T_UNDEFINED (t) ((OPTIONAL_T(t)){.defined = false})
```


#### OPTIONAL_T_VALUE (t,v)

sets the value of an optional type. 

```c
#define OPTIONAL_T_VALUE (t,v) ((OPTIONAL_T(t)){.value = v, .defined = true})
```


#### in3_ret_t

ERROR types used as return values. 

All values (except IN3_OK) indicate an error. IN3_WAITING may be treated like an error, since we have stop executing until the response has arrived, but it is a valid return value. 

The enum type contains the following values:

```eval_rst
================================= ==== ==================================================================
 **IN3_OK**                       0    Success.
 **IN3_EUNKNOWN**                 -1   Unknown error - usually accompanied with specific error msg.
 **IN3_ENOMEM**                   -2   No memory.
 **IN3_ENOTSUP**                  -3   Not supported.
 **IN3_EINVAL**                   -4   Invalid value.
 **IN3_EFIND**                    -5   Not found.
 **IN3_ECONFIG**                  -6   Invalid config.
 **IN3_ELIMIT**                   -7   Limit reached.
 **IN3_EVERS**                    -8   Version mismatch.
 **IN3_EINVALDT**                 -9   Data invalid, eg. 
                                       
                                       invalid/incomplete JSON
 **IN3_EPASS**                    -10  Wrong password.
 **IN3_ERPC**                     -11  RPC error (i.e. 
                                       
                                       in3_req_t::error set)
 **IN3_ERPCNRES**                 -12  RPC no response.
 **IN3_EUSNURL**                  -13  USN URL parse error.
 **IN3_ETRANS**                   -14  Transport error.
 **IN3_ERANGE**                   -15  Not in range.
 **IN3_WAITING**                  -16  the process can not be finished since we are waiting for responses
 **IN3_EIGNORE**                  -17  Ignorable error.
 **IN3_EPAYMENT_REQUIRED**        -18  payment required
 **IN3_ENODEVICE**                -19  harware wallet device not connected
 **IN3_EAPDU**                    -20  error in hardware wallet communication
 **IN3_EPLGN_NONE**               -21  no plugin could handle specified action
 **IN3_ERETRY**                   -22  request to retry all plugins again
 **IN3_HTTP_BAD_REQUEST**         -400 Bad Request.
 **IN3_HTTP_UNAUTHORIZED**        -401 Unauthorized.
 **IN3_HTTP_PAYMENT_REQUIRED**    -402 Unauthorized.
 **IN3_HTTP_FORBIDDEN**           -403 Forbidden.
 **IN3_HTTP_NOT_FOUND**           -404 not found
 **IN3_HTTP_M_NOT_ALLOWED**       -405 method not allowed
 **IN3_HTTP_NOT_ACCEPTABLE**      -406 Not acceptable.
 **IN3_HTTP_PROX_AUTH_REQUIRED**  -407 Proxy Authentification required.
 **IN3_HTTP_TIMEOUT**             -408 Request timeout.
 **IN3_HTTP_CONFLICT**            -409 conflict
 **IN3_HTTP_GONE**                -410 gone
 **IN3_HTTP_INTERNAL_ERROR**      -500 Internal Server Error.
 **IN3_HTTP_NOT_IMPLEMENTED**     -501 not implemented
 **IN3_HTTP_BAD_GATEWAY**         -502 Bad Gateway.
 **IN3_HTTP_UNAVAILABLE**         -503 service unavailable
================================= ==== ==================================================================
```

#### in3_errmsg

```c
char* in3_errmsg(in3_ret_t err);
```

converts a error code into a string. 

These strings are constants and do not need to be freed. 

arguments:
```eval_rst
========================= ========= ==============
`in3_ret_t <#in3-ret-t>`_  **err**  the error code
========================= ========= ==============
```
returns: `char *`


### scache.h

util helper on byte arrays. 

File: [c/src/core/util/scache.h](https://github.com/slockit/in3-c/blob/master/c/src/core/util/scache.h)

#### cache_props

The enum type contains the following values:

```eval_rst
================================== ==== =================================================================
 **CACHE_PROP_MUST_FREE**          0x1  indicates the content must be freed
 **CACHE_PROP_SRC_REQ**            0x2  the value holds the src-request
 **CACHE_PROP_ONLY_EXTERNAL**      0x4  should only be freed if the context is external
 **CACHE_PROP_ONLY_NOT_EXTERNAL**  0x20 should only be freed if the context is not external
 **CACHE_PROP_JSON**               0x8  indicates the content is a json_ctxt and must be freed as such
 **CACHE_PROP_INHERIT**            0x10 indicates the content will be inherited when creating sub_request
 **CACHE_PROP_PAYMENT**            0x80 This cache-entry is a payment.data.
================================== ==== =================================================================
```

#### cache_props_t


The enum type contains the following values:

```eval_rst
================================== ==== =================================================================
 **CACHE_PROP_MUST_FREE**          0x1  indicates the content must be freed
 **CACHE_PROP_SRC_REQ**            0x2  the value holds the src-request
 **CACHE_PROP_ONLY_EXTERNAL**      0x4  should only be freed if the context is external
 **CACHE_PROP_ONLY_NOT_EXTERNAL**  0x20 should only be freed if the context is not external
 **CACHE_PROP_JSON**               0x8  indicates the content is a json_ctxt and must be freed as such
 **CACHE_PROP_INHERIT**            0x10 indicates the content will be inherited when creating sub_request
 **CACHE_PROP_PAYMENT**            0x80 This cache-entry is a payment.data.
================================== ==== =================================================================
```

#### cache_entry_t

represents a single cache entry in a linked list. 

These are used within a request context to cache values and automaticly free them. 


The stuct contains following fields:

```eval_rst
======================================= ============ ==============================================================================
`bytes_t <#bytes-t>`_                    **key**     an optional key of the entry
`bytes_t <#bytes-t>`_                    **value**   the value
``uint8_t``                              **buffer**  the buffer is used to store extra data, which will be cleaned when freed.
``cache_props_t``                        **props**   if true, the cache-entry will be freed when the request context is cleaned up.
`cache_entrystruct , * <#cache-entry>`_  **next**    pointer to the next entry.
======================================= ============ ==============================================================================
```

#### in3_cache_get_entry

```c
bytes_t* in3_cache_get_entry(cache_entry_t *cache, bytes_t *key);
```

get the entry for a given key. 

arguments:
```eval_rst
=================================== =========== ==================================
`cache_entry_t * <#cache-entry-t>`_  **cache**  the root entry of the linked list.
`bytes_t * <#bytes-t>`_              **key**    the key to compare with
=================================== =========== ==================================
```
returns: [`bytes_t *`](#bytes-t)


#### in3_cache_get_entry_by_prop

```c
cache_entry_t* in3_cache_get_entry_by_prop(cache_entry_t *cache, cache_props_t prop);
```

get the entry for a given property. 

arguments:
```eval_rst
=================================== =========== ==================================
`cache_entry_t * <#cache-entry-t>`_  **cache**  the root entry of the linked list.
``cache_props_t``                    **prop**   the prop, which must match exactly
=================================== =========== ==================================
```
returns: [`cache_entry_t *`](#cache-entry-t)


#### in3_cache_add_entry

```c
cache_entry_t* in3_cache_add_entry(cache_entry_t **cache, bytes_t key, bytes_t value);
```

adds an entry to the linked list. 

arguments:
```eval_rst
==================================== =========== ==================================
`cache_entry_t ** <#cache-entry-t>`_  **cache**  the root entry of the linked list.
`bytes_t <#bytes-t>`_                 **key**    an optional key
`bytes_t <#bytes-t>`_                 **value**  the value of the entry
==================================== =========== ==================================
```
returns: [`cache_entry_t *`](#cache-entry-t)


#### in3_cache_free

```c
void in3_cache_free(cache_entry_t *cache, bool is_external);
```

clears all entries in the linked list. 

arguments:
```eval_rst
=================================== ================= ================================================
`cache_entry_t * <#cache-entry-t>`_  **cache**        the root entry of the linked list.
``bool``                             **is_external**  true if this is the root context or an external.
=================================== ================= ================================================
```

#### in3_cache_add_ptr

```c
static NONULL cache_entry_t* in3_cache_add_ptr(cache_entry_t **cache, void *ptr);
```

adds a pointer, which should be freed when the context is freed. 

arguments:
```eval_rst
==================================== =========== =======================================
`cache_entry_t ** <#cache-entry-t>`_  **cache**  the root entry of the linked list.
``void *``                            **ptr**    pointer to memory which shold be freed.
==================================== =========== =======================================
```
returns: [`cache_entry_tNONULL , *`](#cache-entry-t)


### stringbuilder.h

simple string buffer used to dynamicly add content. 

File: [c/src/core/util/stringbuilder.h](https://github.com/slockit/in3-c/blob/master/c/src/core/util/stringbuilder.h)

#### sb_add_hexuint (sb,i)

shortcut macro for adding a uint to the stringbuilder using sizeof(i) to automaticly determine the size 

```c
#define sb_add_hexuint (sb,i) sb_add_hexuint_l(sb, i, sizeof(i))
```


#### sb_t

string build struct, which is able to hold and modify a growing string. 


The stuct contains following fields:

```eval_rst
========== ============== ====================================
``char *``  **data**      the current string (null terminated)
``size_t``  **allocted**  number of bytes currently allocated
``size_t``  **len**       the current length of the string
========== ============== ====================================
```

#### sb_stack

```c
static NONULL sb_t sb_stack(char *p);
```

creates a stringbuilder which is allocating any new memory, but uses an existing string and is used directly on the stack. 

Since it will not grow the memory you need to pass a char* which allocated enough memory. 

arguments:
```eval_rst
========== ======= 
``char *``  **p**  
========== ======= 
```
returns: [`sb_tNONULL `](#sb-t)


#### sb_new

```c
sb_t* sb_new(const char *chars);
```

creates a new stringbuilder and copies the inital characters into it. 

arguments:
```eval_rst
================ =========== 
``const char *``  **chars**  
================ =========== 
```
returns: [`sb_t *`](#sb-t)


#### sb_init

```c
NONULL sb_t* sb_init(sb_t *sb);
```

initializes a stringbuilder by allocating memory. 

arguments:
```eval_rst
================= ======== 
`sb_t * <#sb-t>`_  **sb**  
================= ======== 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### sb_free

```c
NONULL void sb_free(sb_t *sb);
```

frees all resources of the stringbuilder 

arguments:
```eval_rst
================= ======== 
`sb_t * <#sb-t>`_  **sb**  
================= ======== 
```
returns: `NONULL void`


#### sb_add_char

```c
NONULL sb_t* sb_add_char(sb_t *sb, char c);
```

add a single character 

arguments:
```eval_rst
================= ======== 
`sb_t * <#sb-t>`_  **sb**  
``char``           **c**   
================= ======== 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### sb_add_chars

```c
NONULL sb_t* sb_add_chars(sb_t *sb, const char *chars);
```

adds a string 

arguments:
```eval_rst
================= =========== 
`sb_t * <#sb-t>`_  **sb**     
``const char *``   **chars**  
================= =========== 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### sb_add_range

```c
NONULL sb_t* sb_add_range(sb_t *sb, const char *chars, int start, int len);
```

add a string range 

arguments:
```eval_rst
================= =========== 
`sb_t * <#sb-t>`_  **sb**     
``const char *``   **chars**  
``int``            **start**  
``int``            **len**    
================= =========== 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### sb_add_key_value

```c
NONULL sb_t* sb_add_key_value(sb_t *sb, const char *key, const char *value, int value_len, bool as_string);
```

adds a value with an optional key. 

if as_string is true the value will be quoted. 

arguments:
```eval_rst
================= =============== 
`sb_t * <#sb-t>`_  **sb**         
``const char *``   **key**        
``const char *``   **value**      
``int``            **value_len**  
``bool``           **as_string**  
================= =============== 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### sb_add_bytes

```c
sb_t* sb_add_bytes(sb_t *sb, const char *prefix, const bytes_t *bytes, int len, bool as_array);
```

add bytes as 0x-prefixed hexcoded string (including an optional prefix), if len>1 is passed bytes maybe an array ( if as_array==true) 


 

arguments:
```eval_rst
============================== ============== 
`sb_t * <#sb-t>`_               **sb**        
``const char *``                **prefix**    
`bytes_tconst , * <#bytes-t>`_  **bytes**     
``int``                         **len**       
``bool``                        **as_array**  
============================== ============== 
```
returns: [`sb_t *`](#sb-t)


#### sb_add_hexuint_l

```c
NONULL sb_t* sb_add_hexuint_l(sb_t *sb, uintmax_t uint, size_t l);
```

add a integer value as hexcoded, 0x-prefixed string 

Other types not supported

arguments:
```eval_rst
================= ========== 
`sb_t * <#sb-t>`_  **sb**    
``uintmax_t``      **uint**  
``size_t``         **l**     
================= ========== 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### sb_add_escaped_chars

```c
NONULL sb_t* sb_add_escaped_chars(sb_t *sb, const char *chars);
```

add chars but escapes all quotes 

arguments:
```eval_rst
================= =========== 
`sb_t * <#sb-t>`_  **sb**     
``const char *``   **chars**  
================= =========== 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### sb_add_int

```c
NONULL sb_t* sb_add_int(sb_t *sb, int64_t val);
```

adds a numeric value to the stringbuilder 

arguments:
```eval_rst
================= ========= 
`sb_t * <#sb-t>`_  **sb**   
``int64_t``        **val**  
================= ========= 
```
returns: [`sb_tNONULL , *`](#sb-t)


#### format_json

```c
NONULL char* format_json(const char *json);
```

format a json string and returns a new string, which needs to be freed 

arguments:
```eval_rst
================ ========== 
``const char *``  **json**  
================ ========== 
```
returns: `NONULL char *`


#### sb_add_rawbytes

```c
sb_t* sb_add_rawbytes(sb_t *sb, char *prefix, bytes_t b, int fix_size);
```

arguments:
```eval_rst
===================== ============== 
`sb_t * <#sb-t>`_      **sb**        
``char *``             **prefix**    
`bytes_t <#bytes-t>`_  **b**         
``int``                **fix_size**  
===================== ============== 
```
returns: [`sb_t *`](#sb-t)


#### sb_print

```c
sb_t* sb_print(sb_t *sb, const char *fmt,...);
```

arguments:
```eval_rst
================= ========= 
`sb_t * <#sb-t>`_  **sb**   
``const char *``   **fmt**  
``...``                     
================= ========= 
```
returns: [`sb_t *`](#sb-t)


#### sb_vprint

```c
sb_t* sb_vprint(sb_t *sb, const char *fmt, va_list args);
```

arguments:
```eval_rst
================= ========== 
`sb_t * <#sb-t>`_  **sb**    
``const char *``   **fmt**   
``va_list``        **args**  
================= ========== 
```
returns: [`sb_t *`](#sb-t)


#### sb_add_json

```c
sb_t* sb_add_json(sb_t *sb, const char *prefix, d_token_t *token);
```

arguments:
```eval_rst
=========================== ============ 
`sb_t * <#sb-t>`_            **sb**      
``const char *``             **prefix**  
`d_token_t * <#d-token-t>`_  **token**   
=========================== ============ 
```
returns: [`sb_t *`](#sb-t)


#### sb_printx

```c
sb_t* sb_printx(sb_t *sb, const char *fmt,...);
```

arguments:
```eval_rst
================= ========= 
`sb_t * <#sb-t>`_  **sb**   
``const char *``   **fmt**  
``...``                     
================= ========= 
```
returns: [`sb_t *`](#sb-t)


### utils.h

utility functions. 

File: [c/src/core/util/utils.h](https://github.com/slockit/in3-c/blob/master/c/src/core/util/utils.h)

#### _strtoull (str,endptr,base)

```c
#define _strtoull (str,endptr,base) strtoull(str, endptr, base)
```


#### SWAP (a,b)

simple swap macro for integral types 

```c
#define SWAP (a,b)   {                \
    void* p = a;   \
    a       = b;   \
    b       = p;   \
  }
```


#### min (a,b)

simple min macro for interagl types 

```c
#define min (a,b) ((a) < (b) ? (a) : (b))
```


#### max (a,b)

simple max macro for interagl types 

```c
#define max (a,b) ((a) > (b) ? (a) : (b))
```


#### IS_APPROX (n1,n2,err)

Check if n1 & n2 are at max err apart Expects n1 & n2 to be integral types. 

```c
#define IS_APPROX (n1,n2,err) ((n1 > n2) ? ((n1 - n2) <= err) : ((n2 - n1) <= err))
```


#### DIFF_ATMOST (n1,n2,diff)

```c
#define DIFF_ATMOST (n1,n2,diff) IS_APPROX(n1, n2, diff)
```


#### DIFF_ATLEAST (n1,n2,err)

```c
#define DIFF_ATLEAST (n1,n2,err) ((n1 > n2) ? ((n1 - n2) >= err) : ((n2 - n1) >= err))
```


#### STR_IMPL_ (x)

simple macro to stringify other macro defs eg. 

usage - to concatenate a const with a string at compile time -> define SOME_CONST_UINT 10U printf("Using default value of " STR(SOME_CONST_UINT)); 

```c
#define STR_IMPL_ (x) #x
```


#### STR (x)

```c
#define STR (x) STR_IMPL_(x)
```


#### optimize_len (a,l)

changes to pointer (a) and it length (l) to remove leading 0 bytes. 

it will reduce it to max len=1 

```c
#define optimize_len (a,l)   while (l > 1 && *a == 0) { \
    l--;                     \
    a++;                     \
  }
```


#### TRY (exp)

executes the expression and expects the return value to be a int indicating the error. 

if the return value is negative it will stop and return this value otherwise continue. 

```c
#define TRY (exp)   {                        \
    int _r = (exp);        \
    if (_r < 0) return _r; \
  }
```


#### TRY_FINAL (exp,final)

executes the expression and expects the return value to be a int indicating the error. 

if the return value is negative it will stop and return this value otherwise continue. 

```c
#define TRY_FINAL (exp,final)   {                           \
    int _r = (exp);           \
    final;                    \
    if (_r < 0) return _r;    \
  }
```


#### TRY_CATCH (exp,catch)

executes the expression and expects the return value to be a int indicating the error. 

if the return value is negative it will stop and return this value otherwise continue. 

```c
#define TRY_CATCH (exp,catch)   {                           \
    int _r = (exp);           \
    if (_r < 0) {             \
      catch;                  \
      return _r;              \
    }                         \
  }
```


#### TRY_RPC (name,fn)

```c
#define TRY_RPC (name,fn)   if (strcmp(ctx->method, name) == 0) return fn;
```


#### VERIFY_RPC (name)

used in if-conditions and returns true if the vc->method mathes the name. 

It is also used as marker. 

```c
#define VERIFY_RPC (name) (strcmp(vc->method, name) == 0)
```


#### CONFIG_KEY (name)

```c
#define CONFIG_KEY (name) key(name)
```


#### EXPECT_EQ (exp,val)

executes the expression and expects value to equal val. 

if not it will return IN3_EINVAL 

```c
#define EXPECT_EQ (exp,val)   if ((exp) != val) return IN3_EINVAL;
```


#### TRY_SET (var,exp)

executes the expression and expects the return value to be a int indicating the error. 

the return value will be set to a existing variable (var). if the return value is negative it will stop and return this value otherwise continue. 

```c
#define TRY_SET (var,exp)   {                          \
    var = (exp);             \
    if (var < 0) return var; \
  }
```


#### TRY_GOTO (exp)

executes the expression and expects the return value to be a int indicating the error. 

if the return value is negative it will stop and jump (goto) to a marked position "clean". it also expects a previously declared variable "in3_ret_t res". 

```c
#define TRY_GOTO (exp)   {                          \
    res = (exp);             \
    if (res < 0) goto clean; \
  }
```


#### WORD_ADR (index,right)

calculates the address of a word in a abi-encoded data (assuming data = bytes_t res exists) 

```c
#define WORD_ADR (index,right) (res.data + 4 + (index) *32 + 32 - (right))
```


#### ABI_ADDRESS (index,adr)

sets an address at the word index in a abi-encoded data (assuming data = bytes_t res exists) 

```c
#define ABI_ADDRESS (index,adr) memcpy(WORD_ADR(index, 20), adr, 20)
```


#### ABI_UINT32 (index,val)

sets an int at the word index in a abi-encoded data (assuming data = bytes_t res exists) 

```c
#define ABI_UINT32 (index,val) int_to_bytes(val, WORD_ADR(index, 4))
```


#### ABI_UINT256 (index,data,len)

sets an uint256 as bytes at the word index in a abi-encoded data (assuming data = bytes_t res exists) 

```c
#define ABI_UINT256 (index,data,len) memcpy(WORD_ADR(index, len), data, len)
```


#### ABI_BYTES (index,bytes)

writes the bytes at the word index in a abi-encoded data (assuming data = bytes_t res exists) 

```c
#define ABI_BYTES (index,bytes)   {                                                                     \
    if (bytes.data) memcpy(WORD_ADR(index, 32), bytes.data, bytes.len); \
  }
```


#### ABI_FNC (hash)

writes the functionhash in a abi-encoded data (assuming data = bytes_t res exists) 

```c
#define ABI_FNC (hash) memcpy(res.data, (void*) hash, 4)
```


#### ABI_BYTES_CALLOC (words)

allocates memory filled with zeros with the size words*32 +4 for e3ncoding abi-data 

```c
#define ABI_BYTES_CALLOC (words) bytes(_calloc(4 + (words) *32, 1), 4 + (words) *32)
```


#### ABI_WORDS (byte_len)

calculates the number of words (32 bytes) needed to hold the specified bytes 

```c
#define ABI_WORDS (byte_len) ((byte_len + 31) / 32)
```


#### ABI_OFFSET (index,word)

writes the offset (as word) at the word index in a abi-encoded data (assuming data = bytes_t res exists) 

```c
#define ABI_OFFSET (index,word) ABI_UINT32(index, (word * 32))
```


#### INIT_LOCK (NAME)

```c
#define INIT_LOCK (NAME)   {}
```


#### LOCK (NAME,code)

```c
#define LOCK (NAME,code)   { code }
```


#### time_func

Pluggable functions: Mechanism to replace library functions with custom alternatives. 

This is particularly useful for embedded systems which have their own time or rand functions.

eg. // define function with specified signature uint64_t my_time(void* t) { // ... }

// then call in3_set_func_*() int main() { in3_set_func_time(my_time); // Henceforth, all library calls will use my_time() instead of the platform default time function } time function defaults to k_uptime_get() for zeohyr and time(NULL) for other platforms expected to return a u64 value representative of time (from epoch/start) 


```c
typedef uint64_t(* time_func) (void *t)
```

returns: `uint64_t(*`


#### rand_func

rand function defaults to k_uptime_get() for zeohyr and rand() for other platforms expected to return a random number 


```c
typedef int(* rand_func) (void *s)
```

returns: `int(*`


#### srand_func

srand function defaults to NOOP for zephyr and srand() for other platforms expected to set the seed for a new sequence of random numbers to be returned by in3_rand() 


```c
typedef void(* srand_func) (unsigned int s)
```


#### bytes_to_long

```c
uint64_t bytes_to_long(const uint8_t *data, int len);
```

converts the bytes to a unsigned long (at least the last max len bytes). 

arguments:
```eval_rst
=================== ========== 
``const uint8_t *``  **data**  
``int``              **len**   
=================== ========== 
```
returns: `uint64_t`


#### bytes_to_int

```c
static uint32_t bytes_to_int(const uint8_t *data, int len);
```

converts the bytes to a unsigned int (at least the last max len bytes) 

arguments:
```eval_rst
=================== ========== 
``const uint8_t *``  **data**  
``int``              **len**   
=================== ========== 
```
returns: `uint32_t`


#### char_to_long

```c
uint64_t char_to_long(const char *a, int l);
```

converts a character into a uint64_t 

arguments:
```eval_rst
================ ======= 
``const char *``  **a**  
``int``           **l**  
================ ======= 
```
returns: `uint64_t`


#### hexchar_to_int

```c
uint8_t hexchar_to_int(char c);
```

converts a hexchar to byte (4bit). 

In case of a nonhex char 0xff will be returned. 

arguments:
```eval_rst
======== ======= 
``char``  **c**  
======== ======= 
```
returns: `uint8_t`


#### hex_to_bytes

```c
int hex_to_bytes(const char *hexdata, int hexlen, uint8_t *out, int outlen);
```

convert a c hex string to a byte array storing it into an existing buffer. 

arguments:
```eval_rst
================ ============= 
``const char *``  **hexdata**  
``int``           **hexlen**   
``uint8_t *``     **out**      
``int``           **outlen**   
================ ============= 
```
returns: `int`


#### hex_to_new_bytes

```c
bytes_t* hex_to_new_bytes(const char *buf, int len);
```

convert a c string to a byte array creating a new buffer 

arguments:
```eval_rst
================ ========= 
``const char *``  **buf**  
``int``           **len**  
================ ========= 
```
returns: [`bytes_t *`](#bytes-t)


#### bytes_to_hex

```c
int bytes_to_hex(const uint8_t *buffer, int len, char *out);
```

convefrts a bytes into hex 

arguments:
```eval_rst
=================== ============ 
``const uint8_t *``  **buffer**  
``int``              **len**     
``char *``           **out**     
=================== ============ 
```
returns: `int`


#### keccak

```c
int keccak(bytes_t data, void *dst);
```

writes 32 bytes to the pointer. 

arguments:
```eval_rst
===================== ========== 
`bytes_t <#bytes-t>`_  **data**  
``void *``             **dst**   
===================== ========== 
```
returns: `int`


#### long_to_bytes

```c
void long_to_bytes(uint64_t val, uint8_t *dst);
```

converts a a uin64_t to 8 bytes written to dst using big endian 

arguments:
```eval_rst
============= ========= 
``uint64_t``   **val**  
``uint8_t *``  **dst**  
============= ========= 
```

#### int_to_bytes

```c
void int_to_bytes(uint32_t val, uint8_t *dst);
```

converts a unsigned int to 4 bytes written to dst using big endian 

arguments:
```eval_rst
============= ========= 
``uint32_t``   **val**  
``uint8_t *``  **dst**  
============= ========= 
```

#### _strdupn

```c
char* _strdupn(const char *src, int len);
```

duplicate the string. 

A len=-1 will determine the len with strlen. 

arguments:
```eval_rst
================ ========= 
``const char *``  **src**  
``int``           **len**  
================ ========= 
```
returns: `char *`


#### min_bytes_len

```c
int min_bytes_len(uint64_t val);
```

calculate the min number of byte to represents the len 

arguments:
```eval_rst
============ ========= 
``uint64_t``  **val**  
============ ========= 
```
returns: `int`


#### uint256_set

```c
void uint256_set(const uint8_t *src, wlen_t src_len, bytes32_t dst);
```

sets a variable value to 32byte word. 

arguments:
```eval_rst
========================= ============= 
``const uint8_t *``        **src**      
`wlen_t <#wlen-t>`_        **src_len**  
`bytes32_t <#bytes32-t>`_  **dst**      
========================= ============= 
```

#### str_replace

```c
char* str_replace(char *orig, const char *rep, const char *with);
```

replaces a string and returns a copy. 

arguments:
```eval_rst
================ ========== 
``char *``        **orig**  
``const char *``  **rep**   
``const char *``  **with**  
================ ========== 
```
returns: `char *`


#### str_replace_pos

```c
char* str_replace_pos(char *orig, size_t pos, size_t len, const char *rep);
```

replaces a string at the given position. 

arguments:
```eval_rst
================ ========== 
``char *``        **orig**  
``size_t``        **pos**   
``size_t``        **len**   
``const char *``  **rep**   
================ ========== 
```
returns: `char *`


#### str_find

```c
char* str_find(char *haystack, const char *needle);
```

lightweight strstr() replacements 

arguments:
```eval_rst
================ ============== 
``char *``        **haystack**  
``const char *``  **needle**    
================ ============== 
```
returns: `char *`


#### str_remove_html

```c
char* str_remove_html(char *data);
```

remove all html-tags in the text. 

This function will modify the orifinal data and return the same pointer as the input. 

arguments:
```eval_rst
========== ========== 
``char *``  **data**  
========== ========== 
```
returns: `char *`


#### current_ms

```c
uint64_t current_ms();
```

current timestamp in ms. 

returns: `uint64_t`


#### memiszero

```c
static bool memiszero(uint8_t *ptr, size_t l);
```

returns true if all pytes (specified by l) of pts have a value of zero. 

arguments:
```eval_rst
============= ========= 
``uint8_t *``  **ptr**  
``size_t``     **l**    
============= ========= 
```
returns: `bool`


#### in3_set_func_time

```c
void in3_set_func_time(time_func fn);
```

arguments:
```eval_rst
========================= ======== 
`time_func <#time-func>`_  **fn**  
========================= ======== 
```

#### in3_time

```c
uint64_t in3_time(void *t);
```

arguments:
```eval_rst
========== ======= 
``void *``  **t**  
========== ======= 
```
returns: `uint64_t`


#### in3_set_func_rand

```c
void in3_set_func_rand(rand_func fn);
```

arguments:
```eval_rst
========================= ======== 
`rand_func <#rand-func>`_  **fn**  
========================= ======== 
```

#### in3_rand

```c
int in3_rand(void *s);
```

arguments:
```eval_rst
========== ======= 
``void *``  **s**  
========== ======= 
```
returns: `int`


#### in3_set_func_srand

```c
void in3_set_func_srand(srand_func fn);
```

arguments:
```eval_rst
=========================== ======== 
`srand_func <#srand-func>`_  **fn**  
=========================== ======== 
```

#### in3_srand

```c
void in3_srand(unsigned int s);
```

arguments:
```eval_rst
================ ======= 
``unsigned int``  **s**  
================ ======= 
```

#### in3_sleep

```c
void in3_sleep(uint32_t ms);
```

arguments:
```eval_rst
============ ======== 
``uint32_t``  **ms**  
============ ======== 
```

#### parse_float_val

```c
int64_t parse_float_val(const char *data, int32_t expo);
```

parses a float-string and returns the value as int 

arguments:
```eval_rst
================ ========== ===============
``const char *``  **data**  the data string
``int32_t``       **expo**  the exponent
================ ========== ===============
```
returns: `int64_t`


#### b256_add

```c
void b256_add(bytes32_t a, uint8_t *b, wlen_t len_b);
```

simple add function, which adds the bytes (b) to a 

arguments:
```eval_rst
========================= =========== 
`bytes32_t <#bytes32-t>`_  **a**      
``uint8_t *``              **b**      
`wlen_t <#wlen-t>`_        **len_b**  
========================= =========== 
```

#### bytes_to_hex_string

```c
char* bytes_to_hex_string(char *out, const char *prefix, const bytes_t b, const char *postfix);
```

prints a bytes into a string 

arguments:
```eval_rst
=========================== ============= 
``char *``                   **out**      
``const char *``             **prefix**   
`bytes_tconst  <#bytes-t>`_  **b**        
``const char *``             **postfix**  
=========================== ============= 
```
returns: `char *`


## Module init 




### in3_init.h

IN3 init module for auto initializing verifiers and transport based on build config. 

File: [c/src/init/in3_init.h](https://github.com/slockit/in3-c/blob/master/c/src/init/in3_init.h)

#### in3_for_chain (chain_id)

```c
#define in3_for_chain (chain_id) in3_for_chain_auto_init(chain_id)
```


#### in3_init

```c
void in3_init();
```

Global initialization for the in3 lib. 

Note: This function is not MT-safe and is expected to be called early during during program startup (i.e. in main()) before other threads are spawned. 


#### in3_for_chain_auto_init

```c
in3_t* in3_for_chain_auto_init(chain_id_t chain_id);
```

Auto-init fallback for easy client initialization meant for single-threaded apps. 

This function automatically calls `in3_init()` before calling `in3_for_chain_default()`. To enable this feature, make sure you include this header file (i.e. `in3_init.h`) before `client.h`. Doing so will replace the call to `in3_for_chain()` with this function. 

arguments:
```eval_rst
=========================== ============== 
`chain_id_t <#chain-id-t>`_  **chain_id**  
=========================== ============== 
```
returns: [`in3_t *`](#in3-t)


## Module pay 




### pay_eth.h

USN API. 

This header-file defines easy to use function, which are verifying USN-Messages. 

File: [c/src/pay/eth/pay_eth.h](https://github.com/slockit/in3-c/blob/master/c/src/pay/eth/pay_eth.h)

#### in3_pay_eth_node_t


The stuct contains following fields:

```eval_rst
================================================= ============= 
`address_t <#address-t>`_                          **address**  
``uint32_t``                                       **price**    
``uint64_t``                                       **payed**    
`in3_pay_eth_nodestruct , * <#in3-pay-eth-node>`_  **next**     
================================================= ============= 
```

#### in3_pay_eth

```c
in3_ret_t in3_pay_eth(void *plugin_data, in3_plugin_act_t action, void *plugin_ctx);
```

Eth payment implementation. 

arguments:
```eval_rst
======================================= ================= 
``void *``                               **plugin_data**  
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**       
``void *``                               **plugin_ctx**   
======================================= ================= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_pay_eth_data

```c
static in3_pay_eth_t* in3_pay_eth_data(in3_t *c);
```

get access to internal plugin data if registered 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_pay_eth_t *`](#in3-pay-eth-t)


#### in3_register_pay_eth

```c
in3_ret_t in3_register_pay_eth(in3_t *c);
```

registers the Eth payment plugin 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### zksync.h

ZKSync API. 

This header-file registers zksync api functions. 

File: [c/src/pay/zksync/zksync.h](https://github.com/slockit/in3-c/blob/master/c/src/pay/zksync/zksync.h)

#### zk_msg_type

message type. 

The enum type contains the following values:

```eval_rst
================= = ===========
 **ZK_TRANSFER**  5 transfer tx
 **ZK_WITHDRAW**  3 withdraw tx
================= = ===========
```

#### zk_sign_type

signature-type which can be configured in the config 

The enum type contains the following values:

```eval_rst
====================== = ===============================
 **ZK_SIGN_PK**        1 sign with PK (default)
 **ZK_SIGN_CONTRACT**  2 use eip1271 contract signatures
 **ZK_SIGN_CREATE2**   3 use creat2 code
====================== = ===============================
```

#### zk_fee_t


```c
typedef uint64_t zk_fee_t
```

#### zk_fee_p_t


```c
typedef uint64_t zk_fee_p_t
```

#### zk_msg_type_t

message type. 


The enum type contains the following values:

```eval_rst
================= = ===========
 **ZK_TRANSFER**  5 transfer tx
 **ZK_WITHDRAW**  3 withdraw tx
================= = ===========
```

#### zk_sign_type_t

signature-type which can be configured in the config 


The enum type contains the following values:

```eval_rst
====================== = ===============================
 **ZK_SIGN_PK**        1 sign with PK (default)
 **ZK_SIGN_CONTRACT**  2 use eip1271 contract signatures
 **ZK_SIGN_CREATE2**   3 use creat2 code
====================== = ===============================
```

#### zk_create2_t

create2-arguments 


The stuct contains following fields:

```eval_rst
========================= ============== =========================
`address_t <#address-t>`_  **creator**   address of the creator
`bytes32_t <#bytes32-t>`_  **salt_arg**  saltarg
`bytes32_t <#bytes32-t>`_  **codehash**  hash of the deploy-txdata
========================= ============== =========================
```

#### zk_musig_session_t

a musig session to create a combined signature 


The stuct contains following fields:

```eval_rst
================================================= ====================== ========================================================
``uint64_t``                                       **id**                identifier of a current session
`bytes32_t <#bytes32-t>`_                          **seed**              a random seed used
`bytes_t <#bytes-t>`_                              **pub_keys**          the seesion id
``unsigned int``                                   **pos**               the position within the pub_keys
``unsigned int``                                   **len**               number of participants
`bytes_t <#bytes-t>`_                              **precommitments**    all precommits
`bytes_t <#bytes-t>`_                              **commitments**       all commits
`bytes_t <#bytes-t>`_                              **signature_shares**  all signatures shares
``void *``                                         **signer**            handle for the signer
``char *``                                         **proof_data**        the proof needed by the server to verify. 
                                                                         
                                                                         This is created by the client and checked in the server.
`zk_musig_sessionstruct , * <#zk-musig-session>`_  **next**              next session
================================================= ====================== ========================================================
```

#### zksync_config_t

internal configuration-object 


The stuct contains following fields:

```eval_rst
============================================= ========================= ================================================================================
``char *``                                     **provider_url**         url of the zksync-server
``char *``                                     **rest_api**             url of the zksync-rest-api
``uint8_t *``                                  **account**              address of the account
``uint8_t *``                                  **main_contract**        address of the main zksync contract
``uint8_t *``                                  **gov_contract**         address of the government contract
``uint64_t``                                   **account_id**           the id of the account as used in the messages
``uint64_t``                                   **nonce**                the current nonce
`address_t <#address-t>`_                      **pub_key_hash_set**     the pub_key_hash as set in the account
`address_t <#address-t>`_                      **pub_key_hash_pk**      the pub_key_hash based on the sync_key
`bytes32_t <#bytes32-t>`_                      **pub_key**              the pub_key
``uint16_t``                                   **token_len**            number of tokens in the tokenlist
`bytes32_t <#bytes32-t>`_                      **sync_key**             the raw key to sign with
`zksync_token_t * <#zksync-token-t>`_          **tokens**               the token-list
`zk_sign_type_t <#zk-sign-type-t>`_            **sign_type**            the signature-type to use
``uint32_t``                                   **version**              zksync version
`zk_create2_t <#zk-create2-t>`_                **create2**              create2 args
`bytes_t <#bytes-t>`_                          **musig_pub_keys**       the public keys of all participants of a schnorr musig signature
`zk_musig_session_t * <#zk-musig-session-t>`_  **musig_sessions**       linked list of open musig sessions
``char **``                                    **musig_urls**           urls to get signatureshares, the order must be in the same order as the pub_keys
`pay_criteriastruct , * <#pay-criteria>`_      **incentive**            incentive payment configuration
``char *``                                     **proof_verify_method**  the rpc-method used to verify the proof before creating a signature
``char *``                                     **proof_create_method**  the rpc-method used to create the proof before creating a signature
============================================= ========================= ================================================================================
```

#### zksync_valid_t


The stuct contains following fields:

```eval_rst
============ ========== 
``uint64_t``  **from**  
``uint64_t``  **to**    
============ ========== 
```

#### pay_criteria_t


The stuct contains following fields:

```eval_rst
===================================== ================================ =========================================================
``uint_fast32_t``                      **payed_nodes**                 max number of nodes payed at the same time
``uint64_t``                           **max_price_per_hundred_igas**  the max price per 100 gas units to accept a payment offer
``char *``                             **token**                       token-name
`zksync_config_t <#zksync-config-t>`_  **config**                      the account configuration
===================================== ================================ =========================================================
```

#### in3_register_zksync

```c
NONULL in3_ret_t in3_register_zksync(in3_t *c);
```

registers the zksync-plugin in the client 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_set_key

```c
NONULL in3_ret_t zksync_set_key(zksync_config_t *conf, in3_rpc_handle_ctx_t *ctx, bool only_update);
```

sets a PubKeyHash for the current Account 

arguments:
```eval_rst
================================================= ================= 
`zksync_config_t * <#zksync-config-t>`_            **conf**         
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **ctx**          
``bool``                                           **only_update**  
================================================= ================= 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_transfer

```c
NONULL in3_ret_t zksync_transfer(zksync_config_t *conf, in3_rpc_handle_ctx_t *ctx, zk_msg_type_t type);
```

sends a transfer transaction in Layer 2 

arguments:
```eval_rst
================================================= ========== 
`zksync_config_t * <#zksync-config-t>`_            **conf**  
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **ctx**   
`zk_msg_type_t <#zk-msg-type-t>`_                  **type**  
================================================= ========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_deposit

```c
NONULL in3_ret_t zksync_deposit(zksync_config_t *conf, in3_rpc_handle_ctx_t *ctx);
```

sends a deposit transaction in Layer 1 

arguments:
```eval_rst
================================================= ========== 
`zksync_config_t * <#zksync-config-t>`_            **conf**  
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **ctx**   
================================================= ========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_emergency_withdraw

```c
NONULL in3_ret_t zksync_emergency_withdraw(zksync_config_t *conf, in3_rpc_handle_ctx_t *ctx);
```

sends a emergency withdraw transaction in Layer 1 

arguments:
```eval_rst
================================================= ========== 
`zksync_config_t * <#zksync-config-t>`_            **conf**  
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **ctx**   
================================================= ========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_sign_transfer

```c
NONULL in3_ret_t zksync_sign_transfer(sb_t *sb, zksync_tx_data_t *data, in3_req_t *req, zksync_config_t *conf);
```

creates message data and signs a transfer-message 

arguments:
```eval_rst
========================================= ========== 
`sb_t * <#sb-t>`_                          **sb**    
`zksync_tx_data_t * <#zksync-tx-data-t>`_  **data**  
`in3_req_t * <#in3-req-t>`_                **req**   
`zksync_config_t * <#zksync-config-t>`_    **conf**  
========================================= ========== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_sign_change_pub_key

```c
NONULL in3_ret_t zksync_sign_change_pub_key(sb_t *sb, in3_req_t *req, uint8_t *sync_pub_key, uint32_t nonce, zksync_config_t *conf, zk_fee_t fee, zksync_token_t *token, zksync_valid_t valid);
```

creates message data and signs a change_pub_key-message 

arguments:
```eval_rst
======================================= ================== 
`sb_t * <#sb-t>`_                        **sb**            
`in3_req_t * <#in3-req-t>`_              **req**           
``uint8_t *``                            **sync_pub_key**  
``uint32_t``                             **nonce**         
`zksync_config_t * <#zksync-config-t>`_  **conf**          
``zk_fee_t``                             **fee**           
`zksync_token_t * <#zksync-token-t>`_    **token**         
`zksync_valid_t <#zksync-valid-t>`_      **valid**         
======================================= ================== 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_musig_sign

```c
in3_ret_t zksync_musig_sign(zksync_config_t *conf, in3_rpc_handle_ctx_t *ctx);
```

arguments:
```eval_rst
================================================= ========== 
`zksync_config_t * <#zksync-config-t>`_            **conf**  
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **ctx**   
================================================= ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zk_musig_session_free

```c
zk_musig_session_t* zk_musig_session_free(zk_musig_session_t *s);
```

arguments:
```eval_rst
============================================= ======= 
`zk_musig_session_t * <#zk-musig-session-t>`_  **s**  
============================================= ======= 
```
returns: [`zk_musig_session_t *`](#zk-musig-session-t)


#### zksync_sign

```c
in3_ret_t zksync_sign(zksync_config_t *conf, bytes_t msg, in3_req_t *req, uint8_t *sig);
```

arguments:
```eval_rst
======================================= ========== 
`zksync_config_t * <#zksync-config-t>`_  **conf**  
`bytes_t <#bytes-t>`_                    **msg**   
`in3_req_t * <#in3-req-t>`_              **req**   
``uint8_t *``                            **sig**   
======================================= ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_check_payment

```c
in3_ret_t zksync_check_payment(zksync_config_t *conf, in3_pay_followup_ctx_t *ctx);
```

arguments:
```eval_rst
===================================================== ========== 
`zksync_config_t * <#zksync-config-t>`_                **conf**  
`in3_pay_followup_ctx_t * <#in3-pay-followup-ctx-t>`_  **ctx**   
===================================================== ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_add_payload

```c
in3_ret_t zksync_add_payload(in3_pay_payload_ctx_t *ctx);
```

arguments:
```eval_rst
=================================================== ========= 
`in3_pay_payload_ctx_t * <#in3-pay-payload-ctx-t>`_  **ctx**  
=================================================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### update_nodelist_from_cache

```c
in3_ret_t update_nodelist_from_cache(in3_req_t *req, unsigned int nodelen);
```

arguments:
```eval_rst
=========================== ============= 
`in3_req_t * <#in3-req-t>`_  **req**      
``unsigned int``             **nodelen**  
=========================== ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### handle_zksync

```c
in3_ret_t handle_zksync(void *conf, in3_plugin_act_t action, void *arg);
```

arguments:
```eval_rst
======================================= ============ 
``void *``                               **conf**    
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**  
``void *``                               **arg**     
======================================= ============ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_tx_data

```c
in3_ret_t zksync_tx_data(zksync_config_t *conf, in3_rpc_handle_ctx_t *ctx);
```

arguments:
```eval_rst
================================================= ========== 
`zksync_config_t * <#zksync-config-t>`_            **conf**  
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **ctx**   
================================================= ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_account_history

```c
in3_ret_t zksync_account_history(zksync_config_t *conf, in3_rpc_handle_ctx_t *ctx);
```

arguments:
```eval_rst
================================================= ========== 
`zksync_config_t * <#zksync-config-t>`_            **conf**  
`in3_rpc_handle_ctx_t * <#in3-rpc-handle-ctx-t>`_  **ctx**   
================================================= ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### zksync_get_conf

```c
NONULL zksync_config_t* zksync_get_conf(in3_req_t *req);
```

arguments:
```eval_rst
=========================== ========= 
`in3_req_t * <#in3-req-t>`_  **req**  
=========================== ========= 
```
returns: [`zksync_config_tNONULL , *`](#zksync-config-t)


## Module signer 




### ethereum_apdu_client.h

this file defines the incubed configuration struct and it registration. 

File: [c/src/signer/ledger-nano/signer/ethereum_apdu_client.h](https://github.com/slockit/in3-c/blob/master/c/src/signer/ledger-nano/signer/ethereum_apdu_client.h)

#### eth_ledger_set_signer_txn

```c
in3_ret_t eth_ledger_set_signer_txn(in3_t *in3, uint8_t *bip_path);
```

attaches ledger nano hardware wallet signer with incubed . 

bip32 path to be given to point the specific public/private key in HD tree for Ethereum! 

arguments:
```eval_rst
=================== ============== 
`in3_t * <#in3-t>`_  **in3**       
``uint8_t *``        **bip_path**  
=================== ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_ledger_get_public_addr

```c
in3_ret_t eth_ledger_get_public_addr(uint8_t *i_bip_path, uint8_t *o_public_key);
```

returns public key at the bip_path . 

returns IN3_ENODEVICE error if ledger nano device is not connected 

arguments:
```eval_rst
============= ================== 
``uint8_t *``  **i_bip_path**    
``uint8_t *``  **o_public_key**  
============= ================== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### ledger_signer.h

this file defines the incubed configuration struct and it registration. 

File: [c/src/signer/ledger-nano/signer/ledger_signer.h](https://github.com/slockit/in3-c/blob/master/c/src/signer/ledger-nano/signer/ledger_signer.h)

#### eth_ledger_set_signer

```c
in3_ret_t eth_ledger_set_signer(in3_t *in3, uint8_t *bip_path);
```

attaches ledger nano hardware wallet signer with incubed . 

bip32 path to be given to point the specific public/private key in HD tree for Ethereum! 

arguments:
```eval_rst
=================== ============== 
`in3_t * <#in3-t>`_  **in3**       
``uint8_t *``        **bip_path**  
=================== ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_ledger_get_public_key

```c
in3_ret_t eth_ledger_get_public_key(uint8_t *bip_path, uint8_t *public_key);
```

returns public key at the bip_path . 

returns IN3_ENODEVICE error if ledger nano device is not connected 

arguments:
```eval_rst
============= ================ 
``uint8_t *``  **bip_path**    
``uint8_t *``  **public_key**  
============= ================ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### signer.h

Ethereum Nano verification. 

File: [c/src/signer/pk-signer/signer.h](https://github.com/slockit/in3-c/blob/master/c/src/signer/pk-signer/signer.h)

#### hasher_t

The enum type contains the following values:

```eval_rst
================================ == 
 **hasher_sha2**                 0  
 **hasher_sha2d**                1  
 **hasher_sha2_ripemd**          2  
 **hasher_sha3**                 3  
 **hasher_sha3k**                4  
 **hasher_blake**                5  
 **hasher_blaked**               6  
 **hasher_blake_ripemd**         7  
 **hasher_groestld_trunc**       8  
 **hasher_overwinter_prevouts**  9  
 **hasher_overwinter_sequence**  10 
 **hasher_overwinter_outputs**   11 
 **hasher_overwinter_preimage**  12 
 **hasher_sapling_preimage**     13 
================================ == 
```

#### eth_set_pk_signer

```c
in3_ret_t eth_set_pk_signer(in3_t *in3, bytes32_t pk);
```

simply signer with one private key. 

since the pk pointting to the 32 byte private key is not cloned, please make sure, you manage memory allocation correctly!

simply signer with one private key. 

arguments:
```eval_rst
========================= ========= 
`in3_t * <#in3-t>`_        **in3**  
`bytes32_t <#bytes32-t>`_  **pk**   
========================= ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_register_pk_signer

```c
in3_ret_t eth_register_pk_signer(in3_t *in3);
```

registers pk signer as plugin so you can use config or in3_addKeys as rpc 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_set_request_signer

```c
in3_ret_t eth_set_request_signer(in3_t *in3, bytes32_t pk);
```

sets the signer and a pk to the client 

arguments:
```eval_rst
========================= ========= 
`in3_t * <#in3-t>`_        **in3**  
`bytes32_t <#bytes32-t>`_  **pk**   
========================= ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_set_pk_signer_hex

```c
void eth_set_pk_signer_hex(in3_t *in3, char *key);
```

simply signer with one private key as hex. 

simply signer with one private key as hex. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
``char *``           **key**  
=================== ========= 
```

#### ec_sign_pk_hash

```c
in3_ret_t ec_sign_pk_hash(uint8_t *message, size_t len, uint8_t *pk, hasher_t hasher, uint8_t *dst);
```

Signs message after hashing it with hasher function given in 'hasher_t', with the given private key. 

Signs message after hashing it with hasher function given in 'hasher_t', with the given private key. 

arguments:
```eval_rst
============= ============= 
``uint8_t *``  **message**  
``size_t``     **len**      
``uint8_t *``  **pk**       
``hasher_t``   **hasher**   
``uint8_t *``  **dst**      
============= ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### ec_sign_pk_raw

```c
in3_ret_t ec_sign_pk_raw(uint8_t *message, uint8_t *pk, uint8_t *dst);
```

Signs message raw with the given private key. 

Signs message raw with the given private key. 

arguments:
```eval_rst
============= ============= 
``uint8_t *``  **message**  
``uint8_t *``  **pk**       
``uint8_t *``  **dst**      
============= ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_create_prefixed_msg_hash

```c
void eth_create_prefixed_msg_hash(bytes32_t dst, bytes_t msg);
```

hashes the msg by adding the Ethereum Signed Message-Prefix 

arguments:
```eval_rst
========================= ========= 
`bytes32_t <#bytes32-t>`_  **dst**  
`bytes_t <#bytes-t>`_      **msg**  
========================= ========= 
```

#### sign_with_pk

```c
bytes_t sign_with_pk(const bytes32_t pk, const bytes_t data, const d_signature_type_t type);
```

signs with a pk bases on the type 

arguments:
```eval_rst
================================================= ========== 
`bytes32_tconst  <#bytes32-t>`_                    **pk**    
`bytes_tconst  <#bytes-t>`_                        **data**  
`d_signature_type_tconst  <#d-signature-type-t>`_  **type**  
================================================= ========== 
```
returns: [`bytes_t`](#bytes-t)


## Module transport 




### in3_curl.h

transport-handler using libcurl. 

File: [c/src/transport/curl/in3_curl.h](https://github.com/slockit/in3-c/blob/master/c/src/transport/curl/in3_curl.h)

#### send_curl

```c
in3_ret_t send_curl(void *plugin_data, in3_plugin_act_t action, void *plugin_ctx);
```

a transport function using curl. 

arguments:
```eval_rst
======================================= ================= 
``void *``                               **plugin_data**  
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**       
``void *``                               **plugin_ctx**   
======================================= ================= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_curl

```c
in3_ret_t in3_register_curl(in3_t *c);
```

registers curl as a default transport. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### in3_http.h

transport-handler using simple http. 

File: [c/src/transport/http/in3_http.h](https://github.com/slockit/in3-c/blob/master/c/src/transport/http/in3_http.h)

#### send_http

```c
in3_ret_t send_http(void *plugin_data, in3_plugin_act_t action, void *plugin_ctx);
```

a very simple transport function, which allows to send http-requests without a dependency to curl. 

Here each request will be transformed to http instead of https.

You can use it by setting the transport-function-pointer in the in3_t->transport to this function:

```c
#include <in3/in3_http.h>
...
c->transport = send_http;
```
arguments:
```eval_rst
======================================= ================= 
``void *``                               **plugin_data**  
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**       
``void *``                               **plugin_ctx**   
======================================= ================= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_http

```c
in3_ret_t in3_register_http(in3_t *c);
```

registers http as a default transport. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### in3_winhttp.h

transport-handler using simple http. 

File: [c/src/transport/winhttp/in3_winhttp.h](https://github.com/slockit/in3-c/blob/master/c/src/transport/winhttp/in3_winhttp.h)

#### send_winhttp

```c
in3_ret_t send_winhttp(void *plugin_data, in3_plugin_act_t action, void *plugin_ctx);
```

arguments:
```eval_rst
======================================= ================= 
``void *``                               **plugin_data**  
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**       
``void *``                               **plugin_ctx**   
======================================= ================= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_winhttp

```c
in3_ret_t in3_register_winhttp(in3_t *c);
```

registers http as a default transport. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


## Module verifier 




### btc.h

Bitcoin verification. 

File: [c/src/verifier/btc/btc.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/btc/btc.h)

#### in3_register_btc

```c
in3_ret_t in3_register_btc(in3_t *c);
```

this function should only be called once and will register the bitcoin verifier. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### eth_basic.h

Ethereum Nanon verification. 

File: [c/src/verifier/eth1/basic/eth_basic.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/basic/eth_basic.h)

#### in3_filter_type_t

Filter type used internally when managing filters. 

The enum type contains the following values:

```eval_rst
==================== = ============================
 **FILTER_EVENT**    0 Event filter.
 **FILTER_BLOCK**    1 Block filter.
 **FILTER_PENDING**  2 Pending filter (Unsupported)
==================== = ============================
```

#### in3_filter_t


The stuct contains following fields:

```eval_rst
========================================= ==================== ==========================================================
``bool``                                   **is_first_usage**  if true the filter was not used previously
`in3_filter_type_t <#in3-filter-type-t>`_  **type**            filter type: (event, block or pending)
``uint64_t``                               **last_block**      block no. 
                                                               
                                                               when filter was created OR eth_getFilterChanges was called
``char *``                                 **options**         associated filter options
``void(*``                                 **release**         method to release owned resources
========================================= ==================== ==========================================================
```

#### in3_filter_handler_t

Handler which is added to client config in order to handle filter. 


The stuct contains following fields:

```eval_rst
================================== =========== ================
`in3_filter_t ** <#in3-filter-t>`_  **array**  
``size_t``                          **count**  array of filters
================================== =========== ================
```

#### eth_basic_get_filters

```c
in3_filter_handler_t* eth_basic_get_filters(in3_t *c);
```

returns the filters 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_filter_handler_t *`](#in3-filter-handler-t)


#### eth_verify_tx_values

```c
in3_ret_t eth_verify_tx_values(in3_vctx_t *vc, d_token_t *tx, bytes_t *raw);
```

verifies internal tx-values. 

arguments:
```eval_rst
============================= ========= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**   
`d_token_t * <#d-token-t>`_    **tx**   
`bytes_t * <#bytes-t>`_        **raw**  
============================= ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_eth_getTransaction

```c
in3_ret_t eth_verify_eth_getTransaction(in3_vctx_t *vc, bytes_t *tx_hash);
```

verifies a transaction. 

arguments:
```eval_rst
============================= ============= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**       
`bytes_t * <#bytes-t>`_        **tx_hash**  
============================= ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_eth_getTransactionByBlock

```c
in3_ret_t eth_verify_eth_getTransactionByBlock(in3_vctx_t *vc, d_token_t *blk, uint32_t tx_idx);
```

verifies a transaction by block hash/number and id. 

arguments:
```eval_rst
============================= ============ 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**      
`d_token_t * <#d-token-t>`_    **blk**     
``uint32_t``                   **tx_idx**  
============================= ============ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_account_proof

```c
in3_ret_t eth_verify_account_proof(in3_vctx_t *vc);
```

verify account-proofs 

arguments:
```eval_rst
============================= ======== 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**  
============================= ======== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_eth_getBlock

```c
in3_ret_t eth_verify_eth_getBlock(in3_vctx_t *vc, bytes_t *block_hash, uint64_t blockNumber);
```

verifies a block 

arguments:
```eval_rst
============================= ================= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**           
`bytes_t * <#bytes-t>`_        **block_hash**   
``uint64_t``                   **blockNumber**  
============================= ================= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_eth_getBlockTransactionCount

```c
in3_ret_t eth_verify_eth_getBlockTransactionCount(in3_vctx_t *vc, bytes_t *block_hash, uint64_t blockNumber);
```

verifies block transaction count by number or hash 

arguments:
```eval_rst
============================= ================= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**           
`bytes_t * <#bytes-t>`_        **block_hash**   
``uint64_t``                   **blockNumber**  
============================= ================= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_eth_basic

```c
in3_ret_t in3_register_eth_basic(in3_t *c);
```

this function should only be called once and will register the eth-nano verifier. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_eth_getLog

```c
in3_ret_t eth_verify_eth_getLog(in3_vctx_t *vc, int l_logs);
```

verify logs 

arguments:
```eval_rst
============================= ============ 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**      
``int``                        **l_logs**  
============================= ============ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_prepare_unsigned_tx

```c
in3_ret_t eth_prepare_unsigned_tx(d_token_t *tx, in3_req_t *req, bytes_t *dst, sb_t *meta);
```

prepares a transaction and writes the data to the dst-bytes. 

In case of success, you MUST free only the data-pointer of the dst. 

arguments:
```eval_rst
=========================== ========== =================================================================================
`d_token_t * <#d-token-t>`_  **tx**    a json-token desribing the transaction
`in3_req_t * <#in3-req-t>`_  **req**   the current context
`bytes_t * <#bytes-t>`_      **dst**   the bytes to write the result to.
`sb_t * <#sb-t>`_            **meta**  a stringbuilder in order write the wallet_state and metadata depending on the tx.
=========================== ========== =================================================================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_sign_raw_tx

```c
in3_ret_t eth_sign_raw_tx(bytes_t raw_tx, in3_req_t *req, address_t from, bytes_t *dst);
```

signs a unsigned raw transaction and writes the raw data to the dst-bytes. 

In case of success, you MUST free only the data-pointer of the dst. 

arguments:
```eval_rst
=========================== ============ =======================================
`bytes_t <#bytes-t>`_        **raw_tx**  the unsigned raw transaction to sign
`in3_req_t * <#in3-req-t>`_  **req**     the current context
`address_t <#address-t>`_    **from**    the address of the account to sign with
`bytes_t * <#bytes-t>`_      **dst**     the bytes to write the result to.
=========================== ============ =======================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### handle_eth_sendTransaction

```c
in3_ret_t handle_eth_sendTransaction(in3_req_t *req, d_token_t *req_data);
```

expects a req-object for a transaction and converts it into a sendRawTransaction after signing. 

expects a req-object for a transaction and converts it into a sendRawTransaction after signing. 

arguments:
```eval_rst
=========================== ============== ===================
`in3_req_t * <#in3-req-t>`_  **req**       the current context
`d_token_t * <#d-token-t>`_  **req_data**  the request
=========================== ============== ===================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### empty_hash

```c
const uint8_t* empty_hash();
```

returns a pointer to 32 bytes marking a empty hash (keccakc(0x)) 

returns: `const uint8_t *`


#### eth_wallet_sign

```c
RETURNS_NONULL NONULL char* eth_wallet_sign(const char *key, const char *data);
```

minimum signer for the wallet, returns the signed message which needs to be freed 

arguments:
```eval_rst
================ ========== 
``const char *``  **key**   
``const char *``  **data**  
================ ========== 
```
returns: `RETURNS_NONULL NONULL char *`


#### get_from_address

```c
in3_ret_t get_from_address(d_token_t *tx, in3_req_t *ctx, address_t res);
```

determines the from-address in case no from-address has been specified. 

determines the from-address in case no from-address has been specified. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **tx**   
`in3_req_t * <#in3-req-t>`_  **ctx**  
`address_t <#address-t>`_    **res**  
=========================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### trie.h

Patricia Merkle Tree Imnpl. 

File: [c/src/verifier/eth1/basic/trie.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/basic/trie.h)

#### trie_node_type_t

Node types. 

The enum type contains the following values:

```eval_rst
================= = ============================
 **NODE_EMPTY**   0 empty node
 **NODE_BRANCH**  1 a Branch
 **NODE_LEAF**    2 a leaf containing the value.
 **NODE_EXT**     3 a extension
================= = ============================
```

#### in3_hasher_t

hash-function 


```c
typedef void(* in3_hasher_t) (bytes_t *src, uint8_t *dst)
```


#### in3_codec_add_t

codec to organize the encoding of the nodes 


```c
typedef void(* in3_codec_add_t) (bytes_builder_t *bb, bytes_t *val)
```


#### in3_codec_finish_t


```c
typedef void(* in3_codec_finish_t) (bytes_builder_t *bb, bytes_t *dst)
```


#### in3_codec_decode_size_t


```c
typedef int(* in3_codec_decode_size_t) (bytes_t *src)
```

returns: `int(*`


#### in3_codec_decode_index_t


```c
typedef int(* in3_codec_decode_index_t) (bytes_t *src, int index, bytes_t *dst)
```

returns: `int(*`


#### trie_node_t

single node in the merkle trie. 


The stuct contains following fields:

```eval_rst
======================================= ================ ===============================================
``uint8_t``                              **hash**        the hash of the node
`bytes_t <#bytes-t>`_                    **data**        the raw data
`bytes_t <#bytes-t>`_                    **items**       the data as list
``uint8_t``                              **own_memory**  if true this is a embedded node with own memory
`trie_node_type_t <#trie-node-type-t>`_  **type**        type of the node
`trie_nodestruct , * <#trie-node>`_      **next**        used as linked list
======================================= ================ ===============================================
```

#### trie_codec_t

the codec used to encode nodes. 


The stuct contains following fields:

```eval_rst
===================================== =================== 
`in3_codec_add_t <#in3-codec-add-t>`_  **encode_add**     
``in3_codec_finish_t``                 **encode_finish**  
``in3_codec_decode_size_t``            **decode_size**    
``in3_codec_decode_index_t``           **decode_item**    
===================================== =================== 
```

#### trie_t

a merkle trie implementation. 

This is a Patricia Merkle Tree. 


The stuct contains following fields:

```eval_rst
================================= ============ ==============================
`in3_hasher_t <#in3-hasher-t>`_    **hasher**  hash-function.
`trie_codec_t * <#trie-codec-t>`_  **codec**   encoding of the nocds.
`bytes32_t <#bytes32-t>`_          **root**    The root-hash.
`trie_node_t * <#trie-node-t>`_    **nodes**   linked list of containes nodes
================================= ============ ==============================
```

#### trie_new

```c
trie_t* trie_new();
```

creates a new Merkle Trie. 

returns: [`trie_t *`](#trie-t)


#### trie_free

```c
void trie_free(trie_t *val);
```

frees all resources of the trie. 

arguments:
```eval_rst
===================== ========= 
`trie_t * <#trie-t>`_  **val**  
===================== ========= 
```

#### trie_set_value

```c
void trie_set_value(trie_t *t, bytes_t *key, bytes_t *value);
```

sets a value in the trie. 

The root-hash will be updated automaticly. 

arguments:
```eval_rst
======================= =========== 
`trie_t * <#trie-t>`_    **t**      
`bytes_t * <#bytes-t>`_  **key**    
`bytes_t * <#bytes-t>`_  **value**  
======================= =========== 
```

### big.h

Ethereum Nanon verification. 

File: [c/src/verifier/eth1/evm/big.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/evm/big.h)

#### big_is_zero

```c
uint8_t big_is_zero(uint8_t *data, wlen_t l);
```

arguments:
```eval_rst
=================== ========== 
``uint8_t *``        **data**  
`wlen_t <#wlen-t>`_  **l**     
=================== ========== 
```
returns: `uint8_t`


#### big_shift_left

```c
void big_shift_left(uint8_t *a, wlen_t len, int bits);
```

arguments:
```eval_rst
=================== ========== 
``uint8_t *``        **a**     
`wlen_t <#wlen-t>`_  **len**   
``int``              **bits**  
=================== ========== 
```

#### big_shift_right

```c
void big_shift_right(uint8_t *a, wlen_t len, int bits);
```

arguments:
```eval_rst
=================== ========== 
``uint8_t *``        **a**     
`wlen_t <#wlen-t>`_  **len**   
``int``              **bits**  
=================== ========== 
```

#### big_cmp

```c
int big_cmp(const uint8_t *a, const wlen_t len_a, const uint8_t *b, const wlen_t len_b);
```

arguments:
```eval_rst
========================= =========== 
``const uint8_t *``        **a**      
`wlen_tconst  <#wlen-t>`_  **len_a**  
``const uint8_t *``        **b**      
`wlen_tconst  <#wlen-t>`_  **len_b**  
========================= =========== 
```
returns: `int`


#### big_signed

```c
int big_signed(uint8_t *val, wlen_t len, uint8_t *dst);
```

returns 0 if the value is positive or 1 if negavtive. 

in this case the absolute value is copied to dst. 

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **val**  
`wlen_t <#wlen-t>`_  **len**  
``uint8_t *``        **dst**  
=================== ========= 
```
returns: `int`


#### big_int

```c
int32_t big_int(uint8_t *val, wlen_t len);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **val**  
`wlen_t <#wlen-t>`_  **len**  
=================== ========= 
```
returns: `int32_t`


#### big_add

```c
int big_add(uint8_t *a, wlen_t len_a, uint8_t *b, wlen_t len_b, uint8_t *out, wlen_t max);
```

arguments:
```eval_rst
=================== =========== 
``uint8_t *``        **a**      
`wlen_t <#wlen-t>`_  **len_a**  
``uint8_t *``        **b**      
`wlen_t <#wlen-t>`_  **len_b**  
``uint8_t *``        **out**    
`wlen_t <#wlen-t>`_  **max**    
=================== =========== 
```
returns: `int`


#### big_sub

```c
int big_sub(uint8_t *a, wlen_t len_a, uint8_t *b, wlen_t len_b, uint8_t *out);
```

arguments:
```eval_rst
=================== =========== 
``uint8_t *``        **a**      
`wlen_t <#wlen-t>`_  **len_a**  
``uint8_t *``        **b**      
`wlen_t <#wlen-t>`_  **len_b**  
``uint8_t *``        **out**    
=================== =========== 
```
returns: `int`


#### big_mul

```c
int big_mul(uint8_t *a, wlen_t la, uint8_t *b, wlen_t lb, uint8_t *res, wlen_t max);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **la**   
``uint8_t *``        **b**    
`wlen_t <#wlen-t>`_  **lb**   
``uint8_t *``        **res**  
`wlen_t <#wlen-t>`_  **max**  
=================== ========= 
```
returns: `int`


#### big_div

```c
int big_div(uint8_t *a, wlen_t la, uint8_t *b, wlen_t lb, wlen_t sig, uint8_t *res);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **la**   
``uint8_t *``        **b**    
`wlen_t <#wlen-t>`_  **lb**   
`wlen_t <#wlen-t>`_  **sig**  
``uint8_t *``        **res**  
=================== ========= 
```
returns: `int`


#### big_mod

```c
int big_mod(uint8_t *a, wlen_t la, uint8_t *b, wlen_t lb, wlen_t sig, uint8_t *res);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **la**   
``uint8_t *``        **b**    
`wlen_t <#wlen-t>`_  **lb**   
`wlen_t <#wlen-t>`_  **sig**  
``uint8_t *``        **res**  
=================== ========= 
```
returns: `int`


#### big_exp

```c
int big_exp(uint8_t *a, wlen_t la, uint8_t *b, wlen_t lb, uint8_t *res);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **la**   
``uint8_t *``        **b**    
`wlen_t <#wlen-t>`_  **lb**   
``uint8_t *``        **res**  
=================== ========= 
```
returns: `int`


#### big_log256

```c
int big_log256(uint8_t *a, wlen_t len);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **len**  
=================== ========= 
```
returns: `int`


### code.h

code cache. 

File: [c/src/verifier/eth1/evm/code.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/evm/code.h)

#### in3_get_code

```c
in3_ret_t in3_get_code(in3_vctx_t *vc, address_t address, cache_entry_t **target);
```

fetches the code and adds it to the context-cache as cache_entry. 

So calling this function a second time will take the result from cache. 

arguments:
```eval_rst
==================================== ============= 
`in3_vctx_t * <#in3-vctx-t>`_         **vc**       
`address_t <#address-t>`_             **address**  
`cache_entry_t ** <#cache-entry-t>`_  **target**   
==================================== ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### evm.h

main evm-file. 

File: [c/src/verifier/eth1/evm/evm.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/evm/evm.h)

#### gas_options


#### EVM_ERROR_EMPTY_STACK

the no more elements on the stack 


 

```c
#define EVM_ERROR_EMPTY_STACK -20
```


#### EVM_ERROR_INVALID_OPCODE

the opcode is not supported 


 

```c
#define EVM_ERROR_INVALID_OPCODE -21
```


#### EVM_ERROR_BUFFER_TOO_SMALL

reading data from a position, which is not initialized 


 

```c
#define EVM_ERROR_BUFFER_TOO_SMALL -22
```


#### EVM_ERROR_ILLEGAL_MEMORY_ACCESS

the memory-offset does not exist 


 

```c
#define EVM_ERROR_ILLEGAL_MEMORY_ACCESS -23
```


#### EVM_ERROR_INVALID_JUMPDEST

the jump destination is not marked as valid destination 


 

```c
#define EVM_ERROR_INVALID_JUMPDEST -24
```


#### EVM_ERROR_INVALID_PUSH

the push data is empy 

```c
#define EVM_ERROR_INVALID_PUSH -25
```


#### EVM_ERROR_UNSUPPORTED_CALL_OPCODE

error handling the call, usually because static-calls are not allowed to change state 


 

```c
#define EVM_ERROR_UNSUPPORTED_CALL_OPCODE -26
```


#### EVM_ERROR_TIMEOUT

the evm ran into a loop 


 

```c
#define EVM_ERROR_TIMEOUT -27
```


#### EVM_ERROR_INVALID_ENV

the enviroment could not deliver the data 


 

```c
#define EVM_ERROR_INVALID_ENV -28
```


#### EVM_ERROR_OUT_OF_GAS

not enough gas to exewcute the opcode 


 

```c
#define EVM_ERROR_OUT_OF_GAS -29
```


#### EVM_ERROR_BALANCE_TOO_LOW

not enough funds to transfer the requested value. 

```c
#define EVM_ERROR_BALANCE_TOO_LOW -30
```


#### EVM_ERROR_STACK_LIMIT

stack limit reached 


 

```c
#define EVM_ERROR_STACK_LIMIT -31
```


#### EVM_ERROR_SUCCESS_CONSUME_GAS

write success but consume all gas 

```c
#define EVM_ERROR_SUCCESS_CONSUME_GAS -32
```


#### EVM_ERROR_MAX_CODE_SIZE_EXCEEDED

tried to create a contract with code bigger than the maximum size limit 

```c
#define EVM_ERROR_MAX_CODE_SIZE_EXCEEDED -33
```


#### EVM_PROP_FRONTIER

```c
#define EVM_PROP_FRONTIER 1
```


#### EVM_PROP_EIP150

```c
#define EVM_PROP_EIP150 2
```


#### EVM_PROP_EIP158

```c
#define EVM_PROP_EIP158 4
```


#### EVM_PROP_CONSTANTINOPL

```c
#define EVM_PROP_CONSTANTINOPL 16
```


#### EVM_PROP_ISTANBUL

```c
#define EVM_PROP_ISTANBUL 32
```


#### EVM_PROP_NO_FINALIZE

```c
#define EVM_PROP_NO_FINALIZE 32768
```


#### EVM_PROP_STATIC

```c
#define EVM_PROP_STATIC 256
```


#### EVM_PROP_TXCREATE

executing a creation transaction > 

```c
#define EVM_PROP_TXCREATE 512
```


#### EVM_PROP_CALL_DEPEND_ON_REFUND

executing code that depends on subcall gas refund to succeed > 

```c
#define EVM_PROP_CALL_DEPEND_ON_REFUND 1024
```


#### EVM_ENV_BALANCE

```c
#define EVM_ENV_BALANCE 1
```


#### EVM_ENV_CODE_SIZE

```c
#define EVM_ENV_CODE_SIZE 2
```


#### EVM_ENV_CODE_COPY

```c
#define EVM_ENV_CODE_COPY 3
```


#### EVM_ENV_BLOCKHASH

```c
#define EVM_ENV_BLOCKHASH 4
```


#### EVM_ENV_STORAGE

```c
#define EVM_ENV_STORAGE 5
```


#### EVM_ENV_BLOCKHEADER

```c
#define EVM_ENV_BLOCKHEADER 6
```


#### EVM_ENV_CODE_HASH

```c
#define EVM_ENV_CODE_HASH 7
```


#### EVM_ENV_NONCE

```c
#define EVM_ENV_NONCE 8
```


#### MATH_ADD

```c
#define MATH_ADD 1
```


#### MATH_SUB

```c
#define MATH_SUB 2
```


#### MATH_MUL

```c
#define MATH_MUL 3
```


#### MATH_DIV

```c
#define MATH_DIV 4
```


#### MATH_SDIV

```c
#define MATH_SDIV 5
```


#### MATH_MOD

```c
#define MATH_MOD 6
```


#### MATH_SMOD

```c
#define MATH_SMOD 7
```


#### MATH_EXP

```c
#define MATH_EXP 8
```


#### MATH_SIGNEXP

```c
#define MATH_SIGNEXP 9
```


#### CALL_CALL

```c
#define CALL_CALL 0
```


#### CALL_CODE

```c
#define CALL_CODE 1
```


#### CALL_DELEGATE

```c
#define CALL_DELEGATE 2
```


#### CALL_STATIC

```c
#define CALL_STATIC 3
```


#### OP_AND

```c
#define OP_AND 0
```


#### OP_OR

```c
#define OP_OR 1
```


#### OP_XOR

```c
#define OP_XOR 2
```


#### EVM_DEBUG_BLOCK (...)


#### OP_LOG (...)

```c
#define OP_LOG (...) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### OP_SLOAD_GAS (...)


#### OP_CREATE (...)

```c
#define OP_CREATE (...) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### OP_ACCOUNT_GAS (...)

```c
#define OP_ACCOUNT_GAS (...) exit_zero()
```


#### OP_SELFDESTRUCT (...)

```c
#define OP_SELFDESTRUCT (...) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### OP_EXTCODECOPY_GAS (evm)


#### OP_SSTORE (...)

```c
#define OP_SSTORE (...) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### EVM_CALL_MODE_STATIC

```c
#define EVM_CALL_MODE_STATIC 1
```


#### EVM_CALL_MODE_DELEGATE

```c
#define EVM_CALL_MODE_DELEGATE 2
```


#### EVM_CALL_MODE_CALLCODE

```c
#define EVM_CALL_MODE_CALLCODE 3
```


#### EVM_CALL_MODE_CALL

```c
#define EVM_CALL_MODE_CALL 4
```


#### evm_state

the current state of the evm 

The enum type contains the following values:

```eval_rst
======================== = =====================================
 **EVM_STATE_INIT**      0 just initialised, but not yet started
 **EVM_STATE_RUNNING**   1 started and still running
 **EVM_STATE_STOPPED**   2 successfully stopped
 **EVM_STATE_REVERTED**  3 stopped, but results must be reverted
======================== = =====================================
```

#### evm_state_t

the current state of the evm 


The enum type contains the following values:

```eval_rst
======================== = =====================================
 **EVM_STATE_INIT**      0 just initialised, but not yet started
 **EVM_STATE_RUNNING**   1 started and still running
 **EVM_STATE_STOPPED**   2 successfully stopped
 **EVM_STATE_REVERTED**  3 stopped, but results must be reverted
======================== = =====================================
```

#### evm_get_env

This function provides data from the enviroment. 

depending on the key the function will set the out_data-pointer to the result. This means the enviroment is responsible for memory management and also to clean up resources afterwards.


```c
typedef int(* evm_get_env) (void *evm, uint16_t evm_key, uint8_t *in_data, int in_len, uint8_t **out_data, int offset, int len)
```

returns: `int(*`


#### storage_t


The stuct contains following fields:

```eval_rst
=============================================== =========== 
`bytes32_t <#bytes32-t>`_                        **key**    
`bytes32_t <#bytes32-t>`_                        **value**  
`account_storagestruct , * <#account-storage>`_  **next**   
=============================================== =========== 
```

#### logs_t


The stuct contains following fields:

```eval_rst
========================= ============= 
`bytes_t <#bytes-t>`_      **topics**   
`bytes_t <#bytes-t>`_      **data**     
`address_t <#address-t>`_  **address**  
`logsstruct , * <#logs>`_  **next**     
========================= ============= 
```

#### account_t


The stuct contains following fields:

```eval_rst
=============================== ============= 
`address_t <#address-t>`_        **address**  
`bytes32_t <#bytes32-t>`_        **balance**  
`bytes32_t <#bytes32-t>`_        **nonce**    
`bytes_t <#bytes-t>`_            **code**     
`storage_t * <#storage-t>`_      **storage**  
`accountstruct , * <#account>`_  **next**     
=============================== ============= 
```

#### evm_t


The stuct contains following fields:

```eval_rst
===================================== ====================== ======================================================
`bytes_builder_t <#bytes-builder-t>`_  **stack**             
`bytes_builder_t <#bytes-builder-t>`_  **memory**            
``int``                                **stack_size**        
`bytes_t <#bytes-t>`_                  **code**              
``uint32_t``                           **pos**               
`evm_state_t <#evm-state-t>`_          **state**             
`bytes_t <#bytes-t>`_                  **last_returned**     
`bytes_t <#bytes-t>`_                  **return_data**       
``uint32_t *``                         **invalid_jumpdest**  
``uint32_t``                           **properties**        
`evm_get_env <#evm-get-env>`_          **env**               
``void *``                             **env_ptr**           
``uint64_t``                           **chain_id**          the chain_id as returned by the opcode
``uint8_t *``                          **address**           the address of the current storage
``uint8_t *``                          **account**           the address of the code
``uint8_t *``                          **origin**            the address of original sender of the root-transaction
``uint8_t *``                          **caller**            the address of the parent sender
`bytes_t <#bytes-t>`_                  **call_value**        value send
`bytes_t <#bytes-t>`_                  **call_data**         data send in the tx
`bytes_t <#bytes-t>`_                  **gas_price**         current gasprice
``uint64_t``                           **gas**               
````                                   **gas_options**       
===================================== ====================== ======================================================
```

#### exit_zero

```c
int exit_zero(void);
```

arguments:
```eval_rst
========  
``void``  
========  
```
returns: `int`


#### evm_stack_push

```c
int evm_stack_push(evm_t *evm, uint8_t *data, uint8_t len);
```

arguments:
```eval_rst
=================== ========== 
`evm_t * <#evm-t>`_  **evm**   
``uint8_t *``        **data**  
``uint8_t``          **len**   
=================== ========== 
```
returns: `int`


#### evm_stack_push_ref

```c
int evm_stack_push_ref(evm_t *evm, uint8_t **dst, uint8_t len);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t **``       **dst**  
``uint8_t``          **len**  
=================== ========= 
```
returns: `int`


#### evm_stack_push_int

```c
int evm_stack_push_int(evm_t *evm, uint32_t val);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint32_t``         **val**  
=================== ========= 
```
returns: `int`


#### evm_stack_push_long

```c
int evm_stack_push_long(evm_t *evm, uint64_t val);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint64_t``         **val**  
=================== ========= 
```
returns: `int`


#### evm_stack_get_ref

```c
int evm_stack_get_ref(evm_t *evm, uint8_t pos, uint8_t **dst);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t``          **pos**  
``uint8_t **``       **dst**  
=================== ========= 
```
returns: `int`


#### evm_stack_pop

```c
int evm_stack_pop(evm_t *evm, uint8_t *dst, uint8_t len);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t *``        **dst**  
``uint8_t``          **len**  
=================== ========= 
```
returns: `int`


#### evm_stack_pop_ref

```c
int evm_stack_pop_ref(evm_t *evm, uint8_t **dst);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t **``       **dst**  
=================== ========= 
```
returns: `int`


#### evm_stack_pop_byte

```c
int evm_stack_pop_byte(evm_t *evm, uint8_t *dst);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t *``        **dst**  
=================== ========= 
```
returns: `int`


#### evm_stack_pop_int

```c
int32_t evm_stack_pop_int(evm_t *evm);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
=================== ========= 
```
returns: `int32_t`


#### evm_run

```c
int evm_run(evm_t *evm, address_t code_address);
```

arguments:
```eval_rst
========================= ================== 
`evm_t * <#evm-t>`_        **evm**           
`address_t <#address-t>`_  **code_address**  
========================= ================== 
```
returns: `int`


#### evm_sub_call

```c
int evm_sub_call(evm_t *parent, uint8_t address[20], uint8_t account[20], uint8_t *value, wlen_t l_value, uint8_t *data, uint32_t l_data, uint8_t caller[20], uint8_t origin[20], uint64_t gas, wlen_t mode, uint32_t out_offset, uint32_t out_len);
```

handle internal calls. 

arguments:
```eval_rst
=================== ================ 
`evm_t * <#evm-t>`_  **parent**      
``uint8_t``          **address**     
``uint8_t``          **account**     
``uint8_t *``        **value**       
`wlen_t <#wlen-t>`_  **l_value**     
``uint8_t *``        **data**        
``uint32_t``         **l_data**      
``uint8_t``          **caller**      
``uint8_t``          **origin**      
``uint64_t``         **gas**         
`wlen_t <#wlen-t>`_  **mode**        
``uint32_t``         **out_offset**  
``uint32_t``         **out_len**     
=================== ================ 
```
returns: `int`


#### evm_ensure_memory

```c
int evm_ensure_memory(evm_t *evm, uint32_t max_pos);
```

arguments:
```eval_rst
=================== ============= 
`evm_t * <#evm-t>`_  **evm**      
``uint32_t``         **max_pos**  
=================== ============= 
```
returns: `int`


#### in3_get_env

```c
int in3_get_env(void *evm_ptr, uint16_t evm_key, uint8_t *in_data, int in_len, uint8_t **out_data, int offset, int len);
```

arguments:
```eval_rst
============== ============== 
``void *``      **evm_ptr**   
``uint16_t``    **evm_key**   
``uint8_t *``   **in_data**   
``int``         **in_len**    
``uint8_t **``  **out_data**  
``int``         **offset**    
``int``         **len**       
============== ============== 
```
returns: `int`


#### evm_call

```c
int evm_call(void *vc, uint8_t address[20], uint8_t *value, wlen_t l_value, uint8_t *data, uint32_t l_data, uint8_t caller[20], uint64_t gas, uint64_t chain_id, bytes_t **result, json_ctx_t *receipt);
```

run a evm-call 

arguments:
```eval_rst
============================= ============== 
``void *``                     **vc**        
``uint8_t``                    **address**   
``uint8_t *``                  **value**     
`wlen_t <#wlen-t>`_            **l_value**   
``uint8_t *``                  **data**      
``uint32_t``                   **l_data**    
``uint8_t``                    **caller**    
``uint64_t``                   **gas**       
``uint64_t``                   **chain_id**  
`bytes_t ** <#bytes-t>`_       **result**    
`json_ctx_t * <#json-ctx-t>`_  **receipt**   
============================= ============== 
```
returns: `int`


#### evm_print_stack

```c
void evm_print_stack(evm_t *evm, uint64_t last_gas, uint32_t pos);
```

arguments:
```eval_rst
=================== ============== 
`evm_t * <#evm-t>`_  **evm**       
``uint64_t``         **last_gas**  
``uint32_t``         **pos**       
=================== ============== 
```

#### evm_free

```c
void evm_free(evm_t *evm);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
=================== ========= 
```

#### evm_execute

```c
int evm_execute(evm_t *evm);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
=================== ========= 
```
returns: `int`


### gas.h

evm gas defines. 

File: [c/src/verifier/eth1/evm/gas.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/evm/gas.h)

#### op_exec (m,gas)

```c
#define op_exec (m,gas) return m;
```


#### subgas (g)


#### GAS_CC_NET_SSTORE_NOOP_GAS

Once per SSTORE operation if the value doesn't change. 

```c
#define GAS_CC_NET_SSTORE_NOOP_GAS 200
```


#### GAS_CC_NET_SSTORE_INIT_GAS

Once per SSTORE operation from clean zero. 

```c
#define GAS_CC_NET_SSTORE_INIT_GAS 20000
```


#### GAS_CC_NET_SSTORE_CLEAN_GAS

Once per SSTORE operation from clean non-zero. 

```c
#define GAS_CC_NET_SSTORE_CLEAN_GAS 5000
```


#### GAS_CC_NET_SSTORE_DIRTY_GAS

Once per SSTORE operation from dirty. 

```c
#define GAS_CC_NET_SSTORE_DIRTY_GAS 200
```


#### GAS_CC_NET_SSTORE_CLEAR_REFUND

Once per SSTORE operation for clearing an originally existing storage slot. 

```c
#define GAS_CC_NET_SSTORE_CLEAR_REFUND 15000
```


#### GAS_CC_NET_SSTORE_RESET_REFUND

Once per SSTORE operation for resetting to the original non-zero value. 

```c
#define GAS_CC_NET_SSTORE_RESET_REFUND 4800
```


#### GAS_CC_NET_SSTORE_RESET_CLEAR_REFUND

Once per SSTORE operation for resetting to the original zero valuev. 

```c
#define GAS_CC_NET_SSTORE_RESET_CLEAR_REFUND 19800
```


#### G_ZERO

Nothing is paid for operations of the set Wzero. 

```c
#define G_ZERO 0
```


#### G_JUMPDEST

JUMP DEST. 

```c
#define G_JUMPDEST 1
```


#### G_BASE

This is the amount of gas to pay for operations of the set Wbase. 

```c
#define G_BASE 2
```


#### G_VERY_LOW

This is the amount of gas to pay for operations of the set Wverylow. 

```c
#define G_VERY_LOW 3
```


#### G_LOW

This is the amount of gas to pay for operations of the set Wlow. 

```c
#define G_LOW 5
```


#### G_MID

This is the amount of gas to pay for operations of the set Wmid. 

```c
#define G_MID 8
```


#### G_HIGH

This is the amount of gas to pay for operations of the set Whigh. 

```c
#define G_HIGH 10
```


#### G_EXTCODE

This is the amount of gas to pay for operations of the set Wextcode. 

```c
#define G_EXTCODE 700
```


#### G_BALANCE

This is the amount of gas to pay for a BALANCE operation. 

```c
#define G_BALANCE 400
```


#### G_SLOAD

This is paid for an SLOAD operation. 

```c
#define G_SLOAD 200
```


#### G_SSET

This is paid for an SSTORE operation when the storage value is set to non-zero from zero. 

```c
#define G_SSET 20000
```


#### G_SRESET

This is the amount for an SSTORE operation when the storage value's zeroness remains unchanged or is set to zero. 

```c
#define G_SRESET 5000
```


#### R_SCLEAR

This is the refund given (added into the refund counter) when the storage value is set to zero from non-zero. 

```c
#define R_SCLEAR 15000
```


#### R_SELFDESTRUCT

This is the refund given (added into the refund counter) for self-destructing an account. 

```c
#define R_SELFDESTRUCT 24000
```


#### G_SELFDESTRUCT

This is the amount of gas to pay for a SELFDESTRUCT operation. 

```c
#define G_SELFDESTRUCT 5000
```


#### G_CREATE

This is paid for a CREATE operation. 

```c
#define G_CREATE 32000
```


#### G_CODEDEPOSIT

This is paid per byte for a CREATE operation to succeed in placing code into the state. 

```c
#define G_CODEDEPOSIT 200
```


#### G_CALL

This is paid for a CALL operation. 

```c
#define G_CALL 700
```


#### G_CALLVALUE

This is paid for a non-zero value transfer as part of the CALL operation. 

```c
#define G_CALLVALUE 9000
```


#### G_CALLSTIPEND

This is a stipend for the called contract subtracted from Gcallvalue for a non-zero value transfer. 

```c
#define G_CALLSTIPEND 2300
```


#### G_NEWACCOUNT

This is paid for a CALL or for a SELFDESTRUCT operation which creates an account. 

```c
#define G_NEWACCOUNT 25000
```


#### G_EXP

This is a partial payment for an EXP operation. 

```c
#define G_EXP 10
```


#### G_EXPBYTE

This is a partial payment when multiplied by dlog256(exponent)e for the EXP operation. 

```c
#define G_EXPBYTE 50
```


#### G_MEMORY

This is paid for every additional word when expanding memory. 

```c
#define G_MEMORY 3
```


#### G_TXCREATE

This is paid by all contract-creating transactions after the Homestead transition. 

```c
#define G_TXCREATE 32000
```


#### G_TXDATA_ZERO

This is paid for every zero byte of data or code for a transaction. 

```c
#define G_TXDATA_ZERO 4
```


#### G_TXDATA_NONZERO

This is paid for every non-zero byte of data or code for a transaction. 

```c
#define G_TXDATA_NONZERO 68
```


#### G_TRANSACTION

This is paid for every transaction. 

```c
#define G_TRANSACTION 21000
```


#### G_LOG

This is a partial payment for a LOG operation. 

```c
#define G_LOG 375
```


#### G_LOGDATA

This is paid for each byte in a LOG operation's data. 

```c
#define G_LOGDATA 8
```


#### G_LOGTOPIC

This is paid for each topic of a LOG operation. 

```c
#define G_LOGTOPIC 375
```


#### G_SHA3

This is paid for each SHA3 operation. 

```c
#define G_SHA3 30
```


#### G_SHA3WORD

This is paid for each word (rounded up) for input data to a SHA3 operation. 

```c
#define G_SHA3WORD 6
```


#### G_COPY

This is a partial payment for *COPY operations, multiplied by the number of words copied, rounded up. 

```c
#define G_COPY 3
```


#### G_BLOCKHASH

This is a payment for a BLOCKHASH operation. 

```c
#define G_BLOCKHASH 20
```


#### G_PRE_EC_RECOVER

Precompile EC RECOVER. 

```c
#define G_PRE_EC_RECOVER 3000
```


#### G_PRE_SHA256

Precompile SHA256. 

```c
#define G_PRE_SHA256 60
```


#### G_PRE_SHA256_WORD

Precompile SHA256 per word. 

```c
#define G_PRE_SHA256_WORD 12
```


#### G_PRE_RIPEMD160

Precompile RIPEMD160. 

```c
#define G_PRE_RIPEMD160 600
```


#### G_PRE_RIPEMD160_WORD

Precompile RIPEMD160 per word. 

```c
#define G_PRE_RIPEMD160_WORD 120
```


#### G_PRE_IDENTITY

Precompile IDENTIY (copyies data) 

```c
#define G_PRE_IDENTITY 15
```


#### G_PRE_IDENTITY_WORD

Precompile IDENTIY per word. 

```c
#define G_PRE_IDENTITY_WORD 3
```


#### G_PRE_MODEXP_GQUAD_DIVISOR

Gquaddivisor from modexp precompile for gas calculation. 

```c
#define G_PRE_MODEXP_GQUAD_DIVISOR 20
```


#### G_PRE_ECADD

Gas costs for curve addition precompile. 

```c
#define G_PRE_ECADD 500
```


#### G_PRE_ECMUL

Gas costs for curve multiplication precompile. 

```c
#define G_PRE_ECMUL 40000
```


#### G_PRE_ECPAIRING

Base gas costs for curve pairing precompile. 

```c
#define G_PRE_ECPAIRING 100000
```


#### G_PRE_ECPAIRING_WORD

Gas costs regarding curve pairing precompile input length. 

```c
#define G_PRE_ECPAIRING_WORD 80000
```


#### EVM_STACK_LIMIT

max elements of the stack 

```c
#define EVM_STACK_LIMIT 1024
```


#### EVM_MAX_CODE_SIZE

max size of the code 

```c
#define EVM_MAX_CODE_SIZE 24576
```


#### FRONTIER_G_EXPBYTE

fork values 

This is a partial payment when multiplied by dlog256(exponent)e for the EXP operation. 

```c
#define FRONTIER_G_EXPBYTE 10
```


#### FRONTIER_G_SLOAD

This is a partial payment when multiplied by dlog256(exponent)e for the EXP operation. 

```c
#define FRONTIER_G_SLOAD 50
```


#### FREE_EVM (...)


#### INIT_EVM (...)


#### INIT_GAS (...)


#### SUBGAS (...)


#### FINALIZE_SUBCALL_GAS (...)


#### UPDATE_SUBCALL_GAS (...)


#### FINALIZE_AND_REFUND_GAS (...)


#### KEEP_TRACK_GAS (evm)

```c
#define KEEP_TRACK_GAS (evm) 0
```


#### UPDATE_ACCOUNT_CODE (...)


### eth_full.h

Ethereum Nanon verification. 

File: [c/src/verifier/eth1/full/eth_full.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/full/eth_full.h)

#### in3_register_eth_full

```c
in3_ret_t in3_register_eth_full(in3_t *c);
```

this function should only be called once and will register the eth-full verifier. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### chainspec.h

Ethereum chain specification. 

File: [c/src/verifier/eth1/nano/chainspec.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/nano/chainspec.h)

#### BLOCK_LATEST

```c
#define BLOCK_LATEST 0xFFFFFFFFFFFFFFFF
```


#### eth_consensus_type_t

the consensus type. 

The enum type contains the following values:

```eval_rst
==================== = ================================
 **ETH_POW**         0 Pro of Work (Ethash)
 **ETH_POA_AURA**    1 Proof of Authority using Aura.
 **ETH_POA_CLIQUE**  2 Proof of Authority using clique.
==================== = ================================
```

#### eip_transition_t


The stuct contains following fields:

```eval_rst
============ ====================== 
``uint64_t``  **transition_block**  
``eip_t``     **eips**              
============ ====================== 
```

#### consensus_transition_t


The stuct contains following fields:

```eval_rst
=============================================== ====================== 
``uint64_t``                                     **transition_block**  
`eth_consensus_type_t <#eth-consensus-type-t>`_  **type**              
`bytes_t <#bytes-t>`_                            **validators**        
``uint8_t *``                                    **contract**          
=============================================== ====================== 
```

#### chainspec_t


The stuct contains following fields:

```eval_rst
===================================================== =============================== 
``uint64_t``                                           **network_id**                 
``uint64_t``                                           **account_start_nonce**        
``uint32_t``                                           **eip_transitions_len**        
`eip_transition_t * <#eip-transition-t>`_              **eip_transitions**            
``uint32_t``                                           **consensus_transitions_len**  
`consensus_transition_t * <#consensus-transition-t>`_  **consensus_transitions**      
===================================================== =============================== 
```

#### __attribute__

```c
struct __attribute__((__packed__)) eip_;
```

defines the flags for the current activated EIPs. 

Since it does not make sense to support a evm defined before Homestead, homestead EIP is always turned on! 

< REVERT instruction

< Bitwise shifting instructions in EVM

< Gas cost changes for IO-heavy operations

< Simple replay attack protection

< EXP cost increase

< Contract code size limit

< Precompiled contracts for addition and scalar multiplication on the elliptic curve alt_bn128 




< Precompiled contracts for optimal ate pairing check on the elliptic curve alt_bn128 




< Big integer modular exponentiation 




< New opcodes: RETURNDATASIZE and RETURNDATACOPY 




< New opcode STATICCALL 




< Embedding transaction status code in receipts

< Skinny CREATE2 




< EXTCODEHASH opcode

< Net gas metering for SSTORE without dirty maps 




arguments:
```eval_rst
================  
``(__packed__)``  
================  
```
returns: `struct`


#### chainspec_create_from_json

```c
chainspec_t* chainspec_create_from_json(json_ctx_t *data);
```

arguments:
```eval_rst
============================= ========== 
`json_ctx_t * <#json-ctx-t>`_  **data**  
============================= ========== 
```
returns: [`chainspec_t *`](#chainspec-t)


#### chainspec_get_eip

```c
eip_t chainspec_get_eip(chainspec_t *spec, uint64_t block_number);
```

arguments:
```eval_rst
=============================== ================== 
`chainspec_t * <#chainspec-t>`_  **spec**          
``uint64_t``                     **block_number**  
=============================== ================== 
```
returns: `eip_t`


#### chainspec_get_consensus

```c
consensus_transition_t* chainspec_get_consensus(chainspec_t *spec, uint64_t block_number);
```

arguments:
```eval_rst
=============================== ================== 
`chainspec_t * <#chainspec-t>`_  **spec**          
``uint64_t``                     **block_number**  
=============================== ================== 
```
returns: [`consensus_transition_t *`](#consensus-transition-t)


#### chainspec_to_bin

```c
in3_ret_t chainspec_to_bin(chainspec_t *spec, bytes_builder_t *bb);
```

arguments:
```eval_rst
======================================= ========== 
`chainspec_t * <#chainspec-t>`_          **spec**  
`bytes_builder_t * <#bytes-builder-t>`_  **bb**    
======================================= ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### chainspec_from_bin

```c
chainspec_t* chainspec_from_bin(void *raw);
```

arguments:
```eval_rst
========== ========= 
``void *``  **raw**  
========== ========= 
```
returns: [`chainspec_t *`](#chainspec-t)


#### chainspec_get

```c
chainspec_t* chainspec_get(chain_id_t chain_id);
```

arguments:
```eval_rst
=========================== ============== 
`chain_id_t <#chain-id-t>`_  **chain_id**  
=========================== ============== 
```
returns: [`chainspec_t *`](#chainspec-t)


#### chainspec_put

```c
void chainspec_put(chain_id_t chain_id, chainspec_t *spec);
```

arguments:
```eval_rst
=============================== ============== 
`chain_id_t <#chain-id-t>`_      **chain_id**  
`chainspec_t * <#chainspec-t>`_  **spec**      
=============================== ============== 
```

### eth_nano.h

Ethereum Nanon verification. 

File: [c/src/verifier/eth1/nano/eth_nano.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/nano/eth_nano.h)

#### in3_verify_eth_nano

```c
NONULL in3_ret_t in3_verify_eth_nano(void *p_data, in3_plugin_act_t action, void *pctx);
```

entry-function to execute the verification context. 

arguments:
```eval_rst
======================================= ============ 
``void *``                               **p_data**  
`in3_plugin_act_t <#in3-plugin-act-t>`_  **action**  
``void *``                               **pctx**    
======================================= ============ 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_blockheader

```c
in3_ret_t eth_verify_blockheader(in3_vctx_t *vc, bytes_t *header, bytes_t *expected_blockhash);
```

verifies a blockheader. 

verifies a blockheader. 

arguments:
```eval_rst
============================= ======================== 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**                  
`bytes_t * <#bytes-t>`_        **header**              
`bytes_t * <#bytes-t>`_        **expected_blockhash**  
============================= ======================== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_signature

```c
NONULL unsigned int eth_verify_signature(in3_vctx_t *vc, bytes_t *msg_hash, d_token_t *sig);
```

verifies a single signature blockheader. 

This function will return a positive integer with a bitmask holding the bit set according to the address that signed it. This is based on the signatiures in the request-config. 

arguments:
```eval_rst
============================= ============== 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**        
`bytes_t * <#bytes-t>`_        **msg_hash**  
`d_token_t * <#d-token-t>`_    **sig**       
============================= ============== 
```
returns: `NONULL unsigned int`


#### ecrecover_signature

```c
NONULL bytes_t* ecrecover_signature(bytes_t *msg_hash, d_token_t *sig);
```

returns the address of the signature if the msg_hash is correct 

arguments:
```eval_rst
=========================== ============== 
`bytes_t * <#bytes-t>`_      **msg_hash**  
`d_token_t * <#d-token-t>`_  **sig**       
=========================== ============== 
```
returns: [`bytes_tNONULL , *`](#bytes-t)


#### eth_verify_eth_getTransactionReceipt

```c
NONULL in3_ret_t eth_verify_eth_getTransactionReceipt(in3_vctx_t *vc, bytes_t *tx_hash);
```

verifies a transaction receipt. 

arguments:
```eval_rst
============================= ============= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**       
`bytes_t * <#bytes-t>`_        **tx_hash**  
============================= ============= 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_eth_nano

```c
NONULL in3_ret_t in3_register_eth_nano(in3_t *c);
```

this function should only be called once and will register the eth-nano verifier. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_tNONULL `](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### create_tx_path

```c
bytes_t* create_tx_path(uint32_t index);
```

helper function to rlp-encode the transaction_index. 

The result must be freed after use! 

arguments:
```eval_rst
============ =========== 
``uint32_t``  **index**  
============ =========== 
```
returns: [`bytes_t *`](#bytes-t)


### merkle.h

Merkle Proof Verification. 

File: [c/src/verifier/eth1/nano/merkle.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/nano/merkle.h)

#### MERKLE_DEPTH_MAX

```c
#define MERKLE_DEPTH_MAX 64
```


#### trie_verify_proof

```c
int trie_verify_proof(bytes_t *rootHash, bytes_t *path, bytes_t **proof, bytes_t *expectedValue);
```

verifies a merkle proof. 

expectedValue == NULL : value must not exist expectedValue.data ==NULL : please copy the data I want to evaluate it afterwards. expectedValue.data !=NULL : the value must match the data.

arguments:
```eval_rst
======================== =================== 
`bytes_t * <#bytes-t>`_   **rootHash**       
`bytes_t * <#bytes-t>`_   **path**           
`bytes_t ** <#bytes-t>`_  **proof**          
`bytes_t * <#bytes-t>`_   **expectedValue**  
======================== =================== 
```
returns: `int`


#### trie_path_to_nibbles

```c
NONULL uint8_t* trie_path_to_nibbles(bytes_t path, int use_prefix);
```

helper function split a path into 4-bit nibbles. 

The result must be freed after use!

arguments:
```eval_rst
===================== ================ 
`bytes_t <#bytes-t>`_  **path**        
``int``                **use_prefix**  
===================== ================ 
```
returns: `NONULL uint8_t *` : the resulting bytes represent a 4bit-number each and are terminated with a 0xFF. 




#### trie_matching_nibbles

```c
NONULL int trie_matching_nibbles(uint8_t *a, uint8_t *b);
```

helper function to find the number of nibbles matching both paths. 

arguments:
```eval_rst
============= ======= 
``uint8_t *``  **a**  
``uint8_t *``  **b**  
============= ======= 
```
returns: `NONULL int`


### rlp.h

RLP-En/Decoding as described in the [Ethereum RLP-Spec](https://github.com/ethereum/wiki/wiki/RLP). 

This decoding works without allocating new memory. 

File: [c/src/verifier/eth1/nano/rlp.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/nano/rlp.h)

#### rlp_decode

```c
int rlp_decode(bytes_t *b, int index, bytes_t *dst);
```

this function decodes the given bytes and returns the element with the given index by updating the reference of dst. 

the bytes will only hold references and do **not** need to be freed!

```c
bytes_t* tx_raw = serialize_tx(tx);

bytes_t item;

// decodes the tx_raw by letting the item point to range of the first element, which should be the body of a list.
if (rlp_decode(tx_raw, 0, &item) !=2) return -1 ;

// now decode the 4th element (which is the value) and let item point to that range.
if (rlp_decode(&item, 4, &item) !=1) return -1 ;
```
arguments:
```eval_rst
======================= =========== 
`bytes_t * <#bytes-t>`_  **b**      
``int``                  **index**  
`bytes_t * <#bytes-t>`_  **dst**    
======================= =========== 
```
returns: `int` : - 0 : means item out of range
- 1 : item found
- 2 : list found ( you can then decode the same bytes again)




#### rlp_decode_in_list

```c
int rlp_decode_in_list(bytes_t *b, int index, bytes_t *dst);
```

this function expects a list item (like the blockheader as first item and will then find the item within this list). 

It is a shortcut for

```c
// decode the list
if (rlp_decode(b,0,dst)!=2) return 0;
// and the decode the item
return rlp_decode(dst,index,dst);
```
arguments:
```eval_rst
======================= =========== 
`bytes_t * <#bytes-t>`_  **b**      
``int``                  **index**  
`bytes_t * <#bytes-t>`_  **dst**    
======================= =========== 
```
returns: `int` : - 0 : means item out of range
- 1 : item found
- 2 : list found ( you can then decode the same bytes again)




#### rlp_decode_len

```c
int rlp_decode_len(bytes_t *b);
```

returns the number of elements found in the data. 

arguments:
```eval_rst
======================= ======= 
`bytes_t * <#bytes-t>`_  **b**  
======================= ======= 
```
returns: `int`


#### rlp_encode_item

```c
void rlp_encode_item(bytes_builder_t *bb, bytes_t *val);
```

encode a item as single string and add it to the bytes_builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
`bytes_t * <#bytes-t>`_                  **val**  
======================================= ========= 
```

#### rlp_encode_list

```c
void rlp_encode_list(bytes_builder_t *bb, bytes_t *val);
```

encode a the value as list of already encoded items. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
`bytes_t * <#bytes-t>`_                  **val**  
======================================= ========= 
```

#### rlp_encode_to_list

```c
bytes_builder_t* rlp_encode_to_list(bytes_builder_t *bb);
```

converts the data in the builder to a list. 

This function is optimized to not increase the memory more than needed and is fastet than creating a second builder to encode the data.

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```
returns: [`bytes_builder_t *`](#bytes-builder-t) : the same builder. 




#### rlp_encode_to_item

```c
bytes_builder_t* rlp_encode_to_item(bytes_builder_t *bb);
```

converts the data in the builder to a rlp-encoded item. 

This function is optimized to not increase the memory more than needed and is faster than creating a second builder to encode the data.

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```
returns: [`bytes_builder_t *`](#bytes-builder-t) : the same builder. 




#### rlp_add_length

```c
void rlp_add_length(bytes_builder_t *bb, uint32_t len, uint8_t offset);
```

helper to encode the prefix for a value 

arguments:
```eval_rst
======================================= ============ 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**      
``uint32_t``                             **len**     
``uint8_t``                              **offset**  
======================================= ============ 
```

### serialize.h

serialization of ETH-Objects. 

This incoming tokens will represent their values as properties based on [JSON-RPC](https://github.com/ethereum/wiki/wiki/JSON-RPC). 

File: [c/src/verifier/eth1/nano/serialize.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/eth1/nano/serialize.h)

#### BLOCKHEADER_PARENT_HASH

```c
#define BLOCKHEADER_PARENT_HASH 0
```


#### BLOCKHEADER_SHA3_UNCLES

```c
#define BLOCKHEADER_SHA3_UNCLES 1
```


#### BLOCKHEADER_MINER

```c
#define BLOCKHEADER_MINER 2
```


#### BLOCKHEADER_STATE_ROOT

```c
#define BLOCKHEADER_STATE_ROOT 3
```


#### BLOCKHEADER_TRANSACTIONS_ROOT

```c
#define BLOCKHEADER_TRANSACTIONS_ROOT 4
```


#### BLOCKHEADER_RECEIPT_ROOT

```c
#define BLOCKHEADER_RECEIPT_ROOT 5
```


#### BLOCKHEADER_LOGS_BLOOM

```c
#define BLOCKHEADER_LOGS_BLOOM 6
```


#### BLOCKHEADER_DIFFICULTY

```c
#define BLOCKHEADER_DIFFICULTY 7
```


#### BLOCKHEADER_NUMBER

```c
#define BLOCKHEADER_NUMBER 8
```


#### BLOCKHEADER_GAS_LIMIT

```c
#define BLOCKHEADER_GAS_LIMIT 9
```


#### BLOCKHEADER_GAS_USED

```c
#define BLOCKHEADER_GAS_USED 10
```


#### BLOCKHEADER_TIMESTAMP

```c
#define BLOCKHEADER_TIMESTAMP 11
```


#### BLOCKHEADER_EXTRA_DATA

```c
#define BLOCKHEADER_EXTRA_DATA 12
```


#### BLOCKHEADER_SEALED_FIELD1

```c
#define BLOCKHEADER_SEALED_FIELD1 13
```


#### BLOCKHEADER_SEALED_FIELD2

```c
#define BLOCKHEADER_SEALED_FIELD2 14
```


#### BLOCKHEADER_SEALED_FIELD3

```c
#define BLOCKHEADER_SEALED_FIELD3 15
```


#### BLOCKHEADER_BASE_GAS_FEE

```c
#define BLOCKHEADER_BASE_GAS_FEE 15
```


#### serialize_tx_receipt

```c
bytes_t* serialize_tx_receipt(d_token_t *receipt);
```

creates rlp-encoded raw bytes for a receipt. 

The bytes must be freed with b_free after use!

arguments:
```eval_rst
=========================== ============= 
`d_token_t * <#d-token-t>`_  **receipt**  
=========================== ============= 
```
returns: [`bytes_t *`](#bytes-t)


#### serialize_tx

```c
bytes_t* serialize_tx(d_token_t *tx);
```

creates rlp-encoded raw bytes for a transaction. 

The bytes must be freed with b_free after use!

arguments:
```eval_rst
=========================== ======== 
`d_token_t * <#d-token-t>`_  **tx**  
=========================== ======== 
```
returns: [`bytes_t *`](#bytes-t)


#### serialize_tx_raw

```c
bytes_t* serialize_tx_raw(bytes_t nonce, bytes_t gas_price, bytes_t gas_limit, bytes_t to, bytes_t value, bytes_t data, uint64_t v, bytes_t r, bytes_t s);
```

creates rlp-encoded raw bytes for a transaction from direct values. 

The bytes must be freed with b_free after use! 

arguments:
```eval_rst
===================== =============== 
`bytes_t <#bytes-t>`_  **nonce**      
`bytes_t <#bytes-t>`_  **gas_price**  
`bytes_t <#bytes-t>`_  **gas_limit**  
`bytes_t <#bytes-t>`_  **to**         
`bytes_t <#bytes-t>`_  **value**      
`bytes_t <#bytes-t>`_  **data**       
``uint64_t``           **v**          
`bytes_t <#bytes-t>`_  **r**          
`bytes_t <#bytes-t>`_  **s**          
===================== =============== 
```
returns: [`bytes_t *`](#bytes-t)


#### serialize_account

```c
bytes_t* serialize_account(d_token_t *a);
```

creates rlp-encoded raw bytes for a account. 

The bytes must be freed with b_free after use! 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **a**  
=========================== ======= 
```
returns: [`bytes_t *`](#bytes-t)


#### serialize_block_header

```c
bytes_t* serialize_block_header(d_token_t *block);
```

creates rlp-encoded raw bytes for a blockheader. 

The bytes must be freed with b_free after use!

arguments:
```eval_rst
=========================== =========== 
`d_token_t * <#d-token-t>`_  **block**  
=========================== =========== 
```
returns: [`bytes_t *`](#bytes-t)


#### rlp_add

```c
int rlp_add(bytes_builder_t *rlp, d_token_t *t, int ml);
```

adds the value represented by the token rlp-encoded to the byte_builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **rlp**  
`d_token_t * <#d-token-t>`_              **t**    
``int``                                  **ml**   
======================================= ========= 
```
returns: `int` : 0 if added -1 if the value could not be handled. 




### ipfs.h

IPFS verification. 

File: [c/src/verifier/ipfs/ipfs.h](https://github.com/slockit/in3-c/blob/master/c/src/verifier/ipfs/ipfs.h)

#### ipfs_verify_hash

```c
in3_ret_t ipfs_verify_hash(const char *content, const char *encoding, const char *requsted_hash);
```

verifies an IPFS hash. 

Supported encoding schemes - hex, utf8 and base64 

arguments:
```eval_rst
================ =================== 
``const char *``  **content**        
``const char *``  **encoding**       
``const char *``  **requsted_hash**  
================ =================== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_ipfs

```c
in3_ret_t in3_register_ipfs(in3_t *c);
```

this function should only be called once and will register the IPFS verifier. 

arguments:
```eval_rst
=================== ======= 
`in3_t * <#in3-t>`_  **c**  
=================== ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


