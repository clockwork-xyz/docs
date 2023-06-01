# 1. Scheduling an SPL Transfer

## What you will learn

In this guide, you will learn how to schedule an SPL token transfer using Clockwork. This example will demonstrate many key concepts of working with Clockwork:

1. How the Clockwork program model works.
2. How to sign transactions with threads.
3. How to monitor an automation.

{% hint style="info" %}
All code in this guide is open-source and and free to fork [**on Github**](https://github.com/clockwork-xyz/examples)**.**
{% endhint %}

## 1. The Clockwork programming model

Let's start with the big picture. Solana is a really fast, globally distributed computer. Just as programs on a traditional computer needs to be able to execute an automated series of instructions, so do programs on Solana. Clockwork threads are an automation primitive analogous to [**computer threads**](https://en.wikipedia.org/wiki/Thread\_\(computing\)) that developers can use to automate programs on Solana. In simple terms, this means we can point Clockwork at any Solana program to automate it. A model of this relationship is presented in the diagram below.&#x20;

![The Clockwork programming model.](https://user-images.githubusercontent.com/8634334/222291232-ce195a01-7bdc-4567-8907-14485d19ee91.png)



## 2. Setting up our project

Let's begin by creating a new vanilla Node Typescript project:

```sh
mkdir spl_transfer
cd spl_transfer
```

Create a new `tsconfig.json` file:

```tsconfig
{
  "compilerOptions": {
    "lib": ["es2015"],
    "module": "commonjs",
    "target": "es6",
    "esModuleInterop": true
  }
}
```

Create a new `package.json` file with the below content. The main dependencies you really need in your project are:

* `@clockwork-xyz/sdk` for interacting with Clockwork.
* `@solana/spl-token` to build token transfer instructions.

```json
{
  "name": "spl-transfer",
  "version": "1.0.0",
  "description": "SPL Transfer Example",
  "scripts": {
    "test": "yarn run ts-mocha -p tsconfig.json -t 1000000 main.ts"
  },
  "dependencies": {
    "@clockwork-xyz/sdk": "^0.3.4",
    "@coral-xyz/anchor": "^0.27.0",
    "@solana/spl-token": "^0.3.6",
    "@solana/web3.js": "^1.73.0"
  },
  "devDependencies": {
    "@types/chai": "^4.3.3",
    "@types/mocha": "^9.1.1",
    "chai": "^4.3.6",
    "mocha": "^10.0.0",
    "ts-mocha": "^10.0.0",
    "typescript": "^4.8.3"
  }
}
```

Install the dependencies by running:

```
yarn
```

Now, let's scaffold a simple test in `main.ts`:

```typescript
import { expect } from "chai";
import {
  Connection,
  Keypair,
  LAMPORTS_PER_SOL,
  PublicKey,
  Transaction,
} from "@solana/web3.js";
import {
  createMint,
  getAccount,
  getOrCreateAssociatedTokenAccount,
  createTransferInstruction,
  mintTo,
} from "@solana/spl-token";
import { AnchorProvider } from "@coral-xyz/anchor";
import NodeWallet from "@coral-xyz/anchor/dist/cjs/nodewallet";
import { ClockworkProvider } from "@clockwork-xyz/sdk";


describe("spl-transfer", async () => {
  it("It transfers tokens every 10s", async () => {
    const connection = new Connection("http://localhost:8899", "processed");
    const payer = keypairFromFile(
      require("os").homedir() + "/.config/solana/id.json"
    );

    // Prepare clockworkProvider
    const provider = new AnchorProvider(
      connection,
      new NodeWallet(payer),
      AnchorProvider.defaultOptions()
    );
    const clockworkProvider = ClockworkProvider.fromAnchorProvider(provider);
  });
});
```

* We use your default paper keypair as the payer, this of course will change depending on your use case.
* Finally, we initialize a `ClockworkProvider`. This will be required later to create your thread.

## 3. Signing with threads

In this section, we will focus on how signing works with Threads. But first, let's prepare the accounts needed for the SPL Transfer Instruction to be scheduled. If you ever worked with Solana, you might know by now that SPL Transfers don't happen between system accounts, but instead between associated token accounts.

```typescript
/**
 * Construct a Transfer instruction
 *
 * @param source       Source account (ATA)
 * @param destination  Destination account (ATA)
 * @param owner        Owner of the source account
 * @param amount       Number of tokens to transfer
 * ...
 *
 * @return Instruction to add to a transaction
 */
export function createTransferInstruction(
    source: PublicKey,
    destination: PublicKey,
    owner: PublicKey,
    amount: number | bigint,
    ...
): TransactionInstruction
```

&#x20;Let's start by creating the associated token account for the recipient account.

> In another guide, we will see how to lazily create the recipient ata.

```typescript
describe("spl-transfer", async () => {
  it("It transfers tokens every 10s", async () => {
    ...
    
    // Prepare dest
    const dest = Keypair.generate().publicKey;
    const destAta = (await getOrCreateAssociatedTokenAccount(
      connection,
      payer,
      mint,        // the address of the mint
      dest,
      false        // is dest a pda?
    )).address;
    console.log(`dest: ${dest}, destAta: ${destAta}`);    
  });
});
```

Then, let's start do the same with the source account. I have already prepared a function called `fundSource` which helps fund our source account with some SPL token. You probably won't need this in a real world scenario.

```typescript
describe("spl-transfer", async () => {
  it("It transfers tokens every 10s", async () => {
    // Prepare dest
    ...
    
    // Prepare source
    const source = ?
    const [sourceAta] = await fundSource(connection, payer, source);
    console.log(`source: ${source}, sourceAta: ${sourceAta}`);
  });
});
```



**Let's talk about the elephant in the room, who should be the source account?**

When doing a a transfer we need to deduct fund and authorize this debit, thus source should be a signer. This works fine in a traditional scenario, you provide the signer when submitting the transaction and voila!

When working with Threads, we schedule our instructions to be executed by Threads, more precisely by the Clockwork thread program. For this reason, the signer for your automated instruction is actually your **thread**:

```typescript
  it("It transfers tokens every 10s", async () => {
    // Prepare dest
    ...
    
    // Prepare source
    const threadId = "spljs" + new Date().getTime();
    const [thread] = clockworkProvider.getThreadPDA(
      provider.wallet.publicKey,  // thread authority
      threadId                    // thread id
    );
    console.log(`Thread id: ${threadId}, address: ${thread}`);

    // We will use the thread pda as the source and fund it with some tokens
    const source = thread;
    const [sourceAta] = await fundSource(connection, payer, source);
    console.log(`source: ${source}, sourceAta: ${sourceAta}`);
  });
});    
```

## 4. Scheduling a SPL token transfer instruction

Now that we have the ingredients in place, we can finally build our SPL token transfer instruction and schedule a thread to run this instruction:

```typescript
it("Transfers SOL every 10 seconds", async () => {
  ...
  
  // 1️⃣ Build the SPL Transfer Instruction
  const targetIx = createTransferInstruction(sourceAta, destAta, source, amount);

  // 2️⃣  Define a trigger condition for the thread.
  const trigger = {
    cron: {
      schedule: "*/10 * * * * * *",
      skippable: true,
    },
  };

  // 3️⃣  Create the thread.
  const ix = await clockworkProvider.threadCreate(
    provider.wallet.publicKey,    // authority
    threadId,                     // id
    [targetIx],                   // instructions to execute
    trigger,                      // trigger condition
    LAMPORTS_PER_SOL              // amount to fund the thread with for execution fees
  );
  const tx = new Transaction().add(ix);
  const sig = await clockworkProvider.anchorProvider.sendAndConfirm(tx);
  console.log(`Thread created: ${sig}`);
});
```

We can see the `threadCreate` function asks for 5 arguments. These include some basic information needed to initialize the thread account.

* `authority` – The owner of the thread. This account must be the transaction signer and will have permission to delete, pause, resume, stop, and update the thread.
* `id` – An identifier for the thread (can also use buffer or vec u8).
* `instructions` – The list of instructions to execute when the trigger condition becomes valid.
* `trigger` – The trigger condition for the thread. When this condition is valid, the thread will begin executing the provided instructions. You can read more about [triggers](https://docs.clockwork.xyz/developers/threads/triggers).
* `amount` – The number of lamports to fund the thread account with. Remember to provide a small amount of SOL. Read more about how fees are calculated [here](https://docs.clockwork.xyz/developers/threads/fees).

## 5. Running the tests

Now we need to get our app running. If you have not done so already, you will need to install the Clockwork CLI by running the cargo command below. If you face any trouble here, please refer to the [**installation**](../welcome/installation.md) docs.&#x20;

```shell
cargo install -f --locked clockwork-cli
```

Now that we have Clockwork installed, we can go ahead and spin up a local Clockwork node:

```bash
clockwork localnet
```

In a separate terminal window, we'll run the test:

```bash
anchor test --skip-local-validator
```

## 6. Monitoring our automation

If you setup everything correctly, you can now watch your automated program run all on its own. Grab the Clockwork explorer link that was printed out to the console. Using the Clockwork explorer, you can get simulation logs and inspect if your thread is not running and why. For example, here's mine: [https://app.clockwork.xyz/threads/GB7YgYK3bKF8J4Rr9Z2oeA3hwxrJdvW5zgXuNaxWWmUF?cluster=devnet](https://app.clockwork.xyz/threads/GB7YgYK3bKF8J4Rr9Z2oeA3hwxrJdvW5zgXuNaxWWmUF?cluster=devnet)

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Of course you can also look up your thread account in your favorite Solana explorer. You can alternatively use the Solana CLI to stream program logs by running the command provided below. Here's [**an example thread**](https://explorer.solana.com/address/3ohRKgNyLS1iTGiUqnzoiFiQcrCLGmr3NWHzq4HW8BdJ?cluster=devnet) that was created in a test on May 24th, 2023.

```bash
solana logs -u devnet YOUR_PROGRAM_ID
```

<figure><img src="https://user-images.githubusercontent.com/8634334/222591908-bbaa04c5-83b4-46c2-b83b-68e1fef473eb.png" alt=""><figcaption></figcaption></figure>

## Key insights

1. Threads are an automation primitive for Solana.
2. You can use threads to automate any program instruction on Solana.&#x20;
3. [**Triggers**](../reference/threads/triggers.md) allow you to define when a thread should begin execution.
4. Threads must be funded with a small amount of SOL to pay for [**automation fees**](../reference/threads/fees.md).&#x20;
5. The signer for your instruction is your thread pda.

## Appendix

This guide was written using the following environment dependencies.

<table><thead><tr><th width="340">Dependency</th><th>Version</th></tr></thead><tbody><tr><td>Anchor</td><td>v0.27.0</td></tr><tr><td>Clockwork</td><td>v2.0.17</td></tr><tr><td>Clockwork TS SDK</td><td>v0.3.4</td></tr><tr><td>Rust</td><td>v1.65.0</td></tr><tr><td>Solana</td><td>v1.14.16</td></tr><tr><td>Ubuntu</td><td>v20.04</td></tr></tbody></table>

## Continue learning

* A complete copy of all code provided in this guide can be found in the [**examples repo**](https://github.com/clockwork-xyz/examples) on GitHub.
* Ask questions on [**Discord**](https://discord.gg/epHsTsnUre).
