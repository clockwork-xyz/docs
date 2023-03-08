# 0. Hello, Clockwork

## Goals

In this guide, we will learn how to automate a [**Solana**](https://solana.com) program using [**Clockwork**](https://clockwork.xyz). We will start by building a simple program with [**Anchor**](https://www.anchor-lang.com/), and then use the Clockwork SDK to schedule an instruction to run every 10 seconds. Our goals will be to:

1. Understand the Clockwork programming model.
2. Schedule a program instruction.
3. Monitor an automated program.

## 1. Understanding the Clockwork programming model

Let's start with the big picture. Solana is a really fast, globally distributed computer. Just as programs on a traditional computer needs to be able to execute an automated series of instructions, so too do programs on Solana. Clockwork threads are an automation primitive analogous to [**computer threads**](https://en.wikipedia.org/wiki/Thread\_\(computing\)) that developers can use to automate programs on Solana. In simple terms, this means we can point Clockwork at any Solana program to automate it. A model of this relationship is presented in the diagram below.&#x20;

![Figure 1](https://user-images.githubusercontent.com/8634334/222291232-ce195a01-7bdc-4567-8907-14485d19ee91.png)

## 2. Printing "Hello, world" on Solana

To get started, we will assume you have a beginner's knowledge of Solana programming and some experience working with Anchor. If you are unfamiliar with these concepts, we recommend checking out  [**Solana's developer resources**](https://solana.com/developers) and setting up your local environment for Solana programming.&#x20;

Let's begin by initializing a new Anchor workspace for our project:

```sh
anchor init hello_clockwork
cd hello_clockwork
```

Now let's open up the program file located at `programs/hello_clockwork/src/lib.rs`. In here, we have an entire Solana program. Let's add an instruction named `hello` that prints out `"Hello, {name}"` followed by the current Solana cluster timestamp.

```rust
use anchor_lang::prelude::*;

declare_id!("HCWy1dTbHUUk64LyqBeQxha23AMrRgKfShPXVmDuiUbY");

#[program]
pub mod hello_clockwork {
    use super::*;

    pub fn hello(_ctx: Context<Hello>, name: String) -> Result<()> {
        msg!(
            "Hello, {}! The current time is: {}",
            name,
            Clock::get().unwrap().unix_timestamp
        );
        Ok(())
    }
}

#[derive(Accounts)]
#[instruction(name: String)]
pub struct Hello {}
```

Anchor will automatically deploy your program when you run your tests. To configure your tests, we'll first need to install the test dependencies using `yarn`. After this, we can open up the test file located at `tests/hello_clockwork.ts` and add a test case that calls our program's `hello` instruction.

```ts
describe("hello_clockwork", () => {
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.HelloClockwork as Program<HelloClockwork>;

  it("It says hello", async () => {
    const tx = await program.methods.hello("world").rpc();
    console.log(tx);
  });
});
```

To run the test, simply execute the command below in your local project directory. You can verify the program ran successfully by looking up the transaction signature printed out to the console in your favorite [Solana explorer](https://explorer.solana.com).

```sh
anchor test
```

## 3. Creating a Clockwork thread

We just deployed our program to the Solana devnet and tested it by sumbitting a transaction to call `hello`. Now we will learn how to automate that instruction to run every 10 seconds. But first, we need to install the [**Clockwork Typescript SDK**](https://www.npmjs.com/package/@clockwork-xyz/sdk).

```
yarn add @clockwork-xyz/sdk
```

Let's jump back into the test file located at `tests/hello_clockwork.ts`. In our first test case, we submitted a transaction to invoke `hello` directly. This time, let's add a new test case that builds the `hello` instruction and passes it to a Clockwork thread to be automated.

```ts
// 0️⃣  Import the Clockwork SDK.
import * from "@clockwork-xyz/sdk";

describe("hello_clockwork", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);
  const wallet = provider.wallet;
  const program = anchor.workspace.HelloClockwork as Program<HelloClockwork>;
  const clockworkProvider = new ClockworkProvider(wallet, provider.connection);

  it("It runs every 10 seconds", async () => {
    // 1️⃣  Prepare an instruction to be automated.
    const targetIx = await program.methods.hello("world").accounts({}).instruction();

    // 2️⃣  Define a trigger condition.
    const trigger = {
      cron: {
        schedule: "*/10 * * * * * *",
        skippable: true,
      },
    };
    
    // 3️⃣ Create the thread.
    const threadId = "test-" + new Date().getTime() / 1000;
    const tx = await clockworkProvider.threadCreate(
        wallet.publicKey,             // authority
        threadId,                     // id
        [targetIx],                   // instructions
        trigger,                      // trigger
        anchor.web3.LAMPORTS_PER_SOL, // amount
    );
    const [threadAddress, threadBump] = clockworkProvider.getThreadPDA(wallet.publicKey, threadId)
    console.log(threadAddress);
    console.log(tx);
  });
  
});
```

We can see the `threadCreate` function asks for 5 arguments. These include some basic information needed to initialize the thread account.

* `authority` – The owner of the thread. This account must be the transaction signer and will have permission to delete, pause, resume, stop, and update the thread.
* `id` – An identifier for the thread (can also use buffer or vec u8).
* `instructions` – The list of instructions to execute when the trigger condition becomes valid.
* `trigger` – The trigger condition for the thread. When this condition is valid, the thread will begin executing the provided instructions.
* `amount` – The number of lamports to fund the thread account with. Remember to provide a small amount of SOL. The Clockwork base fee starts at 1000 lamports per executed instruction.

## 4. Monitoring an automated program

If you setup everything correctly, you can now watch your automated program run all on its own. Grab the thread address that was printed out to the console and look it up in your favorite Solana explorer. You can alternatively use the Solana CLI to stream program logs from devnet by running the command provided below. Here's [**an example thread**](https://explorer.solana.com/address/3ohRKgNyLS1iTGiUqnzoiFiQcrCLGmr3NWHzq4HW8BdJ?cluster=devnet) that was created in a test on March 2nd, 2023.

```bash
solana logs -u devnet YOUR_PROGRAM_ID
```

<figure><img src="https://user-images.githubusercontent.com/8634334/222591908-bbaa04c5-83b4-46c2-b83b-68e1fef473eb.png" alt=""><figcaption></figcaption></figure>

## Key Learnings

1. Threads are an automation primitive for Solana.
2. You can use threads to automate any program instruction on Solana.&#x20;
3. ****[**Triggers**](../threads/triggers.md) allow you to define when a thread should begin execution.
4. Threads must be funded with a small amount of SOL to pay for [**automation fees**](../threads/fees.md).&#x20;

## Appendix

This guide was written using the following environment dependencies.

| Dependency       | Version  |
| ---------------- | -------- |
| Anchor           | v0.26.0  |
| Clockwork        | v2.0.1   |
| Clockwork TS SDK | v0.2.3   |
| Rust             | v1.65.0  |
| Solana           | v1.14.15 |
| Ubuntu           | v20.04   |

## Learn more

* A complete copy of all code provided in this guide can be found in the [**examples repo**](https://github.com/clockwork-xyz/examples/tree/main/hello\_clockwork) on Github.
* Ask questions on [**Discord**](https://discord.gg/epHsTsnUre).
