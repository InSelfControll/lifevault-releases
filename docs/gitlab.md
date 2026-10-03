# GitLab

The `gitlab` provider creates a project access token with fixed scopes and
access level, and deletes it when the lease ends. It works with GitLab.com and
self-managed instances over HTTPS.

Set up the broker first: see [broker basics](broker-basics.md).

## Prerequisites

- Python 3 on the broker host. No GitLab CLI is needed.
- A project, and a provisioning token (`admin_token`) for a user allowed to
  create and delete that project's access tokens. Project access tokens depend on
  your GitLab edition, tier and the issuer's role.

## Create a least-privilege provisioning setup

- Use a provisioning token limited to what token management needs, held by a
  user with the lowest project role that can manage project access tokens
  (usually Maintainer).
- Choose the narrowest `scopes`. They must include `read_api` or `api`, because
  the broker verifies the token through the API.
- Choose the lowest `access_level`: 10 (Guest), 20 (Reporter), 30 (Developer) or
  40 (Maintainer). GitLab decides which levels are allowed.

## Profile

Merge this entry into the `profiles` object of the broker configuration:

```json
{
  "ci-gitlab": {
    "provider": "gitlab",
    "ttl_seconds": 3600,
    "rotation_seconds": 2700,
    "overlap_seconds": 0,
    "config": {
      "base_url": "https://gitlab.example.com",
      "project_id": "<PROJECT_ID>",
      "admin_token": "<PROVISIONING_TOKEN>",
      "scopes": ["read_api", "read_repository"],
      "access_level": 20
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `project_id` | yes | Numeric project ID. |
| `admin_token` | yes | Token that can manage the project's access tokens. |
| `scopes` | yes | Nonempty array; must include `read_api` or `api`. |
| `access_level` | yes | 10 to 40. |
| `base_url` | no | Default `https://gitlab.com`. HTTPS only. |

## Issued credentials

`GET /v1/credentials/<PROFILE>` returns these keys in `credentials`:

| Key | Value |
|---|---|
| `GITLAB_TOKEN` | Project access token |

## Lifetime, rotation and revocation

- GitLab expiry is a calendar date (it expires at midnight UTC after the lease
  ends). The broker worker deletes the token at the exact lease deadline, so it
  must keep running for sub-day lifetimes.
- Revocation deletes the named project token.

## Replacing the provisioning credential

Stop the broker. Create an owner-only JSON file containing only `admin_token`:

```json
{"admin_token": "<NEW_PROVISIONING_TOKEN>"}
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
- **Issuance fails.** Check that the instance and tier support project access
  tokens and that the provisioning user's role allows the requested
  `access_level`.
