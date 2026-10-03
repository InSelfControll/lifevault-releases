# Cloudflare

The `cloudflare` provider creates a user-owned API token with explicit
policies and an exact expiry, and deletes it when the lease ends.

Set up the broker first: see [broker basics](broker-basics.md).

## Prerequisites

- Python 3 on the broker host. No Cloudflare CLI is needed.
- A provisioning API token (`admin_token`) allowed to create and delete API
  tokens for its user.
- The account or zone IDs and permission group IDs for the access you want to
  grant.

## Create a least-privilege provisioning setup

- Give the provisioning token only API-token management rights (Cloudflare's
  "create additional tokens" template); it does not need resource permissions
  itself beyond what it delegates.
- Write `policies` with explicit `resources` and `permission_groups`. Lifevault
  never adds blanket permissions. Scope resources to specific zones or accounts.

## Profile

Merge this entry into the `profiles` object of the broker configuration:

```json
{
  "dns-edit": {
    "provider": "cloudflare",
    "ttl_seconds": 3600,
    "rotation_seconds": 2700,
    "overlap_seconds": 0,
    "config": {
      "admin_token": "<PROVISIONING_TOKEN>",
      "policies": [
        {
          "effect": "allow",
          "resources": {"com.cloudflare.api.account.zone.<ZONE_ID>": "*"},
          "permission_groups": [{"id": "<PERMISSION_GROUP_ID>"}]
        }
      ]
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `admin_token` | yes | Token that can create and delete API tokens. |
| `policies` | yes | Nonempty array. Each policy needs `resources` and `permission_groups`, and a provider-supported `effect`. |

## Issued credentials

`GET /v1/credentials/<PROFILE>` returns these keys in `credentials`:

| Key | Value |
|---|---|
| `CLOUDFLARE_API_TOKEN` | New API token |

## Lifetime, rotation and revocation

- The token's `expires_on` is the lease deadline.
- Revocation deletes the matching named token. Cloudflare API propagation
  applies.

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
- **Issuance fails.** Check permission group IDs and resource keys against
  Cloudflare's API documentation, and that the provisioning token can create
  tokens.
