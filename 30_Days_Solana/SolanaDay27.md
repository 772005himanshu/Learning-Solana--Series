## init_if_needed in Anchor and the Reinitialization Attack

We had to initialize an account in the seperate transaction before we write data to it. 
if we want to write the data and initialize the account in the same transaction.
Anchor provides a macro called `init_if_needed` , as name sugeest , initialize the acount if doesn't exist

```rust
use anchor_lang::prelude::*;
use std::mem:;size_of;

declare_id!("...");

#[program]
pub mod init_if_needed {
    use super::*;

    pub fn increament(ctx: Context<Initialize>) -> Result<()> {
        let current_counter = ctx.accounts.my_pda.counter;
        ctx.accounts.my_pda.counter = current_counter + 1;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init_if_needed,
        payer = signer,
        space = size_of::<MyPDA>() + 8,
        seeds = [],
        bump
    )]
    pub my_pda: Account<'info,MyPDA>,

    #[account(mut)]
    pub signer : Signer<'info>,
    pub system_program: program<'info,System>,
}

#[account]
pub struct MyPDA {
    pub counter : u64,
}
```


Testing with typescript

```typescript
import * as anchor from "@coral-xyz/anchor";
import {Program} from "@coral-xyz/anchor";
import {InitIfNeeded} from "../target/types/init_if_needed";

decribe("init_if_needed", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const program = anchor.workspace.InitIfNeeded as Program<InitIfNeeded>;

    it("Is Initialized!" , async () => {
        const seeds = []
        const [] = anchor.web3.PublicKey.findProgramAddressSync(seeds,program.programId);
        await program.methods.increment().accounts({myPda: myPda}).rpc();
        await program.methods.increment().accounts({myPda: myPda}).rpc();
        await program.methods.increment().accounts({myPda: myPda}).rpc();

        let result = await program.account.myPda.fetch(myPda);
        console.log(`counter is ${result.counter}`);
    })
})


```

if we try anchor build we get the error , because we donot add `init_if_needed` in the `Cargo.toml` file

```
[dependencies]
anchor-lang = {version = "0.29.0", features = [init_if_needed]}
```

#### In Anchor programs , accounts cannot be initialized twice (by default) - If we try to initialize an account that has already been initialized, the transaction will fail

## How does anchor know an account is already initialized??

From Anchor’s perspective, if the account has a non-zero lamport balance OR the account is owned by the system program, then it is not initialized, So we can initiaized them again

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;
use anchor_lang::system_program;

declare_id!("...");

#[program]
pub mod reinit_attack {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn drain_lamports(ctx: Context<DrainLamports>) -> Result<()> {
        let lamports = ctx.accounts.my_pda.to_account_info().lamports();
        ctx.accounts.my_pda.sub_lamports(lamports)?;
				ctx.accounts.signer.add_lamports(lamports)?;
        Ok(())
    }

    pub fn give_to_system_program(ctx: Context<GiveToSystemProgram>) -> Result<()> {
        let account_info = &mut ctx.accounts.my_pda.to_account_info();
        // the assign method changes the owner
		account_info.assign(&system_program::ID);
        account_info.realloc(0, false)?;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct DrainLamports<'info> {
    #[account(mut)]
    pub my_pda: Account<'info, MyPDA>,
    #[account(mut)]
    pub signer: Signer<'info>,
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8, seeds = [], bump)]
    pub my_pda: Account<'info, MyPDA>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct GiveToSystemProgram<'info> {
    #[account(mut)]
    pub my_pda: Account<'info, MyPDA>,
}

#[account]
pub struct MyPDA {}


```


```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ReinitAttack } from "../target/types/reinit_attack";

describe("Program", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.ReinitAttack as Program<ReinitAttack>;

  it("initialize after giving to system program or draining lamports", async () => {
    const [myPda, _bump] = anchor.web3.PublicKey.findProgramAddressSync([], program.programId);

    await program.methods.initialize().accounts({myPda: myPda}).rpc();

    await program.methods.giveToSystemProgram().accounts({myPda: myPda}).rpc();

    await program.methods.initialize().accounts({myPda: myPda}).rpc();
    console.log("account initialized after giving to system program!")

    await program.methods.drainLamports().accounts({myPda: myPda}).rpc();

    await program.methods.initialize().accounts({myPda: myPda}).rpc();
    console.log("account initialized after draining lamports!")
  });
});


```

- Again, Solana does not have an “initialized” flag or anything. Anchor will allow an initialize transaction to succeed if the owner is the system program or the lamport balance is zero.

## Why reinitialization might be a problem in our example

- Transferring ownership to the system program requires erasing the data in the account. Removing all the lamports “communicates” that you don’t want the account to continue to exist

- If you are okay with the counter getting reset back at some point and counting back up, reinitialization is not an issue. But if the counter should never reset to zero under any circumstance, then it would probably be better for you to implement the initialization function separately and add a safeguard to make sure it can only be called once in it’s lifetime (for example, storing a boolean flag in a separate account)

- your program might not necessarily have the mechanism to transfer the account to the system program or withdraw lamports from the account. But Anchor has no way of knowing this, so it always throws out the warning about init_if_needed because it cannot determine whether the account can go back into an initializable state

## Having two initialization paths could lead to an off-by-one error or other surprising behavior

In our counter example with init_if_needed, the counter is never equal to zero because the first initialization transaction also increments the value from zero to one

If we also had a regular initialization function that did not increment the counter, then the counter would be initialized and have a value of zero. If some business logic never expects to see a counter with a value of zero, then unexpected behavior may happen

## “Initialization” does not always mean “init” in Anchor