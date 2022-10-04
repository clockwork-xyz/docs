# Dollar cost averaging



## Summary

* Executes a swap on Serum on a user-defined schedule.
* **This example is open-source and free-to-fork on Github.**

{% embed url="https://github.com/clockwork-xyz/examples/tree/main/investments" %}

## Accounts

{% tabs %}
{% tab title="Investment" %}
The `Investment` account holds onto basic information such as the payer, mint a, mint b, and the amount to swap per invocation. Note the Investment account's PDA seeds consist of the payer and both mints. This allows our program to create a new investment for every (payer, mint a, mint b) tuple.&#x20;

```rust
use {
    anchor_lang::{prelude::*, AnchorDeserialize},
    std::convert::TryFrom,
};

pub const SEED_INVESTMENT: &[u8] = b"investment";

/**
 * Investment
 */

#[account]
#[derive(Debug)]
pub struct Investment {
    pub payer: Pubkey,
    pub mint_a: Pubkey,
    pub mint_b: Pubkey,
    pub swap_amount: u64,
}

impl Investment {
    pub fn pubkey(payer: Pubkey, mint_a: Pubkey, mint_b: Pubkey) -> Pubkey {
        Pubkey::find_program_address(
            &[
                SEED_INVESTMENT,
                payer.as_ref(),
                mint_a.as_ref(),
                mint_b.as_ref(),
            ],
            &crate::ID,
        )
        .0
    }
}

impl TryFrom<Vec<u8>> for Investment {
    type Error = Error;
    fn try_from(data: Vec<u8>) -> std::result::Result<Self, Self::Error> {
        Investment::try_deserialize(&mut data.as_slice())
    }
}

/**
 * InvestmentAccount
 */

pub trait InvestmentAccount {
    fn new(
        &mut self,
        payer: Pubkey,
        mint_a: Pubkey,
        mint_b: Pubkey,
        swap_amount: u64,
    ) -> Result<()>;
}

impl InvestmentAccount for Account<'_, Investment> {
    fn new(
        &mut self,
        payer: Pubkey,
        mint_a: Pubkey,
        mint_b: Pubkey,
        swap_amount: u64,
    ) -> Result<()> {
        self.payer = payer;
        self.mint_a = mint_a;
        self.mint_b = mint_b;
        self.swap_amount = swap_amount;
        Ok(())
    }
}

```
{% endtab %}
{% endtabs %}

## Instructions

{% tabs %}
{% tab title="CreateInvestment" %}
This instruction initializes an `Investment` account and then creates a queue to invoke the `Swap` instruction according to the user-defined scheduler.&#x20;

