## Attack vector 1

# Summary
[Critical] Incorrect Decimal Conversion Leads to Significant Fund loss

In the `usd_star_mint` and `usd_star_burn` function , the decimals scaling logic is incorrect , leading to a significant loss of funds for both users and the protcol . The issue arises in the following snippet:

```rust
if asset_info.decimals > usd_star_decimal {
    amount = amount.checked_mul( // @audit - this should be divided here 
        10u128.checked_pow((asset_info.decimals - usd_star_decimal) as u32).ok_or(ProgramError::ArithmeticOverflow)?;
    )
}

else if asset_info.decimals < usd_star_decimal{ // @audit - Here should be mul if the decimals are less 
    amount = amount.checked_div(10u128.checked_pow ((usd_star_decimal - asset_info.decimals)as u32).ok_or(programError::ArithmeticOverflow)?,).ok_or(ProgramError::ArithemticOverflow)?;
}
```
the `amount` value is initially scaled according to the asset's decimals . The conversion logic mistakely applies the scaling operation in the wrong direction . This result in incorrect token amounts being minted , causing substantial discrepancies in value

### Impact 
- User may recieve significantly fewer tokens than expected
- The Protocol may suffer a substantial loss of funds due to incorrect scaling .
- This could lead to arbitrage exploit and financial imbalances in the system

```diff
if asset_info.decimals > usd_star_decimal {
-    amount = amount.checked_mul( // @audit - this should be divided here
+    amount = amount.checked_div(
        10u128.checked_pow((asset_info.decimals - usd_star_decimal) as u32).ok_or(ProgramError::ArithmeticOverflow)?;
    )
}

else if asset_info.decimals < usd_star_decimal{ // @audit - Here should be mul if the decimals are less 
-    amount = amount.checked_div(10u128.checked_pow 
+   amount = amount.checked_mul(10u128.checked_pow 
    ((usd_star_decimal - asset_info.decimals)as u32).ok_or(programError::ArithmeticOverflow)?,).ok_or(ProgramError::ArithemticOverflow)?;
}
```

## What went Wrong and how to fix it next time
When using the multi token / assets in protcol always check from decimals used is correct or not ?? 

## Take aways
Decimals Check Should Be Checked ??
## Findings
Link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Perena_Audit_Final_report.pdf)


# Summary
[Critical] Token Decimals Are not Handled in `usd_prime_mint` and `usd_prime_burn` Leading to huge Loss of Funds for Both protocol

```rust
let mut amount = instruction_data.prime_amount.checked_mul(FEE_RATE_SCALE - bank.prime_fee_rate as u64).ok_or(ProgramError::ArithmeticOverflow)?.checked_div(FEE_RATE_SCALE).ok_or(ProgramError::ArithmeticOverflow)?;

amount = amount.checked_div(asset_price.price as u64).ok_or(ProgramError::ArithmeticOverflow)?;

amount = amount.checked_mul(10u64.checked_pow(asset_price.exponent.unsigned_abs()).ok_or(ProgramError::ArithmeticOverflow)?,).ok_or(ProgranError::ArithmeticOverflow)?;
```

## What went Wrong and how to fix it next time
Again Decimal Problem Should be Checked in Converting the assets 

## Take aways
Decimals Checks

## Findings
Link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Perena_Audit_Final_report.pdf)

# Summary
[Critical] Swap Function Vulnerable to self-Transfer Attack , Leading to Pool Drainage 

In the `swap_out()` function there is a critical Vulnerability in the transfer operations for the input asset . the function does not properly validate the the input accounts , which allows an attacker to manipulate the ttransaction to perform a self-transfer while still recieving token from the reserve

- There is no Validation for the account , leads to critical Vulnerability 

In the Swap_out() function accepts accepts user_provided account without validating that:
(Simple user can change From or to address and getting token from the bank on every swap call)

Recommendation: Implement proper account validation before performing transfer
1. Verify that `user_info` is the signer of the transaction
2. Verify the `input_user_data` and `output_user_ata` are valid ATAs owned by `user_info` (This should apply to all mint,burn , and swap function)
3. Add checks that the input and output accounts are distinct to prevent self-transfer (This should apply to all mint, burn ,and sap function)


## What went Wrong and how to fix it next time
Check the User Provided data Should be Correct or not 

## Take aways
Inmplement Check on the Input Data 

