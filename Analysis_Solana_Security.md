## Analysis of Solana Security 

Solana is Combination of Proof of Stake(PoS) and Proof Of History(PoH)

### Incidents are categorized into four different Categories 
1. Application Exploits (24)
2. Supply Chain Attacks(3)
3. Core Protocol Vulnerabilites(4)
4. Network Level Attacks(6)


###  Application Exploits
- Target Vulnerabilities in the software application, smart Contract or Protocol Logic built on top of Solana Blockchain

1. WormHole Bridge Exploit
Root Cause : A Signature verification flaw in Solana Side Smart Contract allowed an attacker to forge a valid Signature , bypassing Guardian Validation. This enabled the unauthorized minting of 120,000 (Weth) without depositing equivalent Ethereum Collateral 

```rust
pub fn load_current_index(data: &[u8]) -> u16 {
    let mut instr_fixed_data = [0u8;2];
    let len = data.len();
    instr_fixed_data.copy_from_slice(&data[len - 2..len]);
    u16::from_le_bytes(instr_fixed_data)
}

/// Load the current `Instruction` 's index in the Currently executing
/// Transactions
pub fn load_current_index_checked(
    instruction_sysvar_account_info: &AccountInfo,
) -> Result<(u16, ProgramError)> {
    if !check_id(instruction_sysvar_account_info.key) {
        return Err(ProgramError::UnsupportedSysvar);
    }  // this is Vulnerable To check the Account

    let instruction_sysvar = instruction_sysvar_account_info.try_borrow_data()?;
    let mut instr_fixed_data = [0u8;2];
    let len = instruction_sysvar.len();
    instr_fixed_data.copy_from_slice(&instruction_sysvar[len - 2..len]);
    Ok(u16::from_le_bytes(instr_fixed_data));
}
```

Remediations: Enhanced Signature verifications with stricter input validation and improved account checks to prevent Spoofing Attacks

2. Cashio Exploit
Root Cause: A vulnerability in Cashio’s smart contract collateral validation allowed an attacker to mint 2 billion CASH tokens using fake accounts with worthless collateral. The flaw, due to missing validation of the mint field in the saber_swap.arrow account, enabled the attacker to bypass checks for Saber USDT-USDC LP tokens, exploiting an “infinite mint glitch.”

- Technical Reports

```rust
pub fn print_cash(ctx: Context<PrintCash>, deposit_amount: u64) -> Result<()> {
    ctx.accounts.print_cash(deposit_amount)
}

impl<'info> PrintCash<'info> {
    fn print_cash(&self,deposit_amount: u64) -> Result<()> {
        let current_balance = self.common.crate_collateral_tokens.amount;
        require!(unwrap_int!(current_balance.checked_add(deposit_amount)) <= self.common.collateral.hard_cap, CollateralHardCapHit);

        let swap: CashSwap = (&self.common.saber_swap).try_into()?;

        let print_amount = unwrap_int!(swap.calculate_cash_for_pool_tokens(deposit_amount));

        if print_amount == 0 {
            return Ok(())
        }
    }
}

```


3. Crema Finance Exploit

- Root Cause: A vulnerability in Crema Finance’s Concentrated Liquidity Market Maker (CLMM) allowed an attacker to create a fake tick account, bypassing owner verification. Using flash loans from Solend, the attacker manipulated transaction fee data to claim excessive fees, draining funds from multiple liquidity pools.

- Remediations: Enhanced tick account validation and owner checks to prevent fake data manipulation, alongside stricter flash loan protections

- Liquidity protocols must secure price tick data and transaction fee logic to prevent flash loan exploits


4. Audius Governance Exploit

- Root Cause: A vulnerability in Audius’ governance contract allowed an attacker to submit and execute malicious proposals, bypassing proper validation. The attacker reconfigured treasury permissions, transferring 18.5 million AUDIO tokens to their wallet.

- Remediations: Enhanced proposal validation, added timelocks for governance actions, and migrated to a new governance system with stricter access controls.

- Governance contracts require robust validation and delays to prevent unauthorized actions

5. Nirvana Finance Exploit
- Root Cause: Pricing Mechanism Vulnerability in Nirvana Finance Smart Contract using Flash Loan. By purchasing ANA tokens and manipulating the bonding curve, minted tokens at an inflated rate, draining $3.5 million in stablecoins.

