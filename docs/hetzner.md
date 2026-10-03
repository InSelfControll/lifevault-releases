# Hetzner

Lifevault can store two kinds of existing Hetzner credentials in the local vault:
a Hetzner Cloud API token (`hetzner-cloud`) and Storage Box login details
(`hetzner-storagebox`). The optional broker can also issue temporary, read-only
Storage Box subaccounts.

| Need | Use |
|---|---|
| Run `hcloud`, Terraform or scripts against Hetzner Cloud | [Hetzner Cloud bundle](#hetzner-cloud-api-token) |
| Store backup or mount credentials for a Storage Box | [Storage Box bundle](#hetzner-storage-box-credentials) |
| Hand out short-lived, read-only Storage Box access | [Storage Box broker leases](#storage-box-broker-leases) |

## Hetzner Cloud API token

### Prerequisites

- A Hetzner Cloud project.
- Only the `lifevault` binary. No Hetzner CLI is needed to store the token.

### Create the token

API tokens are created per project in Hetzner Console. Choose **Read** if your
tools only read state; choose **Read & Write** only when they must change
resources. Use a separate project, or a separate token, for each environment so
you can revoke one without touching the others. See Hetzner's
[token guide](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/).

Lifevault cannot create, rotate or scope Cloud tokens. The broker does not
issue Cloud project tokens.

### Store it

```sh
read -rs HCLOUD_TOKEN && export HCLOUD_TOKEN
lifevault import-provider hetzner-cloud
unset HCLOUD_TOKEN
```

| Variable | Required | Stored as |
|---|---|---|
| `HCLOUD_TOKEN` | yes | `HCLOUD_TOKEN` (or `<PREFIX>HCLOUD_TOKEN`) |

For several projects, use a prefix and map the name back when running:

```sh
lifevault import-provider hetzner-cloud --prefix PROD_
lifevault run PROD_HCLOUD_TOKEN=HCLOUD_TOKEN -- hcloud server list
```

### Use it

```sh
lifevault run HCLOUD_TOKEN -- hcloud server list
lifevault run HCLOUD_TOKEN -- terraform plan
```

To make the token part of a project, use a `PROJECT__` prefix. The token is
then injected by `run --project` and pushed with that project to any push target:

```sh
lifevault import-provider hetzner-cloud --prefix MY_APP__
lifevault run --project my-app -- terraform apply
```

## Hetzner Storage Box credentials

### Prerequisites

- A Storage Box and the host, username and password of an account on it
  (the main account or a subaccount).

### Create the credential

Prefer a subaccount restricted to one directory, read-only when the client only
reads. Create it in Hetzner Console. See Hetzner's
[Storage Box access guide](https://docs.hetzner.com/storage/storage-box/general/).
Local imports accept writable credentials as well as read-only ones.

### Store it

```sh
read -rs HETZNER_STORAGEBOX_HOST && export HETZNER_STORAGEBOX_HOST
read -rs HETZNER_STORAGEBOX_USERNAME && export HETZNER_STORAGEBOX_USERNAME
read -rs HETZNER_STORAGEBOX_PASSWORD && export HETZNER_STORAGEBOX_PASSWORD
lifevault import-provider hetzner-storagebox --prefix BACKUP_
unset HETZNER_STORAGEBOX_HOST HETZNER_STORAGEBOX_USERNAME HETZNER_STORAGEBOX_PASSWORD
```

| Variable | Required |
|---|---|
| `HETZNER_STORAGEBOX_HOST` | yes |
| `HETZNER_STORAGEBOX_USERNAME` | yes |
| `HETZNER_STORAGEBOX_PASSWORD` | yes |

These names are Lifevault conventions, not names Hetzner tools read. Map them to
what your client expects:

```sh
lifevault run BACKUP_HETZNER_STORAGEBOX_HOST=<CLIENT_HOST_VAR> \
  BACKUP_HETZNER_STORAGEBOX_USERNAME=<CLIENT_USER_VAR> \
  BACKUP_HETZNER_STORAGEBOX_PASSWORD=<CLIENT_PASSWORD_VAR> -- <your-backup-command>
```

## Updating stored bundles

- Imports make no Hetzner API calls. They do not create, rotate or refresh
  credentials, and `--auto-refresh` does not apply.
- After you rotate a token or password in Hetzner Console, export the new value
  and import again with `--replace`.
- All required variables must be present, nonempty UTF-8. The bundle is saved in
  one write: a missing variable or a name collision leaves the vault unchanged.
- Without `--replace`, existing names cause an error.

## Storage Box broker leases

The broker creates a read-only subaccount restricted to one directory, verifies
it, hands it out, and deletes it when the lease ends. Set up the broker first:
see [broker basics](broker-basics.md).

### Prerequisites

- A Storage Box managed through the current Hetzner Console API
  (`https://api.hetzner.com/v1`). A Robot-only Storage Box must be migrated to
  Console first.
- A Hetzner Console API token for that project that may create, inspect and
  delete subaccounts. This is the broker's provisioning token.
- An existing relative directory on the Storage Box, owned by a trusted
  administrator, with no inherited SSH `authorized_keys`.
- Python 3 on the broker host. No Hetzner CLI is needed.
- The broker host must reach the Storage Box on port 21 (FTPS) and have a
  current CA trust store.

### Profile

Add a profile like this under `profiles` in the broker configuration:

```json
"backup-read": {
  "provider": "hetzner-storagebox",
  "ttl_seconds": 3600,
  "rotation_seconds": 2700,
  "overlap_seconds": 0,
  "config": {
    "storage_box_id": <STORAGE_BOX_ID>,
    "admin_token": "<PROVISIONING_TOKEN>",
    "home_directory": "<relative/dir>",
    "access_settings": {
      "readonly": true,
      "reachable_externally": true,
      "ssh_enabled": false,
      "samba_enabled": false,
      "webdav_enabled": false
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `storage_box_id` | yes | Numeric Storage Box ID. |
| `admin_token` | yes | Provisioning token with subaccount create/inspect/delete rights. |
| `home_directory` | yes | Existing relative path. Empty, absolute and `..` paths are rejected. |
| `access_settings` | no | Boolean map. `readonly` must be `true`. Defaults: read-only, not externally reachable, SSH/Samba/WebDAV off. |

- Set `reachable_externally` to `true` when the broker or consumer is outside
  Hetzner's network. Also enable external reachability on the parent Storage Box.
- FTP/FTPS (port 21) and SFTP/SCP (port 22) are always enabled, subject to
  network settings. `ssh_enabled` controls port 23 only.
- Writable managed subaccounts are refused: a writer could leave SSH keys in the
  shared directory that later subaccounts would inherit.

### Issued credentials

| Key | Value |
|---|---|
| `HETZNER_STORAGEBOX_USERNAME` | New subaccount username |
| `HETZNER_STORAGEBOX_PASSWORD` | Generated password |
| `HETZNER_STORAGEBOX_HOST` | `<username>.your-storagebox.de` |

The broker verifies each new password with an FTPS login before handing it out.
Verification writes no files.

### Lifetime and cleanup

- Passwords have no native expiry. The running broker worker deletes the
  subaccount at the lease deadline and retries failed deletions. If the broker is
  down, access stays active until it recovers.
- Deleting the subaccount stops new logins. A transfer already in progress may
  not stop immediately. Data in the directory is kept.
- Each active lease uses one subaccount slot, including rotation overlap and
  pending cleanup.
- Consumers must fetch the replacement after rotation, or use the broker agent.

### Replacing the provisioning token

Stop the broker. Create an owner-only file (`umask 077`, then use an editor)
containing only the new token:

```json
{"admin_token": "<NEW_PROVISIONING_TOKEN>"}
```

Then run:

```sh
lifevault --vault /secure/broker.vault broker provider-auth backup-read /secure/storagebox-auth.json
```

Only `admin_token` is accepted. Outstanding leases and cleanup records are kept;
issued credentials are not rotated. Start the broker again and delete the file.
