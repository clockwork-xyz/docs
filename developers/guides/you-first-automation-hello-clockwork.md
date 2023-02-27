---
description: Version 1.4.2
---

# You First Automation - Hello Clockwork

## Goals

* [ ] Understand the Clockwork workflow
* [ ] Turning a simple Solana Instruction into an Automated Instruction
* [ ] Getting familiar with Clockwork libs
* [ ] Creating a Time Triggered Cron Thread
* [ ] Debugging

### Our Roadmap

In this walkthrough, we will go over the basic steps of creating a thread. We start from a standard Solana program with a basic **Instruction**, and turn it into a **Threadable** using the Clockwork SDK. We will follow this method:

1. 🛠 Setup
2. ✍️ Code
3. 🔮 Observe



## ⏳ The Clockwork Workflow

Let's start with the big picture, overall this is what the workflow feels like

<figure><img src="../../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

## ⚓️ Anchor Program

We assume you have experience with Anchor, so we will skip through the basics, and directly clone this starter project:

```bash
git clone git@github.com:clockwork-xyz/examples.git
cd walkthroughs/hello_clockwork/starter/hello_clockwork
```



### Deploy a Solana Program

<figure><img src="../../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

> 🗺 Big Picture Reminder
>
> * [ ] Deploy a Solana Program 👈
> * [ ] Create a Thread _(later)_

Let's take a look at the program in`programs/hello_clockwork/src/lib.rs:`

```rust
use anchor_lang::prelude::*;

#[program]
pub mod hello_clockwork {
    use super::*;

    pub fn hello_ix(_ctx: Context<HelloClockwork>) -> Result<()> {
        msg!(
            "Hello! The current time is: {}",
            Clock::get().unwrap().unix_timestamp
        );
        Ok(())
    }
}

#[derive(Accounts)]
pub struct HelloClockwork {}
```

* Nothing crazy in here, it is just a simple instruction that will print a log



Before going further, let's update the Anchor **Program Id** with your own, and make sure you can deploy. Just run this script:

```bash
./deploy.sh devnet
```

{% hint style="info" %}
In this article, we will only cover how to use devnet for now. Localnet involves more setup steps.
{% endhint %}



## ⏳ Automating Our Instruction With Clockwork

> 🗺 Big Picture Reminder
>
> * [x] Deploy a Solana Program
> * [ ] Adapt our Solana Program For Automation __ 👈
> * [ ] Create an Automation

### Off-Chain - Using Anchor Tests

To keep this walkthrough succinct. We will be using existing anchor tests to interact with the Solana RPC. Let's take a look at the existing tests in `tests/hello_clockwork.ts`

```typescript
describe("hello_clockwork", () => {
  it("It logs hello", async () => {
    const tx = await program.methods.helloIx().rpc();

    print_address("🤖 Program", program.programId.toString());
    print_tx("✍️  Transaction", tx);
  });
});
```



Run the tests, and click on the **transaction** **link** to check our program logs

```bash
anchor test
```

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

### Automated Instruction

We deployed our Solana program and called our instruction through the RPC with anchor tests. Now, how do we go from a simple instruction that needs to be manually called to an automated instruction?

{% hint style="info" %}
👩‍🍳 The Clockwork Recipe

* [ ] Modify your instruction to return a `ThreadResponse` 👈
* [ ] Create a `Thread`
{% endhint %}

### Program Side - Return a ThreadResponse From Your Instruction

