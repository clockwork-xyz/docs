# Authority

{% embed url="https://github.com/clockwork-xyz/clockwork/blob/781b42fd06f2926899597ce5ea1f19b8ecd3d2e4/programs/thread/src/state/thread.rs#L11" %}
authority: Pubkey
{% endembed %}

On Solana, the **“authority”** property is often used by convention to refer to the user-space owner of a particular account. For Clockwork, a thread's authority is its creator – the account which signed the transaction to create the thread. A thread's authority has the following permissions:

* Pause and resume the thread.
* Update the thread’s `trigger` and `instructions`.
* Withdraw from the thread’s balance.
* Close the thread account.

An authority may be any valid public address (i.e. a wallet pubkey or PDA). When the authority is a PDA, we occasionally refer to this as a “program authority” since the account is managed by a program. If you are unfamiliar with PDAs, the Solana Cookbook has a [great writeup](https://solanacookbook.com/core-concepts/pdas.html) on what they are and how to build secure programs with them.
