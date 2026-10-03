# Bitwarden and Vaultwarden

Lifevault pushes each project, one way, into Bitwarden or a self-hosted
Vaultwarden server through the official `bw` CLI. Each project becomes a secure
note named `lifevault/PROJECT` with one hidden field per key.

## Prerequisites

- The `bw` CLI on `PATH`.
- A Bitwarden or Vaultwarden account, its personal API key (`client_id` and
  `client_secret`) and its master password.
- Projects in the vault (for example from [`import-env`](env-files.md)).
- For an organization destination: an existing collection. Lifevault never
  creates collections.

## Least privilege

Personal API keys cannot be scoped. Prefer one of:

- A dedicated account that only holds the target folder.
- An organization member with edit access only to the target collection.

Find the API key in the web vault: **Settings > Security > Keys > View API key**.
`BW_PASSWORD` is the master password needed to unlock.

## Guided setup: `target connect bitwarden`

```sh
lifevault target connect bitwarden <TARGET> --server https://<vaultwarden-host> \
  --organization "<ORG_NAME_OR_ID>" --project <PROJECT> --project <PROJECT2>
```

What happens:

1. **Credentials.** `BW_CLIENTID`, `BW_CLIENTSECRET` and `BW_PASSWORD` are read
   from the environment when set; otherwise you are asked at hidden prompts.
2. **Server.** `--server`, else `BW_SERVER` or `BITWARDEN_SERVER`, else Bitwarden
   cloud.
3. **Login check.** `bw` logs in, unlocks and syncs in this target's private
   state directory. Any failure stops here and nothing is saved.
4. **Destination.** `--organization` (ID or case-insensitive name) with
   `--collection`, or `--folder NAME` in the personal vault (created on first
   push). Without them, a terminal menu lists your organizations, the personal
   vault and collections. A single collection is chosen automatically.
5. **Save.** The credentials are stored as `TARGET_<NAME>_BW_CLIENTID`,
   `TARGET_<NAME>_BW_CLIENTSECRET` and `TARGET_<NAME>_BW_PASSWORD` together with
   the target, in one vault write. `<NAME>` is the target name uppercased, with
   other characters collapsed to `_` (target `vw` gives `TARGET_VW_...`). Existing
   secrets with those names are overwritten only after confirmation or with `--yes`.
6. **First push.** The key names to push are listed and the projects are pushed.
   A terminal asks first unless `--yes`. A declined push stays pending.

| Option | Meaning |
|---|---|
| `--server URL` | Vaultwarden or other self-hosted server. |
| `--organization NAME_OR_ID` | Organization destination. |
| `--collection NAME_OR_ID` | Collection in that organization. |
| `--folder NAME` | Personal-vault folder instead of an organization. |
| `--project P` (repeatable) or `--all-projects` | Projects to push. |
| `--yes` | Skip confirmations (overwrite credential names, first push). |

Personal-vault folder example, non-interactive:

```sh
lifevault target connect bitwarden <TARGET> --folder <FOLDER> --all-projects --yes
```

## Scripted setup: `target add --type bitwarden`

Store the credentials yourself, then reference them by name. Organization and
collection must be IDs here (`target connect` finds them for you).

```sh
lifevault set BW_CLIENT_ID
lifevault set BW_CLIENT_SECRET
lifevault set BW_MASTER
lifevault target add <TARGET> --type bitwarden --project <PROJECT> \
  --server https://<vaultwarden-host> \
  --organization <ORG_ID> --collection <COLLECTION_ID> \
  --auth BW_CLIENTID=BW_CLIENT_ID --auth BW_CLIENTSECRET=BW_CLIENT_SECRET \
  --auth BW_PASSWORD=BW_MASTER
```

| Option | Meaning |
|---|---|
| `--server URL` | Optional. HTTP(S) URL without credentials, query or fragment. |
| `--folder NAME` | Personal-vault folder. Use either this or the next two. |
| `--organization ID --collection ID` | Organization collection. Both are required together. |
| `--auth BW_CLIENTID=<SECRET>` | Required. |
| `--auth BW_CLIENTSECRET=<SECRET>` | Required. |
| `--auth BW_PASSWORD=<SECRET>` | Required. |

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

## Bitwarden specifics

- Each target keeps its own `bw` state in an owner-only directory below
  `~/.config/lifevault/push/`. The vault is locked after each push.
- Lifevault marks its notes with the line `managed-by: lifevault`.
- `push --prune` moves the note to the trash.

## Troubleshooting

- **Login fails in `target connect`.** Nothing is saved. Check the API key, master
  password and `--server` URL, then run the command again.
- **No collection offered.** Create the collection in the web vault and give the
  account edit access; Lifevault does not create collections.
- **"bitwarden needs --folder NAME or --organization ID --collection ID".** Pass
  exactly one destination to `target add`.
- **Push pending after a password change.** Store the new value with
  `lifevault set TARGET_<NAME>_BW_PASSWORD` (or the secret you referenced with
  `--auth`), then run `lifevault push`.
