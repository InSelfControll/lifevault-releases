# Bitwarden and Vaultwarden

Lifevault pushes each project, one way, into Bitwarden or a self-hosted
Vaultwarden server. In an organization, each project becomes one secure note
named exactly like the project, with one hidden field per key, in a collection
you name (the [section layout](#layouts)). `.env` files can then refer to those
fields and be resolved at run time ([references](#env-references-and-remotes)).

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
  open. `target connect` creates a missing `--parent` collection with access only
  for you (see [Who can see the secrets](#who-can-see-the-secrets)).
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
  --organization "<ORG_NAME_OR_ID>" --parent <COLLECTION> \
  --project <PROJECT> --project <PROJECT2>
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
4. **Destination.** `--organization` (ID or case-insensitive name) uses the
   section layout in the collection named by `--parent` (default: the
   organization's name), created with access only for you when missing.
   `--layout per-key` writes one note per variable instead. `--collection` or
   `--create-collection NAME` (one collection) and `--folder NAME` (personal
   vault, created on first push) use the per-project layout. Without them, a
   terminal menu lists your organizations, the personal vault and collections. A
   single collection is chosen automatically. See [Layouts](#layouts).
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
| `--layout section\|per-key` | Organization layout: one note per project (`section`, the default) or one note per variable (`per-key`). |
| `--parent NAME` | Collection for the project group (default: the organization's name). Created with access only for you when missing (built-in client). |
| `--layout per-project` | One item `lifevault/PROJECT`; used with `--collection`, `--create-collection` or `--folder`. |
| `--collection NAME_OR_ID` | Existing collection in that organization (per-project layout). |
| `--create-collection NAME` | Create a collection only the connecting account can access, and push there (per-project layout). Built-in client only. Not combined with `--collection`. |
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

## Layouts

| Layout | Chosen by | Remote items |
|---|---|---|
| `section` | Default for new organization targets | One secure note per project, named exactly like the project, with a hidden field per variable, in the collection `PARENT` |
| `per-key` | `--layout per-key` | One secure note per variable, named like the variable, holding the value in the hidden field `value`, in the nested collection `PARENT/PROJECT` |
| `per-project` | `--folder`, `--collection` or `--create-collection` | One secure note `lifevault/PROJECT` with a hidden field per variable (the layout of earlier releases) |

`--parent NAME` names one collection per group of projects, for example
`dnsfabric`. With the projects `DNS_FABRIC_LICENSE` and `DNS_FABRIC_API`:

```text
section (--parent dnsfabric)
  dnsfabric/                          collection
    DNS_FABRIC_LICENSE                secure note: hidden fields DATABASE_URL, LICENSE_KEY, ...
    DNS_FABRIC_API                    secure note: hidden fields API_TOKEN, ...

per-key (--layout per-key --parent dnsfabric)
  dnsfabric/DNS_FABRIC_LICENSE/       collection
    DATABASE_URL                      secure note: hidden field "value"
    LICENSE_KEY                       secure note: hidden field "value"
```

- **Collections.** `--parent` defaults to the organization's name and is matched
  ignoring case. A missing parent (and, for per-key, a missing `PARENT/PROJECT`
  collection) is created with access only for the connecting account. This
  needs the built-in client; with `--client cli` create the collections in the
  web vault first, or the push fails with a message saying so. The web vault
  shows `A/B` nested under `A` only when a collection named exactly `A` exists.
- **Notes.** Each note carries the lines `managed-by: lifevault` and
  `Project: PROJECT (managed by lifevault; edits are overwritten)`. A note
  without the marker is never changed: a same-named note from someone else fails
  that project (or variable) instead.
- **Pushes.** A section note's fields are rewritten when the project changes, and
  removed variables disappear from it. Per-key pushes rewrite only notes whose
  value changed and move the notes of removed variables to the trash.
  `push --prune` trashes the notes of projects without secrets.

### Migrating an existing target

Targets saved by earlier releases keep the per-project layout. Switch one with:

```sh
lifevault target connect bitwarden <TARGET> --organization "<ORG_NAME_OR_ID>" \
  --layout section --parent dnsfabric --project <PROJECT> --yes
```

After a project's new notes are written, its items of the old layout (the
`lifevault/PROJECT` item, or the per-variable notes in `PARENT/PROJECT`) are
moved to the trash. Collections are never deleted: an emptied one is pointed out
so you can delete it in the web vault. `--layout per-key` switches the same way.

Vaults with per-key or section targets can't be opened by releases older than
0.5.0; update every machine that uses the vault first.

### Folders (0.5.2)

Bitwarden folders are personal: every user files items, organization items
included, in their own folders. Apps list folders separately from collections.

| Goal | Command |
|---|---|
| Organization notes, also shown under *Folders > dnsfabric* for you | `lifevault target connect bitwarden <TARGET> --organization <ORG> --layout section --parent <COLLECTION> --folder dnsfabric --project <PROJECT> --yes` |
| New target, personal vault only | `lifevault target connect bitwarden <TARGET> --layout section --folder dnsfabric --project <PROJECT>` |
| Move an organization target to the personal vault | `lifevault target connect bitwarden <TARGET> --personal --folder dnsfabric --yes` |
| Move it back to the organization | `lifevault target connect bitwarden <TARGET> --organization <ORG> --layout section --parent <COLLECTION> --folder dnsfabric --yes` |

- The folder is created when missing and matched ignoring case. Filing an
  unchanged note only updates your folder assignment; its content is not rewritten.
- Moving between the organization and the personal vault writes the new notes
  first, then moves the old ones to the trash.
- Reconnecting an organization target with `--folder` alone keeps it in the
  organization; leaving needs `--personal`.
- `--folder` without an organization and without `--layout section` keeps the
  older per-project layout (one `lifevault/PROJECT` item in the folder).
- References resolve through folders as well as collections:
  `lifevault://dnsfabric/DNS_FABRIC_LICENSE/DATABASE_URL`.

## Who can see the secrets

In an organization, an item is visible to everyone with access to its
collection. There are no per-item permissions, so the collection is the access
boundary: with the section layout, everyone who can open `dnsfabric` can read
every project note in it. Keep pushed secrets out of
collections that every member can open (such as a default collection shared with
the whole organization).

- **Private collection.** A missing `--parent` collection, and
  `--create-collection <NAME>`, are created with access only for the connecting
  account. Add members or groups later under the
  collection's **Access** in the web vault. This needs the built-in client: `bw`
  cannot look up your organization membership, so with `--client cli` create the
  collection in the web vault and pass `--collection`.
- **Moving a target.** When a target's collection changes (reconnect with another
  `--collection` or `--create-collection`), its existing item is moved there and
  removed from every other collection. No copy is left behind in the old one.
- **Trash.** Items in the trash keep their collections, and Lifevault only moves
  items that are not in the trash. A trashed copy, including the old items a
  layout switch trashed, stays visible to that collection's members until it is
  permanently deleted, so permanently delete old copies.

Fixing a target that pushed into a collection everyone can open:

```sh
lifevault target connect bitwarden <TARGET> --organization "<ORG_NAME_OR_ID>" \
  --create-collection <COLLECTION_NAME> --project <PROJECT> --yes
```

For an existing target this is a reconnect: the credentials are read from the
environment or asked for again, the private collection is created, and the
existing `lifevault/<PROJECT>` item is moved into it and out of every other
collection. Then permanently delete any older copies from the trash.

## `.env` references and remotes

A `.env` file can refer to notes instead of holding values, so it can be
committed:

```sh
# .env
DATABASE_URL=lifevault://dnsfabric/DNS_FABRIC_LICENSE/DATABASE_URL   # COLLECTION/NOTE/FIELD
LICENSE_KEY=lifevault://dnsfabric/DNS_FABRIC_LICENSE/LICENSE_KEY     # another field
LOG_LEVEL=info                                                       # plain value

lifevault run --env-file .env -- ./server
```

- `lifevault://COLLECTION/NOTE/FIELD` reads the custom field `FIELD` of the note
  `NOTE` (section layout). `lifevault://PARENT/PROJECT/KEY` reads the note `KEY`
  in the collection `PARENT/PROJECT` (per-key layout): its field `value`, else a
  field named like the variable, else a login item's password. Two segments
  always mean `COLLECTION/NOTE`.
- Names match ignoring case. Any item the account can read works, not only items
  Lifevault pushed.
- A reference that fits both readings is an error asking you to rename or move
  one; nothing is guessed. An unresolvable reference stops `run` before the
  command starts, naming the reference but never a value.

**Remotes** are the accounts references are read from: every Bitwarden push
target, plus read-only remotes for people who only read the collections:

```sh
lifevault remote connect bitwarden team --server https://<vaultwarden-host>
lifevault remote list
lifevault remote remove team --yes          # also deletes its credentials and cache
lifevault remote clear-cache [team]         # drop cached values
```

| Option | Meaning |
|---|---|
| `--server URL` | Server, else `BW_SERVER` or `BITWARDEN_SERVER`, else Bitwarden cloud. |
| `--client native` | Built-in client (the only one for remotes). |
| `--cache-ttl SECONDS` | How long resolved values stay fresh (default 3600; `0` always fetches). |
| `--yes` | Skip confirmations. |
| `--no-browser` | Print the API key page link instead of opening it. |

`remote connect` asks for and verifies the same credentials as
`target connect bitwarden` and stores them as `REMOTE_<NAME>_BW_CLIENTID`,
`REMOTE_<NAME>_BW_CLIENTSECRET` and `REMOTE_<NAME>_BW_PASSWORD`. Targets and
remotes with the same server and API key count as one account. With several
accounts, the one whose organizations have the collection is used; `run --remote
NAME` picks one explicitly.

Resolved values are cached encrypted in the vault (never listed or pushed).
Fresh values need no network; stale ones are fetched with one login and sync per
account. If that fails, a cached value is used with a warning. `run --offline`
uses only the cache and `run --refresh` always fetches. Vaults with remotes or
cached references can't be opened by releases older than 0.5.0. See
[.env files](env-files.md#committing-env-files-with-references).

## Scripted setup: `target add --type bitwarden`

Store the credentials yourself, then reference them by name. Organization and
collection must be IDs here (`target connect` finds them for you). With
`--organization ID` alone the target uses the section layout; add `--parent NAME`
to choose the collection.

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
| `--organization ID [--layout section\|per-key] [--parent NAME]` | Organization, section layout by default. `target add` takes IDs only; passing a name points you to `target connect`. |
| `--organization ID --collection ID` | One organization collection (per-project layout). |
| `--client native\|cli` | Built-in client (default) or the official `bw` CLI. |
| `--auth BW_CLIENTID=<SECRET>` | Required. |
| `--auth BW_CLIENTSECRET=<SECRET>` | Required. |
| `--auth BW_PASSWORD=<SECRET>` | Required. |

`target add` pushes its projects right away.

## How pushes behave

- **One way.** Lifevault is the source of truth. Each project's `PROJECT__KEY`
  secrets become one remote note (per-key: one note per key); edits made in the remote manager are
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
  items until `lifevault push --prune`. `target remove` never deletes remote data.
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
- `push --prune` and layout switches move notes to the trash; collections are
  never deleted.

## Troubleshooting

- **Login fails in `target connect`.** Nothing is saved. Check the API key, master
  password and `--server` URL, then run the command again.
- **No collection offered.** Pass `--create-collection <NAME>`, or create the
  collection in the web vault and give the account edit access.
- **"--create-collection needs --client native".** Create the collection in the
  web vault and use `--collection`, or drop `--client cli`.
- **Login fails for an SSO or newer-encryption account.** Use `--client cli`.
- **"bitwarden needs --organization ID (section layout), --folder NAME or
  --organization ID --collection ID".** Pass exactly one destination to
  `target add`.
- **Push fails for a missing collection with `--client cli`.** `bw` can't create
  collections; create the `--parent` (and per-key `PARENT/PROJECT`) collection in
  the web vault, or use the built-in client.
- **A reference is ambiguous.** It matches both a section field and a per-key
  note, or several accounts; rename or move one, or pass `run --remote NAME`.
- **Push pending after a password change.** Store the new value with
  `lifevault set TARGET_<NAME>_BW_PASSWORD` (or the secret you referenced with
  `--auth`), then run `lifevault push`.
