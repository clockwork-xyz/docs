# Getting started

## Build from source

Currently, the fastest way to get started Clockwork is to download the open-source repo, checkout the latest release, and build the toolkit from source:

```bash
git clone https://github.com/clockwork-xyz/clockwork
cd clockwork
git checkout tags/$(git describe --tags)
./scripts/build-all.sh . && export PATH=$PWD/bin:$PATH
```

## Deploy to localnet

First, remember to configure your Solana CLI for local development:

```bash
solana config set --url localhost
```

Now you can use the Clockwork CLI to launch a localnet with the Clockwork plugin and programs initialized and ready to go:

```bash
clockwork localnet
```

To deploy a program to your localnet, you can use either the Anchor or Solana CLI toolkits. As a shortcut, the Clockwork CLI also provides a `--bpf-program` flag to deploy the localnet with a pre-built program binary.&#x20;

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

To stream logs from your deployed programs, you can use the Solana CLI.

```bash
solana logs --url localhost
```
