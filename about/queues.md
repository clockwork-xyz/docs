# Use-cases

Clockwork automations massively expands the design space for blockchain developers. Below is a list of use-cases developers are using threads to build today:

### Defi

* **Defi protection** – Automatically repay debt on a lending market like [**Solend**](https://solend.fi/) or [**Jet**](https://www.jetprotocol.io/) when collateral prices approach liquidation levels.&#x20;
* **Trading bots** – **** Execute trades on an order book like [**Serum**](https://www.projectserum.com/) based on price feeds and technical indicators.&#x20;
* **Auto-claim yield** – Automatically claim yield from your validator or favorite defi application.&#x20;

### Payments

* **Scheduled payments** – Schedule token transfers and automate payroll or subscription payments.
* **Dollar-cost averaging** – Run an automated dollar-cost averaging program on-chain to ease into an investment position without hassle or stress. &#x20;

### Oracles

* **Derivative data feeds** – Calculate stats, moving averages, and derivatives from oracle data feeds.



## Flow control

As soon as a thread's trigger condition is met, the worker network will begin submitting transactions to execute the thread's "kickoff instruction". Here, the target program can do whatever it needs to with the accounts and data it receives. When finished, the program can respond with a "next instruction" to be invoked during the next execution of the thread. In this way, threads provide a simple interface for building complex and dynamically branching workflows via smart-contracts.

<figure><img src="../.gitbook/assets/Blank document (19).png" alt=""><figcaption><p>When a thread begins, it will execute its "kickoff instruction".</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Blank document (20).png" alt=""><figcaption><p>Thereafter, a thread will recursively execute its "next instruction" until the target programs says there is no more work to do.</p></figcaption></figure>

## Fees

Fees are a cost, paid by threads, for the automation services provided by the worker network. The minimum fee is **0.000001 SOL / instruction**, but thread authorities can choose to pay a higher fee to prioritize their threads with block builders.&#x20;
