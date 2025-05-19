## Modifying accounts using different signers

In this we will initializing an account with one wallet and updating it with another

### The initialization Step

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("...");

#[program]
pub mod other_write {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Account)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = signer,
        space = size_of::<MyStorage>() + 8,
        seeds = [],
        bump
    )]
    pub  my_storage: Account<'info,MyStorage>,

    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info,System>,
}

#[account]
pub struct MyStorage {
    x: u64,
}
```


## Doing the initialization transaction with an alternate wallet
- For testing purpose , we create a wallett called `newKeypair`.This is different froom one Anchor provides by default . We airdrop that new wallet 1 SOL so it can pay for transaction.
- We are passing the publicKey of that wallet for the `Signer` field. When we use the default signer built into Anchor, Anchor passs this in the background for us. 

```typescript
import * as anchor from "@coral-xyz/anchor";
import {Program} from "@coral-xyz/anchor";
import {OtheWrite} from "../target/types/other_write";

async function airdropSol(publicKey, amount) {
    let airdroptx = await anchor.getProvider().connection.requestAirdrop(publicKey,amount);
    await confirmTransaction(airdropTx);
}

async function confirmTransaction(tx) {
    const latestBlockHash = await anchor.getProvider().connection.getLatestBlockhash();
    await anchor.getProvider().connection.confrimTransaction({
        blockhash: latestBlockhash.blockhash,
        lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
        signature: tx,
    })
}

describe("other-write", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const program = anchor.workspace.otherWrite as Program<OtherWrite>;

    it("Is initialized!" , async () => {
        const newKeypair = anchor.web3.Keypair.generate();
    await airdropSol(newKeypair.publicKey, 1e9); // 1 SOL

    let seeds = [];
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    await program.methods.initialize().accounts({
        myStorage: myStorage,
        signer: newKeypair.publicKey // ** THIS MUST BE EXPLICITLY SPECIFIED **
        }).signers([newKeypair]).rpc();
    });
})
```

Exercise: In the Rust code, change payer = signer to payer = fren and pub signer: Signer<'info>, to pub fren: Signer<'info>, and change signer: newKeypair.publicKey to fren: newKeypair.publicKey in the test. The initialization should succeed and the test should pass.



## Why does the Anchor require specifying the Signer asn the publicKey?

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("...");

#[program]
pub mod other_write {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Account)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = fren,
        space = size_of::<MyStorage>() + 8,
        seeds = [],
        bump
    )]
    pub  my_storage: Account<'info,MyStorage>,

    #[account(mut)]
    pub fren: Signer<'info>, // A public key is passed here 
    pub system_program: Program<'info,System>,
}

#[account]
pub struct MyStorage {
    x: u64,
}
```

In upper code , we see the `fren` specified to be a `Signer Account`.The `Signer` type means Anchor will look at the signature of the transaction and make sure the signature matches the address passed here 

## Error : unknown signer in Solana Anchor

The `unknown signer ` error occurs when the signer of the transaction does not match the public key passed to `Signer`.

- we modify the test to remove the .signers([newKeypair]) spec. Anchor will use the default signer instead, and the default signer will not match the publicKey of our newKeypair wallet

We will get the error like - `Signature verification failed`

Similarly, if we don’t pass in the publicKey explicitly, Anchor will silently use the default signer:

We will get error : `Error : unknown signer`


- The following code will result in a successful initialization because both the Signer public key and the account that signs the transaction are the Anchor default signer.

```typescript
  await program.methods.initialize().accounts({
    myStorage: myStorage
  }).rpc();
});

// Anchor by default initialize the Anchor account in background
```

## Bob can write to an account Alice initialized

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("61As9Y8pREgvFZzps6rpFai8UkageeHT6kW1dnGRiefb");

#[program]
pub mod other_write {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn update_value(ctx: Context<UpdateValue>, new_value: u64) -> Result<()> {
        ctx.accounts.my_storage.x = new_value;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init,
              payer = fren,
              space=size_of::<MyStorage>() + 8,
              seeds = [],
              bump)]
    pub my_storage: Account<'info, MyStorage>,

    #[account(mut)]
    pub fren: Signer<'info>, // A public key is passed here

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct UpdateValue<'info> {
    #[account(mut, seeds = [], bump)]
    pub my_storage: Account<'info, MyStorage>,

    // THIS FIELD MUST BE INCLUDED
    #[account(mut)]
    pub fren: Signer<'info>,
}

#[account]
pub struct MyStorage {
    x: u64,
}

```

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { OtherWrite } from "../target/types/other_write";


async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount);
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

describe("other_write", () => {

  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.OtherWrite as Program<OtherWrite>;

  it("Is initialized!", async () => {
    const alice = anchor.web3.Keypair.generate();
    const bob = anchor.web3.Keypair.generate();

    const airdrop_alice_tx = await anchor.getProvider().connection.requestAirdrop(alice.publicKey, 1 * anchor.web3.LAMPORTS_PER_SOL);
    await confirmTransaction(airdrop_alice_tx);

    const airdrop_alice_bob = await anchor.getProvider().connection.requestAirdrop(bob.publicKey, 1 * anchor.web3.LAMPORTS_PER_SOL);
    await confirmTransaction(airdrop_alice_bob);

    let seeds = [];
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    
    await program.methods.initialize().accounts({
      myStorage: myStorage,
      fren: alice.publicKey
    }).signers([alice]).rpc();


    await program.methods.updateValue(new anchor.BN(3)).accounts({
      myStorage: myStorage,
      fren: bob.publicKey
    }).signers([bob]).rpc();

    let value = await program.account.myStorage.fetch(myStorage);
    console.log(`value stored is ${value.x}`);
  });
});


```