Install the \[clockwork-sdk for Solana Programs]\([https://crates.io/crates/clockwork-sdk](https://crates.io/crates/clockwork-sdk)):

```
cargo add clockwork-sdk@1.4.2
```

Jump into `programs/hello_clockwork/src/lib.rs` and add

```rust
...
use clockwork_sdk::{
    state::{ThreadResponse}, 👈
};


#[program]
pub mod hello_clockwork {
    use super::*;

    pub fn hello_ix(_ctx: Context<HelloClockwork>, name: String) 
    -> Result<ThreadResponse> 👈 {
        msg!(
            "Hello {}! The current time is: {}",
            name,
            Clock::get().unwrap().unix_timestamp
        );
        
        Ok(ThreadResponse::default()) 👈
    }
}
```

1. Import the `clockwork_sdk` crate
2. Modify the function signature to return a `Result<ThreadResponse>`
3. Finally, return an `ThreadResponse`. In future tutorials, we will go over what exactly is that `ThreadResponse`, for the time being, let's just return a `::default()` one.

Check that everything is fine

```
anchor build
```

### Client Side - Create a Thread

{% hint style="info" %}
👩‍🍳 The Clockwork Recipe

* [x] Modify your instruction to return a`ThreadResponse`
* [ ] Create an `Automation` 👈
{% endhint %}

Time to finally create a Thread, this is where things will feel a bit different. In this guide, we will create a Thread from the client side.

> Again, we are leveraging anchor tests to keep the guide succinct, but this can be applied to any Solana client.

#### Clockwork Typescript SDK

Start by installing the \[Clockwork Typescript SDK]\([https://www.npmjs.com/package/@clockwork-xyz/sdk](https://www.npmjs.com/package/@clockwork-xyz/sdk))

```
yarn add @clockwork-xyz/sdk
```

<figure><img src="../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

#### Create An Automation - Prepare An Instruction

{% hint style="info" %}
👩‍🍳 The Clockwork Recipe

* [x] Modify your instruction to return an `ThreadResponse`
* [ ] Create a `Thread`
  * [ ] 1\) Prepare an instruction 👈
{% endhint %}

Jump to `tests/hello_clockwork.ts` and add

```typescript
...
// 👇 The new import
import { getThreadAddress, createThread } from "@clockwork-xyz/sdk";

const step1_buildHelloInstruction = async (name: string) => { 👈
  return program.methods
    .helloIx(name)
    .accounts({})
    .instruction();
}

describe("hello_clockwork", () => {
  it("It logs hello", async () => {
    // 1. Prepare an instruction to feed to the Automation
    const targetIx = await buildHelloInstruction("Chronos") 👈
    
    // Create Automation
    const createThreadIx = createThread({
      instruction: targetIx,
      trigger: ?,
      threadName: ?,
      threadAuthority: ?,
    }, provider);
  });
});
```

* Previously we submitted the hello world instruction directly
* Now, we build the hello instruction without submitting it.

> 💡 Paradigm Shift #1: The instruction will be run by our Thread not manually

### Create An Automation - Triggers

👩‍🍳 The Clockwork Recipe

* [x] Modify your instruction to return an `ThreadResponse`
* [ ] Create a `Thread`
  * [x] 1\) Prepare an instruction&#x20;
  * [ ] 2\) Define a trigger 👈

<figure><img src="../../.gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure>

Then the question is how do we want the Automation to run the instruction? That's the t`rigger`

Today, we have two types of triggers:

* \[Time based]\([https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Cron](https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Cron))
* \[Account based]\([https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Account](https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Account))

We will talk more about triggers in other examples. For now, let's use a time-based one, so that our Automation runs the instruction every 10 seconds

<pre class="language-typescript"><code class="lang-typescript">// 👇 The new import
<strong>import { getThreadAddress, createThread } from "@clockwork-xyz/sdk";
</strong>
it("It logs hello", async () => {
    // 1. Prepare an instruction to feed to the Automation
    const targetIx = await buildHelloInstruction("Chronos")

    // 2. Define a trigger for the Automation to execute
    const trigger = {
      cron: {
        schedule: "*/10 * * * * * *",
        skippable: true,
      },
    }
    
    // 3. Create Thread
    const createThreadIx = createThread({
      instruction: targetIx,
      trigger: trigger,
      threadName: ?,
      threadAuthority: ?,
    }, provider);
    
  });
});
</code></pre>

> 💡 Paradigm Shift #2: Automation can be triggered by time or conditions

#### Create A Thread - Accounts

