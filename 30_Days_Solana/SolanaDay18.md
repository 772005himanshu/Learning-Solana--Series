## Read Account data with Solana web3.js and Anchor

-> How to read the account data directly form the solana web3 javascript client so that a web app could read it on the frontend

-> we use the `solana account <account address>` to read the data , bu it couldn't work in building the dapp

-> We use this approach , calculate the address of the storage account, read the data, and deserialize the data from the Solana web3 client.

Using the previous rust Code

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
    #[account(mut,seeds = [], bump)]
    pub my_storage: Account<'info,MyStorage>,
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

    await program.methods.set(new anchor.BN(170)).accounts.({myStorage: myStorage}).rpc();

    /// Testing
    /// We are doing this in this tutorial but we already done it before for testing purpose
    const storageAccount = await program.account.myStorage.fetch(myStorage);
    console.log("TH evalue of x is:", storageAccount.x.toString());

    assert.equal(storageAccount.x.toString(),"170");
  });
});
```

### Fetching data from accounts created by Anchor Solana Programs

Lets make another program 

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("...");


#[program]
pub mod other_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn setbool(ctx: Context<SetFlag>, flag: bool) -> Result<()> {
        ctx.accounts.true_or_false.flag = flag;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    signer: Signer<'info>,

    system_program: Program<'info, System>,

    #[account(init, payer = signer, space = size_of::<TrueOrFalse>() + 8, seeds=[], bump)]
    true_or_false: Account<'info, TrueOrFalse>,
}

#[derive(Accounts)]
pub struct SetFlag<'info> {
    #[account(mut)]
    true_or_false: Account<'info, TrueOrFalse>, 
}

#[account]
pub struct TrueOrFalse {
    flag: bool,
}
```

Typescript Code

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { OtherProgram } from "../target/types/other_program";

describe("other_program", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.OtherProgram as Program<OtherProgram>;

  it("Is initialized!", async () => {
    const seeds = []
    const [TrueOrFalse, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    console.log("address: ", program.programId.toBase58());

    await program.methods.initialize().accounts({trueOrFalse: TrueOrFalse}).rpc();
    await program.methods.setbool(true).accounts({trueOrFalse: TrueOrFalse}).rpc();
  });
});
```

- `programId` that gets printed out after testing against local validator . We will need it to derive the address of `other_program`s account


## Read Program

`anchor init` another program. We call it read , only for read the `TrueOrFalse` struct of `other_program`, no Rust is Used.

- the `otherProgramAddress` matches the one from printedaove(declare_id!(...))
- ensure that you are reading the `other_program.json` IDL from the right file location
- make sure to run the tests with `--skip-local-validator` to ensure that this code reads the account the other program created

```typescript
import * as anchor from "@coral-xyz/anchor";

describe("read", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    it("Read other account", async () => {
        // Other program's programId -- make sure the address is correct
        const otherProgramAddress = "...";
        const otherProgramId = new anchor.web3.PublicKey(otherProgramAddress);

        // load other program idl - sure path is correct
        constt otherIdl = JSON.parse(
            require("fs").readFileSync("../other_program/target/idl/other_program.json","utf8")
        );

        const otherProgram = new anchor.Program(otherIdl, otherProgramId);

        const seeds = [];
        const [trueOrFalseAcc,_bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, otherProgramId);
        let otherStorageStruct  = await otherProgram.account.trueOrFalse.fetch(trueOrFalseAcc);

        console.log("The value of Flag is:" otherStorageStruct.flag.toString());

        console.log("The value of flag is: ", otherStorageStruct.flag.toString());
    });
})
```

-> This only work if the other Solana Program was Built with Anchor , This is relying on how Anchor serializes structs



## Fetching the data for an arbitary account

Your best bet for trying to find the Solana web3 Typescript function you need is to look at the `HTTP JSON RPC Methods` and look for one that seems promising. In our case, `getAccountInfo`

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


  it("Set !" , async() => {
    const seeds = []
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    console.log("the storage account address is", myStorage.toBase58());

    await program.methods.initialize().accounts({ myStorage: myStorage }).rpc();

    await program.methods.set(new anchor.BN(170)).accounts.({myStorage: myStorage}).rpc();

    /// Testing
    /// We are doing this in this tutorial but we already done it before for testing purpose
    const storageAccount = await program.account.myStorage.fetch(myStorage);
    console.log("TH evalue of x is:", storageAccount.x.toString());

    assert.equal(storageAccount.x.toString(),"170");

    let myStorageInfo = await anchor.getProvider().connection.getAccountInfo(myStorage);
    console.log(myStorageInfo);
  });
});
```