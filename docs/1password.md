# 1Password

Lifevault pushes each project, one way, into a 1Password vault through the
official `op` CLI. Each project becomes an item named `lifevault/PROJECT` with one
concealed field per key.

## Prerequisites

- The `op` CLI on `PATH`. Unlike the other push targets, 1Password has no
  built-in client; `op` is always required.
- A 1Password account that can create service accounts.
- A shared vault for the pushed items. Service accounts cannot reach Personal or
  Private vaults.

## Least privilege

Create a service account limited to one vault, with read and write access to
items only:

```sh
op service-account create lifevault-push --vault <VAULT>:read_items,write_items
```

Store the token it prints at a hidden prompt:

```sh
lifevault set OP_PUSH_TOKEN
```

## Guided setup: `target connect onepassword`

```sh
lifevault target connect onepassword <TARGET> --vault <VAULT> --all-projects
```

What happens:

1. **Token.** `OP_SERVICE_ACCOUNT_TOKEN` is used when set. Otherwise the
   1Password.com service account page (**Developer > Service accounts**) opens in
   your browser and you paste the token at a hidden prompt. `--no-browser` (or
   `LIFEVAULT_NO_BROWSER`) prints the link instead.
2. **Check.** `op vault list` verifies the token. Any failure stops here and
   nothing is saved.
3. **Destination.** `--vault` matches an ID or case-insensitive name. Without it,
   a single vault is chosen automatically; otherwise a terminal menu asks.
4. **Save.** The token is stored as `TARGET_<NAME>_OP_SERVICE_ACCOUNT_TOKEN`
   together with the target, in one vault write. Existing secrets with that name
   are overwritten only after confirmation or with `--yes`.
5. **First push.** The key names to push are listed and the projects are pushed.
   A terminal asks first unless `--yes`.

| Option | Meaning |
|---|---|
| `--vault NAME_OR_ID` | Destination vault. |
| `--project P` (repeatable) or `--all-projects` | Projects to push. |
| `--yes` | Skip confirmations. |
| `--no-browser` | Print the link instead of opening it. |

Run the same command again to replace the stored token (for example after
rotating it). The vault and projects are kept unless you pass options, and
pending pushes are retried right away.

## Scripted setup: `target add --type onepassword`

```sh
lifevault target add <TARGET> --type onepassword --vault <VAULT> --all-projects \
  --auth OP_SERVICE_ACCOUNT_TOKEN=OP_PUSH_TOKEN
```

| Option | Meaning |
|---|---|
| `--vault NAME` | Required. Destination vault. |
| `--project P` (repeatable) or `--all-projects` | Projects to push. |
| `--auth OP_SERVICE_ACCOUNT_TOKEN=<SECRET>` | Required. Stored secret holding the service-account token. |

`target add` pushes its projects right away.

## How pushes behave

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

## Day-to-day commands

```sh
lifevault target list                    # targets, projects, last push, pending errors
lifevault push --dry-run                 # key names that would be added, changed or removed
lifevault push                           # push out-of-date projects, retry pending ones
lifevault push --target <TARGET> --force # ignore the change cache
lifevault push --target <TARGET> --project <PROJECT>
lifevault push --prune                   # delete remote items of projects with no secrets
lifevault target remove <TARGET>         # stop pushing; remote data is kept
```

## 1Password specifics

- Lifevault tags its items `lifevault-managed`.
- `push --prune` archives the item.

## Troubleshooting

- **Push fails with an access error.** Check that the service account has
  `read_items,write_items` on the vault named by `--vault`, and that the vault is
  not a Personal/Private vault.
- **Token rotated.** Run `target connect onepassword <TARGET>` again, or store the
  new token with `lifevault set <SECRET>` (the name you passed to `--auth`), then
  run `lifevault push`.