- Remediations: No immediate contract fixes were implemented due to the shutdown. The relaunched Nirvana V2 (announced September 2024) introduced a “rising floor” price mechanism and protocol-owned liquidity to enhance stability

- Custom pricing mechanisms are vulnerable to flash loan attacks, requiring robust bonding curves, external oracles


6. Slope Mobile wallet Exploit
- Root Cause: Insecure handling of private key material in Slope’s mobile wallet application led to the leakage of users’ seed phrases. The app inadvertently transmitted encrypted seed phrases to Slope’s central logging server, where they were potentially intercepted or mishandled, allowing an attacker to access and drain affected wallets.

- Remediations: Slope implemented stricter data handling policies, removed seed phrase logging, and enhanced encryption practices to prevent future leaks.

- Wallet applications must prioritize secure key management and avoid transmitting sensitive data to centralized servers, highlighting the risks of custodial-like practices in non-custodial wallets.

7. OptiFi Lockup Bug

- Root Cause: A coding error during a program update led to the accidental use of the  `solana program close` command, permanently shutting down OptiFi’s mainnet and locking $661,000 in USDC within program-derived accounts (PDAs).

- Remediations: Implemented a peer-surveillance system requiring at least three team members to review deployments, aiming to prevent future coding errors.

- Non-malicious bugs can permanently lock funds in DeFi due to blockchain immutability, emphasizing the need for rigorous deployment processes and testing.

8. Mango Market Exploit

- Root Cause: The attacker manipulated mango Market Price Oracle By inflating the MNGO token through leveraged perpetual futures trades

- Remediations: Improved oracle security with external price feeds (e.g., Pyth, Chainlink) and implemented leverage limits to reduce manipulation risks.

- Low-liquidity tokens and oracle-dependent systems are vulnerable to economic manipulation, requiring robust price feeds and risk controls.

9. UXD Protocol Exploit

- Root Cause: UXD Protocol was indirectly impacted by the Mango Markets oracle manipulation exploit, where attacker Avraham Eisenberg inflated MNGO prices to drain $116 million. UXD had $19.9 million in USDC deposited in Mango’s lending pools, which were frozen during the attack

- Remediations: UXD reset its Asset Liability Management Module to restore functionality and planned to diversify away from Mango Markets to reduce single-point reliance

- Dependency on external DeFi protocols exposes stablecoins to third-party risks, necessitating diversified strategies and robust insurance funds.

10. Tulip Protocol Exploit

- Root cause: Alos Impacted By Mango Finance oracle manipulation Exploit, Tulip had $2.5 million in USDC and RAY strategy vaults deposited in Mango’s lending pools, which were frozen during the attack.

- Remediations: Tulip restricted vault deposits to its own lending pools and reevaluated risk exposure to external protocols to reduce dependency on platforms like Mango.

- Yield aggregators relying on third-party protocols face significant risks from external exploits, necessitating diversified strategies and robust risk management.

11. Save(Formerly , Solend Protocol) Exploit 

- Root Cause: Oracle Price Manipulation allowed attacker to over-Borrow against inflated collateral values, exploit outdated price feeds

- Remediations: Enhanced oracle validation with faster price feed updates and stricter collateral checks to prevent manipulation

- Accurate and timely oracle data is critical for lending protocols to prevent over-borrowing exploits, emphasizing robust price feed integration.

12. Raydium Exploit

- Root Cause: A Trojan horse attack compromised the private key of Raydium’s Pool Owner account, granting the attacker access to the V4 liquidity pool’s admin functions. The attacker used the withdrawPNL function to inflate and withdraw fees, draining funds from eight constant product liquidity pools.

- Remediations: Admin parameters were removed via a Squads multisig upgrade, and ownership was transferred to a hardware wallet. A compensation plan was later enacted using RAY buyback funds and team token

- Privileged account security is critical in DeFi; private key compromises can bypass smart contract protections, necessitating robust infrastructure security.

13. Cypher Protocol Exploit

- Root Cause : A vulnerability in Cypher's smart contract, likely in its margin or futures trading logic, allowed an attacker to steal 38,530 SOL and 123,184 USDC by exploiting unauthorized access to funds

