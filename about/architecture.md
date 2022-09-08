---
cover: ../.gitbook/assets/gitbook-header-3.png
coverY: 0
---

# Architecture

## Overview

Clockwork has two primary compute layers:

1. A **smart-contract** where users can pay SOL to create transaction queues.
2. A **worker network** where node operators can crank queues and collect automations fees as yield.

These two layers work together to power automations for other programs. At a network level, Clockwork is unique in that all workers on the network are Solana validators and RPC nodes under the hood. Since the network runs on the same physical hardware as the validators, this design is generally more reliable, less spammy, and lower cost than traditional SaaS-based alternatives.

<figure><img src="../.gitbook/assets/Blank diagram (5).png" alt=""><figcaption><p>The Clockwork program serves as a proxy contract which verifies and forwards cranks on to your target program.</p></figcaption></figure>

## Transaction queues

As a smart-contract primitive, Clockwork provides **transaction queues** for users to manage the state of on-chain jobs and workflows. Queues tell worker nodes which transactions to submit and pay yield to workers who submit transactions promptly. They doubly serve as a communication and payment channel, allowing users to securely transact with and delegate to the worker network.

<figure><img src="../.gitbook/assets/Blank document (18).png" alt=""><figcaption><p>Queues facilitate bi-directional exchange of value between users and worker nodes.</p></figcaption></figure>

## Automating signatures&#x20;

**Clockwork is a non-custodial service and does not hold onto users' private keys.** This creates unique security challenges for scheduling transactions on behalf of a user.&#x20;

### Pre-signed transactions

One naive approach to scheduling transactions would be to save pre-signed transaction data somewhere for submission at a later date. This is problematic since it would be impossible to prevent a malicious actor from submitting pre-signed transactions ahead of their intended schedules. Solana explicitly protects against this by requiring every transaction to contain a [**recent blockhash**](https://docs.solana.com/developing/programming-model/transactions#recent-blockhash). This has the consequence of causing Solana transactions to go stale if they're not submitted to blockchain within a couple minutes of being signed.

### Delegated signatories

Instead, Clockwork utilizes a **delegated signatory** model. When a worker submits a crank transaction, the Clockwork program receives it first and adds an additional PDA signer to the transaction context before forwarding it on to the target program.&#x20;

Target programs can verify the Clockwork signature is valid to know if the crank request is safe to process. This proxy-contract model protects programs against spam and from unwanted invocations. Code samples for how to correctly verify crank requests can be found in the [**examples** ](https://github.com/clockwork-xyz/examples/blob/main/hello\_clockwork/programs/hello\_clockwork/src/instructions/hello\_world.rs)repo on Github.&#x20;
