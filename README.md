# Metacat Protocol specification (from 0x)

## Table of contents

1.  [Architecture](#architecture)
1.  [Contracts](#contracts)
    1.  [Exchange](#exchange)
    1.  [AssetProxy](#assetproxy)
    1.  [ZeroExGovernor](#zeroexgovernor)
    1.  [Staking](#staking)
1.  [Contract Interactions](#contract-interactions)
    1.  [Trade settlement](#trade-settlement)
    1.  [Protocol fees](#protocol-fees)
1.  [Orders](#orders)
    1.  [Message format](#order-message-format)
    1.  [Hashing an order](#hashing-an-order)
    1.  [Creating an order](#creating-an-order)
    1.  [Filling orders](#filling-orders)
    1.  [Matching orders](#matching-orders)
    1.  [Cancelling orders](#cancelling-orders)
    1.  [Querying state of an order](#querying-state-of-an-order)
    1.  [Simulating fill transfers](#simulating-fill-transfers)
1.  [Transactions](#transactions)
    1.  [Message format](#transaction-message-format)
    1.  [Hashing a transaction](#hashing-a-transaction)
    1.  [Creating a transaction](#creating-a-transaction)
    1.  [Executing a transaction](#executing-a-transaction)
    1.  [Transaction context](#transaction-context)
1.  [Signatures](#signatures)
    1.  [Validating signatures](#validating-signatures)
    1.  [Signature types](#signature-types)
1.  [Events](#events)
    1.  [Exchange events](#exchange-events)
    1.  [AssetProxy events](#assetproxy-events)
    1.  [ZeroExGovernor events](#zeroexgovernor-events)
    1.  [Staking events](#staking-events)
1.  [Types](#types)
1.  [Rich Reverts](#rich-reverts)
1.  [Miscellaneous](#miscellaneous)
    1.  [EIP-712 usage](#eip-712-usage)
    1.  [EIP-1271 Usage](#eip-1271-usage)
    1.  [ECRECOVER usage](#ecrecover-usage)
    1.  [Rounding errors](#rounding-errors)
    1.  [Differences from 2.0](#differences-from-20)

# Architecture

Metacat protocol uses an approach we refer to as **off-chain order relay with on-chain settlement**. In this approach, cryptographically signed [orders](#orders) are broadcast off of the blockchain through any arbitrary communication channel; an interested counterparty may inject one or more of these [orders](#orders) into 0x protocol's [`Exchange`](#exchange) contract to execute and settle trades directly to the blockchain.

Metacat uses a modular system of Ethereum smart contracts which allows each component of the system to be upgraded via governance without affecting other components of the system and without causing active markets to be disrupted.

# Contracts

## Exchange

The `Exchange` contract contains the bulk of the business logic within 0x protocol. It is the entry point for:

1.  Filling [orders](#orders)
2.  Canceling [orders](#orders)
3.  Executing [transactions](#transactions)
4.  Validating [signatures](#signatures)
5.  Registering new [`AssetProxy`](#assetproxy) contracts into the system
6.  Paying protocol fees to the [`Staking`](#staking) contract
7.  Setting protocol fee specific parameters

## AssetProxy

Doing any sort of division in the EVM may result in rounding errors. [`fillOrder`](#fillorder) and [`matchOrder`](#matchorder) variants limit the allowed rounding error to 0.1% (and will otherwise revert). Note that the rounding is _always_ done in favor of an order's maker.

## Differences from 2.0

### Changes to order fills

- The introduction of [protocol fees](#protocol-fees)
- Order fees can now be charges in arbitrary assets with the new [`makerFeeAssetData` and `takerFeeAssetData` fields](#order-message-format)
- `marketBuyOrders` and `marketSellOrders` have been deprecated
- The addition of [`marketBuyOrdersFillOrKill`](#marketbuyordersfillorkill) and [`marketSellOrdersFillOrKill`](#marketsellordersfillorkill)
- All of the `marketBuy*` and `marketSell*` functions now allow multiple different assets to be bought or sold (be careful with this!)
- The ordering of transfers during a single fill has changed. A fill now first transfers an asset from the taker to the maker, opening up the ability to integrate the [`ERC20BridgeProxy`](https://github.com/0xProject/ZEIPs/issues/47) (along with other similar contracts)

### Changes to order matching

- The addition of a new matching strategy with [`matchOrdersWithMaximalFill`](#matchorderswithmaximalfill)
- The addition of new functions for matching batches of orders, [`batchMatchOrders`](#batchmatchorders) and [`batchMatchOrdersWithMaximalFill`](#batchmatchorderswithmaximalfill)

### Changes to order cancellations

- The [`cancelOrder`](#cancelorder) function is much less strict and will only revert if called with an invalid context

### Changes to 0x transactions

- The addition of new [`expirationTimeSeconds` and `gasPrice` fields](#transaction-message-format) to 0x transactions
- [`executeTransaction`](#executetransaction) now takes a [`ZeroExTransaction`](#zeroextransaction) struct as an input
- [`executeTransaction`](#executetransaction) now emits a [`TransactionExecution`](#transactionexecution) event upon successful execution
- Addition of [`batchExecuteTransaction`](#batchexecutetransactions) for atomically executing multiple 0x transactions

### Changes to signature validation

- The [`Validator`](#validator) signature type now expects the verifying contract to be [`EIP-1271`](<#[EIP-1271](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1271.md)>) compliant, allowing it to validate individual fields of orders or transactions
- The addition of a new [`EIP1271Wallet`](#eip1271wallet) signature type that allows a wallet contract to validate individual fields of orders or transactions

### Changes to developer experience

- The introduction of [rich reverts](#rich-reverts)
- The addition of a helper function for [simulating actual transfers](#simulating-fill-transfers)
- `getOrdersInfo` has been deprecated (an external contract can be deployed for querying the state of multiple orders in a single call)

### Miscellaneous changes

- All non-administrative functions that can write to state have been made `payable` to allow paying protocol fees in ETH
- The [EIP-712](#eip-712-usage) domain hash calculation now includes a `chainId`

