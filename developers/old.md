# OLD

Version 1.4.0

## &#x20;Goals

* [ ] Understand the Clockwork workflow
* [ ] Turning a simple Solana Instruction into a Threadable Instruction
* [ ] Getting familiar with Clockwork libs
* [ ] Creating a Time Triggered Cron Thread
* [ ] Debugging

### Our Roadmap

In this walkthrough, we will go over the basic steps of creating a thread. We start from a standard Solana program with a basic **Instruction**, and turn it into a **Threadable Instruction** using the Clockwork SDK. We will follow this method:

1. 🛠 Setup
2. ✍️ Code
3. 🔮 Observe

{% hint style="info" %}
This walkthrough is using rust to interact with the RPC. If you are looking for a typescript example please refer to TODO
{% endhint %}

## The Clockwork Workflow

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

1. Deploy a (program) instruction
2. Define and create a thread
3. Submit the thread creation instruction

## ⚓️ Anchor Program

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

### Setup

#### Creating a new anchor project

```bash
anchor init hello_clockwork
```

Before going further, let's make sure Anchor and Solana work correctly on your machine

```bash
anchor build
```

Next, let's do the usual dance of replacing the program anchor id, note the program id of your anchor program by running

```bash
solana address -k target/deploy/hello_clockwork-keypair.json
```

Then replace the address in `Anchor.toml`

```toml
[programs.devnet]
hello_clockwork = "YOUR_PROGRAM_ID"
```

And also in `programs/hello_clockwork/src/lib.rs`

```rust
declare_id!("YOUR_PROGRAM_ID");
```

Finally, before going further. It's important to make sure everything so far is working so you can distinguish in the future if you have a clockwork error or a solana/anchor error. So let's deploy to devnet.

```bash
solana config set -u devnet
solana airdrop 1 # (as many times as needed)
anchor deploy
```

If everything is fine so far, you should get

```bash
Program Id: YOUR_PROGRAM_ID
Deploy success
```

### ⛓ On-Chain - Hello Instruction

Now that you got a working anchor project let's start modifying it. Replace the content of `programs/hello_clockwork/src/lib.rs` with

```rust
use anchor_lang::prelude::*;

declare_id!("YOUR_PROGRAM_ID");

#[program]
pub mod hello_clockwork {
    use super::*;

    pub fn hello_ix(_ctx: Context<HelloClockwork>, name: String) -> Result<()> {
        msg!(
            "Hello {}! The current time is: {}",
            name,
            Clock::get().unwrap().unix_timestamp
        );
        Ok(())
    }
}

#[derive(Accounts)]
#[instruction(name: String)]
pub struct HelloClockwork {}
```

```bash
anchor deploy
```

{% hint style="info" %}
Set your cluster to devnet if not already with

`solana config set -u devnet`
{% endhint %}

### Off-Chain -  Setup

_In this tutorial, we will use rust for our Solana client, if you are using typescript please refer to TODO._

#### Clockwork for Rust Clients

We have deployed our program with the `hello` instruction. Let's start "calling" our program by interacting with the Solana RPC. Create a new cargo package with `cargo new rust-client`, and update `Cargo.toml` with

```toml
[package]
name = "rust-client"
version = "0.1.0"
edition = "2021"

[dependencies]
hello_clockwork = { path = "../programs/hello_clockwork", features = ["no-entrypoint"], version = "0.1.0" }
clockwork-client = { version = "1.4.0" }
clockwork-utils = { version = "1.4.0" }
anchor-lang = "0.26.0"
solana-cli-config = "1.10.4"
solana-sdk = "1.13.5"

```

{% hint style="info" %}
Note that since we are also using rust for the client, we don't need to use IDLs, and can directly reference the hello-clockwork anchor program.
{% endhint %}

#### Cargo workspace

Let's modify the root `Cargo.toml` to include the rust-client `Cargo.toml`

```toml
[workspace]
members = [
    "programs/*",
    "rust-client" // 👈 add this
]
```

Let's make sure the project is configured properly

```bash
cargo build
```



### Off-Chain (Rust) - "Calling" Our Hello Instruction

### Solana Rust Client

Let's start by adding the basic imports for Solana clients&#x20;

```rust
use {
    solana_sdk::{
        signature::read_keypair_file, transaction::Transaction
    },
    anchor_lang::{
        solana_program::{
            instruction::Instruction,
            native_token::LAMPORTS_PER_SOL,
        },
        InstructionData,
    },
};
```

We will be interacting with Solana RPC, so we need to configure an RPC client with:

1. A keypair.
2. A cluster to target (devnet, testnet, etc).

