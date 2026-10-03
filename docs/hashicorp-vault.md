# HashiCorp Vault

Lifevault works with HashiCorp Vault in both directions. The built-in client
(`--client native`, the default) talks to Vault's HTTP API, so nothing needs to be
installed. `--client cli` uses the official `vault` CLI instead.

- **Push**: mirror each project, one way, into a KV v1 or v2 secret.
- **Import**: copy selected string fields from a KV path into the local vault,
  optionally keeping them in sync.

Neither direction needs HashiCorp Vault for Lifevault's own storage. For dynamic
cloud and database credentials without HashiCorp Vault, see
[broker basics](broker-basics.md).

## Prerequisites

- Nothing to install. Only `--client cli` needs the `vault` CLI on `PATH`; a
  `--client cli` target is refused up front when `vault` is missing.
- A KV secrets engine (default mount `secret`, KV v2).
- Vault reachable over `https://` (plain `http://` only to `localhost`).
- For push: an OIDC login or a token (`target connect hashicorp` stores it for
  you). For import: `VAULT_TOKEN`, or the token `vault login` saved in
  `~/.vault-token`.

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

### Guided setup: `target connect hashicorp`

```sh
# Sign in through your browser (OIDC, the default):
lifevault target connect hashicorp <TARGET> --address https://<vault-host>:8200 \
  --role <ROLE> --all-projects

# Or paste a token:
lifevault target connect hashicorp <TARGET> --address https://<vault-host>:8200 \
  --method token --all-projects
```

What happens:

1. **Credentials.** With `--method oidc` (the default) Lifevault runs Vault's OIDC
   login itself, like `vault login -method=oidc`: it opens the provider's login
   page in your browser and receives the redirect on a one-time listener at
   `http://localhost:8250/oidc/callback`. That redirect URI must be in the role's
   `allowed_redirect_uris`. The wait ends after 5 minutes. The token is not
   written to `~/.vault-token`. With `--method token`, `VAULT_TOKEN` is used when
   set; otherwise `<ADDRESS>/ui/` opens and you paste a token at a hidden prompt.
2. **Check.** A token lookup verifies it and shows its display name, policies and
   expiry. Any failure stops here and nothing is saved.
3. **Save.** The token is stored as `TARGET_<NAME>_VAULT_TOKEN` together with the
   target, in one vault write. Existing secrets with that name are overwritten
   only after confirmation or with `--yes`.
4. **First push.** The key names to push are listed and the projects are pushed.
   A terminal asks first unless `--yes`.

| Option | Meaning |
|---|---|
| `--address URL` | Required. Vault address. |
| `--method oidc\|token` | Browser OIDC login (default) or a pasted token. |
| `--role ROLE` | OIDC role. |
| `--mount M`, `--path-prefix P`, `--namespace NS`, `--ca-cert PATH`, `--kv-version 1\|2` | As for `target add` below. |
| `--client native\|cli` | Built-in client (default) or the `vault` CLI (`vault login -method=oidc` for OIDC). |
| `--project P` (repeatable) or `--all-projects` | Projects to push. |
| `--yes` | Skip confirmations. |
| `--no-browser` | Print links instead of opening them. |

**Token expiry.** When the token expires, pushes fail and stay pending. Run the
same `target connect hashicorp <TARGET> ...` again to sign in and store a new
token: the destination and projects are kept and pending pushes are retried right
away.

### Scripted setup: `target add --type hashicorp`

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
| `--client native\|cli` | `native` | Built-in client or the `vault` CLI. |
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
- **No values in argv or URLs.** The built-in client sends the token only in the
  `X-Vault-Token` header (and `--namespace` as `X-Vault-Namespace`), trusts
  `--ca-cert` in addition to the system roots, and reports only HTTP statuses.
  With `--client cli`, values reach `vault` on stdin and credentials go in its
  environment, which is cleared except for `PATH`, `HOME`, `LANG`, the auth
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
  run `target connect hashicorp` again, or create a new token, store it with
  `lifevault set <SECRET>` (the name you passed to `--auth`), and run
  `lifevault push`.

## Import from Vault

```sh
export VAULT_ADDR=https://<vault-host>:8200
read -rs VAULT_TOKEN && export VAULT_TOKEN    # or rely on ~/.vault-token from vault login
lifevault import-hashicorp secret/<APP> api_key=<APP>_API_KEY db_password=<APP>_DB_PASSWORD
lifevault import-hashicorp secret/<APP> --kv-version 1 --all
lifevault import-hashicorp secret/<APP> api_key=<APP>_API_KEY --auto-refresh 300
lifevault import-hashicorp secret/<APP> --all --client cli   # use vault kv get instead
```

- `PATH` is the normal combined KV path, as for `vault kv get`, not an API
  `/data/` path. The mount comes from Vault itself. `--kv-version` selects the
  response format (default 2).
- The built-in client reads the same settings as the `vault` CLI: `VAULT_ADDR`
  (default `https://127.0.0.1:8200`; plain `http://` only to `localhost`),
  `VAULT_TOKEN` (else `~/.vault-token`), `VAULT_NAMESPACE` and `VAULT_CACERT`.
  `--client cli` runs `vault kv get` with the CLI's own authentication instead;
  tracked imports remember the choice. Lifevault never logs in for imports and
  never disables TLS verification.
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
| `--kv-version 1\|2` | KV response format (default 2). |
| `--client native\|cli` | Built-in client (default) or `vault kv get`. |
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

- **Import fails with a generic error.** Check `VAULT_ADDR`, the token and the
  path (for example with `vault kv get <PATH>`). Lifevault hides source errors.
- **OIDC login never completes.** Add `http://localhost:8250/oidc/callback` to the
  role's `allowed_redirect_uris`, or use `--method token`.
- **Push marks the project pending.** Check the token's policy against the
  `--mount` and `--path-prefix` you configured, and `--namespace` for Enterprise.
- **"--kv-version must be 1 or 2".** Pass `1` or `2`.
