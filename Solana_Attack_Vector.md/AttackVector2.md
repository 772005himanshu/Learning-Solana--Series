## Attack Vector / Audit Report -2 

# Summary
[High] Incorrect Comparison in `buy_token` Leading to Graduation Threshold Violation

In the buy_token function , the comparison between `next_buy_amount` and `max_buy_amount_with_fees ` is incorrect.the issue arises because `max_buy_amount_with_fees` represent the remaining tokens in the bonding curev after adding fees , while `net_buy_amount` is the user;s token amount after subtracting fees

This mismatch can cause the function to allow purchase beyond the graduation threshold, leading to users paying more for fewer token at an inflated fees

## What went Wrong and how to fix it next time

```rust
if net_buy_amount <= max_buy_amount_with_fees {
    actual_buy_amount = net_buy_amount; // amount after taking fee
    actual_swap_fee = swap_fee;
}else {
    actual_buy_amount = max_buy_amount_without_fees; // amount without taking fee

    actual_swap_fee = max_buy_amount_with_fees.checked_sub(max_buy_amount_without_fees).unwrap();
};
```
## Take aways

```diff
- if net_buy_amount <= max_buy_amount_with_fees {
+ if net_buy_amount < max_buy_amount_without_fees {
    actual_buy_amount = net_buy_amount; // amount after taking fee
    actual_swap_fee = swap_fee;
}else {
    actual_buy_amount = max_buy_amount_without_fees; // amount without taking fee

    actual_swap_fee = max_buy_amount_with_fees.checked_sub(max_buy_amount_without_fees).unwrap();
};
```

## Findings
link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Boop_Audit_Final_Report.pdf)

# Summary
[Medium] Slippage Protection ByPass in `buy_token`

In the `buy_token` and `sell_token` function , users can specify a slippage paarmeter to protect their traddes from the executing at unfavorable prices.This is neforced by checking the trade reaulting amount against the users slippage protection

## What went Wrong and how to fix it next time
Slippage Protection checks is not implemented in the functions
```rust
actual_buy_amount = max_buy_amount_without_fees;

actual_swap_fee = max_buy_amount_with_fees.checked_sub(max_buy_amount_without_fees).unwrap();
```

## Take aways
- the Trade can execute at a price worse than what the user intended.
- users are exposed to price slippage beyond their specified tolerance
- This result in a loss of funds , as trades may complete at unpreffered or unfair prices

Recommendation:
To maintain slippage protection , a slippage factor should be calculated based in the original input amounts

## Findings
link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Boop_Audit_Final_Report.pdf)


# Summary
[low] incorrect Macro Usage for Pubkey Comparison

The `require_neq!` macro is used to ensure two non-Pubkey values are not equal.
It is incorrectly used to compare two public keys:

```rust
require_neq!(creator, Pubkey::deafault(), ErrorCode::CreatorIsNotProvided);
```

Use `require_keys_neq!()` macro for comparing two public keys.

## What went Wrong and how to fix it next time

```rust
require_keys_neq!(creator, Pubkey::deafault(), ErrorCode::CreatorIsNotProvided);
```

## Take aways
In rust different Require statement is used to comapare diffenerent type check them next time


## Findings
link(https://github.com/Frankcastleauditor/public-audits/blob/main/reports/Boop_Audit_Final_Report.pdf)