```rust
use {
    ...
    clockwork_client::{
        Client, ClientResult,
    },
    clockwork_utils::{
        explorer::Explorer
    },
}

fn default_client() -> Client {
    let config_file = solana_cli_config::CONFIG_FILE.as_ref().unwrap().as_str();
    let config = solana_cli_config::Config::load(config_file).unwrap();
    let payer = read_keypair_file(&config.keypair_path).unwrap();
    let host = &config.json_rpc_url;
    Client::new(payer, host.into())
}

fn main() -> ClientResult<()> {
    // 0. Creating a Client with your default paper keypair as payer
    let client = default_client();
    client.airdrop(&client.payer_pubkey(), 2 * LAMPORTS_PER_SOL)?;

    Ok(())
}
```

1. We import `clockwork_client` to help us build an RPC client
2. We use this little helper `default_client` to initialize a client with your config as specified by `solana config get`
3. We initialize and airdrop some SOL to this client&#x20;

{% hint style="info" %}
By the way, make sure to use devnet if not already with `solana config set -u devnet`
{% endhint %}

#### Testing Our Instruction

```rust
use {
    ...
};

fn default_client() -> Client {
    ...
}

fn send_and_confirm_ix<S: std::fmt::Display>(
    client: &Client, ix: Instruction, label: S
) -> ClientResult<()> {
    // Create tx
    let mut tx = Transaction::new_with_payer(&[ix], Some(&client.payer_pubkey()));
    tx.sign(&[client.payer()], client.latest_blockhash().unwrap());

    // Send and confirm tx
    match client.send_and_confirm_transaction(&tx) {
        Ok(sig) => println!("{} tx: ✅ {}", label, Explorer::devnet().tx_url(sig)),
        Err(err) => println!("{} tx: ❌ {:#?}", label, err),
    }
    Ok(())
}

fn main() -> ClientResult<()> {
    // 0. Creating a Client with your default paper keypair as payer
    let client = default_client();
    client.airdrop(&client.payer_pubkey(), 2 * LAMPORTS_PER_SOL)?;

    // 1. Build the Hello Instruction
    let hello_ix = Instruction {
        program_id: hello_clockwork::ID,
        accounts: vec![],
        data: hello_clockwork::instruction::HelloIx { name: "Chronos".into() }.data(),
    };

    send_and_confirm_ix(&client, hello_ix, "hello ix")?;

    Ok(())
}
```

```bash
cargo run
```

```bash
hello ix tx: ✅ https://explorer.solana.com/tx/2DRSktpmib1BKtGn5KF1vbxXwobZ8n9RfefJ84UQbVKwY1SezhRaVqHpTYhhdSHzS4nm5dajKvUzGcgLYJ15ymAG?cluster=devnet
```

### 🔮 Observe - Let's See What's Happening

