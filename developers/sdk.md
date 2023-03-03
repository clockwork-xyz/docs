# SDK

{% hint style="warning" %}
This page needs to updated for v2 👷🏼
{% endhint %}

## Getting started

{% embed url="https://crates.io/crates/clockwork-sdk" %}

## Account model

```rust
pub struct Thread {
    /// The owner of this thread.
    pub authority: Pubkey,
    /// The cluster clock at the moment the thread was created.
    pub created_at: ClockData,
    /// The current execution context.
    pub exec_context: Option<ExecContext>,
    /// The number of lamports to payout to workers per instruction execution.
    pub fee: u64,
    /// The id of the thread, set by the authority.
    pub id: String,
    /// The instruction to kickoff the thread.
    pub kickoff_instruction: InstructionData,
    /// The next instruction in the thread.
    pub next_instruction: Option<InstructionData>,
    /// Whether or not the thread is currently paused.
    pub paused: bool,
    /// The maximum number of instructions allowed per slot.
    pub rate_limit: u64,
    /// The triggering event to kickoff a thread.
    pub trigger: Trigger,
}
```

### Public address

The public address of every thread is derived deterministically from its `authority` and `id`. These properties are immutable and may never change throughout the lifetime of the thread. To verify a Clockwork thread account in your programs, you can use the `pubkey()` helper function provided by the SDK:

```rust
#[account(
    address = thread.pubkey(),
    contraint = thread.id.eq("my_thread"),
    has_one = authority,
)]
pub thread: Account<'info, Thread>
```

### Authority

On Solana, the term “authority” is often used by convention to refer to the user-space owner of a particular account. For Clockwork, a thread's authority is its creator – the account which signed the transaction to originally create the thread. A thread's authority has the following permissions:

* Pause and resume the thread.
* Update the thread’s `trigger` and `kickoff_instruction`.
* Withdraw from the thread’s balance.
* Close the thread account.

An authority may be any valid public address (i.e. a wallet pubkey or PDA). When the authority is a PDA, we occasionally refer to this as a “program authority” since the account is managed by a program. If you are unfamiliar with PDAs, the Solana Cookbook has a [great writeup](https://solanacookbook.com/core-concepts/pdas.html) on what they are and how to build secure programs with them.

## Dynamic payers

Anchor currently does not support PDAs as payers for account initialization. This means that if one of your instructions initializes a new account, you must specify a keypair signer as the `payer`. For this, we provide a special Clockwork payer address:

```rust
C1ockworkPayer11111111111111111111111111111
```

If an account in your instruction references the Clockwork payer account, workers will automatically inject their address in its place. By doing this, the worker node will pay for any account initializations your programs need to do, and Clockwork will reimburse them from the thread’s balance.

{% hint style="info" %}
Once Anchor adds support for PDA payers (expected in the next release), the “payer injection” feature will be deprecated in favor of using PDAs to pay for account initializations.

[https://github.com/coral-xyz/anchor/pull/1938](https://github.com/coral-xyz/anchor/pull/1938)
{% endhint %}
