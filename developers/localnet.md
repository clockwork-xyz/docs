# Localnet

## 1. Install the clockwork-cli

If you are on linux, you might need to run this:

```sh
sudo apt-get update && sudo apt-get upgrade && sudo apt-get install -y pkg-config build-essential libudev-dev libssl-dev
```

Install with cargo:

```sh
cargo install -f --locked clockwork-cli
```

## 2. Deploy a localnet

First, remember to configure your Solana CLI for local development:

```bash
solana config set --url localhost
```

Now you can use the Clockwork CLI to launch a localnet with all the Clockwork plugin and programs initialized and ready to go:

```bash
clockwork localnet
```

To deploy your own programs to the localnet, you can use either the Anchor CLI, Clockwork CLI, or Solana CLI. As a shortcut, the Clockwork CLI provides a `--bpf-program` flag for deploying the localnet with a pre-built program binary.&#x20;

{% tabs %}
{% tab title="Anchor" %}
```
anchor deploy
```
{% endtab %}

{% tab title="Clockwork" %}
```
clockwork localnet \
    --bpf-program <ADDRESS_OR_KEYPAIR> <PROGRAM_FILEPATH>
```
{% endtab %}

{% tab title="Solana" %}
```bash
solana program deploy \ 
    --program-id <ADDRESS_OR_KEYPAIR> <PROGRAM_FILEPATH>
```
{% endtab %}
{% endtabs %}

## 3. Stream logs

To stream logs from your deployed program, you can use the Solana CLI.

```bash
solana logs --url localhost
```