```rust
use {
    crate::state::*,
    anchor_lang::{
        prelude::*,
        solana_program::{
            system_program, sysvar, instruction::Instruction
        },
    },
    anchor_spl::{token::{self, Mint, TokenAccount},associated_token::{self,AssociatedToken}},
    clockwork_sdk::queue_program::{self, QueueProgram, accounts::{Queue, Trigger}},
    std::mem::size_of,
};

#[derive(Accounts)]
#[instruction(swap_amount: u64)]
pub struct CreateInvestment<'info> {
    #[account(address = anchor_spl::associated_token::ID)]
    pub associated_token_program: Program<'info, AssociatedToken>,

    #[account(address = queue_program::ID)]
    pub clockwork_program: Program<'info, QueueProgram>,

    #[account(address = anchor_spl::dex::ID)]
    pub dex_program: Program<'info, anchor_spl::dex::Dex>,

    #[account(
        init,
        address = Investment::pubkey(payer.key(), mint_a.key(), mint_b.key()),
        payer = payer,
        space = 8 + size_of::<Investment>(),
    )]
    pub investment: Box<Account<'info, Investment>>,
    
    #[account(
        init,
        payer = payer, 
        associated_token::authority = investment, 
        associated_token::mint = mint_a
    )]
    pub investment_mint_a_token_account: Box<Account<'info, TokenAccount>>,

    #[account(
        init,
        payer = payer, 
        associated_token::authority = investment, 
        associated_token::mint = mint_b
    )]
    pub investment_mint_b_token_account: Box<Account<'info, TokenAccount>>,

    #[account(address = Queue::pubkey(investment.key(), "investment".into()))]
    pub investment_queue: SystemAccount<'info>,
    
    #[account()]
    pub mint_a: Box<Account<'info, Mint>>,

    #[account()]
    pub mint_b: Box<Account<'info, Mint>>,

    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init,
        payer = payer,
        associated_token::authority = payer,
        associated_token::mint = mint_a,
    )]
    pub payer_mint_a_token_account: Box<Account<'info, TokenAccount>>,
    
    #[account(
        init,
        payer = payer,
        associated_token::authority = payer,
        associated_token::mint = mint_b,
    )]
    pub payer_mint_b_token_account: Box<Account<'info, TokenAccount>>,

    #[account(address = sysvar::rent::ID)]
    pub rent: Sysvar<'info, Rent>,

    #[account(address = system_program::ID)]
    pub system_program: Program<'info, System>,

    #[account(address = anchor_spl::token::ID)]
    pub token_program: Program<'info, anchor_spl::token::Token>,
}

pub fn handler<'info>(ctx: Context<'_, '_, '_, 'info, CreateInvestment<'info>>, swap_amount: u64) -> Result<()> {
    // Get accounts
    let clockwork_program = &ctx.accounts.clockwork_program;
    let dex_program = &ctx.accounts.dex_program;
    let investment = &mut ctx.accounts.investment;
    let investment_mint_a_token_account = &ctx.accounts.investment_mint_a_token_account;
    let mint_a = &ctx.accounts.mint_a;
    let mint_b = &ctx.accounts.mint_b;
    let payer = &ctx.accounts.payer;
    let investment_queue = &mut ctx.accounts.investment_queue;
    let system_program = &ctx.accounts.system_program;

    // Get remaining accounts
    let market = ctx.remaining_accounts.get(0).unwrap();
    let mint_a_vault = ctx.remaining_accounts.get(1).unwrap();
    let mint_b_vault = ctx.remaining_accounts.get(2).unwrap();
    let request_queue = ctx.remaining_accounts.get(3).unwrap();
    let event_queue = ctx.remaining_accounts.get(4).unwrap();
    let market_bids = ctx.remaining_accounts.get(5).unwrap();
    let market_asks = ctx.remaining_accounts.get(6).unwrap();
    let open_orders = ctx.remaining_accounts.get(7).unwrap();

    // get investment bump
    let bump = *ctx.bumps.get("investment").unwrap();

    // initialize investment account
    investment.new(
    payer.key(), 
    mint_a.key(), 
    mint_b.key(),
    swap_amount
    )?;

    // create swap ix
    let swap_ix = Instruction {
        program_id: crate::ID,
        accounts: vec![
            AccountMeta::new_readonly(associated_token::ID, false),
            AccountMeta::new_readonly(dex_program.key(), false),
            AccountMeta::new_readonly(investment.key(), false),
            AccountMeta::new(investment_mint_a_token_account.key(), false),
            AccountMeta::new_readonly(investment_queue.key(), false),
            AccountMeta::new(queue_program::utils::PAYER_PUBKEY, true),
            AccountMeta::new_readonly(sysvar::rent::ID, false),
            AccountMeta::new_readonly(system_program::ID, false),
            AccountMeta::new_readonly(token::ID, false),
            // Extra Accounts
            AccountMeta::new(market.key(), false),
            AccountMeta::new(mint_a_vault.key(), false),
            AccountMeta::new(mint_b_vault.key(), false),
            AccountMeta::new(request_queue.key(), false),
            AccountMeta::new(event_queue.key(), false),
            AccountMeta::new(market_bids.key(), false),
            AccountMeta::new(market_asks.key(), false),
            AccountMeta::new(open_orders.key(), false),
        ],
        data: queue_program::utils::anchor_sighash("swap").into(),
    };

    // Create queue
    clockwork_sdk::queue_program::cpi::queue_create(
        CpiContext::new_with_signer(
            clockwork_program.to_account_info(),
            clockwork_sdk::queue_program::cpi::accounts::QueueCreate {
                authority: investment.to_account_info(),
                payer: payer.to_account_info(),
                queue: investment_queue.to_account_info(),
                system_program: system_program.to_account_info(),
            },
            &[&[SEED_INVESTMENT, investment.payer.as_ref(), investment.mint_a.as_ref(), investment.mint_b.as_ref(), &[bump]]],
        ),
        "investment".into(),
        swap_ix.into(),
        Trigger::Cron { 
            schedule: "*/15 * * * * * *".into(),
            skippable: true
        }    
    )?;

    Ok(())
}

```
{% endtab %}

