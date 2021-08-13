# smart-NFT

This is a full explanation of how to interact with the smart contract NFTs currently used on Dartroom. This guide can be used to deploy your own or interact with the ones you bought on [Dartroom](https://dartroom.xyz).

I highly encourage anyone using this guide to create their smart contract-based NFT to not copy this example directly. This type of NFT is not a replacement for the standard ASA (Algorand Standard Asset). It is meant as a highly programmable NFT to cover more specific use cases that might not be well suited to ASA. Add extra transfer logic or metadata references to cover your specific use case.

In this guide, Atomic transfers are mentioned multiple times. These are group transactions where the entire group fails if one transaction fails. If you want to learn more about atomic transfers, you can read the [official documentation](https://developer.algorand.org/docs/features/atomic_transfers/).

## Table of Contents

- [Setup and requirements](#setup-and-requirements)
- [Deploying](#deploying-the-smart-contracts)
- [Burning](#burning-the-stateful-contract)
- [Start and stop auctions](#start-and-stop-auctions)
- [Bidding](#bidding)
- [Ending auctions](#ending-auctions)

# Setup and requirements

To deploy and interact with the contracts, you will need access to an Algorand node and indexer. For testnet, you can use one of the free developer API's from either [Algo Explorer](https://algoexplorer.io/api-dev/v2) or [Pure Stake](https://developer.purestake.io/). For mainnet, I would recommend you to [set up a node and indexer](https://developer.algorand.org/docs/run-a-node/setup/types/) yourself. To support the Algorand network and gain a significantly faster response time.

```js
const algosdk = require('algosdk');

const indexerServer = "https://testnet-algorand.api.purestake.io/idx2";
const indexerPort = "";
const indexerToken = {'X-API-key' : 'your api key',}

let indexerClient = new algosdk.Indexer(indexerToken, indexerServer, indexerPort);

const nodeServer = "https://testnet-algorand.api.purestake.io/ps2";
const nodePort = '';
const nodeToken =  {'X-API-key' : 'your api key',};

let algodClient = new algosdk.Algodv2(nodeToken, nodeServer, nodePort);
```

To be able to deploy the contracts to the network, they first need to be compiled. To properly compile the contracts, a helper function is needed.

```js
async function compileProgram(client, programSource) {
  let encoder = new TextEncoder();
  let programBytes = encoder.encode(programSource);
  let compileResponse = await client.compile(programBytes).do().catch((err) => {console.log(err)});
  let compiledBytes = new Uint8Array(Buffer.from(compileResponse.result, "base64"));
  return compiledBytes;
}
```

This example will use another helper function to wait for the transactions to be committed to the network. Some transactions rely on another transaction being committed to the network before it can be sent. With this function, we can await that moment.

```js
const waitForConfirmation = async function (algodClient, txId, timeout) {
  if (algodClient == null || txId == null || timeout < 0) {
    throw new Error("Bad arguments");
  }

  const status = (await algodClient.status().do());
  if (status === undefined) {
    throw new Error("Unable to get node status");
  }

  const startround = status["last-round"] + 1;
  let currentround = startround;

  while (currentround < (startround + timeout)) {
    const pendingInfo = await algodClient.pendingTransactionInformation(txId).do();
    if (pendingInfo !== undefined) {
      if (pendingInfo["confirmed-round"] !== null && pendingInfo["confirmed-round"] > 0) {
        //Got the completed Transaction
        return pendingInfo;
      } else {
        if (pendingInfo["pool-error"] != null && pendingInfo["pool-error"].length > 0) {
            // If there was a pool error, then the transaction has been rejected!
            throw new Error("Transaction " + txId + " rejected - pool error: " + pendingInfo["pool-error"]);
        }
      }
    }
    await algodClient.statusAfterBlock(currentround).do();
    currentround++;
  }

  throw new Error("Transaction " + txId + " not confirmed after " + timeout + " rounds!");
};
```

You can sign the transactions with the passphrase directly, but that is an unrealistic scenario in deployment. The AlgoSigner will be used in this example to sign transactions, although there are other options like [MyAlgoConnect](https://github.com/randlabs/myalgo-connect).
The only thing you need to use the AlgoSigner is the [chrome extension](https://chrome.google.com/webstore/detail/algosigner/kmmolakhbgdlpkjkcjkebenjheonagdm). When the extension is installed, the AlgoSigner can be directly called from the frontend code.

```js
await AlgoSigner.connect()
```


# Deploying the smart contracts

Deploying the contracts involves 4 steps:
- [Deploy stateful contract](#deploy-stateful-contract)
- [Deploy the stateless escrow contract](#deploy-the-stateless-escrow-contract)
- [Fund the escrow](#fund-the-escrow)
- [Update the stateful contract](#update-the-stateful-contract)

## Deploy stateful contract

First, we need to check if the Algo Signer is installed and that we have access to it. By calling AlgoSigner.connect() we check if the browser extension is installed and ask the user to give us access. An extra browser window will pop up from the Algo Signer asking the user to fill in their password. Afterwards, we gain access to a list of addresses in their wallet.

```js
try {
  await AlgoSigner.connect()
} catch (err) {
  const error = new Error(
    'Failed to connect with AlgoSigner. Have you installed the browser extension?'
  )
  throw error
}
```

You can get either the addresses from the 'TestNet' wallet or the 'MainNet' wallet. Addresses that are created on the mainnet also exist on the testnet and the other way around. Only in the AlgoSigner, you will need to add them separately. Addresses added to the testnet wallet don't show up on the mainnet wallet.

```js
let addresses = await AlgoSigner.accounts({ ledger: 'TestNet' }) 

// response
[
  {
    "address": "U2VHSZL3LNGATL3IBCXFCPBTYSXYZBW2J4OGMPLTA4NA2CB4PR7AW7C77E"
  },
  {
    "address": "MTHFSNXBMBD4U46Z2HAYAOLGD2EV6GQBPXVTL727RR3G44AJ3WVFMZGSBE"
  }
]
```

Now that we have access to the Algo Signer we can set the transaction parameters. The stateful contract we are deploying here stores 11 values in the global state. 4 integers and 7 strings (which includes addresses). The minimum balance of the contract creator is increased based on the number of stored values. Every value increases the minimum balance with 0.05 Algo, in addition to 0.1 for just the contract. The number of values you want to store can not be changed once the contract is deployed. This contract does not use the local state of users, therefore localInts and localBytes are not used.

```js
const localInts = 0
const localBytes = 0
const globalInts = 4
const globalBytes = 7
```

Every transaction on Algorand requires the most recent blockchain status. You can get this by calling the getTransactionParams method, which will get the information from the connected node.

```js
let params = await algodClient.getTransactionParams().do()

// response
{
  "flatFee": false,
  "fee": 0,
  "firstRound": 15956724,
  "lastRound": 15957724,
  "genesisID": "testnet-v1.0",
  "genesisHash": "SGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiI="
}
```

The transaction fee on Algorand is 1000 μAlgo at the moment. All transactions should default to this amount, but it is standard practice to set this fee yourself. By setting the flatFee property to true we enforce this fee, if the fee is not high enough for the network the transaction will fail instead of getting its fee adjusted.

```js
params.fee = 1000
params.flatFee = true
```

The transaction we are calling is not the standard 'pay' type but an application call of type 'appl'. When calling an application you will also have to specify what part of the code of the stateful contract needs to be executed. In this case, a NoOpOC. This will simply execute the approval program. You can find the full list in the [SDK documentation](https://algorand.github.io/js-algorand-sdk/enums/onapplicationcomplete.html).

```js
const type = 'appl'
const onComplete = algosdk.OnApplicationComplete.NoOpOC
```

Now only the approval program and clear state program need to be set before we can sign and send the transaction. The approval program contains the TEAL that would execute when you want to use the stateful contract. The clear state program is only used as a backup method to opt a user out of the contract. This can be done with a 'soft' opt-out within the approval program, or if that fails a 'hard' opt-out with the clear state program. In this example, we deploy a contract that does not use opt-in/opt-out functionality.

The TEAL code can be set in template literals and its values dynamically filled in just like you would assemble an email in javascript. In this case, we need to fill in the title, artist, created at and manager properties. Here is part of the approval program that requires dynamic values, although you will need to add the entire contract.

```js
let approvalProgramSource = `
...

// nft creation
  nft_creation:
  byte "Title"
  byte "${}"
  app_global_put

  byte "Artist"
  byte "${}"
  app_global_put

  byte "Created at"
  byte "${}"
  app_global_put

  byte "Owner"
  global CreatorAddress
  app_global_put

  byte "Manager" // set manager address
  addr ${}
  app_global_put

  byte "Escrow"
  global CreatorAddress
  app_global_put

  byte "Highest bid"
  int 0
  app_global_put

  byte "for sale"
  int 0
  app_global_put

  byte "Duration"
  int 0
  app_global_put

  byte "End round"
  int 0
  app_global_put

  byte "Highest bidder"
  global CreatorAddress
  app_global_put

  int 1
  return
//

...
`
```

Users never opt-in, so we can set the clear state program to just 'int 1'. This will make a clear state app cal always suceed.

```js
let clearProgramSource = `#pragma version 4
int 1
`
```

We need to use the earlier mentioned compile program function to compile the two programs.

```js
const approvalProgram = compileProgram(algodClient, approvalProgramSource)
const clearProgram = compileProgram(algodClient, clearProgramSource)
```

The only thing left to set is the sender address. If the asset is associated with a person or company it's important who signs it. In the case of digital artwork, the NFT is only valuable if the artist signed it. The sender address you use to deploy the contract is set as the creator. This address should be the known address of the artist.

You can get the address from the Algo Signer address list.

```js
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address
```

Now we can assemble the transaction for signing with the Algo Signer.

```js
const signedTxn = await AlgoSigner.sign({
  type: type,
  from: sender,
  ...params,
  note: null,
  appApprovalProgram: approvalProgram.result,
  appClearProgram: clearProgram.result,
  appOnComplete: onComplete,
  appLocalInts: localInts,
  appLocalByteSlices: localBytes,
  appGlobalInts: globalInts,
  appGlobalByteSlices: globalBytes,
})
```

The response you get back from the Algo Signer is an object with two properties 'blob' and 'txID'. The blob contains the signed transaction in base64 which we first need to buffer to bytes before we can send it. The txID can be used to track the transactio with the waitForConfirmation function.

The waitForConfirmation function will wait until the transaction is confirmed before returning a response. This way we can let the user wait and give them a response when the transaction is sent. This is particularly usefull in this case since the user needs to wait for the stateful contract to be deployed before deploying the stateless contract.

```js
const compiledBytes = Buffer.from(signedTxn.blob, "base64")
await algodClient.sendRawTransaction(compiledBytes).do().catch((err) => {console.log(err)})
const txInfo = await waitForConfirmation(algodClient, signedTxn.txID, 4)
```

We need the appId from the just deployed contract for the stateless escrow. We can extract it from the transaction info.

```js
const appId = txInfo['application-index']
```

## Deploy the stateless escrow contract

Deploying the escrow contract does not require signing by a user. Stateless contracts are not deployed the same way as stateful contracts. Instead of placing the program on the chain as you do with stateful, you send the program with every transaction with stateless contracts. Therefore they can't store data, which makes them stateless. This means that we will need to save our program or reconstruct it when we want to interact with the contract again.

Instead of signing with a user, you sign transactions from the stateless contracts with the program. The logic in the program decides if the transaction is valid. 'Deploying' the escrow is as easy a just compiling it. The response will contain a 'hash' which is the stateless contracts address and a 'result' which is a base64 encoded program with which you sign the transactions. 

The example code below only shows a part of the escrow program, but you should again fill in the entire program. Here you will need to fill in the application id we got previously.

```js
const escrowProgramSource =`#pragma version 3
gtxn 0 TypeEnum
int appl
==

gtxn 0 ApplicationID
int ${appId}
==
&&

global GroupSize
int 2
==
&&

bnz check_2_fees

...
`

const escrow = await algodClient.compile(escrowProgramSource).do()

const escrowAddress = escrow.hash
const escrowProgram = escrow.result
```

Because we have a stateful contract combined with a stateless, we can offload a lot of transaction checks to the stateful contract. We primarily check if the first transaction in the atomic group is sent to the particular appId. If the check in the app fails then the escrow transaction fails as well, because of this functionality we don't have to double-check.

## Fund the escrow

Now that the escrow is deployed we need to fund it. Since it's just a normal Algorand address only with a different signing method, it needs a minimum balance to function. Every address on Algorand needs 0.1 algo to be able to send transactions.(All algorand token amounts need to be converted to μAlgo)

We do this with a normal transaction. We first need to get the latest blockchain info and set the fee again. This time it's a normal 'pay' transaction with the receiver set to the previously received hash of the stateless contract, which is just an address. The sender can be anyone, but we will need the address to be added to the AlgoSigner again. 

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000
params.flatFee = true

const type = 'pay'
const reviever = escrowAddress
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address
const amount = 100000

const signedTxn = await AlgoSigner.sign({
  type: type,
  to: receiver,
  from: sender,
  amount: amount,
  ...params,
  note: null,
});

const compiledBytes = Buffer.from(signedTxn.blob, "base64")
await algodClient.sendRawTransaction(compiledBytes).do().catch((err) => {console.log(err)})
const txInfo = await waitForConfirmation(algodClient, signedTxn.txID, 4)
```

## Update the stateful contract

We now need to add the escrow address to the stateful contract. This transaction needs to be a group transaction, also called atomic transfer. The build-up is different than with a single transaction.

First, the transactions need to be created before we send them to the Algo Signer. The transaction to the stateful contract is made with a NoOpTxn, which will run the approval program. The transaction to the escrow is a normal payment transaction, only we don't send any funds to it.

Application/stateful contract calls can have arguments passed in, to for example to store in the global state. We don't use that functionality here, therefore it is set to undefined. Instead of sending a transaction to an address, app calls are sent to a certain contract defined with the index.

In the payment transaction, the two undefined parameters are in order 'closeRemainderTo' and 'note'. Both of which we don't use. 

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000
params.flatFee = true

const index = appId
const appArgs = undefined
const reviever = escrowAddress
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address

let txn1 = algosdk.makeApplicationNoOpTxn(sender, params, index, appArgs)
let txn2 = algosdk.makePaymentTxnWithSuggestedParams(sender, reviever, 0, undefined, undefined, params)

```

Because it is a group transaction we need to assign a group id to both transactions.

```js
const txns = [txn1, txn2]
const txGroup = algosdk.assignGroupID(txns)
```

To be able to send the transactions to the singTxn method of the Algo Signer we need to encode the transactions. First to byte array and then to message pack. The algoSigner then gives us signed transactions back in base64 format. We then send a array of transactions with the same function as before. The wait for confirmation function only needs one txID. Since it's an atomic group if one transaction fails all do and the other way around.

```js
let binaryTxs = [txn1.toByte(), txn2.toByte()]
let base64Txs = binaryTxs.map((binary) => AlgoSigner.encoding.msgpackToBase64(binary))

let signedTxs = await AlgoSigner.signTxn([
  {
    txn: base64Txs[0],
  },
  {
    txn: base64Txs[1],
  },
])

let binarySignedTxs = signedTxs.map((tx) => Buffer.from(tx.blob, "base64"));

await algodClient.sendRawTransaction(binarySignedTxs).do().catch((err) => {console.log(err)});
await waitForConfirmation(algodClient, signedTxs[0].txID, 4)
```

The contract is now fully operational and ready for use. You can view the app and escrow on block explorers like the [Algo Explorer](https://testnet.algoexplorer.io/), or query the indexer for info. Note that all none integer values are base64 encoded and that addresses are base64 encoded raw addresses. The address first needs to be encoded to get the 'human readable' version. Here is an example of extracting the owner address.

```js
const app = await indexerClient.lookupApplications(appId).do().catch((err) => {console.log(err)})

const globalState = app.application.params['global-state']
const ownerBase64 = globalState.find(x => x.key === 'T3duZXI=').value.bytes
const owner = algosdk.encodeAddress(new Uint8Array(Buffer.from(ownerBase64, "base64")))
```

# Burning the stateful contract

There is a 10 stateful contract limit for Algorand accounts at the moment. This means that every address is only allowed to opt into or create 10 stateful contracts. If you were opted-in to a stateful contract, you can simply opt-out to free up a slot. If you deployed the contract you will need to destroy it to clear a slot. 
You can program when and who is allowed to destroy the contract. In this contract, only the creator is allowed to destroy it and only when the creator is also the owner. 

### Delete logic in the approval program
```js
handle_deleteapp:

global CreatorAddress
txn Sender
==

byte "Owner"
app_global_get
txn Sender
==
&&

return
```

When you destroy an app its global state is deleted and you can no longer interact with it. The global state can still be found in the creation and update transactions. The info is not completely gone.

For simplicity, we will keep using the group transaction layout even if there is only one transaction. There is a dedicated delete app transaction function.

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000
params.flatFee = true

const index = appId
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address

let txn = algosdk.makeApplicationDeleteTxn(sender, params, index)
let base64Txs = AlgoSigner.encoding.msgpackToBase64(txn.toByte());

let signedTxs = await AlgoSigner.signTxn([
  {
    txn: base64Txs[0]
  }
])

let binarySignedTxs = signedTxs.map((tx) => Buffer.from(tx.blob, "base64"));

await algodClient.sendRawTransaction(binarySignedTxs).do().catch((err) => {console.log(err)});
await waitForConfirmation(algodClient, signedTxs[0].txID, 4)
```

# Start and stop auctions

Starting and stopping auctions can only be done in specific scenarios. We make use of the standard NoOpTxn function to execute the approval program. You could pass in an argument that signals that it's a start auction call, but in this contract it is not necessary. 

## Starting an auction

Starting an auction is only possible when the contract is not in the auction. The only other possible action at that state is destroying the contract. which is not a problem, since it makes use of the delete app transaction and not the NoOpTxn. 

Two arguments need to be included in the transaction. The reserve price, which is the minimum bid someone needs to place. And the duration, which is the amount of rounds that the auction takes. Both need to be converted to a supported format.

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000
params.flatFee = true
const index = appId

let appArgs = [];
const big0 = new Int64BE(reservePrice * 1000000) // convert to μAlgo with * 1000000
const big1 = new Int64BE(durationRounds)

appArgs.push(new Uint8Array(big0.toBuffer()))
appArgs.push(new Uint8Array(big1.toBuffer()))

const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address

let txn = algosdk.makeApplicationNoOpTxn(sender, params, index, appArgs)
let base64Txs =AlgoSigner.encoding.msgpackToBase64(txn.toByte())

const signedTxn = await AlgoSigner.signTxn([
  {
    txn: base64Txs[0]
  }
])

let binarySignedTxs = signedTxs.map((tx) => Buffer.from(tx.blob, "base64"));

await algodClient.sendRawTransaction(binarySignedTxs).do().catch((err) => {console.log(err)});
await waitForConfirmation(algodClient, signedTxs[0].txID, 4)
```
There are three limitations build into the contract that restricts the values of the reserve price and duration. 

 - The reserve price needs to be at least 1 algo
 - The reserve price needs to be dividable by 100
 - The duration needs to be at least 800 rounds

This check can be easily removed if undesired, but note that the dividable by 100 is also checked when someone bids. If you remove the check here, it should also be removed in the bid check. But this creates the possibility to set an unsplittable price and brick the contract.

### Start auction checks
```js
// start auction
  start_auction:
  txn TypeEnum
  int appl
  ==

  txn NumAppArgs
  int 2
  ==
  &&

  txn Sender
  byte "Owner"
  app_global_get
  ==
  &&

  txn ApplicationArgs 0
  btoi
  int 100
  %                     // modulus: the remainder of price / 100
  int 0                 // needs to be equal to 0 to make sure the royalty split works
  ==
  &&

  txn ApplicationArgs 0
  btoi                 
  int 1000000                 
  >=                   // minimum price of 1 algo
  &&

  int 800
  txn ApplicationArgs 1
  btoi
  <=                  // minimum auction duration of 800 rounds (~ 1 hour)
  &&

  bnz set_auction
  int 0
  return
//
```

## Stopping an auction

Stopping an auction is only possible if the owner sends a NoOpTxn and no one has bid yet.

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000
params.flatFee = true
const index = appId
let appArgs = undefined
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address

let txn = algosdk.makeApplicationNoOpTxn(sender, params, index, appArgs)
let base64Txs =AlgoSigner.encoding.msgpackToBase64(txn.toByte())

const signedTxn = await AlgoSigner.signTxn([
  {
    txn: base64Txs[0]
  }
])

let binarySignedTxs = signedTxs.map((tx) => Buffer.from(tx.blob, "base64"));

await algodClient.sendRawTransaction(binarySignedTxs).do().catch((err) => {console.log(err)});
await waitForConfirmation(algodClient, signedTxs[0].txID, 4)
```

By default, the highest bidder property is set to the owner address. To check if no one has bid yet we can simply check if the owner address is equal to the highest bidder.

### Check highest bidder
```js
byte "for sale"
app_global_get
int 1
==

byte "Owner"
app_global_get
byte "Highest bidder"
app_global_get
==
&&

byte "Owner"
app_global_get
txn Sender
==
&&
bnz reset_auction
```

# Bidding

When bidding there are two options, either someone is the first bidder or someone is overbidding. We can check this again with the highest bidder property, if it is still equal to the owner address it's the first bid. We distinguish this action from the stop auction by checking that the sender is not the owner.

### Check if first bidder
```js
byte "for sale"
app_global_get
int 1
==

byte "Owner"
app_global_get
byte "Highest bidder"
app_global_get
==
&&

byte "Owner"
app_global_get
txn Sender
!=
&&
bnz bid_0
```

## First bid

On the first bid, we sent two transactions. An app cal to the stateful contract and a payment transaction to the escrow. The escrow will store the funds until someone overbids or the auction ends. 

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000;
params.flatFee = true;
let index = appId
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address
const escrow = escrowAddress
let appArgs = undefined

let txn1 = algosdk.makeApplicationNoOpTxn(sender, params, index, appArgs)
let txn2 = algosdk.makePaymentTxnWithSuggestedParams(sender, escrow, (bid * 1000000), undefined, undefined, params)

const txns = [txn1, txn2]
const txGroup = algosdk.assignGroupID(txns)

let binaryTxs = [txn1.toByte(), txn2.toByte()]
let base64Txs = binaryTxs.map((binary) => AlgoSigner.encoding.msgpackToBase64(binary))

let signedTxs = await AlgoSigner.signTxn([
  {
    txn: base64Txs[0],
  },
  {
    txn: base64Txs[1],
  },
])

let binarySignedTxs = signedTxs.map((tx) => Buffer.from(tx.blob, "base64"));

await algodClient.sendRawTransaction(binarySignedTxs).do().catch((err) => {console.log(err)});
await waitForConfirmation(algodClient, signedTxs[0].txID, 4)
```

## Overbidding

Overbidding is slightly more complex. We now need to send a transaction from the escrow to the last bidder to return the last bid. For that transaction, the escrow program is needed. If you did not save the program you will need to fill in the appId and compile it again. 

With the escrow program, you can generate a 'lsig'. Which you can see as the signing key for stateless contract transactions. We can then sign the actual transaction, which is just a normal payment transaction, with the signLocigSigTransactionObject function.

There are two things different in this group transaction in comparison to the previous ones. The first difference is in the transaction fees. Since the escrow receives only the exact amount of the last bid, there are no funds left to pay fees. We can use a relatively new feature on Algorand to solve this, grouped fees. We can let transactions in a group pay for each other's fees. As long as the total of paid fees is enough the transactions will not fail. We let the app cal pay the escrows fees.

We will need to generate two new params objects, one for 2000 μAlgo fee and one with 0 μAlgo fee. To create a new object in memory and not just create a reference, we will need to convert the params object to JSON and back. (alternatively you could use the clone deep function to do this)

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000;
params.flatFee = true;

const index = appId
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address
const escrow = escrowAddress
let appArgs = undefined
const highestBidder = highestBidder
const highestBid = highestBid
const program = escrowProgram
const lsig = algosdk.makeLogicSig(new Uint8Array(Buffer.from(program, "base64")));

let params2000 = JSON.parse(JSON.stringify(params))
let params0 = JSON.parse(JSON.stringify(params))

params2000.fee = 2000
params2000.flatFee = true
params0.fee = 0
params0.flatFee = true

let txn1 = algosdk.makeApplicationNoOpTxn(sender, params2000, index, appArgs)
let txn2 = algosdk.makePaymentTxnWithSuggestedParams(sender, escrow, (payload.amount * 1000000), undefined, undefined, params);
let txn3 = algosdk.makePaymentTxnWithSuggestedParams(escrow, highestBidder, highestBid, undefined, undefined, params0);
```

The second difference has to do with the Algo Signer. To my knowledge, there is no way to send lsig signed transactions to the Algo Signer. This means that we can only send two of the three transactions out of the group to the Algo Signer. The Algo Signer does not accept that and gives back an error saying a transaction in the group is missing.

We can get around this limitation by sending the transactions individually to the Algo Signer. The Algo Signer will now warn the user that the transactions are part of an unknown group but they can now be signed. This is of course not ideal as there are now two sign interactions necessary instead of one.

```js
const txns = [txn1, txn2, txn3]
const txGroup = algosdk.assignGroupID(txns)

let signedTxn3 = algosdk.signLogicSigTransactionObject(txGroup[2], lsig)

let binaryTxs = [txn1.toByte(), txn2.toByte()]
let base64Txs = binaryTxs.map((binary) => AlgoSigner.encoding.msgpackToBase64(binary))

let signedTxs1 = await AlgoSigner.signTxn([
  {
    txn: base64Txs[0],
  }
])

let signedTxs2 = await AlgoSigner.signTxn([
  {
    txn: base64Txs[1],
  }
])

const signedTxs = [signedTxs1, signedTxs2, signedTxn3]

let binarySignedTxs = signedTxs.map((tx) => Buffer.from(tx.blob, "base64"));

await algodClient.sendRawTransaction(binarySignedTxs).do().catch((err) => {console.log(err)});
await waitForConfirmation(algodClient, signedTxs[0].txID, 4)
```

If there are less than 400 rounds left in the auction when someone overbids, the timer resets to 400 rounds. This gives someone always the opportunity to overbid again.

```js
// set bid 1 (end statement must return 1 to top of stack)
  set_bid_1:
  byte "Highest bid"
  gtxn 1 Amount
  app_global_put

  byte "Highest bidder"
  gtxn 1 Sender
  app_global_put

  global Round
  int 400
  +                           // increase timer with 400 rounds (~ 30 min) if lower than 400 rounds
  byte "End round"            // to always have an opportunity to overbid
  app_global_get
  >=

  bnz add_rounds

  int 1
  return
//

// add rounds (end statement must return 1 to top of stack)
  add_rounds:
  global Round
  int 400
  +
  store 10

  byte "End round"
  load 10
  app_global_put

  int 1
  return 
//
```

## Getting auction parameters

You can get the auction info from the global state of the stateful contract. In this example, we get the highest bid, highest bidder and end round from the contract. With the end round, we can calculate the time left in the auction.

```js
const app = await indexerClient.lookupApplications(auction.appId).do().catch((err) => {console.log(err)})

const globalState = app.application.params['global-state']

const highestBid = globalState.find(x => x.key === 'SGlnaGVzdCBiaWQ=').value.uint // amount in μAlgo

const highestBidderBase64 = globalState.find(x => x.key === 'SGlnaGVzdCBiaWRkZXI=').value.bytes
const highestBidder = algosdk.encodeAddress(new Uint8Array(Buffer.from(highestBidderBase64, "base64")))

const endRound = globalState.find(x => x.key === 'RW5kIHJvdW5k').value.uint
const roundsLeft = globalState.find(x => x.key === 'RW5kIHJvdW5k').value.uint - app['current-round']
```

# Ending auctions

Just like with bidding there are two scenarios with ending auctions. Either it is the first sale and the creator is still the owner or it is a secondary sale and the owner is no longer the creator.

Here just like with the second bidding option the escrow need to send a transaction. Here as well we can't send the lsig signed transactions as part of the group to the Algo Signer. We also need to let the app cal pay for the escrow transactions again.

It does not matter who sends the end auction transaction, since the sender does not need to transfer funds and can only payout in the preset royalty split. The sender only needs to pay the 0.003 - 0.004 algo transaction fees.

## First sale

In the first case, where the creator is the owner, the royalties split is 85% to the creator and 15% to the manager.

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000;
params.flatFee = true;

const index = appId
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address
const escrow = auction.escrowAddress
let appArgs = undefined
const creator = creatorAddress
const manager = managerAddress
const program = escrowProgram
const lsig = algosdk.makeLogicSig(new Uint8Array(Buffer.from(program, "base64")))

let params3000 = JSON.parse(JSON.stringify(params))
let params0 = JSON.parse(JSON.stringify(params))

params3000.fee = 3000
params3000.flatFee = true
params0.fee = 0
params0.flatFee = true

let txn1 = algosdk.makeApplicationNoOpTxn(sender, params3000, index, appArgs)
let txn2 = algosdk.makePaymentTxnWithSuggestedParams(escrow, creator, ((highestBid / 100) * 85), undefined, undefined, params0)
let txn3 = algosdk.makePaymentTxnWithSuggestedParams(escrow, manager, ((highestBid / 100) * 15), undefined, undefined, params0)

const txns = [txn1, txn2, txn3]
const txGroup = algosdk.assignGroupID(txns)

let signedTxs2 = algosdk.signLogicSigTransactionObject(txGroup[1], lsig)
let signedTxs3 = algosdk.signLogicSigTransactionObject(txGroup[2], lsig)

let binaryTxs = txn1.toByte()
let base64Txs = AlgoSigner.encoding.msgpackToBase64(binaryTxs)

let signedTxs1 = await AlgoSigner.signTxn([
  {
    txn: base64Txs,
  }
])

const signedTxs = [signedTxs1, signedTxs2, signedTxs3]

let binarySignedTxs = signedTxs.map((tx) => Buffer.from(tx.blob, "base64"))

await algodClient.sendRawTransaction(binarySignedTxs).do().catch((err) => {console.log(err)})
await waitForConfirmation(algodClient, signedTxs[0].txID, 4)
```

## Secondary sales

In the second case, the royalty split is 85% to the owner, 10% to the creator and 5% to the manager.

```js
let params = await algodClient.getTransactionParams().do()

params.fee = 1000;
params.flatFee = true;

const index = appId
const sender = await AlgoSigner.accounts({ ledger: 'TestNet' })[0].address
const escrow = auction.escrowAddress
let appArgs = undefined
const owner = ownerAddress
const creator = creatorAddress
const manager = managerAddress
const program = escrowProgram
const lsig = algosdk.makeLogicSig(new Uint8Array(Buffer.from(program, "base64")));

let params4000 = JSON.parse(JSON.stringify(params))
let params0 = JSON.parse(JSON.stringify(params));

params4000.fee = 4000
params4000.flatFee = true
params0.fee = 0
params0.flatFee = true

let txn1 = algosdk.makeApplicationNoOpTxn(sender, params4000, index, appArgs)
let txn2 = algosdk.makePaymentTxnWithSuggestedParams(escrow, owner, ((highestBid / 100) * 85), undefined, undefined, params0)
let txn3 = algosdk.makePaymentTxnWithSuggestedParams(escrow, creator, ((highestBid / 100) * 10), undefined, undefined, params0)
let txn4 = algosdk.makePaymentTxnWithSuggestedParams(escrow, manager, ((highestBid / 100) * 5), undefined, undefined, params0)

const txns = [txn1, txn2, txn3, txn4]
const txGroup = algosdk.assignGroupID(txns)

let signedTxs2 = algosdk.signLogicSigTransactionObject(txGroup[1], lsig)
let signedTxs3 = algosdk.signLogicSigTransactionObject(txGroup[2], lsig)
let signedTxs4 = algosdk.signLogicSigTransactionObject(txGroup[3], lsig)

let binaryTxs = txn1.toByte()
let base64Txs = AlgoSigner.encoding.msgpackToBase64(binaryTxs)

let signedTxs1 = await AlgoSigner.signTxn([
  {
    txn: base64Txs,
  }
])

const signedTxs = [signedTxs1, signedTxs2, signedTxs3, signedTxs4]

let binarySignedTxs = signedTxs.map((tx) => Buffer.from(tx.blob, "base64"))

await algodClient.sendRawTransaction(binarySignedTxs).do().catch((err) => {console.log(err)})
await waitForConfirmation(algodClient, signedTxs[0].txID, 4)
```

After the auction is ended the owner can start a new one.