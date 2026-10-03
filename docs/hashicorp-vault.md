# HashiCorp Vault

Lifevault works with HashiCorp Vault in both directions, through the official
`vault` CLI:

- **Push**: mirror each project, one way, into a KV v1 or v2 secret.
- **Import**: copy selected string fields from a KV path into the local vault,
  optionally keeping them in sync.

Neither direction needs HashiCorp Vault for Lifevault's own storage. For dynamic
cloud and database credentials without HashiCorp Vault, see
[broker basics](broker-basics.md).

## Prerequisites

- The `vault` CLI on `PATH`.
- A KV secrets engine (default mount `secret`, KV v2).
- For push: a token stored in Lifevault. For import: the `vault` CLI already
  authenticated in your shell (`vault login` or a token helper).

## Push projects to Vault

### Least-privilege token

Grant write access below the push prefix only. For KV v2 at `secret/` with the
default prefix `lifevault`:

```hcl
path "secret/data/lifevault/*"     { capabilities = ["create", "update", "read"] }
path "secret/metadata/lifevault/*" { capabilities = ["read", "update", "delete"] }
```

The metadata rule lets Lifevault mark its items (`managed_by=lifevault`) and
allows `--prune`. For KV v1, use `path "secret/lifevault/*"` with `create`,
`update` and `delete`.

```sh
vault policy write lifevault-push policy.hcl
vault token create -policy=lifevault-push -period=768h
lifevault set VAULT_PUSH_TOKEN      # paste the token at the hidden prompt
```

### Set up the target

```sh
lifevault target add <TARGET> --type hashicorp --all-projects \
  --address https://<vault-host>:8200 \
  --mount secret --path-prefix lifevault \
  --auth VAULT_TOKEN=VAULT_PUSH_TOKEN
```

| Option | Default | Meaning |
|---|---|---|
| `--address URL` | required | Vault address (HTTP(S), no credentials, query or fragment). |
| `--mount NAME` | `secret` | KV mount. |
| `--path-prefix P` | `lifevault` | Path below the mount. |
| `--namespace NS` | none | Vault Enterprise namespace. |
| `--ca-cert PATH` | none | PEM CA for a private CA. |
| `--kv-version 1\|2` | `2` | KV engine version. |
| `--auth VAULT_TOKEN=<SECRET>` | required | Stored secret holding the token. |

Each project is written to `<MOUNT>/<PREFIX>/<PROJECT>`, for example
`secret/lifevault/MY_APP`, with one key per `MY_APP__KEY` secret.

### How pushes behave

- **One way.** Lifevault is the source of truth. Each project's `PROJECT__KEY`
  secrets become one remote item; edits made in the remote manager are
  overwritten on the next push, and keys removed locally disappear remotely.
- **Automatic.** After a command changes secrets (`set`, `remove`, `import`,
  `import-*`, `import-env` including `--scan --yes`, `refresh`), affected projects
  are pushed once the vault write is committed. When `get`, `run` or `ssh-add`
  refresh a tracked source, changed projects are pushed too.
- **Retried.** A failed push prints a warning naming the target, project and
  failing subcommand, and the project stays pending. Later commands and
  `lifevault push` retry it.
- **Only its own items.** A same-named item that Lifevault did not create makes
  that project fail instead of being taken over.
- **Nothing deleted by default.** A project with no secrets left keeps its remote
  item until `lifevault push --prune`. `target remove` never deletes remote data.
- **Credentials by reference.** `--auth VAR=SECRET_NAME` maps the CLI's variable
  to a stored secret. Auth secrets are never pushed.
- **No values in argv.** Values reach the CLI on stdin; credentials go in the
  child environment, which is cleared except for `PATH`, `HOME`, `LANG`, the auth
  variables and backend settings. CLI error output is discarded.

### Day-to-day commands

```sh
lifevault target list                    # targets, projects, last push, pending errors
lifevault push --dry-run                 # key names that would be added, changed or removed
lifevault push                           # push out-of-date projects, retry pending ones
lifevault push --target <TARGET> --force # ignore the change cache
lifevault push --target <TARGET> --project <PROJECT>
lifevault push --prune                   # delete remote items of projects with no secrets
lifevault target remove <TARGET>         # stop pushing; remote data is kept
```

### Vault specifics

- KV v2 items are marked with custom metadata `managed_by=lifevault`.
- `push --prune` deletes all versions and metadata of the item.
- A periodic token (`-period`) must be renewed before it expires. If it expires,
  create a new one, store it with `lifevault set VAULT_PUSH_TOKEN`, and run
  `lifevault push`.

## Import from Vault

```sh
export VAULT_ADDR=https://<vault-host>:8200
vault login                                   # or your usual auth method
lifevault import-hashicorp secret/<APP> api_key=<APP>_API_KEY db_password=<APP>_DB_PASSWORD
lifevault import-hashicorp secret/<APP> --kv-version 1 --all
lifevault import-hashicorp secret/<APP> api_key=<APP>_API_KEY --auto-refresh 300
```

- `PATH` is the normal combined KV path passed to `vault kv get`, not an API
  `/data/` path. `--kv-version` selects the response format (default 2).
- Lifevault uses the CLI's existing authentication, `VAULT_ADDR`, namespace and TLS
  settings. It never logs in and never disables TLS verification.
- Tracked imports pin `VAULT_ADDR` and `VAULT_NAMESPACE`; a later shell's
  `VAULT_AGENT_ADDR` cannot redirect them. Tokens are never stored by tracking.
- Responses with duplicate keys or non-string values are rejected. Each source
  call has a 60-second limit.

Import options:

| Option | Meaning |
|---|---|
| `FIELDS...` | Fields to import. `SOURCE=NAME` stores field `SOURCE` as `NAME`. |
| `--all` | Import every field. Use either `FIELDS...` or `--all`, not both. |
| `--replace` | Allow overwriting existing names. |
| `--auto-refresh SECONDS` | Track the source and re-read it before use when due (1 to 604800). |

- Source values must be flat strings. Nested maps and non-string values are
  rejected; quote numeric YAML values.
- All selected fields are saved together, or the vault is unchanged.
- Source output is held in memory (at most 16 MiB) and never printed. Source
  errors are reported generically because they can contain secrets.
- Destination names may contain letters, digits, `_` and `-`, and must start with
  a letter or `_`.

### Keeping imports in sync

With `--auto-refresh SECONDS`, `get`, `run` and `ssh-add` re-read a due source
before using one of its secrets. Nothing polls while Lifevault is idle.

```sh
lifevault refresh             # refresh all tracked sources now; fails on any error
lifevault refresh --disable   # stop tracking; keep current values
```

- If a refresh during `get`/`run` fails, a warning is printed and the stored value
  is used.
- `--all` freezes the original field set; new remote fields are not imported.
- `set`, `import`, `remove` and replacement imports detach a name from its source,
  so refresh never overwrites your manual edits.
- Imports made before tracking existed must be re-imported once with
  `--replace --auto-refresh SECONDS`. Re-import to change an interval.
- Tracking switches the vault to a newer file format that older binaries cannot
  open. Back up the vault first.

## Troubleshooting

- **Import fails with a generic error.** Run `vault kv get <PATH>` yourself to
  check authentication, address and path. Lifevault hides source errors.
- **Push marks the project pending.** Check the token's policy against the
  `--mount` and `--path-prefix` you configured, and `--namespace` for Enterprise.
- **"--kv-version must be 1 or 2".** Pass `1` or `2`.
