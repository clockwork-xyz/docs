# Serum crank

## Summary

* Indefinitely processes open orders on a permissioned Serum market.
* **This example is open-source and free-to-fork on Github.**

{% embed url="https://github.com/clockwork-xyz/examples/tree/main/serum_crank" %}

## Accounts

{% tabs %}
{% tab title="Crank" %}
The `Crank` account holds onto basic information such as a  current open orders, a given market, the event queue associated with said market, mint vaults, and the limit of orders to crank per invocation. Note the Crank account's PDA seeds consist of just the market. This allows our program to create a new crank account for every market.&#x20;

```rust
use {
    anchor_lang::{prelude::*, AnchorDeserialize},
    std::convert::TryFrom,
};

pub const SEED_CRANK: &[u8] = b"crank";

/**
 * Crank
 */

#[account]
#[derive(Debug)]
pub struct Crank {
    pub open_orders: Vec<Pubkey>,
    pub market: Pubkey,
    pub event_queue: Pubkey,
    pub mint_a_vault: Pubkey,
    pub mint_b_vault: Pubkey,
    pub limit: u16,
}

impl Crank {
    pub fn pubkey(market: Pubkey) -> Pubkey {
        Pubkey::find_program_address(&[SEED_CRANK, market.as_ref()], &crate::ID).0
    }
}

impl TryFrom<Vec<u8>> for Crank {
    type Error = Error;
    fn try_from(data: Vec<u8>) -> std::result::Result<Self, Self::Error> {
        Crank::try_deserialize(&mut data.as_slice())
    }
}

/**
 * CrankAccount
 */

pub trait CrankAccount {
    fn new(
        &mut self,
        market: Pubkey,
        event_queue: Pubkey,
        mint_a_vault: Pubkey,
        mint_b_vault: Pubkey,
        limit: u16,
    ) -> Result<()>;
}

impl CrankAccount for Account<'_, Crank> {
    fn new(
        &mut self,
        market: Pubkey,
        event_queue: Pubkey,
        mint_a_vault: Pubkey,
        mint_b_vault: Pubkey,
        limit: u16,
    ) -> Result<()> {
        self.open_orders = Vec::new();
        self.market = market;
        self.event_queue = event_queue;
        self.mint_a_vault = mint_a_vault;
        self.mint_b_vault = mint_b_vault;
        self.limit = limit;
        Ok(())
    }
}

```
{% endtab %}
{% endtabs %}

## Instructions

{% tabs %}
{% tab title="Initialize" %}
This instruction initializes a `Crank` account and then creates a queue to invoke the `ReadEvents` instruction according to the user-defined scheduler.&#x20;

