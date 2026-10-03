# Linode (Akamai)

Lifevault stores an existing Linode (Akamai) API credential in the local vault with the
`linode` bundle and injects it into commands with `run`. Lifevault does not
create, scope or rotate Linode (Akamai) credentials, and the broker has no Linode (Akamai)
adapter.

## Prerequisites

- A Linode (Akamai) account.
- Only the `lifevault` binary to store the credential. Install the Linode (Akamai) CLI or
  other tools you want to run with it separately.

## Create a least-privilege credential

Create a personal access token in Cloud Manager. Give it an expiry and grant
only the services it needs, read-only where possible. Use one token per tool or
environment.

## Store it

```sh
read -rs LINODE_TOKEN && export LINODE_TOKEN
lifevault import-provider linode
unset LINODE_TOKEN
```

| Variable | Required | Stored as |
|---|---|---|
| `LINODE_TOKEN` | yes | `LINODE_TOKEN`, or `<PREFIX>LINODE_TOKEN` with `--prefix <PREFIX>` |

For a single variable, `lifevault set LINODE_TOKEN` (hidden prompt) gives the same
result. `import-provider` is useful when the value is already in your environment.

## Use it

Tools that read `LINODE_TOKEN` (for example the Terraform provider) use the
stored name directly. `linode-cli` reads `LINODE_CLI_TOKEN`, so map it:

```sh
lifevault run LINODE_TOKEN -- terraform plan
lifevault run LINODE_TOKEN=LINODE_CLI_TOKEN -- linode-cli linodes list
```

Keep several accounts apart with a prefix and map the name back:

```sh
lifevault import-provider linode --prefix STAGING_
lifevault run STAGING_LINODE_TOKEN=LINODE_TOKEN -- terraform plan
```

To make the credential part of a project, use a `PROJECT__` prefix
(`--prefix MY_APP__`). `lifevault run --project my-app -- <cmd>` then injects it as
`LINODE_TOKEN`, and project push targets receive it too.

## Updating

- Imports make no Linode (Akamai) API calls. They do not create, rotate or refresh
  credentials, and `--auto-refresh` does not apply.
- After rotating at Linode (Akamai), export the new value and import again with
  `--replace`. Without `--replace`, an existing name is an error.
- Values must be nonempty UTF-8. The bundle is saved in one write; a missing
  variable or collision leaves the vault unchanged.
