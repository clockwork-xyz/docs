# Quickstart

## Getting started

In this quickstart, you will learn how to automate a SOL transfer using Clockwork. We will install the Clockwork CLI and spawn a Clockwork thread from a Typescript application. Within 10 minutes, you will have your first automation up and running.

{% hint style="info" %}
All code in this quickstart is open-source and free to fork [**on Github**](https://github.com/clockwork-xyz/examples/tree/main/0-quickstart\_transfer)**.**
{% endhint %}

## 1. Create your first automation (5 min)

Let's begin by creating a new vanilla Node Typescript project:

```sh
mkdir clockwork_quickstart
cd clockwork_quickstart
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

Create a new `package.json` file with the below content. The main dependencies you really need in your project are the [**Clockwork SDK**](https://www.npmjs.com/package/@clockwork-xyz/sdk) and Anchor.

```json
{
  "scripts": {
    "test": "yarn run ts-mocha -p tsconfig.json -t 1000000 ./tests/main.ts"
  },
  "dependencies": {
    "@solana/web3.js": "^1.73.0",
    "@coral-xyz/anchor": "^0.27.0",
    "@clockwork-xyz/sdk": "^0.3.4"
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

After this, create a new file called `main.ts`. Import the necessary libraries and set up a Clockwork provider.

```ts
import { expect } from "chai";
import {
    Connection,
    Keypair,}
    LAMPORTS_PER_SOL,
    PublicKey,
    Transaction,
} from "@solana/web3.js";
import * as anchor from "@coral-xyz/anchor";
import NodeWallet from "@coral-xyz/anchor/dist/cjs/nodewallet";
import { ClockworkProvider, PAYER_PUBKEY } from "@clockwork-xyz/sdk";

const connection = new Connection("http://localhost:8899", "processed");
const payer = Keypair.fromSecretKey(
    Buffer.from(JSON.parse(require("fs").readFileSync(
        require("os").homedir() + "/.config/solana/id.json",
        "utf-8"
    )))
);

// Prepare clockworkProvider
const provider = new anchor.AnchorProvider(
    connection,
    new NodeWallet(payer),
    anchor.AnchorProvider.defaultOptions()
);
const clockworkProvider = ClockworkProvider.fromAnchorProvider(provider);
```



Next, we'll prepare a `SystemTransfer` instruction and automate it with a Clockwork thread. Threads are the basic building blocks of automations on Solana. Every thread has a trigger condition and a set of instructions to run. In the code sample below, we create a thread that schedules a SOL transfer to happen every 10 seconds.&#x20;

```typescript
...

describe("transfer", async () => {
    it("Transfers SOL every 10 seconds", async () => {
      const threadId = "sol_transferjs" + new Date().getTime();
      const [threadAddress] = clockworkProvider.getThreadPDA(
          payer.publicKey,   // authority
          threadId
       )
  
      const recipient = Keypair.generate().publicKey;
      console.log(`🫴  recipient: ${recipient.toString()}\n`);

      // 1️⃣  Prepare an instruction to be automated.
      const transferIx = SystemProgram.transfer({
          fromPubkey: PAYER_PUBKEY,
          toPubkey: recipient,
          lamports: LAMPORTS_PER_SOL,
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
          payer.publicKey,           // authority
          threadId,                  // id
          [transferIx],              // instructions
          trigger,                   // trigger
          LAMPORTS_PER_SOL,      // amount to fund the thread with
      );
      const tx = new Transaction().add(ix);
      const signature = await clockworkProvider.anchorProvider.sendAndConfirm(tx);
      console.log(`🗺️  explorer: https://app.clockwork.xyz/threads/${threadAddress}?cluster=custom&customUrl=${connection.rpcEndpoint}\n`);
      
      // Check balance of recipient address
      await new Promise((resolve) => setTimeout(resolve, 10 * 1000));
      let balance = await connection.getBalance(recipient) / LAMPORTS_PER_SOL;
      console.log(`✅ recipient balance: ${balance} SOL\n`);
      expect(balance).to.eq(1);

      await new Promise((resolve) => setTimeout(resolve, 10 * 1000));
      balance = await connection.getBalance(recipient) / LAMPORTS_PER_SOL;
      console.log(`✅ recipient balance: ${balance} SOL\n`);
      expect(balance).to.eq(2);
    });
});
```

We can see the `threadCreate` function asks for 5 arguments. These include some basic information needed to initialize the thread account.

* `authority` – The owner of the thread. This account must be the transaction signer and will have permission to delete, pause, resume, stop, and update the thread.
* `id` – An identifier for the thread (can also use buffer or vec u8).
* `instructions` – The list of instructions to execute when the trigger condition becomes valid.
* `trigger` – The trigger condition for the thread. When this condition is valid, the thread will begin executing the provided instructions.
* `amount` – The number of lamports to fund the thread account with. Remember to provide a small amount of SOL. Read more about how fees are calculated [**here**](https://docs.clockwork.xyz/developers/threads/fees).

## 2. Run the tests (2 min)

Now we need to get our app running. If you have not done so already, you will need to install the Clockwork CLI by running the cargo command below. If you face any trouble here, please refer to the [**installation**](installation.md) docs.&#x20;

```shell
cargo install -f --locked clockwork-cli
```

Now that we have Clockwork installed, we can go ahead and spin up a local Clockwork node:

```bash
clockwork localnet
```

In a separate terminal window, we'll run the test:

```bash
yarn test
```

And voila:

<figure><img src="../.gitbook/assets/Screenshot 2023-06-01 at 01.48.01.png" alt=""><figcaption></figcaption></figure>

## 3. Monitor your automation (1 min)

You can now watch your automation run all on its own. Grab the Clockwork explorer link that was printed out to the console. Using the Clockwork explorer, you can get simulation logs and inspect your thread. For example, here's mine: [https://app.clockwork.xyz/threads/GB7YgYK3bKF8J4Rr9Z2oeA3hwxrJdvW5zgXuNaxWWmUF?cluster=devnet](https://app.clockwork.xyz/threads/GB7YgYK3bKF8J4Rr9Z2oeA3hwxrJdvW5zgXuNaxWWmUF?cluster=devnet)

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Of course you can also look up your thread account in your favorite Solana explorer. You can alternatively use the Solana CLI to stream program logs by running the command provided below. Here's [**an example thread**](https://explorer.solana.com/address/3ohRKgNyLS1iTGiUqnzoiFiQcrCLGmr3NWHzq4HW8BdJ?cluster=devnet) that was created in a test on May 24th, 2023.

```bash
solana logs -u l YOUR_PROGRAM_ID
```

<figure><img src="https://user-images.githubusercontent.com/8634334/222591908-bbaa04c5-83b4-46c2-b83b-68e1fef473eb.png" alt=""><figcaption></figcaption></figure>

## Key insights

1. Threads are an automation primitive for Solana.
2. You can use threads to automate any program instruction on Solana.&#x20;
3. [**Triggers**](../reference/threads/triggers.md) allow you to define when a thread should begin execution.
4. Threads must be funded with a small amount of SOL to pay for [**automation fees**](../reference/threads/fees.md).&#x20;

## Continue learning

* Many more examples can be found in the [**guides section**](https://docs.clockwork.xyz/developers/guides)**.**
* Ask questions on [**Discord**](https://discord.gg/epHsTsnUre).