```rust
use {
    crate::state::*,
    anchor_lang::{
        prelude::*,
        solana_program::{instruction::Instruction, system_program},
    },
    anchor_spl::{
        dex::serum_dex::state::Market,
        token::{Mint, TokenAccount},
    },
    clockwork_sdk::queue_program::{
        self,
        accounts::{Queue, Trigger},
        QueueProgram,
    },
    std::mem::size_of,
};

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(address = queue_program::ID)]
    pub clockwork_program: Program<'info, QueueProgram>,

    #[account(
        init,
        payer = payer,
        seeds = [SEED_CRANK, market.key().as_ref()],
        bump,
        space = 8 + size_of::<Crank>(),
    )]
    pub crank: Account<'info, Crank>,

    #[account(address = Queue::pubkey(crank.key(), "crank".into()))]
    pub crank_queue: SystemAccount<'info>,

    #[account(address = anchor_spl::dex::ID)]
    pub dex_program: Program<'info, anchor_spl::dex::Dex>,

    /// CHECK: this account is manually verified in handler
    #[account()]
    pub event_queue: AccountInfo<'info>,

    /// CHECK: this account is manually verified in handler
    #[account()]
    pub market: AccountInfo<'info>,

    #[account()]
    pub mint_a: Account<'info, Mint>,

    #[account(
        constraint = mint_a_vault.mint == mint_a.key()
    )]
    pub mint_a_vault: Account<'info, TokenAccount>,

    #[account()]
    pub mint_b: Account<'info, Mint>,

    #[account(
        constraint = mint_b_vault.mint == mint_b.key()
    )]
    pub mint_b_vault: Account<'info, TokenAccount>,

    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(address = system_program::ID)]
    pub system_program: Program<'info, System>,
}

pub fn handler<'info>(ctx: Context<'_, '_, '_, 'info, Initialize<'info>>) -> Result<()> {
    // Get accounts
    let clockwork_program = &ctx.accounts.clockwork_program;
    let crank = &mut ctx.accounts.crank;
    let crank_queue = &ctx.accounts.crank_queue;
    let dex_program = &ctx.accounts.dex_program;
    let event_queue = &ctx.accounts.event_queue;
    let market = &ctx.accounts.market;
    let mint_a_vault = &ctx.accounts.mint_a_vault;
    let mint_b_vault = &ctx.accounts.mint_b_vault;
    let payer = &ctx.accounts.payer;
    let system_program = &ctx.accounts.system_program;

    // validate market
    let market_data = Market::load(market, &dex_program.key()).unwrap();
    let val = unsafe { std::ptr::addr_of!(market_data.event_q).read_unaligned() };
    let market_event_queue = Pubkey::new(safe_transmute::to_bytes::transmute_one_to_bytes(
        core::convert::identity(&val),
    ));
    require_keys_eq!(event_queue.key(), market_event_queue);

    // initialize crank account
    crank.new(
        market.key(),
        event_queue.key(),
        mint_a_vault.key(),
        mint_b_vault.key(),
        10,
    )?;

    // get authorit bump
    let bump = *ctx.bumps.get("crank").unwrap();

    // define ix
    let read_events_ix = Instruction {
        program_id: crate::ID,
        accounts: vec![
            AccountMeta::new(crank.key(), false),
            AccountMeta::new(crank_queue.key(), true),
            AccountMeta::new_readonly(dex_program.key(), false),
            AccountMeta::new_readonly(event_queue.key(), false),
            AccountMeta::new_readonly(market.key(), false),
            AccountMeta::new_readonly(mint_a_vault.key(), false),
            AccountMeta::new_readonly(mint_b_vault.key(), false),
            AccountMeta::new(clockwork_sdk::queue_program::utils::PAYER_PUBKEY, true),
            AccountMeta::new_readonly(system_program.key(), false),
        ],
        data: clockwork_sdk::queue_program::utils::anchor_sighash("read_events").to_vec(),
    };

    // initialize queue
    clockwork_sdk::queue_program::cpi::queue_create(
        CpiContext::new_with_signer(
            clockwork_program.to_account_info(),
            clockwork_sdk::queue_program::cpi::accounts::QueueCreate {
                authority: crank.to_account_info(),
                payer: payer.to_account_info(),
                queue: crank_queue.to_account_info(),
                system_program: system_program.to_account_info(),
            },
            &[&[SEED_CRANK, crank.market.as_ref(), &[bump]]],
        ),
        "crank".into(),
        read_events_ix.into(),
        Trigger::Cron {
            schedule: "*/15 * * * * * *".into(),
            skippable: true,
        },
    )?;

    Ok(())
}

```
{% endtab %}

{% tab title="ReadEvents" %}
The `ReadEvents` instruction deserializes a market's event queue, writes the data to the `Crank` account, and returns a `CrankResponse` that updates the queue with a new instruction to then invoke `ConsumeEvents`

