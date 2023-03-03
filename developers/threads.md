# Threads

## Account

The thread account is an automation primitive for Solana. It tracks the state necessary to manage and execute a series of instructions.

```rust
pub struct Thread {
    /// The owner of this thread.
    pub authority: Pubkey,
    /// The bump, used for PDA validation.
    pub bump: u8,
    /// The cluster clock at the moment the thread was created.
    pub created_at: ClockData,
    /// The context of the thread's current execution state.
    pub exec_context: Option<ExecContext>,
    /// The number of lamports to payout to workers per execution.
    pub fee: u64,
    /// The id of the thread, given by the authority.
    pub id: Vec<u8>,
    /// The instructions to be executed.
    pub instructions: Vec<SerializableInstruction>,
    /// The name of the thread.
    pub name: String,
    /// The next instruction to be executed.
    pub next_instruction: Option<SerializableInstruction>,
    /// Whether or not the thread is currently paused.
    pub paused: bool,
    /// The maximum number of execs allowed per slot.
    pub rate_limit: u64,
    /// The triggering event to kickoff a thread.
    pub trigger: Trigger,

}
```

## Public address

The public address of every thread is derived deterministically from its `authority` and `id`. These properties are immutable and may never change throughout the lifetime of the thread. To verify a Clockwork thread account in your programs, you can use the `pubkey()` helper function provided by the SDK:

```rust
#[account(
    address = thread.pubkey(),
    contraint = thread.id.eq("my_thread"),
    has_one = authority,
)]
pub thread: Account<'info, Thread>
```

## Authority

On Solana, the term “authority” is often used by convention to refer to the user-space owner of a particular account. For Clockwork, a thread's authority is its creator – the account which signed the transaction to originally create the thread. A thread's authority has the following permissions:

* Pause and resume the thread.
* Update the thread’s `trigger` and `instructions`.
* Withdraw from the thread’s balance.
* Close the thread account.

An authority may be any valid public address (i.e. a wallet pubkey or PDA). When the authority is a PDA, we occasionally refer to this as a “program authority” since the account is managed by a program. If you are unfamiliar with PDAs, the Solana Cookbook has a [great writeup](https://solanacookbook.com/core-concepts/pdas.html) on what they are and how to build secure programs with them.
