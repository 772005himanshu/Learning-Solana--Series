## Tx.origin, msg.sender, and OnlyOwner in Solana: identifying the caller

In Solidity,
- `msg.sender` -> a global varaible , represent the address that called or initiated a function call on a smart contract.
- `tx.origin` -> wallet that signed the transaction

- `onlyOwner` -> the contract owner or something can be set by owner of the contract/ modifier for access Control Checks

In Solana, there is no equivalent of `msg.sender`

There is equivalent to `tx.origin` , but aware Solana transaction can have multiple signers, so we could think of its as `multiple tx.origins`

Set up for `tx.origin` address in Solana

```rust
use anchor_lang::prelude::*;

declare_id!(...);

#[program]
pub mod day14 {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let the_signer1 : &mut Signer = &mut ctx.accounts.signer1;

        msg!("The Signer1 is {:?}" , *the_signer1.key);

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub signer1: Signer<'info>,
}
```

Test for the Above Program

```typescript
decscribe("Day14", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const program = anchor.workspace.Day14 as Program<Day14>;

    it("Is signed by a single signer", async() => {
        const tx = await program.methods.initialize().accounts({
            signer1: program.provider.publicKey
        }).rpc();

        console.log("The Signer1: ", program.provider.publicey.toBase58());
    }); 
})
```

## Multiple Signers
- In Solana, we have have more than one signer sign a transaction,think of this a batching up a bunch of signatures and sending it in one transaction. One use-case is doing a multisig transaction in one transaction.

```rust
use anchor_lang::prelude::*;

declare_id(...);

#[program]
pub mod day14 {
    use super::*;

    pub fn initilize(ctx: Context<Initialize>) -> Result<()> {
        let signer_1: &mut Signer = &mut ctx.accounts.signer1;
        let the_signer2: &mut Signer = &mut ctx.accounts.signer2;

        msg!("The signer1: {:?}", *the_signer_1.key);
        msg!("The signer2: {:?}", *the_signer2.key);

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    pub signer1: Signer<'info>,
    pub signer2: Signer<'info>, 
}
```


```typescript
describe("Day14", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const program = anchor.workspace.Day14 as Program<Day14>;

    let myKeypair = anchor.web3.Keypair.generate();

    it("Is signed by multiple signers", async () => {
        const tx = await program.methods.initialize().accounts({
            signer1: program.provider.publicKey,
            signer2: myKeypair.publicKey,
        }).signers([myKeypair]).rpc();  // Here we have added the signer array for multi Signer

        console.log("The signer1: ",program.provider.publicKey.toBase58());
        console.log("The signer2:", myKeypair.publicKey.toBase58());
    });
})
```


### OnlyOwner
- This is common pattern used in the solidity to restrict a function access to only the owner of the contract.
- Using the `#[access_control]` attribute from the anchor, also implement the onlyOwner pattern , that restrict a function's access in our Solana program to a PubKey (owner's address)

```rust
use anchor_lang::prelude::*;

declare_id!(...);

const OWNER: &str = "...";

#[program]
pub mod day14{
    use super::*;

    #[access_control(check(&ctx))]  /// These check are excute before initialize function
    pub fn initialize(ctx: Context<OnlyOwner>) -> Result<()> {
        msg!("Hello, Im the Owner");
        Ok(())
    }
}

fn check(ctx: &Context<OnlyOwner>) -> Result<()> {
    // Check if signer == owner
    require_key_eq!(ctx.accounts.signer_account.key(),OWNER.parse::<Pubkey>().unwrap(),OnlyOwnerError::NotOwner);

    Ok(())
}

#[derive(Accounts)]
pub struct OnlyOwner<'info> {
    signer_account: Signer<'info>,
}

#[error_code]
pub enum OnlyOwnerError {
    #[msg("Only Owner can call this function!")]
    NotOwner,
}
```

## Testing the onlyOwner functionality , when signer is Owner

```typescript
import * as anchor from "@coral-xyz/anchor";
import {Program} from "@coral-xyz/anchor";
import {Day14} from "../target/types/day14";

describe("day14", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const program = anchor.workspace.Day14 as Program<Day14>;

    it("Is called by the owner ", async =() => {
        const tx = await program.methods.initialize().accounts({
            signerAccount: program.provider.publicKey,  // it get data from the OnlyOwner struct and Signer<'info> to validate wallet accoutn actually signed th etransaction 
            // Testing with help of anchor test --skip-local-validator
        }).rpc();

        console.log("Transaction hash: ", tx);
    });
})

```

## Testing if the signer is not Owner - attack case

- Using different keypair that is not owner to call the initialize function

```typescript
import * as anchor from "@coral-xyz/anchor";
import {Program} from "@coral-xyz/anchor";
import {Day14} from "../target/types/day14";
import {Keypair} from "@solana/web3.js";


describe("day14", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const program = anchor.workspace.Day14 as Program<Day14>;

    let Keypair = anchor.web3.Keypair.generate();  // Here we generate random keypair to test it proper functioning

    it("Is called by the owner ", async =() => {
        const tx = await program.methods.initialize().accounts({
            signerAccount: Keypair.publicKey,
        }).signers([Keypair]).rpc();

        console.log("Transaction hash: ", tx);
    });
})

```


Exercise - Upgrade a program like the one above to have a new owner. 

```Rust
use anchor_lang::prelude::*;

declare_id!(...);

const OWNER: &str = "...";

#[program]
pub mod day14 {
    use super::*;

    #[access_control(check_initialOwner(&ctx))]
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let owner_account = &mut ctx.accounts.owner_account;
        owner_account.owner = ctx.accounts.initial_owner.key();
        Ok(())
    }

    #[access_control(check_owner(&ctx))]
    pub fn change_owner(ctx: Context<ChangeOwner>, new_owner: Pubkey) -> Result<()> {
        let owner_account = &mut ctx.accounts.owner_account;
        owner_account.owner = new_owner;
        msg!("Owner changed to: {}", new_owner);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = initial_owner,
        space = 8 + 32 // 8 for discriminator + 32 for pubkey
    )]
    pub owner_account: Account<'info, OwnerAccount>,
    #[account(mut)]
    pub initial_owner: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct ChangeOwner<'info> {
    #[account(mut)]
    pub owner_account: Account<'info, OwnerAccount>,
    #[account(mut)]
    pub current_owner: Signer<'info>,
}

#[account]
pub struct OwnerAccount {
    pub owner: Pubkey,
}

fn check_initalizeOwner(ctx: &Context<Initialize>) -> Result<()> {
    require_key_eq!(ctx.accounts.initial_owner.key(),OWNER.parse::<Pubkey>().unwrap(),OnlyOwnerError::NotOwner);
}

fn check_owner(ctx: &Context<ChangeOwner>) -> Result<()> {
    require!(
        ctx.accounts.current_owner.key() == ctx.accounts.owner_account.owner,
        OnlyOwnerError::NotOwner
    );
    Ok(())
}

#[error_code]
pub enum OnlyOwnerError {
    #[msg("Only the current owner can call this function!")]
    NotOwner,
}
```