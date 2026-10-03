# MySQL

The `mysql` provider creates a new TLS-only user with one default role. When
the lease ends it locks the account, terminates its sessions and drops it.

Set up the broker first: see [broker basics](broker-basics.md).

## Prerequisites

- MySQL 8 or later, and the `mysql` client plus Python 3 on the broker host.
- TLS with identity verification (`VERIFY_IDENTITY`). Pass `ca_file` for a
  private CA. Issued users are created with `REQUIRE SSL`.

## Create a least-privilege provisioning setup

- Create a role with only the data privileges consumers need. Do not grant
  account or role administration privileges.
- Create a provisioning user that can create, lock and drop users, grant the
  role, and kill other users' connections. Check the exact privileges for your
  MySQL version.
- Set `account_host` to a restricted client host pattern, not `%`.
- Reserve the `lv_` name prefix for the broker. MySQL user names are limited to
  32 characters.
- Make sure statement and audit logging redacts user-creation statements.

## Profile

Merge this entry into the `profiles` object of the broker configuration:

```json
{
  "app-mysql": {
    "provider": "mysql",
    "ttl_seconds": 3600,
    "rotation_seconds": 2700,
    "overlap_seconds": 60,
    "config": {
      "host": "<DB_HOST>",
      "database": "<DB_NAME>",
      "admin_user": "<PROVISIONER_USER>",
      "admin_password": "<PROVISIONER_PASSWORD>",
      "role": "<APP_ROLE>",
      "account_host": "<CLIENT_HOST_PATTERN>",
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
| `account_host` | yes | Host part of the created account; use a restricted pattern. |
| `port` | no | Default 3306. |
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

- MySQL has no native absolute account expiry. The broker worker must run to
  lock and drop the user at the lease deadline; during an outage the account
  stays usable.
- Revocation locks the account, terminates its sessions and drops the user.
- Applications must reconnect after rotation.

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
- **Connection fails.** TLS identity verification is mandatory: the server
  certificate must match `host` and chain to a trusted CA (or `ca_file`).
- **Consumer cannot connect.** Check that its address matches `account_host` and
  that it connects with TLS.
