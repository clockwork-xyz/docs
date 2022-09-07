# Recurring payments

## Summary

* Executes an SPL token transfer on a user-defined schedule.
* **This example is open-source and free-to-fork on Github.**

{% embed url="https://github.com/clockwork-xyz/examples/tree/main/payments" %}

## Accounts

{% tabs %}
{% tab title="Payment" %}
The `Payment` account holds onto basic information such as who the sender and recipient are, the mint, and the escrowed balance. Note the payment account's PDA seeds consist of the sender, recipient, and mint properties. This allows our program to create a new payment for every (sender, recipient, mint) tuple.&#x20;

```rust
use {
    anchor_lang::{prelude::*, AnchorDeserialize},
    std::convert::TryFrom,
};

pub const SEED_PAYMENT: &[u8] = b"payment";

/**
 * Payment
 */

#[account]
#[derive(Debug)]
pub struct Payment {
    pub sender: Pubkey,
    pub recipient: Pubkey,
    pub mint: Pubkey,
    pub balance: u64,
    pub disbursement_amount: u64,
    pub schedule: String,
}

impl Payment {
    pub fn pubkey(sender: Pubkey, recipient: Pubkey, mint: Pubkey) -> Pubkey {
        Pubkey::find_program_address(
            &[
                SEED_PAYMENT,
                sender.as_ref(),
                recipient.as_ref(),
                mint.as_ref(),
            ],
            &crate::ID,
        )
        .0
    }
}

pub trait PaymentAccount {
    fn new(
        &mut self,
        sender: Pubkey,
        recipient: Pubkey,
        mint: Pubkey,
        balance: u64,
        disbursement_amount: u64,
        schedule: String,
    ) -> Result<()>;
}

impl PaymentAccount for Account<'_, Payment> {
    fn new(
        &mut self,
        sender: Pubkey,
        recipient: Pubkey,
        mint: Pubkey,
        balance: u64,
        disbursement_amount: u64,
        schedule: String,
    ) -> Result<()> {
        self.sender = sender;
        self.recipient = recipient;
        self.mint = mint;
        self.balance = balance;
        self.disbursement_amount = disbursement_amount;
        self.schedule = schedule;
        Ok(())
    }
}
```
{% endtab %}
{% endtabs %}

## Instructions

{% tabs %}
{% tab title="CreatePayment" %}
This instruction initializes a `Payment` account and then creates a queue to invoke `DisbursePayment` according to the user-defined scheduler.&#x20;

```rust
use {
    crate::state::*,
    anchor_lang::{
        prelude::*,
        solana_program::{
            instruction::Instruction, system_program, sysvar,
        },
    },
    anchor_spl::{
        associated_token::{self, AssociatedToken},
        token::{Mint, TokenAccount},
    },
    clockwork_crank::{
        program::ClockworkCrank,
        state::{Trigger, SEED_QUEUE},
    },
    std::mem::size_of,
};

#[derive(Accounts)]
#[instruction(disbursement_amount: u64, schedule: String)]
pub struct CreatePayment<'info> {
    #[account(address = anchor_spl::associated_token::ID)]
    pub associated_token_program: Program<'info, AssociatedToken>,

    #[account(address = clockwork_crank::ID)]
    pub clockwork_program: Program<'info, ClockworkCrank>,

    #[account(
        init,
        payer = sender,
        associated_token::authority = payment,
        associated_token::mint = mint,
    )]
    pub escrow: Account<'info, TokenAccount>,

    pub mint: Account<'info, Mint>,

    #[account(
        init,
        payer = sender,
        seeds = [
            SEED_PAYMENT, 
            sender.key().as_ref(), 
            recipient.key().as_ref(), 
            mint.key().as_ref()
        ],
        bump,
        space = 8 + size_of::<Payment>(),
    )]
    pub payment: Account<'info, Payment>,

    #[account(
        seeds = [
            SEED_QUEUE, 
            payment.key().as_ref(), 
            "payment".as_bytes()
        ], 
        seeds::program = clockwork_crank::ID,
        bump
    )]
    pub payment_queue: SystemAccount<'info>,

    /// CHECK: The recipient can be any valid address
    #[account()]
    pub recipient: AccountInfo<'info>,

    #[account(
        associated_token::authority = recipient,
        associated_token::mint = mint,
    )]
    pub recipient_token_account: Account<'info, TokenAccount>,

    #[account(address = sysvar::rent::ID)]
    pub rent: Sysvar<'info, Rent>,

    #[account(mut)]
    pub sender: Signer<'info>,

    #[account(address = system_program::ID)]
    pub system_program: Program<'info, System>,

    #[account(address = anchor_spl::token::ID)]
    pub token_program: Program<'info, anchor_spl::token::Token>,
}

pub fn handler<'info>(
    ctx: Context<'_, '_, '_, 'info, CreatePayment<'info>>,
    disbursement_amount: u64,
    schedule: String,
) -> Result<()> {
    // Get accounts
    let clockwork_program = &ctx.accounts.clockwork_program;
    let escrow = &ctx.accounts.escrow;
    let mint = &ctx.accounts.mint;
    let payment = &mut ctx.accounts.payment;
    let payment_queue = &mut ctx.accounts.payment_queue;
    let recipient = &ctx.accounts.recipient;
    let recipient_token_account = &ctx.accounts.recipient_token_account;
    let sender = &ctx.accounts.sender;
    let system_program = &ctx.accounts.system_program;
    let token_program = &ctx.accounts.token_program;

    // Initialize payment account
    payment.new(
        sender.key(),
        recipient.key(),
        mint.key(),
        0,
        disbursement_amountx,
        schedule,
    )?;

    // Build disburse_payment ix
    let disburse_payment_ix = Instruction {
        program_id: crate::ID,
        accounts: vec![
            AccountMeta::new_readonly(associated_token::ID, false),
            AccountMeta::new(escrow.key(), false),
            AccountMeta::new_readonly(payment.mint, false),
            AccountMeta::new(payment.key(), false),
            AccountMeta::new_readonly(payment_queue.key(), true),
            AccountMeta::new_readonly(payment.recipient, false),
            AccountMeta::new(recipient_token_account.key(), false),
            AccountMeta::new_readonly(payment.sender, false),
            AccountMeta::new_readonly(token_program.key(), false),
        ],
        data: clockwork_crank::anchor::sighash("disburse_payment").into(),
    };

    // Create queue to automate disburse_payment ix
    let bump = *ctx.bumps.get("payment").unwrap();
    clockwork_crank::cpi::queue_create(
        CpiContext::new_with_signer(
            clockwork_program.to_account_info(),
            clockwork_crank::cpi::accounts::QueueCreate {
                authority: payment.to_account_info(),
                payer: sender.to_account_info(),
                queue: payment_queue.to_account_info(),
                system_program: system_program.to_account_info(),
            },
            &[&[
                SEED_PAYMENT,
                payment.sender.as_ref(),
                payment.recipient.as_ref(),
                payment.mint.as_ref(),
                &[bump],
            ]],
        ),
        disburse_payment_ix.into(),
        "payment".into(),
        Trigger::Cron {
            schedule: payment.schedule.to_string()
        },
    )?;

    Ok(())
}
```
{% endtab %}

