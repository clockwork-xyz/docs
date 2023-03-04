# Localnet

## 1. Compile from source

Currently, the fastest way to get up and running with a Clockwork localnet is to clone the repo, checkout a latest release version, and build the toolkit from scratch.

```bash
git clone https://github.com/clockwork-xyz/clockwork
cd clockwork
git checkout tags/$(git describe --tags)
./scripts/build-all.sh . && export PATH=$PWD/bin:$PATH
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

## Stream logs

To stream logs from your deployed program, you can use the Solana CLI.

```bash
solana logs --url localhost
```
