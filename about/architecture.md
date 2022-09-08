---
cover: ../.gitbook/assets/gitbook-header-3.png
coverY: 0
---

# Architecture

## Network design

Clockwork has two primary trust layers:

1. A **smart-contract** where users can pay SOL to create transaction queues.
2. A **worker network** where node operators can crank queues to collect automations fees as yield.

These two layers work together to power automations for other on-chain programs. At a network level Clockwork is unique in that all workers on the network are Solana validators and RPC nodes under the hood. This design offers many benefits over centralized backend services. Since the network runs on the same physical hardware as the Solana validators, it is generally cheaper, more reliable, and less spammy than traditional bots.

<figure><img src="../.gitbook/assets/Blank diagram (5).png" alt=""><figcaption><p>The Clockwork program serves as a proxy contract which verifies and forwards cranks on to your target program.</p></figcaption></figure>

## Automating signatures&#x20;

**Clockwork is non-custodial and does not hold onto users' private keys.** This creates unique security challenges for scheduling transactions on behalf of a user.&#x20;

### Pre-signed transactions

One naive approach to scheduling transactions would be to save pre-signed transaction data somewhere for submission at a later date. This is problematic since it would be impossible to prevent a malicious actor from submitting a pre-signed transaction ahead of its intended schedule. Solana explicitly protects against this by requiring every transaction to contain a [**recent blockhash**](https://docs.solana.com/developing/programming-model/transactions#recent-blockhash). This has the consequence of causing Solana transactions to go stale if they're not submitted to blockchain within a couple minutes of being signed.

### Delegated signatories

Instead Clockwork uses a **delegated signatory** model. When a worker submits a crank transaction, the Clockwork program receives it first and adds an additional PDA signer to the transaction context before forwarding it on to the target program.&#x20;

Target programs can verify the signature from Clockwork is valid to know if the crank request and safe to process. This proxy-contract model protects programs against spam and unwanted invocations. Code samples for how to correctly verify crank requests can be found in the [**examples** ](https://github.com/clockwork-xyz/examples/blob/main/hello\_clockwork/programs/hello\_clockwork/src/instructions/hello\_world.rs)repo on Github.&#x20;
