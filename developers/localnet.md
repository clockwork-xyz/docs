# Localnet

## Deploy localnet

To deploy Clockwork on your machine for local development, you must first [**install the CLI**](../about/installation.md). When this is done, you can launch a local Clockwork instance with the following command:

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

To stream logs from your localnet, you can use the Solana CLI.

```bash
solana logs --url l
```