14. SVT Token Exploit

- Root Cause - A Flash Loan Attack Exploit economic model in SVT transaction contract, allowing the attacker to manipulate token prices through repeated buying and selling operation , leveraging flash loan to amplify profits

- Remediations: Flash loan attacks exploit poorly designed economic models, requiring secure contract logic, oracle integration, and liquidity protections to prevent price manipulation.

15. Synthetify DAO Exploit

- Root Cause: An attacker exploited Synthetify’s inactive DAO by creating and voting on malicious governance proposals. They submitted ten proposals, nine harmless and one containing code to transfer $230,000 in USDC, mSOL, and stSOL to their address, using their own tokens to meet the voting quorum unnoticed.

- Lessons Learned: Inactive DAOs with pure token-based voting are vulnerable to governance attacks, requiring active monitoring, engagement incentives, and robust proposal scrutiny.

16. Thunder Terminal Exploit

- Root Cause: A compromised MongoDB connection URL, a third-party service vulnerability, allowed an attacker to access Thunder Terminal’s system, withdrawing 86.5 ETH and 439 SOL from user wallets via malicious approvals.

- Remediations: Enhanced security for third-party integrations, including stricter access controls and monitoring for MongoDB connections, to prevent similar compromises.

17. Saga DAO Incidents

- Root Cause : Saga DAO, fan club for solana saga phone had a security breach in Saga DAO’s multisig wallet, reportedly requiring only 1/12 wallet confirmations, allowed an attacker to drain approximately $60,000 in SOL from the treasury. The breach was linked to a compromised founder’s account, though some community members alleged insider involvement due to the low confirmation threshold.

- Remediations: Proper check to be included for multi-sig wallet 

18. Pump.fun Exploit

- Root Cause: A former Pump.fun employee, abusing privileged access to withdrawal authority, used flash loans on a Solana lending protocol to borrow SOL and manipulate token bonding curves. By pushing token values to 100%, the attacker accessed $1.9 million in bonding curve liquidity to repay the loans

- Remediations: Upgraded contract security to revoke unauthorized access and implemented stricter internal access controls to prevent future insider exploits.

- Insider threats and privileged access vulnerabilities can bypass DeFi safeguards, necessitating robust employee oversight and secure contract design.

19. Banana Gun Exploit

- Root Cause : A vulnerability in Banana Gun’s Telegram message oracle allowed an attacker to intercept messages and manually transfer 563 ETH ($1.4 million) from 11 user wallets during live trading sessions. The flaw affected both Ethereum and Solana bots, targeting experienced traders with notable social or trading presence. This flaw allowed the attacker to intercept messages and gain unauthorized access to user wallets. Once the attacker had access, they could initiate manual transfers while the bot was still in use by victims.

- Telegram-based bots are vulnerable to oracle and front-end exploits, requiring robust security measures and user verification to protect against targeted attacks on high-value traders.

20. DEXX Exploit:

- Root Cause - A private key leak in DEXX’s centralized custody model, due to improper key management, allowed an attacker to access and drain user wallets. The plaintext display of private keys during export_wallet requests on the official server facilitated the breach.

- Centralized custody of private keys in DeFi platforms poses significant risks, requiring secure key management and encrypted communication to protect user assets.

21. NoOnes PlatForm Exploit 

- Root cause: A vulnerability in NoOnes’ Solana cross-chain bridge allowed attackers to exploit the platform’s hot wallets, enabling hundreds of small transactions (each under $7,000) across Ethereum, TRON, Solana, and Binance Smart Chain.

- Remediations: Planned comprehensive penetration testing and enhanced bridge security to prevent future exploits, though specific fixes are undisclosed.

- Cross-chain bridges require robust security to prevent unauthorized access, highlighting the need for rigorous testing and monitoring in P2P platforms.

22. Loop Scale Exploit

- Root cause: An attacker exploited a vulnerability in Loopscale’s pricing mechanism for RateX-based collateral, enabling a series of undercollateralized loans that drained 5.7 million USDC and 1,200 SOL from the protocol’s USDC and SOL vaults.

- Remediations: Novel collateral pricing models in DeFi require rigorous testing and audits, as vulnerabilities can lead to significant losses, especially in newly launched protocols.

