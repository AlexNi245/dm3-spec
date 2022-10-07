# API Reference TS/JS

Even though the IN3 client is written in C, we are using emscripten to build wasm. Together with some binding-code IN3 runs in any Javascript-Runtime.
Using WASM gives us 3 important features:

1. Performance.
   Since WASM runs at almost native speed it is very fast

2. Security
   Since the WASM-Module has no dependencies it reduces the risk of using a malicious dependency, which would be able to manipulate Prototypes.
   Also, since the real work is happening inside the wasm, trying to change Prototype would not work.

3. Size
   The current wasm-file is about 200kb. This is smaller then most other libraries and can easily be used in any app or website.


## Installing

This client uses the in3-core sources compiled to wasm. The wasm is included into the js-file wich makes it easier to include the data.
This module has **no** dependencies! All it needs is included inta a wasm of about 300kB.

Installing IN3 is as easy as installing any other module:

```
npm install --save in3
```


### WASM-support

Even though most browsers and javascript enviroment such as nodejs, have full support for wasm, there are ocasions, where WASM is fully supported.
In case you want to run IN3 within a react native app, you might face such issues. In this case you can use [in3-asmjs](https://www.npmjs.com/package/in3-asmjs), which has the same API, but runs on pure javascript (a bit slower and bigger, but full support everywhere).

## Building from Source

### install emscripten

In order to build the wasm or asmjs from source you need to install emscripten first. In case you have not done it yet:

```sh
# Get the emsdk repo
git clone https://github.com/emscripten-core/emsdk.git

# Enter that directory
cd emsdk

# install the latest-upstream sdk and activate it
./emsdk install latest-upstream && ./emsdk activate latest-upstream
```

```sh
# Please make sure you add this line to your .bash_profile or .zshrc
source <PATH_TO_EMSDK>/emsdk_env.sh > /dev/null
```

### CMake

With emscripten set up, you can now configure the wasm and build it (in the in3-c directory):

```sh
# create a build directory
mkdir -p build
cd build

# configure CMake
emcmake cmake -DWASM=true -DCMAKE_BUILD_TYPE=MINSIZEREL ..

# and build it
make -j8 in3_wasm

# optionally you can also run the tests
make test
```

Per default the generated wasm embedded the wasm-data as base64 and resulted in the build/module.
If you want to build asmjs, use the `-DASMJS=true` as an additional option.
If you don't want to embedd the wasm, add `-DWASM_EMBED=false`.
If you want to set the `-DCMAKE_BUILD_TYPE=DEBUG` your filesize increases but all function names are kept (resulting in readable stacktraces) and emscriptten will add a lot of checks and assertions.

For more options please see the [CMake Options](https://in3.readthedocs.io/en/develop/api-c.html#cmake-options).
## Examples

### get_block_api

source : [in3-c/wasm/examples/get_block_api.ts](https://github.com/blockchainsllc/in3/blob/master/wasm/examples/get_block_api.ts)

Reads the latest block by calling IN3's web3.js-compatible eth API.
Read the eth api from web3.js docs: https://web3js.readthedocs.io/en/v1.3.0/web3-eth.html


```js
/// Reads the latest block by calling IN3's web3.js-compatible eth API.
/// Read the eth api from web3.js docs: https://web3js.readthedocs.io/en/v1.3.0/web3-eth.html
import { IN3 } from 'in3'

async function showLatestBlock() {

  // create new IN3 instance
  const client = new IN3({
    proof              : 'standard',
    signatureCount     : 1,
    chainId            : 'goerli'
  })

  const lastBlock = await client.eth.getBlockByNumber()

  console.log("latest Block: ", JSON.stringify(lastBlock, null, 2))

}

showLatestBlock().catch(console.error)

```

### get_block_rpc

source : [in3-c/wasm/examples/get_block_rpc.ts](https://github.com/blockchainsllc/in3/blob/master/wasm/examples/get_block_rpc.ts)

Reads the latest block by calling IN3's internal RPC to the WASM core.
Learn other exclusive IN3 RPC calls here: https://in3.readthedocs.io/en/develop/rpc.html


```js
/// Reads the latest block by calling IN3's internal RPC to the WASM core.
/// Learn other exclusive IN3 RPC calls here: https://in3.readthedocs.io/en/develop/rpc.html
import { IN3 } from 'in3'

async function showLatestBlock() {

    // create new IN3 instance
    var c = new IN3({
        chainId: '0x5' // use goerli
    })

    // make a RPC (this would throw if the response contains an error)
    const lastBlockResponse = await c.sendRPC('eth_getBlockByNumber', ['latest', false])

    console.log("latest Block: ", JSON.stringify(lastBlockResponse, null, 2))

}

showLatestBlock().catch(console.error)

```

### register_pugin

source : [in3-c/wasm/examples/register_pugin.ts](https://github.com/blockchainsllc/in3/blob/master/wasm/examples/register_pugin.ts)



```js
// Register a custom RPC for sha256 hashing using a plugin
// Read about IN3 Plugins in the docs: https://in3.readthedocs.io/en/develop/api-c.html#plugins
import { IN3, RPCRequest, IN3Plugin } from 'in3'
import * as crypto from 'crypto'

class Sha256Plugin<BigIntType, BufferType> implements IN3Plugin<BigIntType, BufferType> {

  // this function will register for handling rpc-methods
  // only if we return something other then `undefined`, it will be taken as the result of the rpc.
  // if we don't return, the request will be forwarded to the IN3 nodes
  handleRPC(client, request: RPCRequest): any {

    if (request.method === 'sha256') {
      // assert params
      if (request.params.length != 1 || typeof (request.params[0]) != 'string')
        throw new Error('Only one parameter with as string is expected!')

      // create hash
      const hash = crypto.createHash('sha256').update(Buffer.from(request.params[0], 'utf8')).digest()

      // return the result
      return '0x' + hash.toString('hex')
    }
  }

}

async function registerPlugin() {

  // create new IN3 instance
  const client = new IN3()

  // register the plugin
  client.registerPlugin(new Sha256Plugin())

  // exeucte a rpc-call
  const result = await client.sendRPC("sha256", ["testdata"])

  console.log(" sha256: ", result)

}

registerPlugin().catch(console.error)

```

### use_web3

source : [in3-c/wasm/examples/use_web3.ts](https://github.com/blockchainsllc/in3/blob/master/wasm/examples/use_web3.ts)

use IN3 as Web3Provider in web3.js


```js
/// use IN3 as Web3Provider in web3.js
import {IN3} from 'in3'

const Web3 = require('web3')

const in3 = new IN3({
    proof: 'standard',
    signatureCount: 1,
    requestCount: 1,
    chainId: 'goerli',
    replaceLatestBlock: 10
})

// Use IN3 network client as a Http-Provider
const web3 = new Web3(in3.createWeb3Provider());

(async () => {
    const block = await web3.eth.getBlock('latest')
    console.log("Block : ", block)
})().catch(console.error);

```

### in3_in_browser

source : [in3-c/wasm/examples/in3_in_browser.html](https://github.com/blockchainsllc/in3/blob/master/wasm/examples/in3_in_browser.html)

use IN3 in html 


```html
<!-- use IN3 in html -->
<html>

<head>
    <script src="node_modules/in3/index.js"></script>
</head>

<body>
    IN3-Demo
    <div>
        result:
        <pre id="result"> ...waiting... </pre>
    </div>
    <script>
        var in3 = new IN3({ chainId: 0x1, replaceLatestBlock: 6, requestCount: 3 });
        in3.eth.getBlockByNumber('latest', false)
            .then(block => document.getElementById('result').innerHTML = JSON.stringify(block, null, 2))
            .catch(alert)
    </script>
</body>

</html>

```


### Building

In order to run those examples, you need to install in3 and typescript first.
The build.sh will do this and the run the tsc-compiler

```sh
./build.sh
```

In order to run a example use

```
node build/get_block_api.ts
```
## Incubed - from Typescript to WASM

When we started to implement Incubed, it all began with a Proof of Concept. This was implemented in Typescript and showed that it was possible to verify all important ethereum rpc-requests. Out of thie PoC we continued to develop the first release of the Typescript Client at Devcon 2017. But the goal of incubed was always to be able to run in embedded devices with minimal specs (256kB RAM), so we started to implement the incubed client in C from scratch.

### Why C? (not Rust?)

Don't get me wrong, I love Rust and in the beginning this seemed the best way to implement it, but when looking at our target platforms (embeded devices) you soon find out that almost all of them use C or C++. It would be way easier to get a C-developer to inculde a C-Library than to change its toolchain to include Rust. Also at that time Rust was not ready for embedded yet. 
„Integrating Rust with an RTOS such as FreeRTOS or ChibiOS is still a work in progress; especially calling RTOS functions from Rust can be tricky.“ (https://rust-embedded.org)
When we looked at Stackoverflow Developer Survey 2019 for the most loved Language, 83% of developers made Rust the #1, but looking at the results for actually usage, only 3% used Rust in their products.
Another reason was the fact, that if you write code in C it will run everywhere. Especially embedded devices like to come with their own toolchain only supporting C/C++.

### Size matters

While most Dektop-PCs are powerful enough so you wouldn't care how big a library is, there are still 2 platforms, where size matters a lot:
1. Browsers - when including it in a webapp each user needs to download the code. So having a small library means a faster and better user experience.
2. Embedded Devices - if you only have 1MB flash memory you simply can't use huge libraries.

The smallest possible incubed client is around 300kb including a full evm and running on devices with about 100kB RAM. 

### Emscripten

With Incubed implemented in C we used emscripten to compile it to WASM. This allows us to run almost at native speed in the browser. While almost all browsers support WASM, yet there are still JS-Engines without WebAssembly-Support. Luckely emscripten is able to compile it to asmjs as well, which allows us to also run ( a bit slower and bigger ) even there (which by the way includes react native !).

### Security - no dependencies

Packing all in one wasm-file is not only creating a very compact size (The minimal wasm-size is 162kB), but also is a lot safer. 
When you look at your node_modules-folder after installing the old typescript-client, you would find 267 packages from various authors. 
if you do the same with the wasm-client version 3.2 now, you will find only one folder : `in3`. Yes, We are very proud of our library with ZERO dependencies!
Why are dependencies bad? According to a security survey by npm, 77% of respondents were concerned with the security of OSS/third-party code. Especially when writing security critical applications, auditing all even nested dependencies is very hard. (Remember when an innocent package like [event-stream](https://snyk.io/blog/malicious-code-found-in-npm-package-event-stream/) became malicious and got downloaded 8 Million times? )

## How to migrate?

For the WASM-Client we tried to keep the API as close as possible to the old TypeScript-Version, but due to new features and some WASM-specific behaviors there are some changes you need to be aware of:

1. **WASM loads async**
    Since WebAssembly is always loaded async, some function (like special util-functions) may not be available if you execute code directly after requiring the client. For all those cases the Incubed-Client now offers a onInit-function, which also returns a promis with the result. ( his is not be needed for normale rpc-request, since they are already async)

    ```js
    import IN3 from 'in3'

    // the code will be executed as soon as the client is ready (wasm loaded)
    IN3.onInit(()=>{
        console.log(' Address :  ',  IN3.util.toChecksumAddress('0xb3b7874f13387d44a3398d298B075b7a3505d8d4'))
    })
    ```

2. **Freeing Memory**
    As every C/C++-Developer knows, in order to avoid memory leaks, you need to free memory after usage. While it may not make a big difference if you only use one instance, if you create a temp instance, make sure to call `.free()` afterwards. 

3. **Deprecated Functions**
    We removed a lot of internal functions from the Client, like
    - `getChainContext()`
    - `updateWhiteListNodes()`
    - `updateNodeList()`
    - `call()`
    - `verifyResponse()`
    - `clearStats()`
    - `addListener()`

    On the other hand we also added some useful functions, like
    - `setConfig(config: Partial<IN3Config>)` - changes (partial) configurations
    - `execLocal(method: string, params?: any[])` - to execute a rpc request synchronously directly in the client, like ( `web3_version` or )

4. **Changes in the Eth-API**
    We extended the Eth-API (`in3.eth`) and added the following functions.

    - `accounts` - the Account API for adding signer keys
    - `toWei()` - converts any value and unit to wei as hex.
    - `resolveENS()` - ENS-Resolver
    - `web3ContractAt()` - creates a instance of a Contract with the same methods as web3 would do
    - `contractAt()`- creates a instance of an contract with the same methods as ethers would do

5. **New APIs**
    - `btc`-  this API support verified Bitcoin-Responses ( we will have a seperate blogpost about it )
    - `zksync` - API for using a zksync-service


### Optimizing size

The official `in3`-module release contains different configurations you can choose from. This way you can pick a version which is just small enough to fit your needs. Currently those Versions are included:

- `require('in3')` - default, uses asmjs and includes all modules (eth,btc, zksync) - 900kB
- `require('in3/wasm')` - wasm-version which includes all modules (eth,btc, zksync) - 525kB
- `require('in3/eth1')` - Ethereum only asmjs module - 823kB
- `require('in3/eth1-wasm')` - Ethereum only wasm module - 470kB
- `require('in3/btc')` - Bitcoin - the asmjs-version with only eth_nano and btc - 656kB
- `require('in3/btc-wasm')` - Bitcoin - the wasm-version with only eth_nano and btc - 400kB
- `require('in3/zksync')` - Zksync - the asmjs-version with only eth_full, ipfs and zksync - 2.8MB
- `require('in3/zksync-wasm')` - Zksync - the wasm-version with only eth_full, ipfs and zksync - 819kB
- `require('in3/min')` - Minimal - the asmjs-version with only eth_nano - 524kB
- `require('in3/min-wasm')` - Minimal - the wasm-version with only eth_nano - 313kB

For all wasm-versions we are embedding the wasm-code as base64 directly into the code making it easier to bundle the package.


## IN3 Module

This page contains a list of all Datastructures and Classes used within the IN3 WASM-Client

Importing IN3 is as easy as
```ts
import {IN3} from "in3"
```


### BufferType and BigIntType

The WASM-Module comes with no dependencies. This means per default it uses the standard classes provided as part of the EMCAScript-Standard.

If you work with a library which expects different types, you can change the generic-type and giving a converter:

#### Type BigIntType

Per default we use `bigint`. This is used whenever we work with number too big to be stored as a `number`-type.

If you want to change this type, use [setConverBigInt()](#setconvertbigint) function.

#### Type Buffer

Per default we use `UInt8Array`. This is used whenever we work with raw bytes.

If you want to change this type, use [setConverBuffer()](#setconvertbuffer) function.

#### Generics

```js
import {IN3Generic} from 'in3'
import BN from 'bn.js'

// create a new client by setting the Generic Types
const c = new IN3Generic<BN,Buffer>()

// set the converter-functions
IN3Generic.setConverBuffer(val => Buffer.from(val))
IN3Generic.setConverBigInt(val => new BN(val))

```



### Package

While the In3Client-class is also the default import, the following imports can be used:
Signer, which stores the key encrypted in the indexDB in the browser.



#### constructor()


creates a BrowserSigner, which will use the given callback-function in order to retrieve the password, which is used to encrypt the stored keys. 

```eval_rst
`BrowserSigner <#type-browsersigner>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L737>`_ (
      passwordCB:() => `String <#type-string>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | passwordCB
       - | () => `String <#type-string>`_ 
       - | password cb

```


Returns: 
```eval_rst
`BrowserSigner <#type-browsersigner>`_ 
```



#### sign()


signing of any data.
if hashFirst is true the data should be hashed first, otherwise the data is the hash. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L754>`_ (
      data:`Hex <#type-hex>`_ ,
      account:`Address <#type-address>`_ ,
      sign_type:`SignType <#type-signtype>`_ ,
      payloadType:`SignPayload <#type-signpayload>`_ ,
      meta:``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_ 
       - | a Hexcoded String (starting with 0x)
     * - | account
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | sign_type
       - | `SignType <#type-signtype>`_ 
       - | the type of signature to create.
         |     - ec_hash : the data needs to be hashed first ( using keccak) before signing
         |     - ec_raw : the data is the ryw value (32bytes) to sign
         |     - ec_prefix : the data is a message which needs to be prefixed with the EthereumSignedMessage and length to before hashing and signing
     * - | payloadType
       - | `SignPayload <#type-signpayload>`_ 
       - | The type of the payload to sign
     * - | meta
       - | ``any``
       - | meta

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### canSign()


returns true if the account is supported (or unlocked) 

```eval_rst
``Promise<boolean>`` `canSign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L748>`_ (
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<boolean>``
```



#### generateAndStorePrivateKey()


adds a private key to the signer and returns the address associated with it. 

```eval_rst
``Promise<string>`` `generateAndStorePrivateKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L745>`_ (
      pk:`BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `BufferType <#type-buffertype>`_ 
       - | pk

```


Returns: 
```eval_rst
``Promise<string>``
```



#### getAccounts()


returns all addresses managed by the signer. 

```eval_rst
`Address <#type-address>`_  [] `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L743>`_ ()
```

Returns: 
```eval_rst
`Address <#type-address>`_  []
```

default Incubed client with
bigint for big numbers
Uint8Array for bytes

```eval_rst
  .. list-table::
     :widths: auto

     * - | `default <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L570>`_
       - | 
       - | supporting both ES6 and UMD usage 
     * - | `util <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L556>`_
       - | `Utils<any> <#type-utils>`_ 
       - | collection of util-functions. 
     * - | `btc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L540>`_
       - | `BtcAPI<Uint8Array> <#type-btcapi>`_ 
       - | btc API 
     * - | `config <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L440>`_
       - | `IN3Config <#type-in3config>`_ 
       - | IN3 config 
     * - | `eth <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L532>`_
       - | `EthAPI<bigint,Uint8Array> <#type-ethapi>`_ 
       - | eth1 API. 
     * - | `ipfs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L538>`_
       - | `IpfsAPI<Uint8Array> <#type-ipfsapi>`_ 
       - | ipfs API 
     * - | `signer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L496>`_
       - | `Signer<bigint,Uint8Array> <#type-signer>`_ 
       - | the signer, if specified this interface will be used to sign transactions, if not, sending transaction will not be possible. 
     * - | `util <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L545>`_
       - | `Utils<Uint8Array> <#type-utils>`_ 
       - | collection of util-functions. 
     * - | `zksync <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L536>`_
       - | `ZksyncAPI<Uint8Array> <#type-zksyncapi>`_ 
       - | zksync API 

```


#### freeAll()


frees all Incubed instances. 

```eval_rst
static ``void`` `freeAll <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L526>`_ ()
```



#### onInit()


registers a function to be called as soon as the wasm is ready.
If it is already initialized it will call it right away. 

```eval_rst
static `Promise<T> <#type-t>`_  `onInit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L520>`_ (
      fn:() => `T <#type-t>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | fn
       - | () => `T <#type-t>`_ 
       - | the function to call

```


Returns: 
```eval_rst
static `Promise<T> <#type-t>`_ 
```



#### setConvertBigInt()


sets the convert-function, which converts any kind of type to Type defined for BigInt-operation.
if not set the default type would be bigint. 

```eval_rst
static ``void`` `setConvertBigInt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L590>`_ (
      convert:(``any``) => ``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | convert
       - | (``any``) => ``any``
       - | convert

```




#### setConvertBuffer()


sets the convert-function, which converts any kind of type to Type defined for Buffer or Bytes-operation.
if not set the default type would be UInt8Array. 

```eval_rst
static ``void`` `setConvertBuffer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L596>`_ (
      convert:(``any``) => ``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | convert
       - | (``any``) => ``any``
       - | convert

```




#### setStorage()


changes the storage handler, which is called to read and write to the cache. 

```eval_rst
static ``void`` `setStorage <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L509>`_ (
      handler:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | handler
       - | 
       - | handler

```




#### setTransport()


changes the default transport-function. 

```eval_rst
static ``void`` `setTransport <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L504>`_ (
      fn:(``string`` , ``string`` , ``number``) => ``Promise<string>``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | fn
       - | (``string`` , ``string`` , ``number``) => ``Promise<string>``
       - | the function to fetch the response for the given url
         | 

```




#### constructor()


creates a new client. 

```eval_rst
`IN3 <#type-in3>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L578>`_ (
      config:`Partial<IN3Config> <#type-in3config>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | config
       - | `Partial<IN3Config> <#type-in3config>`_ 
       - | a optional config
         | 

```


Returns: 
```eval_rst
`IN3 <#type-in3>`_ 
```



#### createWeb3Provider()


returns a Object, which can be used as Web3Provider.

```
const web3 = new Web3(new IN3().createWeb3Provider())
```
 

```eval_rst
``any`` `createWeb3Provider <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L491>`_ ()
```

Returns: 
```eval_rst
``any``
```



#### execLocal()


sends a RPC-Requests specified by name and params as a sync call. This is only alowed if the request is handled internally, like web3_sha3,

if the response contains an error, this will be thrown. if not the result will be returned.
 

```eval_rst
``any`` `execLocal <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L477>`_ (
      method:``string``,
      params:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | method
       - | ``string``
       - | the method to call.
         | 
     * - | params
       - | ``any`` []
       - | params

```


Returns: 
```eval_rst
``any``
```



#### free()


disposes the Client. This must be called in order to free allocated memory! 

```eval_rst
``void`` `free <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L482>`_ ()
```



#### registerPlugin()


rregisters a plugin. The plugin may define methods which will be called by the client. 

```eval_rst
``void`` `registerPlugin <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L551>`_ (
      plugin:`IN3Plugin<bigint,Uint8Array> <#type-in3plugin>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | plugin
       - | `IN3Plugin<bigint,Uint8Array> <#type-in3plugin>`_ 
       - | the plugin-object to register
         | 

```




#### send()


sends a raw request.
if the request is a array the response will be a array as well.
If the callback is given it will be called with the response, if not a Promise will be returned.
This function supports callback so it can be used as a Provider for the web3. 

```eval_rst
`Promise<RPCResponse> <#type-rpcresponse>`_  `send <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L458>`_ (
      request:`RPCRequest <#type-rpcrequest>`_ ,
      callback:(`Error <#type-error>`_  , `RPCResponse <#type-rpcresponse>`_ ) => ``void``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | request
       - | `RPCRequest <#type-rpcrequest>`_ 
       - | a JSONRPC-Request with N3-Extension
     * - | callback
       - | (`Error <#type-error>`_  , `RPCResponse <#type-rpcresponse>`_ ) => ``void``
       - | callback

```


Returns: 
```eval_rst
`Promise<RPCResponse> <#type-rpcresponse>`_ 
```



#### sendRPC()


sends a RPC-Requests specified by name and params.

if the response contains an error, this will be thrown. if not the result will be returned.
 

```eval_rst
``Promise<any>`` `sendRPC <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L467>`_ (
      method:``string``,
      params:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | method
       - | ``string``
       - | the method to call.
         | 
     * - | params
       - | ``any`` []
       - | params

```


Returns: 
```eval_rst
``Promise<any>``
```



#### setConfig()


sets configuration properties. You can pass a partial object specifieing any of defined properties. 

```eval_rst
``void`` `setConfig <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L450>`_ (
      config:`Partial<IN3Config> <#type-in3config>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | config
       - | `Partial<IN3Config> <#type-in3config>`_ 
       - | config

```




```eval_rst
  .. list-table::
     :widths: auto

     * - | `default <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L570>`_
       - | 
       - | supporting both ES6 and UMD usage 
     * - | `util <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L556>`_
       - | `Utils<any> <#type-utils>`_ 
       - | collection of util-functions. 
     * - | `btc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L540>`_
       - | `BtcAPI<BufferType> <#type-btcapi>`_ 
       - | btc API 
     * - | `config <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L440>`_
       - | `IN3Config <#type-in3config>`_ 
       - | IN3 config 
     * - | `eth <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L532>`_
       - | `EthAPI<BigIntType,BufferType> <#type-ethapi>`_ 
       - | eth1 API. 
     * - | `ipfs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L538>`_
       - | `IpfsAPI<BufferType> <#type-ipfsapi>`_ 
       - | ipfs API 
     * - | `signer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L496>`_
       - | `Signer<BigIntType,BufferType> <#type-signer>`_ 
       - | the signer, if specified this interface will be used to sign transactions, if not, sending transaction will not be possible. 
     * - | `util <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L545>`_
       - | `Utils<BufferType> <#type-utils>`_ 
       - | collection of util-functions. 
     * - | `zksync <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L536>`_
       - | `ZksyncAPI<BufferType> <#type-zksyncapi>`_ 
       - | zksync API 

```


#### freeAll()


frees all Incubed instances. 

```eval_rst
static ``void`` `freeAll <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L526>`_ ()
```



#### onInit()


registers a function to be called as soon as the wasm is ready.
If it is already initialized it will call it right away. 

```eval_rst
static `Promise<T> <#type-t>`_  `onInit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L520>`_ (
      fn:() => `T <#type-t>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | fn
       - | () => `T <#type-t>`_ 
       - | the function to call

```


Returns: 
```eval_rst
static `Promise<T> <#type-t>`_ 
```



#### setConvertBigInt()


sets the convert-function, which converts any kind of type to Type defined for BigInt-operation.
if not set the default type would be bigint. 

```eval_rst
static ``void`` `setConvertBigInt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L561>`_ (
      convert:(``any``) => ``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | convert
       - | (``any``) => ``any``
       - | convert

```




#### setConvertBuffer()


sets the convert-function, which converts any kind of type to Type defined for Buffer or Bytes-operation.
if not set the default type would be UInt8Array. 

```eval_rst
static ``void`` `setConvertBuffer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L567>`_ (
      convert:(``any``) => ``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | convert
       - | (``any``) => ``any``
       - | convert

```




#### setStorage()


changes the storage handler, which is called to read and write to the cache. 

```eval_rst
static ``void`` `setStorage <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L509>`_ (
      handler:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | handler
       - | 
       - | handler

```




#### setTransport()


changes the default transport-function. 

```eval_rst
static ``void`` `setTransport <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L504>`_ (
      fn:(``string`` , ``string`` , ``number``) => ``Promise<string>``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | fn
       - | (``string`` , ``string`` , ``number``) => ``Promise<string>``
       - | the function to fetch the response for the given url
         | 

```




#### constructor()


creates a new client. 

```eval_rst
`IN3Generic <#type-in3generic>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L440>`_ (
      config:`Partial<IN3Config> <#type-in3config>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | config
       - | `Partial<IN3Config> <#type-in3config>`_ 
       - | a optional config
         | 

```


Returns: 
```eval_rst
`IN3Generic <#type-in3generic>`_ 
```



#### createWeb3Provider()


returns a Object, which can be used as Web3Provider.

```
const web3 = new Web3(new IN3().createWeb3Provider())
```
 

```eval_rst
``any`` `createWeb3Provider <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L491>`_ ()
```

Returns: 
```eval_rst
``any``
```



#### execLocal()


sends a RPC-Requests specified by name and params as a sync call. This is only alowed if the request is handled internally, like web3_sha3,

if the response contains an error, this will be thrown. if not the result will be returned.
 

```eval_rst
``any`` `execLocal <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L477>`_ (
      method:``string``,
      params:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | method
       - | ``string``
       - | the method to call.
         | 
     * - | params
       - | ``any`` []
       - | params

```


Returns: 
```eval_rst
``any``
```



#### free()


disposes the Client. This must be called in order to free allocated memory! 

```eval_rst
``void`` `free <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L482>`_ ()
```



#### registerPlugin()


rregisters a plugin. The plugin may define methods which will be called by the client. 

```eval_rst
``void`` `registerPlugin <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L551>`_ (
      plugin:`IN3Plugin<BigIntType,BufferType> <#type-in3plugin>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | plugin
       - | `IN3Plugin<BigIntType,BufferType> <#type-in3plugin>`_ 
       - | the plugin-object to register
         | 

```




#### send()


sends a raw request.
if the request is a array the response will be a array as well.
If the callback is given it will be called with the response, if not a Promise will be returned.
This function supports callback so it can be used as a Provider for the web3. 

```eval_rst
`Promise<RPCResponse> <#type-rpcresponse>`_  `send <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L458>`_ (
      request:`RPCRequest <#type-rpcrequest>`_ ,
      callback:(`Error <#type-error>`_  , `RPCResponse <#type-rpcresponse>`_ ) => ``void``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | request
       - | `RPCRequest <#type-rpcrequest>`_ 
       - | a JSONRPC-Request with N3-Extension
     * - | callback
       - | (`Error <#type-error>`_  , `RPCResponse <#type-rpcresponse>`_ ) => ``void``
       - | callback

```


Returns: 
```eval_rst
`Promise<RPCResponse> <#type-rpcresponse>`_ 
```



#### sendRPC()


sends a RPC-Requests specified by name and params.

if the response contains an error, this will be thrown. if not the result will be returned.
 

```eval_rst
``Promise<any>`` `sendRPC <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L467>`_ (
      method:``string``,
      params:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | method
       - | ``string``
       - | the method to call.
         | 
     * - | params
       - | ``any`` []
       - | params

```


Returns: 
```eval_rst
``Promise<any>``
```



#### setConfig()


sets configuration properties. You can pass a partial object specifieing any of defined properties. 

```eval_rst
``void`` `setConfig <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L450>`_ (
      config:`Partial<IN3Config> <#type-in3config>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | config
       - | `Partial<IN3Config> <#type-in3config>`_ 
       - | config

```




```eval_rst
  .. list-table::
     :widths: auto

     * - | `accounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L713>`_
       - | 
       - | the accounts 

```


#### constructor()


constructor 

```eval_rst
`SimpleSigner <#type-simplesigner>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L715>`_ (
      pks:`Hash <#type-hash>`_  | `BufferType <#type-buffertype>`_  [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pks
       - | `Hash <#type-hash>`_  
         | | `BufferType <#type-buffertype>`_  []
       - | pks

```


Returns: 
```eval_rst
`SimpleSigner <#type-simplesigner>`_ 
```



#### sign()


signing of any data.
if hashFirst is true the data should be hashed first, otherwise the data is the hash. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L730>`_ (
      data:`Hex <#type-hex>`_ ,
      account:`Address <#type-address>`_ ,
      sign_type:`SignType <#type-signtype>`_ ,
      payloadType:`SignPayload <#type-signpayload>`_ ,
      meta:``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_ 
       - | a Hexcoded String (starting with 0x)
     * - | account
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | sign_type
       - | `SignType <#type-signtype>`_ 
       - | the type of signature to create.
         |     - ec_hash : the data needs to be hashed first ( using keccak) before signing
         |     - ec_raw : the data is the ryw value (32bytes) to sign
         |     - ec_prefix : the data is a message which needs to be prefixed with the EthereumSignedMessage and length to before hashing and signing
     * - | payloadType
       - | `SignPayload <#type-signpayload>`_ 
       - | The type of the payload to sign
     * - | meta
       - | ``any``
       - | meta

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### addAccount()


adds a private key to the signer and returns the address associated with it. 

```eval_rst
``string`` `addAccount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L721>`_ (
      pk:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``string``
```



#### canSign()


returns true if the account is supported (or unlocked) 

```eval_rst
``Promise<boolean>`` `canSign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L724>`_ (
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<boolean>``
```



#### getAccounts()


returns all addresses managed by the signer. 

```eval_rst
`Address <#type-address>`_  [] `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L719>`_ ()
```

Returns: 
```eval_rst
`Address <#type-address>`_  []
```

The Account API



#### add()


adds a private key to sign with.
This method returns address of the pk 

```eval_rst
``string`` `add <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1464>`_ (
      pk:``string`` | `BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | 
         | 

```


Returns: 
```eval_rst
``string``
```



#### addKeyStore()


adds a key from a JSON Keystore file as defined in the Web3 Secret Storage Definition . This method returns address of the pk. 

```eval_rst
`String <#type-string>`_  `addKeyStore <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1478>`_ (
      keystore:``any``,
      passphrase:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | keystore
       - | ``any``
       - | the keystore data
     * - | passphrase
       - | ``string``
       - | the passphrase
         | 

```


Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### decryptKeystore()


decrypts a JSON Keystore file as defined in the Web3 Secret Storage Definition . The result is the raw private key. 

```eval_rst
`BufferType <#type-buffertype>`_  `decryptKeystore <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1471>`_ (
      keystore:``any``,
      passphrase:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | keystore
       - | ``any``
       - | the keystore data
     * - | passphrase
       - | ``string``
       - | the passphrase
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### ecrecover()


recovers a ecdsa signature. 

```eval_rst
 `ecrecover <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1486>`_ (
      msg:``string`` | `BufferType <#type-buffertype>`_ ,
      sig:``string`` | `BufferType <#type-buffertype>`_ ,
      msgtype:``'eth_sign'`` | ``'raw'`` | ``'hash'``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the message
     * - | sig
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the signature (must be 65bytes in hex or Buffer)
     * - | msgtype
       - | ``'eth_sign'`` | ``'raw'`` | ``'hash'``
       - | the type (raw : the message must be hashed first (default), 'hash' the message is already a 32byte hash, 'eth_sign' - the message will be hashed with EthereumSignedMessage-Prefix)
         | 

```




#### prepareTx()


prepares a Transaction by creating a unsigned raw transaction. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `prepareTx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1512>`_ (
      tx:`TxRequest <#type-txrequest>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tx
       - | `TxRequest <#type-txrequest>`_ 
       - | the tx parameter
         | 

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### signData()


creates a signature with a previously registered signer based on the data 

```eval_rst
`Promise<Signature> <#type-signature>`_  `signData <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1497>`_ (
      msg:``string`` | `BufferType <#type-buffertype>`_ ,
      account:``string`` | `BufferType <#type-buffertype>`_ ,
      msgtype:``'eth_sign'`` | ``'raw'`` | ``'hash'``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the message to sign
     * - | account
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the address of the account ( if null, the first available account is used ). If the account is a 32byte hex or buffer, it it will be used as raw private key.
     * - | msgtype
       - | ``'eth_sign'`` | ``'raw'`` | ``'hash'``
       - | the type (raw : the message must be hashed first (default), 'hash' the message is already a 32byte hash, 'eth_sign' - the message will be hashed with EthereumSignedMessage-Prefix)
         | 

```


Returns: 
```eval_rst
`Promise<Signature> <#type-signature>`_ 
```



#### signRawTx()


creates a signature with a previously registered signer based on the data 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `signRawTx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1506>`_ (
      msg:``string`` | `BufferType <#type-buffertype>`_ ,
      account:``string`` | `BufferType <#type-buffertype>`_ ,
      msgtype:``'eth_sign'`` | ``'raw'`` | ``'hash'``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the message to sign
     * - | account
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the address of the account ( if null, the first available account is used ). If the account is a 32byte hex or buffer, it it will be used as raw private key.
     * - | msgtype
       - | ``'eth_sign'`` | ``'raw'`` | ``'hash'``
       - | the type (raw : the message must be hashed first (default), 'hash' the message is already a 32byte hash, 'eth_sign' - the message will be hashed with EthereumSignedMessage-Prefix)
         | 

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```

a full Block including the transactions

```eval_rst
  .. list-table::
     :widths: auto

     * - | `bits <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2060>`_
       - | ``string``
       - | bits (target) for the block as hex 
     * - | `chainwork <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2064>`_
       - | ``string``
       - | total amount of work since genesis 
     * - | `confirmations <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2044>`_
       - | ``number``
       - | number of confirmations or blocks mined on top of the containing block 
     * - | `difficulty <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2062>`_
       - | ``number``
       - | difficulty of the block 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2042>`_
       - | ``string``
       - | the hash of the blockheader 
     * - | `height <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2046>`_
       - | ``number``
       - | block number 
     * - | `mediantime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2056>`_
       - | ``string``
       - | unix timestamp in seconds since 1970 
     * - | `merkleroot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2052>`_
       - | ``string``
       - | merkle root of the trie of all transactions in the block 
     * - | `nTx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2066>`_
       - | ``number``
       - | number of transactions in the block 
     * - | `nextblockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2070>`_
       - | ``string``
       - | hash of the next blockheader 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2058>`_
       - | ``number``
       - | nonce-field of the block 
     * - | `previousblockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2068>`_
       - | ``string``
       - | hash of the parent blockheader 
     * - | `time <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2054>`_
       - | ``string``
       - | unix timestamp in seconds since 1970 
     * - | `tx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2076>`_
       - | `T <#type-t>`_  []
       - | the transactions 
     * - | `version <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2048>`_
       - | ``number``
       - | used version 
     * - | `versionHex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2050>`_
       - | ``string``
       - | version as hex 

```
a Block header

```eval_rst
  .. list-table::
     :widths: auto

     * - | `bits <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2060>`_
       - | ``string``
       - | bits (target) for the block as hex 
     * - | `chainwork <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2064>`_
       - | ``string``
       - | total amount of work since genesis 
     * - | `confirmations <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2044>`_
       - | ``number``
       - | number of confirmations or blocks mined on top of the containing block 
     * - | `difficulty <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2062>`_
       - | ``number``
       - | difficulty of the block 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2042>`_
       - | ``string``
       - | the hash of the blockheader 
     * - | `height <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2046>`_
       - | ``number``
       - | block number 
     * - | `mediantime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2056>`_
       - | ``string``
       - | unix timestamp in seconds since 1970 
     * - | `merkleroot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2052>`_
       - | ``string``
       - | merkle root of the trie of all transactions in the block 
     * - | `nTx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2066>`_
       - | ``number``
       - | number of transactions in the block 
     * - | `nextblockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2070>`_
       - | ``string``
       - | hash of the next blockheader 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2058>`_
       - | ``number``
       - | nonce-field of the block 
     * - | `previousblockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2068>`_
       - | ``string``
       - | hash of the parent blockheader 
     * - | `time <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2054>`_
       - | ``string``
       - | unix timestamp in seconds since 1970 
     * - | `version <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2048>`_
       - | ``number``
       - | used version 
     * - | `versionHex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2050>`_
       - | ``string``
       - | version as hex 

```
Block state.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L951>`_
       - | ``number``
       - | the blockNumber 
     * - | `committed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L952>`_
       - | ``boolean``
       - | the committed 
     * - | `verified <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L953>`_
       - | ``boolean``
       - | the verified 

```
API for handling BitCoin data



#### getBlockBytes()


retrieves the serialized block (bytes) including all transactions 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `getBlockBytes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2103>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### getBlockCount()


Returns the number of blocks in the longest blockchain. 

```eval_rst
`Promise<Number> <#type-number>`_  `getBlockCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2106>`_ ()
```

Returns: 
```eval_rst
`Promise<Number> <#type-number>`_ 
```



#### getBlockHeader()


retrieves the blockheader and returns the data as json. 

```eval_rst
`Promise<BTCBlockHeader> <#type-btcblockheader>`_  `getBlockHeader <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2091>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BTCBlockHeader> <#type-btcblockheader>`_ 
```



#### getBlockHeaderBytes()


retrieves the serialized blockheader (bytes) 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `getBlockHeaderBytes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2094>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### getBlockWithTxData()


retrieves the block including all tx data as json. 

```eval_rst
`Promise<BTCBlock> <#type-btcblock>`_  `getBlockWithTxData <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2097>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BTCBlock> <#type-btcblock>`_ 
```



#### getBlockWithTxIds()


retrieves the block including all tx ids as json. 

```eval_rst
`Promise<BTCBlock> <#type-btcblock>`_  `getBlockWithTxIds <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2100>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BTCBlock> <#type-btcblock>`_ 
```



#### getTransaction()


retrieves the transaction and returns the data as json. 

```eval_rst
`Promise<BtcTransaction> <#type-btctransaction>`_  `getTransaction <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2085>`_ (
      txid:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | txid
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BtcTransaction> <#type-btctransaction>`_ 
```



#### getTransactionBytes()


retrieves the serialized transaction (bytes) 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `getTransactionBytes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2088>`_ (
      txid:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | txid
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```

a BitCoin Transaction.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `blockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2021>`_
       - | `Hash <#type-hash>`_ 
       - | the block hash of the block containing this transaction. 
     * - | `blocktime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2030>`_
       - | ``number``
       - | The block time in seconds since epoch (Jan 1 1970 GMT) 
     * - | `confirmations <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2024>`_
       - | ``number``
       - | The confirmations. 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2003>`_
       - | `Hash <#type-hash>`_ 
       - | The transaction hash (differs from txid for witness transactions) 
     * - | `hex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1997>`_
       - | `Data <#type-data>`_ 
       - | the hex representation of raw data 
     * - | `in_active_chain <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1994>`_
       - | ``boolean``
       - | true if this transaction is part of the longest chain 
     * - | `locktime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2018>`_
       - | ``number``
       - | The locktime 
     * - | `size <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2006>`_
       - | ``number``
       - | The serialized transaction size 
     * - | `time <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2027>`_
       - | ``number``
       - | The transaction time in seconds since epoch (Jan 1 1970 GMT) 
     * - | `txid <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2000>`_
       - | `Hash <#type-hash>`_ 
       - | The requested transaction id. 
     * - | `version <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2015>`_
       - | ``number``
       - | The version 
     * - | `vin <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2033>`_
       - | `BtcTransactionInput <#type-btctransactioninput>`_  []
       - | the transaction inputs 
     * - | `vout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2036>`_
       - | `BtcTransactionOutput <#type-btctransactionoutput>`_  []
       - | the transaction outputs 
     * - | `vsize <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2009>`_
       - | ``number``
       - | The virtual transaction size (differs from size for witness transactions) 
     * - | `weight <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2012>`_
       - | ``number``
       - | The transaction’s weight (between vsize4-3 and vsize4) 

```
a Input of a Bitcoin Transaction

```eval_rst
  .. list-table::
     :widths: auto

     * - | `scriptSig <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1943>`_
       - | 
       - | the script 
     * - | `sequence <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1952>`_
       - | ``number``
       - | The script sequence number 
     * - | `txid <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1937>`_
       - | `Hash <#type-hash>`_ 
       - | the transaction id 
     * - | `txinwitness <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1955>`_
       - | `Data <#type-data>`_  []
       - | hex-encoded witness data (if any) 
     * - | `vout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1940>`_
       - | ``number``
       - | the index of the transactionoutput 

```
a Input of a Bitcoin Transaction

```eval_rst
  .. list-table::
     :widths: auto

     * - | `n <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1965>`_
       - | ``number``
       - | the index 
     * - | `scriptPubKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1971>`_
       - | 
       - | the script 
     * - | `value <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1962>`_
       - | ``number``
       - | the value in BTC 
     * - | `vout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1968>`_
       - | ``number``
       - | the index of the transactionoutput 

```
Response of a Deposit-Transaction.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `receipt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1011>`_
       - | `TransactionReceipt <#type-transactionreceipt>`_ 
       - | the receipt 

```
L1 Operation State

```eval_rst
  .. list-table::
     :widths: auto

     * - | `block <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L971>`_
       - | `BlockInfo <#type-blockinfo>`_ 
       - | the block 
     * - | `executed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L970>`_
       - | ``boolean``
       - | the executed 

```
The API for ethereum operations.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `accounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1601>`_
       - | `AccountAPI<BufferType> <#type-accountapi>`_ 
       - | accounts-API 
     * - | `client <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1591>`_
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | the client used. 
     * - | `signer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1596>`_
       - | `Signer<BigIntType,BufferType> <#type-signer>`_ 
       - | a custom signer  *(optional)* 

```


#### blockNumber()


Returns the number of most recent block. (as number) 

```eval_rst
``Promise<number>`` `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1607>`_ ()
```

Returns: 
```eval_rst
``Promise<number>``
```



#### call()


Executes a new message call immediately without creating a transaction on the block chain. 

```eval_rst
``Promise<string>`` `call <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1615>`_ (
      tx:`Transaction <#type-transaction>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tx
       - | `Transaction <#type-transaction>`_ 
       - | tx
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<string>``
```



#### callFn()


Executes a function of a contract, by passing a [method-signature](https://github.com/ethereumjs/ethereumjs-abi/blob/master/README.md#simple-encoding-and-decoding) and the arguments, which will then be ABI-encoded and send as eth_call. 

```eval_rst
``Promise<any>`` `callFn <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1619>`_ (
      to:`Address <#type-address>`_ ,
      method:``string``,
      args:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | to
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | method
       - | ``string``
       - | method
     * - | args
       - | ``any`` []
       - | args

```


Returns: 
```eval_rst
``Promise<any>``
```



#### chainId()


Returns the EIP155 chain ID used for transaction signing at the current best block. Null is returned if not available. 

```eval_rst
``Promise<string>`` `chainId <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1623>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### clientVersion()


Returns the clientVersion. This may differ in case of an network, depending on the node it communicates with. 

```eval_rst
``Promise<string>`` `clientVersion <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1627>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### constructor()


The API for ethereum operations. 

```eval_rst
`EthAPI<BigIntType,BufferType> <#type-ethapi>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1603>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | client

```


Returns: 
```eval_rst
`EthAPI<BigIntType,BufferType> <#type-ethapi>`_ 
```



#### contractAt()


contract at 

```eval_rst
 `contractAt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1802>`_ (
      abi:`ABI <#type-abi>`_  [],
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | abi
       - | `ABI <#type-abi>`_  []
       - | abi
     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)

```




#### decodeEventData()


decode event data 

```eval_rst
``any`` `decodeEventData <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1842>`_ (
      log:`Log <#type-log>`_ ,
      d:`ABI <#type-abi>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | log
       - | `Log <#type-log>`_ 
       - | log
     * - | d
       - | `ABI <#type-abi>`_ 
       - | d

```


Returns: 
```eval_rst
``any``
```



#### estimateGas()


Makes a call or transaction, which won’t be added to the blockchain and returns the used gas, which can be used for estimating the used gas. 

```eval_rst
``Promise<number>`` `estimateGas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1631>`_ (
      tx:`Transaction <#type-transaction>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tx
       - | `Transaction <#type-transaction>`_ 
       - | tx

```


Returns: 
```eval_rst
``Promise<number>``
```



#### fromWei()


Returns a formated String in the specified unit (or eth if not specified). If digits are specified, the number of digits behind the comma can be limited. 

```eval_rst
``string`` `fromWei <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1761>`_ (
      value:``string``,
      unit:``string``,
      digits:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | value
       - | ``string``
       - | value
     * - | unit
       - | ``string``
       - | unit
     * - | digits
       - | ``number``
       - | digits

```


Returns: 
```eval_rst
``string``
```



#### gasPrice()


Returns the current price per gas in wei. (as number) 

```eval_rst
``Promise<number>`` `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1611>`_ ()
```

Returns: 
```eval_rst
``Promise<number>``
```



#### getAccounts()


returns the public addresses  accounts 

```eval_rst
``Promise<>`` `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1692>`_ ()
```

Returns: 
```eval_rst
``Promise<>``
```



#### getBalance()


Returns the balance of the account of given address in wei (as hex). 

```eval_rst
`Promise<BigIntType> <#type-biginttype>`_  `getBalance <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1635>`_ (
      address:`Address <#type-address>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
`Promise<BigIntType> <#type-biginttype>`_ 
```



#### getBlockByHash()


Returns information about a block by hash. 

```eval_rst
`Promise<Block> <#type-block>`_  `getBlockByHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1647>`_ (
      hash:`Hash <#type-hash>`_ ,
      includeTransactions:``boolean``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)
     * - | includeTransactions
       - | ``boolean``
       - | include transactions

```


Returns: 
```eval_rst
`Promise<Block> <#type-block>`_ 
```



#### getBlockByNumber()


Returns information about a block by block number. 

```eval_rst
`Promise<Block> <#type-block>`_  `getBlockByNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1651>`_ (
      block:`BlockType <#type-blocktype>`_ ,
      includeTransactions:``boolean``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block
     * - | includeTransactions
       - | ``boolean``
       - | include transactions

```


Returns: 
```eval_rst
`Promise<Block> <#type-block>`_ 
```



#### getBlockTransactionCountByHash()


Returns the number of transactions in a block from a block matching the given block hash. 

```eval_rst
``Promise<number>`` `getBlockTransactionCountByHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1655>`_ (
      block:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<number>``
```



#### getBlockTransactionCountByNumber()


Returns the number of transactions in a block from a block matching the given block number. 

```eval_rst
``Promise<number>`` `getBlockTransactionCountByNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1659>`_ (
      block:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<number>``
```



#### getCode()


Returns code at a given address. 

```eval_rst
``Promise<string>`` `getCode <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1639>`_ (
      address:`Address <#type-address>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<string>``
```



#### getFilterChanges()


Polling method for a filter, which returns an array of logs which occurred since last poll. 

```eval_rst
``Promise<>`` `getFilterChanges <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1663>`_ (
      id:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | id
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
``Promise<>``
```



#### getFilterLogs()


Returns an array of all logs matching filter with given id. 

```eval_rst
``Promise<>`` `getFilterLogs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1667>`_ (
      id:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | id
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
``Promise<>``
```



#### getLogs()


Returns an array of all logs matching a given filter object. 

```eval_rst
``Promise<>`` `getLogs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1671>`_ (
      filter:`LogFilter <#type-logfilter>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | filter
       - | `LogFilter <#type-logfilter>`_ 
       - | filter

```


Returns: 
```eval_rst
``Promise<>``
```



#### getStorageAt()


Returns the value from a storage position at a given address. 

```eval_rst
``Promise<string>`` `getStorageAt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1643>`_ (
      address:`Address <#type-address>`_ ,
      pos:`Quantity <#type-quantity>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<string>``
```



#### getTransactionByBlockHashAndIndex()


Returns information about a transaction by block hash and transaction index position. 

```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_  `getTransactionByBlockHashAndIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1675>`_ (
      hash:`Hash <#type-hash>`_ ,
      pos:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_ 
```



#### getTransactionByBlockNumberAndIndex()


Returns information about a transaction by block number and transaction index position. 

```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_  `getTransactionByBlockNumberAndIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1679>`_ (
      block:`BlockType <#type-blocktype>`_ ,
      pos:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_ 
```



#### getTransactionByHash()


Returns the information about a transaction requested by transaction hash. 

```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_  `getTransactionByHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1683>`_ (
      hash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_ 
```



#### getTransactionCount()


Returns the number of transactions sent from an address. (as number) 

```eval_rst
``Promise<number>`` `getTransactionCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1687>`_ (
      address:`Address <#type-address>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<number>``
```



#### getTransactionReceipt()


Returns the receipt of a transaction by transaction hash.
Note That the receipt is available even for pending transactions. 

```eval_rst
`Promise<TransactionReceipt> <#type-transactionreceipt>`_  `getTransactionReceipt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1698>`_ (
      hash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<TransactionReceipt> <#type-transactionreceipt>`_ 
```



#### getUncleByBlockHashAndIndex()


Returns information about a uncle of a block by hash and uncle index position.
Note: An uncle doesn’t contain individual transactions. 

```eval_rst
`Promise<Block> <#type-block>`_  `getUncleByBlockHashAndIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1703>`_ (
      hash:`Hash <#type-hash>`_ ,
      pos:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<Block> <#type-block>`_ 
```



#### getUncleByBlockNumberAndIndex()


Returns information about a uncle of a block number and uncle index position.
Note: An uncle doesn’t contain individual transactions. 

```eval_rst
`Promise<Block> <#type-block>`_  `getUncleByBlockNumberAndIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1708>`_ (
      block:`BlockType <#type-blocktype>`_ ,
      pos:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<Block> <#type-block>`_ 
```



#### getUncleCountByBlockHash()


Returns the number of uncles in a block from a block matching the given block hash. 

```eval_rst
``Promise<number>`` `getUncleCountByBlockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1712>`_ (
      hash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<number>``
```



#### getUncleCountByBlockNumber()


Returns the number of uncles in a block from a block matching the given block hash. 

```eval_rst
``Promise<number>`` `getUncleCountByBlockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1716>`_ (
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<number>``
```



#### hashMessage()


a Hexcoded String (starting with 0x) 

```eval_rst
`Hex <#type-hex>`_  `hashMessage <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1843>`_ (
      data:`Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Data <#type-data>`_ 
       - | data encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### newBlockFilter()


Creates a filter in the node, to notify when a new block arrives. To check if the state has changed, call eth_getFilterChanges. 

```eval_rst
``Promise<string>`` `newBlockFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1726>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### newFilter()


Creates a filter object, based on filter options, to notify when the state changes (logs). To check if the state has changed, call eth_getFilterChanges.

A note on specifying topic filters:
Topics are order-dependent. A transaction with a log with topics [A, B] will be matched by the following topic filters:

[] “anything”
[A] “A in first position (and anything after)”
[null, B] “anything in first position AND B in second position (and anything after)”
[A, B] “A in first position AND B in second position (and anything after)”
[[A, B], [A, B]] “(A OR B) in first position AND (A OR B) in second position (and anything after)”
 

```eval_rst
``Promise<string>`` `newFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1739>`_ (
      filter:`LogFilter <#type-logfilter>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | filter
       - | `LogFilter <#type-logfilter>`_ 
       - | filter

```


Returns: 
```eval_rst
``Promise<string>``
```



#### newPendingFilter()


adds a filter for pending transaction (only available for local rpc) 

```eval_rst
``Promise<string>`` `newPendingFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1721>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### newPendingTransactionFilter()


Creates a filter in the node, to notify when new pending transactions arrive.

To check if the state has changed, call eth_getFilterChanges.
 

```eval_rst
``Promise<string>`` `newPendingTransactionFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1745>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### protocolVersion()


Returns the current ethereum protocol version. 

```eval_rst
``Promise<string>`` `protocolVersion <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1753>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### resolveENS()


resolves a name as an ENS-Domain. 

```eval_rst
`Promise<Address> <#type-address>`_  `resolveENS <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1780>`_ (
      name:``string``,
      type:`Address <#type-address>`_ ,
      registry:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | name
       - | ``string``
       - | the domain name
     * - | type
       - | `Address <#type-address>`_ 
       - | the type (currently only addr is supported)
     * - | registry
       - | ``string``
       - | optionally the address of the registry (default is the mainnet ens registry)
         | 

```


Returns: 
```eval_rst
`Promise<Address> <#type-address>`_ 
```



#### sendRawTransaction()


Creates new message call transaction or a contract creation for signed transactions. 

```eval_rst
``Promise<string>`` `sendRawTransaction <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1785>`_ (
      data:`Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Data <#type-data>`_ 
       - | data encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<string>``
```



#### sendTransaction()


sends a Transaction 

```eval_rst
``Promise<>`` `sendTransaction <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1793>`_ (
      args:`TxRequest <#type-txrequest>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | args
       - | `TxRequest <#type-txrequest>`_ 
       - | args

```


Returns: 
```eval_rst
``Promise<>``
```



#### sign()


signs any kind of message using the `\x19Ethereum Signed Message:\n`-prefix 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1791>`_ (
      account:`Address <#type-address>`_ ,
      data:`Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | account
       - | `Address <#type-address>`_ 
       - | the address to sign the message with (if this is a 32-bytes hex-string it will be used as private key)
     * - | data
       - | `Data <#type-data>`_ 
       - | the data to sign (Buffer, hexstring or utf8-string)
         | 

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### syncing()


Returns the state of the underlying node. 

```eval_rst
``Promise<>`` `syncing <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1765>`_ ()
```

Returns: 
```eval_rst
``Promise<>``
```



#### toWei()


Returns the value in wei as hexstring. 

```eval_rst
``string`` `toWei <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1757>`_ (
      value:``string``,
      unit:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | value
       - | ``string``
       - | value
     * - | unit
       - | ``string``
       - | unit

```


Returns: 
```eval_rst
``string``
```



#### uninstallFilter()


Uninstalls a filter with given id. Should always be called when watch is no longer needed. Additonally Filters timeout when they aren’t requested with eth_getFilterChanges for a period of time. 

```eval_rst
`Promise<Quantity> <#type-quantity>`_  `uninstallFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1749>`_ (
      id:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | id
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<Quantity> <#type-quantity>`_ 
```



#### web3ContractAt()


web3 contract at 

```eval_rst
`Web3Contract <#type-web3contract>`_  `web3ContractAt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1795>`_ (
      abi:`ABI <#type-abi>`_  [],
      address:`Address <#type-address>`_ ,
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | abi
       - | `ABI <#type-abi>`_  []
       - | abi
     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
`Web3Contract <#type-web3contract>`_ 
```

fee collection.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `feeType <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L985>`_
       - | `TxType <#type-txtype>`_ 
       - | the feeType 
     * - | `gasFee <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L988>`_
       - | ``number``
       - | the gasFee 
     * - | `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L987>`_
       - | ``number``
       - | the gasPrice 
     * - | `totalFee <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L990>`_
       - | ``number``
       - | the totalFee 
     * - | `totalGas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L986>`_
       - | ``number``
       - | the totalGas 
     * - | `zkpFee <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L989>`_
       - | ``number``
       - | the zkpFee 

```
the configuration of the IN3-Client. This can be changed at any time.
All properties are optional and will be verified when sending the next request.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `autoUpdateList <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L52>`_
       - | ``boolean``
       - | if true the nodelist will be automaticly updated if the lastBlock is newer.
         | 
         | default: true
         |   *(optional)* 
     * - | `bootWeights <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L107>`_
       - | ``boolean``
       - | if true, the first request (updating the nodelist) will also fetch the current health status
         | and use it for blacklisting unhealthy nodes. This is used only if no nodelist is availabkle from cache.
         | 
         | default: false
         |   *(optional)* 
     * - | `btc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L262>`_
       - | `btc_config <#type-btc_config>`_ 
       - | config for btc  *(optional)* 
     * - | `chainId <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L67>`_
       - | ``string``
       - | The chain-id based on EIP-155.
         | or the name of the supported chain.
         | 
         | Currently we support 'mainnet', 'goerli', 'ipfs' and 'local'
         | 
         | While most of the chains use preconfigured chain settings,
         | 'local' actually uses the local running client turning of proof.
         | 
         | example: '0x1' or 'mainnet' or 'goerli'
         | 
         | default: 'mainnet'
         |  
     * - | `chainRegistry <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L194>`_
       - | ``string``
       - | main chain-registry contract
         | example: 0xe36179e2286ef405e929C90ad3E70E649B22a945  *(optional)* 
     * - | `experimental <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L99>`_
       - | ``boolean``
       - | if true, features marked as experimental are allowed.
         | Otherwise an exception would be thrown if those features would be used.
         | 
         | default: false
         |   *(optional)* 
     * - | `finality <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L82>`_
       - | ``number``
       - | the number in percent needed in order reach finality if you run on a POA-Chain.
         | (% of signature of the validators)
         | 
         | default: 0
         |   *(optional)* 
     * - | `includeCode <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L91>`_
       - | ``boolean``
       - | if true, the request should include the codes of all accounts.
         | Otherwise only the the codeHash is returned.
         | In this case the client may ask by calling eth_getCode() afterwards
         | 
         | default: false
         |   *(optional)* 
     * - | `keepIn3 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L125>`_
       - | ``boolean``
       - | if true, the in3-section of the response will be kept and returned.
         | Otherwise it will be removed after validating the data.
         | This is useful for debugging or if the proof should be used afterwards.
         | 
         | default: false
         |   *(optional)* 
     * - | `key <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L130>`_
       - | `Hash <#type-hash>`_ 
       - | the key to sign requests. This is required for payments.  *(optional)* 
     * - | `mainChain <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L199>`_
       - | ``string``
       - | main chain-id, where the chain registry is running.
         | example: 0x1  *(optional)* 
     * - | `maxAttempts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L115>`_
       - | ``number``
       - | max number of attempts in case a response is rejected.
         | Incubed will retry to find a different node giving a verified response.
         | 
         | default: 5
         |   *(optional)* 
     * - | `minDeposit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L157>`_
       - | ``number``
       - | min stake of the server. Only nodes owning at least this amount will be chosen.
         | 
         | default: 0
         |  
     * - | `nodeLimit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L137>`_
       - | ``number``
       - | the limit of nodes to store in the client. If set a random seed will be picked, which is the base for a deterministic verifiable partial nodelist.
         | 
         | default: 0
         |   *(optional)* 
     * - | `nodeProps <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L162>`_
       - | ``number`` | `Hex <#type-hex>`_ 
       - | a bitmask-value combining the minimal properties as filter for the selected nodes. See https://in3.readthedocs.io/en/develop/spec.html#node-structure for details. 
     * - | `nodes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L203>`_
       - | 
       - | the nodelists per chain. the chain_id will be used as key within the object.  *(optional)* 
     * - | `proof <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L169>`_
       - | ``'none'`` | ``'standard'`` | ``'full'``
       - | if true the nodes should send a proof of the response
         | 
         | default: 'standard'
         |   *(optional)* 
     * - | `replaceLatestBlock <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L176>`_
       - | ``number``
       - | if specified, the blocknumber *latest* will be replaced by blockNumber- specified value
         | 
         | default: 6
         |   *(optional)* 
     * - | `requestCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L183>`_
       - | ``number``
       - | the number of request send when getting a first answer
         | 
         | default: 1
         |  
     * - | `rpc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L188>`_
       - | ``string``
       - | url of a rpc-endpoints to use. If this is set proof will be turned off and it will be treated like local_chain.  *(optional)* 
     * - | `signatureCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L74>`_
       - | ``number``
       - | number of signatures requested. The more signatures, the more security you get, but responses may take longer.
         | 
         | default: 0
         |   *(optional)* 
     * - | `stats <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L144>`_
       - | ``boolean``
       - | if false, the requests will not be included in the stats of the nodes ( or marked as intern ).
         | 
         | default: true
         |   *(optional)* 
     * - | `timeout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L151>`_
       - | ``number``
       - | specifies the number of milliseconds before the request times out. increasing may be helpful if the device uses a slow connection.
         | 
         | default: 5000
         |   *(optional)* 
     * - | `zksync <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L260>`_
       - | `zksync_config <#type-zksync_config>`_ 
       - | config for zksync  *(optional)* 

```


#### transport()


sets the transport-function. 

```eval_rst
``Promise<string>`` `transport <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L45>`_ (
      url:``string``,
      payload:``string``,
      timeout:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | url
       - | ``string``
       - | url
     * - | payload
       - | ``string``
       - | payload
     * - | timeout
       - | ``number``
       - | timeout

```


Returns: 
```eval_rst
``Promise<string>``
```

a configuration of a in3-server.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L277>`_
       - | ``string``
       - | the address of the node, which is the public address it iis signing with.
         | example: 0x6C1a01C2aB554930A937B0a2E8105fB47946c679 
     * - | `capacity <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L302>`_
       - | ``number``
       - | the capacity of the node.
         | example: 100  *(optional)* 
     * - | `chainIds <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L292>`_
       - | ``string`` []
       - | the list of supported chains
         | example: 0x1 
     * - | `deposit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L297>`_
       - | ``number``
       - | the deposit of the node in wei
         | example: 12350000 
     * - | `index <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L272>`_
       - | ``number``
       - | the index within the contract
         | example: 13  *(optional)* 
     * - | `props <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L307>`_
       - | ``number``
       - | the properties of the node.
         | example: 3  *(optional)* 
     * - | `registerTime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L312>`_
       - | ``number``
       - | the UNIX-timestamp when the node was registered
         | example: 1563279168  *(optional)* 
     * - | `timeout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L282>`_
       - | ``number``
       - | the time (in seconds) until an owner is able to receive his deposit back after he unregisters himself
         | example: 3600  *(optional)* 
     * - | `unregisterTime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L317>`_
       - | ``number``
       - | the UNIX-timestamp when the node is allowed to be deregister
         | example: 1563279168  *(optional)* 
     * - | `url <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L287>`_
       - | ``string``
       - | the endpoint to post to
         | example: https://in3.slock.it 

```
a local weight of a n3-node. (This is used internally to weight the requests)

```eval_rst
  .. list-table::
     :widths: auto

     * - | `avgResponseTime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L337>`_
       - | ``number``
       - | average time of a response in ms
         | example: 240  *(optional)* 
     * - | `blacklistedUntil <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L351>`_
       - | ``number``
       - | blacklisted because of failed requests until the timestamp
         | example: 1529074639623  *(optional)* 
     * - | `lastRequest <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L346>`_
       - | ``number``
       - | timestamp of the last request in ms
         | example: 1529074632623  *(optional)* 
     * - | `pricePerRequest <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L341>`_
       - | ``number``
       - | last price  *(optional)* 
     * - | `responseCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L332>`_
       - | ``number``
       - | number of uses.
         | example: 147  *(optional)* 
     * - | `weight <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L327>`_
       - | ``number``
       - | factor the weight this noe (default 1.0)
         | example: 0.5  *(optional)* 

```
a Incubed plugin.

Depending on the methods this will register for those actions.




#### getAccount()


returns address  *(optional)* 

```eval_rst
``string`` `getAccount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L418>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | 
         | 

```


Returns: 
```eval_rst
``string``
```



#### getAccounts()


returns list of addresses  *(optional)* 

```eval_rst
`Address <#type-address>`_  [] `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L424>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | 
         | 

```


Returns: 
```eval_rst
`Address <#type-address>`_  []
```



#### handleRPC()


called for each request.
If the plugin wants to handle the request, this function should return the value or a Promise for the value.
If the plugin does not want to handle it, it should rreturn undefined.  *(optional)* 

```eval_rst
``undefined`` | ``Promise<any>`` `handleRPC <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L433>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ ,
      request:`RPCRequest <#type-rpcrequest>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | the current client
     * - | request
       - | `RPCRequest <#type-rpcrequest>`_ 
       - | the rpc-request
         | 

```


Returns: 
```eval_rst
``undefined`` | ``Promise<any>``
```



#### term()


this is called when the client is cleaned up.  *(optional)* 

```eval_rst
``void`` `term <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L412>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | the client object
         | 

```


API for storing and retrieving IPFS-data.



#### get()


retrieves the content for a hash from IPFS. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `get <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1889>`_ (
      multihash:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | multihash
       - | ``string``
       - | the IPFS-hash to fetch
         | 
         | 

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### put()


stores the data on ipfs and returns the IPFS-Hash. 

```eval_rst
``Promise<string>`` `put <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1894>`_ (
      content:`BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | content
       - | `BufferType <#type-buffertype>`_ 
       - | puts a IPFS content
         | 

```


Returns: 
```eval_rst
``Promise<string>``
```

a JSONRPC-Request with N3-Extension

```eval_rst
  .. list-table::
     :widths: auto

     * - | `id <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L371>`_
       - | ``number`` | ``string``
       - | the identifier of the request
         | example: 2  *(optional)* 
     * - | `jsonrpc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L361>`_
       - | ``'2.0'``
       - | the version 
     * - | `method <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L366>`_
       - | ``string``
       - | the method to call
         | example: eth_getBalance 
     * - | `params <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L376>`_
       - | ``any`` []
       - | the params
         | example: 0xe36179e2286ef405e929C90ad3E70E649B22a945,latest  *(optional)* 

```
a JSONRPC-Responset with N3-Extension

```eval_rst
  .. list-table::
     :widths: auto

     * - | `error <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L394>`_
       - | ``string`` | 
       - | in case of an error this needs to be set  *(optional)* 
     * - | `id <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L390>`_
       - | ``string`` | ``number``
       - | the id matching the request
         | example: 2 
     * - | `jsonrpc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L385>`_
       - | ``'2.0'``
       - | the version 
     * - | `result <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L399>`_
       - | ``any``
       - | the params
         | example: 0xa35bc  *(optional)* 

```




#### sign()


signing of any data.
if hashFirst is true the data should be hashed first, otherwise the data is the hash. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L709>`_ (
      data:`Hex <#type-hex>`_ ,
      account:`Address <#type-address>`_ ,
      sign_type:`SignType <#type-signtype>`_ ,
      payloadType:`SignPayload <#type-signpayload>`_ ,
      meta:``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_ 
       - | a Hexcoded String (starting with 0x)
     * - | account
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | sign_type
       - | `SignType <#type-signtype>`_ 
       - | the type of signature to create.
         |     - ec_hash : the data needs to be hashed first ( using keccak) before signing
         |     - ec_raw : the data is the ryw value (32bytes) to sign
         |     - ec_prefix : the data is a message which needs to be prefixed with the EthereumSignedMessage and length to before hashing and signing
     * - | payloadType
       - | `SignPayload <#type-signpayload>`_ 
       - | The type of the payload to sign
     * - | meta
       - | ``any``
       - | meta

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### canSign()


returns true if the account is supported (or unlocked) 

```eval_rst
``Promise<boolean>`` `canSign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L700>`_ (
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<boolean>``
```



#### getAccounts()


returns all addresses managed by the signer. 

```eval_rst
`Address <#type-address>`_  [] `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L703>`_ ()
```

Returns: 
```eval_rst
`Address <#type-address>`_  []
```

Token representation

```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1001>`_
       - | `String <#type-string>`_ 
       - | the address 
     * - | `decimals <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1002>`_
       - | ``number``
       - | the decimals 
     * - | `id <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1003>`_
       - | ``number``
       - | the id 
     * - | `symbol <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1004>`_
       - | `String <#type-string>`_ 
       - | the symbol 

```
Token List.

Transaction state

```eval_rst
  .. list-table::
     :widths: auto

     * - | `block <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L960>`_
       - | `BlockInfo <#type-blockinfo>`_ 
       - | the block 
     * - | `executed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L961>`_
       - | ``boolean``
       - | the executed 
     * - | `failReason <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L962>`_
       - | ``string``
       - | the failReason 
     * - | `success <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L963>`_
       - | ``boolean``
       - | the success 

```
Defines the type of a transaction.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `type <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L978>`_
       - | ``'Withdraw'`` 
         | | ``'Transfer'`` 
         | | ``'TransferToNew'``
       - | the type 

```
Collection of different util-functions.



#### abiDecode()


decodes the given data as ABI-encoded (without the methodHash) 

```eval_rst
``any`` [] `abiDecode <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L780>`_ (
      signature:``string``,
      data:`Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | signature
       - | ``string``
       - | the method signature, which must contain a return description
     * - | data
       - | `Data <#type-data>`_ 
       - | the data to decode
         | 

```


Returns: 
```eval_rst
``any`` []
```



#### abiEncode()


encodes the given arguments as ABI-encoded (including the methodHash) 

```eval_rst
`Hex <#type-hex>`_  `abiEncode <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L773>`_ (
      signature:``string``,
      args:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | signature
       - | ``string``
       - | the method signature
     * - | args
       - | ``any`` []
       - | the arguments
         | 

```


Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### checkAddressChecksum()


checks whether the given address is a correct checksumAddress
If the chainId is passed, it will be included accord to EIP 1191 

```eval_rst
``boolean`` `checkAddressChecksum <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L796>`_ (
      address:`Address <#type-address>`_ ,
      chainId:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | the address (as hex)
     * - | chainId
       - | ``number``
       - | the chainId (if supported)
         | 

```


Returns: 
```eval_rst
``boolean``
```



#### createSignatureHash()


a Hexcoded String (starting with 0x) 

```eval_rst
`Hex <#type-hex>`_  `createSignatureHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L763>`_ (
      def:`ABI <#type-abi>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | def
       - | `ABI <#type-abi>`_ 
       - | def

```


Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### decodeEvent()


decode event 

```eval_rst
``any`` `decodeEvent <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L765>`_ (
      log:`Log <#type-log>`_ ,
      d:`ABI <#type-abi>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | log
       - | `Log <#type-log>`_ 
       - | log
     * - | d
       - | `ABI <#type-abi>`_ 
       - | d

```


Returns: 
```eval_rst
``any``
```



#### ecSign()


create a signature (65 bytes) for the given message and kexy 

```eval_rst
`BufferType <#type-buffertype>`_  `ecSign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L865>`_ (
      pk:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ ,
      msg:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ ,
      signType:`SignType <#type-signtype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the private key
     * - | msg
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the message
     * - | signType
       - | `SignType <#type-signtype>`_ 
       - | the type of signature to create
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### getVersion()


returns the incubed version. 

```eval_rst
``string`` `getVersion <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L831>`_ ()
```

Returns: 
```eval_rst
``string``
```



#### isAddress()


checks whether the given address is a valid hex string with 0x-prefix and 20 bytes 

```eval_rst
``boolean`` `isAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L802>`_ (
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | the address (as hex)
         | 

```


Returns: 
```eval_rst
``boolean``
```



#### keccak()


calculates the keccack hash for the given data. 

```eval_rst
`BufferType <#type-buffertype>`_  `keccak <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L808>`_ (
      data:`BufferType <#type-buffertype>`_  | `Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `BufferType <#type-buffertype>`_  
         | | `Data <#type-data>`_ 
       - | the data as Uint8Array or hex data.
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### private2address()


generates the public address from the private key. 

```eval_rst
`Address <#type-address>`_  `private2address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L879>`_ (
      pk:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the private key.
         | 

```


Returns: 
```eval_rst
`Address <#type-address>`_ 
```



#### private2public()


generates the public address (64 bytes) from the private key 

```eval_rst
`BufferType <#type-buffertype>`_  `private2public <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L885>`_ (
      pk:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the raw private key
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### randomBytes()


returns a Buffer with strong random bytes.
Thsi will use the browsers crypto-module or in case of nodejs use the crypto-module there. 

```eval_rst
`BufferType <#type-buffertype>`_  `randomBytes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L821>`_ (
      len:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | len
       - | ``number``
       - | the number of bytes to generate.
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### sha256()


calculates the sha256 hash for the given data. 

```eval_rst
`BufferType <#type-buffertype>`_  `sha256 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L814>`_ (
      data:`BufferType <#type-buffertype>`_  | `Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `BufferType <#type-buffertype>`_  
         | | `Data <#type-data>`_ 
       - | the data as Uint8Array or hex data.
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### soliditySha3()


solidity sha3 

```eval_rst
``string`` `soliditySha3 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L766>`_ (
      args:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | args
       - | ``any`` []
       - | args

```


Returns: 
```eval_rst
``string``
```



#### splitSignature()


takes raw signature (65 bytes) and splits it into a signature object. 

```eval_rst
`Signature <#type-signature>`_  `splitSignature <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L873>`_ (
      signature:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ ,
      message:`BufferType <#type-buffertype>`_  | `Hex <#type-hex>`_ ,
      hashFirst:``boolean``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | signature
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the 65 byte-signature
     * - | message
       - | `BufferType <#type-buffertype>`_  
         | | `Hex <#type-hex>`_ 
       - | the message
     * - | hashFirst
       - | ``boolean``
       - | if true (default) this will be taken as raw-data and will be hashed first.
         | 

```


Returns: 
```eval_rst
`Signature <#type-signature>`_ 
```



#### toBuffer()


converts any value to a Buffer.
optionally the target length can be specified (in bytes) 

```eval_rst
`BufferType <#type-buffertype>`_  `toBuffer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L846>`_ (
      data:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_  | ``number`` | ``bigint``,
      len:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_  
         | | ``number`` 
         | | ``bigint``
       - | data
     * - | len
       - | ``number``
       - | len

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### toChecksumAddress()


generates a checksum Address for the given address.
If the chainId is passed, it will be included accord to EIP 1191 

```eval_rst
`Address <#type-address>`_  `toChecksumAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L788>`_ (
      address:`Address <#type-address>`_ ,
      chainId:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | the address (as hex)
     * - | chainId
       - | ``number``
       - | the chainId (if supported)
         | 

```


Returns: 
```eval_rst
`Address <#type-address>`_ 
```



#### toHex()


converts any value to a hex string (with prefix 0x).
optionally the target length can be specified (in bytes) 

```eval_rst
`Hex <#type-hex>`_  `toHex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L827>`_ (
      data:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_  | ``number`` | ``bigint``,
      len:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_  
         | | ``number`` 
         | | ``bigint``
       - | data
     * - | len
       - | ``number``
       - | len

```


Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### toMinHex()


removes all leading 0 in the hexstring 

```eval_rst
``string`` `toMinHex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L834>`_ (
      key:``string`` | `BufferType <#type-buffertype>`_  | ``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | key
       - | ``string`` 
         | | `BufferType <#type-buffertype>`_  
         | | ``number``
       - | key

```


Returns: 
```eval_rst
``string``
```



#### toNumber()


converts any value to a hex string (with prefix 0x).
optionally the target length can be specified (in bytes) 

```eval_rst
``number`` `toNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L852>`_ (
      data:``string`` | `BufferType <#type-buffertype>`_  | ``number`` | ``bigint``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | ``string`` 
         | | `BufferType <#type-buffertype>`_  
         | | ``number`` 
         | | ``bigint``
       - | data

```


Returns: 
```eval_rst
``number``
```



#### toUint8Array()


converts any value to a Uint8Array.
optionally the target length can be specified (in bytes) 

```eval_rst
`BufferType <#type-buffertype>`_  `toUint8Array <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L840>`_ (
      data:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_  | ``number`` | ``bigint``,
      len:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_  
         | | ``number`` 
         | | ``bigint``
       - | data
     * - | len
       - | ``number``
       - | len

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### toUtf8()


convert to String 

```eval_rst
``string`` `toUtf8 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L857>`_ (
      val:``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | val
       - | ``any``
       - | val

```


Returns: 
```eval_rst
``string``
```



```eval_rst
  .. list-table::
     :widths: auto

     * - | `events <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1562>`_
       - | 
       - | the events 
     * - | `methods <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1556>`_
       - | 
       - | the methods 
     * - | `options <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1542>`_
       - | 
       - | the options 

```


#### deploy()


deploy 

```eval_rst
`Web3TransactionObject <#type-web3transactionobject>`_  `deploy <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1552>`_ (
      args:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | args
       - | 
       - | args

```


Returns: 
```eval_rst
`Web3TransactionObject <#type-web3transactionobject>`_ 
```



#### once()


once 

```eval_rst
``void`` `once <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1560>`_ (
      eventName:``string``,
      options:,
      handler:(`Error <#type-error>`_  , `Web3Event <#type-web3event>`_ ) => ``void``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | eventName
       - | ``string``
       - | event name
     * - | options
       - | 
       - | options
     * - | handler
       - | (`Error <#type-error>`_  , `Web3Event <#type-web3event>`_ ) => ``void``
       - | handler

```




#### getPastEvents()


get past events 

```eval_rst
``Promise<>`` `getPastEvents <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1575>`_ (
      evName:``string``,
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | evName
       - | ``string``
       - | ev name
     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
``Promise<>``
```



```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1443>`_
       - | `Address <#type-address>`_ 
       - | the address 
     * - | `blockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1445>`_
       - | `Hash <#type-hash>`_ 
       - | the blockHash 
     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1444>`_
       - | ``number``
       - | the blockNumber 
     * - | `event <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1438>`_
       - | ``string``
       - | the event 
     * - | `logIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1440>`_
       - | ``number``
       - | the logIndex 
     * - | `raw <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1446>`_
       - | 
       - | the raw 
     * - | `returnValues <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1435>`_
       - | 
       - | the returnValues 
     * - | `signature <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1439>`_
       - | ``string``
       - | the signature 
     * - | `transactionHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1442>`_
       - | `Hash <#type-hash>`_ 
       - | the transactionHash 
     * - | `transactionIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1441>`_
       - | ``number``
       - | the transactionIndex 

```




#### call()


call 

```eval_rst
``Promise<any>`` `call <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1518>`_ (
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
``Promise<any>``
```



#### encodeABI()


a Hexcoded String (starting with 0x) 

```eval_rst
`Hex <#type-hex>`_  `encodeABI <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1538>`_ ()
```

Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### estimateGas()


estimate gas 

```eval_rst
``Promise<number>`` `estimateGas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1532>`_ (
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
``Promise<number>``
```



#### send()


send 

```eval_rst
``Promise<any>`` `send <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1524>`_ (
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
``Promise<any>``
```

return structure after fetching the current account info.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L925>`_
       - | ``string``
       - | the address 
     * - | `committed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L926>`_
       - | 
       - | the committed 
     * - | `depositing <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L933>`_
       - | 
       - | the depositing 
     * - | `id <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L938>`_
       - | ``number``
       - | the id 
     * - | `verified <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L939>`_
       - | 
       - | the verified 

```
API for zksync.



#### aggregatePubKey()


aggregates the given publickeys into one public key for a Musig Schnorr signature 

```eval_rst
``string`` `aggregatePubKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1138>`_ (
      pubkeys:``string`` | ``string`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pubkeys
       - | ``string`` | ``string`` []
       - | the packed public keys (either a concatenated string or as array of public keys)
         | 

```


Returns: 
```eval_rst
``string``
```



#### deposit()


deposits the declared amount into the rollup 

```eval_rst
`Promise<DepositResponse> <#type-depositresponse>`_  `deposit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1095>`_ (
      amount:``number``,
      token:``string``,
      approveDepositAmountForERC20:``boolean``,
      account:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | amount
       - | ``number``
       - | amount in wei to deposit
     * - | token
       - | ``string``
       - | the token identifier e.g. ETH
     * - | approveDepositAmountForERC20
       - | ``boolean``
       - | bool that is set to true if it is a erc20 token that needs approval
     * - | account
       - | ``string``
       - | address of the account that wants to deposit (if left empty it will be taken from current signer)
         | 

```


Returns: 
```eval_rst
`Promise<DepositResponse> <#type-depositresponse>`_ 
```



#### emergencyWithdraw()


executes an emergency withdrawel onchain 

```eval_rst
`Promise<String> <#type-string>`_  `emergencyWithdraw <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1119>`_ (
      token:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | token
       - | ``string``
       - | the token identifier e.g. ETH
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### getAccountAddress()


returns the address of the account used. 

```eval_rst
`String <#type-string>`_  `getAccountAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1033>`_ ()
```

Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### getAccountInfo()


gets current account Infoa and balances. 

```eval_rst
`Promise<ZKAccountInfo> <#type-zkaccountinfo>`_  `getAccountInfo <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1023>`_ (
      account:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | account
       - | ``string``
       - | the address of the account . if not specified, the first signer is used.
         | 

```


Returns: 
```eval_rst
`Promise<ZKAccountInfo> <#type-zkaccountinfo>`_ 
```



#### getContractAddress()


gets the contract address of the zksync contract 

```eval_rst
`Promise<String> <#type-string>`_  `getContractAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1028>`_ ()
```

Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### getEthopInfo()


returns the state of receipt of the PriorityOperation 

```eval_rst
`Promise<ETHOpInfoResp> <#type-ethopinforesp>`_  `getEthopInfo <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1057>`_ (
      opId:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | opId
       - | ``number``
       - | the id of the PriorityOperation
         | 

```


Returns: 
```eval_rst
`Promise<ETHOpInfoResp> <#type-ethopinforesp>`_ 
```



#### getSyncKey()


returns private key used for signing zksync transactions 

```eval_rst
`String <#type-string>`_  `getSyncKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1076>`_ ()
```

Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### getSyncPubKey()


returns public key used for signing zksync transactions 

```eval_rst
`String <#type-string>`_  `getSyncPubKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1086>`_ ()
```

Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### getSyncPubKeyHash()


returns public key used for signing zksync transactions 

```eval_rst
`String <#type-string>`_  `getSyncPubKeyHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1081>`_ ()
```

Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### getTokenPrice()


returns the current token price 

```eval_rst
`Promise<Number> <#type-number>`_  `getTokenPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1063>`_ (
      tokenSymbol:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tokenSymbol
       - | ``string``
       - | the address of the token
         | 

```


Returns: 
```eval_rst
`Promise<Number> <#type-number>`_ 
```



#### getTokens()


returns an object containing Token objects with its short name as key 

```eval_rst
`Promise<Tokens> <#type-tokens>`_  `getTokens <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1038>`_ ()
```

Returns: 
```eval_rst
`Promise<Tokens> <#type-tokens>`_ 
```



#### getTxFee()


returns the transaction fee 

```eval_rst
`Promise<Fee> <#type-fee>`_  `getTxFee <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1071>`_ (
      txType:`TxType <#type-txtype>`_ ,
      receipient:``string``,
      token:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | txType
       - | `TxType <#type-txtype>`_ 
       - | either Withdraw or Transfer
     * - | receipient
       - | ``string``
       - | the address the transaction is send to
     * - | token
       - | ``string``
       - | the token identifier e.g. ETH
         | 

```


Returns: 
```eval_rst
`Promise<Fee> <#type-fee>`_ 
```



#### getTxInfo()


get transaction info 

```eval_rst
`Promise<TxInfo> <#type-txinfo>`_  `getTxInfo <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1044>`_ (
      txHash:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | txHash
       - | ``string``
       - | the has of the tx you want the info about
         | 

```


Returns: 
```eval_rst
`Promise<TxInfo> <#type-txinfo>`_ 
```



#### setKey()


set the signer key based on the current pk 

```eval_rst
`Promise<String> <#type-string>`_  `setKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1051>`_ (
      tokenSymbol:``string``,
      newKey:`BufferType <#type-buffertype>`_  | ``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tokenSymbol
       - | ``string``
       - | the address of the token
     * - | newKey
       - | `BufferType <#type-buffertype>`_  | ``string``
       - | the seed of the new key ( this is optional, if ommited the derrived key will be set in the rollup)
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### sign()


signs the message based on the config and returns a Musig Schnorr signature 

```eval_rst
`Promise<String> <#type-string>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1125>`_ (
      msg:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string``
       - | the message
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### transfer()


transfers the specified amount to another address within the zksync rollup 

```eval_rst
`Promise<String> <#type-string>`_  `transfer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1104>`_ (
      to:``string``,
      amount:``number``,
      token:``string``,
      account:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | to
       - | ``string``
       - | address of the receipient
     * - | amount
       - | ``number``
       - | amount to send in wei
     * - | token
       - | ``string``
       - | the token indentifier e.g. ETH
     * - | account
       - | ``string``
       - | address of the account that wants to transfer (if left empty it will be taken from current signer)
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### verify()


verifies a Musig Schnorr signature 

```eval_rst
``boolean`` `verify <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1132>`_ (
      msg:``string``,
      sig:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string``
       - | the message
     * - | sig
       - | ``string``
       - | the 96byte long signature
         | 

```


Returns: 
```eval_rst
``boolean``
```



#### withdraw()


withdraws the specified amount from the rollup to a specific address 

```eval_rst
`Promise<String> <#type-string>`_  `withdraw <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1113>`_ (
      ethAddress:``string``,
      amount:``number``,
      token:``string``,
      account:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | ethAddress
       - | ``string``
       - | the receipient address
     * - | amount
       - | ``number``
       - | amount to withdraw in wei
     * - | token
       - | ``string``
       - | the token identifier e.g. ETH
     * - | account
       - | ``string``
       - | address of the account that wants to withdraw (if left empty it will be taken from current signer)
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



```eval_rst
  .. list-table::
     :widths: auto

     * - | `anonymous <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L666>`_
       - | ``boolean``
       - | the anonymous  *(optional)* 
     * - | `components <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L670>`_
       - | `ABIField <#type-abifield>`_  []
       - | the components  *(optional)* 
     * - | `constant <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L667>`_
       - | ``boolean``
       - | the constant  *(optional)* 
     * - | `inputs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L671>`_
       - | `ABIField <#type-abifield>`_  []
       - | the inputs  *(optional)* 
     * - | `internalType <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L675>`_
       - | ``string``
       - | the internalType  *(optional)* 
     * - | `name <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L673>`_
       - | ``string``
       - | the name  *(optional)* 
     * - | `outputs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L672>`_
       - | `ABIField <#type-abifield>`_  [] | ``any`` []
       - | the outputs  *(optional)* 
     * - | `payable <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L668>`_
       - | ``boolean``
       - | the payable  *(optional)* 
     * - | `stateMutability <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L669>`_
       - | ``'pure'`` 
         | | ``'view'`` 
         | | ``'nonpayable'`` 
         | | ``'payable'`` 
         | | ``string``
       - | the stateMutability  *(optional)* 
     * - | `type <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L674>`_
       - | ``'function'`` 
         | | ``'constructor'`` 
         | | ``'event'`` 
         | | ``'fallback'`` 
         | | ``string``
       - | the type 

```


```eval_rst
  .. list-table::
     :widths: auto

     * - | `indexed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L661>`_
       - | ``boolean``
       - | the indexed  *(optional)* 
     * - | `internalType <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L660>`_
       - | ``string``
       - | the internalType  *(optional)* 
     * - | `name <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L662>`_
       - | ``string``
       - | the name 
     * - | `type <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L663>`_
       - | ``string``
       - | the type 

```
a 20 byte Address encoded as Hex (starting with 0x)
a Hexcoded String (starting with 0x)
 = `string`



```eval_rst
  .. list-table::
     :widths: auto

     * - | `author <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1338>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - the address of the author of the block (the beneficiary to whom the mining rewards were given) 
     * - | `difficulty <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1342>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the difficulty for this block 
     * - | `extraData <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1346>`_
       - | `Data <#type-data>`_ 
       - | the ‘extra data’ field of this block 
     * - | `gasLimit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1350>`_
       - | `Quantity <#type-quantity>`_ 
       - | the maximum gas allowed in this block 
     * - | `gasUsed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1352>`_
       - | `Quantity <#type-quantity>`_ 
       - | the total used gas by all transactions in this block 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1322>`_
       - | `Hash <#type-hash>`_ 
       - | hash of the block. null when its pending block 
     * - | `logsBloom <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1330>`_
       - | `Data <#type-data>`_ 
       - | 256 Bytes - the bloom filter for the logs of the block. null when its pending block 
     * - | `miner <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1340>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - alias of ‘author’ 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1326>`_
       - | `Data <#type-data>`_ 
       - | 8 bytes hash of the generated proof-of-work. null when its pending block. Missing in case of PoA. 
     * - | `number <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1320>`_
       - | `Quantity <#type-quantity>`_ 
       - | The block number. null when its pending block 
     * - | `parentHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1324>`_
       - | `Hash <#type-hash>`_ 
       - | hash of the parent block 
     * - | `receiptsRoot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1336>`_
       - | `Data <#type-data>`_ 
       - | 32 Bytes - the root of the receipts trie of the block 
     * - | `sealFields <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1360>`_
       - | `Data <#type-data>`_  []
       - | PoA-Fields 
     * - | `sha3Uncles <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1328>`_
       - | `Data <#type-data>`_ 
       - | SHA3 of the uncles data in the block 
     * - | `size <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1348>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer the size of this block in bytes 
     * - | `stateRoot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1334>`_
       - | `Data <#type-data>`_ 
       - | 32 Bytes - the root of the final state trie of the block 
     * - | `timestamp <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1354>`_
       - | `Quantity <#type-quantity>`_ 
       - | the unix timestamp for when the block was collated 
     * - | `totalDifficulty <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1344>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the total difficulty of the chain until this block 
     * - | `transactions <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1356>`_
       - | `Hash <#type-hash>`_  
         | | `Transaction <#type-transaction>`_  []
       - | Array of transaction objects, or 32 Bytes transaction hashes depending on the last given parameter 
     * - | `transactionsRoot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1332>`_
       - | `Data <#type-data>`_ 
       - | 32 Bytes - the root of the transaction trie of the block 
     * - | `uncles <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1358>`_
       - | `Hash <#type-hash>`_  []
       - | Array of uncle hashes 

```
BlockNumber or predefined Block
 = `number` | `'latest'` | `'earliest'` | `'pending'`

data encoded as Hex (starting with 0x)
a Hexcoded String (starting with 0x)
 = `string`

a 32 byte Hash encoded as Hex (starting with 0x)
a Hexcoded String (starting with 0x)
 = `string`

a Hexcoded String (starting with 0x)
 = `string`



```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1376>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - address from which this log originated. 
     * - | `blockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1372>`_
       - | `Hash <#type-hash>`_ 
       - | Hash, 32 Bytes - hash of the block where this log was in. null when its pending. null when its pending log. 
     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1374>`_
       - | `Quantity <#type-quantity>`_ 
       - | the block number where this log was in. null when its pending. null when its pending log. 
     * - | `data <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1378>`_
       - | `Data <#type-data>`_ 
       - | contains the non-indexed arguments of the log. 
     * - | `logIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1366>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the log index position in the block. null when its pending log. 
     * - | `removed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1364>`_
       - | ``boolean``
       - | true when the log was removed, due to a chain reorganization. false if its a valid log. 
     * - | `topics <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1380>`_
       - | `Data <#type-data>`_  []
       - | - Array of 0 to 4 32 Bytes DATA of indexed log arguments. (In solidity: The first topic is the hash of the signature of the event (e.g. Deposit(address,bytes32,uint256)), except you declared the event with the anonymous specifier.) 
     * - | `transactionHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1370>`_
       - | `Hash <#type-hash>`_ 
       - | Hash, 32 Bytes - hash of the transactions this log was created from. null when its pending log. 
     * - | `transactionIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1368>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the transactions index position log was created from. null when its pending log. 

```


```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1389>`_
       - | `Address <#type-address>`_ 
       - | (optional) 20 Bytes - Contract address or a list of addresses from which logs should originate. 
     * - | `fromBlock <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1385>`_
       - | `BlockType <#type-blocktype>`_ 
       - | Quantity or Tag - (optional) (default: latest) Integer block number, or 'latest' for the last mined block or 'pending', 'earliest' for not yet mined transactions.  *(optional)* 
     * - | `limit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1393>`_
       - | `Quantity <#type-quantity>`_ 
       - | å(optional) The maximum number of entries to retrieve (latest first).  *(optional)* 
     * - | `toBlock <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1387>`_
       - | `BlockType <#type-blocktype>`_ 
       - | Quantity or Tag - (optional) (default: latest) Integer block number, or 'latest' for the last mined block or 'pending', 'earliest' for not yet mined transactions.  *(optional)* 
     * - | `topics <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1391>`_
       - | ``string`` | ``string`` [] []
       - | (optional) Array of 32 Bytes Data topics. Topics are order-dependent. It’s possible to pass in null to match any topic, or a subarray of multiple topics of which one should be matching.  *(optional)* 

```
a BigInteger encoded as hex.
 = `number` | [Hex](#type-hex) 

the type of signature to create.
- ec_hash : the data needs to be hashed first ( using keccak) before signing
- ec_raw : the data is the ryw value (32bytes) to sign
- ec_prefix : the data is a message which needs to be prefixed with the EthereumSignedMessage and length to before hashing and signing
 = `'ec_hash'` | `'ec_prefix'` | `'ec_raw'`

Signature


```eval_rst
  .. list-table::
     :widths: auto

     * - | `message <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L651>`_
       - | `Data <#type-data>`_ 
       - | the message 
     * - | `messageHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L652>`_
       - | `Hash <#type-hash>`_ 
       - | the messageHash 
     * - | `r <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L654>`_
       - | `Hash <#type-hash>`_ 
       - | the r 
     * - | `s <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L655>`_
       - | `Hash <#type-hash>`_ 
       - | the s 
     * - | `signature <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L656>`_
       - | `Data <#type-data>`_ 
       - | the signature  *(optional)* 
     * - | `v <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L653>`_
       - | `Hex <#type-hex>`_ 
       - | the v 

```


```eval_rst
  .. list-table::
     :widths: auto

     * - | `chainId <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L693>`_
       - | ``any``
       - | optional chain id  *(optional)* 
     * - | `data <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L689>`_
       - | ``string``
       - | 4 byte hash of the method signature followed by encoded parameters. For details see Ethereum Contract ABI. 
     * - | `from <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L679>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - The address the transaction is send from. 
     * - | `gas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L683>`_
       - | `Quantity <#type-quantity>`_ 
       - | Integer of the gas provided for the transaction execution. eth_call consumes zero gas, but this parameter may be needed by some executions. 
     * - | `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L685>`_
       - | `Quantity <#type-quantity>`_ 
       - | Integer of the gas price used for each paid gas. 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L691>`_
       - | `Quantity <#type-quantity>`_ 
       - | nonce 
     * - | `to <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L681>`_
       - | `Address <#type-address>`_ 
       - | (optional when creating new contract) 20 Bytes - The address the transaction is directed to.  *(optional)* 
     * - | `value <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L687>`_
       - | `Quantity <#type-quantity>`_ 
       - | Integer of the value sent with this transaction. 

```


```eval_rst
  .. list-table::
     :widths: auto

     * - | `blockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1281>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - hash of the block where this transaction was in. null when its pending. 
     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1283>`_
       - | `BlockType <#type-blocktype>`_ 
       - | block number where this transaction was in. null when its pending. 
     * - | `chainId <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1309>`_
       - | `Quantity <#type-quantity>`_ 
       - | the chain id of the transaction, if any. 
     * - | `condition <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1313>`_
       - | ``any``
       - | (optional) conditional submission, Block number in block or timestamp in time or null. (parity-feature)  *(optional)* 
     * - | `creates <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1311>`_
       - | `Address <#type-address>`_ 
       - | creates contract address 
     * - | `from <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1287>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - address of the sender. 
     * - | `gas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1295>`_
       - | `Quantity <#type-quantity>`_ 
       - | gas provided by the sender. 
     * - | `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1293>`_
       - | `Quantity <#type-quantity>`_ 
       - | gas price provided by the sender in Wei. 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1277>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - hash of the transaction. 
     * - | `input <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1297>`_
       - | `Data <#type-data>`_ 
       - | the data send along with the transaction. 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1279>`_
       - | `Quantity <#type-quantity>`_ 
       - | the number of transactions made by the sender prior to this one. 
     * - | `pk <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1315>`_
       - | ``any``
       - | optional: the private key to use for signing  *(optional)* 
     * - | `publicKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1307>`_
       - | `Hash <#type-hash>`_ 
       - | public key of the signer. 
     * - | `r <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1303>`_
       - | `Quantity <#type-quantity>`_ 
       - | the R field of the signature. 
     * - | `raw <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1305>`_
       - | `Data <#type-data>`_ 
       - | raw transaction data 
     * - | `standardV <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1301>`_
       - | `Quantity <#type-quantity>`_ 
       - | the standardised V field of the signature (0 or 1). 
     * - | `to <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1289>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - address of the receiver. null when its a contract creation transaction. 
     * - | `transactionIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1285>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the transactions index position in the block. null when its pending. 
     * - | `v <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1299>`_
       - | `Quantity <#type-quantity>`_ 
       - | the standardised V field of the signature. 
     * - | `value <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1291>`_
       - | `Quantity <#type-quantity>`_ 
       - | value transferred in Wei. 

```
a  Receipt of a Transaction containing the events and execution status


```eval_rst
  .. list-table::
     :widths: auto

     * - | `blockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1243>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - hash of the block where this transaction was in. 
     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1245>`_
       - | `BlockType <#type-blocktype>`_ 
       - | block number where this transaction was in. 
     * - | `contractAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1247>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - The contract address created, if the transaction was a contract creation, otherwise null. 
     * - | `cumulativeGasUsed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1249>`_
       - | `Quantity <#type-quantity>`_ 
       - | The total amount of gas used when this transaction was executed in the block. 
     * - | `events <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1269>`_
       - | 
       - | event objects, which are only added in the web3Contract  *(optional)* 
     * - | `from <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1251>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - The address of the sender. 
     * - | `gasUsed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1255>`_
       - | `Quantity <#type-quantity>`_ 
       - | The amount of gas used by this specific transaction alone. 
     * - | `logs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1257>`_
       - | `Log <#type-log>`_  []
       - | Array of log objects, which this transaction generated. 
     * - | `logsBloom <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1259>`_
       - | `Data <#type-data>`_ 
       - | 256 Bytes - A bloom filter of logs/events generated by contracts during transaction execution. Used to efficiently rule out transactions without expected logs. 
     * - | `root <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1261>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - Merkle root of the state trie after the transaction has been executed (optional after Byzantium hard fork EIP609)  *(optional)* 
     * - | `status <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1263>`_
       - | `Quantity <#type-quantity>`_ 
       - | 0x0 indicates transaction failure , 0x1 indicates transaction success. Set for blocks mined after Byzantium hard fork EIP609, null before. 
     * - | `to <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1253>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - The address of the receiver. null when it’s a contract creation transaction. 
     * - | `transactionHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1265>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - hash of the transaction. 
     * - | `transactionIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1267>`_
       - | `Quantity <#type-quantity>`_ 
       - | Integer of the transactions index position in the block. 

```


```eval_rst
  .. list-table::
     :widths: auto

     * - | `args <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1422>`_
       - | ``any`` []
       - | the argument to pass to the method  *(optional)* 
     * - | `confirmations <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1428>`_
       - | ``number``
       - | number of block to wait before confirming  *(optional)* 
     * - | `data <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1404>`_
       - | `Data <#type-data>`_ 
       - | the data to send  *(optional)* 
     * - | `from <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1401>`_
       - | `Address <#type-address>`_ 
       - | address of the account to use  *(optional)* 
     * - | `gas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1407>`_
       - | ``number``
       - | the gas needed  *(optional)* 
     * - | `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1410>`_
       - | ``number``
       - | the gasPrice used  *(optional)* 
     * - | `method <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1419>`_
       - | ``string``
       - | the ABI of the method to be used  *(optional)* 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1413>`_
       - | ``number``
       - | the nonce  *(optional)* 
     * - | `pk <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1425>`_
       - | `Hash <#type-hash>`_ 
       - | raw private key in order to sign  *(optional)* 
     * - | `timeout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1431>`_
       - | ``number``
       - | number of seconds to wait for confirmations before giving up. Default: 10  *(optional)* 
     * - | `to <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1398>`_
       - | `Address <#type-address>`_ 
       - | contract  *(optional)* 
     * - | `value <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1416>`_
       - | `Quantity <#type-quantity>`_ 
       - | the value in wei  *(optional)* 

```
bitcoin configuration.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `maxDAP <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2116>`_
       - | ``number``
       - | max number of DAPs (Difficulty Adjustment Periods) allowed when accepting new targets.  *(optional)* 
     * - | `maxDiff <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2121>`_
       - | ``number``
       - | max increase (in percent) of the difference between targets when accepting new targets.  *(optional)* 

```
zksync configuration.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `account <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1154>`_
       - | ``string``
       - | the account to be used. if not specified, the first signer will be used.  *(optional)* 
     * - | `create2 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1170>`_
       - | 
       - | create2 arguments  *(optional)* 
     * - | `musig_pub_keys <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1195>`_
       - | ``string``
       - | concated packed public keys of the signers of the multisig  *(optional)* 
     * - | `musig_urls <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1190>`_
       - | ``string`` []
       - | if used as a musig-signature the  *(optional)* 
     * - | `provider_url <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1149>`_
       - | ``string``
       - | url of the zksync-server  *(optional)* 
     * - | `signer_type <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1159>`_
       - | ``'pk'`` | ``'contract'`` | ``'create2'``
       - | defines the type of the signer. Must be one of those 3 values. (default: pk)  *(optional)* 
     * - | `sync_key <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1165>`_
       - | ``string``
       - | optionaly the private seephrase to use when signing sync-transaction.
         | If ommited this key is derrived from the signer.  *(optional)* 

```


## Package index


### Type BrowserSigner


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L737)


Signer, which stores the key encrypted in the indexDB in the browser.



#### constructor()


creates a BrowserSigner, which will use the given callback-function in order to retrieve the password, which is used to encrypt the stored keys. 

```eval_rst
`BrowserSigner <#type-browsersigner>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L737>`_ (
      passwordCB:() => `String <#type-string>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | passwordCB
       - | () => `String <#type-string>`_ 
       - | password cb

```


Returns: 
```eval_rst
`BrowserSigner <#type-browsersigner>`_ 
```



#### sign()


signing of any data.
if hashFirst is true the data should be hashed first, otherwise the data is the hash. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L754>`_ (
      data:`Hex <#type-hex>`_ ,
      account:`Address <#type-address>`_ ,
      sign_type:`SignType <#type-signtype>`_ ,
      payloadType:`SignPayload <#type-signpayload>`_ ,
      meta:``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_ 
       - | a Hexcoded String (starting with 0x)
     * - | account
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | sign_type
       - | `SignType <#type-signtype>`_ 
       - | the type of signature to create.
         |     - ec_hash : the data needs to be hashed first ( using keccak) before signing
         |     - ec_raw : the data is the ryw value (32bytes) to sign
         |     - ec_prefix : the data is a message which needs to be prefixed with the EthereumSignedMessage and length to before hashing and signing
     * - | payloadType
       - | `SignPayload <#type-signpayload>`_ 
       - | The type of the payload to sign
     * - | meta
       - | ``any``
       - | meta

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### canSign()


returns true if the account is supported (or unlocked) 

```eval_rst
``Promise<boolean>`` `canSign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L748>`_ (
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<boolean>``
```



#### generateAndStorePrivateKey()


adds a private key to the signer and returns the address associated with it. 

```eval_rst
``Promise<string>`` `generateAndStorePrivateKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L745>`_ (
      pk:`BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `BufferType <#type-buffertype>`_ 
       - | pk

```


Returns: 
```eval_rst
``Promise<string>``
```



#### getAccounts()


returns all addresses managed by the signer. 

```eval_rst
`Address <#type-address>`_  [] `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L743>`_ ()
```

Returns: 
```eval_rst
`Address <#type-address>`_  []
```



### Type Hex


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L607)


a Hexcoded String (starting with 0x)
 = `string`



### Type Address


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L619)


a 20 byte Address encoded as Hex (starting with 0x)
a Hexcoded String (starting with 0x)
 = `string`



### Type SignType


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L631)


the type of signature to create.
- ec_hash : the data needs to be hashed first ( using keccak) before signing
- ec_raw : the data is the ryw value (32bytes) to sign
- ec_prefix : the data is a message which needs to be prefixed with the EthereumSignedMessage and length to before hashing and signing
 = `'ec_hash'` | `'ec_prefix'` | `'ec_raw'`



### Type SignPayload


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L636)




### Type Utils


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L760)


Collection of different util-functions.



#### abiDecode()


decodes the given data as ABI-encoded (without the methodHash) 

```eval_rst
``any`` [] `abiDecode <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L780>`_ (
      signature:``string``,
      data:`Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | signature
       - | ``string``
       - | the method signature, which must contain a return description
     * - | data
       - | `Data <#type-data>`_ 
       - | the data to decode
         | 

```


Returns: 
```eval_rst
``any`` []
```



#### abiEncode()


encodes the given arguments as ABI-encoded (including the methodHash) 

```eval_rst
`Hex <#type-hex>`_  `abiEncode <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L773>`_ (
      signature:``string``,
      args:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | signature
       - | ``string``
       - | the method signature
     * - | args
       - | ``any`` []
       - | the arguments
         | 

```


Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### checkAddressChecksum()


checks whether the given address is a correct checksumAddress
If the chainId is passed, it will be included accord to EIP 1191 

```eval_rst
``boolean`` `checkAddressChecksum <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L796>`_ (
      address:`Address <#type-address>`_ ,
      chainId:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | the address (as hex)
     * - | chainId
       - | ``number``
       - | the chainId (if supported)
         | 

```


Returns: 
```eval_rst
``boolean``
```



#### createSignatureHash()


a Hexcoded String (starting with 0x) 

```eval_rst
`Hex <#type-hex>`_  `createSignatureHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L763>`_ (
      def:`ABI <#type-abi>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | def
       - | `ABI <#type-abi>`_ 
       - | def

```


Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### decodeEvent()


decode event 

```eval_rst
``any`` `decodeEvent <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L765>`_ (
      log:`Log <#type-log>`_ ,
      d:`ABI <#type-abi>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | log
       - | `Log <#type-log>`_ 
       - | log
     * - | d
       - | `ABI <#type-abi>`_ 
       - | d

```


Returns: 
```eval_rst
``any``
```



#### ecSign()


create a signature (65 bytes) for the given message and kexy 

```eval_rst
`BufferType <#type-buffertype>`_  `ecSign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L865>`_ (
      pk:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ ,
      msg:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ ,
      signType:`SignType <#type-signtype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the private key
     * - | msg
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the message
     * - | signType
       - | `SignType <#type-signtype>`_ 
       - | the type of signature to create
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### getVersion()


returns the incubed version. 

```eval_rst
``string`` `getVersion <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L831>`_ ()
```

Returns: 
```eval_rst
``string``
```



#### isAddress()


checks whether the given address is a valid hex string with 0x-prefix and 20 bytes 

```eval_rst
``boolean`` `isAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L802>`_ (
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | the address (as hex)
         | 

```


Returns: 
```eval_rst
``boolean``
```



#### keccak()


calculates the keccack hash for the given data. 

```eval_rst
`BufferType <#type-buffertype>`_  `keccak <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L808>`_ (
      data:`BufferType <#type-buffertype>`_  | `Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `BufferType <#type-buffertype>`_  
         | | `Data <#type-data>`_ 
       - | the data as Uint8Array or hex data.
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### private2address()


generates the public address from the private key. 

```eval_rst
`Address <#type-address>`_  `private2address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L879>`_ (
      pk:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the private key.
         | 

```


Returns: 
```eval_rst
`Address <#type-address>`_ 
```



#### private2public()


generates the public address (64 bytes) from the private key 

```eval_rst
`BufferType <#type-buffertype>`_  `private2public <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L885>`_ (
      pk:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the raw private key
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### randomBytes()


returns a Buffer with strong random bytes.
Thsi will use the browsers crypto-module or in case of nodejs use the crypto-module there. 

```eval_rst
`BufferType <#type-buffertype>`_  `randomBytes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L821>`_ (
      len:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | len
       - | ``number``
       - | the number of bytes to generate.
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### sha256()


calculates the sha256 hash for the given data. 

```eval_rst
`BufferType <#type-buffertype>`_  `sha256 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L814>`_ (
      data:`BufferType <#type-buffertype>`_  | `Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `BufferType <#type-buffertype>`_  
         | | `Data <#type-data>`_ 
       - | the data as Uint8Array or hex data.
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### soliditySha3()


solidity sha3 

```eval_rst
``string`` `soliditySha3 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L766>`_ (
      args:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | args
       - | ``any`` []
       - | args

```


Returns: 
```eval_rst
``string``
```



#### splitSignature()


takes raw signature (65 bytes) and splits it into a signature object. 

```eval_rst
`Signature <#type-signature>`_  `splitSignature <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L873>`_ (
      signature:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_ ,
      message:`BufferType <#type-buffertype>`_  | `Hex <#type-hex>`_ ,
      hashFirst:``boolean``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | signature
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_ 
       - | the 65 byte-signature
     * - | message
       - | `BufferType <#type-buffertype>`_  
         | | `Hex <#type-hex>`_ 
       - | the message
     * - | hashFirst
       - | ``boolean``
       - | if true (default) this will be taken as raw-data and will be hashed first.
         | 

```


Returns: 
```eval_rst
`Signature <#type-signature>`_ 
```



#### toBuffer()


converts any value to a Buffer.
optionally the target length can be specified (in bytes) 

```eval_rst
`BufferType <#type-buffertype>`_  `toBuffer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L846>`_ (
      data:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_  | ``number`` | ``bigint``,
      len:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_  
         | | ``number`` 
         | | ``bigint``
       - | data
     * - | len
       - | ``number``
       - | len

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### toChecksumAddress()


generates a checksum Address for the given address.
If the chainId is passed, it will be included accord to EIP 1191 

```eval_rst
`Address <#type-address>`_  `toChecksumAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L788>`_ (
      address:`Address <#type-address>`_ ,
      chainId:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | the address (as hex)
     * - | chainId
       - | ``number``
       - | the chainId (if supported)
         | 

```


Returns: 
```eval_rst
`Address <#type-address>`_ 
```



#### toHex()


converts any value to a hex string (with prefix 0x).
optionally the target length can be specified (in bytes) 

```eval_rst
`Hex <#type-hex>`_  `toHex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L827>`_ (
      data:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_  | ``number`` | ``bigint``,
      len:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_  
         | | ``number`` 
         | | ``bigint``
       - | data
     * - | len
       - | ``number``
       - | len

```


Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### toMinHex()


removes all leading 0 in the hexstring 

```eval_rst
``string`` `toMinHex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L834>`_ (
      key:``string`` | `BufferType <#type-buffertype>`_  | ``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | key
       - | ``string`` 
         | | `BufferType <#type-buffertype>`_  
         | | ``number``
       - | key

```


Returns: 
```eval_rst
``string``
```



#### toNumber()


converts any value to a hex string (with prefix 0x).
optionally the target length can be specified (in bytes) 

```eval_rst
``number`` `toNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L852>`_ (
      data:``string`` | `BufferType <#type-buffertype>`_  | ``number`` | ``bigint``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | ``string`` 
         | | `BufferType <#type-buffertype>`_  
         | | ``number`` 
         | | ``bigint``
       - | data

```


Returns: 
```eval_rst
``number``
```



#### toUint8Array()


converts any value to a Uint8Array.
optionally the target length can be specified (in bytes) 

```eval_rst
`BufferType <#type-buffertype>`_  `toUint8Array <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L840>`_ (
      data:`Hex <#type-hex>`_  | `BufferType <#type-buffertype>`_  | ``number`` | ``bigint``,
      len:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_  
         | | `BufferType <#type-buffertype>`_  
         | | ``number`` 
         | | ``bigint``
       - | data
     * - | len
       - | ``number``
       - | len

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### toUtf8()


convert to String 

```eval_rst
``string`` `toUtf8 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L857>`_ (
      val:``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | val
       - | ``any``
       - | val

```


Returns: 
```eval_rst
``string``
```



### Type IN3Config


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L39)


the configuration of the IN3-Client. This can be changed at any time.
All properties are optional and will be verified when sending the next request.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `autoUpdateList <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L52>`_
       - | ``boolean``
       - | if true the nodelist will be automaticly updated if the lastBlock is newer.
         | 
         | default: true
         |   *(optional)* 
     * - | `bootWeights <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L107>`_
       - | ``boolean``
       - | if true, the first request (updating the nodelist) will also fetch the current health status
         | and use it for blacklisting unhealthy nodes. This is used only if no nodelist is availabkle from cache.
         | 
         | default: false
         |   *(optional)* 
     * - | `btc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L262>`_
       - | `btc_config <#type-btc_config>`_ 
       - | config for btc  *(optional)* 
     * - | `chainId <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L67>`_
       - | ``string``
       - | The chain-id based on EIP-155.
         | or the name of the supported chain.
         | 
         | Currently we support 'mainnet', 'goerli', 'ipfs' and 'local'
         | 
         | While most of the chains use preconfigured chain settings,
         | 'local' actually uses the local running client turning of proof.
         | 
         | example: '0x1' or 'mainnet' or 'goerli'
         | 
         | default: 'mainnet'
         |  
     * - | `chainRegistry <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L194>`_
       - | ``string``
       - | main chain-registry contract
         | example: 0xe36179e2286ef405e929C90ad3E70E649B22a945  *(optional)* 
     * - | `experimental <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L99>`_
       - | ``boolean``
       - | if true, features marked as experimental are allowed.
         | Otherwise an exception would be thrown if those features would be used.
         | 
         | default: false
         |   *(optional)* 
     * - | `finality <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L82>`_
       - | ``number``
       - | the number in percent needed in order reach finality if you run on a POA-Chain.
         | (% of signature of the validators)
         | 
         | default: 0
         |   *(optional)* 
     * - | `includeCode <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L91>`_
       - | ``boolean``
       - | if true, the request should include the codes of all accounts.
         | Otherwise only the the codeHash is returned.
         | In this case the client may ask by calling eth_getCode() afterwards
         | 
         | default: false
         |   *(optional)* 
     * - | `keepIn3 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L125>`_
       - | ``boolean``
       - | if true, the in3-section of the response will be kept and returned.
         | Otherwise it will be removed after validating the data.
         | This is useful for debugging or if the proof should be used afterwards.
         | 
         | default: false
         |   *(optional)* 
     * - | `key <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L130>`_
       - | `Hash <#type-hash>`_ 
       - | the key to sign requests. This is required for payments.  *(optional)* 
     * - | `mainChain <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L199>`_
       - | ``string``
       - | main chain-id, where the chain registry is running.
         | example: 0x1  *(optional)* 
     * - | `maxAttempts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L115>`_
       - | ``number``
       - | max number of attempts in case a response is rejected.
         | Incubed will retry to find a different node giving a verified response.
         | 
         | default: 5
         |   *(optional)* 
     * - | `minDeposit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L157>`_
       - | ``number``
       - | min stake of the server. Only nodes owning at least this amount will be chosen.
         | 
         | default: 0
         |  
     * - | `nodeLimit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L137>`_
       - | ``number``
       - | the limit of nodes to store in the client. If set a random seed will be picked, which is the base for a deterministic verifiable partial nodelist.
         | 
         | default: 0
         |   *(optional)* 
     * - | `nodeProps <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L162>`_
       - | ``number`` | `Hex <#type-hex>`_ 
       - | a bitmask-value combining the minimal properties as filter for the selected nodes. See https://in3.readthedocs.io/en/develop/spec.html#node-structure for details. 
     * - | `nodes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L203>`_
       - | 
       - | the nodelists per chain. the chain_id will be used as key within the object.  *(optional)* 
     * - | `proof <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L169>`_
       - | ``'none'`` | ``'standard'`` | ``'full'``
       - | if true the nodes should send a proof of the response
         | 
         | default: 'standard'
         |   *(optional)* 
     * - | `replaceLatestBlock <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L176>`_
       - | ``number``
       - | if specified, the blocknumber *latest* will be replaced by blockNumber- specified value
         | 
         | default: 6
         |   *(optional)* 
     * - | `requestCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L183>`_
       - | ``number``
       - | the number of request send when getting a first answer
         | 
         | default: 1
         |  
     * - | `rpc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L188>`_
       - | ``string``
       - | url of a rpc-endpoints to use. If this is set proof will be turned off and it will be treated like local_chain.  *(optional)* 
     * - | `signatureCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L74>`_
       - | ``number``
       - | number of signatures requested. The more signatures, the more security you get, but responses may take longer.
         | 
         | default: 0
         |   *(optional)* 
     * - | `stats <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L144>`_
       - | ``boolean``
       - | if false, the requests will not be included in the stats of the nodes ( or marked as intern ).
         | 
         | default: true
         |   *(optional)* 
     * - | `timeout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L151>`_
       - | ``number``
       - | specifies the number of milliseconds before the request times out. increasing may be helpful if the device uses a slow connection.
         | 
         | default: 5000
         |   *(optional)* 
     * - | `zksync <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L260>`_
       - | `zksync_config <#type-zksync_config>`_ 
       - | config for zksync  *(optional)* 

```


#### transport()


sets the transport-function. 

```eval_rst
``Promise<string>`` `transport <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L45>`_ (
      url:``string``,
      payload:``string``,
      timeout:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | url
       - | ``string``
       - | url
     * - | payload
       - | ``string``
       - | payload
     * - | timeout
       - | ``number``
       - | timeout

```


Returns: 
```eval_rst
``Promise<string>``
```



### Type IN3


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L578)


default Incubed client with
bigint for big numbers
Uint8Array for bytes

```eval_rst
  .. list-table::
     :widths: auto

     * - | `default <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L570>`_
       - | 
       - | supporting both ES6 and UMD usage 
     * - | `util <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L556>`_
       - | `Utils<any> <#type-utils>`_ 
       - | collection of util-functions. 
     * - | `btc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L540>`_
       - | `BtcAPI<Uint8Array> <#type-btcapi>`_ 
       - | btc API 
     * - | `config <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L440>`_
       - | `IN3Config <#type-in3config>`_ 
       - | IN3 config 
     * - | `eth <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L532>`_
       - | `EthAPI<bigint,Uint8Array> <#type-ethapi>`_ 
       - | eth1 API. 
     * - | `ipfs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L538>`_
       - | `IpfsAPI<Uint8Array> <#type-ipfsapi>`_ 
       - | ipfs API 
     * - | `signer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L496>`_
       - | `Signer<bigint,Uint8Array> <#type-signer>`_ 
       - | the signer, if specified this interface will be used to sign transactions, if not, sending transaction will not be possible. 
     * - | `util <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L545>`_
       - | `Utils<Uint8Array> <#type-utils>`_ 
       - | collection of util-functions. 
     * - | `zksync <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L536>`_
       - | `ZksyncAPI<Uint8Array> <#type-zksyncapi>`_ 
       - | zksync API 

```


#### freeAll()


frees all Incubed instances. 

```eval_rst
static ``void`` `freeAll <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L526>`_ ()
```



#### onInit()


registers a function to be called as soon as the wasm is ready.
If it is already initialized it will call it right away. 

```eval_rst
static `Promise<T> <#type-t>`_  `onInit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L520>`_ (
      fn:() => `T <#type-t>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | fn
       - | () => `T <#type-t>`_ 
       - | the function to call

```


Returns: 
```eval_rst
static `Promise<T> <#type-t>`_ 
```



#### setConvertBigInt()


sets the convert-function, which converts any kind of type to Type defined for BigInt-operation.
if not set the default type would be bigint. 

```eval_rst
static ``void`` `setConvertBigInt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L590>`_ (
      convert:(``any``) => ``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | convert
       - | (``any``) => ``any``
       - | convert

```




#### setConvertBuffer()


sets the convert-function, which converts any kind of type to Type defined for Buffer or Bytes-operation.
if not set the default type would be UInt8Array. 

```eval_rst
static ``void`` `setConvertBuffer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L596>`_ (
      convert:(``any``) => ``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | convert
       - | (``any``) => ``any``
       - | convert

```




#### setStorage()


changes the storage handler, which is called to read and write to the cache. 

```eval_rst
static ``void`` `setStorage <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L509>`_ (
      handler:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | handler
       - | 
       - | handler

```




#### setTransport()


changes the default transport-function. 

```eval_rst
static ``void`` `setTransport <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L504>`_ (
      fn:(``string`` , ``string`` , ``number``) => ``Promise<string>``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | fn
       - | (``string`` , ``string`` , ``number``) => ``Promise<string>``
       - | the function to fetch the response for the given url
         | 

```




#### constructor()


creates a new client. 

```eval_rst
`IN3 <#type-in3>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L578>`_ (
      config:`Partial<IN3Config> <#type-in3config>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | config
       - | `Partial<IN3Config> <#type-in3config>`_ 
       - | a optional config
         | 

```


Returns: 
```eval_rst
`IN3 <#type-in3>`_ 
```



#### createWeb3Provider()


returns a Object, which can be used as Web3Provider.

```
const web3 = new Web3(new IN3().createWeb3Provider())
```
 

```eval_rst
``any`` `createWeb3Provider <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L491>`_ ()
```

Returns: 
```eval_rst
``any``
```



#### execLocal()


sends a RPC-Requests specified by name and params as a sync call. This is only alowed if the request is handled internally, like web3_sha3,

if the response contains an error, this will be thrown. if not the result will be returned.
 

```eval_rst
``any`` `execLocal <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L477>`_ (
      method:``string``,
      params:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | method
       - | ``string``
       - | the method to call.
         | 
     * - | params
       - | ``any`` []
       - | params

```


Returns: 
```eval_rst
``any``
```



#### free()


disposes the Client. This must be called in order to free allocated memory! 

```eval_rst
``void`` `free <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L482>`_ ()
```



#### registerPlugin()


rregisters a plugin. The plugin may define methods which will be called by the client. 

```eval_rst
``void`` `registerPlugin <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L551>`_ (
      plugin:`IN3Plugin<bigint,Uint8Array> <#type-in3plugin>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | plugin
       - | `IN3Plugin<bigint,Uint8Array> <#type-in3plugin>`_ 
       - | the plugin-object to register
         | 

```




#### send()


sends a raw request.
if the request is a array the response will be a array as well.
If the callback is given it will be called with the response, if not a Promise will be returned.
This function supports callback so it can be used as a Provider for the web3. 

```eval_rst
`Promise<RPCResponse> <#type-rpcresponse>`_  `send <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L458>`_ (
      request:`RPCRequest <#type-rpcrequest>`_ ,
      callback:(`Error <#type-error>`_  , `RPCResponse <#type-rpcresponse>`_ ) => ``void``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | request
       - | `RPCRequest <#type-rpcrequest>`_ 
       - | a JSONRPC-Request with N3-Extension
     * - | callback
       - | (`Error <#type-error>`_  , `RPCResponse <#type-rpcresponse>`_ ) => ``void``
       - | callback

```


Returns: 
```eval_rst
`Promise<RPCResponse> <#type-rpcresponse>`_ 
```



#### sendRPC()


sends a RPC-Requests specified by name and params.

if the response contains an error, this will be thrown. if not the result will be returned.
 

```eval_rst
``Promise<any>`` `sendRPC <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L467>`_ (
      method:``string``,
      params:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | method
       - | ``string``
       - | the method to call.
         | 
     * - | params
       - | ``any`` []
       - | params

```


Returns: 
```eval_rst
``Promise<any>``
```



#### setConfig()


sets configuration properties. You can pass a partial object specifieing any of defined properties. 

```eval_rst
``void`` `setConfig <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L450>`_ (
      config:`Partial<IN3Config> <#type-in3config>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | config
       - | `Partial<IN3Config> <#type-in3config>`_ 
       - | config

```




### Type BtcAPI


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2083)


API for handling BitCoin data



#### getBlockBytes()


retrieves the serialized block (bytes) including all transactions 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `getBlockBytes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2103>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### getBlockCount()


Returns the number of blocks in the longest blockchain. 

```eval_rst
`Promise<Number> <#type-number>`_  `getBlockCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2106>`_ ()
```

Returns: 
```eval_rst
`Promise<Number> <#type-number>`_ 
```



#### getBlockHeader()


retrieves the blockheader and returns the data as json. 

```eval_rst
`Promise<BTCBlockHeader> <#type-btcblockheader>`_  `getBlockHeader <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2091>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BTCBlockHeader> <#type-btcblockheader>`_ 
```



#### getBlockHeaderBytes()


retrieves the serialized blockheader (bytes) 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `getBlockHeaderBytes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2094>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### getBlockWithTxData()


retrieves the block including all tx data as json. 

```eval_rst
`Promise<BTCBlock> <#type-btcblock>`_  `getBlockWithTxData <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2097>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BTCBlock> <#type-btcblock>`_ 
```



#### getBlockWithTxIds()


retrieves the block including all tx ids as json. 

```eval_rst
`Promise<BTCBlock> <#type-btcblock>`_  `getBlockWithTxIds <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2100>`_ (
      blockHash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | blockHash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BTCBlock> <#type-btcblock>`_ 
```



#### getTransaction()


retrieves the transaction and returns the data as json. 

```eval_rst
`Promise<BtcTransaction> <#type-btctransaction>`_  `getTransaction <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2085>`_ (
      txid:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | txid
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BtcTransaction> <#type-btctransaction>`_ 
```



#### getTransactionBytes()


retrieves the serialized transaction (bytes) 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `getTransactionBytes <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2088>`_ (
      txid:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | txid
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



### Type EthAPI


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1587)


The API for ethereum operations.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `accounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1601>`_
       - | `AccountAPI<BufferType> <#type-accountapi>`_ 
       - | accounts-API 
     * - | `client <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1591>`_
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | the client used. 
     * - | `signer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1596>`_
       - | `Signer<BigIntType,BufferType> <#type-signer>`_ 
       - | a custom signer  *(optional)* 

```


#### blockNumber()


Returns the number of most recent block. (as number) 

```eval_rst
``Promise<number>`` `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1607>`_ ()
```

Returns: 
```eval_rst
``Promise<number>``
```



#### call()


Executes a new message call immediately without creating a transaction on the block chain. 

```eval_rst
``Promise<string>`` `call <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1615>`_ (
      tx:`Transaction <#type-transaction>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tx
       - | `Transaction <#type-transaction>`_ 
       - | tx
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<string>``
```



#### callFn()


Executes a function of a contract, by passing a [method-signature](https://github.com/ethereumjs/ethereumjs-abi/blob/master/README.md#simple-encoding-and-decoding) and the arguments, which will then be ABI-encoded and send as eth_call. 

```eval_rst
``Promise<any>`` `callFn <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1619>`_ (
      to:`Address <#type-address>`_ ,
      method:``string``,
      args:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | to
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | method
       - | ``string``
       - | method
     * - | args
       - | ``any`` []
       - | args

```


Returns: 
```eval_rst
``Promise<any>``
```



#### chainId()


Returns the EIP155 chain ID used for transaction signing at the current best block. Null is returned if not available. 

```eval_rst
``Promise<string>`` `chainId <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1623>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### clientVersion()


Returns the clientVersion. This may differ in case of an network, depending on the node it communicates with. 

```eval_rst
``Promise<string>`` `clientVersion <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1627>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### constructor()


The API for ethereum operations. 

```eval_rst
`EthAPI<BigIntType,BufferType> <#type-ethapi>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1603>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | client

```


Returns: 
```eval_rst
`EthAPI<BigIntType,BufferType> <#type-ethapi>`_ 
```



#### contractAt()


contract at 

```eval_rst
 `contractAt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1802>`_ (
      abi:`ABI <#type-abi>`_  [],
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | abi
       - | `ABI <#type-abi>`_  []
       - | abi
     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)

```




#### decodeEventData()


decode event data 

```eval_rst
``any`` `decodeEventData <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1842>`_ (
      log:`Log <#type-log>`_ ,
      d:`ABI <#type-abi>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | log
       - | `Log <#type-log>`_ 
       - | log
     * - | d
       - | `ABI <#type-abi>`_ 
       - | d

```


Returns: 
```eval_rst
``any``
```



#### estimateGas()


Makes a call or transaction, which won’t be added to the blockchain and returns the used gas, which can be used for estimating the used gas. 

```eval_rst
``Promise<number>`` `estimateGas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1631>`_ (
      tx:`Transaction <#type-transaction>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tx
       - | `Transaction <#type-transaction>`_ 
       - | tx

```


Returns: 
```eval_rst
``Promise<number>``
```



#### fromWei()


Returns a formated String in the specified unit (or eth if not specified). If digits are specified, the number of digits behind the comma can be limited. 

```eval_rst
``string`` `fromWei <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1761>`_ (
      value:``string``,
      unit:``string``,
      digits:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | value
       - | ``string``
       - | value
     * - | unit
       - | ``string``
       - | unit
     * - | digits
       - | ``number``
       - | digits

```


Returns: 
```eval_rst
``string``
```



#### gasPrice()


Returns the current price per gas in wei. (as number) 

```eval_rst
``Promise<number>`` `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1611>`_ ()
```

Returns: 
```eval_rst
``Promise<number>``
```



#### getAccounts()


returns the public addresses  accounts 

```eval_rst
``Promise<>`` `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1692>`_ ()
```

Returns: 
```eval_rst
``Promise<>``
```



#### getBalance()


Returns the balance of the account of given address in wei (as hex). 

```eval_rst
`Promise<BigIntType> <#type-biginttype>`_  `getBalance <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1635>`_ (
      address:`Address <#type-address>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
`Promise<BigIntType> <#type-biginttype>`_ 
```



#### getBlockByHash()


Returns information about a block by hash. 

```eval_rst
`Promise<Block> <#type-block>`_  `getBlockByHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1647>`_ (
      hash:`Hash <#type-hash>`_ ,
      includeTransactions:``boolean``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)
     * - | includeTransactions
       - | ``boolean``
       - | include transactions

```


Returns: 
```eval_rst
`Promise<Block> <#type-block>`_ 
```



#### getBlockByNumber()


Returns information about a block by block number. 

```eval_rst
`Promise<Block> <#type-block>`_  `getBlockByNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1651>`_ (
      block:`BlockType <#type-blocktype>`_ ,
      includeTransactions:``boolean``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block
     * - | includeTransactions
       - | ``boolean``
       - | include transactions

```


Returns: 
```eval_rst
`Promise<Block> <#type-block>`_ 
```



#### getBlockTransactionCountByHash()


Returns the number of transactions in a block from a block matching the given block hash. 

```eval_rst
``Promise<number>`` `getBlockTransactionCountByHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1655>`_ (
      block:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<number>``
```



#### getBlockTransactionCountByNumber()


Returns the number of transactions in a block from a block matching the given block number. 

```eval_rst
``Promise<number>`` `getBlockTransactionCountByNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1659>`_ (
      block:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<number>``
```



#### getCode()


Returns code at a given address. 

```eval_rst
``Promise<string>`` `getCode <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1639>`_ (
      address:`Address <#type-address>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<string>``
```



#### getFilterChanges()


Polling method for a filter, which returns an array of logs which occurred since last poll. 

```eval_rst
``Promise<>`` `getFilterChanges <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1663>`_ (
      id:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | id
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
``Promise<>``
```



#### getFilterLogs()


Returns an array of all logs matching filter with given id. 

```eval_rst
``Promise<>`` `getFilterLogs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1667>`_ (
      id:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | id
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
``Promise<>``
```



#### getLogs()


Returns an array of all logs matching a given filter object. 

```eval_rst
``Promise<>`` `getLogs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1671>`_ (
      filter:`LogFilter <#type-logfilter>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | filter
       - | `LogFilter <#type-logfilter>`_ 
       - | filter

```


Returns: 
```eval_rst
``Promise<>``
```



#### getStorageAt()


Returns the value from a storage position at a given address. 

```eval_rst
``Promise<string>`` `getStorageAt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1643>`_ (
      address:`Address <#type-address>`_ ,
      pos:`Quantity <#type-quantity>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<string>``
```



#### getTransactionByBlockHashAndIndex()


Returns information about a transaction by block hash and transaction index position. 

```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_  `getTransactionByBlockHashAndIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1675>`_ (
      hash:`Hash <#type-hash>`_ ,
      pos:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_ 
```



#### getTransactionByBlockNumberAndIndex()


Returns information about a transaction by block number and transaction index position. 

```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_  `getTransactionByBlockNumberAndIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1679>`_ (
      block:`BlockType <#type-blocktype>`_ ,
      pos:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_ 
```



#### getTransactionByHash()


Returns the information about a transaction requested by transaction hash. 

```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_  `getTransactionByHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1683>`_ (
      hash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<TransactionDetail> <#type-transactiondetail>`_ 
```



#### getTransactionCount()


Returns the number of transactions sent from an address. (as number) 

```eval_rst
``Promise<number>`` `getTransactionCount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1687>`_ (
      address:`Address <#type-address>`_ ,
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<number>``
```



#### getTransactionReceipt()


Returns the receipt of a transaction by transaction hash.
Note That the receipt is available even for pending transactions. 

```eval_rst
`Promise<TransactionReceipt> <#type-transactionreceipt>`_  `getTransactionReceipt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1698>`_ (
      hash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Promise<TransactionReceipt> <#type-transactionreceipt>`_ 
```



#### getUncleByBlockHashAndIndex()


Returns information about a uncle of a block by hash and uncle index position.
Note: An uncle doesn’t contain individual transactions. 

```eval_rst
`Promise<Block> <#type-block>`_  `getUncleByBlockHashAndIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1703>`_ (
      hash:`Hash <#type-hash>`_ ,
      pos:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<Block> <#type-block>`_ 
```



#### getUncleByBlockNumberAndIndex()


Returns information about a uncle of a block number and uncle index position.
Note: An uncle doesn’t contain individual transactions. 

```eval_rst
`Promise<Block> <#type-block>`_  `getUncleByBlockNumberAndIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1708>`_ (
      block:`BlockType <#type-blocktype>`_ ,
      pos:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block
     * - | pos
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<Block> <#type-block>`_ 
```



#### getUncleCountByBlockHash()


Returns the number of uncles in a block from a block matching the given block hash. 

```eval_rst
``Promise<number>`` `getUncleCountByBlockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1712>`_ (
      hash:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | hash
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<number>``
```



#### getUncleCountByBlockNumber()


Returns the number of uncles in a block from a block matching the given block hash. 

```eval_rst
``Promise<number>`` `getUncleCountByBlockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1716>`_ (
      block:`BlockType <#type-blocktype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | block
       - | `BlockType <#type-blocktype>`_ 
       - | BlockNumber or predefined Block

```


Returns: 
```eval_rst
``Promise<number>``
```



#### hashMessage()


a Hexcoded String (starting with 0x) 

```eval_rst
`Hex <#type-hex>`_  `hashMessage <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1843>`_ (
      data:`Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Data <#type-data>`_ 
       - | data encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### newBlockFilter()


Creates a filter in the node, to notify when a new block arrives. To check if the state has changed, call eth_getFilterChanges. 

```eval_rst
``Promise<string>`` `newBlockFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1726>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### newFilter()


Creates a filter object, based on filter options, to notify when the state changes (logs). To check if the state has changed, call eth_getFilterChanges.

A note on specifying topic filters:
Topics are order-dependent. A transaction with a log with topics [A, B] will be matched by the following topic filters:

[] “anything”
[A] “A in first position (and anything after)”
[null, B] “anything in first position AND B in second position (and anything after)”
[A, B] “A in first position AND B in second position (and anything after)”
[[A, B], [A, B]] “(A OR B) in first position AND (A OR B) in second position (and anything after)”
 

```eval_rst
``Promise<string>`` `newFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1739>`_ (
      filter:`LogFilter <#type-logfilter>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | filter
       - | `LogFilter <#type-logfilter>`_ 
       - | filter

```


Returns: 
```eval_rst
``Promise<string>``
```



#### newPendingFilter()


adds a filter for pending transaction (only available for local rpc) 

```eval_rst
``Promise<string>`` `newPendingFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1721>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### newPendingTransactionFilter()


Creates a filter in the node, to notify when new pending transactions arrive.

To check if the state has changed, call eth_getFilterChanges.
 

```eval_rst
``Promise<string>`` `newPendingTransactionFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1745>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### protocolVersion()


Returns the current ethereum protocol version. 

```eval_rst
``Promise<string>`` `protocolVersion <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1753>`_ ()
```

Returns: 
```eval_rst
``Promise<string>``
```



#### resolveENS()


resolves a name as an ENS-Domain. 

```eval_rst
`Promise<Address> <#type-address>`_  `resolveENS <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1780>`_ (
      name:``string``,
      type:`Address <#type-address>`_ ,
      registry:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | name
       - | ``string``
       - | the domain name
     * - | type
       - | `Address <#type-address>`_ 
       - | the type (currently only addr is supported)
     * - | registry
       - | ``string``
       - | optionally the address of the registry (default is the mainnet ens registry)
         | 

```


Returns: 
```eval_rst
`Promise<Address> <#type-address>`_ 
```



#### sendRawTransaction()


Creates new message call transaction or a contract creation for signed transactions. 

```eval_rst
``Promise<string>`` `sendRawTransaction <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1785>`_ (
      data:`Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Data <#type-data>`_ 
       - | data encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<string>``
```



#### sendTransaction()


sends a Transaction 

```eval_rst
``Promise<>`` `sendTransaction <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1793>`_ (
      args:`TxRequest <#type-txrequest>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | args
       - | `TxRequest <#type-txrequest>`_ 
       - | args

```


Returns: 
```eval_rst
``Promise<>``
```



#### sign()


signs any kind of message using the `\x19Ethereum Signed Message:\n`-prefix 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1791>`_ (
      account:`Address <#type-address>`_ ,
      data:`Data <#type-data>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | account
       - | `Address <#type-address>`_ 
       - | the address to sign the message with (if this is a 32-bytes hex-string it will be used as private key)
     * - | data
       - | `Data <#type-data>`_ 
       - | the data to sign (Buffer, hexstring or utf8-string)
         | 

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### syncing()


Returns the state of the underlying node. 

```eval_rst
``Promise<>`` `syncing <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1765>`_ ()
```

Returns: 
```eval_rst
``Promise<>``
```



#### toWei()


Returns the value in wei as hexstring. 

```eval_rst
``string`` `toWei <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1757>`_ (
      value:``string``,
      unit:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | value
       - | ``string``
       - | value
     * - | unit
       - | ``string``
       - | unit

```


Returns: 
```eval_rst
``string``
```



#### uninstallFilter()


Uninstalls a filter with given id. Should always be called when watch is no longer needed. Additonally Filters timeout when they aren’t requested with eth_getFilterChanges for a period of time. 

```eval_rst
`Promise<Quantity> <#type-quantity>`_  `uninstallFilter <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1749>`_ (
      id:`Quantity <#type-quantity>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | id
       - | `Quantity <#type-quantity>`_ 
       - | a BigInteger encoded as hex.

```


Returns: 
```eval_rst
`Promise<Quantity> <#type-quantity>`_ 
```



#### web3ContractAt()


web3 contract at 

```eval_rst
`Web3Contract <#type-web3contract>`_  `web3ContractAt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1795>`_ (
      abi:`ABI <#type-abi>`_  [],
      address:`Address <#type-address>`_ ,
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | abi
       - | `ABI <#type-abi>`_  []
       - | abi
     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
`Web3Contract <#type-web3contract>`_ 
```



### Type IpfsAPI


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1883)


API for storing and retrieving IPFS-data.



#### get()


retrieves the content for a hash from IPFS. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `get <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1889>`_ (
      multihash:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | multihash
       - | ``string``
       - | the IPFS-hash to fetch
         | 
         | 

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### put()


stores the data on ipfs and returns the IPFS-Hash. 

```eval_rst
``Promise<string>`` `put <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1894>`_ (
      content:`BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | content
       - | `BufferType <#type-buffertype>`_ 
       - | puts a IPFS content
         | 

```


Returns: 
```eval_rst
``Promise<string>``
```



### Type Signer


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L697)






#### sign()


signing of any data.
if hashFirst is true the data should be hashed first, otherwise the data is the hash. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L709>`_ (
      data:`Hex <#type-hex>`_ ,
      account:`Address <#type-address>`_ ,
      sign_type:`SignType <#type-signtype>`_ ,
      payloadType:`SignPayload <#type-signpayload>`_ ,
      meta:``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_ 
       - | a Hexcoded String (starting with 0x)
     * - | account
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | sign_type
       - | `SignType <#type-signtype>`_ 
       - | the type of signature to create.
         |     - ec_hash : the data needs to be hashed first ( using keccak) before signing
         |     - ec_raw : the data is the ryw value (32bytes) to sign
         |     - ec_prefix : the data is a message which needs to be prefixed with the EthereumSignedMessage and length to before hashing and signing
     * - | payloadType
       - | `SignPayload <#type-signpayload>`_ 
       - | The type of the payload to sign
     * - | meta
       - | ``any``
       - | meta

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### canSign()


returns true if the account is supported (or unlocked) 

```eval_rst
``Promise<boolean>`` `canSign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L700>`_ (
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<boolean>``
```



#### getAccounts()


returns all addresses managed by the signer. 

```eval_rst
`Address <#type-address>`_  [] `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L703>`_ ()
```

Returns: 
```eval_rst
`Address <#type-address>`_  []
```



### Type ZksyncAPI


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1017)


API for zksync.



#### aggregatePubKey()


aggregates the given publickeys into one public key for a Musig Schnorr signature 

```eval_rst
``string`` `aggregatePubKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1138>`_ (
      pubkeys:``string`` | ``string`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pubkeys
       - | ``string`` | ``string`` []
       - | the packed public keys (either a concatenated string or as array of public keys)
         | 

```


Returns: 
```eval_rst
``string``
```



#### deposit()


deposits the declared amount into the rollup 

```eval_rst
`Promise<DepositResponse> <#type-depositresponse>`_  `deposit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1095>`_ (
      amount:``number``,
      token:``string``,
      approveDepositAmountForERC20:``boolean``,
      account:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | amount
       - | ``number``
       - | amount in wei to deposit
     * - | token
       - | ``string``
       - | the token identifier e.g. ETH
     * - | approveDepositAmountForERC20
       - | ``boolean``
       - | bool that is set to true if it is a erc20 token that needs approval
     * - | account
       - | ``string``
       - | address of the account that wants to deposit (if left empty it will be taken from current signer)
         | 

```


Returns: 
```eval_rst
`Promise<DepositResponse> <#type-depositresponse>`_ 
```



#### emergencyWithdraw()


executes an emergency withdrawel onchain 

```eval_rst
`Promise<String> <#type-string>`_  `emergencyWithdraw <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1119>`_ (
      token:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | token
       - | ``string``
       - | the token identifier e.g. ETH
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### getAccountAddress()


returns the address of the account used. 

```eval_rst
`String <#type-string>`_  `getAccountAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1033>`_ ()
```

Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### getAccountInfo()


gets current account Infoa and balances. 

```eval_rst
`Promise<ZKAccountInfo> <#type-zkaccountinfo>`_  `getAccountInfo <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1023>`_ (
      account:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | account
       - | ``string``
       - | the address of the account . if not specified, the first signer is used.
         | 

```


Returns: 
```eval_rst
`Promise<ZKAccountInfo> <#type-zkaccountinfo>`_ 
```



#### getContractAddress()


gets the contract address of the zksync contract 

```eval_rst
`Promise<String> <#type-string>`_  `getContractAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1028>`_ ()
```

Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### getEthopInfo()


returns the state of receipt of the PriorityOperation 

```eval_rst
`Promise<ETHOpInfoResp> <#type-ethopinforesp>`_  `getEthopInfo <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1057>`_ (
      opId:``number``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | opId
       - | ``number``
       - | the id of the PriorityOperation
         | 

```


Returns: 
```eval_rst
`Promise<ETHOpInfoResp> <#type-ethopinforesp>`_ 
```



#### getSyncKey()


returns private key used for signing zksync transactions 

```eval_rst
`String <#type-string>`_  `getSyncKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1076>`_ ()
```

Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### getSyncPubKey()


returns public key used for signing zksync transactions 

```eval_rst
`String <#type-string>`_  `getSyncPubKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1086>`_ ()
```

Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### getSyncPubKeyHash()


returns public key used for signing zksync transactions 

```eval_rst
`String <#type-string>`_  `getSyncPubKeyHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1081>`_ ()
```

Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### getTokenPrice()


returns the current token price 

```eval_rst
`Promise<Number> <#type-number>`_  `getTokenPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1063>`_ (
      tokenSymbol:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tokenSymbol
       - | ``string``
       - | the address of the token
         | 

```


Returns: 
```eval_rst
`Promise<Number> <#type-number>`_ 
```



#### getTokens()


returns an object containing Token objects with its short name as key 

```eval_rst
`Promise<Tokens> <#type-tokens>`_  `getTokens <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1038>`_ ()
```

Returns: 
```eval_rst
`Promise<Tokens> <#type-tokens>`_ 
```



#### getTxFee()


returns the transaction fee 

```eval_rst
`Promise<Fee> <#type-fee>`_  `getTxFee <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1071>`_ (
      txType:`TxType <#type-txtype>`_ ,
      receipient:``string``,
      token:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | txType
       - | `TxType <#type-txtype>`_ 
       - | either Withdraw or Transfer
     * - | receipient
       - | ``string``
       - | the address the transaction is send to
     * - | token
       - | ``string``
       - | the token identifier e.g. ETH
         | 

```


Returns: 
```eval_rst
`Promise<Fee> <#type-fee>`_ 
```



#### getTxInfo()


get transaction info 

```eval_rst
`Promise<TxInfo> <#type-txinfo>`_  `getTxInfo <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1044>`_ (
      txHash:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | txHash
       - | ``string``
       - | the has of the tx you want the info about
         | 

```


Returns: 
```eval_rst
`Promise<TxInfo> <#type-txinfo>`_ 
```



#### setKey()


set the signer key based on the current pk 

```eval_rst
`Promise<String> <#type-string>`_  `setKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1051>`_ (
      tokenSymbol:``string``,
      newKey:`BufferType <#type-buffertype>`_  | ``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tokenSymbol
       - | ``string``
       - | the address of the token
     * - | newKey
       - | `BufferType <#type-buffertype>`_  | ``string``
       - | the seed of the new key ( this is optional, if ommited the derrived key will be set in the rollup)
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### sign()


signs the message based on the config and returns a Musig Schnorr signature 

```eval_rst
`Promise<String> <#type-string>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1125>`_ (
      msg:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string``
       - | the message
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### transfer()


transfers the specified amount to another address within the zksync rollup 

```eval_rst
`Promise<String> <#type-string>`_  `transfer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1104>`_ (
      to:``string``,
      amount:``number``,
      token:``string``,
      account:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | to
       - | ``string``
       - | address of the receipient
     * - | amount
       - | ``number``
       - | amount to send in wei
     * - | token
       - | ``string``
       - | the token indentifier e.g. ETH
     * - | account
       - | ``string``
       - | address of the account that wants to transfer (if left empty it will be taken from current signer)
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



#### verify()


verifies a Musig Schnorr signature 

```eval_rst
``boolean`` `verify <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1132>`_ (
      msg:``string``,
      sig:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string``
       - | the message
     * - | sig
       - | ``string``
       - | the 96byte long signature
         | 

```


Returns: 
```eval_rst
``boolean``
```



#### withdraw()


withdraws the specified amount from the rollup to a specific address 

```eval_rst
`Promise<String> <#type-string>`_  `withdraw <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1113>`_ (
      ethAddress:``string``,
      amount:``number``,
      token:``string``,
      account:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | ethAddress
       - | ``string``
       - | the receipient address
     * - | amount
       - | ``number``
       - | amount to withdraw in wei
     * - | token
       - | ``string``
       - | the token identifier e.g. ETH
     * - | account
       - | ``string``
       - | address of the account that wants to withdraw (if left empty it will be taken from current signer)
         | 

```


Returns: 
```eval_rst
`Promise<String> <#type-string>`_ 
```



### Type IN3Plugin


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L407)


a Incubed plugin.

Depending on the methods this will register for those actions.




#### getAccount()


returns address  *(optional)* 

```eval_rst
``string`` `getAccount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L418>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | 
         | 

```


Returns: 
```eval_rst
``string``
```



#### getAccounts()


returns list of addresses  *(optional)* 

```eval_rst
`Address <#type-address>`_  [] `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L424>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | 
         | 

```


Returns: 
```eval_rst
`Address <#type-address>`_  []
```



#### handleRPC()


called for each request.
If the plugin wants to handle the request, this function should return the value or a Promise for the value.
If the plugin does not want to handle it, it should rreturn undefined.  *(optional)* 

```eval_rst
``undefined`` | ``Promise<any>`` `handleRPC <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L433>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ ,
      request:`RPCRequest <#type-rpcrequest>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | the current client
     * - | request
       - | `RPCRequest <#type-rpcrequest>`_ 
       - | the rpc-request
         | 

```


Returns: 
```eval_rst
``undefined`` | ``Promise<any>``
```



#### term()


this is called when the client is cleaned up.  *(optional)* 

```eval_rst
``void`` `term <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L412>`_ (
      client:`IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | client
       - | `IN3Generic<BigIntType,BufferType> <#type-in3generic>`_ 
       - | the client object
         | 

```




### Type RPCRequest


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L357)


a JSONRPC-Request with N3-Extension

```eval_rst
  .. list-table::
     :widths: auto

     * - | `id <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L371>`_
       - | ``number`` | ``string``
       - | the identifier of the request
         | example: 2  *(optional)* 
     * - | `jsonrpc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L361>`_
       - | ``'2.0'``
       - | the version 
     * - | `method <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L366>`_
       - | ``string``
       - | the method to call
         | example: eth_getBalance 
     * - | `params <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L376>`_
       - | ``any`` []
       - | the params
         | example: 0xe36179e2286ef405e929C90ad3E70E649B22a945,latest  *(optional)* 

```


### Type RPCResponse


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L381)


a JSONRPC-Responset with N3-Extension

```eval_rst
  .. list-table::
     :widths: auto

     * - | `error <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L394>`_
       - | ``string`` | 
       - | in case of an error this needs to be set  *(optional)* 
     * - | `id <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L390>`_
       - | ``string`` | ``number``
       - | the id matching the request
         | example: 2 
     * - | `jsonrpc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L385>`_
       - | ``'2.0'``
       - | the version 
     * - | `result <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L399>`_
       - | ``any``
       - | the params
         | example: 0xa35bc  *(optional)* 

```


### Type IN3Generic


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L436)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `default <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L570>`_
       - | 
       - | supporting both ES6 and UMD usage 
     * - | `util <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L556>`_
       - | `Utils<any> <#type-utils>`_ 
       - | collection of util-functions. 
     * - | `btc <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L540>`_
       - | `BtcAPI<BufferType> <#type-btcapi>`_ 
       - | btc API 
     * - | `config <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L440>`_
       - | `IN3Config <#type-in3config>`_ 
       - | IN3 config 
     * - | `eth <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L532>`_
       - | `EthAPI<BigIntType,BufferType> <#type-ethapi>`_ 
       - | eth1 API. 
     * - | `ipfs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L538>`_
       - | `IpfsAPI<BufferType> <#type-ipfsapi>`_ 
       - | ipfs API 
     * - | `signer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L496>`_
       - | `Signer<BigIntType,BufferType> <#type-signer>`_ 
       - | the signer, if specified this interface will be used to sign transactions, if not, sending transaction will not be possible. 
     * - | `util <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L545>`_
       - | `Utils<BufferType> <#type-utils>`_ 
       - | collection of util-functions. 
     * - | `zksync <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L536>`_
       - | `ZksyncAPI<BufferType> <#type-zksyncapi>`_ 
       - | zksync API 

```


#### freeAll()


frees all Incubed instances. 

```eval_rst
static ``void`` `freeAll <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L526>`_ ()
```



#### onInit()


registers a function to be called as soon as the wasm is ready.
If it is already initialized it will call it right away. 

```eval_rst
static `Promise<T> <#type-t>`_  `onInit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L520>`_ (
      fn:() => `T <#type-t>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | fn
       - | () => `T <#type-t>`_ 
       - | the function to call

```


Returns: 
```eval_rst
static `Promise<T> <#type-t>`_ 
```



#### setConvertBigInt()


sets the convert-function, which converts any kind of type to Type defined for BigInt-operation.
if not set the default type would be bigint. 

```eval_rst
static ``void`` `setConvertBigInt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L561>`_ (
      convert:(``any``) => ``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | convert
       - | (``any``) => ``any``
       - | convert

```




#### setConvertBuffer()


sets the convert-function, which converts any kind of type to Type defined for Buffer or Bytes-operation.
if not set the default type would be UInt8Array. 

```eval_rst
static ``void`` `setConvertBuffer <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L567>`_ (
      convert:(``any``) => ``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | convert
       - | (``any``) => ``any``
       - | convert

```




#### setStorage()


changes the storage handler, which is called to read and write to the cache. 

```eval_rst
static ``void`` `setStorage <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L509>`_ (
      handler:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | handler
       - | 
       - | handler

```




#### setTransport()


changes the default transport-function. 

```eval_rst
static ``void`` `setTransport <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L504>`_ (
      fn:(``string`` , ``string`` , ``number``) => ``Promise<string>``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | fn
       - | (``string`` , ``string`` , ``number``) => ``Promise<string>``
       - | the function to fetch the response for the given url
         | 

```




#### constructor()


creates a new client. 

```eval_rst
`IN3Generic <#type-in3generic>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L440>`_ (
      config:`Partial<IN3Config> <#type-in3config>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | config
       - | `Partial<IN3Config> <#type-in3config>`_ 
       - | a optional config
         | 

```


Returns: 
```eval_rst
`IN3Generic <#type-in3generic>`_ 
```



#### createWeb3Provider()


returns a Object, which can be used as Web3Provider.

```
const web3 = new Web3(new IN3().createWeb3Provider())
```
 

```eval_rst
``any`` `createWeb3Provider <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L491>`_ ()
```

Returns: 
```eval_rst
``any``
```



#### execLocal()


sends a RPC-Requests specified by name and params as a sync call. This is only alowed if the request is handled internally, like web3_sha3,

if the response contains an error, this will be thrown. if not the result will be returned.
 

```eval_rst
``any`` `execLocal <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L477>`_ (
      method:``string``,
      params:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | method
       - | ``string``
       - | the method to call.
         | 
     * - | params
       - | ``any`` []
       - | params

```


Returns: 
```eval_rst
``any``
```



#### free()


disposes the Client. This must be called in order to free allocated memory! 

```eval_rst
``void`` `free <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L482>`_ ()
```



#### registerPlugin()


rregisters a plugin. The plugin may define methods which will be called by the client. 

```eval_rst
``void`` `registerPlugin <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L551>`_ (
      plugin:`IN3Plugin<BigIntType,BufferType> <#type-in3plugin>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | plugin
       - | `IN3Plugin<BigIntType,BufferType> <#type-in3plugin>`_ 
       - | the plugin-object to register
         | 

```




#### send()


sends a raw request.
if the request is a array the response will be a array as well.
If the callback is given it will be called with the response, if not a Promise will be returned.
This function supports callback so it can be used as a Provider for the web3. 

```eval_rst
`Promise<RPCResponse> <#type-rpcresponse>`_  `send <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L458>`_ (
      request:`RPCRequest <#type-rpcrequest>`_ ,
      callback:(`Error <#type-error>`_  , `RPCResponse <#type-rpcresponse>`_ ) => ``void``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | request
       - | `RPCRequest <#type-rpcrequest>`_ 
       - | a JSONRPC-Request with N3-Extension
     * - | callback
       - | (`Error <#type-error>`_  , `RPCResponse <#type-rpcresponse>`_ ) => ``void``
       - | callback

```


Returns: 
```eval_rst
`Promise<RPCResponse> <#type-rpcresponse>`_ 
```



#### sendRPC()


sends a RPC-Requests specified by name and params.

if the response contains an error, this will be thrown. if not the result will be returned.
 

```eval_rst
``Promise<any>`` `sendRPC <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L467>`_ (
      method:``string``,
      params:``any`` [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | method
       - | ``string``
       - | the method to call.
         | 
     * - | params
       - | ``any`` []
       - | params

```


Returns: 
```eval_rst
``Promise<any>``
```



#### setConfig()


sets configuration properties. You can pass a partial object specifieing any of defined properties. 

```eval_rst
``void`` `setConfig <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L450>`_ (
      config:`Partial<IN3Config> <#type-in3config>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | config
       - | `Partial<IN3Config> <#type-in3config>`_ 
       - | config

```




### Type Hash


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L615)


a 32 byte Hash encoded as Hex (starting with 0x)
a Hexcoded String (starting with 0x)
 = `string`



### Type SimpleSigner


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L712)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `accounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L713>`_
       - | 
       - | the accounts 

```


#### constructor()


constructor 

```eval_rst
`SimpleSigner <#type-simplesigner>`_  `constructor <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L715>`_ (
      pks:`Hash <#type-hash>`_  | `BufferType <#type-buffertype>`_  [])
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pks
       - | `Hash <#type-hash>`_  
         | | `BufferType <#type-buffertype>`_  []
       - | pks

```


Returns: 
```eval_rst
`SimpleSigner <#type-simplesigner>`_ 
```



#### sign()


signing of any data.
if hashFirst is true the data should be hashed first, otherwise the data is the hash. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `sign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L730>`_ (
      data:`Hex <#type-hex>`_ ,
      account:`Address <#type-address>`_ ,
      sign_type:`SignType <#type-signtype>`_ ,
      payloadType:`SignPayload <#type-signpayload>`_ ,
      meta:``any``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | data
       - | `Hex <#type-hex>`_ 
       - | a Hexcoded String (starting with 0x)
     * - | account
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)
     * - | sign_type
       - | `SignType <#type-signtype>`_ 
       - | the type of signature to create.
         |     - ec_hash : the data needs to be hashed first ( using keccak) before signing
         |     - ec_raw : the data is the ryw value (32bytes) to sign
         |     - ec_prefix : the data is a message which needs to be prefixed with the EthereumSignedMessage and length to before hashing and signing
     * - | payloadType
       - | `SignPayload <#type-signpayload>`_ 
       - | The type of the payload to sign
     * - | meta
       - | ``any``
       - | meta

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### addAccount()


adds a private key to the signer and returns the address associated with it. 

```eval_rst
``string`` `addAccount <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L721>`_ (
      pk:`Hash <#type-hash>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | `Hash <#type-hash>`_ 
       - | a 32 byte Hash encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``string``
```



#### canSign()


returns true if the account is supported (or unlocked) 

```eval_rst
``Promise<boolean>`` `canSign <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L724>`_ (
      address:`Address <#type-address>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | address
       - | `Address <#type-address>`_ 
       - | a 20 byte Address encoded as Hex (starting with 0x)

```


Returns: 
```eval_rst
``Promise<boolean>``
```



#### getAccounts()


returns all addresses managed by the signer. 

```eval_rst
`Address <#type-address>`_  [] `getAccounts <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L719>`_ ()
```

Returns: 
```eval_rst
`Address <#type-address>`_  []
```



### Type TxRequest


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1396)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `args <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1422>`_
       - | ``any`` []
       - | the argument to pass to the method  *(optional)* 
     * - | `confirmations <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1428>`_
       - | ``number``
       - | number of block to wait before confirming  *(optional)* 
     * - | `data <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1404>`_
       - | `Data <#type-data>`_ 
       - | the data to send  *(optional)* 
     * - | `from <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1401>`_
       - | `Address <#type-address>`_ 
       - | address of the account to use  *(optional)* 
     * - | `gas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1407>`_
       - | ``number``
       - | the gas needed  *(optional)* 
     * - | `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1410>`_
       - | ``number``
       - | the gasPrice used  *(optional)* 
     * - | `method <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1419>`_
       - | ``string``
       - | the ABI of the method to be used  *(optional)* 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1413>`_
       - | ``number``
       - | the nonce  *(optional)* 
     * - | `pk <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1425>`_
       - | `Hash <#type-hash>`_ 
       - | raw private key in order to sign  *(optional)* 
     * - | `timeout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1431>`_
       - | ``number``
       - | number of seconds to wait for confirmations before giving up. Default: 10  *(optional)* 
     * - | `to <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1398>`_
       - | `Address <#type-address>`_ 
       - | contract  *(optional)* 
     * - | `value <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1416>`_
       - | `Quantity <#type-quantity>`_ 
       - | the value in wei  *(optional)* 

```


### Type Signature


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L650)


Signature


```eval_rst
  .. list-table::
     :widths: auto

     * - | `message <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L651>`_
       - | `Data <#type-data>`_ 
       - | the message 
     * - | `messageHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L652>`_
       - | `Hash <#type-hash>`_ 
       - | the messageHash 
     * - | `r <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L654>`_
       - | `Hash <#type-hash>`_ 
       - | the r 
     * - | `s <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L655>`_
       - | `Hash <#type-hash>`_ 
       - | the s 
     * - | `signature <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L656>`_
       - | `Data <#type-data>`_ 
       - | the signature  *(optional)* 
     * - | `v <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L653>`_
       - | `Hex <#type-hex>`_ 
       - | the v 

```


### Type BTCBlockHeader


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2040)


a Block header

```eval_rst
  .. list-table::
     :widths: auto

     * - | `bits <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2060>`_
       - | ``string``
       - | bits (target) for the block as hex 
     * - | `chainwork <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2064>`_
       - | ``string``
       - | total amount of work since genesis 
     * - | `confirmations <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2044>`_
       - | ``number``
       - | number of confirmations or blocks mined on top of the containing block 
     * - | `difficulty <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2062>`_
       - | ``number``
       - | difficulty of the block 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2042>`_
       - | ``string``
       - | the hash of the blockheader 
     * - | `height <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2046>`_
       - | ``number``
       - | block number 
     * - | `mediantime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2056>`_
       - | ``string``
       - | unix timestamp in seconds since 1970 
     * - | `merkleroot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2052>`_
       - | ``string``
       - | merkle root of the trie of all transactions in the block 
     * - | `nTx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2066>`_
       - | ``number``
       - | number of transactions in the block 
     * - | `nextblockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2070>`_
       - | ``string``
       - | hash of the next blockheader 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2058>`_
       - | ``number``
       - | nonce-field of the block 
     * - | `previousblockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2068>`_
       - | ``string``
       - | hash of the parent blockheader 
     * - | `time <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2054>`_
       - | ``string``
       - | unix timestamp in seconds since 1970 
     * - | `version <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2048>`_
       - | ``number``
       - | used version 
     * - | `versionHex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2050>`_
       - | ``string``
       - | version as hex 

```


### Type BTCBlock


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2074)


a full Block including the transactions

```eval_rst
  .. list-table::
     :widths: auto

     * - | `bits <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2060>`_
       - | ``string``
       - | bits (target) for the block as hex 
     * - | `chainwork <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2064>`_
       - | ``string``
       - | total amount of work since genesis 
     * - | `confirmations <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2044>`_
       - | ``number``
       - | number of confirmations or blocks mined on top of the containing block 
     * - | `difficulty <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2062>`_
       - | ``number``
       - | difficulty of the block 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2042>`_
       - | ``string``
       - | the hash of the blockheader 
     * - | `height <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2046>`_
       - | ``number``
       - | block number 
     * - | `mediantime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2056>`_
       - | ``string``
       - | unix timestamp in seconds since 1970 
     * - | `merkleroot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2052>`_
       - | ``string``
       - | merkle root of the trie of all transactions in the block 
     * - | `nTx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2066>`_
       - | ``number``
       - | number of transactions in the block 
     * - | `nextblockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2070>`_
       - | ``string``
       - | hash of the next blockheader 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2058>`_
       - | ``number``
       - | nonce-field of the block 
     * - | `previousblockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2068>`_
       - | ``string``
       - | hash of the parent blockheader 
     * - | `time <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2054>`_
       - | ``string``
       - | unix timestamp in seconds since 1970 
     * - | `tx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2076>`_
       - | `T <#type-t>`_  []
       - | the transactions 
     * - | `version <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2048>`_
       - | ``number``
       - | used version 
     * - | `versionHex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2050>`_
       - | ``string``
       - | version as hex 

```


### Type BtcTransaction


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1992)


a BitCoin Transaction.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `blockhash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2021>`_
       - | `Hash <#type-hash>`_ 
       - | the block hash of the block containing this transaction. 
     * - | `blocktime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2030>`_
       - | ``number``
       - | The block time in seconds since epoch (Jan 1 1970 GMT) 
     * - | `confirmations <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2024>`_
       - | ``number``
       - | The confirmations. 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2003>`_
       - | `Hash <#type-hash>`_ 
       - | The transaction hash (differs from txid for witness transactions) 
     * - | `hex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1997>`_
       - | `Data <#type-data>`_ 
       - | the hex representation of raw data 
     * - | `in_active_chain <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1994>`_
       - | ``boolean``
       - | true if this transaction is part of the longest chain 
     * - | `locktime <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2018>`_
       - | ``number``
       - | The locktime 
     * - | `size <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2006>`_
       - | ``number``
       - | The serialized transaction size 
     * - | `time <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2027>`_
       - | ``number``
       - | The transaction time in seconds since epoch (Jan 1 1970 GMT) 
     * - | `txid <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2000>`_
       - | `Hash <#type-hash>`_ 
       - | The requested transaction id. 
     * - | `version <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2015>`_
       - | ``number``
       - | The version 
     * - | `vin <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2033>`_
       - | `BtcTransactionInput <#type-btctransactioninput>`_  []
       - | the transaction inputs 
     * - | `vout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2036>`_
       - | `BtcTransactionOutput <#type-btctransactionoutput>`_  []
       - | the transaction outputs 
     * - | `vsize <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2009>`_
       - | ``number``
       - | The virtual transaction size (differs from size for witness transactions) 
     * - | `weight <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2012>`_
       - | ``number``
       - | The transaction’s weight (between vsize4-3 and vsize4) 

```


### Type Data


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L623)


data encoded as Hex (starting with 0x)
a Hexcoded String (starting with 0x)
 = `string`



### Type BtcTransactionInput


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1935)


a Input of a Bitcoin Transaction

```eval_rst
  .. list-table::
     :widths: auto

     * - | `scriptSig <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1943>`_
       - | 
       - | the script 
     * - | `sequence <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1952>`_
       - | ``number``
       - | The script sequence number 
     * - | `txid <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1937>`_
       - | `Hash <#type-hash>`_ 
       - | the transaction id 
     * - | `txinwitness <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1955>`_
       - | `Data <#type-data>`_  []
       - | hex-encoded witness data (if any) 
     * - | `vout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1940>`_
       - | ``number``
       - | the index of the transactionoutput 

```


### Type BtcTransactionOutput


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1960)


a Input of a Bitcoin Transaction

```eval_rst
  .. list-table::
     :widths: auto

     * - | `n <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1965>`_
       - | ``number``
       - | the index 
     * - | `scriptPubKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1971>`_
       - | 
       - | the script 
     * - | `value <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1962>`_
       - | ``number``
       - | the value in BTC 
     * - | `vout <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1968>`_
       - | ``number``
       - | the index of the transactionoutput 

```


### Type TransactionReceipt


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1241)


a  Receipt of a Transaction containing the events and execution status


```eval_rst
  .. list-table::
     :widths: auto

     * - | `blockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1243>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - hash of the block where this transaction was in. 
     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1245>`_
       - | `BlockType <#type-blocktype>`_ 
       - | block number where this transaction was in. 
     * - | `contractAddress <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1247>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - The contract address created, if the transaction was a contract creation, otherwise null. 
     * - | `cumulativeGasUsed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1249>`_
       - | `Quantity <#type-quantity>`_ 
       - | The total amount of gas used when this transaction was executed in the block. 
     * - | `events <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1269>`_
       - | 
       - | event objects, which are only added in the web3Contract  *(optional)* 
     * - | `from <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1251>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - The address of the sender. 
     * - | `gasUsed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1255>`_
       - | `Quantity <#type-quantity>`_ 
       - | The amount of gas used by this specific transaction alone. 
     * - | `logs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1257>`_
       - | `Log <#type-log>`_  []
       - | Array of log objects, which this transaction generated. 
     * - | `logsBloom <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1259>`_
       - | `Data <#type-data>`_ 
       - | 256 Bytes - A bloom filter of logs/events generated by contracts during transaction execution. Used to efficiently rule out transactions without expected logs. 
     * - | `root <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1261>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - Merkle root of the state trie after the transaction has been executed (optional after Byzantium hard fork EIP609)  *(optional)* 
     * - | `status <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1263>`_
       - | `Quantity <#type-quantity>`_ 
       - | 0x0 indicates transaction failure , 0x1 indicates transaction success. Set for blocks mined after Byzantium hard fork EIP609, null before. 
     * - | `to <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1253>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - The address of the receiver. null when it’s a contract creation transaction. 
     * - | `transactionHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1265>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - hash of the transaction. 
     * - | `transactionIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1267>`_
       - | `Quantity <#type-quantity>`_ 
       - | Integer of the transactions index position in the block. 

```


### Type BlockInfo


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L950)


Block state.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L951>`_
       - | ``number``
       - | the blockNumber 
     * - | `committed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L952>`_
       - | ``boolean``
       - | the committed 
     * - | `verified <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L953>`_
       - | ``boolean``
       - | the verified 

```


### Type AccountAPI


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1457)


The Account API



#### add()


adds a private key to sign with.
This method returns address of the pk 

```eval_rst
``string`` `add <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1464>`_ (
      pk:``string`` | `BufferType <#type-buffertype>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | pk
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | 
         | 

```


Returns: 
```eval_rst
``string``
```



#### addKeyStore()


adds a key from a JSON Keystore file as defined in the Web3 Secret Storage Definition . This method returns address of the pk. 

```eval_rst
`String <#type-string>`_  `addKeyStore <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1478>`_ (
      keystore:``any``,
      passphrase:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | keystore
       - | ``any``
       - | the keystore data
     * - | passphrase
       - | ``string``
       - | the passphrase
         | 

```


Returns: 
```eval_rst
`String <#type-string>`_ 
```



#### decryptKeystore()


decrypts a JSON Keystore file as defined in the Web3 Secret Storage Definition . The result is the raw private key. 

```eval_rst
`BufferType <#type-buffertype>`_  `decryptKeystore <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1471>`_ (
      keystore:``any``,
      passphrase:``string``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | keystore
       - | ``any``
       - | the keystore data
     * - | passphrase
       - | ``string``
       - | the passphrase
         | 

```


Returns: 
```eval_rst
`BufferType <#type-buffertype>`_ 
```



#### ecrecover()


recovers a ecdsa signature. 

```eval_rst
 `ecrecover <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1486>`_ (
      msg:``string`` | `BufferType <#type-buffertype>`_ ,
      sig:``string`` | `BufferType <#type-buffertype>`_ ,
      msgtype:``'eth_sign'`` | ``'raw'`` | ``'hash'``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the message
     * - | sig
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the signature (must be 65bytes in hex or Buffer)
     * - | msgtype
       - | ``'eth_sign'`` | ``'raw'`` | ``'hash'``
       - | the type (raw : the message must be hashed first (default), 'hash' the message is already a 32byte hash, 'eth_sign' - the message will be hashed with EthereumSignedMessage-Prefix)
         | 

```




#### prepareTx()


prepares a Transaction by creating a unsigned raw transaction. 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `prepareTx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1512>`_ (
      tx:`TxRequest <#type-txrequest>`_ )
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | tx
       - | `TxRequest <#type-txrequest>`_ 
       - | the tx parameter
         | 

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



#### signData()


creates a signature with a previously registered signer based on the data 

```eval_rst
`Promise<Signature> <#type-signature>`_  `signData <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1497>`_ (
      msg:``string`` | `BufferType <#type-buffertype>`_ ,
      account:``string`` | `BufferType <#type-buffertype>`_ ,
      msgtype:``'eth_sign'`` | ``'raw'`` | ``'hash'``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the message to sign
     * - | account
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the address of the account ( if null, the first available account is used ). If the account is a 32byte hex or buffer, it it will be used as raw private key.
     * - | msgtype
       - | ``'eth_sign'`` | ``'raw'`` | ``'hash'``
       - | the type (raw : the message must be hashed first (default), 'hash' the message is already a 32byte hash, 'eth_sign' - the message will be hashed with EthereumSignedMessage-Prefix)
         | 

```


Returns: 
```eval_rst
`Promise<Signature> <#type-signature>`_ 
```



#### signRawTx()


creates a signature with a previously registered signer based on the data 

```eval_rst
`Promise<BufferType> <#type-buffertype>`_  `signRawTx <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1506>`_ (
      msg:``string`` | `BufferType <#type-buffertype>`_ ,
      account:``string`` | `BufferType <#type-buffertype>`_ ,
      msgtype:``'eth_sign'`` | ``'raw'`` | ``'hash'``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | msg
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the message to sign
     * - | account
       - | ``string`` | `BufferType <#type-buffertype>`_ 
       - | the address of the account ( if null, the first available account is used ). If the account is a 32byte hex or buffer, it it will be used as raw private key.
     * - | msgtype
       - | ``'eth_sign'`` | ``'raw'`` | ``'hash'``
       - | the type (raw : the message must be hashed first (default), 'hash' the message is already a 32byte hash, 'eth_sign' - the message will be hashed with EthereumSignedMessage-Prefix)
         | 

```


Returns: 
```eval_rst
`Promise<BufferType> <#type-buffertype>`_ 
```



### Type Transaction


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L677)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `chainId <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L693>`_
       - | ``any``
       - | optional chain id  *(optional)* 
     * - | `data <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L689>`_
       - | ``string``
       - | 4 byte hash of the method signature followed by encoded parameters. For details see Ethereum Contract ABI. 
     * - | `from <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L679>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - The address the transaction is send from. 
     * - | `gas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L683>`_
       - | `Quantity <#type-quantity>`_ 
       - | Integer of the gas provided for the transaction execution. eth_call consumes zero gas, but this parameter may be needed by some executions. 
     * - | `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L685>`_
       - | `Quantity <#type-quantity>`_ 
       - | Integer of the gas price used for each paid gas. 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L691>`_
       - | `Quantity <#type-quantity>`_ 
       - | nonce 
     * - | `to <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L681>`_
       - | `Address <#type-address>`_ 
       - | (optional when creating new contract) 20 Bytes - The address the transaction is directed to.  *(optional)* 
     * - | `value <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L687>`_
       - | `Quantity <#type-quantity>`_ 
       - | Integer of the value sent with this transaction. 

```


### Type BlockType


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L603)


BlockNumber or predefined Block
 = `number` | `'latest'` | `'earliest'` | `'pending'`



### Type ABI


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L665)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `anonymous <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L666>`_
       - | ``boolean``
       - | the anonymous  *(optional)* 
     * - | `components <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L670>`_
       - | `ABIField <#type-abifield>`_  []
       - | the components  *(optional)* 
     * - | `constant <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L667>`_
       - | ``boolean``
       - | the constant  *(optional)* 
     * - | `inputs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L671>`_
       - | `ABIField <#type-abifield>`_  []
       - | the inputs  *(optional)* 
     * - | `internalType <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L675>`_
       - | ``string``
       - | the internalType  *(optional)* 
     * - | `name <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L673>`_
       - | ``string``
       - | the name  *(optional)* 
     * - | `outputs <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L672>`_
       - | `ABIField <#type-abifield>`_  [] | ``any`` []
       - | the outputs  *(optional)* 
     * - | `payable <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L668>`_
       - | ``boolean``
       - | the payable  *(optional)* 
     * - | `stateMutability <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L669>`_
       - | ``'pure'`` 
         | | ``'view'`` 
         | | ``'nonpayable'`` 
         | | ``'payable'`` 
         | | ``string``
       - | the stateMutability  *(optional)* 
     * - | `type <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L674>`_
       - | ``'function'`` 
         | | ``'constructor'`` 
         | | ``'event'`` 
         | | ``'fallback'`` 
         | | ``string``
       - | the type 

```


### Type Log


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1362)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1376>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - address from which this log originated. 
     * - | `blockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1372>`_
       - | `Hash <#type-hash>`_ 
       - | Hash, 32 Bytes - hash of the block where this log was in. null when its pending. null when its pending log. 
     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1374>`_
       - | `Quantity <#type-quantity>`_ 
       - | the block number where this log was in. null when its pending. null when its pending log. 
     * - | `data <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1378>`_
       - | `Data <#type-data>`_ 
       - | contains the non-indexed arguments of the log. 
     * - | `logIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1366>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the log index position in the block. null when its pending log. 
     * - | `removed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1364>`_
       - | ``boolean``
       - | true when the log was removed, due to a chain reorganization. false if its a valid log. 
     * - | `topics <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1380>`_
       - | `Data <#type-data>`_  []
       - | - Array of 0 to 4 32 Bytes DATA of indexed log arguments. (In solidity: The first topic is the hash of the signature of the event (e.g. Deposit(address,bytes32,uint256)), except you declared the event with the anonymous specifier.) 
     * - | `transactionHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1370>`_
       - | `Hash <#type-hash>`_ 
       - | Hash, 32 Bytes - hash of the transactions this log was created from. null when its pending log. 
     * - | `transactionIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1368>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the transactions index position log was created from. null when its pending log. 

```


### Type Block


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1318)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `author <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1338>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - the address of the author of the block (the beneficiary to whom the mining rewards were given) 
     * - | `difficulty <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1342>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the difficulty for this block 
     * - | `extraData <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1346>`_
       - | `Data <#type-data>`_ 
       - | the ‘extra data’ field of this block 
     * - | `gasLimit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1350>`_
       - | `Quantity <#type-quantity>`_ 
       - | the maximum gas allowed in this block 
     * - | `gasUsed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1352>`_
       - | `Quantity <#type-quantity>`_ 
       - | the total used gas by all transactions in this block 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1322>`_
       - | `Hash <#type-hash>`_ 
       - | hash of the block. null when its pending block 
     * - | `logsBloom <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1330>`_
       - | `Data <#type-data>`_ 
       - | 256 Bytes - the bloom filter for the logs of the block. null when its pending block 
     * - | `miner <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1340>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - alias of ‘author’ 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1326>`_
       - | `Data <#type-data>`_ 
       - | 8 bytes hash of the generated proof-of-work. null when its pending block. Missing in case of PoA. 
     * - | `number <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1320>`_
       - | `Quantity <#type-quantity>`_ 
       - | The block number. null when its pending block 
     * - | `parentHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1324>`_
       - | `Hash <#type-hash>`_ 
       - | hash of the parent block 
     * - | `receiptsRoot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1336>`_
       - | `Data <#type-data>`_ 
       - | 32 Bytes - the root of the receipts trie of the block 
     * - | `sealFields <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1360>`_
       - | `Data <#type-data>`_  []
       - | PoA-Fields 
     * - | `sha3Uncles <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1328>`_
       - | `Data <#type-data>`_ 
       - | SHA3 of the uncles data in the block 
     * - | `size <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1348>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer the size of this block in bytes 
     * - | `stateRoot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1334>`_
       - | `Data <#type-data>`_ 
       - | 32 Bytes - the root of the final state trie of the block 
     * - | `timestamp <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1354>`_
       - | `Quantity <#type-quantity>`_ 
       - | the unix timestamp for when the block was collated 
     * - | `totalDifficulty <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1344>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the total difficulty of the chain until this block 
     * - | `transactions <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1356>`_
       - | `Hash <#type-hash>`_  
         | | `Transaction <#type-transaction>`_  []
       - | Array of transaction objects, or 32 Bytes transaction hashes depending on the last given parameter 
     * - | `transactionsRoot <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1332>`_
       - | `Data <#type-data>`_ 
       - | 32 Bytes - the root of the transaction trie of the block 
     * - | `uncles <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1358>`_
       - | `Hash <#type-hash>`_  []
       - | Array of uncle hashes 

```


### Type Quantity


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L611)


a BigInteger encoded as hex.
 = `number` | [Hex](#type-hex) 



### Type LogFilter


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1383)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1389>`_
       - | `Address <#type-address>`_ 
       - | (optional) 20 Bytes - Contract address or a list of addresses from which logs should originate. 
     * - | `fromBlock <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1385>`_
       - | `BlockType <#type-blocktype>`_ 
       - | Quantity or Tag - (optional) (default: latest) Integer block number, or 'latest' for the last mined block or 'pending', 'earliest' for not yet mined transactions.  *(optional)* 
     * - | `limit <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1393>`_
       - | `Quantity <#type-quantity>`_ 
       - | å(optional) The maximum number of entries to retrieve (latest first).  *(optional)* 
     * - | `toBlock <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1387>`_
       - | `BlockType <#type-blocktype>`_ 
       - | Quantity or Tag - (optional) (default: latest) Integer block number, or 'latest' for the last mined block or 'pending', 'earliest' for not yet mined transactions.  *(optional)* 
     * - | `topics <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1391>`_
       - | ``string`` | ``string`` [] []
       - | (optional) Array of 32 Bytes Data topics. Topics are order-dependent. It’s possible to pass in null to match any topic, or a subarray of multiple topics of which one should be matching.  *(optional)* 

```


### Type TransactionDetail


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1275)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `blockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1281>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - hash of the block where this transaction was in. null when its pending. 
     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1283>`_
       - | `BlockType <#type-blocktype>`_ 
       - | block number where this transaction was in. null when its pending. 
     * - | `chainId <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1309>`_
       - | `Quantity <#type-quantity>`_ 
       - | the chain id of the transaction, if any. 
     * - | `condition <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1313>`_
       - | ``any``
       - | (optional) conditional submission, Block number in block or timestamp in time or null. (parity-feature)  *(optional)* 
     * - | `creates <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1311>`_
       - | `Address <#type-address>`_ 
       - | creates contract address 
     * - | `from <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1287>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - address of the sender. 
     * - | `gas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1295>`_
       - | `Quantity <#type-quantity>`_ 
       - | gas provided by the sender. 
     * - | `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1293>`_
       - | `Quantity <#type-quantity>`_ 
       - | gas price provided by the sender in Wei. 
     * - | `hash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1277>`_
       - | `Hash <#type-hash>`_ 
       - | 32 Bytes - hash of the transaction. 
     * - | `input <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1297>`_
       - | `Data <#type-data>`_ 
       - | the data send along with the transaction. 
     * - | `nonce <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1279>`_
       - | `Quantity <#type-quantity>`_ 
       - | the number of transactions made by the sender prior to this one. 
     * - | `pk <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1315>`_
       - | ``any``
       - | optional: the private key to use for signing  *(optional)* 
     * - | `publicKey <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1307>`_
       - | `Hash <#type-hash>`_ 
       - | public key of the signer. 
     * - | `r <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1303>`_
       - | `Quantity <#type-quantity>`_ 
       - | the R field of the signature. 
     * - | `raw <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1305>`_
       - | `Data <#type-data>`_ 
       - | raw transaction data 
     * - | `standardV <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1301>`_
       - | `Quantity <#type-quantity>`_ 
       - | the standardised V field of the signature (0 or 1). 
     * - | `to <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1289>`_
       - | `Address <#type-address>`_ 
       - | 20 Bytes - address of the receiver. null when its a contract creation transaction. 
     * - | `transactionIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1285>`_
       - | `Quantity <#type-quantity>`_ 
       - | integer of the transactions index position in the block. null when its pending. 
     * - | `v <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1299>`_
       - | `Quantity <#type-quantity>`_ 
       - | the standardised V field of the signature. 
     * - | `value <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1291>`_
       - | `Quantity <#type-quantity>`_ 
       - | value transferred in Wei. 

```


### Type Web3Contract


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1541)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `events <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1562>`_
       - | 
       - | the events 
     * - | `methods <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1556>`_
       - | 
       - | the methods 
     * - | `options <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1542>`_
       - | 
       - | the options 

```


#### deploy()


deploy 

```eval_rst
`Web3TransactionObject <#type-web3transactionobject>`_  `deploy <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1552>`_ (
      args:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | args
       - | 
       - | args

```


Returns: 
```eval_rst
`Web3TransactionObject <#type-web3transactionobject>`_ 
```



#### once()


once 

```eval_rst
``void`` `once <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1560>`_ (
      eventName:``string``,
      options:,
      handler:(`Error <#type-error>`_  , `Web3Event <#type-web3event>`_ ) => ``void``)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | eventName
       - | ``string``
       - | event name
     * - | options
       - | 
       - | options
     * - | handler
       - | (`Error <#type-error>`_  , `Web3Event <#type-web3event>`_ ) => ``void``
       - | handler

```




#### getPastEvents()


get past events 

```eval_rst
``Promise<>`` `getPastEvents <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1575>`_ (
      evName:``string``,
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | evName
       - | ``string``
       - | ev name
     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
``Promise<>``
```



### Type TxType


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L977)


Defines the type of a transaction.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `type <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L978>`_
       - | ``'Withdraw'`` 
         | | ``'Transfer'`` 
         | | ``'TransferToNew'``
       - | the type 

```


### Type btc_config


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2112)


bitcoin configuration.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `maxDAP <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2116>`_
       - | ``number``
       - | max number of DAPs (Difficulty Adjustment Periods) allowed when accepting new targets.  *(optional)* 
     * - | `maxDiff <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L2121>`_
       - | ``number``
       - | max increase (in percent) of the difference between targets when accepting new targets.  *(optional)* 

```


### Type zksync_config


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1145)


zksync configuration.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `account <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1154>`_
       - | ``string``
       - | the account to be used. if not specified, the first signer will be used.  *(optional)* 
     * - | `create2 <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1170>`_
       - | 
       - | create2 arguments  *(optional)* 
     * - | `musig_pub_keys <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1195>`_
       - | ``string``
       - | concated packed public keys of the signers of the multisig  *(optional)* 
     * - | `musig_urls <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1190>`_
       - | ``string`` []
       - | if used as a musig-signature the  *(optional)* 
     * - | `provider_url <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1149>`_
       - | ``string``
       - | url of the zksync-server  *(optional)* 
     * - | `signer_type <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1159>`_
       - | ``'pk'`` | ``'contract'`` | ``'create2'``
       - | defines the type of the signer. Must be one of those 3 values. (default: pk)  *(optional)* 
     * - | `sync_key <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1165>`_
       - | ``string``
       - | optionaly the private seephrase to use when signing sync-transaction.
         | If ommited this key is derrived from the signer.  *(optional)* 

```


### Type Web3TransactionObject


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1517)






#### call()


call 

```eval_rst
``Promise<any>`` `call <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1518>`_ (
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
``Promise<any>``
```



#### encodeABI()


a Hexcoded String (starting with 0x) 

```eval_rst
`Hex <#type-hex>`_  `encodeABI <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1538>`_ ()
```

Returns: 
```eval_rst
`Hex <#type-hex>`_ 
```



#### estimateGas()


estimate gas 

```eval_rst
``Promise<number>`` `estimateGas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1532>`_ (
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
``Promise<number>``
```



#### send()


send 

```eval_rst
``Promise<any>`` `send <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1524>`_ (
      options:)
```

Parameters: 
```eval_rst
  .. list-table::
     :widths: auto

     * - | options
       - | 
       - | options

```


Returns: 
```eval_rst
``Promise<any>``
```



### Type Web3Event


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1434)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1443>`_
       - | `Address <#type-address>`_ 
       - | the address 
     * - | `blockHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1445>`_
       - | `Hash <#type-hash>`_ 
       - | the blockHash 
     * - | `blockNumber <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1444>`_
       - | ``number``
       - | the blockNumber 
     * - | `event <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1438>`_
       - | ``string``
       - | the event 
     * - | `logIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1440>`_
       - | ``number``
       - | the logIndex 
     * - | `raw <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1446>`_
       - | 
       - | the raw 
     * - | `returnValues <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1435>`_
       - | 
       - | the returnValues 
     * - | `signature <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1439>`_
       - | ``string``
       - | the signature 
     * - | `transactionHash <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1442>`_
       - | `Hash <#type-hash>`_ 
       - | the transactionHash 
     * - | `transactionIndex <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1441>`_
       - | ``number``
       - | the transactionIndex 

```


### Type DepositResponse


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1010)


Response of a Deposit-Transaction.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `receipt <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L1011>`_
       - | `TransactionReceipt <#type-transactionreceipt>`_ 
       - | the receipt 

```


### Type ZKAccountInfo


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L924)


return structure after fetching the current account info.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `address <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L925>`_
       - | ``string``
       - | the address 
     * - | `committed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L926>`_
       - | 
       - | the committed 
     * - | `depositing <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L933>`_
       - | 
       - | the depositing 
     * - | `id <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L938>`_
       - | ``number``
       - | the id 
     * - | `verified <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L939>`_
       - | 
       - | the verified 

```


### Type ETHOpInfoResp


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L969)


L1 Operation State

```eval_rst
  .. list-table::
     :widths: auto

     * - | `block <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L971>`_
       - | `BlockInfo <#type-blockinfo>`_ 
       - | the block 
     * - | `executed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L970>`_
       - | ``boolean``
       - | the executed 

```


### Type Tokens


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L996)


Token List.



### Type Fee


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L984)


fee collection.

```eval_rst
  .. list-table::
     :widths: auto

     * - | `feeType <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L985>`_
       - | `TxType <#type-txtype>`_ 
       - | the feeType 
     * - | `gasFee <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L988>`_
       - | ``number``
       - | the gasFee 
     * - | `gasPrice <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L987>`_
       - | ``number``
       - | the gasPrice 
     * - | `totalFee <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L990>`_
       - | ``number``
       - | the totalFee 
     * - | `totalGas <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L986>`_
       - | ``number``
       - | the totalGas 
     * - | `zkpFee <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L989>`_
       - | ``number``
       - | the zkpFee 

```


### Type TxInfo


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L959)


Transaction state

```eval_rst
  .. list-table::
     :widths: auto

     * - | `block <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L960>`_
       - | `BlockInfo <#type-blockinfo>`_ 
       - | the block 
     * - | `executed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L961>`_
       - | ``boolean``
       - | the executed 
     * - | `failReason <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L962>`_
       - | ``string``
       - | the failReason 
     * - | `success <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L963>`_
       - | ``boolean``
       - | the success 

```


### Type ABIField


Source: [index.d.ts](https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L659)




```eval_rst
  .. list-table::
     :widths: auto

     * - | `indexed <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L661>`_
       - | ``boolean``
       - | the indexed  *(optional)* 
     * - | `internalType <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L660>`_
       - | ``string``
       - | the internalType  *(optional)* 
     * - | `name <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L662>`_
       - | ``string``
       - | the name 
     * - | `type <https://github.com/slockit/in3-c/blob/master/wasm/src/index.d.ts#L663>`_
       - | ``string``
       - | the type 

```


