## PDA (Program Derived Address) vs Keypair Account in Solana

A program derived address (PDA) is an account whose address is derived from the address of the program that created it and the `seeds` passed in to the `init` transaction.Up until this point we only used PDAs

It is also possible to create an account outside the program , then `init` that account inside the program

The account we create outside the program will have a private key, but we will see this won't have the security implications it would seem to have. We will refers to it as a "Keypair Account"

## Account Creation Revisited

The program we used so far are PDA(Program Derived Address)

```rust
use anchor_lang::prelude::*;
use std::mem::size_of; 

declare_id!("...");

#[program]
pub mod keypair_vs_pda {
    use super::*;

    pub fn initialize_pda(ctx: Context<InitializePDA>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializePDA<'info> {

    // This is the program derived address
    #[account(init,
              payer = signer,
              space=size_of::<MyPDA>() + 8,
              seeds = [],
              bump)]
    pub my_pda: Account<'info, MyPDA>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[account]
pub struct MyPDA {
    x: u64,
}

```

## The Testing Typescript
```typescript
import * as anchor from "@coral-xyz/anchor";
import {Program} from "@coral-xyz/anchor";
import {KeypairVsPda} from "../target/types/keypair_vs_pda";

decribe("keypair_vs_pda", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const program = anchor.workspace.KeypairVsPda as Program<KeypairVsPda>;

    it("is initialized -- PDA version", async() => {
        const seeds = []
        const [myPda,_bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

        console.log("the storage account address is" , myPda.toBase58());

        const tx = await program.methods.initilizePda().accounts({myPda: myPda}).rpc();
    });
});

```

## Program Derived Address

- An accounts is program Derived Address(PDA) if the address of the accounts is derived from the address of the program , i.e `programId` in `findProgramAddressSync(seeds,program.programId)`.It is also function of seeds
- We know if is a PDA because teh `seeds` and `bump` are present in the 'init' macro 

## Keypair Account
We remove the `seeds` and `bump` form the accounts

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("...");

#[program]
pub mod keypair_vs_pda {
    use super::*;

    pub fn initialize_keypair_account(ctx: Context<InitializeKeypairAccount>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeKeypairAccount<'info> {
    // This is the program derived address
    #[account(init,
              payer = signer,
              space = size_of::<MyKeypairAccount>() + 8,)]  // Here we say we remove the seeds and bump 
    pub my_keypair_account: Account<'info, MyKeypairAccount>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[account]
pub struct MyKeypairAccount {
    x: u64,
}
```

When we `seed` and `bump` are absent, the Anchor Program is now expecting us to create an account first , then pass teh account to the program . Since we create the account ourselves, its address will not be `derived from` the address of the program.in other words,it will not the PDA

Creating the account for the program is as simple as generating a new keypair . 
We hold the secret key for an account the program is using to store data

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { KeypairVsPda } from "../target/types/keypair_vs_pda";

// this airdrops sol to an address
async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount * anchor.web3.LAMPORTS_PER_SOL);
  await confirmTransaction(airdropTx);
}

async function confirmTransaction(tx) {
  const latestBlockHash = await anchor.getProvider().connection.getLatestBlockhash();
  await anchor.getProvider().connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: tx,
  });
}

describe("keypair_vs_pda", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.KeypairVsPda as Program<KeypairVsPda>;

  it("Is initialized -- keypair version", async () => {
		
    const newKeypair = anchor.web3.Keypair.generate();
    await airdropSol(newKeypair.publicKey, 1e9); // 1 SOL

    console.log("the keypair account address is", newKeypair.publicKey.toBase58());

    await program.methods.initializeKeypairAccount()
      .accounts({myKeypairAccount: newKeypair.publicKey})
      .signers([newKeypair]) // the signer must be the keypair
      .rpc();
  });
});

```

## It is not possible to initialize a keypair account don't holds the priavte key - throw the error : unkown signer like

## What if we tried to fake teh PDA's address?

```typescript
const seeds = []
const [pda, _bump] = anchor
                        .web3
                        .PublicKey
                        .findProgramAddressSync(
                            seeds,
                            program.programId);
```

This also throw the error like : unknown signer error

## Is it an issue that someone has an account for the private key?

