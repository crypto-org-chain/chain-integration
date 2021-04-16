# Crypto.org Chain Blocks and Transactions

This document describes the block and transaction structure of Crypto.org Chain and explain different ways to extract and parse the details of them.

- Based on Crypto.org Chain [v1.2.1](https://github.com/crypto-org-chain/chain-main/releases/tag/v1.2.1) release. 
- Last updated at: 2021-04-16

## Table of Content

- [Changelog](#changelog)
- [Common APIs](#common-apis)
- [Common Transaction Details](#common-transaction-details)
  - [Block Height](#block-height)
  - [Transaction Hash](#transaction-hash)
  - [Transaction Fee](#transaction-fee)
  - [Assets and Amount](#assets-and-amount)
- [Bank](#bank)
  - [MsgSend](#msg-send)
  - [MsgMultiSend](#msg-multisend)
- [Distribution](#distribution)
  - [MsgSetWithdrawAddress](#msg-set-withdraw-address)
  - [MsgWithdrawDelegatorReward](#msg-withdraw-delegator-reward)
  - [MsgWithdrawValidatorCommission](#msg-withdraw-validator-commission)
  - [MsgFundCommunityPool](#msg-fund-community-pool)

## Changelog

- 2021-04-16 First Draft
- 2021-04-16 Add Distribution module

## Common APIs

<a id="tendermint-block-api">

#### 1. Tendermint Block API

- URL: https://mainnet.crypto.org:26657/block?height=[height]
- This API returns block details, list of transaction bytes and consensus commits.
- Example: https://mainnet.crypto.org:26657/block?height=10000

<a id="tendermint-block-results-api">

#### 2. Tendermint Block Results API

- URL: https://mainnet.crypto.org:26657/block?height=[height]
- This API returns the events of the block. These events include the outcomes from transactions, and block changes such as block rewards minted and distributed as well as consensus state updates such as validator missing block counts
- Example: https://mainnet.crypto.org:26657/block?height=10000

<a id="cosmos-transaction-query-api">

#### 3. Cosmos Transaction Query API

- https://mainnet.crypto.org:1317/comsos/tx/v1beta1/txs/[hash]
- This API returns the parsed transaction details and events of a particular transaction hash
- Example: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/0C5E617B0577B047D78EBF5313B8B70DF69E9535E17B964303BD04947B11B660

<a id="cosmos-transaction-search-api">

#### 4. Cosmos Transaction Search API

- URL: https://mainnet.crypto.org:1317/comsos/tx/v1beta1/txs
- This API support event based query and returns parsed transactions. Common events includes

| event | Description | Example |
| --- | --- | --- |
| tx.height | Transaction in a particular block | [txs?events=tx.height%3D10000](https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs?events=tx.height%3D10000) |
| message.module & message.action | Search for messages belonged to a particular module and actions. Note that this index will degrade when more transaction of its kind grows in number | [txs?events=message.module%3D%27bank%27&events=message.action%3D%27send%27](https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs?events=message.module%3D%27bank%27&events=message.action%3D%27send%27) |
| message.sender | Search for message with particular signer | [/txs?events=message.sender=%27cro18undzhe3fmmav2x3csx8m00m5yupkcc7qzz4ec%27](https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs?events=message.sender=%27cro18undzhe3fmmav2x3csx8m00m5yupkcc7qzz4ec%27)  

Note:
1. The API supports pagination, make sure you have iterate all the pages `pagination.offset=[offset starting from 0]&pagination.limit=[record per page]`
2. The performance will degrade if you are searching for a result set that will grow over time. For example if 
3. Multiple events in a single search is queried by `AND` condition. i.e If you do `tx.height` and `message.sender`. It will search for transactions happened on that block height AND signed by the sender.

## Common Transaction Details

<a id="block-height" />

### 1. Block Height

- [Tendermint Block API](#tendermint-block-api): `result.block.header.height`
- [Tendermint Block Results API](#tendermint-block-results-api): `result.height`
- [Cosmos Transaction Query API](#cosmos-transaction-query-api):  `tx_response.height`
- [Cosmos Transaction Search API](#cosmos-transaction-query-api): `tx_response[index].height`

<a id="transaction-hash" />

### 2. Transaction Hash

- [Tendermint Block API](#tendermint-block-api): `Uppercase(SHA256(Base64Decode(result.block.data.txs[index])))`
- [Tendermint Block Results API](#tendermint-block-results-api): Not available, should use [Tendermint Block API](#tendermint-block-api). Match transaction `[index]` in `result.txs_results` with `result.block.data.txs[index]`
- [Cosmos Transaction Query API](#cosmos-transaction-query-api):  `tx_response.txhash`
- [Cosmos Transaction Search API](#cosmos-transaction-query-api): `tx_response[index].txhash`

<a id="transaction-fee" />

### 3. Transaction Fee

- [Cosmos Transaction Query API](#cosmos-transaction-query-api):  `tx.auth_info.fee.amount`
- [Cosmos Transaction Search API](#cosmos-transaction-query-api): `tx[index].auth_info.fee.amount`

Transaction fee is an [Asset Array](#asset-array), meaning that a transaction can pay fee in more than one token types.

<a id="assets-and-amount" />

### 4. Assets and Amount

There are mainly two types of assets and amount representation:

<a id="asset-object" />

#### 1. Single object

This is commonly seen in `staking` module but may appear in other module as well. It represents a single token type.

Example
```json
{
    "denom": "basecro",
    "amount": "10000"
}
```

`denom` is the asset type and `amount` is the amount. The `amount` is always in string for precision accuracy. Please make sure your language is capable to handle big integer / number.

It is highly recommended to use library similar to [bignumber.js](https://mikemcl.github.io/bignumber.js/) in your language to handle the `amount`.

##### Total supply:
The total supply of CRO is *30 billion* (`30,000,000,000`). `1 CRO=1_0000_0000 basecro`. So in basic unit `basecro` it can be up to `3,000,000,000,000,000,000`.

<a id="asset-array" />

#### 2. Array

This is commonly seen in most message types. It represents a list of tokens.

At the time of writing there will only be a single entry in this array because `basecro` (or `basetcro` in Croeseid Testnet) is the only supported asset on Crypto.org Chain. However, after IBC transfer and other coins issuance methods are enabled, there will be more asset types.

Example
```json
[
    {
        "denom": "basecro",
        "amount": "10000"
    },
    {
        "denom": "apple",
        "amount": "1000"
    }
]
```

Each object in the array has the same format as [Single Object](#asset-object).


<a id="asset-string">

#### 3. String

This is commonly seen in events' attributes of block and transaction.

At the time of writing there will only be a single entry in this array because `basecro` (or `basetcro` in Croeseid Testnet) is the only supported asset on Crypto.org Chain. However, after IBC transfer and other coins issuance methods are enabled, there will be more asset types.

Example
```json
{
    "key": "amount",
    "value": "1234basecro,5678apple"
}
```

## Bank

<a id="msg-send" />

### 1. MsgSend

Simple transfer message

- Funds movement: Yes


#### Protobuf Structure

```go
type MsgSend struct {
	FromAddress string                                   `protobuf:"bytes,1,opt,name=from_address,json=fromAddress,proto3" json:"from_address,omitempty" yaml:"from_address"`
	ToAddress   string                                   `protobuf:"bytes,2,opt,name=to_address,json=toAddress,proto3" json:"to_address,omitempty" yaml:"to_address"`
	Amount      github_com_cosmos_cosmos_sdk_types.Coins `protobuf:"bytes,3,rep,name=amount,proto3,castrepeated=github.com/cosmos/cosmos-sdk/types.Coins" json:"amount"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/0C5E617B0577B047D78EBF5313B8B70DF69E9535E17B964303BD04947B11B660

#### Details

| Detail | How | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.bank.v1beta1.MsgSend"` | String |
| From address | `tx.body.messages[message_index].from_address` | String |
| To address | `tx.body.messages[message_index].to_address` | String |
| Amount | `tx.body.messages[message_index].amount` | [Asset Array](#asset-array)

<a id="msg-multisend" />

### 2. MsgMultiSend

Multiple inputs, multiple outputs transfer message.

- Funds movement: Yes

#### Protobuf

```go
type MsgMultiSend struct {
	Inputs  []Input  `protobuf:"bytes,1,rep,name=inputs,proto3" json:"inputs"`
	Outputs []Output `protobuf:"bytes,2,rep,name=outputs,proto3" json:"outputs"`
}
type Input struct {
	Address string                                   `protobuf:"bytes,1,opt,name=address,proto3" json:"address,omitempty"`
	Coins   github_com_cosmos_cosmos_sdk_types.Coins `protobuf:"bytes,2,rep,name=coins,proto3,castrepeated=github.com/cosmos/cosmos-sdk/types.Coins" json:"coins"`
}
type Output struct {
	Address string                                   `protobuf:"bytes,1,opt,name=address,proto3" json:"address,omitempty"`
	Coins   github_com_cosmos_cosmos_sdk_types.Coins `protobuf:"bytes,2,rep,name=coins,proto3,castrepeated=github.com/cosmos/cosmos-sdk/types.Coins" json:"coins"`
}
```

#### Example 

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/6CD89C9F32A4F4E918B2BCD722A9429693E3372E3F882BA4A460F2588A2EE0B3


#### Details

| Detail | How | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.bank.v1beta1.MsgMultiSend"` | String |
| From Addresses | `tx.body.messages[message_index].inputs[m].address` where `m>=1`. There can be multiple (`m`) from addresses. | String |
| From Amounts | `tx.body.messages[message_index].inputs[m].coins` where `m>=1`. There can be multiple (`m`) from addresses and their corresponding input amount.  | [Asset Array](#asset-array)|
| To Addresses | `tx.body.messages[message_index].outputs[n].address` where `n>=1`. There can be multiple (`n`) destination addresses. | String |
| To Amounts | `tx.body.messages[message_index].outputs[n].coins` where `n>=1`. There can be multiple (`n`) destination addresses and their corresponding input amount.  | [Asset Array](#asset-array)

## Distribution

<a id="msg-set-withdraw-address" />

### 1. MsgSetWithdrawAddress

Sets the withdraw address for a delegator (or validator self-delegation)

- Funds movement: No (Pay for fee only)

#### Protobuf

```go
type MsgSetWithdrawAddress struct {
	DelegatorAddress string `protobuf:"bytes,1,opt,name=delegator_address,json=delegatorAddress,proto3" json:"delegator_address,omitempty" yaml:"delegator_address"`
	WithdrawAddress  string `protobuf:"bytes,2,opt,name=withdraw_address,json=withdrawAddress,proto3" json:"withdraw_address,omitempty" yaml:"withdraw_address"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/D4FCC8E1403677157D367A88A0832B9E411BDC4E029954FC133DB60296CF3DE3

#### Details

TODO

<a id="msg-withdraw-delegator-reward">

### 2. MsgWithdrawDelegatorReward

Withdraw delegation rewards from a single validator

Funds movement: Yes

#### Protobuf

```go
type MsgWithdrawDelegatorReward struct {
	DelegatorAddress string `protobuf:"bytes,1,opt,name=delegator_address,json=delegatorAddress,proto3" json:"delegator_address,omitempty" yaml:"delegator_address"`
	ValidatorAddress string `protobuf:"bytes,2,opt,name=validator_address,json=validatorAddress,proto3" json:"validator_address,omitempty" yaml:"validator_address"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/3B36AA1AC81ACD58E7A06C21353DB0FC40A70EDBF6BD2CD23D7BEDC7A0F56318

#### Details

| Detail | How | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.distribution.v1beta1.MsgWithdrawDelegatorReward"` | String |
| Delegator | `tx.body.messages[message_index].delegator_address` | String |
| Withdraw From Validator | `tx.body.messages[message_index].validator_address` | String |
| Withdraw To Address | `tx_response.logs[message_index].events[event_index].attributes[attribute_index].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "recipient"`.  | String |
| Withdraw Reward Amount | `tx_response.logs[message_index].events[event_index].attributes[attribute_index].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "amount"`.  | [Asset String](#asset-string)|

<a id="msg-withdraw-validator-commission">

### 3. MsgWithdrawValidatorCommission

Withdraws the full commission of a validator to the validator creator (initial delegator) address.

#### Protofbuf

```json
type MsgWithdrawValidatorCommission struct {
	ValidatorAddress string `protobuf:"bytes,1,opt,name=validator_address,json=validatorAddress,proto3" json:"validator_address,omitempty" yaml:"validator_address"`
}
```

#### Example

- Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/3739F76EF67A61D6F0163A5B177EA64ED80B67D9AEF8435C525913E69026D320
- Message Index: `1`

#### Details

This transaction will trigger an internal transfer from the "Distribution" module account to the withdraw to address. Note that the "Distribution" module account is an internal account in Crypto.org Chain to hold the rewards, commissions and community pool funds before they are being distributed.

Ths "Distribution" module account is different on different chain. In Crypto.org Chain Mainnet, it is [cro1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8lyv94w](https://mainnet.crypto.org:1317/cosmos/auth/v1beta1/accounts/cro1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8lyv94w).

| Detail | How | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.distribution.v1beta1.MsgWithdrawValidatorCommission"` | String |
| Validator | `tx.body.messages[message_index].validator_address` | String |
| Withdraw From Validator | `tx.body.messages[message_index].validator_address` | String |
| Withdraw To Address | `tx_response.logs[message_index].events[event_index].attributes[attribute_index].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "recipient"`.  | String |
| Withdraw Commission Amount | `tx_response.logs[message_index].events[event_index].attributes[m].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "amount"`.  | [Asset String](#asset-string)|


<a id="msg-fund-community-pool">

### 4. MsgFundCommunityPool

Fund from an account to the community pool. The community pool can later be sent to another by submitting a MsgSubmitEcProposal

- Funds movement: Yes

#### Protobuf

```go
type MsgFundCommunityPool struct {
	Amount    github_com_cosmos_cosmos_sdk_types.Coins `protobuf:"bytes,1,rep,name=amount,proto3,castrepeated=github.com/cosmos/cosmos-sdk/types.Coins" json:"amount"`
	Depositor string                                   `protobuf:"bytes,2,opt,name=depositor,proto3" json:"depositor,omitempty"`
}
```

#### Example

https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/7C1747E0189DCA88BBA55A1720809C8DF6075799C11ECBE4C4E1F89C91D4F55F

#### Details

This transaction will initiate a transfer from an account to the "Distribution" module account. Note that the "Distribution" module account is an internal account in Crypto.org Chain to hold the rewards, commissions and community pool funds before they are being distributed.

Ths "Distribution" module account is different on different chain. In Crypto.org Chain Mainnet, it is [cro1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8lyv94w](https://mainnet.crypto.org:1317/cosmos/auth/v1beta1/accounts/cro1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8lyv94w).

| Detail | How | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.distribution.v1beta1.MsgFundCommunityPool"` | String |
| Deposit From Account | `tx.body.messages[message_index].depositor` | String |
| Deposit Amount | `tx.body.messages[message_index].amount` | [Asset Array](#asset-array) |
| Delegate To Address ("Distribution" module account) | `tx_response.logs[message_index].events[event_index].attributes[attribute_index].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "recipient"`.  | String |