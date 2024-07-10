# Taco Parallel Zk Rollup Specification

This document specifies Taco, a parallel zk rollup designed to enhance scalability and security. It serves as a specification for future implementations of Taco.

## Table of Contents

-   [Intro](#intro)
-   [Architecture](#architecture)
    -   [RPC Server](#rpc-server)
    -   [Executors](#executors)
    -   [Accounts and Locks](#accounts-and-locks)
    -   [Transactions](#transactions)
    -   [Proof Manager, Generators and Mergers](#proof-manager-generators-and-mergers)
    -   [Smart Contract](#smart-contract)
    -   [Minting and Faucets](#minting-and-faucets)
    -   [Bridges](#bridges)
    -   [Wallets](#wallets)
-   [Notes](#notes)

## Intro

Taco is a zk rollup specialized for ultimate scalability.
It processes transactions and generates zk proofs in parallel.

## Architecture

Taco consists of several components such as an RPC Server, executors, accounts storage, account locks, transactions storage, proof manager, proof generators, proof mergers and a smart contract on the layer 1 network.

![](/assets/architecture.png)

### RPC Server

Taco's RPC server complies with the [JSON-RPC 2.0 specification](https://www.jsonrpc.org/specification) and includes the following methods.

-   ### `taco_getBalance`:

    Retrieves the balance of the specified account.

    ### Params

    -   `address`: The account address as a string.

    ### Result

    -   `balance`: The account balance as a 64-bit unsigned integer.

    ### Example Request

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "taco_getBalance",
        "params": ["0x..."]
    }
    ```

    ### Example Response

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "result": {
            "balance": 2100000000000
        }
    }
    ```

-   ### `taco_getNonce`:

    Retrieves the nonce of the specified account.

    ### Params

    -   `address`: The account address as a string.

    ### Result

    -   `nonce`: The account nonce as a 64-bit unsigned integer.

    ### Example Request

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "taco_getNonce",
        "params": ["0x..."]
    }
    ```

    ### Example Response

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "result": {
            "nonce": 17
        }
    }
    ```

-   ### `taco_transfer`:

    Transfers the specified amount from the sender to the receiver.

    ### Params

    -   `sender`: The sender's address as a string.

    -   `receiver`: The receiver's address as a string.

    -   `amount`: The amount to be transferred in this transaction as a 64-bit unsigned integer.

    -   `nonce`: The nonce of the sender as a 64-bit unsigned integer.

    -   `signature`: The sender's signature for this transaction as a string.

    ### Result

    -   `hash`: The transaction hash as a string.

    ### Example Request

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "taco_transfer",
        "params": ["0x...", "0x...", 9800000000, 17, "0x..."]
    }
    ```

    ### Example Response

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "result": {
            "hash": "0x..."
        }
    }
    ```

-   ### `taco_getTransaction`:

    Retrieves the details of the specified transaction.

    ### Params

    -   `hash`: The transaction hash as a string.

    ### Result

    -   `sender`: The sender's address as a string.

    -   `receiver`: The receiver's address as a string.

    -   `amount`: The amount to be transferred in this transaction as a 64-bit unsigned integer.

    -   `timestamp`: The transaction timestamp as a 64-bit unsigned integer.

    -   `status`: The transaction status as a string from the following values: `"received"`, `"proved"` and `"submitted"`.

    ### Example Request

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "taco_getTransaction",
        "params": ["0x..."]
    }
    ```

    ### Example Response

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "result": {
            "sender": "0x...",
            "receiver": "0x...",
            "amount": 123456789,
            "timestamp": 1777777777777,
            "status": "proved"
        }
    }
    ```

-   ### `taco_getHistory`:

    Retrieves all the transaction history of the specified account.

    ### Params

    -   `address`: The account address as a string.

    ### Result

    -   `transactions`: The transactions as an array of transactions whose definition can be found at [`taco_getTransaction`](#taco_gettransaction).

    ### Example Request

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "taco_getHistory",
        "params": ["0x..."]
    }
    ```

    ### Example Response

    ```json
    {
        "jsonrpc": "2.0",
        "id": 1,
        "result": {
            "transactions": [
                {
                    "sender": "0x...",
                    "receiver": "0x...",
                    "amount": 123456789,
                    "timestamp": 1777777777777,
                    "status": "submitted"
                },
                {
                    "sender": "0x...",
                    "receiver": "0x...",
                    "amount": 234567891,
                    "timestamp": 1888888888888,
                    "status": "submitted"
                },
                {
                    "sender": "0x...",
                    "receiver": "0x...",
                    "amount": 345678912,
                    "timestamp": 1999999999999,
                    "status": "submitted"
                }
            ]
        }
    }
    ```

### Executors

Taco's executors handle transaction execution and state updates within Taco.

Each executor executes transactions by verifying the provided signature of the message consisting of sender, receiver, amount, and nonce. The nonce must be greater than that of the last processed transaction to maintain the sequence of transactions and prevent the reuse of compromised signatures. If everything is okay, the executor runs the transaction logic and updates the state accordingly.

When an executor receives a transaction to execute, it first checks if the related accounts are locked. If the accounts are not locked, the executor proceeds to lock them and begins executing the transaction. Simultaneously, other executors can handle transactions involving different accounts, enabling parallel execution. Once the execution is complete, executors update the state and unlock the accounts.

**Non-Blocking Transactions**

![](/assets/non-blocking.png)

However, if the accounts are already locked, the executor waits for them to become unlocked, which is a blocking operation. Meanwhile, other executors can continue executing transactions involving different accounts in parallel. Once the accounts become unlocked, the previously blocked executor can proceed to execute the transaction.

**Blocking Transactions**

![](/assets/blocking.png)

### Accounts and Locks

In Taco, every address corresponds to an account containing the following data: `balance` and `nonce`.

These accounts can be stored in an optimized in-memory database for efficient access.

Account locks prevent concurrent transactions from modifying the same account simultaneously, ensuring that only one transaction can operate on a specific account at any given time.

This mechanism enables parallel execution of multiple transactions, as long as they involve different accounts that are not currently locked.

### Transactions

Taco supports a single type of transaction: transfers.

Each transfer transaction includes the following data: `sender`, `receiver`, `amount`, `nonce`, `signature`. Additional metadata, such as timestamps, is also stored alongside transactions, but these are not included in zk proofs.

### Proof Manager, Generators and Mergers

Taco generates zk proofs for each executed transaction through its proof manager.

The proof manager reads executed transactions starting from the first one and delegates the proof generation tasks to other computers for enhanced speed. Once two sequential proofs are generated, they can be merged into a single proof. This merging process continues until Taco merges all proofs into a single proof.

**Generating and Merging Proofs**

![](/assets/proving.png)

Once Taco ends up with a single proof representing a specified amount of transactions and/or a specific interval, it submits that proof to the layer 1 network.

**Submitting Proofs**

![](/assets/submittion.png)

To reduce the time required for generating proofs for a large number (N) of transactions from O(N) to nearly O(log N), Taco can use a technique similar to [SIMD (Single Instruction, Multiple Data)](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data) instructions used in CPUs. This approach allows performing the same operation on a vector of items simultaneously.

Taco can have proof generator functions such as `transferX1`, `transferX2`, `transferX4`, `transferX8`, `transferX16`, etc., as many as the complexity of the zk circuit allows. By processing transactions in batches, Taco achieves faster proof generation.
**SIMD-Like Functions For Proof Generation**

![](/assets/functions.png)

### Smart Contract

Taco's smart contract ensures the state of Taco is regularly checkpointed on the layer 1 network by verifying merged proofs.

The smart contract stores the hash of the rollup's state and includes a `submitProof` function. This function takes a merged proof as a parameter, verifies it and ensures that the contract's state is equal to the public input of that proof. It then updates the smart contract state using that proof's public output, which represents the hash of Taco's state.

To represent the whole state as a single hash, Taco uses a data structure like [Merkle Trees](https://en.wikipedia.org/wiki/Merkle_tree).
It also makes it possible to ensure data integrity inside proof generator functions.

### Minting and Faucets

The minting feature can be implemented in Taco to allow assets to be transferred from the zero address to the receiver's address by requiring a signature from an authorized address.

This feature can also be used to build a faucet.

### Bridges

A trusted bridge can be implemented by listening to deposit events on the layer 1 network and minting the corresponding amount on Taco using an authorized account's signature. To move assets back to the layer 1 network, users can burn their assets on Taco by sending them to the zero address. Once the zk proof containing this transaction is submitted to the layer 1 network, users can then claim their assets on the layer 1 network.

However, if the layer 1 network also utilizes zk technology, similar to how [Mina](https://minaprotocol.com) does, a cryptographically secure bridge could potentially be built by integrating Taco with it.

### Wallets

Taco have some RPC methods that allow building a fully functioning wallet for Taco.

An account's balance can be retrieved using [`taco_getBalance`](#taco_getbalance) RPC method.

An account's nonce can be retrieved using [`taco_getNonce`](#taco_getnonce) RPC method.

An account's transaction history using [`taco_getHistory`](#taco_gethistory) RPC method.

Transactions can be sent using [`taco_transfer`](#taco_transfer) RPC method.

## Notes

Taco is a showcase specification and many optimizations such as parallel processing have no impact in real life performance, because transfer transactions are simple enough for a single executor to execute quickly.

> The author is [Berzan](https://x.com/berzanorg).
