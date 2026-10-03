# .env files

Lifevault imports `.env` files as **projects**: each `KEY=value` line is stored as
`PROJECT__KEY`. `lifevault run --project` then starts your app with every key of
that project in its environment, so the plaintext file can be deleted.

A `.env` file can also hold **references** to Bitwarden/Vaultwarden notes instead
of values. `lifevault run --env-file` resolves them at run time, so the file can
be committed (see [Committing .env files with references](#committing-env-files-with-references)).

## Prerequisites

- The `lifevault` binary and a vault (`lifevault init`).
- `.env` files that are regular files owned by you, at most 1 MiB, valid UTF-8.

## Import one file

```sh
cd ~/code/my-app && lifevault import-env            # ./.env as project MY_APP
lifevault import-env ~/code/my-app                   # PATH/.env
lifevault import-env ~/code/my-app/.env.production --project my-app-prod
lifevault import-env ~/code/my-app --replace         # overwrite existing names
```

| Option | Meaning |
|---|---|
| `PATH` | A directory (reads `PATH/.env`) or a file. Default `.`. |
| `--project NAME` | Project name. Default: the file's directory name. |
| `--replace` | Allow overwriting existing names. Without it, existing names are an error. |
| `--auto-refresh SECONDS` | Keep the vault in sync with the file (1 to 604800). |

Project names are uppercased; other characters become one `_`, and leading or
trailing `_` are trimmed (`my-app` becomes `MY_APP`). A name must start with a
letter after this, otherwise pass `--project NAME`. The same normalization applies
to `run --project` and `list --project`.

## Import many files

```sh
lifevault import-env --scan ~/code                   # dry run: prints the plan only
lifevault import-env --scan ~/code --depth 2 --yes   # import all, one vault write
```

| Option | Meaning |
|---|---|
| `DIR` | Directory to scan. |
| `--depth N` | Maximum depth (default 4). |
| `--yes` | Import. Without it, only the plan is printed: path, project, key count, conflicting names. |
| `--replace`, `--auto-refresh SECONDS` | As for a single file. |

- Only files named exactly `.env` match; `.env.example`, `.env.sample` and
  `.env.template` are ignored.
- Scans never follow symlinks and skip `.git`, `node_modules`, `target`,
  `vendor`, `venv`, `.venv`, `dist`, `build`, `__pycache__` and `.cache`.
- Every file is validated first. Any error leaves the vault unchanged. Two
  folders that normalize to the same project abort the scan; import them one by
  one with `--project`.

## Use a project

```sh
lifevault list --project my-app                      # names only, never values
lifevault run --project my-app -- npm start          # app sees DATABASE_URL, ...
lifevault run --project my-app OTHER_TOKEN=API_TOKEN -- ./server
```

`--project` adds every `MY_APP__KEY` as `KEY`. Explicit `NAME=ENV_NAME` mappings
are added on top and win on conflict.

## Committing .env files with references

Push the project to a [Bitwarden/Vaultwarden](bitwarden.md) target (the section
layout puts project `DNS_FABRIC_LICENSE` in the note
`dnsfabric/DNS_FABRIC_LICENSE`), then replace the values with references:

```sh
# .env (safe to commit)
DATABASE_URL=lifevault://dnsfabric/DNS_FABRIC_LICENSE/DATABASE_URL
LICENSE_KEY=lifevault://dnsfabric/DNS_FABRIC_LICENSE/LICENSE_KEY
LOG_LEVEL=info
```

```sh
lifevault run --env-file .env -- ./server
lifevault run --env-file .env --env-file .env.local -- make test
lifevault run --env-file .env --offline -- ./server      # cached values only
```

| Option | Meaning |
|---|---|
| `--env-file PATH` | Load a `.env` file's variables (repeatable; later files win). |
| `--remote NAME` | Account to read references from, when more than one has the collection. |
| `--refresh` | Always fetch references, ignoring fresh cached values. |
| `--offline` | Use cached values only; never contact the server. |

- `lifevault://COLLECTION/NOTE/FIELD` reads a custom field of a note (section
  layout); `lifevault://PARENT/PROJECT/KEY` reads a per-key note. Names match
  ignoring case. A reference that fits both readings is an error.
- Other values are used literally. The files are parsed with the rules in
  [File syntax](#file-syntax), with no interpolation.
- Explicit `NAME[=ENV_NAME]` mappings win over `--env-file` variables, which win
  over `--project`.
- Resolved values are cached encrypted in the vault (an hour by default; never
  listed or pushed). When the server can't be reached, a cached value is used
  with a warning. An unresolvable reference stops `run` before the command
  starts.
- References are read from your Bitwarden push targets. A teammate who only reads
  the collections adds a read-only remote and needs nothing else besides the
  `.env` file:

  ```sh
  lifevault remote connect bitwarden team --server https://<vaultwarden-host>
  lifevault run --env-file .env -- ./server
  ```

See [`.env` references and remotes](bitwarden.md#env-references-and-remotes).

## File syntax

- `KEY=value` lines, optional `export `, blank lines and `#` comments.
- Unquoted values are trimmed; ` #` starts a comment.
- Single quotes are literal. Double quotes support `\n \r \t \" \\ \$`. Both may
  span lines; only a comment may follow the closing quote.
- CRLF is accepted. A BOM is ignored. Empty values are stored as empty strings.
- No interpolation: `$VAR` and `${VAR}` are stored literally.
- Errors name the file and line, never the value.

## Keeping in sync

```sh
lifevault import-env ~/code/my-app --replace --auto-refresh 300
```

- When due, `get`, `run` and `ssh-add` re-read the file before use. Changed
  values are updated and new keys added.
- A new key whose name already exists outside this import is skipped with a
  warning.
- Keys removed from the file are **kept** in the vault, with a warning naming
  them, so a truncated file cannot wipe secrets. Use `lifevault remove` to delete
  them.
- Names you `set`, `import` or `remove` are detached and never resurrected.
- `lifevault refresh` re-reads now; `lifevault refresh --disable` stops tracking.

## Pushing projects elsewhere

Projects can be mirrored to [Bitwarden/Vaultwarden](bitwarden.md),
[1Password](1password.md), [HashiCorp Vault](hashicorp-vault.md) or
[Ansible Vault](ansible-vault.md). Importing or refreshing a `.env` file pushes
the changed projects automatically. Bitwarden-pushed projects can then be loaded
back through `.env` references.

## Security notes

- Lifevault never modifies or deletes the `.env` file. It warns that the
  plaintext copy still exists: delete it after checking the vault copy
  (`lifevault list --project <PROJECT>`).
- A symlinked `.env` is refused; a symlinked parent directory is fine.
- Remember other plaintext copies: backups, editor swap files, and git history.
