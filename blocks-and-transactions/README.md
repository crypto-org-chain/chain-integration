# Crypto.org Chain Blocks and Transactions

This document describes the block and transaction structure of Crypto.org Chain and explain different ways to extract and parse the details of them.

- Based on Crypto.org Chain [v1.2.1](https://github.com/crypto-org-chain/chain-main/releases/tag/v1.2.1) release. 
- Last updated at: 2021-04-20

## Table of Content

- [Changelog](#changelog)
- [Common APIs](#common-apis)
- [Common Block Details](#common-block-details)
  - [Mint](#mint)
  - [Block Rewards](#block-rewards)
  - [Proposer Rewards](#proposer-rewards)
  - [Commissions](#commission)
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
- [Staking](#stkaing)
  - [MsgCreateValidator](#msg-create-validator)
  - [MsgEditValidator](#msg-edit-validator)
  - [MsgDelegate](#msg-delegate)
  - [MsgBeginRedelegate](#msg-begin-redelegate)
  - [MsgUndelegate](#msg-undelegate)
- [Slashing](#slashing)
  - [MsgUnjail](#msg-unjail)

## Changelog

- 2021-04-16 First Draft
- 2021-04-16 Add Distribution module
- 2021-04-16 Add Mint, Block Rewards, Proposer Rewards and Commissions
- 2021-04-20 Add Staking

## Common APIs

<a id="tendermint-block-api">

#### 1. Tendermint Block API

- URL: https://mainnet.crypto.org:26657/block?height=[height]
- This API returns block details, list of transaction bytes and consensus commits.
- **Example**: https://mainnet.crypto.org:26657/block?height=10000

<a id="tendermint-block-results-api">

#### 2. Tendermint Block Results API

- URL: https://mainnet.crypto.org:26657/block_results?height=[height]
- This API returns the events of the block. These events include the outcomes from transactions, and block changes such as block rewards minted (`"mint"`) and distributed as well as consensus state updates such as validator missing block counts (`"liveness"`)
- **Example**: https://mainnet.crypto.org:26657/block_results?height=10000

<a id="cosmos-transaction-query-api">

#### 3. Cosmos Transaction Query API

- https://mainnet.crypto.org:1317/comsos/tx/v1beta1/txs/[hash]
- This API returns the parsed transaction details and events of a particular transaction hash
- **Example**: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/0C5E617B0577B047D78EBF5313B8B70DF69E9535E17B964303BD04947B11B660

<a id="cosmos-transaction-search-api">

#### 4. Cosmos Transaction Search API

- URL: https://mainnet.crypto.org:1317/comsos/tx/v1beta1/txs
- This API support event based query and returns parsed transactions. Common events includes

| event                           | Description                                                                                                                                            | Example                                                                                                                                                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| tx.height                       | Transaction(s) in a particular block                                                                                                                   | [txs?events=tx.height%3D10000](https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs?events=tx.height%3D10000)                                                                                              |
| message.module & message.action | Search for messages belonged to a particular module and actions. _Note that this index will degrade when more transaction of its kind grows in number_ | [txs?events=message.module%3D%27bank%27&events=message.action%3D%27send%27](https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs?events=message.module%3D%27bank%27&events=message.action%3D%27send%27)    |
| message.sender                  | Search for message with particular signer                                                                                                              | [/txs?events=message.sender=%27cro18undzhe3fmmav2x3csx8m00m5yupkcc7qzz4ec%27](https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs?events=message.sender=%27cro18undzhe3fmmav2x3csx8m00m5yupkcc7qzz4ec%27) |

**Note**:

1. The API supports pagination, make sure you have iterate all the pages `pagination.offset=[offset starting from 0]&pagination.limit=[record per page]`
2. The performance will degrade if you are searching for a result set that will grow over time. For example if we search for events that grow with the block height, such as validator's reward distrubtion.
3. Multiple events in a single search is queried by `AND` condition. i.e If you do `tx.height` and `message.sender`. It will search for transactions happened on that particular block height **AND** signed by the sender.

[Top](#table-of-content)

## Common Block Details

Most of the block events can be accessed using the [Tendermint Block Results API](#tendermint-block-results-api). One caveat of using this API is that all the events key-value attributes are bas64 encoded. So it is not human readable.

A simple Node.js tool has been written to help parse and decode all key-value attributes in the block results API response. It can be downloaded at [https://github.com/calvinaco/cosmos-api-tools](https://github.com/calvinaco/cosmos-api-tools).

Usage example:

```bash
$ git clone https://github.com/calvinaco/cosmos-api-tools
$ cd cosmos-api-tools
$ node block-results-decoder.js "https://mainnet.crypto.org:26657/block_results?height=10000"
```

Note that when you integrate with the API you should still base64 decode the attributes programmatically.

<a id="mint" />

### 1. Mint

In every block, CRO is minted and offered to the active validators as block rewards. The actual minted token is subject to inflation and is adjusted every block. Further details on the miniting parameters and configuration can be found [here](https://crypto.org/docs/chain-details/module_overview.html#mint)

Minted tokens are distributed as block and proposer rewards in the same block. However, since Cosmos SDK do the [lazy rewards calculation and collection](https://docs.cosmos.network/master/modules/distribution/), the minted tokens are first sent to "Distribution" module account and is later transferred to an account when a deleagtor withdraws the rewards or commissions by sending a [MsgWithdrawDelegatorReward](#msg-withdraw-delegator-reward) or [MsgWithdrawValidatorCommission](#msg-withdraw-validator-commission).

So [Block Rewards](#block-rewards), [Proposer Rewards](#proposer-rewards) and [Commissions](#commissions) events are for record keeping only and does not represent any actual token transfer between accounts.

To get the minted token every block:

| Accessor                                                                                                                                                                                                                                                                             | Type                          |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------- |
| `Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].value)`<br />where<br />`Base64Decode(result.begin_block_events[event_index].type) === "mint" && Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].key) === "amount"` | [Asset String](#asset-string) |

<a id="block-rewards" />

### 2. Block Rewards

In every block, mint tokens and transaction fees are distributed to every active validator in the network. As a result, there will be multiple events, each corresponding to a validator and the rewards it receives.

Block rewards are **not** credited to the delegator account directly. This event serve as record keeping purpose only. Each delegator account must explicitly send a [MsgWithdrawDelegatorReward](#msg-withdraw-delegator-reward) message transaction to collect the rewards.

To get the reward **per validator**:

| Detail            | Accessor                                                                                                                                                                                                                                                                                   | Type                          |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------- |
| Validator Address | `Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].value)`<br />where<br />`Base64Decode(result.begin_block_events[event_index].type) === "rewards" && Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].key) === "validator"` | [Asset String](#asset-string) |
| Reward Amount     | `Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].value)`<br />where<br />`Base64Decode(result.begin_block_events[event_index].type) === "rewards" && Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].key) === "amount"`    | String                        |

<a id="proposer-rewards" />

### 3. Proposer Rewards

Block proposer can get extra transaction fees bonus for the block they have successfully proposed. More detial can be found [here](https://crypto.org/docs/chain-details/module_overview.html#transaction-fees-bonus) for reference.

Similar to block rewards, proposer rewards are **not** credited to the account directly. This event serve as record keeping purpose only. Each validator creator account must explicitly send a [MsgWithdrawDelegatorReward](#msg-withdraw-delegator-reward) message transaction to collect the rewards.

| Detail            | Accessor                                                                                                                                                                                                                                                                                           | Type                          |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| Validator Address | `Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].value)`<br />where<br />`Base64Decode(result.begin_block_events[event_index].type) === "proposer_reward" && Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].key) === "validator"` | [Asset String](#asset-string) |
| Reward Amount     | `Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].value)`<br />where<br />`Base64Decode(result.begin_block_events[event_index].type) === "proposer_reward" && Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].key) === "amount"`    | String                        |

<a id="commissions" />

### 4. Commissions

Validator can charge commission to the block rewards received by delegators. Commissions is already included in [Block Rewards](#block-rewards) and [Proposer Rewards](#proposer-rewards).

Similar to block rewards, commission rewards are **not** credited to the account directly. This event serve as record keeping purpose only. Each validator creator account must explicitly send a [MsgWithdrawValidatorCommission](#msg-withdraw-validator-commission) message transaction to collect the rewards.

To get the commission received by **each validator**:

| Detail            | Accessor                                                                                                                                                                                                                                                                                   | Type                          |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------- |
| Validator Address | `Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].value)`<br />where<br />`Base64Decode(result.begin_block_events[event_index].type) === "rewards" && Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].key) === "validator"` | [Asset String](#asset-string) |
| Commission Amount | `Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].value)`<br />where<br />`Base64Decode(result.begin_block_events[event_index].type) === "rewards" && Base64Decode(result.begin_block_events[event_index].attributes[attribute_index].key) === "amount"`    | String                        |

[Top](#table-of-content)

## Common Transaction Details

<a id="block-height" />

### 1. Block Height

- [Tendermint Block API](#tendermint-block-api): `result.block.header.height`
- [Tendermint Block Results API](#tendermint-block-results-api): `result.height`
- [Cosmos Transaction Query API](#cosmos-transaction-query-api): `tx_response.height`
- [Cosmos Transaction Search API](#cosmos-transaction-query-api): `tx_response[index].height`

<a id="transaction-hash" />

### 2. Transaction Hash

- [Tendermint Block API](#tendermint-block-api): `Uppercase(SHA256(Base64Decode(result.block.data.txs[index])))`
- [Tendermint Block Results API](#tendermint-block-results-api): Not available, should use [Tendermint Block API](#tendermint-block-api). Match transaction `[index]` in `result.txs_results` with `result.block.data.txs[index]`
- [Cosmos Transaction Query API](#cosmos-transaction-query-api): `tx_response.txhash`
- [Cosmos Transaction Search API](#cosmos-transaction-query-api): `tx_response[index].txhash`

<a id="transaction-fee" />

### 3. Transaction Fee

- [Cosmos Transaction Query API](#cosmos-transaction-query-api): `tx.auth_info.fee.amount`
- [Cosmos Transaction Search API](#cosmos-transaction-query-api): `tx[index].auth_info.fee.amount`

Transaction fee is an [Asset Array](#asset-array), meaning that a transaction can pay fee in more than one token types.

<a id="assets-and-amount" />

### 4. Assets and Amount

There are mainly two types of assets and amount representation:

<a id="asset-object" />

#### 1. Single object

This is commonly seen in `staking` module but may appear in other module as well. It represents a single token type.

**Example**

```json
{
  "denom": "basecro",
  "amount": "10000"
}
```

where `denom` is the asset type and `amount` is the amount and `basecro` is the basic unit of CRO token (10^8 `basecro` = 1 cro)

Note that the `amount` is always in string for precision accuracy. Please make sure your language is capable to handle big integer number.It is highly recommended to use library similar to [bignumber.js](https://mikemcl.github.io/bignumber.js/) in your language to handle the `amount`.

##### Total supply:

The total supply of CRO is _30 billion_ (`30,000,000,000`). `1 CRO=1_0000_0000 basecro`. So in basic unit `basecro` it can be up to `3,000,000,000,000,000,000`.

<a id="asset-array" />

#### 2. Array

This is commonly seen in most message types. It represents a list of tokens.

At the time of writing there will only be a single entry in this array because `basecro` (or `basetcro` in Croeseid Testnet) is the only supported asset on Crypto.org Chain. However, after IBC transfer and other coins issuance methods are enabled, there will be more asset types, the coin source tracing and their denomination can be found in [here](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-001-coin-source-tracing.md)

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

[Top](#table-of-content)

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

| Detail           | Accessor                                                                      | Type                        |
| ---------------- | ----------------------------------------------------------------------------- | --------------------------- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.bank.v1beta1.MsgSend"` | String                      |
| From address     | `tx.body.messages[message_index].from_address`                                | String                      |
| To address       | `tx.body.messages[message_index].to_address`                                  | String                      |
| Amount           | `tx.body.messages[message_index].amount`                                      | [Asset Array](#asset-array) |

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

| Detail           | Accessor                                                                                                                                                 | Type                        |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.bank.v1beta1.MsgMultiSend"`                                                                       | String                      |
| From Addresses   | `tx.body.messages[message_index].inputs[m].address` where `m>=1`. There can be multiple (`m`) from addresses.                                            | String                      |
| From Amounts     | `tx.body.messages[message_index].inputs[m].coins` where `m>=1`. There can be multiple (`m`) from addresses and their corresponding input amount.         | [Asset Array](#asset-array) |
| To Addresses     | `tx.body.messages[message_index].outputs[n].address` where `n>=1`. There can be multiple (`n`) destination addresses.                                    | String                      |
| To Amounts       | `tx.body.messages[message_index].outputs[n].coins` where `n>=1`. There can be multiple (`n`) destination addresses and their corresponding input amount. | [Asset Array](#asset-array) |

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

| Detail                  | Accessor                                                                                                                                                                                                                                                                                    | Type                          |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| Transaction Type        | `tx.body.messages[message_index]["@type"] === "/cosmos.distribution.v1beta1.MsgWithdrawDelegatorReward"`                                                                                                                                                                                    | String                        |
| Delegator               | `tx.body.messages[message_index].delegator_address`                                                                                                                                                                                                                                         | String                        |
| Withdraw From Validator | `tx.body.messages[message_index].validator_address`                                                                                                                                                                                                                                         | String                        |
| Withdraw To Address     | `tx_response.logs[message_index].events[event_index].attributes[attribute_index].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "recipient"`. | String                        |
| Withdraw Reward Amount  | `tx_response.logs[message_index].events[event_index].attributes[attribute_index].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "amount"`.    | [Asset String](#asset-string) |

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

| Detail                     | Accessor                                                                                                                                                                                                                                                                                    | Type                          |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| Transaction Type           | `tx.body.messages[message_index]["@type"] === "/cosmos.distribution.v1beta1.MsgWithdrawValidatorCommission"`                                                                                                                                                                                | String                        |
| Validator                  | `tx.body.messages[message_index].validator_address`                                                                                                                                                                                                                                         | String                        |
| Withdraw From Validator    | `tx.body.messages[message_index].validator_address`                                                                                                                                                                                                                                         | String                        |
| Withdraw To Address        | `tx_response.logs[message_index].events[event_index].attributes[attribute_index].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "recipient"`. | String                        |
| Withdraw Commission Amount | `tx_response.logs[message_index].events[event_index].attributes[m].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "amount"`.                  | [Asset String](#asset-string) |

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

| Detail                                              | Accessor                                                                                                                                                                                                                                                                                    | Type                        |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- |
| Transaction Type                                    | `tx.body.messages[message_index]["@type"] === "/cosmos.distribution.v1beta1.MsgFundCommunityPool"`                                                                                                                                                                                          | String                      |
| Deposit From Account                                | `tx.body.messages[message_index].depositor`                                                                                                                                                                                                                                                 | String                      |
| Deposit Amount                                      | `tx.body.messages[message_index].amount`                                                                                                                                                                                                                                                    | [Asset Array](#asset-array) |
| Delegate To Address ("Distribution" module account) | `tx_response.logs[message_index].events[event_index].attributes[attribute_index].value` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "recipient"`. | String                      |

[Top](#table-of-content)

## Staking

<a id="msg-create-validator">

### 1. MsgCreateValidator

Create a new validator

Funds movement: Yes

#### Protobuf

```go
type MsgCreateValidator struct {
	Description       Description                            `protobuf:"bytes,1,opt,name=description,proto3" json:"description"`
	Commission        CommissionRates                        `protobuf:"bytes,2,opt,name=commission,proto3" json:"commission"`
	MinSelfDelegation github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=min_self_delegation,json=minSelfDelegation,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"min_self_delegation" yaml:"min_self_delegation"`
	DelegatorAddress  string                                 `protobuf:"bytes,4,opt,name=delegator_address,json=delegatorAddress,proto3" json:"delegator_address,omitempty" yaml:"delegator_address"`
	ValidatorAddress  string                                 `protobuf:"bytes,5,opt,name=validator_address,json=validatorAddress,proto3" json:"validator_address,omitempty" yaml:"validator_address"`
	Pubkey            *types.Any                             `protobuf:"bytes,6,opt,name=pubkey,proto3" json:"pubkey,omitempty"`
	Value             types1.Coin                            `protobuf:"bytes,7,opt,name=value,proto3" json:"value"`
}

type Description struct {
	// moniker defines a human-readable name for the validator.
	Moniker string `protobuf:"bytes,1,opt,name=moniker,proto3" json:"moniker,omitempty"`
	// identity defines an optional identity signature (ex. UPort or Keybase).
	Identity string `protobuf:"bytes,2,opt,name=identity,proto3" json:"identity,omitempty"`
	// website defines an optional website link.
	Website string `protobuf:"bytes,3,opt,name=website,proto3" json:"website,omitempty"`
	// security_contact defines an optional email for security contact.
	SecurityContact string `protobuf:"bytes,4,opt,name=security_contact,json=securityContact,proto3" json:"security_contact,omitempty" yaml:"security_contact"`
	// details define other optional details.
	Details string `protobuf:"bytes,5,opt,name=details,proto3" json:"details,omitempty"`
}

type CommissionRates struct {
	// rate is the commission rate charged to delegators, as a fraction.
	Rate github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,1,opt,name=rate,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"rate"`
	// max_rate defines the maximum commission rate which validator can ever charge, as a fraction.
	MaxRate github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,2,opt,name=max_rate,json=maxRate,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"max_rate" yaml:"max_rate"`
	// max_change_rate defines the maximum daily increase of the validator commission, as a fraction.
	MaxChangeRate github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,3,opt,name=max_change_rate,json=maxChangeRate,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"max_change_rate" yaml:"max_change_rate"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/7B3C19A3674C9EF856C43FFF50B021085AC4DA693AA47F82882FFAC78F21DE05

#### Details

| Detail | Accessor | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.staking.v1beta1.MsgCreateValidator"` | String |
| Initial Delegator Address (Validator Creator) | `tx.body.messages[message_index].delegator_address` | String |
| Delegated Amount | `tx.body.messages[message_index].value | [Asset Object](#asset-object) |

[Top](#table-of-content)

<a id="msg-edit-validator">

### 2. MsgEditValidator

Edit and existing validator

Funds Movement: No

#### Protobuf

```go
type MsgEditValidator struct {
	Description      Description `protobuf:"bytes,1,opt,name=description,proto3" json:"description"`
	ValidatorAddress string      `protobuf:"bytes,2,opt,name=validator_address,json=validatorAddress,proto3" json:"validator_address,omitempty" yaml:"address"`
	// We pass a reference to the new commission rate and min self delegation as
	// it's not mandatory to update. If not updated, the deserialized rate will be
	// zero with no way to distinguish if an update was intended.
	// REF: #2373
	CommissionRate    *github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,3,opt,name=commission_rate,json=commissionRate,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"commission_rate,omitempty" yaml:"commission_rate"`
	MinSelfDelegation *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,4,opt,name=min_self_delegation,json=minSelfDelegation,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"min_self_delegation,omitempty" yaml:"min_self_delegation"`
}

type Description struct {
	// moniker defines a human-readable name for the validator.
	Moniker string `protobuf:"bytes,1,opt,name=moniker,proto3" json:"moniker,omitempty"`
	// identity defines an optional identity signature (ex. UPort or Keybase).
	Identity string `protobuf:"bytes,2,opt,name=identity,proto3" json:"identity,omitempty"`
	// website defines an optional website link.
	Website string `protobuf:"bytes,3,opt,name=website,proto3" json:"website,omitempty"`
	// security_contact defines an optional email for security contact.
	SecurityContact string `protobuf:"bytes,4,opt,name=security_contact,json=securityContact,proto3" json:"security_contact,omitempty" yaml:"security_contact"`
	// details define other optional details.
	Details string `protobuf:"bytes,5,opt,name=details,proto3" json:"details,omitempty"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/F4A1D7757AD20979D540C0CD29DD335D17E121F15AA447990B87E0EE94531BD7

#### Details

TODO

[Top](#table-of-content)


<a id="msg-delegate">

### 3. MsgDelegate

Perform a delegation of coins from a delegator to a validator

Funds movement: Yes

#### Protobuf

```go
type MsgDelegate struct {
	DelegatorAddress string      `protobuf:"bytes,1,opt,name=delegator_address,json=delegatorAddress,proto3" json:"delegator_address,omitempty" yaml:"delegator_address"`
	ValidatorAddress string      `protobuf:"bytes,2,opt,name=validator_address,json=validatorAddress,proto3" json:"validator_address,omitempty" yaml:"validator_address"`
	Amount           types1.Coin `protobuf:"bytes,3,opt,name=amount,proto3" json:"amount"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/CCB45B0C6EC18A327ADFC8C36478A163D8C2A8BD9EB13687F73ED3D4559318A3

#### Details

| Detail | Accessor | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.staking.v1beta1.MsgDelegate"` | String |
| Delegate From Address | `tx.body.messages[message_index].delegator_address` | String |
| Delegate To Validator | `tx.body.messages[message_index].validator_address` | String |
| Delegate Amount | `tx.body.messages[message_index].amount` | [Asset Object](#asset-object) |

[Top](#table-of-content)

<a id="msg-begin-redelegate">

### 4. MsgBeginRedelegate

Perform a redelegation of coins from a delegator and source validator to a destination validator.

Note that the redelegation is just a record update of the internal state of a delegator' staked funds. The delegator account won't have any funds movement based on the redelegation except.

There is a side effect of `MsgBeginRedelegate`, upon successful execution of this message, all the rewards of the delegator from previous (source) validator will be withdrew automatically to the delegator account.

Funds movement: Yes

#### Protobuf

```go
type MsgBeginRedelegate struct {
	DelegatorAddress    string      `protobuf:"bytes,1,opt,name=delegator_address,json=delegatorAddress,proto3" json:"delegator_address,omitempty" yaml:"delegator_address"`
	ValidatorSrcAddress string      `protobuf:"bytes,2,opt,name=validator_src_address,json=validatorSrcAddress,proto3" json:"validator_src_address,omitempty" yaml:"validator_src_address"`
	ValidatorDstAddress string      `protobuf:"bytes,3,opt,name=validator_dst_address,json=validatorDstAddress,proto3" json:"validator_dst_address,omitempty" yaml:"validator_dst_address"`
	Amount              types1.Coin `protobuf:"bytes,4,opt,name=amount,proto3" json:"amount"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/5D43A55463C8FB30A89306C26C5E3318826AD075D36E9B5E72F7019C00F14549

#### Details

| Detail | Accessor | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.staking.v1beta1.MsgBeginRedelegate"` | String |
| Redelegate From Address | `tx.body.messages[message_index].delegator_address` | String |
| Redelegate From Validator | `tx.body.messages[message_index].validator_src_address` | String |
| Redelegate To Validator | `tx.body.messages[message_index].validator_dst_address` | String |
| Redelegate Amount | `tx.body.messages[message_index].amount` | [Asset Object](#asset-object) |
| Auto Withdraw Rewards To Address # | `tx_response.logs[message_index].events` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "recipient"`.  | String |
| Auto Withdraw Rewards From Address (Always the "distribution" module account) # | `tx_response.logs[message_index].events` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "sender"`.  | String |
| Auto Withdraw Rewards Amount # | `tx_response.logs[message_index].events` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "amount"`.  | String |

\# Note: there may be multiple auto rewards withdrawal happen. In such a case the `transfer` event will have the multiple `{"recipient":"","sender":"","amount":""}`. An example is
```
{
  "type": "transfer",
  "attributes": [
    {
      "key": "recipient",
      "value": "cro1hr6wx9mm6ycjr4x2p4ek4c9fl2prkxsx79y5qu"
    },
    {
      "key": "sender",
      "value": "cro1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8lyv94w"
    },
    {
      "key": "amount",
      "value": "30763basecro"
    },
    {
      "key": "recipient",
      "value": "cro1hr6wx9mm6ycjr4x2p4ek4c9fl2prkxsx79y5qu"
    },
    {
      "key": "sender",
      "value": "cro1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8lyv94w"
    },
    {
      "key": "amount",
      "value": "6881basecro"
    }
  ]
}
```

[Top](#table-of-content)

<a id="msg-undelegate">

### 5. MsgUndelegate

Perform an undelegation from a delegate and a validator.

Funds movement: Yes

#### Protobuf

```go
type MsgUndelegate struct {
	DelegatorAddress string      `protobuf:"bytes,1,opt,name=delegator_address,json=delegatorAddress,proto3" json:"delegator_address,omitempty" yaml:"delegator_address"`
	ValidatorAddress string      `protobuf:"bytes,2,opt,name=validator_address,json=validatorAddress,proto3" json:"validator_address,omitempty" yaml:"validator_address"`
	Amount           types1.Coin `protobuf:"bytes,3,opt,name=amount,proto3" json:"amount"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/3B36AA1AC81ACD58E7A06C21353DB0FC40A70EDBF6BD2CD23D7BEDC7A0F56318

#### Details

| Detail | Accessor | Type |
| --- | --- | --- |
| Transaction Type | `tx.body.messages[message_index]["@type"] === "/cosmos.distribution.v1beta1.MsgFundCommunityPool"` | String |
| Undelegate From Address | `tx.body.messages[message_index].delegator_address` | String |
| Undelegate From Validator | `tx.body.messages[message_index].validator_src_address` | String |
| Undeleate Amount | `tx.body.messages[message_index].amount` | [Asset Object](#asset-object) |
| Auto Withdraw Rewards To Address # | `tx_response.logs[message_index].events` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "recipient"`.  | String |
| Auto Withdraw Rewards From Address (Always the "distribution" module account) # | `tx_response.logs[message_index].events` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "sender"`.  | String |
| Auto Withdraw Rewards Amount # | `tx_response.logs[message_index].events` <br />where <br />`tx_response.logs[message_index].events[event_index].type === "transfer" && tx_response.logs[message_index].events[event_index].attributes[attribute_index].key === "amount"`.  | String |

\# Note: Similar to MsgBeginRedelegate, there may be multiple auto rewards withdrawal happen. In such a case the `transfer` event will have the multiple `{"recipient":"","sender":"","amount":""}`. An example is

[Top](#table-of-content)

## Slashing

<a id="msg-unjail">

### 1. MsgUnjail

Unjail a validator

Funds movement: No

#### Protobuf

```go
type MsgUnjail struct {
	ValidatorAddr string `protobuf:"bytes,1,opt,name=validator_addr,json=validatorAddr,proto3" json:"address" yaml:"address"`
}
```

#### Example

Cosmos Transaction Query API: https://mainnet.crypto.org:1317/cosmos/tx/v1beta1/txs/58BF8EBD17FF9500F395E4A9B2AE93EF21306E5706B3EC31CE116654D78B8684

#### Details

TODO

[Top](#table-of-content)
