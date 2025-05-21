## Understanding Account Ownership in Solana: Transferring SOL out of a PDA

The owner of an account in Solana is able to reduce the SOL balance , write data to the account , change the owner.


1. The system program owns wallets and keypair accounts that haven't assigned ownership to a program (initialized).
2. The BPFLoader owns program
3. A program owns Solana PDAs. It can also own keypair accounts if ownership has been transferred to the program (this is what happens during initialization).

## The system program owns keypair accounts 

- When CLI run this command -. solana account <CONTRACT_ADDRESS>

That shows the - Public Key , Balance , Owner , Executable , Rent Epoch

The owner is not your address , buther an account address is `1111...111` is the system program, the smae system program that moves SOL 

## Only the owner of an account has the ability to modify the data in it.

Who this works - The reason you are able to spend the SOL , you possess the private key generated address or public key , when the system program recognizes that you have produced a valid signature for the public key, then it will recognize your request to spend the lamports in the account as legitimate , then spend them according to your instruction

## PDAs and keypair accounts initialized by the programs are owned by the program

A program owns Solana PDAs. It can also own keypair accounts if ownership has been transferred to the program (this is what happens during initialization).

###  initializing an account changes the owner of the account from the system program the program

- If we try to ascertain the owner of an address that does not exist, we get a `null`.


```rust 
use anchor_lang::prelude::*;

declare_id!("...")

#[program]
pub mod owner {
    use super::*;

    pub fn initialize_keypair(ctx: Context<InititalizeKeypair>) -> Result<()> {
        Ok(())
    }

    pub fn initialize_pda(ctx: Context<InitializePda>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeKeypair<'info> {
    #[account(init, payer = signer , space = 8)]
    keypair: Account<'info, Keypair>,
    #[account(mut)]
    signer : Signer<'info>,
    system_program: Program<'info,System>,
}

#[deriive(Accounts)]
pub struct InitializePda<'info> {
    #[account(init, payer = signer , space = 8 , seeds = [] , bump)]
    pda: Account<'info,Pda>,
    #[account(mut)]
    signer: Signer<'info>,
    system_program : Program<'info, System>,
}

#[account]
pub struct Keypair() {

}

#[account]
pub struct Pda() {

}
```

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Owner } from "../target/types/owner";

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

describe("owner", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Owner as Program<Owner>;

  it("Is initialized!", async () => {
    console.log("program address", program.programId.toBase58());    
    const seeds = []
    const [pda, bump_] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    console.log("owner of pda before initialize:",
    await anchor.getProvider().connection.getAccountInfo(pda));

    await program.methods.initializePda().accounts({pda: pda}).rpc();

    console.log("owner of pda after initialize:",
    (await anchor.getProvider().connection.getAccountInfo(pda)).owner.toBase58());

    let keypair = anchor.web3.Keypair.generate();

    console.log("owner of keypair before airdrop:",
    await anchor.getProvider().connection.getAccountInfo(keypair.publicKey));

    await airdropSol(keypair.publicKey, 1); // 1 SOL
   
    console.log("owner of keypair after airdrop:",
    (await anchor.getProvider().connection.getAccountInfo(keypair.publicKey)).owner.toBase58());
    
    await program.methods.initializeKeypair()
      .accounts({keypair: keypair.publicKey})
      .signers([keypair]) // the signer must be the keypair
      .rpc();

    console.log("owner of keypair after initialize:",
    (await anchor.getProvider().connection.getAccountInfo(keypair.publicKey)).owner.toBase58());
 
  });
});


```


## The BPFLoaderUpgradeable owns the programs

- The command that help to check the owner of the program is - `solana account <PROGRAM_ID>`

The wallet that deployed the program is not the owner of it. The reason Solana programs are able to be upgraded by the deploying wallet is because the BpfLoaderUpgradeable is able to write new bytecode to the program, and it will only accept new bytecode from a predesignated address: the address that originally deployed the program.


## Program can transfer ownership of owned accounts 

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;
use anchor_lang::system_program;

declare_id!("...");

#[program]
pub mod chage_owner {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn change_owner(ctx: Context<ChangeOwner>) -> Result<()>{
        let account_info = &mut ctx.accounts.my_storage.to_account_info();

        account_info.assign(&system_program::ID);

        let res = account_info.realloc(0,false);

        if !res.is_ok() {
            return err!(Err::ReallocFailed);
        }

        Ok(())
    }
}

#[error_code] 
pub enum Err {
    #[msg("realloc failed")]
    ReallocFailed,
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = signer,
        space = size_of::<MyStorage>() + 8,
        seeds = [],
        bump
    )]
    pub my_storage : Account<'info,MyStorage>,

    #[account(mut)]
    pub signer : Signer<'info>,

    pub system_program: Program<'info,System>,
}

#[derive(Accounts)]
pub struct ChangeOwner<'info> {
    #[account(mut)]
    pub my_storage: Account<'info, MyStorage>,
}

#[account]
pub struct MyStrorage {
    x: u64,
}
```

