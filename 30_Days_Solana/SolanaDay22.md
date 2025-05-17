## Function modifier (view, pure , payable ) and fall back function in Solana : why they don't exist


### Solana Doesnot have fallback and recieve functions
- A Solana transactions must specify in advance the accounts it will modify or reads as part od a transaction. 
- If a "fallback" function accesses an indeterminant account, then the entire transaction fails. 
- This would put an onus on the user to anticipate the accounts the fallback function will access.It much simpler as a result to simply disallow these kinds of fuunction

## Solana does not have the notion of `view` or `pure` functions

- A `view` function in solidity creates a guarantee that the state will not change using two mechanisms

- All external calls iin a view function are `staticalls`(calls that reverts if a state change happens)
- If the compiler detects state-changing opcodes , it throws an error

- `pure` function by compiler checking if there are opcodes that view the state

Anchor does not implement any of these compiler checks. if the account is not included in Context struct defination , that function woon't access that account

- There is no such thing as `staticcall` in the Solana Virtual machine or runtime

### View function aren't necessary in Solana anyway
Because a Solana program can read any account passed to it , it can read an account owned by another program

The program owning the account doesnot need to implement a view function to grant access for another program to view function, The web3js client - or another - can view the Solana account datat directly 

`It is not possible to use reentrancy locks to directly defend against read only reentrancy in Solana. The program must expose flag for readers to know if the data is reliable`

- Read only reentrancy happens when a victim contract accesses a view function that is showing the manipulated data.
- In Solidity we can defened against this , by adding the nonReentrant modifier to te view function
- In Solana there is no way to prevent another program from viewing data in an account

Solana programs can still implement the flags reentrancy guard use.
Programs consuming the account of another program can check those flags to see if the account might currently be in a reentrant state and should not be trusted.

### There is no custim modifier in Rust 

### Custom units are not available in Rust or Anchor - like in Solidity - ethers and wei , days = 84600 seconds

### There is no such thing as “payable” functions in Solana. Programs transfer SOL from the user, users do not transfer SOL to the program