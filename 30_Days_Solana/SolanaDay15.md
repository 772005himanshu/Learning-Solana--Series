## Introduction to Solana Compute units and Transaction Fees

- In Ethereum , the price of a transaction is computed as `gasUsed * gasPrice`. This tell us how much ether we wna to spent to iinclude it in blockchain,Before a transaction is send gasLimit is specified and paid upfront. If transaction fee is more than `gasLimit` it reverts

- In solana we have the compute units , and each transaction is capped at 200,000 compute Units . If the transaction cost more than 200,000 compute units , it reverts

In ethereum, gas costs for computing are treated the same gas costs associated with storage.In Solana , Storage is handled differently

Both chains execute compiled bytecode and charge a fee for each instruction executed

- Ethereum has EVM bytecode, Solana has modified version of `berkeley packet filter` (eBPF) called solana packet filter (different from eBPF removal of bytecode verifier by adding the compute units)

- Ethereum charges different prices for different op codes depending on how long they take to execute, ranging form one gas to thousands of gas, In Solana each opcode coasts one compute unit.

- Heavy computational operations that cannot be done below the limit --> do it multiple transaction


### Compute unit optimization

- The computational resources used in a transaction does not affect the fees paid for that transaction.

- In addition to compute units, the number of `signers for the Solana transaction` affects the compute unit cost.

-  Transaction fees are solely determined by the number of signatures that need to be verified in a transaction. The only limit on the number of signatures in a transaction is the max size of transaction itself. Each signature (64 bytes) in a transaction (max 1232 bytes) must reference a unique public key (32 bytes) so a single transaction could contain as many as 12 signatures

- Get the simple anchor solana Program , To find out the transaction fee / Compute Unit we find the balanace before and after the transcation


TestFile Looks like that:
```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ComputeUnit } from "../target/types/compute_unit";

describe("compute_unit", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.ComputeUnit as Program<ComputeUnit>;
  const defaultKeyPair = new anchor.web3.PublicKey(
    "Replace your defaukt keypair get it by solana address"
  );

  it("Is initialized!", async () => {
    let bal_before = await program.provider.connection.getBalance(
      defaultKeyPair
    );
    console.log("before:", bal_before);

    // Transaction call 
    const tx = await program.methods.initialize().rpc();

    let bal_after = await program.provider.connection.getBalance(
      defaultKeyPair
    );
    console.log("after:", bal_after);

    console.log(
      "diff:",
      BigInt(bal_before.toString()) - BigInt(bal_after.toString())
    );
  });
});

```

In terminal we get 

```
# test logs
		compute_unit
before: 15538436120
after: 15538431120
diff: 5000n


# solana logs
Status: Ok
Log Messages:
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC invoke [1]
  Program log: Instruction: Initialize
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC consumed 320 of 200000 compute units
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC success


```

Here we see the compute Units is `320` in Log Messages

if we add complexity to code

```rust
pub fn initilize(ctx: Context<Initialize>) -> Result<()> {
    let mut a = Vec::new();
    a.push(1);
    a.push(2);
    a.push(3);
    a.psuh(4);

    Ok(())
}
```

OutPut at terminal
```
# test logs
compute_unit
before: 15538436120
after: 15538431120
diff: 5000n


# solana logs
Status: Ok
Log Messages:
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC invoke [1]
  Program log: Instruction: Initialize
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC consumed 593 of 200000 compute units
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC success


```

- The compute units get increased , but the transaction fee remians same, This tells us the compute units doesnot effect the transaction fee paid by the user

- Regardless of compute units , the transaction fee is 5000 lamports or 0.0000005 sol

- Smaller integers save compute units (large types takes up large space in memory than the smaller types)

Same thing go for program derived account(PDA) on chain using `find_program_address` use more compute units (iteration over call and go back and getting back 256 subtracting unique bump(Learned in Security Check)), So use the `find_program_address` off-chain and pass the resulting bump seed to teh program when possible