## Findings
Link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Perena_Audit_Final_report.pdf)


# Summary
[Critcial] Missing validation on USD-Star Mint Address Allows Unlimited USD-Prime Minting

An Exploit lack of validation exists in the `unstake` processor function , this allows attacker to bypass intended controls and mint unlimited USD-Prime tokens

- The code fails to check that the `usd_star_mint_info` provided by the user is the legitimate USD-Star token mint
1. Attacker create a worthless `trash` token they fully control
2. they apss this token mint as `user_star_mint_info` when calling `unstake()` function
3. The function proceeds to burn their worthless tokens and mint legitimate USD-Prime tokens in return
4. Attacker uses the newly minted USD_prime to swap to other tokens , efficiently draining all tokens

## What went Wrong and how to fix it next time
Add proper validation for the `usd_star_mint_info` parameter similar to how `usd_prime_mint_info` is validated in both `stake()` and `unstake()` function

## Take aways
proper user input data check should be validated 

## Findings
Link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Perena_Audit_Final_report.pdf)


# Summary
[High] Integer Overflow in `usd_prime_mint` Calculation will lead to making it impossible to mint USD tokens unde normal condition

The usd_prime_mint function multiplies `asset_amount` by `asset_price.price` and then divides by `10.pow(asset_price.exponent)`., the multiplication is performed in a u64 varaible, which is too small to hold large intermediate results before division, leading to an overflow error in normal minting scenarios

```
   amount = asset_amount * price / 10 ^ price.exponent
```

To minize the precision loss , multiplication is performed befor ethe division. since u64 can only store values up to 1.8 * 10 ^ 19 large taoken amounts and prices can cause an overflow before division occurs

## What went Wrong and how to fix it next time

Calculation in mint function can overflow in normal condition due to not correct type is implemented leads overflow 

```rust

let amount: u64 = instruction_data.asset_amount.checked_mul(asset_price.price as u64).ok_or(ProgramError::ArithmeticOverflow)?;

amount = amount.checked_div(10u64.checked_pow(asset_price.exponent.unsigned_abs()).ok_or(ProgramError::ArithmeticOverflow)?,).ok_or(ProgramError::ArithmeticOverflow)?;

```

- A normal token supply might to be 10^9 with 9 decimals
- Token amount would therefore have upto 18 decimals
- Price Feed often have 6-8 decimals places 
- Multiplication step result in 10 ^26 far exceeding the u64 limit, causing an overflow

This issue will cause the function to revert in the normal condition to mint USD token

## Take aways
```rust
let amount: u128 = (instruction_data.asset_amount as u128).checked_mul(asset_price.price as u128).ok_or(ProgramError::ArithmeticOverflow)?;

let divisior : u128 = 10u128.checked_pow(asset_price.exponent.unsigned_abs() as u32).ok_or(ProgramError::ArithmeticOverflow)?;

let amount = amount.checked_div(divisor).ok_or(ProgramResult::ArithmeticOverflow)?;

// Safely downcast to u64 if it fits , otherwise return an overflow error
let amount :u64 = amount.try_into().map_err(|_|ProgramError::ArithmeticOverflow)?;

```

Next time calculation is fit in its types also check for it..

## Findings
Link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Perena_Audit_Final_report.pdf)


# Summary
Division Before Multiplication Issue Will lead to Precision Loss and Rounding Down to Zero , Causing Loss of Funds To Users



## What went Wrong and how to fix it next time
For good Precison we have to multiply first then divide,
```rust
amount = amount.checked_div(asset_price.price as u64).ok_or(ProgramError::ArithmeticOverflow)?;
amount = amount.checked_mul(
    10u64.checked_pow(asset_price.exponent.unsigned_abs()).ok_or(ProgramError::ArithmeticOverflow)?,
).ok_or(ProgramError::ArithmeticOverflow)?;
```
## Take aways

```rust
amount = amount.checked_mul(10u64.checked_pow(asset_price.exponent.unsigned_abs()).ok_or(ProgramError::ArithmeticOverflow)?,).ok_or(ProgramError::Arithmeticverflow)?;
amount = amount.checked_div(asset_price.price as u64).ok_or(ProgramError::ArithmeticOverflow)?;
```
## Findings
Link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Perena_Audit_Final_report.pdf)

# Summary
[high]
## What went Wrong and how to fix it next time
## Take aways
## Findings