```rust
use {
    crate::state::*,
    anchor_lang::{
        system_program::{transfer, Transfer},
        prelude::*,
        solana_program::{system_program,instruction::Instruction},
    },
    anchor_spl::{dex::serum_dex::state::{strip_header, EventQueueHeader, Event, Queue as SerumDexQueue}, token::TokenAccount},
    clockwork_sdk::queue_program::{self, accounts::{CrankResponse, Queue, QueueAccount}}
};

#[derive(Accounts)]
pub struct ReadEvents<'info> {
    #[account(
        mut, 
        seeds = [SEED_CRANK, crank.market.as_ref()],
        bump, 
        has_one = event_queue,
        has_one = market,
        has_one = mint_a_vault,
        has_one = mint_b_vault,
    )]
    pub crank: Account<'info, Crank>,

    #[account(
        signer, 
        mut,
        address = crank_queue.pubkey(),
        constraint = crank_queue.id.eq("crank"),
    )]
    pub crank_queue: Account<'info, Queue>,

    #[account(address = anchor_spl::dex::ID)]
    pub dex_program: Program<'info, anchor_spl::dex::Dex>,

    /// CHECK: this account is validated against the crank account
    pub event_queue: AccountInfo<'info>,

    /// CHECK: this account is validated against the crank account
    pub market: AccountInfo<'info>,

    pub mint_a_vault: Account<'info, TokenAccount>,

    pub mint_b_vault: Account<'info, TokenAccount>,

    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(address = system_program::ID)]
    pub system_program: Program<'info, System>,
}

pub fn handler<'info>(ctx: Context<'_, '_, '_, 'info, ReadEvents<'info>>) -> Result<CrankResponse> {
    // Get accounts
    let crank = &mut ctx.accounts.crank;
    let crank_queue = &mut ctx.accounts.crank_queue;
    let dex_program = &ctx.accounts.dex_program;
    let event_queue = &ctx.accounts.event_queue;
    let market = &ctx.accounts.market;
    let mint_a_vault = &ctx.accounts.mint_a_vault;
    let mint_b_vault = &ctx.accounts.mint_b_vault;
    let payer = &mut ctx.accounts.payer;
    let system_program = &ctx.accounts.system_program;

    let mut next_ix_accounts = vec![
        AccountMeta::new_readonly(crank.key(), false),
        AccountMeta::new_readonly(crank_queue.key(), true),
        AccountMeta::new_readonly(dex_program.key(), false),
        AccountMeta::new(event_queue.key(), false),
        AccountMeta::new(market.key(), false),
        AccountMeta::new(mint_a_vault.key(), false),
        AccountMeta::new(mint_b_vault.key(), false),
        AccountMeta::new_readonly(system_program::ID, false),
    ];

    // deserialize event queue
    let mut open_orders = Vec::new();

    let (header, buf) = 
        strip_header::<EventQueueHeader, Event>(event_queue, false).unwrap();
    let events = SerumDexQueue::new(header, buf);
    for event in events.iter() {
        // <https://github.com/rust-lang/rust/issues/82523>
        let val = unsafe { std::ptr::addr_of!(event.owner).read_unaligned() };
        let owner = Pubkey::new(safe_transmute::to_bytes::transmute_one_to_bytes(core::convert::identity(&val)));
        open_orders.push(owner);
        next_ix_accounts.push(AccountMeta::new(owner, false));
    }
        
    // write event queue data to crank account
    crank.open_orders = open_orders;

    // realloc memory for crank's account
    let new_size = 8 + crank.try_to_vec()?.len();
    crank.to_account_info().realloc(new_size, false)?;

    // pay rent if more space has been allocated
    let minimum_rent = Rent::get().unwrap().minimum_balance(new_size);

    if minimum_rent > crank.to_account_info().lamports() {
        transfer(
            CpiContext::new(
                system_program.to_account_info(),
                Transfer {
                    from: payer.to_account_info(),
                    to: crank.to_account_info(),
                },
            ),
            minimum_rent.checked_sub(crank.to_account_info().lamports()).unwrap()
        )?;
    }  
    
    // return consume events ix
    Ok(CrankResponse { 
        next_instruction: Some(
            Instruction {
                program_id: crate::ID,
                accounts: next_ix_accounts,
                data: queue_program::utils::anchor_sighash("consume_events").into(),
            }
            .into()
        )
    }) 
}

```
{% endtab %}

{% tab title="ConsumeEvents" %}
The `ConsumeEvents` instruction reads the data from the `Crank` account to check if there are any available open orders and invokes serum's `ConsumeEvents` instruction to crank that markets open orders. Finally, this instructions returns a `CrankResponse` that updates the queue with a new instruction to then invoke `ReadEvents`

