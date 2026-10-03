# DigitalOcean

Lifevault stores an existing DigitalOcean API credential in the local vault with the
`digitalocean` bundle and injects it into commands with `run`. Lifevault does not
create, scope or rotate DigitalOcean credentials, and the broker has no DigitalOcean
adapter.

## Prerequisites

- A DigitalOcean account.
- Only the `lifevault` binary to store the credential. Install the DigitalOcean CLI or
  other tools you want to run with it separately.

## Create a least-privilege credential

Create a personal access token in the DigitalOcean control panel (API section).
Give it an expiry and, where offered, custom scopes limited to the resources
your tool manages. Use one token per tool or environment so each can be revoked
on its own.

## Store it

Recommended: the guided form opens the DigitalOcean API tokens page in your
browser, asks for the token at a hidden prompt, checks it with one read-only
request (`GET /v2/account`) and stores it. A rejected token saves nothing.

```sh
lifevault import-provider digitalocean --connect
lifevault import-provider digitalocean --connect --prefix STAGING_   # optional prefix
```

`DIGITALOCEAN_TOKEN` is used as-is when already set in your environment.
`--no-browser` (or `LIFEVAULT_NO_BROWSER`) prints the link instead of opening
it; `--no-verify` skips the check.

From the environment, without prompts or requests:

```sh
read -rs DIGITALOCEAN_TOKEN && export DIGITALOCEAN_TOKEN
lifevault import-provider digitalocean
unset DIGITALOCEAN_TOKEN
```

| Variable | Required | Stored as |
|---|---|---|
| `DIGITALOCEAN_TOKEN` | yes | `DIGITALOCEAN_TOKEN`, or `<PREFIX>DIGITALOCEAN_TOKEN` with `--prefix <PREFIX>` |

For a single variable, `lifevault set DIGITALOCEAN_TOKEN` (hidden prompt) gives the same
result, without the token check.

## Use it

Tools that read `DIGITALOCEAN_TOKEN` (for example the Terraform provider) use the
stored name directly. `doctl` reads `DIGITALOCEAN_ACCESS_TOKEN`, so map it:

```sh
lifevault run DIGITALOCEAN_TOKEN -- terraform plan
lifevault run DIGITALOCEAN_TOKEN=DIGITALOCEAN_ACCESS_TOKEN -- doctl compute droplet list
```

Keep several accounts apart with a prefix and map the name back:

```sh
lifevault import-provider digitalocean --prefix STAGING_
lifevault run STAGING_DIGITALOCEAN_TOKEN=DIGITALOCEAN_TOKEN -- terraform plan
```

To make the credential part of a project, use a `PROJECT__` prefix
(`--prefix MY_APP__`). `lifevault run --project my-app -- <cmd>` then injects it as
`DIGITALOCEAN_TOKEN`, and project push targets receive it too.

## Updating

- Imports without `--connect` make no DigitalOcean API calls; `--connect` makes
  one read-only check. No import creates, rotates or refreshes credentials, and
  `--auto-refresh` does not apply.
- After rotating at DigitalOcean, run `lifevault import-provider digitalocean
  --connect --replace` (or export the new value and import again with
  `--replace`). Without `--replace`, an existing name is an error.
- Values must be nonempty UTF-8. The bundle is saved in one write; a missing
  variable or collision leaves the vault unchanged.