{% tab title="Swap" %}
The `Swap` instruction places a new order on a given market every time an invocation occurs.

```rust
use {
    crate::state::{Investment, SEED_INVESTMENT},
    anchor_lang::{
        prelude::*,
        solana_program::{system_program, sysvar},
    },
    anchor_spl::{ 
        associated_token::AssociatedToken,
        dex::{
            serum_dex::{
                instruction::SelfTradeBehavior,
                matching::{OrderType, Side},
            },
            NewOrderV3,
        },
        token::{Token, TokenAccount},
    },
    clockwork_sdk::queue_program::accounts::{Queue, QueueAccount, CrankResponse},
    std::num::NonZeroU64,
};

#[derive(Accounts)]
pub struct Swap<'info> {
    #[account(address = anchor_spl::associated_token::ID)]
    pub associated_token_program: Program<'info, AssociatedToken>,

    #[account(address = anchor_spl::dex::ID)]
    pub dex_program: Program<'info, anchor_spl::dex::Dex>,

    #[account(address = Investment::pubkey(investment.payer, investment.mint_a, investment.mint_b))]
    pub investment: Box<Account<'info, Investment>>,

    #[account(
        mut, 
        associated_token::authority = investment, 
        associated_token::mint = investment.mint_a
    )]
    pub investment_mint_a_token_account: Box<Account<'info, TokenAccount>>,

    #[account(
      address = investment_queue.pubkey(),
      constraint = investment_queue.id.eq("investment")
    )]
    pub investment_queue: Box<Account<'info, Queue>>,

    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(address = sysvar::rent::ID)]
    pub rent: Sysvar<'info, Rent>,

    #[account(address = system_program::ID)]
    pub system_program: Program<'info, System>,

    #[account(address = anchor_spl::token::ID)]
    pub token_program: Program<'info, Token>,
}

pub fn handler<'info>(ctx: Context<'_, '_, '_, 'info, Swap<'info>>) -> Result<CrankResponse> {
    // get accounts
    let dex_program = &ctx.accounts.dex_program;
    let investment = &ctx.accounts.investment;
    let investment_mint_a_token_account = &mut ctx.accounts.investment_mint_a_token_account;
    let rent = &ctx.accounts.rent;
    let token_program = &ctx.accounts.token_program;

    // get remaining accounts
    let market = ctx.remaining_accounts.get(0).unwrap();
    let mint_a_vault = ctx.remaining_accounts.get(1).unwrap();
    let mint_b_vault = ctx.remaining_accounts.get(2).unwrap();
    let request_queue = ctx.remaining_accounts.get(3).unwrap();
    let event_queue = ctx.remaining_accounts.get(4).unwrap();
    let market_bids = ctx.remaining_accounts.get(5).unwrap();
    let market_asks = ctx.remaining_accounts.get(6).unwrap();
    let open_orders = ctx.remaining_accounts.get(7).unwrap();
    
    // get investment bump
    let bump = *ctx.bumps.get("investment").unwrap();

    // place order on serum dex
    anchor_spl::dex::new_order_v3(
        CpiContext::new_with_signer(
            dex_program.to_account_info(),
            NewOrderV3 {
                market: market.to_account_info(),
                coin_vault: mint_b_vault.to_account_info(),
                pc_vault: mint_a_vault.to_account_info(),
                request_queue: request_queue.to_account_info(),
                event_queue: event_queue.to_account_info(),
                market_bids: market_bids.to_account_info(),
                market_asks: market_asks.to_account_info(),
                open_orders: open_orders.to_account_info(),
                order_payer_token_account: investment_mint_a_token_account.to_account_info(),
                open_orders_authority: investment.to_account_info(),
                token_program: token_program.to_account_info(),
                rent: rent.to_account_info(),
            },
            &[&[
                SEED_INVESTMENT,
                investment.payer.as_ref(),
                investment.mint_a.as_ref(),
                investment.mint_b.as_ref(),
                &[bump],
            ]],
        ),
        Side::Bid,
        NonZeroU64::new(500).unwrap(),
        NonZeroU64::new(1_000).unwrap(),
        NonZeroU64::new(investment.swap_amount).unwrap(),
        SelfTradeBehavior::DecrementTake,
        OrderType::Limit,
        019269,
        std::u16::MAX,
    )?;

    // return None
    Ok(CrankResponse { 
        next_instruction: None
    }) 
}

```
{% endtab %}
{% endtabs %}
