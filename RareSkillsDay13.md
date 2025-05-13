## Solana logs, "Events" and Transaction History

Solana programs can emit events similar to how to EVM , There are some difference

Events in Solana are intended to pass information to the frontend rather than document past transaction, o get past history Solana transaction can be quired by address

## Solana logs and Events 
- Solana Events have fields in the struct 

```Rust
use anchor_lang::prelude::*;

declare_id!(...);

#[program]
pub mod emit {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        emit!(MyEvent {value: 42}); // -->  Here we are Emitting the event
        Ok(())
    }
}


#[derive(Accounts)]
pub struct Initialize {

}

#[event]
pub struct MyEvent {
    pub value: u64,
}
```

- Events become the part of the Solana program's IDL

- There is no such thing like indexed or non-indexed in Solana like there is Ethereum

- Unlike ethereum, we cannot directly query for past events over a range of block numbers. 

- THe code below how to listen to events in Solana

```typescript
import * as anchor from "@coral-xyz/anchor";
import { BorshCoder, EventParser, Program } from "@coral-xyz/anchor";
import { Emit } from "../target/types/emit";

describe("emit", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Emit as Program<Emit>;

  it("Is initialized!", async () => {
    const listenerMyEvent = program.addEventListener('MyEvent', (event, slot) => {
      console.log(`slot ${slot} event value ${event.value}`);
    });

    await program.methods.initialize().rpc();

    // This line is only for test purposes to ensure the event
    // listener has time to listen to event.
    await new Promise((resolve) => setTimeout(resolve, 5000));

    program.removeEventListener(listenerMyEvent);
  });
});


```


- In the EVM , logs are emitted by running the log0 , log1 , log2 etc opcode , Using the Foundry in the Smart Contract by `import {console} from "forge-std/console.sol";` by this import we can log the transaction `console.log("{}", anything)`

In Solana, logs are run by calling the system calls `sol_log_data`, it simply a sequence of bytes.

Example of this system calls:

```rust
/// Print some slices as base64.
pub fn sol_log_data(data: &[&[u8]]) {
    #[cfg(target_os = "solana")]
    unsafe {
        crate::syscalls::sol_log_data(data as *const _ as *const u8, data.len() as u64)
    };

    #[cfg(not(target_os = "solana"))]
    crate::program_stubs::sol_log_data(data);
}
```

- The Struct is used to create an events in an abstraction of the bytes sequence , Anchor coonvert the struct into bytes of the Sequence to pass to this function. The Solana System calls only takes a bytes sequence , not a struct


- Solana logs can  not be used for auditing purpose , it used to interact with the forntend, Solana function cannot return data to frontend the way solidity view function can, Solana logs are lightweight way to accomplish this.

## Getting the transaction history in Solana
- Solana has an RPC function `getSignaturesForAddress` whuch list down all the transaction by that address, the address can be program or wallet

```javascript
let web3 = require('@solana/web3.js');

const solanaConnection = new web3.Connection(web3.clusterApiUrl("mainnet-beta"));

const getTransactions = async(address,limit) => {
  const pubKey = new web3.PublicKey(address);
  let transactionList = await solanaConnection.getSignaturesForAddress(pubKey, {limit: limit});
  let signatureList = transactionList.map(transaction => transaction.signature);

  console.log(signatureList);

  for await (const sig of signatureList) {
    console.log(await solanaConnection.getParsedTransaction(sig, {maxSupportedTransactionVersion: 0}));
  }
}

let myAddress = "enter and address here";

getTransactions(myAddress, 3);



```

- Actual content of the transaction is retrieved by using the `getParsedTransaction` RPC Method