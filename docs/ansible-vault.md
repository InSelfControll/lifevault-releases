# Ansible Vault

Lifevault works with Ansible Vault in both directions, through `ansible-vault`:

- **Push**: write each project as a whole-file encrypted YAML map,
  `DIR/<project>.yml`, that playbooks load with `vars_files` or `group_vars`.
- **Import**: copy selected string fields from an encrypted YAML file into the
  local vault, optionally keeping them in sync.

## Prerequisites

- `ansible-vault` on `PATH`.
- For push: an existing directory for the encrypted files, and the vault password
  stored in Lifevault.
- For import: an existing password file or credential helper. Interactive
  `@prompt` is not supported.

## Push projects to Ansible Vault

Store the Ansible Vault password (Ansible strips surrounding whitespace):

```sh
lifevault set ANSIBLE_PUSH_PASSWORD
```

```sh
lifevault target add <TARGET> --type ansible --project <PROJECT> \
  --dir <path/to/group_vars/all> \
  --auth ANSIBLE_VAULT_PASSWORD=ANSIBLE_PUSH_PASSWORD
```

| Option | Meaning |
|---|---|
| `--dir PATH` | Required. Existing directory for the encrypted files. |
| `--project P` (repeatable) or `--all-projects` | Projects to push. |
| `--auth ANSIBLE_VAULT_PASSWORD=<SECRET>` | Required. Stored secret holding the vault password. |

Project `MY_APP` is written to `<DIR>/my_app.yml` as a whole-file encrypted YAML
map with one key per `MY_APP__KEY` secret. Files are mode `0600` and replaced
atomically. Use them in a playbook:

```sh
ansible-playbook site.yml --vault-password-file <path/to/password-source>
```

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

### Ansible specifics

- Lifevault treats a file as its own when it decrypts with the configured
  password. A same-named file that does not decrypt makes that project fail.
- `push --prune` removes the file.

## Import from Ansible Vault

```sh
lifevault import-ansible group_vars/prod/vault.yml \
  --vault-password-file <path/to/password-source> \
  vault_api_key=<APP>_API_KEY
lifevault import-ansible group_vars/prod/vault.yml \
  --vault-id prod@<path/to/password-source> --all
lifevault import-ansible ./secrets.yml api_key=<APP>_API_KEY \
  --vault-password-file <path/to/password-source> --auto-refresh 300
```

| Option | Meaning |
|---|---|
| `--vault-password-file PATH` | Password file or executable helper. A path, never a password. |
| `--vault-id ID@SOURCE` | Named vault ID with a file or helper. `@prompt` is unsupported. |

One of the two is required. The file must be whole-file encrypted; inline
`!vault` values and nested maps are not imported. No decrypted temporary file is
created. Tracked imports store absolute file and helper paths, never password
file contents.

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

- **"Ansible imports require --vault-password-file or --vault-id".** Add one;
  interactive prompting is not supported.
- **Import fails with a generic error.** Check with
  `ansible-vault view <FILE> --vault-password-file <path>`. Lifevault hides
  source errors.
- **Field rejected.** Values must be strings; quote numbers in YAML.
