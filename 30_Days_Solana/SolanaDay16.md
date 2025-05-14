## Accounts In Solana

### Initializing Accounts in Solana and Anchor

- In Solidity and Ethereum , a more exotic design pattern to store data is SSTORE2 or SSTORE3 where the data is stored iin the bytecode of another smart contract

Solana uses same mechanism for data Storage

Storage Slots in Ethereum are effectively a massive key-value store
```
{
    key: [smart_contract_address, storage_slot]
    value: 32_byte_slot  // (For example : 0x00)
}
```

Solana model is Similar: it is massive key-value store where the "key" is a base 58 encoded address , the value is data blob size upto 10 MB

```
{
    // key is a base58 encoded 32 byte sequence
    key: ETnqC8mvPRyUVXyXoph22EQ1GS5sTs1zndkn5eGMYWfs
    value: {
        data: 020000006ad1897139ac2bdb67a3c66a...
        // other fields are omitted
    }
}

```

In ethereum , teh bytecode of the smart contract and the storage varaibles of the smart contract are stored seperately , they are indexed differently and must loaded using teh different APIs

How Ethereum maintains state: Each account is a leaf in a Merkle tree. Note that the “storage variables” are stored “inside” the account of the smart contract

In Solana, everything is an account that potentially hold data. Someties we refers to one account as a "program Account" or another account as `storage Account`, but the only diff is whether the executable flag is set to true and how we intended to use the data field of the account.Solana accounts can be though as files, they hold content, but they also have metadata that indicates whho owns the file,if it is executable 

In Solana, all “storage variables” can be read by any program, but only its owner program can write to it.The way storage is “tied to” a program is via the owner field.

- Solana programs need to be initialized before they can be used

- In Solana Program , particularly Anchor, all storage or rather account data, is treated as a struct. Anchor deserializes and serializes account data into structs when we try to read or write the data.

```rust 
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("...");

#[program]
pub mod basic_storage {
    use super::*;

    /// NOTE - Here We are Initializing the account
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
/// NOTE - Initialize Struct references to the resources to initialize an account this is use in the function we are defining funtion for
pub struct Initialize<'info> {
    #[account(
        init,  /// initialzing the account 
        payer = signer,  // Who is paying the SOL for allocating Storage
        space = size_of::<MyStorage> + 8, // how much space the account will take , we use the `std::mem::size_of` , 8 is used as descriminator between the account to seperate it 

        /// NOTE - A program can own multiple accounts, it “discriminates” among the accounts with the “seed” which is used in calculating a “discriminator”.
        seeds = [], 
        bump
    )]
    pub my_storage: Account<'info,MyStorage>,  /// NOTE - It take the Struct that is used for Data Storage

    #[account(mut)]  // Here is it mut , beacuse balance is changing , some SOL is deducted from the account
    pub signer: Signer<'info>, /// NOTE -  the wallet that is paying for the “gas” for storage of the struct

    pub system_program: Program<'info,System>,
}

#[account]
pub struct MyStorage {
    x: u64,
}
```

- `'info` keyword is a Rust lifetimes -> How much time it will store for with there it is used


Unit Test Initialization

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { BasicStorage } from "../target/types/basic_storage";

describe("basic_storage", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.BasicStorage as Program<BasicStorage>;

  it("Is initialized!", async () => {
    const seeds = []
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    console.log("the storage account address is", myStorage.toBase58());

    await program.methods.initialize().accounts({ myStorage: myStorage }).rpc();
  });
});


```