testing with typescript 

```typescript
import * as anchor from "@coral-xyz/anchor";
import {Program} from "@coral-xyz/anchor";
import {ChangeOwner} from "../target/types/change_owner";

import privateKey from "/Users/himanshu/.config/solana/id.json";

decribe("change_owner", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const Program = anchor.workspace.ChangeOwner as Program<ChangeOwner>;

    it("Is initalized!" ,async() => {
        const deplayer = anchor.web3.Keypair.fromSecretKey(Uint8Array.from(privateKey));

        const seeds = []
        const [myStorage, _bump] = anchor.web3.PublicKey.findProrgamAddressSync(seeds.program.programId);

        console.log("the storage account address is ", myStorage.toBase58());

        await program.methods.initialize().accounts.({myStorage: myStorage}).rpc();

        await program.methods.changeOwner().accounts.({myStorage: myStorage}).rpc();

        // after the ownership has been transfered , then account can still be initialized again

        await program.methods.initialize().accounts({
            myStorage: myStorage
        }).rpc();

    })
})

```

- After transferring the account, the data must be erased in the same transaction. Otherwise, we could insert data into owned accounts of other programs. 
- This is the account_info.realloc(0, false); code. The false means don’t zero out the data, but it makes no difference because there is no data anymore


## Transferring SOL out of a PDA: Crowdfund example 

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program;
use std::mem::size_of;
use std::str::FromStr;

declare_id!("...");

#[program]
pub mod crowdfund {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let initialized_pda = &mut ctx.accounts.pda;
        Ok(())
    }

    pub fn donate(ctx: Context<Donate>, amount: u64) -> Result<()> {
        let cpi_context = CpiContext::new(
            ctx.accounts.system_program.to_account_info(),
            system_program::Transfer {
                from: ctx.accounts.signer.to_account_info().clone(),
                to: ctx.accounts.pda.to_account_info().clone(),
            },
        );

        system_program::transfer(cpi_context, amount)?;

        Ok(())
    }

    pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
        ctx.accounts.pda.sub_lamports(amount)?;
        ctx.accounts.signer.add_lamports(amount)?;

        // in anchor 0.28 or lower, use the following syntax:
        // **ctx.accounts.pda.to_account_info().try_borrow_mut_lamports()? -= amount;
        // **ctx.accounts.signer.to_account_info().try_borrow_mut_lamports()? += amount;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,

    #[account(init, payer = signer, space=size_of::<Pda>() + 8, seeds=[], bump)]
    pub pda: Account<'info, Pda>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Donate<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,

    #[account(mut)]
    pub pda: Account<'info, Pda>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Withdraw<'info> {
    #[account(mut, address = Pubkey::from_str("...").unwrap())] // Here implement your address form solana address
    pub signer: Signer<'info>,

    #[account(mut)]
    pub pda: Account<'info, Pda>,
}

#[account]
pub struct Pda {}


```

In case of PDA . We can directly deduct lamports form it .
You don’t need to own the account you are transferring lamports to.

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Crowdfund } from "../target/types/crowdfund";

describe("crowdfund", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Crowdfund as Program<Crowdfund>;

  it("Is initialized!", async () => {
    const programId = await program.account.pda.programId;

    let seeds = [];
    let pdaAccount = anchor.web3.PublicKey.findProgramAddressSync(seeds, programId)[0];

    const tx = await program.methods.initialize().accounts({
      pda: pdaAccount
    }).rpc();

    // transfer 2 SOL
    const tx2 = await program.methods.donate(new anchor.BN(2_000_000_000)).accounts({
      pda: pdaAccount
    }).rpc();

    console.log("lamport balance of pdaAccount",
    await anchor.getProvider().connection.getBalance(pdaAccount));

    // transfer back 1 SOL
    // the signer is the permitted address
    await program.methods.withdraw(new anchor.BN(1_000_000_000)).accounts({
      pda: pdaAccount
    }).rpc();

    console.log("lamport balance of pdaAccount",
    await anchor.getProvider().connection.getBalance(pdaAccount));

  });
});

```