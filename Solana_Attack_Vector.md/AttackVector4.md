## Audit Assignment 4(GMX Solana Protocol)

# Summary
[High] Incorrect authority for token trasfer in `withdraw_from_treasury_vault()`

The authority for `WithdrawFromTreasuryVault` CPI context is incorrectly set as `self.config_to_account_info()`. It should instead be `self.treasury_vault_config.to_account_info()`

This issue will cause `withdraw_from_treasury_vault()` to always fail

```rust
impl<'info> WithdrawFromTreasuryVault<'info> {
    fn transfer_checked_ctx(&self) -> CpiContext<'_,'_,'_,'info, TransferChecked<'info>> {
        CpiContext::new(
            self.token_program.to_account_info(),
            TransferChecked {
                from: self.treasury_vault.to_account_info(),
                mint: self.token.to_account_info(),
                to: self.target.to_account_info(),
@>                authority: self.config.to_account_info(),
            },
        )
    }
}
```


## What went Wrong and how to fix it next time
```diff
impl<'info> WithdrawFromTreasuryVault<'info> {
    fn transfer_checked_ctx(&self) -> CpiContext<'_,'_,'_,'info, TransferChecked<'info>> {
        CpiContext::new(
            self.token_program.to_account_info(),
            TransferChecked {
                from: self.treasury_vault.to_account_info(),
                mint: self.token.to_account_info(),
                to: self.target.to_account_info(),
-                authority: self.config.to_account_info(),
+                authority: self.treasury_vault_config.to_account_info(),
            },
        )
    }
}
```


## Take aways

Always check the correct account is used in the transfer and Impl of the Accounts

# Summary
[High] It's impossible to set the price feed data if any token uses a Chainlink oracle

Chainlink price result is saved as both the min and max price

```rust
let price = Decimals::try_from_price(
    *answer as u128,
    decimals,
    token_config.token_decimals(),
    token_config.precision(),
).map_err(|_| error!(CoreError::InvalidPriceFeedPrice))?;
Ok((
    round.slot,
    timestamp,
    Price {
@>        min: price,
@>        max: price,
    }
))

```
## What went Wrong and how to fix it next time

When the Price is set in the `price_map` the transaction will fail if any tokeb uses a chainlink oracle

```rust
// Assume the remaining accounts are arranged in the following way:
// [token_config, feed; tokens.len()] [..remaining]
for(idx,token) in tokens.iter().enumerate() {
    let feed = &remaining_accounts[idx];
    let token_config = map.get(token).ok_or_else( || error!(CoreError::NotFound))?;

    require!(token_config.is_enabled(), CoreError::TokenConfigDisabled);

    let oracle_price = OraclePrice::parse_from_feed_account(
        validator.clock(),
        token_config,
        chainlink,
        feed,
    )?;

    validator.validate_one(
        token_config,
        &oracle_pice.provider,
        oracle_price.oracle_ts,
        oracle_price.oracle_slot,
        &oracle_price.price,
    )?;

    self.primary.set(token, oracle_price.price, token_config.is_synthetic())?;

}
```

This is caused by a check that ensures that the max price must always be greater than the min price , which is always false in this case

```rust
pub(create) fn from_price(price: &gmsol_utils::Price , is_synthetic: bool) -> Result<Self> {
    // Validate price data
    require_eq!(
        price.min.decimal_multiplier,
        price.max.decimal_multiplier,
        CodeError::InvalidArgument
    );
    require_neq!(price.min.value , 0 , CoreError::InvalidArgument);
@>    require_gt!(price.max.value , price.min.value , CoreError::InvalidArgument);
}
```
This should always not to be true!!

## Take aways

```diff
    /// ...
    require_neq!(price.min.value , 0 , CoreError::InvalidArgument);
-   require_gt!(price.max.value , price.min.value , CoreError::InvalidArgument);

+   require_gte!(price.max.value , price.min.value , CoreError::InvalidArgument);
}
```

Greater than , equal is good for chainlink oracle price


# Summary
[High] Time-locked instruction will fail to execute due to insufficient signing

Every time-locked instruction is buffered with its `executor` and all its accounts necessary for execution where some of them can be signer . Additionally , the executor is always set as signer too.

However , on execution of the instruction , it is only signed by the executor wallet's signer seeds and not by the executor account which `is set as a signer`. Additionally , it is not signed by any of the account that were `specified as signer` when the instruction was created

Time-locked instruction will fail to execute blocking crucial protocol actions


## What went Wrong and how to fix it next time

It is recommended to revise the instruction signing mechanism

1. Set the executor's wallet instead of the executor as signer in the instruction
2. Ensure the invocation os signed by all accounts that were set as the signer when the instruction was created

## Take aways
Check the Signer field for the instruction do not effect the other functionality of the Protocol


# Summary
[High] Wrong funcding factor on markets with adaptive funding

When a market updates its funding state , the calculation of the funding factor may be wrong on a market with adaptive funding (i.e when `increase_factor_per_second > 0`)

The issue is that when the increase factor is positive , the function that calculates the funding factor returns zero  even if it should not 

This would result in a zero funding factor as the `funding_value` will be multiplied by zero (Which will be wrong when a open interest and or short interest are not zero)


## What went Wrong and how to fix it next time
```diff
let funding_increase_factor_per_second = self.market.funding_fee_params()?.increase_factor_per_second();
- if diff_value.is_zero() {
+ if diff_value.is_zero() && funding_increase_factor_per_second.is_zero() {
    return Ok((Zero::zero() , true , Zero::zero()));
}
```

## Take aways

Always check what we are returning after the Calculation 

# Summary
[High] Missing Feed ID verification in case of SwitchBoard

In case of Switchboard being the price provider , the Provided feed account is never verified against the stored feed ID of the token Configuration , neither in the `parse_from_feed_account` function nor within the subsequent call to `Switchboard::check_and_get_price`

When `set_prices_from_remaining_accounts` is invoked , any valid Switchboard price feed , given in remaining_accounts, could be usd for any token having Switchboard as the configured provider which can lead to severe mispricing

## What went Wrong and how to fix it next time
It is recommeded to also implement a feed ID Verification in case of SwitchBoard being the Price Provider

## Take aways
Check for price feed is Correctly implemented or not