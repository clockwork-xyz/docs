# Hello, Clockwork – Your first automation

## Goals

In this guide, we will build an automated program with Clockwork. We will start with a simple Solana program and then use the Clockwork SDK to schedule one of its instructions to run every 10 seconds. Our goals will be to:

* [ ] Understand the Clockwork programming model.
* [ ] Learn how to schedule an instruction.
* [ ] Debug common programming errors. 

## 0. Understanding the Clockwork programming model

Let's start with the big picture. Solana is a really fast, globally distributed computer. Just on like a traditional computer, Solana programs need be able to execute dynamic and long-running series of instructions. To do this on a traditional computer, developers use a primitive called a [thread](https://en.wikipedia.org/wiki/Thread_(computing)). On Solana, program developers can use Clockwork threads. 

In simple terms, Clockwork is an automation primitive for Solana. We can point Clockwork at any program on Solana to automate it. A simplified model of the relationship between Clockwork and a target automated program is available in the diagram below. As we progress through this guide, we will build our way from right-to-left across the diagram – first deploying a simple Solana program, and then creating a Clockwork thread to automate it. 

![Figure 1](https://user-images.githubusercontent.com/8634334/222291232-ce195a01-7bdc-4567-8907-14485d19ee91.png)

## 1. Deploying a Solana program

To get started, we will assume you have a beginner's knowledge of Solana programming and some experience with Anchor. If you are unfamiliar with these concepts, we recommend you checkout the [Anchor framework](https://www.anchor-lang.com/) and setup your local environment for Solana smart-contract development.

Let's begin by creating a new Anchor workspace for our project:
```sh
anchor init hello_clockwork
cd hello_clockwork
```

Now let's open up the program file located at `programs/hello_clockwork/src/lib.rs`. In here, we have our entire Solana program. Let's add an instruction named `hello` that prints out `"Hello, {name}"` and the current Solana cluster timestamp. 

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

Anchor will automatically deploy your program to devnet when you run tests. To do this, we'll first need to install the test dependencies using `yarn`. Next, we can open up our test file located at `tests/hello_clockwork.ts` and write test case that calls our `hello` instruction. 

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

To run your tests, simply execute `anchor test`.



## 2. Automating a program with Clockwork

We just deployed our Solana program to devnet and tested it by sumbitting a transaction to call the `hello` instruction. Now we'll automate that transaction to happen every 10 seconds. First we'll need to install the [Clockwork Typescript SDK](https://www.npmjs.com/package/@clockwork-xyz/sdk).

```
yarn add @clockwork-xyz/sdk
```

Now, we can jump back into the test file at `tests/hello_clockwork.ts` and spin up a new thread. Previously we executed the `hello` instruction directly. Now we will build the instruction and pass it to a thread to be automated.

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
        [targetIx],                   // instructions to execute
        trigger,                      // trigger condition
        anchor.web3.LAMPORTS_PER_SOL, // pre-fund amount
    );
    const [threadAddress, threadBump] = clockworkProvider.getThreadPDA(wallet.publicKey, threadId)
    console.log(threadAddress);
    console.log(tx);
  });
  
});
```

We can see the `threadCreate` call requires 5 arguments. This includes some basic information needed to initialize the thread account. 
* `authority` – The owner of the thread. This account must be the transaction signer and will have permission to delete/stop/pause/resume and update the thread.
* `id` – An identifier for the thread _(can also use buffer or vec u8)_.
* `instructions` – The list of instructions to execute when the trigger condition becomes valid.
* `trigger` – The trigger condition for the thread. When this condition becomes valid, the the thread will begin executing the provided instructions.
* `amount` – The number of lamports to fund the thread account with. Remember to provide a small amount of SOL. The Clockwork base fee begins at 1000 lamports per executed instruction.


## 3. Monitoring automated programs

You can use the Solana CLI to stream program logs from devnet by running the command below. You can alternatively monitor a thread using the Solana explorer. Here's [an example thread](https://explorer.solana.com/address/3ohRKgNyLS1iTGiUqnzoiFiQcrCLGmr3NWHzq4HW8BdJ?cluster=devnet) that was created in a test on March 2nd, 2022:

```bash
solana logs -u devnet | grep -A 10 YOUR_PROGRAM_ID
```

## Appendix

For a full copy of all example code in this guide, please checkout the `hello_clockwork` project in the [Clockwork examples repo](https://github.com/clockwork-xyz/examples/tree/main/hello_clockwork). This guide was written with the following environment dependencies.

| Dependency | Version |
| --- | --- |
| Anchor | 0.26.0 |
| Clockwork | v1.4.2 |
| Clockwork TS SDK | v0.2.3 |
| Rust | v1.65.0 | 
| Solana | v1.14.15 |

To learn more:
* Read the [Clockwork FAQ](../../FAQ.md#common-errors).
* Ask questions in the [Clockwork Discord](https://discord.gg/epHsTsnUre).
