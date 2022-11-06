# Threads

**Threads are an automation primitive for Solana programs.** Just as traditional applications use[ **threads**](https://en.wikipedia.org/wiki/Thread\_\(computing\)) to execute instructions on a computer, Solana programs can use Clockwork threads to execute a series of instructions on the blockchain. In this way, threads can help bring smart-contracts to life and make them _run_.

## Use-cases

Threads massively expand the design space for blockchain developers. Below is a list of example programs that developers are building with Clockwork today:

**Defi protection** – Automatically repay debt on a lending market like [**Solend**](https://solend.fi/) or [**Jet**](https://www.jetprotocol.io/) when collateral prices approach liquidation levels.&#x20;

**Yield maximization** – One can even take this a step further to maximize returns by automatically rebalancing collateral positions based on relative yield rates.&#x20;

**Scheduled payments** – Schedule token transfers and automate payroll or subscription payments.

**Dollar-cost averaging** – Run an automated dollar-cost averaging program on-chain to ease into an investment position without hassle or stress. &#x20;

**Auto-claim yield** – Automatically claim yield from your validator or favorite defi application.&#x20;

**Trading bots** – **** Execute trades on an order book like [**Serum**](https://www.projectserum.com/) based on price feeds and technical indicators.&#x20;

**Derivative data feeds** – Calculate stats, moving averages, and derivatives from oracle data feeds.

## Triggers

Clockwork provides three different triggering conditions to kickoff threads:&#x20;

1. **Account** – Triggers whenever an account's data changes. This can be useful for listening to account updates, process realtime events, or subscribing to an oracle data stream.
2. **Cron** – Triggers according to a [**cron schedule**](https://en.wikipedia.org/wiki/Cron). This can be useful for scheduling one-off or periodically recurring actions.
3. **On-demand** – Begins executing immediately. This trigger type is useful when for immediately kicking off a complex chain of transactions.

## Flow control

As soon as a thread's trigger condition is met, the worker network will begin submitting transactions to execute the thread's "kickoff instruction". Here, the target program can do whatever it needs to with the accounts and data it receives. When finished, the program can respond with a "next instruction" to be invoked during the next execution of the thread. In this way, threads provide a simple interface for building complex and dynamically branching workflows via smart-contracts.

<figure><img src="../.gitbook/assets/Blank document (19) (1).png" alt=""><figcaption><p>When a thread begins, it will execute its "kickoff instruction".</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Blank document (20).png" alt=""><figcaption><p>Thereafter, a thread will recursively execute its "next instruction" until the target programs says there is no more work to do.</p></figcaption></figure>

## Fees

Fees are a cost, paid by threads, for the automation services provided by the worker network. The minimum fee is **0.000001 SOL / instruction**, but thread authorities can choose to pay a higher fee to prioritize their threads with block builders.&#x20;
