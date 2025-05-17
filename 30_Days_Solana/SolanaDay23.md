## Transferring SOL and building a payment splitter: "msg.value" in Solana

- Solana Anchor can transfer SOL as part of the transaction mechanism

- Unlike Ethereum where wallet specify msg.sender as part of the transaction and `push` the ETH to the contract,Solana programs `pull` the Solana from the wallet 

- As there is no thing like `payable` in Solana or `msg.sender`

Solana Program to transfer the SOL from the sender to the recipient

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program;

declare_id!("...");

#[program]
pub mod sol_splitter {
    use super::*;

    pub fn send_sol(ctx: Context<SendSol>, amount: u64) -> Result<()> {
        let cpi_context = CpiContext::new( // Hold the program we are going to call 
            ctx.accounts.system_program.to_account_info(), 

            system_program::Transfer {  // account that will included as part of that transaction
                form: ctx.accounts.signer.to_account_info(),
                to: ctx.accounts.recipient.to_account_info(),
            }
        );

        let res = system_program::transfer(cpi_context, amount); // We have the cpi_context , we can do a cross program invocation to the syetm_program , while specifying the amount

        if res.is_ok() {
            return Ok(());
        }else {
            return err!(Errors::TransferFailed);
        }
    }
}

#[error_code]
pub enum Errors {
    #[msg("transfer failed")]
    TransferFailed,
}

#[derive(Accounts)]
pub struct SendSol<'info> {
    /// CHECK: We donot read and write the data of this account
    #[account(mut)]
    recipient: UncheckedAccount<'info>,

    system_program: Program<'info,System>,

    #[account(mut)]
    signer: Signer<'info>,
}
```

### Introduction the CPI: Cross Program Invocation
In ethereum , transferring ETH is done simply by specifiying a value in the `msg.value` field, In Solana , a built-in program called the `system program` transfers SOL from one account to another.

We can think the system program as a precompile in Ethereum . Imagine it behaves sort like an ERC20 token built into the Prortcol that is used as the native currency And it has a public function called `transfer`

## Context For CPI transactions

Solana Program function is called a `Context` must be provided . That `Context` holds all the accounts that the program will interact with

Calling the system program in no different . The system program need a `Context` holding the `from` and `to` accounts. The amount that is transferred is passed as a `regular` argument - it is not part of the `Context` (as 'amount' is not an account , it value)


### Do not ignore the retunr values of cross program invocations

- To check if the cross program invocation succeeded, we just need to check the returned value is an `Ok`. Rust makes this straightforward with the `is_ok()` method


### Only the signer can be `from`
- If we call the system program with `from` being an account that is not a signer , then the system program will reject the call . Without a signature , the system program can't know if you authorized the call or not


```typescript
import * as anchor from "@coral-xyz/anchor";
import {Program} from "@coral-xyz/anchor";
import { SolSplitter} from "../traget/types/sol_splitter";

describe("sol_splitter", () => {
    anchor.setProvider(anchor.AnchorProvider.env());

    const program = anchor.workspace.SolSplitter as Program<SolSplitter>;

    async function printAccountBalance(account) {
        const balance = await anchor.getProvider().connection.getBalance(account);
        console.log(`${account} has ${balance / anchor.web3.LAMPORTS_PER_SOL} SOL`);
    }

    it("Transmit SOL", async () => {
        // generate a new wallet
        const recipient = anchor.web3.Keypair.generate();

        await printAccountBalance(recipient.publicKey);

        // send tha account 1 SOL via the program
        let amount = new anchor.BN(1 * anchor.web3.LAMPORTS_PER_SOL);
        await program.methods.sendSol(amount).accounts({recipient: recipient.publicKey}).rpc();

        await printAccountBalance(recipient.publicKey);
    });
});
```

Exercise: Build a Solana program that splits up the incoming SOL evenly among two recipients. You will not be able to accomplish this via function arguments, the accounts need to be in the Context struct.



### Building a payment splitter: using an arbitrary number of accounts with `remaining_accounts`

- Specify a Context struct like if we wanted to split SOL among several accounts - naming to many account to Context struct is not good way to solve this problem

- To solve this , Anchor add a `remaining_accounts` field to `Context` structs

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program;

declare_id!("...");

#[program]
pub mod sol_splitter {
    use super::*;

    pub fn split-sol<'a,'b,'c,'info>(ctx: Context<'a,'b,'c,'info,SplitSol<'info>>, amount: u64) -> Result<()> {
        let amount_each_gets = amount / ctx.remaining_accounts.len() as u64;

        let system_program = &ctx.accounts.system_program;

        for recipient in ctx.remaining_accounts {
            let cpi_accounts = system_program::Transfer(
                from: ctx.accounts.signer.to_account_info(),
                to : recipient.to_account_info(),
            )

            let cpi_program = system_program.to_account_info();
            let cpi_context = CpiContext::new(cpi_program,cpi_accounts);

            let res = system_program::transfer(cpi_context,amount_each_gets);

            if(!res.is_ok()) {
                return err!(Errors::TrabsferFailed);
            }
        }

        Ok(())
    }
}

#[error_code]
pub enum Errors {
    #[msg("transfer failed")]
    Transferfailed,
}

#[derive(Accounts)]
pub struct SplitSol<'info> {
    #[account(mut)]
    signer: Signer<'info>,
    system_program: Program<'info,System>,
}

```


```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { SolSplitter } from "../target/types/sol_splitter";

describe("sol_splitter", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.SolSplitter as Program<SolSplitter>;

  async function printAccountBalance(account) {
    const balance = await anchor.getProvider().connection.getBalance(account);
    console.log(`${account} has ${balance / anchor.web3.LAMPORTS_PER_SOL} SOL`);
  }

  it("Split SOL", async () => {
    const recipient1 = anchor.web3.Keypair.generate();
    const recipient2 = anchor.web3.Keypair.generate();
    const recipient3 = anchor.web3.Keypair.generate();

    await printAccountBalance(recipient1.publicKey);
    await printAccountBalance(recipient2.publicKey);
    await printAccountBalance(recipient3.publicKey);

    const accountMeta1 = {pubkey: recipient1.publicKey, isWritable: true, isSigner: false};
    const accountMeta2 = {pubkey: recipient2.publicKey, isWritable: true, isSigner: false};
    const accountMeta3 = {pubkey: recipient3.publicKey, isWritable: true, isSigner: false};

    let amount = new anchor.BN(1 * anchor.web3.LAMPORTS_PER_SOL);
    await program.methods.splitSol(amount)
      .remainingAccounts([accountMeta1, accountMeta2, accountMeta3])
      .rpc();

    await printAccountBalance(recipient1.publicKey);
    await printAccountBalance(recipient2.publicKey);
    await printAccountBalance(recipient3.publicKey);
  });
});


```

### ctx.remaining_accounts

The loop loops through for `recipient in ctx.remaining_accounts`. The keyword `remaining_acocunts` is the Anchor mechanism for passing in an arbitrary number of accounts without having to create a bunch of keys in the Context struct

In typescript test , we add `remaining_accounts` to transaction like so:

```typescript
await program.methods.splitSol(amount)
  .remainingAccounts([accountMeta1, accountMeta2, accountMeta3])
  .rpc();
```