The person iin possesion of the private key would able to spend SOL out of the account ,and possibly bring it below the rent-exempt threshold. Solana runtime prevents this from happening when the accounts is initialized by a program 

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { KeypairVsPda } from "../target/types/keypair_vs_pda";

// Change this to your path
import privateKey from '/Users/himanshu/.config/solana/id.json';

import { fs } from fs;

async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount * anchor.web3.LAMPORTS_PER_SOL);
  await confirmTransaction(airdropTx);
}

async function confirmTransaction(tx) {
  const latestBlockHash = await anchor.getProvider().connection.getLatestBlockhash();
  await anchor.getProvider().connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: tx,
  });
}


describe("keypair_vs_pda", () => {
  const deployer = anchor.web3.Keypair.fromSecretKey(Uint8Array.from(privateKey));

  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.KeypairVsPda as Program<KeypairVsPda>;

  it("Writing to keypair account fails", async () => {
    const newKeypair = anchor.web3.Keypair.generate();
    var recieverWallet = anchor.web3.Keypair.generate();

    await airdropSol(newKeypair.publicKey, 10);

    var transaction = new anchor.web3.Transaction().add(
      anchor.web3.SystemProgram.transfer({
        fromPubkey: newKeypair.publicKey,
        toPubkey: recieverWallet.publicKey,
        lamports: 1 * anchor.web3.LAMPORTS_PER_SOL,
      }),
    );
    await anchor.web3.sendAndConfirmTransaction(anchor.getProvider().connection, transaction, [newKeypair]);
    console.log('sent 1 lamport') 

    await program.methods.initializeKeypairAccount()
      .accounts({myKeypairAccount: newKeypair.publicKey})
      .signers([newKeypair]) // the signer must be the keypair
      .rpc();
      
    console.log("initialized");
    // try to transfer again, this fails
    var transaction = new anchor.web3.Transaction().add(
      anchor.web3.SystemProgram.transfer({
        fromPubkey: newKeypair.publicKey,
        toPubkey: recieverWallet.publicKey,
        lamports: 1 * anchor.web3.LAMPORTS_PER_SOL,
      }),
    );
    await anchor.web3.sendAndConfirmTransaction(anchor.getProvider().connection, transaction, [newKeypair]);
  });
});

```

Even though we hold the private keys to this account, we cannot “spend SOL” from the account as it is now owned by the program

## Introduction to ownership and initialization
How does the solana runtime kow to block the transfer of Solana after initialization ?

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { KeypairVsPda } from "../target/types/keypair_vs_pda";

import privateKey from '/Users/himanshu/.config/solana/id.json';


async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount * anchor.web3.LAMPORTS_PER_SOL);
  await confirmTransaction(airdropTx);
}

async function confirmTransaction(tx) {
  const latestBlockHash = await anchor.getProvider().connection.getLatestBlockhash();
  await anchor.getProvider().connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: tx,
  });
}


describe("keypair_vs_pda", () => {
  const deployer = anchor.web3.Keypair.fromSecretKey(Uint8Array.from(privateKey));

  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.KeypairVsPda as Program<KeypairVsPda>;
  it("Console log account owner", async () => {

    console.log(`The program address is ${program.programId}`) 
    const newKeypair = anchor.web3.Keypair.generate();
    var recieverWallet = anchor.web3.Keypair.generate();

    // get account owner before initialization
    await airdropSol(newKeypair.publicKey, 10);
    const accountInfoBefore = await anchor.getProvider().connection.getAccountInfo(newKeypair.publicKey);
    console.log(`initial keypair account owner is ${accountInfoBefore.owner}`);

    await program.methods.initializeKeypairAccount()
      .accounts({myKeypairAccount: newKeypair.publicKey})
      .signers([newKeypair]) // the signer must be the keypair
      .rpc();
      
    // get account owner after initialization
	
    const accountInfoAfter = await anchor.getProvider().connection.getAccountInfo(newKeypair.publicKey);
    console.log(`initial keypair account owner is ${accountInfoAfter.owner}`);
  });
});


```
- After initialization, the owner of the keypair account changed from `111...111` to the deployed program

- This should give you an idea of what “initialization” is doing and why the owner of the private key can no longer transfer SOL out of the account.