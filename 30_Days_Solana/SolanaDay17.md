## Solana - Reading and Writing data to accounts

From Previus day we are taking tha same code but adding new functionality , `set()` function to store number in MyStorage and associated Set Struct 

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("...");

#[program]
pub mod basic_storage {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn set(ctx: Context<Set>, new_x : u64) -> Result<()> {
        ctx.accounts.my_storage.x = new_x;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = signer,
        space = size_of::<MyStorage> + 8, 
        seeds = [], 
        bump
    )]
    pub my_storage: Account<'info,MyStorage>,

    #[account(mut)]
    pub signer: Signer<'info>, 
    pub system_program: Program<'info,System>,
}
// New Struct
#[derive(Accounts)]
pub struct Set<'info> {
    #[account(mut,seeds = [], bump)] /// NOTE - mut because we are changing the variable
    pub my_storage: Account<'info,MyStorage>, /// WE are passing the MyStorage as generic parameter to `Account`
}

#[account]
pub struct MyStorage {
    x: u64,
}
```


```typeScript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { BasicStorage } from "../target/types/basic_storage";
import {BN} from "@coral-xyz/anchor";

// For Testing it would be added or not 
import {assert} from "chai";



describe("basic_storage", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.BasicStorage as Program<BasicStorage>;


  it("Set !" , async() => {
    const seeds = []
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    console.log("the storage account address is", myStorage.toBase58());

    await program.methods.initialize().accounts({ myStorage: myStorage }).rpc();

    await program.methods.set(new BN(170)).accounts.({myStorage: myStorage}).rpc();

    /// Testing
    const storageAccount = await program.account.myStorage.fetch(myStorage);
    console.log("TH evalue of x is:", storageAccount.x.toString());

    assert.equal(storageAccount.x.toString(),"170");
  });
});
```


- Solana transaction must specify in advance which account it will access. The Struct for the `set()` function specifies it will be mutably(mut) accessing the `my_storage` account

- The `seeds = []` and `bump` are used to derive the address of the account we will be modifying.The user ia passing the account for us., But Anchor validates that the user is passing an account this program really owns by re-deriving the address and computing it to what the user provided.

- `bump` - checks that the account is not a cryptographically valid public key. This is how runtime knows this will be data storage for programs.

- Solana program could derive the address of the storage acount its own, the user still needs to provide the account `myStorage` , this is required by the Solana runtime 

Defining the new way `set()` function:

```rust
pub fn set(ctx: Context<Set>, new_x: u64) -> Result<()> {
    let my_storage = &mut ctx.accounts.my_storage;
    my_storage.x  = new_x;

    Ok(())
}
```

## Viewing our storage account from within the rust Program

```rust
pub fn print_x(ctx: Context<PrintX>) -> Result<()> {
    let x = ctx.accounts.my_storage.x;
    msg!("The value of x is : {}" ,x );
    Ok(())
}

#[derive(Accounts)]
pub struct PrintX<'info> {
    pub my_storage: Account<'info,MyStorage>,  // Now we donot use the #[account(mut)] because we are not writing it & just reading from the Storage
}
```


```typescript
await program.methods.printX().accounts({myStorage: myStorage}).rpc();
```

Exercise: Write an increment function that reads `x` and stores `x + 1` back in `x`.


```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("...");

#[program]
pub mod basic_storage {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn set(ctx: Context<Set>, new_x : u64) -> Result<()> {
        ctx.accounts.my_storage.x = new_x;
        Ok(())
    }

    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        let my_storage = &mut ctx.accounts.my_storage;
        my_storage.x = my_storage.x.checked_add(1);

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = signer,
        space = size_of::<MyStorage> + 8, 
        seeds = [], 
        bump
    )]
    pub my_storage: Account<'info,MyStorage>,

    #[account(mut)]
    pub signer: Signer<'info>, 
    pub system_program: Program<'info,System>,
}
// New Struct
#[derive(Accounts)]
pub struct Set<'info> {
    #[account(mut,seeds = [], bump)] 
    pub my_storage: Account<'info,MyStorage>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(mut,seeds = [], bump)] 
    pub my_storage: Account<'info,MyStorage>,
}

#[account]
pub struct MyStorage {
    x: u64,
}
```

Test

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { BasicStorage } from "../target/types/basic_storage";
import {BN} from "@coral-xyz/anchor";

// For Testing it would be added or not 
import {assert} from "chai";



describe("basic_storage", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.BasicStorage as Program<BasicStorage>;


  it("Increment !" , async() => {
    const seeds = []
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    console.log("the storage account address is", myStorage.toBase58());

    await program.methods.initialize().accounts({ myStorage: myStorage }).rpc();

    await program.methods.set(new BN(170)).accounts.({myStorage: myStorage}).rpc();

    /// Testing
    const storageAccount = await program.account.myStorage.fetch(myStorage);
    console.log("TH evalue of x is:", storageAccount.x.toString());

    assert.equal(storageAccount.x.toString(),"170");

    await program.methods.increment().accounts.({myStorage: myStorage}).rpc();

    /// Testing after increment
    const storageAccount2 = await program.account.myStorage.fetch(myStorage);
    console.log("TH evalue of x is:", storageAccount2.x.toString());

    assert.equal(storageAccount2.x.toString(),"171");
  });
});
```