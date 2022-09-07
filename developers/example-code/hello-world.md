---
description: >-
  This example program prints "Hello world" and the current timestamp every 60
  seconds.
---

# Hello world

## Accounts

<details>

<summary>Authority</summary>

This program uses a singleton `Authority` account. Once initialized, the authority account can sign for [CPIs](https://docs.solana.com/developing/programming-model/calling-between-programs) on behalf of our program.

```rust
use {
    anchor_lang::{prelude::*, AnchorDeserialize},
    std::convert::TryFrom,
};

pub const SEED_AUTHORITY: &[u8] = b"authority";

/**
 * Authority
 */

#[account]
#[derive(Debug)]
pub struct Authority {}

impl Authority {
    pub fn pubkey() -> Pubkey {
        Pubkey::find_program_address(&[SEED_AUTHORITY], &crate::ID).0
    }
}
```

</details>

## Instructions

<details>

<summary>Initialize</summary>

The `Initialize` instruction is only intended to be called once at the point of program initialization. It creates a Clockwork queue named `"hello"` and schedules it to fire every 15 seconds. Note we pass in the queue account as a `SystemAccount` and verify its PDA seeds follow the expected pattern.&#x20;

```rust
    crate::state::*,
    anchor_lang::{
        prelude::*,
        solana_program::{system_program, instruction::Instruction},
    },
    clockwork_crank::{
        program::ClockworkCrank,
        state::{Trigger, SEED_QUEUE},
    },
    std::mem::size_of,
};

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        seeds = [SEED_AUTHORITY],
        bump,
        payer = payer,
        space = 8 + size_of::<Authority>(),
    )]
    pub authority: Account<'info, Authority>,

    #[account(address = clockwork_crank::ID)]
    pub clockwork_program: Program<'info, ClockworkCrank>,

    #[account(
        seeds = [
            SEED_QUEUE, 
            authority.key().as_ref(), 
            "hello".as_bytes()
        ], 
        seeds::program = clockwork_crank::ID,
        bump
     )]
    pub hello_queue: SystemAccount<'info>,

    #[account(mut)]
    pub payer: Signer<'info>, 

    #[account(address = system_program::ID)]
    pub system_program: Program<'info, System>,
}

pub fn handler<'info>(ctx: Context<'_, '_, '_, 'info, Initialize<'info>>) -> Result<()> {
    // Get accounts
    let authority = &mut ctx.accounts.authority;
    let payer = &ctx.accounts.payer;
    let hello_queue = &ctx.accounts.hello_queue;
    let clockwork_program = &ctx.accounts.clockwork_program;
    let system_program = &ctx.accounts.system_program;

    // define ix
    let hello_clowckwork_ix = Instruction {
        program_id: crate::ID,
        accounts: vec![ 
            AccountMeta::new_readonly(authority.key(), false),
            AccountMeta::new_readonly(hello_queue.key(), true)
        ],
        data: clockwork_crank::anchor::sighash("hello_world").to_vec(),
    };

    // initialize queue
    let bump = *ctx.bumps.get("authority").unwrap();
    clockwork_crank::cpi::queue_create(
        CpiContext::new_with_signer(
            clockwork_program.to_account_info(),
            clockwork_crank::cpi::accounts::QueueCreate {
                authority: authority.to_account_info(),
                payer: payer.to_account_info(),
                queue: hello_queue.to_account_info(),
                system_program: system_program.to_account_info(),
            },
            &[&[SEED_AUTHORITY, &[bump]]],
        ),
        hello_clowckwork_ix.into(),
        "hello".into(),
        Trigger::Cron {
            schedule: "*/15 * * * * * *".into(),
        },
    )?;

    Ok(())
}
```

</details>

<details>

<summary>HelloWorld</summary>

The `HelloWorld` instruction is the target instruction we setup as the kickoff instruction for the queue. Here we simply verify the signer is our program's queue, named `"hello"`, and then print a log message.

```rust
use {
    crate::state::*,
    anchor_lang::prelude::*,
    clockwork_crank::state::{CrankResponse, SEED_QUEUE, Queue},
};

#[derive(Accounts)]
pub struct HelloWorld<'info> {
    #[account(seeds = [SEED_AUTHORITY], bump)]
    pub authority: Account<'info, Authority>,

    #[account(
        signer, 
        seeds = [
            SEED_QUEUE, 
            authority.key().as_ref(), 
            "hello".as_bytes()
        ], 
        seeds::program = clockwork_crank::ID,
        bump,
        has_one = authority
    )]
    pub hello_queue: Account<'info, Queue>,
}

pub fn handler(_ctx: Context<HelloWorld>) -> Result<CrankResponse> {
    msg!(
        "Hello world! The current time is: {}",
        Clock::get().unwrap().unix_timestamp
    );

    Ok(CrankResponse { next_instruction: None })
}
```

</details>
