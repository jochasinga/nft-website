---
title: Understanding Non-fungible Tokens on Solana 🚧
description: Learn the basic of NFTs on Solana and how to work with Metaplex
---

# Understanding NFTs on Solana

This guide will help you to learn the basic of tokens on Solana through building a simple token program on the Solana network and interact with it using the JavaScript client. It does not assume prior knowledge of Solana, but it does require some basic programming in [Rust](https://rustlang.org). Although it will briefly explain Solana, you are encourage to learn more by reading [How Solana Works](https://docs.solana.com/cluster/overview) and  [Programming on Solana - An Introduction
](https://paulx.dev/blog/2021/01/14/programming-on-solana-an-introduction/), and the [whitepaper](http://solana.com/solana-whitepaper.pdf).

## The basic of Solana

Solana is an innovative blockchain with some key innovations such as [Proof of History (POH)](https://medium.com/solana-labs/proof-of-history-a-clock-for-blockchain-cf47a61a9274) and parallel smart contracts run-time to achieve faster, near-instant consensus.

Solana smart contracts are called programs. On Solana, **everything is a program** all the way down. The System Program owns all SOL accounts and other programs, including the Token Program and user-space programs written by us developers.

The focus of this tutorial will be the Token Program in order to understand how to build an NFT program, but in the process it hopefully will give you a better understanding of the platform over all.

## Setting up

To begin this journey, first install:

- [Rust](https://www.rust-lang.org/tools/install)
- [Solana CLI](https://docs.solana.com/cli/install-solana-cli-tools)
- [Node.js](https://nodejs.org/en/) (Use one of the later LTS versions)

Run a test validator locally using the newly install CLI with

```shell
solana-test-validator
> Ledger location: test-ledger
> Log: test-ledger/validator.log
> Identity: 9U8Yk9fEq6hCvaDVtLSSFk5keJUpNutydW1f8LvfAHJf
> Genesis Hash: 5JLFXnecEaDHkkPZRbcN8LoGDPiz6QNLpKDLnB5JA6CF
> Version: 1.8.2
> Shred Version: 44427
> Gossip Address: 127.0.0.1:1024
> TPU Address: 127.0.0.1:1027
> JSON RPC URL: http://127.0.0.1:8899
> ⠦ 00:01:29 | Processed Slot: 181 | Confirmed Slot: 181 | Finalized Slot: 149 | S
```

Note the JSON RPC URL listening port. Target the running validator node with the Solana CLI:

```shell
solana config set --url http://127.0.0.1:8899
> Config File: /Users/pancy/.config/solana/cli/config.yml
> RPC URL: http://localhost:8899
> WebSocket URL: ws://localhost:8900/ (computed)
> Keypair Path: /Users/pancy/.config/solana/id.json
> Commitment: confirmed
```

Now create a Node project we will be using to interact with our on-chain token program by typing the following:

```shell
mkdir filet
cd filet
npm init -y
npm install @solana/web3.js
```

Let's trying to connect to the running Solana validator on port 8899 as a client using the Node REPL by typing the following:

```shell
node --experimental-repl-await
> Welcome to Node.js v14.18.1.
>> const { Connection } = await import("@solana/web3.js")
>> const connection = new Connection("http://localhost:8899", "confirmed")
>> const version = await connection.getVersion()
>> version["solana-core"]
> '1.8.2'
```

If the version was printed out on the prompt for you, you are successfully connected.

## Create an account

Next, are going to create a keypair that will serves as our "wallet"<sup>1</sup>. Still on the Node REPL, import `Keypair` class from `web3.js` module and create a new keypair:

```shell
>> const { Keypair } = await import("@solana/web3.js")
>> const address = keypair.publicKey.toString()
>> const secret = JSON.stringify(Array.from(keypair.secretKey))
>> address
> '6xnUqt2DXQjqN6ub3ZeT2gAqhtZzA7EMwztx5LhjmJB9'
>> secret
> '...'
```

Your address is a ED25519 publickey and is supposed to be shared in the open to receive tokens. The secret, however, should be kept in a secret place known only to you.

Take note of the address and the secret, as we will be using them for the rest of the tutorial.

## Fund the account

Let's get some (test) SOL airdropped to our account so we can start making transactions. On Solana localnet and devnet, you can request these airdrops for free since the fund can't be used on the mainnet. Make sure you instantiated a `Connection` previously, then proceed to request 1 SOL or 1,000,000,000 Lamports (the value of the constant `LAMPORTS_PER_SOL`) and check the account's balance at the end:

```shell
>> const { PublicKey, LAMPORTS_PER_SOL } = await import("@solana/web3.js")
>> const publicKey = new PublicKey(address)
>> const txhash = await connection.requestAirdrop(publicKey, LAMPORTS_PER_SOL)
>> await connection.confirmTransaction(txhash)
>> const balance = await connection.getBalance(publicKey)
>> balance
> '1000000000'
```

Now let's practice transferring some Lamports to another address. To do that, we will have to create a new keypair with a new address to receive the fund. Can you figure this out? (Hint: Repeat [creating an account](#create-an-account))

After you have created a new account and take note of the address and secret, save the address to `recipientAddress` and use the `SystemProgram` and `Transaction` classes to construct a transaction for the transfer:

```shell
>> const recipientAddress = 'NEW_ACCOUNT_ADDRESS'
>> const { SystemProgram, Transaction, sendAndConfirmTransaction } = await import("@solana/web3.js")

>> const fromPubkey = new PublicKey(address)
>> const toPubkey = new PublicKey(recipientAddress)
>> const secretKey = Uint8Array.from(JSON.parse(secret as string))
>> const instructions = SystemProgram.transfer({
    fromPubkey,
    toPubkey,
    lamports,
})
>> const signers = [
  {
    publicKey: fromPubkey,
    secretKey,
  },
]
>> const transaction = new Transaction().add(instructions)
>> const hash = await sendAndConfirmTransaction(connection, transaction, signers)
```

## Creating an NFT program

Start a project with `cargo new filet --lib`. Then open the project's manifest file `filet/Cargo.toml` and add:

```toml
[features]
no-entrypoint = []

[dependencies]
solana-program = "=1.10.2"

[dev-dependencies]
solana-sdk = "=1.10.2"

[lib]
crate-type = ["cdylib", "lib"]
```

Pay special attention to the `[features]` and `[lib]` attributes. Run `cargo update` within the project directory to update the dependencies.

Use an editor to open your project and navigate to the `/src/lib.rs`, remove all the existing code, and add the following to the file:

```rust
use solana_program::{
  msg, entrypoint, entrypoint::ProgramResult,
};

entrypoint!(process_instruction);
fn process_instruction(
  program_id: &Pubkey,
  accounts: &[AccountInfo],
  instruction_data: &[u8],
) -> ProgramResult {
  todo!("Not yet know what I'm doing");
}
```