Let's use the Solana explorer, to check the program logs, here's the link: [https://explorer.solana.com/tx/2DRSktpmib1BKtGn5KF1vbxXwobZ8n9RfefJ84UQbVKwY1SezhRaVqHpTYhhdSHzS4nm5dajKvUzGcgLYJ15ymAG?cluster=devnet](https://explorer.solana.com/tx/2DRSktpmib1BKtGn5KF1vbxXwobZ8n9RfefJ84UQbVKwY1SezhRaVqHpTYhhdSHzS4nm5dajKvUzGcgLYJ15ymAG?cluster=devnet)

So far, we haven't done anything with clockwork yet. Although we imported some of the clockwork crates, it was just to help us build instructions in Rust.&#x20;

## ⏳ The Clockwork Way

In the last section, we deployed a simple Anchor Program and called the instruction. All of this is to make sure we are on the same page. Now, let's get into why you came here for. We will turn this simple instruction into a Threadable Instruction.

Before going further, make sure that:

* Your Solana installation works
* You are using devnet `solana config set -u devnet`
* You can deploy a simple anchor program

### Off-Chain - Creating a Thread

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

> 💡 Paradigm Shift #1: We create a Thread

```rust
use {
    ...
    clockwork_client::{
        Client, ClientResult,
        thread::{
            instruction::thread_create,
        },
    },
    ...
}

fn main() -> ClientResult<()> {
    // 0. Creating a Client with your default paper keypair as payer
    ...

    // 1. Build the Hello Instruction
    let hello_ix = ...
    
    // 2. Build Thread Creation Instruction
    let thread_create_ix = thread_create(...)

    send_and_confirm_ix(&client, thread_create_ix, "thread create ix")?;
}
```

* Import the `thread_create` helper
* Here's the documation about \[thread\_crate()]\([https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/instruction/fn.thread\_create.html](https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/instruction/fn.thread\_create.html))

### What does A Thread Need?

There are several parameters, let's go step by step, and think about them logically.&#x20;

#### Thread -> Instruction -> The What

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

> 💡 Paradigm Shift #2: The instruction is run by our Thread

First and foremost, a Thread needs to know what to run, a Thread run instructions, that's the "Threadable Instruction" we have been talking all along. Let's feed it the `hello_ix` instruction, that's why we had to deploy an Anchor Program earlier.

```rust
use {
    ...
    clockwork_client::{
        Client, ClientResult,
        thread::{
            instruction::thread_create,
            ID as thread_program_ID,
            state::{Thread, Trigger},
        },
    },
    ...
}

fn main() -> ClientResult<()> {
    // 0. Creating a Client with your default paper keypair as payer
    ...

    // 1. Build the Hello Instruction
    let hello_ix = ...
    
    // 2. Build Thread Creation Instruction
    let thread_create_ix = thread_create(
        hello_ix.into(), 👈
    );

    send_and_confirm_ix(&client, thread_create_ix, "thread create ix")?;
}
```

#### Thread -> Trigger -> The When/How

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

💡 Paradigm Shift #3: a Thread can be triggered by time or conditions

Then the question is how do we want the Thread to run the instruction? That's the t`rigger`

Today, we have two types of triggers:

* \[Time based]\([https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Cron](https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Cron))
* \[Account based]\([https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Account](https://docs.rs/clockwork-client/1.4.0/clockwork\_client/thread/state/enum.Trigger.html#variant.Account))

We will talk more about triggers in other examples. For now, let's use a time-based one, so that our Thread runs the instruction \[at every 10th minutes]\([https://crontab.guru/every-10-minutes](https://crontab.guru/every-10-minutes)).

```rust
fn main() -> ClientResult<()> {
    // 1. Build the Hello Instruction
    ...
     
    // 2. Define Thread Params
    // Trigger
    let trigger = Trigger::Cron {
        schedule: "*/10 * * * * * *".into(),
        skippable: true,
    };
    
    // 3. Create Thread
    let thread_create_ix = thread_create(
        hello_ix.into(),
        trigger 👈
    );
    ...
}
```

#### Thread -> Account





We forgot the Solana mantra: _"In solana everything is a... account!"._ A Thread is an account, so we need to give an address that will be used for this account, and also who will pay for that account.

```rust
fn main() -> ClientResult<()> {
    ...
    // 2. Define Thread Params
    // Trigger
    ...
    
    // Accounts
    let thread_label = "hello_clockwork".to_string();
    let payer = client.payer_pubkey();
    let thread_address = Thread::pubkey(thread_authority, thread_label.clone());
    
    // 2. Create Thread
    let thread_create_ix = thread_create(
        thread_label, 👈 
        hello_ix.into(),
        payer, 👈
        thread_address, 👈
        trigger,
    );
    ...
}
```

* `thread_label:` an identifier for the thread _(can also use buffer or vec u8)_
* `thread_address:` \[a program derived address]\([https://docs.rs/clockwork-thread-program/1.4.0/src/clockwork\_thread\_program/state/thread.rs.html#65](https://docs.rs/clockwork-thread-program/1.4.0/src/clockwork\_thread\_program/state/thread.rs.html#65))

```rust
fn main() -> ClientResult<()> {
    ...
    // 2. Define Thread Params
    // Trigger
    ...
    
    // Security
    let thread_authority = client.payer_pubkey();
    
    // Accounts
    let thread_label = "hello_clockwork".to_string();
    let payer = client.payer_pubkey();
    let thread_address = Thread::pubkey(thread_authority, thread_label.clone());
    
    // 3. Create Thread Instruction
    let thread_create_ix = thread_create(
        thread_authority, 👈
        thread_label,
        hello_ix.into(),
        payer,
        thread_address,
        trigger,
    );
    ...
}
```

#### Thread -> Authority

Let's talk security, the only remaining piece is the thread authority that I omitted earlier.&#x20;

```rust
fn main() -> ClientResult<()> {
    ...
    // 2. Define Thread Params
    // Trigger
    ...
    
    // Security
    let thread_authority = client.payer_pubkey();
    
    // Accounts
    let thread_label = "my_first_thread".to_string();
    let payer = client.payer_pubkey();
    let thread_address = Thread::pubkey(thread_authority, thread_label.clone());
    
    // 3. Create Thread Instruction
    let thread_create_ix = thread_create(
        thread_authority, 👈
        thread_label,
        hello_ix.into(),
        payer,
        thread_address,
        trigger,
    );
    ...
}
```

* `thread_authority` the signing authority for the thread account. You will need the corresponding private key to sign any mutation to your Thread account. For example, if you need to delete/stop/pause/resume this Thread via the `clockwork thread` cli command (more on this later).
* To easily debug, let's use `client.payer_pubkey()` (which is your default keypair.)

### 📙 Recap

```rust
fn main() -> ClientResult<()> {
    // 0. Creating a Client with your default paper keypair as payer
    let client = default_client();
    client.airdrop(&client.payer_pubkey(), 2 * LAMPORTS_PER_SOL)?;

    // 1. Build the Instruction to be executed by the Thread
    let hello_ix = Instruction {
        program_id: hello_clockwork::ID,
        accounts: vec![],
        data: hello_clockwork::instruction::HelloIx { name: "Chronos".into() }.data(),
    };

    // 2 Define Thread Params
    // Trigger
    let trigger = Trigger::Cron {
        schedule: "*/10 * * * * * *".into(),
        skippable: true,
    };

    // Security
    let thread_authority = client.payer_pubkey();

    // Accounts
    let thread_label = "my_first_thread".to_string();
    let payer = client.payer_pubkey();
    let thread_address = Thread::pubkey(thread_authority, thread_label.clone());

    // 3. Create Thread Instruction
    let thread_create_ix = thread_create(
        thread_authority,  // thread signer
        thread_label,      // identifier
        hello_ix.into(),   // instruction to run
        payer,             // payer
        thread_address,    // address to assign to the thread account
        trigger            // when to run the thread
    );

    send_and_confirm_ix(&client, thread_create_ix, "thread create ix")?;
    println!(
        "thread: 🔗 https://explorer.solana.com/address/{}?cluster=devnet",
        thread_address,
    );
    Ok(())
}
```

1. We prepared the instruction to be run by our Thread
2. We defined parameters for our Thread
   1. The "What"; the instruction
   2. The "When"; the trigger
   3. The accounts
   4. The authority
3. We build the create thread Instruction using the `thread_create()` helper



### 🔮 Observe - Let's See What's Happening

```bash
cargo run
```

{% hint style="info" %}
You might run into a `ThreadCreate - Instruction - "Address already in use"`. It just means the `thread_address` is already in use, you just need to give modify the `thread_label` or provide a different `thread_address`.
{% endhint %}

```bash
thread create ix tx: ✅ https://explorer.solana.com/tx/2ciV5DLJBgS7tdzYkJ6jXCtDywL7CagipXyjbJQLLp5A7bWVdod143PV23N9gBYDvxWYv1r2cxezWGC1r4xVEKTw?cluster=devnet
thread: 🔗 https://explorer.solana.com/address/5wSdMBSVbYrBPShe9DQssdDv9bQsrCh2YSfvnGhBu7bv?cluster=devnet
```

You can use the command line to check the logs with

```bash
solana logs -u devnet | grep -A 10 YOUR_PROGRAM_ID
```

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

Or use the Solana explorer, here's the result with my program: https://explorer.solana.com/address/5wSdMBSVbYrBPShe9DQssdDv9bQsrCh2YSfvnGhBu7bv?cluster=devnet

<figure><img src="../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

Our Instruction is now Threadable 🧵! As you can see, our instruction is run every 10 seconds by our Thread, and does not need to be called manually anymore!

## 📙 The Clockwork Cookbook

1. Deploy an instruction (and so a program)
2. Create a thread and
   1. Feed it an instruction to run
   2. Give it conditions to run, time-based or conditions based
   3. Tweak the different accounts, addresses, authority, etc
3. Submit that create thread transaction
4. Observe your thread come to life without you having to touch anything

### ⚠️ Security

Finally a note on security. For some reason, you might not want your instruction to be run by anyone or anything than your own Thread. In that case, we can add an anchor constraint and provide the thread address to our instruction.



In your anchor program:

```rust
#[derive(Accounts)]
#[instruction(name: String)]
pub struct HelloClockwork<'info> {
    #[account(address = thread.pubkey(), signer)]
    pub thread: Account<'info, Thread>,
}
```

In the rust client:

```rust
// Create ix
let hello_world_ix = Instruction {
    program_id: hello_clockwork::ID,
    accounts: vec![AccountMeta::new(thread_address, true)],
    data: hello_clockwork::instruction::HelloWorld { name: "Chronos".into() }.data(),
};
```



## 🧐 How Does it Actually Work?

<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

