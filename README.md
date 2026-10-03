# Lifevault

Lifevault is a small, local, encrypted secret vault for Linux and macOS. It keeps
your API keys, tokens, `.env` values and SSH keys in one encrypted file. It hands
them to programs only when they run, so you don't need plaintext `.env` files or
secrets in your shell history.

- **Local first.** No server, database or account. Every command works offline.
- **Project-aware.** Import each project's `.env` file, then run the project with
  its secrets injected: `lifevault run --project my-api -- npm start`.
- **Pushes to the managers you already use.** Lifevault can keep each project in
  sync, one way, with Bitwarden/Vaultwarden, 1Password, HashiCorp Vault or Ansible Vault.
- **Imports from them too.** You can import from Ansible Vault, HashiCorp Vault, environment
  variables, SSH key files and hosting-provider credential bundles, with optional
  automatic refresh.
- **Convenient unlocking.** Unlock once for up to 12 hours, enrol the password in the
  OS keyring, or type it each time.
- **Signed self-updates.** Every release is signed with Ed25519, and the updater
  refuses anything that isn't.

This repository holds the **signed release binaries**. The source code is
private.

---

## Contents

- [Install](#install)
- [Quick start](#quick-start)
- [Command reference](#command-reference)
  - [Vault and unlocking](#vault-and-unlocking)
  - [Storing and using secrets](#storing-and-using-secrets)
  - [Importing .env files by project](#importing-env-files-by-project)
  - [Pushing projects to other secret managers](#pushing-projects-to-other-secret-managers)
  - [Importing from other sources](#importing-from-other-sources)
  - [SSH keys](#ssh-keys)
  - [Automatic refresh](#automatic-refresh)
  - [Updates](#updates)
  - [Credential broker](#credential-broker)
  - [Global options](#global-options)
- [Security model](#security-model)
- [Verifying a download by hand](#verifying-a-download-by-hand)

---

## Install

Download the archive for your platform from the
[latest release](https://github.com/InSelfControll/lifevault-releases/releases/latest):

| Platform              | Archive                                       |
|-----------------------|-----------------------------------------------|
| Linux x86_64          | `lifevault-x86_64-unknown-linux-musl.tar.gz`  |
| Linux ARM64           | `lifevault-aarch64-unknown-linux-musl.tar.gz` |
| macOS Apple silicon   | `lifevault-aarch64-apple-darwin.tar.gz`       |
| macOS Intel           | `lifevault-x86_64-apple-darwin.tar.gz`        |

The Linux builds are static, so there are no runtime dependencies.

```sh
TARGET=x86_64-unknown-linux-musl     # pick yours from the table
VERSION=$(curl -fsSL https://api.github.com/repos/InSelfControll/lifevault-releases/releases/latest \
          | sed -n 's/.*"tag_name": *"\([^"]*\)".*/\1/p')
BASE=https://github.com/InSelfControll/lifevault-releases/releases/download/$VERSION

curl -fLO "$BASE/lifevault-$TARGET.tar.gz"
curl -fLO "$BASE/lifevault-$TARGET.tar.gz.sha256"
curl -fLO "$BASE/lifevault-$TARGET.tar.gz.sig"
sha256sum -c "lifevault-$TARGET.tar.gz.sha256"      # macOS: shasum -a 256 -c
# Recommended: verify the signature (see "Verifying a download by hand")

tar xzf "lifevault-$TARGET.tar.gz"
install -m 755 "lifevault-$TARGET/lifevault" ~/.local/bin/lifevault
lifevault --version
```

Each archive also contains the full documentation, the optional Python provider
adapter used by the broker, and an AI-assistant skill (`skills/lifevault/SKILL.md`).

Once Lifevault is installed, `lifevault update` keeps it current. No GitHub
account or token is needed.

---

## Quick start

```sh
lifevault init                         # create the vault and choose a master password
lifevault unlock                       # optional: stay unlocked for up to 12 hours

lifevault set OPENAI_API_KEY           # type or paste the value at a hidden prompt
lifevault run OPENAI_API_KEY -- python app.py

lifevault import-env ~/code/my-api     # store my-api/.env as MY_API__*
lifevault run --project my-api -- npm start

lifevault lock                         # forget the unlocked session now
```

---

## Command reference

Run `lifevault --help` for the full built-in help, or `lifevault <group> --help`
(for `update`, `broker`, `target` and `push`).

### Vault and unlocking

| Command | What it does |
|---|---|
| `lifevault init [--keyring]` | Create a vault. `--keyring` also saves the password in the OS keyring. |
| `lifevault unlock [--timeout SECONDS]` | Check your password and keep it in RAM for this vault. The default and maximum is 12 hours (43200 s). |
| `lifevault lock` | Revoke the in-memory session immediately. |
| `lifevault keyring enable` | Check the password and save it in the OS keyring (Secret Service on Linux, Keychain on macOS). |
| `lifevault keyring disable` | Check the password and remove its keyring entry. |

Lifevault gets the password from these places, in order:

1. The `LIFEVAULT_PASSWORD` environment variable, for automation.
2. An unlocked session.
3. The OS keyring.
4. A hidden prompt.

The session socket lives in a private per-user runtime directory and only
answers the same Lifevault binary.

### Storing and using secrets

| Command | What it does |
|---|---|
| `lifevault set NAME [--multiline]` | Enter a value at a hidden prompt (create or replace). `--multiline` reads PEM or JSON lines until you press Ctrl-D on an empty line. Piped stdin is stored exactly as given. |
| `lifevault get NAME` | Print one value to stdout with no trailing newline. |
| `lifevault list [--project NAME]` | List stored names (never values), optionally for one project. |
| `lifevault remove NAME...` | Delete secrets. |
| `lifevault import NAME...` | Copy current environment variables into the vault. |
| `lifevault run [--project NAME] NAME[=ENV_NAME]... -- CMD ARGS` | Run a command with selected secrets in its environment. `NAME=ENV_NAME` renames a secret for the program. `--project` adds every `PROJECT__KEY` as `KEY`. |

`run` replaces itself with your command, so the command's exit code and signals pass
straight through. If the command can't be started, `run` exits with 127 (not
found) or 126 (not executable). `LIFEVAULT_PASSWORD` is never passed on to the
command.

Names may use letters, digits, `_` and `-`. They must start with a letter or `_`,
and names used as environment variables can't contain `-`.

### Importing .env files by project

| Command | What it does |
|---|---|
| `lifevault import-env [PATH] [--project NAME] [--replace] [--auto-refresh SECONDS]` | Import a `.env` file. PATH is a file or a directory containing `.env`. Each key is stored as `PROJECT__KEY`. |
| `lifevault import-env --scan [DIR] [--depth N] [--yes] [--replace] [--auto-refresh SECONDS]` | Find every `.env` under DIR (default depth 4) and import each one as its own project. |

```sh
lifevault import-env ~/code/my-api                    # project MY_API
lifevault import-env ./app --project billing          # choose the project name
lifevault import-env --scan ~/code                    # preview only; writes nothing
lifevault import-env --scan ~/code --yes              # import all, in one all-or-nothing write
lifevault import-env ~/code/web --auto-refresh 3600   # keep in sync with the file
lifevault run --project my-api -- npm start           # app sees DATABASE_URL, API_KEY, ...
```

- **Project names** default to the folder name, uppercased, with other characters
  turned into `_` (for example `my-api` becomes `MY_API`).
- **Parsing.** Supported: `KEY=value`, `export KEY=...`, comments (`#`), single-
  and double-quoted values (double quotes support `\n \t \" \\ \$` and can span lines),
  and CRLF line endings. `$VAR` and `${VAR}` are stored literally, not expanded.
  Errors give the file and line number but never the value.
- **Scanning** skips `.git`, `node_modules`, `target`, `vendor`, `venv`/`.venv`,
  `dist`, `build`, `__pycache__` and `.cache`. It never follows symlinks. Only files
  named exactly `.env` are picked up, so `.env.example` and similar are ignored.
  Two folders that map to the same project name cause an error instead of being merged.
- **Existing names are never overwritten** without `--replace`. The error lists
  them.
- **Sync.** With `--auto-refresh`, changed values and new keys are picked up before
  the next `get`/`run`. Keys removed from the file are **kept** in the vault, with a
  warning, so a truncated file can't wipe secrets.
- **Your `.env` file is never modified or deleted.** Lifevault reminds you that the
  plaintext copy still exists; delete it once you've checked the vault copy.

### Pushing projects to other secret managers

Lifevault can mirror each project, one way, into Bitwarden/Vaultwarden, 1Password,
HashiCorp Vault or Ansible Vault. Lifevault is the **source of truth**. After any
command that changes a project's secrets, the affected targets are updated
automatically.

| Command | What it does |
|---|---|
| `lifevault target add NAME --type TYPE (--project P... \| --all-projects) [OPTIONS] --auth VAR=SECRET...` | Add a target and push to it immediately. |
| `lifevault target list` | Show targets, their projects, the last push time, and any pending pushes or errors. |
| `lifevault target remove NAME` | Stop pushing to a target. Remote data is kept. |
| `lifevault push [--target NAME] [--project P] [--dry-run] [--force] [--prune]` | Push now. `--dry-run` lists the key names that would change. `--force` pushes even when nothing changed. `--prune` deletes remote items for projects with no secrets left. |

| Type | Options | Required `--auth` variables | Remote layout |
|---|---|---|---|
| `bitwarden` (Bitwarden and Vaultwarden) | `[--server URL]` and either `--folder NAME` or `--organization ID --collection ID` | `BW_CLIENTID`, `BW_CLIENTSECRET`, `BW_PASSWORD` | A secure note `lifevault/PROJECT` with a hidden field per key |
| `onepassword` | `--vault NAME` | `OP_SERVICE_ACCOUNT_TOKEN` | An item `lifevault/PROJECT` with a concealed field per key |
| `hashicorp` | `--address URL [--mount NAME] [--path-prefix P] [--namespace NS] [--ca-cert PATH] [--kv-version 1\|2]` | `VAULT_TOKEN` | A KV secret at `MOUNT/PREFIX/project` (defaults: `secret`, `lifevault`, KV v2) |
| `ansible` | `--dir PATH` | `ANSIBLE_VAULT_PASSWORD` | An encrypted `DIR/project.yml` file |

The login credentials for each manager are stored **as Lifevault secrets** and
referenced by name. They're handed only to that manager's CLI and are never
pushed anywhere:

```sh
lifevault set BW_ID; lifevault set BW_SECRET; lifevault set BW_PW
lifevault target add home --type bitwarden --server https://vault.example.com \
    --folder Projects --all-projects \
    --auth BW_CLIENTID=BW_ID --auth BW_CLIENTSECRET=BW_SECRET --auth BW_PASSWORD=BW_PW

lifevault set OP_TOKEN
lifevault target add work --type onepassword --vault Dev --project my-api \
    --auth OP_SERVICE_ACCOUNT_TOKEN=OP_TOKEN

lifevault push --dry-run
```

- **Requirements.** The matching official CLI must be installed: `bw`, `op`, `vault`
  or `ansible-vault`.
- **No secrets on command lines.** Values go to the CLI over stdin and credentials
  through its environment, so neither appears in process listings. Each CLI starts
  with a minimal environment. Bitwarden gets its own isolated config folder.
- **Only its own items.** Lifevault only changes items it created and marked. It
  won't take over an existing item that has the same name.
- **Edits in the managers are overwritten** on the next push.
- **Failures are retried.** A failed push prints a warning, stays pending, and is
  retried by later commands.
- **Least privilege.** Give each manager a dedicated, minimal credential: a
  Bitwarden API key for a dedicated account or collection, a 1Password service
  account limited to one vault, or a Vault token whose policy only covers the
  `lifevault/` path.

### Importing from other sources

| Command | What it does |
|---|---|
| `lifevault import-ansible FILE FIELDS... [OPTIONS]` | Import string fields from an Ansible Vault YAML file. Needs `--vault-password-file PATH` or `--vault-id ID@SOURCE`. |
| `lifevault import-hashicorp PATH FIELDS... [OPTIONS]` | Import string fields from HashiCorp Vault KV, using an authenticated `vault` CLI. `--kv-version 1\|2`. |
| `lifevault import-provider PROVIDER [--prefix PREFIX] [--replace]` | Import a hosting-provider credential bundle from the environment. |
| `lifevault import-provider --list` | List bundles and the variables each one needs. |

Options for the Ansible and HashiCorp imports:
- `FIELDS...` selects fields; `SOURCE=NAME` renames one as it's stored.
- `--all` imports every field instead of selecting some.
- `--replace` allows overwriting existing names.
- `--auto-refresh SECONDS` tracks the source (see [Automatic refresh](#automatic-refresh)).

Provider bundles cover Hetzner Cloud, Hetzner Storage Box, OVHcloud (API keys or
OAuth), DigitalOcean, Linode, Vultr and Scaleway.

### SSH keys

| Command | What it does |
|---|---|
| `lifevault import-ssh NAME FILE [--replace]` | Store an owner-only SSH private-key file. |
| `lifevault ssh-add NAME` | Load a stored key into `ssh-agent` for one hour. |

### Automatic refresh

Any import that supports `--auto-refresh SECONDS` (1 to 604800) is tracked. When
the interval has passed, the source is re-read before the next `get`, `run` or
`ssh-add`.

| Command | What it does |
|---|---|
| `lifevault refresh` | Refresh all tracked imports now. Fails loudly on errors. |
| `lifevault refresh --disable` | Stop tracking all sources. Stored values are kept. |

If a refresh triggered by `get`/`run` fails, you get a warning and the stored
value is used.

### Updates

| Command | What it does |
|---|---|
| `lifevault update` | Install the latest stable release for your OS and architecture. |
| `lifevault update --check` | Report whether a newer release exists, without downloading it. |
| `lifevault update enable` / `disable` / `status` | Manage the optional daily automatic update. It's off by default. |

- **Signature checks.** Updates come from this repository. Every archive must carry
  a valid Ed25519 signature from the release key embedded in the binary. The
  updater also rejects downgrades, unsafe archives and wrong architectures.
- **Background updates.** Automatic updates run in the background and never delay
  your command. You see a one-line notice only when something was installed or
  failed.
- **Package managers.** Installs managed by a package manager (including
  `/nix/store`) must be updated through that package manager.
- **Upgrading from 0.2.3 or earlier.** Those versions check a private repository,
  so update them once by reinstalling from this page. After that, updates need no
  token.

### Credential broker

`lifevault broker --help` covers an optional standalone broker. It issues
short-lived credentials to scoped machine and user identities, rotates and revokes
leases, and serves a local credential proxy. Providers include AWS, Azure, GCP,
PostgreSQL, MySQL, GitHub, GitLab, Cloudflare, Hetzner Storage Box and OVHcloud.

| Command | What it does |
|---|---|
| `lifevault broker init CONFIG.json` | Create broker state and print a 12-hour admin token. |
| `lifevault broker serve --listen IP:PORT [--cert PEM --key PEM]` | Run the broker and its rotation/cleanup worker. TLS is required for non-loopback addresses. |
| `lifevault broker agent CONFIG.json` | Run a scoped local credential proxy. |
| `lifevault broker request URL METHOD PATH TOKEN_FILE [BODY_FILE]` | Call the broker with a token read from a private file. |
| `lifevault broker providers` | List the provider adapters. |
| `lifevault broker admin-token`, `configure`, `provider-auth`, `agent-token` | Recovery and configuration commands; see `broker --help`. |

The full broker guide is in `docs/broker.md` inside each release archive.

### Global options

These go before the command:

| Option | What it does |
|---|---|
| `--vault PATH` | Use a specific vault file. The default is `$XDG_DATA_HOME/lifevault/secrets.vault`, or `~/.local/share/lifevault/secrets.vault`. |
| `--no-keyring` | Skip the OS keyring. |
| `-h`, `--help` | Show help. |
| `--version` | Show the version. |

---

## Security model

- **Encryption.** XChaCha20-Poly1305 encrypts names and values. The key comes from
  Argon2id (19 MiB memory, 2 passes), with a fresh random salt and nonce on every
  save. The file header is authenticated, so format downgrades are rejected.
- **Safe writes.** Writes are atomic (temp file, fsync, rename) and serialized with
  a lock file. Vault files are owner-only, and symlinked vault files are refused.
- **Memory.** Secret buffers are wiped after use where practical. The unlock
  agent locks its password in memory, and its process can't be inspected or dumped.
- **What it doesn't protect against.** Encryption protects a copied vault while
  it's locked. It does not protect against a compromised user account or root. Use
  a strong, unique master password: **there is no recovery**.
- **Not externally audited.** Lifevault has not had an external security audit.

## Verifying a download by hand

Release public key (Ed25519):

```
-----BEGIN PUBLIC KEY-----
MCowBQYDK2VwAyEAF728t9PhEWg5dXkllKBPnIAQ3ESBHtOUzWQWxYWIwjE=
-----END PUBLIC KEY-----
```

Save it as `lifevault-release.pub`, then check an archive (OpenSSL 3):

```sh
openssl pkeyutl -verify -rawin -pubin -inkey lifevault-release.pub \
    -in lifevault-$TARGET.tar.gz -sigfile lifevault-$TARGET.tar.gz.sig
# -> Signature Verified Successfully
```

The same key is built into the `lifevault update` command, which refuses any
archive whose signature doesn't verify.
