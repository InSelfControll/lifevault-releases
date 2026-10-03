# Google Cloud

The `gcp` provider impersonates a service account and returns a short-lived
OAuth access token. Nothing persistent is created.

Set up the broker first: see [broker basics](broker-basics.md).

## Prerequisites

- Python 3 on the broker host. No `gcloud` is needed.
- A target service account with only the roles consumers need.
- A provisioning OAuth access token (`admin_token`) for an identity allowed to
  impersonate that service account.

## Create a least-privilege provisioning setup

- Grant consumer permissions to the target service account only.
- Give the provisioning identity the Service Account Token Creator role
  (`roles/iam.serviceAccountTokenCreator`) on that one service account, not on the
  project.
- Pick a `verify_url`: a harmless GET under `*.googleapis.com` that the target
  service account may read, such as bucket metadata. Never use a public endpoint
  that succeeds without authentication.
- OAuth access tokens are usually short-lived. Plan how you renew
  `admin_token` (see below).

## Profile

Merge this entry into the `profiles` object of the broker configuration:

```json
{
  "gcs-read": {
    "provider": "gcp",
    "ttl_seconds": 3600,
    "rotation_seconds": 2700,
    "overlap_seconds": 0,
    "config": {
      "service_account": "<NAME>@<PROJECT_ID>.iam.gserviceaccount.com",
      "admin_token": "<PROVISIONER_ACCESS_TOKEN>",
      "scopes": ["https://www.googleapis.com/auth/cloud-platform"],
      "verify_url": "https://storage.googleapis.com/storage/v1/b/<BUCKET>"
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `service_account` | yes | Email of the service account to impersonate. |
| `admin_token` | yes | Access token of an identity with impersonation rights. |
| `scopes` | yes | Nonempty array (at most 32) of OAuth scopes. |
| `verify_url` | yes | HTTPS GET under `*.googleapis.com` that requires the issued identity. |

## Issued credentials

`GET /v1/credentials/<PROFILE>` returns these keys in `credentials`:

| Key | Value |
|---|---|
| `GOOGLE_OAUTH_ACCESS_TOKEN` | Short-lived access token |

## Lifetime, rotation and revocation

- Lifetime is 1 to 3,600 seconds; the profile TTL may not exceed 3,600.
- Revocation is `expiry_only`: the broker stops handing the token out at once,
  but the token stays valid at Google until it expires.

## Replacing the provisioning credential

Stop the broker. Create an owner-only JSON file containing only `admin_token`:

```json
{"admin_token": "<NEW_PROVISIONER_ACCESS_TOKEN>"}
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
- **TTL rejected.** GCP leases are capped at 3,600 seconds.
- **Verification fails.** Check that the service account can read `verify_url`
  and that the host ends in `.googleapis.com`.
