## Attack Vector 3

# Summary
[low] Insecure Authority Transfer Leading To Potential DoS

In the `transfer_authority` function , if the authority is transferred to an invalid or inaccessible address , it could result in a complete denial of service(DoS) for protocol configuration and core functionalities.

Since there is no verification mechanism ensuring that the `new_authority` is a valid and active entity , the protocol may become permanently loced if an incorrect address is set

## What went Wrong and how to fix it next time
check should be there before transfering authority like should be valid signer, valid account etc

## Take aways

1. `Require the new Authority to be signer` for the transaction , ensuring that they acknowledge and accept the role

2. `Implement a two-step authority transfer process` by introducing a `claim_authority` function . This approach ensures that the new authority explicitly claims their role before the transfer is finalized

Modify the `TransferAuthority` struct to enfore that the `new_authority` sign the transaction

```rust
#[derive(Accounts)]
pub struct TransferAuthority<'info> {
    pub authority : Signer<'info>,  // Current Authority

    #[account(
        mut,
        has_one = authority @ ErrorCode::Unauthorized
    )]
    pub config: Account<'info,Config>,

    pub new_authority: Signer<'info>,  // Ensure the new authority is a valid Signer

    pub system_program: Program<'info,System>,
}
```

# Summary
[low] Setting the destination address to a PDA of a Token Account Closure to SOL Lock

Closing a token account transfer its SOL balance to specified destination account . However , the destination aaccount ,ust be able to move SOL . Setting the destination to a PDA can result in locked SOL since a PDA cannot dierctly move SOL


## What went Wrong and how to fix it next time
```rust
close_vault(
    &ctx.accounts.token_program
    &ctx.accounts.bonding_curve_vault.to_account_info(),
    &ctx.accounts.vault_aauthority.to_account_info(),
@>    &ctx.accounts.recipient_token_account.to_account_info(),
    vault_authority_signer,
)?;

```
Set the destination address to the `recipient` directly instead of a PDA .This ensures that the recipient can receive and move the SOL immediately after closure

## Take aways
Never set close account destination to PDA accounts , SOL Should be locked in that


# Summary
[low] Missing Validation for Raydium Program Address

In the `swap_tokens_for_sol_on_raydium` function , the Raydium Program Address (`cp_swap_program`) is not explicitly validated

## What went Wrong and how to fix it next time

```rust
/// input_token_mint and output_token_mint have the same token program
pub token_program: Interface<'info, TokenInterface>,

pub cp_swap_program: Program<'info, RaydiumCpmm>, // Here cp_swap of Raydium is not checked to be correct
```

## Take aways
To enfore security and ensure the correct Raydium Program is used , add explicit validation by specifying the expected address

```rust
#[account(address = raydium_cpmm_cpi::ID)]
pub cp_swap_program: Program<'info, RaydiumCpmm>,
```

# Summary
[low] Missing Validation for `ammount > 0` in `wrap` function 

The `wrap` function is expected to validation that the `amount` parameter is greater than zero before proceeding.Here , no such validation is currently implemented. Which Contradict the Docs

## What went Wrong and how to fix it next time

```rust
pub fn wrap(ctx: Context<Wrap>, amount; u64) -> Result<()> {
    let transfer_accounts = TransferChecked {
        from: ctx.accounts.depositor_boop_token_account.to_account_info(),
        mint: ctx.accounts.boop.to_account_info(),
        to: ctx.accounts.boop_vault.to_account_info(),
        authority: ctx.accounts.depositor.to_account_info(),
    };

    let cpi_ctx = CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        transfer_accounts,
    );

    transfer_checked(cpi_ctx, amount, ctx.accounts.boop.decimals)?;
}
```

## Take aways
Add a validation check to ensure that `amount` is greater then zero 

```rust
require!(amount > 0, ErrorCode::ZeroAmount);
```

# Summary
[low] Missing Validation for `Protocol_fee_recipient` in `update_config`

In the `update_config` function , the `Protocol_fee_recipient` address is updated without validation. If the recipient address is set to the default `Pubkey::default()` ,it can lead to denial-of-service (Dos) issue, preventing core protocol functionalities from operating correctly

## What went Wrong and how to fix it next time
Current implementation
```rust
ctx.accounts.config.protocol_fee_recipient = new_protocol_fee_recipient;
```

This allow an invalid recipient address to be set , which may cause transaction requiring fee distribution to fail , also the same issue exist in the sboop program in the function `update_config`

## Take aways
Add a validation check to ensure that `new_protocol_fee_recipient` is not set to the default public key
```rust
require!(
    new_protocol_fee_recipient != Pubkey::default(), ErrorCode::InvalidProtocolFeeRecipient
);
```

# Summary
[low] Permanent Freezing of `create_raydium_pool` Due to Seed Contraint

The `create_raydium_pool` function is vulnerable to permanent freezing because the `pool_state` account is derived from a fixed seed. If the same `pool_state` is created oon Raydium before this migration, the initialization will fail with the error `poolAlreadyCreated`, making the function permanently unusable

```rust
seeds = [
    POOL_SEED.as_bytes(),
    amm_config.key().as_ref(),
    token_0_mint.key().as_ref(),
    token_1_mint.key().as_ref(),
],
```

This constraint enforces a deterministic address for the pool , which cause conflicts if the pool has already been created on Raydium.

This issue was prevalent in bonding curves , prompting the raydium tea, to remove this constraint and allow any random account to serve as `pool_state` providd it sign the transaction 

## What went Wrong and how to fix it next time

#### Updated Raydium Implementation

```rust
/// CHECK: Initialize an account to store the pool state ,init by contract
/// PDA acount:
/// seeds = [
///     POOL_SEED.as_bytes(),
///     amm_config.key().as_ref(),
///     token_0_mint.key().as_ref(),
///     token_1_mint.key().as_ref(),
/// ],
/// 
/// Or random account: must be signed by CLI
#[account(mut)]
pub pool_state: UncheckedAccount<'info>,
```

#### Impact
- The `create_raydium_pool` function will be permanently frozeen if the same pool has aready been initialized of Raydium

## Take aways

#### Recommendation
Remove the seed constraint to allow successful pool initialized .Instead , store the pool address off-chain or on-chain as needed

```rust
/// CHECK: Initialize an account to store the pool state ,init by cp-swap
#[account(mut)]
pub pool_state: UncheckedAccount<'info>,
```

This approach ensures that the pool creation is not blocked by pre-existing deployments and aligns with Raydium updated methodology