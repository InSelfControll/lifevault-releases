# PostgreSQL

The `postgres` provider creates a new login role that inherits only one
configured role, with a password that expires (`VALID UNTIL`). When the lease
ends it disables the login, terminates its sessions and drops it.

Set up the broker first: see [broker basics](broker-basics.md).

## Prerequisites

- PostgreSQL 14 or later, and `psql` plus Python 3 on the broker host.
- TLS with full verification (`verify-full`) to the server. Pass `ca_file` for a
  private CA.
- SCRAM password authentication for the issued logins. `VALID UNTIL` does not
  expire trust or certificate authentication.

## Create a least-privilege provisioning setup

- Create a group role (no login) with only the data privileges consumers need.
  Do not grant schema or object ownership or role administration.
- Create a provisioning user that can create and drop login roles, grant the
  group role, and terminate sessions of the issued logins (for example through
  `pg_signal_backend`). Check the exact privileges for your PostgreSQL version.
- Reserve the `lv_` name prefix for the broker.
- Make sure statement and audit logging redacts role-creation statements;
  Lifevault cannot control server log settings.

## Profile

Merge this entry into the `profiles` object of the broker configuration:

```json
{
  "app-db": {
    "provider": "postgres",
    "ttl_seconds": 3600,
    "rotation_seconds": 2700,
    "overlap_seconds": 60,
    "config": {
      "host": "<DB_HOST>",
      "database": "<DB_NAME>",
      "admin_user": "<PROVISIONER_USER>",
      "admin_password": "<PROVISIONER_PASSWORD>",
      "role": "<APP_ROLE>",
      "ca_file": "/secure/db-ca.pem"
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `host` | yes | Database host name. |
| `database` | yes | Database name. |
| `admin_user` | yes | Provisioning user. |
| `admin_password` | yes | Its password. |
| `role` | yes | Existing role whose privileges the new login receives. |
| `port` | no | Default 5432. |
| `ca_file` | no | PEM CA file for server verification. |

## Issued credentials

`GET /v1/credentials/<PROFILE>` returns these keys in `credentials`:

| Key | Value |
|---|---|
| `DB_USERNAME` | New login name (starts with `lv_`) |
| `DB_PASSWORD` | Generated password |
| `DB_HOST` | `host` from the profile |
| `DB_DATABASE` | `database` from the profile |

## Lifetime, rotation and revocation

- `VALID UNTIL` expires password authentication at the lease deadline.
- Revocation disables the login, terminates its sessions (waiting up to one
  second per session and re-checking), then drops the role. Any remaining session
  fails cleanup, which is retried.
- Objects owned by an issued login prevent dropping it. Keep ownership with an
  administrator role.
- Applications must reconnect with the new credentials after rotation; existing
  connection pools are not updated.

## Replacing the provisioning credential

Stop the broker. Create an owner-only JSON file containing only `admin_password`:

```json
{"admin_password": "<NEW_PROVISIONER_PASSWORD>"}
```

```sh
lifevault --vault /secure/broker.vault broker provider-auth <PROFILE> /secure/auth.json
```

Outstanding leases and cleanup records are kept; already issued credentials are
not rotated. Start the broker again and delete the file.

## Troubleshooting

- **Errors are generic.** The adapter never prints provider responses, because
  they can contain secrets. Check the provisioning credential and its
  permissions directly with the provider.
- **Every operation is bounded.** Provider calls time out after 30 seconds and
  each adapter operation after 45 seconds. A timed-out request can still finish;
  check `GET /v1/leases` before retrying a manual rotation.
- **Connection fails.** TLS `verify-full` is mandatory: the certificate must match
  `host` and chain to a trusted CA (or `ca_file`).
- **Cleanup stays `revoke_pending`.** Check that the provisioning user may
  terminate the issued login's sessions and that the login owns no objects.
