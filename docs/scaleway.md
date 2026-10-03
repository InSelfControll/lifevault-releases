# Scaleway

Lifevault stores an existing Scaleway API key pair, plus optional project,
region and zone defaults, in the local vault with the `scaleway` bundle. It
injects them into commands with `run`. Lifevault does not create, scope or rotate
Scaleway keys, and the broker has no Scaleway adapter.

## Prerequisites

- A Scaleway organization and project.
- Only the `lifevault` binary to store the keys. Install `scw`, Terraform or other
  tools separately.

## Create a least-privilege key

Create an IAM application for the tool, attach a policy with only the permission
sets it needs (scoped to one project where possible), and generate an API key for
that application. Avoid API keys tied to an owner or administrator user. See
Scaleway's
[environment variable reference](https://www.scaleway.com/en/docs/terraform/reference-content/environment-variables/).

## Store it

Recommended: the guided form opens Scaleway IAM API keys in your browser, asks
for the access key and secret key (hidden) and the optional project, region and
zone (visible, may be left empty), checks the key with one read-only request
(`GET /iam/v1alpha1/api-keys/<ACCESS_KEY>`) and stores the bundle. A rejected key
saves nothing.

```sh
lifevault import-provider scaleway --connect
lifevault import-provider scaleway --connect --prefix PROD_   # optional prefix
```

Fields already set in your environment are used as-is. `--no-browser` (or
`LIFEVAULT_NO_BROWSER`) prints the link instead of opening it; `--no-verify`
skips the check.

From the environment, without prompts or requests:

```sh
read -rs SCW_ACCESS_KEY && export SCW_ACCESS_KEY
read -rs SCW_SECRET_KEY && export SCW_SECRET_KEY
export SCW_DEFAULT_PROJECT_ID=<PROJECT_ID>    # optional
export SCW_DEFAULT_REGION=<REGION>            # optional, for example fr-par
export SCW_DEFAULT_ZONE=<ZONE>                # optional, for example fr-par-1
lifevault import-provider scaleway
unset SCW_ACCESS_KEY SCW_SECRET_KEY SCW_DEFAULT_PROJECT_ID SCW_DEFAULT_REGION SCW_DEFAULT_ZONE
```

| Variable | Required |
|---|---|
| `SCW_ACCESS_KEY` | yes |
| `SCW_SECRET_KEY` | yes |
| `SCW_PROJECT_ID` | no |
| `SCW_DEFAULT_PROJECT_ID` | no |
| `SCW_DEFAULT_REGION` | no |
| `SCW_DEFAULT_ZONE` | no |

Optional variables are stored only when set. Present optional values must be
nonempty UTF-8.

## Use it

```sh
lifevault run SCW_ACCESS_KEY SCW_SECRET_KEY SCW_DEFAULT_PROJECT_ID \
  SCW_DEFAULT_REGION SCW_DEFAULT_ZONE -- scw instance server list
```

`run` fails if a selected name is not stored, so list only the names you
imported. With `--prefix PROD_`, map each name back, for example
`PROD_SCW_ACCESS_KEY=SCW_ACCESS_KEY`. With `--prefix MY_APP__`, the keys join
project `MY_APP` and `lifevault run --project my-app -- <cmd>` injects all of
them under their original names.

## Updating

- Imports without `--connect` make no Scaleway API calls; `--connect` makes one
  read-only check. No import creates, rotates or refreshes keys.
- After rotating the key at Scaleway, import again with `--connect --replace`
  (or export the new pair and use `--replace`).
- Omitted optional variables leave previously stored values unchanged. Use
  `lifevault remove <NAME>` to drop one.
- Validation or a name collision leaves the vault unchanged.