{% tab title="DisbursePayment" %}
The `DisbursePayment` instruction simply invokes a token transfer from the escrow account (owned by the payment PDA) to the recipient's associated token account.

```rust
use {
    crate::state::*,
    anchor_lang::prelude::*,
    anchor_spl::{ 
        associated_token::AssociatedToken,
        token::{self, Mint, TokenAccount, Transfer}
    },
    clockwork_crank::state::{Queue, SEED_QUEUE, CrankResponse},
};

#[derive(Accounts)]
pub struct DisbursePayment<'info> {
    #[account(address = anchor_spl::associated_token::ID)]
    pub associated_token_program: Program<'info, AssociatedToken>,

    #[account(
        mut,
        associated_token::authority = payment,
        associated_token::mint = mint,
    )]
    pub escrow: Box<Account<'info, TokenAccount>>,

    #[account(address = payment.mint)]
    pub mint: Box<Account<'info, Mint>>,

    #[account(
        mut,
        seeds = [SEED_PAYMENT, payment.sender.key().as_ref(), payment.recipient.key().as_ref(), payment.mint.as_ref()],
        bump,
    )]
    pub payment: Box<Account<'info, Payment>>,

    #[account(
        signer, 
        seeds = [
            SEED_QUEUE, 
            payment.key().as_ref(), 
            "payment".as_bytes()
        ], 
        seeds::program = clockwork_crank::ID,
        bump,
    )]
    pub payment_queue: Box<Account<'info, Queue>>,

    #[account( 
        mut,
        associated_token::authority = payment.recipient,
        associated_token::mint = payment.mint,
    )]
    pub recipient_token_account: Box<Account<'info, TokenAccount>>,

    #[account(address = anchor_spl::token::ID)]
    pub token_program: Program<'info, anchor_spl::token::Token>,
}

pub fn handler(ctx: Context<'_, '_, '_, '_, DisbursePayment<'_>>) -> Result<CrankResponse> {
    // Get accounts
    let escrow = &ctx.accounts.escrow;
    let payment = &mut ctx.accounts.payment;
    let recipient_token_account = &ctx.accounts.recipient_token_account;
    let token_program = &ctx.accounts.token_program;

    // Update balance of payment account
    payment.balance = payment.balance.checked_sub(payment.disbursement_amount).unwrap();

    // Transfer from escrow to recipient's token account
    let bump = *ctx.bumps.get("payment").unwrap();
    token::transfer(
        CpiContext::new_with_signer(
            token_program.to_account_info(), 
            Transfer {
                from: escrow.to_account_info(),
                to: recipient_token_account.to_account_info(),
                authority: payment.to_account_info(),
            },             
            &[&[SEED_PAYMENT, payment.sender.as_ref(), payment.recipient.as_ref(), payment.mint.as_ref(), &[bump]]]),
        payment.disbursement_amount,
    )?;
        
    Ok(CrankResponse{ next_instruction: None })
}
```
{% endtab %}
{% endtabs %}

<details>

<summary>DisbursePayment</summary>



</details>
