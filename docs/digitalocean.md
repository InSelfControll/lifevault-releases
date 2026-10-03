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

```sh
read -rs DIGITALOCEAN_TOKEN && export DIGITALOCEAN_TOKEN
lifevault import-provider digitalocean
unset DIGITALOCEAN_TOKEN
```

| Variable | Required | Stored as |
|---|---|---|
| `DIGITALOCEAN_TOKEN` | yes | `DIGITALOCEAN_TOKEN`, or `<PREFIX>DIGITALOCEAN_TOKEN` with `--prefix <PREFIX>` |

For a single variable, `lifevault set DIGITALOCEAN_TOKEN` (hidden prompt) gives the same
result. `import-provider` is useful when the value is already in your environment.

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

- Imports make no DigitalOcean API calls. They do not create, rotate or refresh
  credentials, and `--auto-refresh` does not apply.
- After rotating at DigitalOcean, export the new value and import again with
  `--replace`. Without `--replace`, an existing name is an error.
- Values must be nonempty UTF-8. The bundle is saved in one write; a missing
  variable or collision leaves the vault unchanged.
