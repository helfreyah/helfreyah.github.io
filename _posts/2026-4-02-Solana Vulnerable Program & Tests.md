---
layout: post
title:  Solana Vulnerable Program & Tests
categories: [WEB3 Security]
---


Hi all,

A couple months ago i started to learn Solana Program Security. Solana web site and github repos effectively teaching how to write Solana Program. Also, public audit reports best way to learn program security. Reading reports and source codes most important way to learn vulnerabilites and fixing methods. 

So, i developed vulnerable solana program with these vulnerabilities;

- [1 - Underflow](#1-underflow)
- [2 - Overflow](#2-overflow)
- [3 -Missing Signer Check](#3-missing-signer-check)
- [4 -Missing Ownership](#4-missing-ownership)
<div style="page-break-before: always;"></div>


Also, I added test files. You can check source code and tests. Please inform me if you can see mistakes or you can reach me for recommendations.

Source code & tests : https://github.com/thereyahc/Vulnerable-Solana-Program




# 1 - Underflow

This test demonstrates an integer underflow vulnerability in the **TransactionUnderflow** instruction of the vault program. The vault's internal balance tracker is initialized to 500 lamports, while an attacker attempts to withdraw 1,000 lamports more than the recorded balance. Because the program neither validates that the balance is sufficient before proceeding nor uses a checked arithmetic operation, it calls **wrapping_sub(1000)** on a u64 value of 500, which silently wraps around to "18,446,744,073,709,551,116" (almost u64::MAX - 499). The real lamport transfer still executes successfully, meaning the attacker receives the funds, and the vault's tracker now shows an astronomically large balance effectively making the program believe the vault has near-unlimited funds available for future withdrawals. The fix is straightforward: validate **vault_data.balance >= amount** before any transfer and replace **wrapping_sub** with **checked_sub**, which returns an error instead of wrapping on underflow.
```Rust
// "vault_data.balance" is should be controlled
    // recommendation: if vault_data.balance < amount { return Err(ProgramError::InsufficientFunds); }

    // recommendation: checked_sub().ok_or(ProgramError::InvalidArgument)?
    vault_data.balance = vault_data.balance.wrapping_sub(amount);

    let serialized = vault_data
        .try_to_vec()
        .map_err(|_| ProgramError::InvalidAccountData)?;
    data.copy_from_slice(&serialized);
    drop(data);

    **vault.try_borrow_mut_lamports()? -= amount;
    **destination.try_borrow_mut_lamports()? += amount;
```

# 2 - Overflow
This test demonstrates an integer overflow vulnerability in the **DepositOverflow** instruction. The vault's balance tracker is intentionally set to **u64::MAX - 100**, which is the maximum value a 64-bit unsigned integer can hold minus 100. When a deposit of 200 lamports is made, the program uses wrapping_add instead of checked_add, so instead of rejecting the operation, it silently wraps around: **(u64::MAX - 100) + 200** becomes just 99. The real lamport transfer goes through correctly  the vault actually receives 200 lamports but the internal balance tracker now shows only 99, completely misrepresenting the vault's true state. This kind of discrepancy between actual lamports and the tracked balance can be exploited to confuse accounting logic, bypass withdrawal limits, or manipulate any feature that relies on the stored balance field. The fix is simply replacing **wrapping_add** with **checked_add**, which returns an error when the result would overflow instead of silently producing a wrong value.

```Rust
 // recommendation: checked_add().ok_or(ProgramError::InvalidArgument)?
    vault_data.balance = vault_data.balance.wrapping_add(amount);

    let serialized = vault_data
        .try_to_vec()
        .map_err(|_| ProgramError::InvalidAccountData)?;
    data.copy_from_slice(&serialized);

    println!(
        " Overflow vulnerability. Deposited {}. New balance: {}",
        amount, vault_data.balance
    );
    Ok(())
```

# 3 - Missing Signer Check

This test demonstrates a missing signer check vulnerability in the Initialize instruction. The attack scenario is simple: a victim_pubkey is created with no associated private key  just a raw public address  and an attacker submits a transaction that initializes a vault with the victim's address as the authority, without the victim ever signing anything. Because the program never checks authority.is_signer, it happily writes the victim's public key into the vault's authority field and returns success. After the transaction, the vault officially belongs to the victim on paper, but since the attacker controls the setup, this can be abused to front-run account initialization, grief users by locking them out of their own vaults before they can initialize them, or register arbitrary accounts under someone else's identity. The fix is a single guard at the top of the initialize function: **if !authority.is_signer,** reject the transaction with **MissingRequiredSignature.**

```Rust
fn initialize(_program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let account_iter = &mut accounts.iter();
    let vault = next_account_info(account_iter)?;
    let authority = next_account_info(account_iter)?;

    // Missing Signer Check
    // Anyone can initialize a vault on behalf of any public key without that key's 	 authorization,
    // since the program never verifies that the authority account signed the transaction.

    // if !authority.is_signer { return Err(ProgramError::MissingRequiredSignature); }

    let vault_data = Vault {
        authority: *authority.key,
        balance: 0,
    };
```


# 4 - Missing Ownership
This test demonstrates a missing ownership check vulnerability in the Deposit instruction. The attacker creates a fake vault account whose data is crafted to look exactly like a legitimate Vault struct same layout, same field sizes but it is an account the attacker controls rather than one legitimately initialized through the program. When the attacker calls Deposit and passes this fake vault as the target account, the program skips the one check that would catch this: verifying that vault.owner program_id in a meaningful way. The deposit goes through, 1,000,000 lamports are transferred in, and the fake vault's balance tracker is updated just like a real one. In a real-world scenario this can be used to redirect deposits into attacker-controlled accounts, manipulate accounting records, or drain funds by later withdrawing from a vault the program believes is legitimate but the attacker fully controls. The fix is a single owner check at the start of the deposit function: **if vault.owner != program_id**, reject the transaction with **IncorrectProgramId**.

```Rust
fn deposit(_program_id: &Pubkey, accounts: &[AccountInfo], amount: u64) -> ProgramResult {
    let account_iter = &mut accounts.iter();
    let vault = next_account_info(account_iter)?;
    let depositor = next_account_info(account_iter)?;
    let system_program = next_account_info(account_iter)?;

    // Missing Ownership Check
    // An attacker can pass a fake account with matching data layout instead of a legitimate vault,
    // causing the program to accept and process it as if it were a real program-owned account.
    // if vault.owner != program_id { return Err(ProgramError::IncorrectProgramId); }

    invoke(
        &system_instruction::transfer(depositor.key, vault.key, amount),
        &[depositor.clone(), vault.clone(), system_program.clone()],
    )?;

```
