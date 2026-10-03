# GitHub

The `github` provider issues GitHub App installation tokens with explicit
permissions and, optionally, a fixed list of repositories.

Set up the broker first: see [broker basics](broker-basics.md).

## Prerequisites

- `openssl` and Python 3 on the broker host.
- A GitHub App you control, installed on the organization or account, with its
  App ID, installation ID and a private key (PEM, at most 16 KiB).

## Create a least-privilege provisioning setup

- Give the App only the permissions consumers could ever need, and install it
  only on the repositories they need.
- In the profile, request the narrowest `permissions` (for example
  `{"contents": "read"}`) and set `repository_ids`. Omit `repository_ids` only when
  every repository in the installation is intended.
- Store the private key in the profile as a JSON string (newlines as `\n`). The
  adapter passes it to `openssl` through a pipe; no PEM file is written.

## Profile

Merge this entry into the `profiles` object of the broker configuration:

```json
{
  "repo-read": {
    "provider": "github",
    "ttl_seconds": 3600,
    "rotation_seconds": 2700,
    "overlap_seconds": 0,
    "config": {
      "app_id": "<APP_ID>",
      "installation_id": "<INSTALLATION_ID>",
      "private_key_pem": "<APP_PRIVATE_KEY_PEM>",
      "permissions": {"contents": "read"},
      "repository_ids": [<REPOSITORY_ID>]
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `app_id` | yes | Numeric App ID. |
| `installation_id` | yes | Numeric installation ID. |
| `private_key_pem` | yes | App private key, at most 16 KiB. |
| `permissions` | yes | Nonempty object of permission names to levels. |
| `repository_ids` | no | Nonempty array of numeric repository IDs. |

## Issued credentials

`GET /v1/credentials/<PROFILE>` returns these keys in `credentials`:

| Key | Value |
|---|---|
| `GITHUB_TOKEN` | Installation access token |

## Lifetime, rotation and revocation

- GitHub tokens expire after one hour; the profile TTL may not exceed 3,600.
  Shorter leases rely on the broker deleting the token.
- Revocation deletes the installation token.
- If a creation response is lost, the token cannot be found again; it expires
  natively within one hour.

## Replacing the provisioning credential

Stop the broker. Create an owner-only JSON file containing only `private_key_pem`:

```json
{"private_key_pem": "<APP_PRIVATE_KEY_PEM>"}
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
- **TTL rejected.** GitHub leases are capped at 3,600 seconds.
- **Issuance fails.** Check that the requested `permissions` are a subset of the
  App's permissions and that the repositories are in the installation.
