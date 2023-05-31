# Scheduled Transfer

## Goals

In this guide, we will demonstrate how to automate a Solana system transfer using Clockwork. We will prepare a simple transfer of SOL between two accounts, then use the Clockwork SDK to schedule the transfer instruction to run every 10 seconds.

1. Understand the Clockwork security model.
2. Schedule a system transfer instruction.
3. Monitor an automated program.

## 1. Understanding the Clockwork programming model

Let's start with the big picture. Solana is a really fast, globally distributed computer. Just as programs on a traditional computer needs to be able to execute an automated series of instructions, so do programs on Solana. Clockwork threads are an automation primitive analogous to [**computer threads**](https://en.wikipedia.org/wiki/Thread\_\(computing\)) that developers can use to automate programs on Solana. In simple terms, this means we can point Clockwork at any Solana program to automate it. A model of this relationship is presented in the diagram below.&#x20;

![Figure 1](https://user-images.githubusercontent.com/8634334/222291232-ce195a01-7bdc-4567-8907-14485d19ee91.png)

## 2. Scheduling a SOL Transfer Instruction

To get started, we will assume you have a beginner's knowledge of Solana programming and some experience working with Anchor. If you are unfamiliar with these concepts, we recommend checking out  [**Solana's developer resources**](https://solana.com/developers) and setting up your local environment for Solana programming.&#x20;

Let's begin by initializing a new Anchor workspace for our project:

```sh
anchor init transfer
cd transfer
```

We will be leveraging the tests folder to write our client code. First, we need to install the test dependencies using `yarn`. After this, we can open up the test file located at `tests/transfer.ts.` Import the necessary libraries and set up the Solana and Clockwork providers.

```ts
import * as anchor from '@coral-xyz/anchor';
import * as web3 from '@solana/web3.js';
import { ClockworkProvider } from '@clockwork-xyz/sdk';

const provider = anchor.Provider.env();
anchor.setProvider(provider);

const wallet = provider.wallet;
const clockworkProvider = ClockworkProvider.fromAnchorProvider(provider);

```



Next, we'll prepare a system transfer instruction and automate it with a Clockwork thread.

```typescript
it("Transfers SOL every 10 seconds", async () => {
    const [threadAddress] = clockworkProvider.getThreadPDA(wallet.publicKey, threadId)

    // 1️⃣  Prepare an instruction to be automated.
    const lamports = web3.LAMPORTS_PER_SOL; // 1 SOL
    const transferIx = web3.SystemProgram.transfer({
        fromPubkey: threadAddress,
        toPubkey: web3.Keypair.generate(),
        lamports,
    });

    // 2️⃣  Define a trigger condition.
    const trigger = {
        cron: {
            schedule: "*/10 * * * * * *",
            skippable: true,
        },
    };

    // 3️⃣ Create the thread.
    const ix = await clockworkProvider.threadCreate(
        wallet.publicKey,          // authority
        "transfer",                // id
        [transferIx],              // instructions
        trigger,                   // trigger
        1 * web3.LAMPORTS_PER_SOL, // amount to fund the thread with
    );
    const tx = new web3.Transaction().add(ix);
    const signature = await clockworkProvider.anchorProvider.sendAndConfirm(tx);

    
    console.log(`https://app.clockwork.xyz/threads/${threadAddress}?cluster=devnet`);
    
    // Check balance of destination address
    const balance = await connection.getBalance(destination) / LAMPORTS_PER_SOL;
    console.log(`${destination.toString()} balance: ${balance} SOL`);

```

We can see the `threadCreate` function asks for 5 arguments. These include some basic information needed to initialize the thread account.

* `authority` – The owner of the thread. This account must be the transaction signer and will have permission to delete, pause, resume, stop, and update the thread.
* `id` – An identifier for the thread (can also use buffer or vec u8).
* `instructions` – The list of instructions to execute when the trigger condition becomes valid.
* `trigger` – The trigger condition for the thread. When this condition is valid, the thread will begin executing the provided instructions.
* `amount` – The number of lamports to fund the thread account with. Remember to provide a small amount of SOL. Read more about how fees are calculated [here](https://docs.clockwork.xyz/developers/threads/fees).



## 3. Monitoring an automated program

If you setup everything correctly, you can now watch your automated program run all on its own. Grab the clockwork explorer link that was printed out to the console. Using the clockwork explorer, you can get simulation logs and inspect if your thread is not running and why. For example, here's mine: [https://app.clockwork.xyz/threads/GB7YgYK3bKF8J4Rr9Z2oeA3hwxrJdvW5zgXuNaxWWmUF?cluster=devnet](https://app.clockwork.xyz/threads/GB7YgYK3bKF8J4Rr9Z2oeA3hwxrJdvW5zgXuNaxWWmUF?cluster=devnet)

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Of course you can also look up your thread account in your favorite Solana explorer. You can alternatively use the Solana CLI to stream program logs by running the command provided below. Here's [**an example thread**](https://explorer.solana.com/address/3ohRKgNyLS1iTGiUqnzoiFiQcrCLGmr3NWHzq4HW8BdJ?cluster=devnet) that was created in a test on May 24th, 2023.

```bash
solana logs -u devnet YOUR_PROGRAM_ID
```

<figure><img src="https://user-images.githubusercontent.com/8634334/222591908-bbaa04c5-83b4-46c2-b83b-68e1fef473eb.png" alt=""><figcaption></figcaption></figure>

## Key Learnings

1. Threads are an automation primitive for Solana.
2. You can use threads to automate any program instruction on Solana.&#x20;
3. [**Triggers**](../threads/triggers.md) allow you to define when a thread should begin execution.
4. Threads must be funded with a small amount of SOL to pay for [**automation fees**](../threads/fees.md).&#x20;

## Appendix

This guide was written using the following environment dependencies.

<table><thead><tr><th width="340">Dependency</th><th>Version</th></tr></thead><tbody><tr><td>Anchor</td><td>v0.27.0</td></tr><tr><td>Clockwork</td><td>v2.0.17</td></tr><tr><td>Clockwork TS SDK</td><td>v0.3.4</td></tr><tr><td>Rust</td><td>v1.65.0</td></tr><tr><td>Solana</td><td>v1.14.15</td></tr><tr><td>Ubuntu</td><td>v20.04</td></tr></tbody></table>

## Learn more

* A complete copy of all code provided in this guide can be found in the [**examples repo**](https://github.com/clockwork-xyz/examples/tree/main/1-hello\_clockwork) on GitHub.
* Ask questions on [**Discord**](https://discord.gg/epHsTsnUre).
