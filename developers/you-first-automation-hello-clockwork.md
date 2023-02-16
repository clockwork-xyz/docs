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

In this walkthrough, we will go over the basic steps of creating a thread. We start from a standard Solana program with a basic **Instruction**, and turn it into a **Automated Instruction** using the Clockwork SDK. We will follow this method:

1. 🛠 Setup
2. ✍️ Code
3. 🔮 Observe



## ⏳ The Clockwork Workflow

Let's start with the big picture, overall this is what the workflow feels like

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

## ⚓️ Anchor Program

We assume you have experience with Anchor, so we will skip through the basics, and directly clone this starter project:

```bash
git clone blablabla walkhtroughs/starter
```



### Deploy a Solana Program

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

> 🗺 Big Picture Reminder
>
> * [ ] Deploy a Solana Program 👈
> * [ ] Create an Automation _(later)_

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
#[instruction(name: String)]
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

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

### Automated Instruction

We deployed our Solana program and called our instruction through the RPC with anchor tests. Now, how do we go from a simple instruction that needs to be manually called to an automated instruction?

{% hint style="info" %}
👩‍🍳 The Clockwork Recipe

* [ ] Modify your instruction to return an `AutomationResponse` 👈
* [ ] Create an `Automation`
{% endhint %}

### Program Side - Return An AutomationResponse From Your Instruction

Install the \[clockwork-sdk for Solana Programs]\([https://crates.io/crates/clockwork-sdk](https://crates.io/crates/clockwork-sdk)):

```
cargo add clockwork-sdk@1.4.2
```

Jump into `programs/hello_clockwork/src/lib.rs` and add

```rust
...
use clockwork_sdk::{
    state::{AutomationResponse}, 👈
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
2. Modify the function signature to return a `Result<AutomationResponse>`
3. Finally, return an `AutomationResponse`. In future tutorials, we will go over what exactly is that `AutomationResponse`, for the time being, let's just return a `::default()` one.

Check that everything is fine

```
anchor build
```

### Client Side - Create an Automation

{% hint style="info" %}
👩‍🍳 The Clockwork Recipe

* [x] Modify your instruction to return an `AutomationResponse`
* [ ] Create an `Automation` 👈
{% endhint %}

Time to finally create an Automation, this is where things will feel a bit different. In this guide, we will create an automation from the client side.

> Again, we are leveraging anchor tests to keep the guide succint, but this can be applied to any Solana client.

#### Clockwork Typescript SDK

Start by installing the \[Clockwork Typescript SDK]\([https://www.npmjs.com/package/@clockwork-xyz/sdk](https://www.npmjs.com/package/@clockwork-xyz/sdk))

```
yarn add @clockwork-xyz/sdk
```

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

#### Create An Automation - Prepare An Instruction

{% hint style="info" %}
👩‍🍳 The Clockwork Recipe

* [x] Modify your instruction to return an `AutomationResponse`
* [ ] Create an `Automation`
  * [ ] 1\) Prepare an instruction 👈
{% endhint %}

Jump to `tests/hello_clockwork.ts` and add

```typescript
...

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
    const automation_ix = buildAutomationInstruction(
      targetIx,
    );
  });
});
```

* Previously we submitted the hello world instruction directly
* Now, we build the hello instruction without submitting it.

> 💡 Paradigm Shift #1: The instruction will be run by our Automation not ourselves

#### Create An Automation - Triggers

👩‍🍳 The Clockwork Recipe

* [x] Modify your instruction to return an `ThreadResponse`
* [ ] Create a `Thread`
  * [x] 1\) Prepare an instruction&#x20;
  * [ ] 2\) Define a trigger 👈

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

Then the question is how do we want the Automation to run the instruction? That's the t`rigger`

Today, we have two types of triggers:

* \[Time based]\([https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Cron](https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Cron))
* \[Account based]\([https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Account](https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Account))

We will talk more about triggers in other examples. For now, let's use a time-based one, so that our Automation runs the instruction \[at every 10th minutes]\([https://crontab.guru/every-10-minutes](https://crontab.guru/every-10-minutes)).

```typescript
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
    
    // 3. Create Automation
    const create_automation = buildAutomationInstruction(
      targetIx,
      trigger,
    );
  });
});
```

> 💡 Paradigm Shift #2: Automation can be triggered by time or conditions

#### Create An Automation - Accounts

We forgot the Solana mantra: _"In solana everything is a... account!"._ A Thread is an account, so we need to give an address that will be used for this account, and also who will pay for that account.

```typescript
import {getThreadAddress} from "@clockwork-xyz/sdk/lib/pdas";

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