## Restricting writes to Solana Accounts
In real life application ,we donot want Bob writing aribtrary data to arbitrary accounts.

## Lets do this With Example - Building a proto-ERC20 program

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("HFmGQX4wPgPYVMFe4WrBi925NKvGySrEG2LGyRXsXJ4Z");

const STARTING_POINTS: u32 = 10;

#[program]
pub mod points {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        ctx.accounts.player.points = STARTING_POINTS;
        ctx.accounts.player.authority = ctx.accounts.signer.key();
        Ok(())
    }

    pub fn transfer_points(ctx: Context<TransferPoints>,
                           amount: u32) -> Result<()> {
        require!(ctx.accounts.from.authority == ctx.accounts.signer.key(), Errors::SignerIsNotAuthority);
        require!(ctx.accounts.from.points >= amount, Errors::InsufficientPoints);

        ctx.accounts.from.points -= amount;
        ctx.accounts.to.points += amount;
        Ok(())
    }
}

#[error_code]
pub enum Errors {
    #[msg("SignerIsNotAuthority")]
    SignerIsNotAuthority,
    #[msg("InsufficientPoints")]
    InsufficientPoints
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init,
              payer = signer,
              space = size_of::<Player>() + 8,
              seeds = [&(signer.as_ref().key().to_bytes())],
              bump)]
    player: Account<'info, Player>,
    #[account(mut)]
    signer: Signer<'info>,
    system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct TransferPoints<'info> {
    #[account(mut)]
    from: Account<'info, Player>,
    #[account(mut)]
    to: Account<'info, Player>,
    #[account(mut)]
    signer: Signer<'info>,
}

#[account]
pub struct Player {
    points: u32,
    authority: Pubkey
}
```

- We use the address of the signer `(&(signer.as_ref().key().to_bytes()))`to derive the address of the account where their points are stored. This behaves like a Solidity mapping in Solana, where the Solana `“msg.sender / tx.origin”` is the key.


```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Points } from "../target/types/points";


async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount);
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

describe("points", () => {
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.Points as Program<Points>;


  it("Alice transfers points to Bob", async () => {
    const alice = anchor.web3.Keypair.generate();
    const bob = anchor.web3.Keypair.generate();

    const airdrop_alice_tx = await anchor.getProvider().connection.requestAirdrop(alice.publicKey, 1 * anchor.web3.LAMPORTS_PER_SOL);
    await confirmTransaction(airdrop_alice_tx);

    const airdrop_alice_bob = await anchor.getProvider().connection.requestAirdrop(bob.publicKey, 1 * anchor.web3.LAMPORTS_PER_SOL);
    await confirmTransaction(airdrop_alice_bob);

    let seeds_alice = [alice.publicKey.toBytes()];
    const [playerAlice, _bumpA] = anchor.web3.PublicKey.findProgramAddressSync(seeds_alice, program.programId);

    let seeds_bob = [bob.publicKey.toBytes()];
    const [playerBob, _bumpB] = anchor.web3.PublicKey.findProgramAddressSync(seeds_bob, program.programId);

    await program.methods.initialize().accounts({
      player: playerAlice,
      signer: alice.publicKey,
    }).signers([alice]).rpc();

    await program.methods.initialize().accounts({
      player: playerBob,
      signer: bob.publicKey,
    }).signers([bob]).rpc();

    await program.methods.transferPoints(5).accounts({
      from: playerAlice,
      to: playerBob,
      signer: alice.publicKey,
    }).signers([alice]).rpc();

    console.log(`Alice has ${(await program.account.player.fetch(playerAlice)).points} points`);
    console.log(`Bob has ${(await program.account.player.fetch(playerBob)).points} points`)
  });
});


```

Exercise: Create a keypair `mallory` and try to get `mallory` to steal points from Alice or Bob by using mallory as the signer in `.signers([mallory])`. Your attack should fail, but you should try anyway.

## Using the Anchor Constraints to replace require! macros

### Anchor `has_one` constraint

The `has_one` constraint assumes that there is 'shared key' between `#[derive(Accounts)]` and `#[account]` , checks that both of those keys have te same value. The best way to demonstrate this is with a picture

```rust
#[derive(Accounts)]
#[instruction(amount: u32)]
pub struct TransferPoints<'info> {
    #[account(mut, has_one = authority)]
    from : Account<'info, Player>,
    #[account(mut)]
    to: Account<'info.Player>,
    auhtority: Signer<'info>, // these two must match
}

#[account]
pub struct Player {
    points: u32,
    authority: Pubkey  // These two must match 
}
```

### Anchor `constraint` constraint
- Replace the macro `require!(ctx.accounts.from.points >= amount, Errors::InsufficientPoints);` with an Anchor constraint


```rust
#[derive(Accounts)]
#[instruction(amount: u32)] // amount must be passed as an instruction
pub struct TransferPoints<'info> {
    #[account(mut,
              has_one = authority,
              constraint = from.points >= amount)]
    from: Account<'info, Player>,
    #[account(mut)]
    to: Account<'info, Player>,
    authority: Signer<'info>,
}

#[account]
pub struct Player {
    points: u32,
    authority: Pubkey
}

```


### Adding custom error messages to Anchor constraints

- Readability of error messages when constraints are violated by adding custom errors, the same custom errors we passed to the `require!` macros using the `@` notation


```rust
#[derive(Accounts)]
#[instruction(amount: u32)]
pub struct TransferPoints<'info> {
    #[account(mut,
              has_one = authority @ Errors::SignerIsNotAuthority,
              constraint = from.points >= amount @ Errors::InsufficientPoints)]
    from: Account<'info, Player>,
    #[account(mut)]
    to: Account<'info, Player>,
    authority: Signer<'info>,
}

#[account]
pub struct Player {
    points: u32,
    authority: Pubkey
}

```