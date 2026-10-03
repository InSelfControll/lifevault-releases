# Bitwarden and Vaultwarden

Lifevault pushes each project, one way, into Bitwarden or a self-hosted
Vaultwarden server. Each project becomes a secure note named `lifevault/PROJECT`
with one hidden field per key.

The built-in client (`--client native`, the default) talks to the server's HTTPS
API and does Bitwarden's encryption itself, so nothing needs to be installed.
`--client cli` uses the official `bw` CLI instead.

## Prerequisites

- Nothing to install. Only `--client cli` needs the `bw` CLI on `PATH`; a
  `--client cli` target is refused up front when `bw` is missing.
- A Bitwarden or Vaultwarden account, its personal API key (`client_id` and
  `client_secret`) and its master password.
- Projects in the vault (for example from [`import-env`](env-files.md)).
- For an organization destination: a collection that only the right people can
  open. `target connect --create-collection NAME` can create one for you (see
  [Who can see the secrets](#who-can-see-the-secrets)).
- An account with a master password. Accounts without one (SSO with trusted
  devices or Key Connector) and accounts on Bitwarden's newer (v2, COSE)
  encryption are not supported by the built-in client yet; use `--client cli`.

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
   from the environment when set. Otherwise the web vault's API key page
   (`<SERVER>/#/settings/security/security-keys`) opens in your browser: click
   **View API key**, then paste `client_id` and `client_secret` at the hidden
   prompts, followed by the master password. `--no-browser` (or
   `LIFEVAULT_NO_BROWSER`) prints the link instead.
2. **Server.** `--server`, else `BW_SERVER` or `BITWARDEN_SERVER`, else Bitwarden
   cloud.
3. **Login check.** The built-in client logs in with the API key over HTTPS,
   unlocks with the master password and syncs; decryption happens in memory on
   your machine. With `--client cli`, `bw` does the same in this target's private
   state directory. Any failure stops here and nothing is saved.
4. **Destination.** `--organization` (ID or case-insensitive name) with
   `--collection`, or `--folder NAME` in the personal vault (created on first
   push). Without them, a terminal menu lists your organizations, the personal
   vault and collections. A single collection is chosen automatically.
   `--create-collection NAME` creates a new private collection instead.
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
| `--collection NAME_OR_ID` | Existing collection in that organization. |
| `--create-collection NAME` | Create a collection only the connecting account can access, and push there. Built-in client only. Not combined with `--collection`. |
| `--folder NAME` | Personal-vault folder instead of an organization. |
| `--client native\|cli` | Built-in client (default) or the official `bw` CLI. |
| `--project P` (repeatable) or `--all-projects` | Projects to push. |
| `--yes` | Skip confirmations (overwrite credential names, first push). |
| `--no-browser` | Print the API key page link instead of opening it. |

**Reconnect.** Run `target connect bitwarden` again for an existing target to
replace its stored credentials (after confirmation or with `--yes`). The
destination and projects are kept unless you pass options, and pending pushes are
retried right away.

Lifevault warns when a named project has no secrets yet and lists the existing
project names. After the first push it prints the outcome, for example
`Pushed 2 projects (14 keys) to <TARGET>.`

Personal-vault folder example, non-interactive:

```sh
lifevault target connect bitwarden <TARGET> --folder <FOLDER> --all-projects --yes
```

## Who can see the secrets

In an organization, an item is visible to everyone with access to its
collection. There are no per-item permissions. Keep pushed secrets out of
collections that every member can open (such as a default collection shared with
the whole organization).

- **Private collection.** `--create-collection <NAME>` creates a collection that
  only the connecting account can access. Add members or groups later under the
  collection's **Access** in the web vault. This needs the built-in client: `bw`
  cannot look up your organization membership, so with `--client cli` create the
  collection in the web vault and pass `--collection`.
- **Moving a target.** When a target's collection changes (reconnect with another
  `--collection` or `--create-collection`), its existing item is moved there and
  removed from every other collection. No copy is left behind in the old one.
- **Trash.** Items in the trash keep their collections, and Lifevault only moves
  items that are not in the trash. A trashed copy stays in its old collection
  until it is permanently deleted, so permanently delete old copies.

Fixing a target that pushed into a collection everyone can open:

```sh
lifevault target connect bitwarden <TARGET> --organization "<ORG_NAME_OR_ID>" \
  --create-collection <COLLECTION_NAME> --project <PROJECT> --yes
```

For an existing target this is a reconnect: the credentials are read from the
environment or asked for again, the private collection is created, and the
existing `lifevault/<PROJECT>` item is moved into it and out of every other
collection. Then permanently delete any older copies from the trash.

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
| `--organization ID --collection ID` | Organization collection. Both are required together. `target add` takes IDs only; passing a name points you to `target connect`. |
| `--client native\|cli` | Built-in client (default) or the official `bw` CLI. |
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
- **No values in argv or URLs.** The built-in client sends credentials only in
  the token request body and values only encrypted; error messages carry an HTTP
  status, never response bodies. With `--client cli`, values reach `bw` on stdin
  and credentials go in its environment, which is cleared except for `PATH`,
  `HOME`, `LANG`, the auth variables and backend settings. CLI error output is
  discarded.

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

- The built-in client uses HTTPS only (plain HTTP just to `localhost`) and
  caches nothing on disk except a random device ID stored with the target, so the
  server sees one stable device. Bitwarden cloud in the US or EU uses its
  regional identity and API hosts; `--server` uses `<SERVER>/identity` and
  `<SERVER>/api`.
- `--client cli` only: each target keeps its own `bw` state in an owner-only
  directory below `~/.config/lifevault/push/`. The `bw` vault is locked after each
  push.
- Lifevault marks its notes with the line `managed-by: lifevault`.
- `push --prune` moves the note to the trash.

## Troubleshooting

- **Login fails in `target connect`.** Nothing is saved. Check the API key, master
  password and `--server` URL, then run the command again.
- **No collection offered.** Pass `--create-collection <NAME>`, or create the
  collection in the web vault and give the account edit access.
- **"--create-collection needs --client native".** Create the collection in the
  web vault and use `--collection`, or drop `--client cli`.
- **Login fails for an SSO or newer-encryption account.** Use `--client cli`.
- **"bitwarden needs --folder NAME or --organization ID --collection ID".** Pass
  exactly one destination to `target add`.
- **Push pending after a password change.** Store the new value with
  `lifevault set TARGET_<NAME>_BW_PASSWORD` (or the secret you referenced with
  `--auth`), then run `lifevault push`.
