# Vultr

Lifevault stores an existing Vultr API credential in the local vault with the
`vultr` bundle and injects it into commands with `run`. Lifevault does not
create, scope or rotate Vultr credentials, and the broker has no Vultr
adapter.

## Prerequisites

- A Vultr account.
- Only the `lifevault` binary to store the credential. Install the Vultr CLI or
  other tools you want to run with it separately.

## Create a least-privilege credential

Vultr API keys are not scoped to individual resources. Limit exposure by
restricting API access to the IP ranges that use it, and consider a separate
user with only the permissions the tool needs. Rotate the key in the customer
portal if it leaks.

## Store it

Recommended: the guided form opens the Vultr API settings page in your browser,
asks for the token at a hidden prompt, checks it with one read-only request
(`GET /v2/account`) and stores it. A rejected token saves nothing.

```sh
lifevault import-provider vultr --connect
lifevault import-provider vultr --connect --prefix STAGING_   # optional prefix
```

`VULTR_API_KEY` is used as-is when already set in your environment.
`--no-browser` (or `LIFEVAULT_NO_BROWSER`) prints the link instead of opening
it; `--no-verify` skips the check.

From the environment, without prompts or requests:

```sh
read -rs VULTR_API_KEY && export VULTR_API_KEY
lifevault import-provider vultr
unset VULTR_API_KEY
```

| Variable | Required | Stored as |
|---|---|---|
| `VULTR_API_KEY` | yes | `VULTR_API_KEY`, or `<PREFIX>VULTR_API_KEY` with `--prefix <PREFIX>` |

For a single variable, `lifevault set VULTR_API_KEY` (hidden prompt) gives the same
result, without the token check.

## Use it

```sh
lifevault run VULTR_API_KEY -- vultr-cli instance list
lifevault run VULTR_API_KEY -- terraform plan
```

Keep several accounts apart with a prefix and map the name back:

```sh
lifevault import-provider vultr --prefix STAGING_
lifevault run STAGING_VULTR_API_KEY=VULTR_API_KEY -- vultr-cli instance list
```

To make the credential part of a project, use a `PROJECT__` prefix
(`--prefix MY_APP__`). `lifevault run --project my-app -- <cmd>` then injects it as
`VULTR_API_KEY`, and project push targets receive it too.

## Updating

- Imports without `--connect` make no Vultr API calls; `--connect` makes one
  read-only check. No import creates, rotates or refreshes credentials, and
  `--auto-refresh` does not apply.
- After rotating at Vultr, run `lifevault import-provider vultr --connect
  --replace` (or export the new value and import again with `--replace`).
  Without `--replace`, an existing name is an error.
- Values must be nonempty UTF-8. The bundle is saved in one write; a missing
  variable or collision leaves the vault unchanged.