We forgot the Solana mantra: _"In solana everything is a... account!"._ A Thread is an account, so we need to give an address that will be used for this account and who will pay for that account.

```typescript
import { getThreadAddress, createThread } from "@clockwork-xyz/sdk";

it("It logs hello", async () => {
    // 1. Prepare an instruction to feed to the Thread
    ...

    // 2. Define a trigger for the Thread to execute
    ...
    
    // Accounts
    const threadLabel = "hello_clockwork";
    const threadAuthority = provider.publicKey;
    const payer = provider.publicKey;
    const threadAddress = getThreadAddress(threadAuthority, threadLabel);
  });
});
```

* `threadLabel:` an identifier for the Automation _(can also use buffer or vec u8)_
* `threadAuthority:` the signing authority for the thread account. You will need the corresponding private key to sign any mutation to your Thread account. For example, if you need to delete/stop/pause/resume this Thread via the `clockwork thread` cli command (more on this later).
* `payer`: the payer from the transaction, will also be used to fund the Thread
* `threadAddress:` \[a program-derived address]\([https://docs.rs/clockwork-thread-program/1.4.0/src/clockwork\_thread\_program/state/thread.rs.html#65](https://docs.rs/clockwork-thread-program/1.4.0/src/clockwork\_thread\_program/state/thread.rs.html#65))

#### Create An Automation - Thread Creation Request

```typescript
import { getThreadAddress, createThread } from "@clockwork-xyz/sdk";

describe("hello_clockwork", () => {
  it("It logs hello", async () => {
    // 1. Prepare an instruction to feed to the Thread
    ...

    // 2. Define a trigger for the Thread to execute
    ...

    // Accounts
    ...

    // 3. Create Thread
    const createThreadIx = createThread({
      instruction: targetIx,
      trigger: trigger,
      threadName: threadLabel,
      threadAuthority: threadAuthority,
    }, provider);

      const tx = await createThreadIx;
      print_address("🤖 Program", program.programId.toString());
      print_tx("✍️  Transaction", tx);
  });
});
```

* createThread`()`: the rest is really just plugging all the parameters we prepared earlier into the createThread() helper
* Finally, we submit the final `createAutomation` instruction



### 🔮 Observe - Let's See What's Happening

Run the tests

```
anchor test
```

> You might run into a `ThreadCreate - Instruction - "Address already in use"`. It just means the `threadAddress` is already in use, you just need to give modify the `threadLabel` or provide a new `threadAddress` to assign

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

You can use the command line to check the logs with

```bash
solana logs -u devnet | grep -A 10 YOUR_PROGRAM_ID
```

<figure><img src="../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

Or use the Clockwork explorer, here's the result with my program: [https://explorer.clockwork.xyz/address/A6KvnEL244thrkJACeuKvonoYa3LTtnG8ViCHf57r6fg?network=devnet](https://explorer.clockwork.xyz/address/A6KvnEL244thrkJACeuKvonoYa3LTtnG8ViCHf57r6fg?network=devnet)

<figure><img src="../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

### ⚠️ Security

Finally a note on security. For some reason, you might not want your instruction to be run by anyone or anything than your own Thread. In that case, we can add an anchor constraint and provide the thread address to our instruction.



In your Anchor Program:

```rust
use clockwork_sdk::{
    state::{Thread, ThreadAccount},
};

...

#[derive(Accounts)]
pub struct HelloClockwork<'info> {
    #[account(address = thread.pubkey(), signer)]
    pub thread: Account<'info, Thread>,
}
```

In the client:

```rust
const buildHelloInstruction = async (threadAddress: PublicKey) => {
  return await program.methods
    .helloIx()
    .accounts({ thread: threadAddress })
    .instruction();
}
```

## Going Further

* Check the \[FAQ]\([https://github.com/clockwork-xyz/docs/blob/main/FAQ.md#common-errors](../../FAQ.md#common-errors))
* Come build with us and ask us questions [Discord](https://discord.gg/epHsTsnUre)!



## 🧐 How Does it Actually Work?