```rust
use {
    crate::state::*,
    anchor_lang::{
        prelude::*,
        solana_program::{system_program,instruction::Instruction},
    },
    anchor_lang::solana_program::program::invoke_signed,
    anchor_spl::{token::TokenAccount, dex::serum_dex},
    clockwork_sdk::queue_program::{self, accounts::{CrankResponse, Queue, QueueAccount}},
};

#[derive(Accounts)]
pub struct ConsumeEvents<'info> {
    #[account(
        address = Crank::pubkey(crank.market.key()),
        has_one = market,
        has_one = event_queue,
        has_one = mint_a_vault,
        has_one = mint_b_vault,
    )]
    pub crank: Account<'info, Crank>,

    #[account(
        signer, 
        address = crank_queue.pubkey(),
        constraint = crank_queue.id.eq("crank"),
    )]
    pub crank_queue: Account<'info, Queue>,
   
    #[account(address = anchor_spl::dex::ID)]
    pub dex_program: Program<'info, anchor_spl::dex::Dex>,

    /// CHECK: this account is validated against the crank account
    #[account(mut)]
    pub event_queue: AccountInfo<'info>,

    /// CHECK: this account is validated against the crank account
    #[account(mut)]
    pub market: AccountInfo<'info>,

    #[account(mut)] 
    pub mint_a_vault: Account<'info, TokenAccount>,

    #[account(mut)]
    pub mint_b_vault: Account<'info, TokenAccount>,

    #[account(address = system_program::ID)]
    pub system_program: Program<'info, System>,
}

pub fn handler<'info>(ctx: Context<'_, '_, '_, 'info, ConsumeEvents<'info>>) -> Result<CrankResponse> {
    // Get accounts
    let crank = &ctx.accounts.crank;
    let crank_queue = &ctx.accounts.crank_queue;
    let dex_program = &ctx.accounts.dex_program;
    let event_queue = &mut ctx.accounts.event_queue;
    let market = &mut ctx.accounts.market;
    let mint_a_vault = &mut ctx.accounts.mint_a_vault;
    let mint_b_vault = &mut ctx.accounts.mint_b_vault;

    // get crank bump
    let bump = *ctx.bumps.get("crank").unwrap();
    
    // coerce open orders type
    let open_orders_accounts = crank.open_orders.iter().map(|pk| pk).collect::<Vec<&Pubkey>>();
    
    if open_orders_accounts.len() > 0 {   
        // derive consume events ix
        let consume_events_ix = 
            serum_dex::instruction::consume_events(
                &dex_program.key(),
                open_orders_accounts,
                &market.key(), 
                &event_queue.key(),
                &mint_b_vault.key(), 
                &mint_a_vault.key(), 
                crank.limit
            ).unwrap();

        // construct account infos vec
        let mut cpi_account_infos = ctx.remaining_accounts.clone().to_vec(); 
        let mut rest_account_infos = vec![
            market.to_account_info(), 
            event_queue.to_account_info(), 
            mint_a_vault.to_account_info(), 
            mint_b_vault.to_account_info(), 
            dex_program.to_account_info()
        ];
        cpi_account_infos.append(&mut rest_account_infos);

        // invoke crank events ix
        invoke_signed(&consume_events_ix, &cpi_account_infos,&[&[SEED_CRANK, crank.market.as_ref(), &[bump]]])?;
    }

    // return read events ix
    Ok(CrankResponse { 
        next_instruction: Some(
            Instruction {
                program_id: crate::ID,
                accounts: vec![
                    AccountMeta::new(crank.key(), false),
                    AccountMeta::new(crank_queue.key(), true),
                    AccountMeta::new_readonly(dex_program.key(), false),
                    AccountMeta::new_readonly(event_queue.key(), false),
                    AccountMeta::new_readonly(market.key(), false),
                    AccountMeta::new_readonly(mint_a_vault.key(), false),
                    AccountMeta::new_readonly(mint_b_vault.key(), false),
                    AccountMeta::new(queue_program::utils::PAYER_PUBKEY, true),
                    AccountMeta::new_readonly(system_program::ID, false),
                ],
                data: clockwork_sdk::queue_program::utils::anchor_sighash("read_events").into(),
            }
            .into()
        )
    }) 
}

```
{% endtab %}
{% endtabs %}
