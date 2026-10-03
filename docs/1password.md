# 1Password

Lifevault pushes each project, one way, into a 1Password vault through the
official `op` CLI. Each project becomes an item named `lifevault/PROJECT` with one
concealed field per key.

## Prerequisites

- The `op` CLI on `PATH`.
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

## Set up the target

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
- **Token rotated.** Store the new token with `lifevault set OP_PUSH_TOKEN`, then
  run `lifevault push`.