* `automationLabel:` an identifier for the Automation _(can also use buffer or vec u8)_
* `automationAuthority:` the signing authority for the thread account. You will need the corresponding private key to sign any mutation to your Automation account. For example, if you need to delete/stop/pause/resume this Thread via the `clockwork thread` cli command (more on this later).
* `payer`: the payer from the transaction, will also be used to fund the Thread
* `threadAddress:` \[a program derived address]\([https://docs.rs/clockwork-thread-program/1.4.0/src/clockwork\_thread\_program/state/thread.rs.html#65](https://docs.rs/clockwork-thread-program/1.4.0/src/clockwork\_thread\_program/state/thread.rs.html#65))

#### Create An Automation - Thread Creation Request

```typescript
...
import {getThreadProgram} from "@clockwork-xyz/sdk";

describe("hello_clockwork", () => {
  it("It logs hello", async () => {
    // 1. Prepare an instruction to feed to the Thread
    ...

    // 2. Define a trigger for the Thread to execute
    ...

    // Accounts
    ...

    // 3. Create Thread
    const threadProgram = getThreadProgram(provider, "1.4.2");
    const createThreadIx = threadProgram.methods
      .threadCreate(
        threadLabel,
        {
          programId: targetIx.programId,
          accounts: [
            { pubkey: threadAddress, isSigner: false, isWritable: true }
          ],
          data: targetIx.data,
        },
        trigger,
      )
      .accounts({
        authority: threadAuthority,
        payer: payer,
        thread: threadAddress,
        systemProgram: anchor.web3.SystemProgram.programId,
      });

      const tx = await createThreadIx.rpc();
      print_address("🤖 Program", program.programId.toString());
      print_tx("✍️  Transaction", tx);
  });
});
```

* `getAutomationProgram()`: first we get an handle to the `automationProgram`. When you need to create a new automation, you are actually talking to the Clockwork Automation Program (which is already deployed by the Clockwork Team.).&#x20;
* IDLs Version: Make sure to use the proper version of the IDLs, you can check the deployed versions \[here]\([https://github.com/clockwork-xyz/clockwork#deployments](https://github.com/clockwork-xyz/clockwork#deployments))
* `automationCreate()`: the rest is really just plugging all the parameters we prepared earlier into the `automationCreate`() helper
* Finally, we submit the final `createAutomation` instruction



### 🔮 Observe - Let's See What's Happening

Run the tests

```
anchor test
```

> You might run into a `ThreadCreate - Instruction - "Address already in use"`. It just means the `threadAddress` is already in use, you just need to give modify the `threadLabel` or provide a new `threadAddress` to assign

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

You can use the command line to check the logs with

```bash
solana logs -u devnet | grep -A 10 YOUR_PROGRAM_ID
```

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

Or use the Clockwork explorer, here's the result with my program: [https://explorer.clockwork.xyz/address/A6KvnEL244thrkJACeuKvonoYa3LTtnG8ViCHf57r6fg?network=devnet](https://explorer.clockwork.xyz/address/A6KvnEL244thrkJACeuKvonoYa3LTtnG8ViCHf57r6fg?network=devnet)

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

### ⚠️ Security

Finally a note on security. For some reason, you might not want your instruction to be run by anyone or anything than your own Thread. In that case, we can add an anchor constraint and provide the thread address to our instruction.



In your Anchor Program:

```rust
use clockwork_sdk::{
    state::{Thread, ThreadAccount},
};

...

#[derive(Accounts)]
#[instruction(name: String)]
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



## 🧐 How Does it Actually Work?
