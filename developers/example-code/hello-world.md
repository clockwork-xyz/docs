# Hello world

## Summary

* Prints "Hello world" and the current timestamp every 60 seconds.
* **This example is open-source and free-to-fork on Github.**

{% embed url="https://github.com/clockwork-xyz/examples/tree/main/hello_clockwork" %}

## Accounts

{% tabs %}
{% tab title="Authority" %}
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

impl TryFrom<Vec<u8>> for Authority {
    type Error = Error;
    fn try_from(data: Vec<u8>) -> std::result::Result<Self, Self::Error> {
        Authority::try_deserialize(&mut data.as_slice())
    }
}

```
{% endtab %}
{% endtabs %}

## Instructions

{% tabs %}
{% tab title="Initialize" %}
The `Initialize` instruction is only intended to be called once at the point of program initialization. It creates a Clockwork queue named `"hello"` and schedules it to fire every 15 seconds. Note we pass in the queue account as a `SystemAccount` and verify its PDA seeds follow the expected pattern.&#x20;

```rust
use {
    crate::state::*,
    anchor_lang::{
        prelude::*,
        solana_program::{instruction::Instruction, system_program},
    },
    clockwork_sdk::queue_program::{
        self,
        accounts::{Queue, Trigger},
    },
    std::mem::size_of,
};

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        address = Authority::pubkey(),
        payer = payer,
        space = 8 + size_of::<Authority>(),
    )]
    pub authority: Account<'info, Authority>,

    #[account(address = queue_program::ID)]
    pub clockwork_program: Program<'info, clockwork_sdk::queue_program::QueueProgram>,

    #[account(address = Queue::pubkey(authority.key(), "hello".into()))]
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
    let hello_clockwork_ix = Instruction {
        program_id: crate::ID,
        accounts: vec![
            AccountMeta::new_readonly(authority.key(), false),
            AccountMeta::new_readonly(hello_queue.key(), true),
        ],
        data: clockwork_sdk::queue_program::utils::anchor_sighash("hello_world").to_vec(),
    };

    // initialize queue
    let bump = *ctx.bumps.get("authority").unwrap();
    clockwork_sdk::queue_program::cpi::queue_create(
        CpiContext::new_with_signer(
            clockwork_program.to_account_info(),
            clockwork_sdk::queue_program::cpi::accounts::QueueCreate {
                authority: authority.to_account_info(),
                payer: payer.to_account_info(),
                queue: hello_queue.to_account_info(),
                system_program: system_program.to_account_info(),
            },
            &[&[SEED_AUTHORITY, &[bump]]],
        ),
        "hello".into(),
        hello_clockwork_ix.into(),
        Trigger::Cron {
            schedule: "*/15 * * * * * *".into(),
            skippable: true,
        },
    )?;

    Ok(())
}
```
{% endtab %}

{% tab title="HelloWorld" %}
The `HelloWorld` instruction is the target instruction we setup as the kickoff instruction for the queue. Here we simply verify the signer is our program's queue, named `"hello"`, and then print a log message.

```rust
use {
    crate::state::*,
    anchor_lang::prelude::*,
    clockwork_sdk::queue_program::accounts::{CrankResponse, Queue, QueueAccount},
};

#[derive(Accounts)]
pub struct HelloWorld<'info> {
    #[account(address = Authority::pubkey())]
    pub authority: Account<'info, Authority>,

    #[account(
        address = hello_queue.pubkey(),
        constraint = hello_queue.id.eq("hello"),
        has_one = authority,
        signer, 
    )]
    pub hello_queue: Account<'info, Queue>,
}

pub fn handler(_ctx: Context<HelloWorld>) -> Result<CrankResponse> {
    msg!(
        "Hello world! The current time is: {}",
        Clock::get().unwrap().unix_timestamp
    );

    Ok(CrankResponse {
        next_instruction: None,
    })
}
```
{% endtab %}
{% endtabs %}
