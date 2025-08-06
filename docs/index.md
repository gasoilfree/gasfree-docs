# GasFree Developer Documentation

## 1. Overview

GasFree aims to provide users with a TRC-20/ERC-20 transfer solution that does not require a native token to pay for transaction gas fees. It specifically includes four roles: GasFree accounts, Service-Providers, wallets, and users, as shown in Figure 1. below.

![overview](https://raw.githubusercontent.com/gasfree/gasfree-docs/main/images/overview.png)

**GasFree Account**: Generated according to a specific algorithm. Its permissions are controlled by the user's EOA (Externally Owned Account) address. The user can sign a signature to authorize GasFree transfer of this account.

**Service-Provider**: The GasFree service-provider is responsible for collecting users' GasFree transfer authorizations, submitting them to the blockchain, and paying the Gas fees on behalf of the users. The Service-Provider may charge a certain handling fee after the transaction is completed.

**Wallet**: After connecting to the GasFree service, the wallet provides end-users with interface functions such as fund inquiry for GasFree accounts and signature for transfer authorization.

## 2. Authorization Process

Integrating GasFree into the wallet involves processes such as constructing transfer authorization, signing the transfer authorization, and the provider submitting the authorization to the blockchain on behalf of the user. For detailed information about the Provider API interface, please refer to Section 5 of this document. After the integration, the main interaction process is as follows:

**1. Prerequisite**:

- The Service-Provider supports GasFree transfers for multiple types of tokens. Call [/api/v1/config/token/all](#get-/api/v1/config/token/all) to obtain the list of supported tokens.

- When a token that the Provider does not currently support is transferred into a GasFree account, the user uses the withdrawal page provided by GasFree official to withdraw the token to their EOA (Externally Owned Account) address. The link to the withdrawal page is: [https://gasfree.io/withdraw](https://gasfree.io/withdraw)

- GasFree supports multiple Service-Providers. Just make a call to [/api/v1/config/provider/all](#get-/api/v1/config/provider/all) to obtain the list of available Service-Providers.

- The GasFree account is in an inactive state by default. An inactive GasFree account will be automatically activated during the first Gasfree transfer. When activating the GasFree account, an additional \`activation fee\` will be charged. For subsequent GasFree transfer authorizations, only the \`transfer fee\` will be charged.

  **2. Preparation**:

  - When a user conducts a GasFree transfer authorization, first call [/api/v1/address/{accountAddress}](#get-/api/v1/address/{accountaddress}) to query his/her GasFree account info, including status, balance, and current nonce. Based on this info, construct a GasFree transfer authorization.

  **3. Sign authorization**:

  - The signature algorithm for GasFree transfer authorization is designed to support subsequent upgrades of the signature algorithm. The signature algorithm of the current version is compatible with the EIP712 specification. For the specific signature algorithm of the authorization, please refer to Section 3.2.

  **4. Submit authorization**:

  - Use [/api/v1/gasfree/submit](#post-/api/v1/gasfree/submit) to send the signed GasFree transfer authorization to the Service-Provider.

  **5. Handling of provider response**:

  - After receiving the GasFree transfer authorization submitted by the user, the provider will conduct a verification and immediately return the verification result to the wallet. The wallet should handle the response from the Provider:

  - If successful, notify the user the authorization verification has passed, and include a globally unique authorization **traceId**. Subsequently, the on-chain status of this authorization can be tracked based on this **traceId**.

  - If failed, notify the user of the failure, stating that the authorization has been discarded and this request will not trigger an actual transaction on the blockchain.

  **6. Monitoring of the transfer authorization**:

  - Call [/api/v1/gasfree/{tranceId}](#get-/api/v1/gasfree/{traceid}) to query the status of the GasFree transfer authorization.

## 3. Signature Algorithm and Provider Endpoint

#### 3.1 Parameters

The GasFree transfer authorization involves the following parameters. Their values on the TRON mainnet and the Nile testnet are shown in Table 1\.

| Parameter         | TRON \- mainnet                    | TRON \- Nile testnet               |
| :---------------- | :--------------------------------- | :--------------------------------- |
| chainId           | ​​728126428                        | 3448148188                         |
| verifyingContract | TFFAMQLZybALaLb4uxHA9RBE7pxhUAjF3U | THQGuFzL87ZqhxkgqYEryRAd7gqFqL5rdc |

Table 1\. GasFree Transfer Authorization Parameters

#### 3.2 Authorization Construction and Signature

**General structure**

- **MessageDomain:**

```javascript
const Permit712MessageDomain = {
  name: 'GasFreeController',
  version: 'V1.0.0',
  chainId: 3448148188 // tronWeb.toDecimal('0xcd8690dc'),
  verifyingContract: 'THQGuFzL87ZqhxkgqYEryRAd7gqFqL5rdc'
}
```

    Field description:

- token: address of transferring token

  - name: ‘GasFreeController’, fixed value
  - version: ‘V1.0.0’, fixed value
  - chainId: chainId in decimal, please refer to section 3.1
  - verifyingContract: GasFreeController contract address, please refer to section 3.1

- **MessageTypes: fixed value**

```javascript
const Permit712MessageTypes = {
  PermitTransfer: [
    { name: "token", type: "address" },
    { name: "serviceProvider", type: "address" },
    { name: "user", type: "address" },
    { name: "receiver", type: "address" },
    { name: "value", type: "uint256" },
    { name: "maxFee", type: "uint256" },
    { name: "deadline", type: "uint256" },
    { name: "version", type: "uint256" },
    { name: "nonce", type: "uint256" },
  ],
};
```

- **Message body:**

```javascript
const message = {
  token: "TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf",
  serviceProvider: "TKtWbdzEq5ss9vTS9kwRhBp5mXmBfBns3E",
  user: "THvMiWQeVPGEMuBtAnuKn2QpuSjqjrGQGu",
  receiver: "TMDKznuDWaZwfZHcM61FVFstyYNmK6Njk1",
  value: "90000000",
  maxFee: "20000000",
  deadline: "1728638679",
  version: 1,
  nonce: 0,
};
```

    Field description:

- token: address of transferring token
  - provider: address of Service-Provider
  - user: user’s EOA address, not GasFree address
  - receiver: recipient address of the transfer
  - value: amount of the transfer, like 90 USDT equals 90 \* 10^6
  - maxFee: maximum fee limit (transfer fee \+ activation fee), the smallest unit; for example, 20 USDT equals 20 \* 10^6
  - deadline: expiration timestamp of the transfer, the unit is seconds; for example, Date.now() / 1000
  - version: version of the signature, the current version is 1
  - nonce: nonce value of the transfer authorization; for example, 0

**Sign with Wallet**

Using the above structure, you can call the TRON wallet to sign the TIP-712 Message, which will provide the required `sig` parameter.

**TronLink example:**

```javascript
const signature = await window.tron.tronWeb.trx._signTypedData(Permit712MessageDomain, Permit712MessageTypes, message); // remove 0x
```

#### 3.3 Endpoint

Wallets/users can interact with the Provider through the following endpoint, including submitting GasFree transfer authorizations, querying subsequent statuses, etc. They can also withdraw tokens that are not yet supported by the Providers to the user's EOA (Externally Owned Account) address.

| Services          | TRON \- Mainnet                                                | TRON \- Nile testnet                                                     |
| :---------------- | :------------------------------------------------------------- | :----------------------------------------------------------------------- |
| provider-\#1      | [https://open.gasfree.io/tron/](https://open.gasfree.io/tron/) | [https://open-test.gasfree.io/nile/](https://open-test.gasfree.io/nile/) |
| GasFree website   | [https://gasfree.io](https://gasfree.io)                       | [https://test.gasfree.io](https://test.gasfree.io)                       |
| Assets Withdrawal | [https://gasfree.io/withdraw](https://gasfree.io/withdraw)     | [https://test.gasfree.io/withdraw](https://test.gasfree.io/withdraw)     |

Table 2\. Endpoints of the Services

Note: Currently, GasFree only provides services in the TRON network and can be extended to Ethereum and various EVM-compatible chains in the future.

#### 3.4 Faucet

Nile testnet faucet: [https://nileex.io/join/getJoinPage](https://nileex.io/join/getJoinPage); developers can obtain TRX and USDT at this page for test purposes.

## 4. Notes

**1. It is not recommended that the testnet environment be provided to users.**

**The GasFree environment on the TRON testnet is only available for developer teams to conduct testing and debugging during integration.**

**It is strongly recommended to close the entrance to the GasFree test environment for ordinary users after the official launch to prevent users from filling in the wrong recipient addresses and incurring asset losses.**

**2. It is recommended to conduct balance and status verification before submitting transfer authorization. For details, please refer to the definition of [GasFree account interface](#get-/api/v1/address/{accountaddress}).**

**Unless otherwise specified, all references to asset balances and amounts in this document are in the smallest unit.**

## 5. APIs

### API Authentication

After applying for access to the GasFree project, the access party will receive a pair of keys: **API Key** and **API Secret**. This key pair is used to sign API requests. Please make sure to keep the API Secret properly. The GasFree server verifies the requests, and the signature information is transmitted through the HTTP header.

Users can apply for API Key and API Secret at [GasFree Developers Center](https://developer.gasfree.io/) once verified backstage, please send an email with your User Id to admin@gasfree.io if your user is not verified in time.

**HTTP header definition:**

```javascript
{
  "Timestamp": 1731912286, // unit: second
  "Authorization": "ApiKey {api_key}:{signature}"
}
```

**API Signature and Authentication**

1. Construct the string to be signed: including the request method, request path, and timestamp.
2. Calculate the signature: Use the HMAC-SHA256 algorithm and the API secret to perform a hash operation on the string to be signed, and then perform base64 encoding on the result.
3. Add the signature to the request header.

**Example:**

```py
import hmac
import hashlib
import base64
import time
import requests

API_KEY = 'YOUR_API_KEY'
API_SECRET = 'YOUR_API_SECRET'

method = 'GET'

// If running in test env(nile testnet)，path prefix should be '/nile'
// If running in mainnet (mainnet), path prefix should be '/tron'
path = '/nile/api/v1/config/token/all'
timestamp = int(time.time())
message = f'{method}{path}{timestamp}'
signature = base64.b64encode(
    hmac.new(
        API_SECRET.encode('utf-8'),
        message.encode('utf-8'),
        hashlib.sha256
        ).digest()
    ).decode('utf-8')

url = 'https://test-nile.gasfree.io' + path
headers = {
    'Timestamp': f'{timestamp}',
    'Authorization': f'ApiKey {API_KEY}:{signature}'
}

response = requests.get(url, headers=headers)
print(response.json())
```

### API Format Definition

When the service runs normally, the HTTP code returned by the API is \`200\`. Parameter errors or runtime errors are reflected in the HTTP body.

**HTTP body definition:**

```javascript
{
  "code": 200, // indicates if request returns normally, 200 means normal, 400 indicates failure due to input error, and 500 indicates failure due to runtime error
  "reason": null, // exception name
  "message": "", // exception info, usually a combined string
  "data": "" //  return data of the API interface; otherwise, it is null
}
```

**Error return example:**

```javascript
{
  "code": 400,
  "reason": "GasFreeAddressNotFoundException",
  "message": "123",
  "data": null
}
```

**Successful return example:**

```javascript

{
  "code": 200,
  "reason": null,
  "message": null,
  "data": {
    "tokens": [
      {
        "tokenAddress": "TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf",
        "createdAt": "2024-09-10T09:46:24.801+00:00",
        "updatedAt": "2024-09-11T06:57:10.244+00:00",
        "activateFee": 10000000,
        "transferFee": 10000000,
        "supported": true,
        "symbol": "USDT",
        "decimal": 6
      }
    ]
  }
}
```

### GET /api/v1/config/token/all

The complete request URL for the services provided by Provider \#1 on the TRON mainnet is: [https://open.gasfree.io/tron/api/v1/config/token/all](https://open.gasfree.io/tron/api/v1/config/token/all)

Get the contract list of all supported tokens.

- **Request parameters**: None
- **Return**: \`tokens\`, contain the contract list of all supported tokens.  
  The field description of the token structure:
  - tokenAddress: token contract address
  - activateFee: the activation handling fee paid in the transfer token, is measured in the smallest unit of that token; this value can be adjusted according to actual circumstances later.
  - transferFee: the transfer fee, paid in the transfer currency, is measured in the smallest unit of that token; this value can be adjusted according to actual circumstances later.
  - symbol: token name
  - decimal: token precision

**Return example:**

```javascript
{
  "code": 200,
  "reason": null,
  "message": null,
  "data": {
    "tokens": [
      {
        "tokenAddress": "TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf",
        "createdAt": "2024-10-09T08:14:12.560+00:00",
        "updatedAt": "2024-10-09T08:14:12.560+00:00",
        "activateFee": 10000000, // stands for 10 USDT
        "transferFee": 10000000, // stands for 10 USDT
        "supported": true,
        "symbol": "USDT",
        "decimal": 6
      }
    ]
  }
}
```

### GET /api/v1/config/provider/all

Get the list of all supported service providers.

- **Request parameters**: None
- **Return**: \`providers\`, contains the list of all available Service-Providers.  
  The field description of the provider structure:
  - address: address of the Service-Provider
  - name: Provider’s name
  - icon: Provider’s icon
  - website: Provider’s website
  - config: system parameters of the Provider
    - maxPendingTransfer: the maximum number of transfer authorizations waiting to be on chain that users are allowed to submit; if the number exceeds this limit, an error will be reported upon submission, and users must wait until the previous transfer authorizations are successfully on chain before they can continue to send new ones. (Currently, only one transfer authorization in the "pending" status is supported for the same account.)
    - minDeadlineDuration: the minimum deadline interval; transfer authorizations with a deadline less than this value from the current time will be rejected; the unit is seconds.
    - maxDeadlineDuration: the maximum deadline interval; transfer authorizations with a deadline more than this value from the current time will be rejected; the unit is seconds.
    - defaultDeadlineDuration: default deadline interval, recommended.

**Return example:**

```javascript
{
  "code": 200,
  "reason": null,
  "message": null,
  "data": {
    "providers": [
      {
        "address": "TQ6qStrS2ZJ96gieZJC8AurTxwqJETmjfp",
        "name": "Provider-1",
        "icon": "",
        "website": "",
        "config": {
          "maxPendingTransfer": 1,
          "minDeadlineDuration": 60,
          "maxDeadlineDuration": 600,
          "defaultDeadlineDuration": 180
        }
      }
    ]
  }
}
```

### GET /api/v1/address/{accountAddress}

Query the relevant information of the user's GasFree account, including whether it is activated, the nonce value, the supported tokens, and assets.

**1. Query the latest assets of the GasFree account on-chain. Combine the “frozen” value provided by this interface to limit the maximum transfer-out amount of users. Otherwise, it may lead to submission failures or transfer failures.**

- **Request parameters**: \`accountAddress\` is the user's EOA address.
- **Return**: asset info of the user's GasFree account.  
  Field description:
  - accountAddress: user’s EOA address
  - gasFreeAddress: user's GasFree account address
  - active: activation status of the GasFree account
  - nonce: the recommended nonce value for the next GasFree Transfer; since there may be transfer authorization waiting to be on-chain, the nonce on the chain may not necessarily be applicable; the back end will provide the recommended nonce value by comprehensively considering the situations on the chain and in the queue.
  - **allowSubmit: whether users are currently allowed to continue submitting transfer authorization.**
  - assets: details of the assets that users are processing through the GasFree service provider.
    - tokenAddress: contract address of the token
    - tokenSymbol: token name
    - activateFee: the activation handling fee for this token, is measured in the smallest unit of this token.
    - transferFee: the transfer fee for this token, is measured in the smallest unit of this token.
    - decimal: precision of the token
    - frozen: current amount of the GasFree transfer in progress, including the fees

**Return example:**

```javascript
{
  "code": 200,
  "reason": null,
  "message": null,
  "data": {
    "accountAddress": "TKtWbdzEq5ss9vTS9kwRhBp5mXmBfBns3E",
    "gasFreeAddress": "TLGVf7MRsLG7XxBkJKy8wnCVcDnAeXYNCb",
    "active": true,
    "nonce": 1,
    "allow_submit": false,
    "assets": [
      {
        "tokenAddress": "TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf",
        "tokenSymbol": "USDT",
        "activateFee": 10000000,
        "transferFee": 10000000,
        "decimal": 6,
        "frozen": 0
      }
    ]
  }
}
```

### POST /api/v1/gasfree/submit

Initiate a GasFree transfer authorization.

- **Request parameters**:
  - requestId: request id in uuid4 format, not mandatory, for issue checking purposes
  - token: address of the token for transfer
  - Provider: address of the service-provider
  - user: user's EOA address, not the GasFree address
  - receiver: recipient address of the transfer
  - value: amount for the transfer
  - maxFee: the maximum fee limit (transfer fee \+ activation fee)
  - deadline: the expiration timestamp of this transfer, in seconds
  - version: signature version of transfer authorization
  - nonce: nonce value of this transfer authorization
  - sig: user's signature of the GasFree transfer authorization

**Request example:**

```javascript
{
  "requestId": "1e0a9e0e-7a91-48e8-9fa1-a473347f54ec",
  "token": "TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf",
  "provider": "TCETRh3aED4kdkaYQY7CcxeTJtrQvwBpNT",
  "user": "TKtWbdzEq5ss9vTS9kwRhBp5mXmBfBns3E",
  "receiver": "TEkj3ndMVEmFLYaFrATMwMjBRZ1EAZkucT",
  "value": 100000,
  "maxFee": 1000000,
  "deadline": 1728638679,
  "version": 1,
  "nonce": 9,
  "sig": "9dfd3638e03af56bc93fa619e20e1e743f6b5c0d9a49bd340a94e27f6d6a6413618cc8481b9d5816a793982d8047daa0badbade02f92814e78a9894efd9877341b"
}
```

- **Return**: basic information of this GasFree transfer authorization.  
  Field description:
  - id: traceId of GasFree transfer authorization, not the transactionId
  - accountAddress: user's EOA address
  - gasFreeAddress: user's GasFree account address
  - providerAddress: address of the service-provider
  - targetAddress: recipient address
  - tokenAddress: contract address of the transferred token
  - amount: amount of the transfer
  - maxFee: limit of maximum fee
  - signature: user's signature
  - version: signature version of transfer authorization
  - nonce: nonce value specified for the transfer
  - expiredAt: expiration time of this transfer
  - state: current state of this transfer, with the following values:
    - WAITING, // Not started
    - INPROGRESS, // In progress
    - CONFIRMING, //Confirming
    - SUCCEED, // Successful
    - FAILED, // Failed
  - estimatedActivateFee: estimated activation fee of the address
  - estimatedTransferFee: estimated transfer fee

**Return example:**

```javascript
{
  "code": 200,
  "reasone": null,
  "message": null,
  "data": {
    "id": "6ab4c27c-f66b-4328-b40f-ffdc6cf1ca60",
    "createdAt": "2024-09-10T08:11:50.822+00:00",
    "updatedAt": "2024-09-10T08:11:50.822+00:00",
    "accountAddress": "TKtWbdzEq5ss9vTS9kwRhBp5mXmBfBns3E",
    "gasFreeAddress": "TLGVf7MRsLG7XxBkJKy8wnCVcDnAeXYNCb",
    "providerAddress": "TQ6qStrS2ZJ96gieZJC8AurTxwqJETmjfp",
    "targetAddress": "TQ6qStrS2ZJ96gieZJC8AurTxwqJETmjfp",
    "tokenAddress": "TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf",
    "amount": 1000000,
    "maxFee": 1000000,
    "signature": "f5bb43b90a5295759819edb029a1c2d6cd50acc9bd4283fd73c4be16fe163d154ec4f06fd70c8d63cc2ff2346ea1c86abf019da944fb62eb28fe0a8f6e12e1931c",
    "nonce": 0,
    "expiredAt": "2024-09-11T08:05:08.000+00:00",
    "state": "WAITING",
"estimatedActivateFee": 1000,
"estimateTransferFee": 300
  }
}
```

**Error example:**

```javascript
{
  "code": 400,
  "reason": "DeadlineExceededException",
  "message": "deadline exceeded",
  "data": null
}
```

- **Error type**s:
  - ProviderAddressNotMatchException: Provider’s address does not match
  - DeadlineExceededException: authorization is expired.
  - InvalidSignatureException: signature is incorrect.
  - UnsupportedTokenException: token transferred is not supported.
  - TooManyPendingTransferException: there are too many pending transfer authorizations.
  - VersionNotSupportedException gasfree: signature version of transfer authorization is not supported.
  - NonceNotMatchException: nonce does not match.
  - MaxFeeExceededException: estimated fee exceeds the limit of the maximum fee.
  - InsufficientBalanceException: insufficient balance

####

### GET /api/v1/gasfree/{traceId}

Query the details of a specified GasFree transfer authorization. **All fields related to the transaction will be empty if the query is made either before the transfer authorization is submitted to the chain or the verification before submitting fails.**

- **Required parameters**:
  - traceId: id of the GasFree transfer authorization; a path parameter.
- **Return**: detailed info on the GasFree transfer authorization.

           Field description:

- id: traceId of the GasFree transfer authorization recorded, not the transactionId.
  - createdAt: creation time of the transfer authorization.
  - accountAddress: user's EOA address.
  - gasFreeAddress: user's GasFree account address.
  - providerAddress: service-provider address.
  - targetAddress: recipient address.
  - nonce: nonce value of theGasFree transfer authorization.
  - tokenAddress: contract address of the token transferred.
  - amount: amount actually transferred to the recipient's address.
  - expiredAt: expiration time of this transfer.
  - state: current state of this transfer, with the following values:
    - WAITING, // Not started
    - INPROGRESS, // In progress
    - CONFIRMING, //Confirming
    - SUCCEED, // Successful
    - FAILED, // Failed
  - estimatedActivateFee: estimated activation fee.
  - estimatedTransferFee: estimated transfer fee.
  - estimatedTotalFee: estimated activation fee pulse estimated transfer fee.
  - estimatedTotalCost: estimated amount that the user needs to pay, including the total fee and the transfer amount.
  - txnHash: transactionId of the corresponding transaction on-chain.
  - txnBlockNum: block height of the corresponding transaction on-chain.
  - txnBlockTimestamp: timestamp of the block contains the corresponding transaction, in milliseconds.
  - txnState: state of the corresponding on-chain transaction, with the following values:
    - INIT, // initial state
    - NOT_ON_CHAIN, // Not on-chain
    - ON_CHAIN, // On-chain, not solidified
    - SOLIDITY, // Solidified
    - ON_CHAIN_FAILED, // failed to be on-chain
  - txnActivateFee: actual activation fee consumed.
  - txnTransferFee: actual transfer fee consumed.
  - txnTotalFee: actual total fee consumed.
  - txnAmount: actual transferred amount to the recipient address
  - txnTotalCost: actual amount paid by the user, including the total fee and the transferred amount.
