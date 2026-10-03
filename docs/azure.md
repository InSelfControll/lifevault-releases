# Azure (Microsoft Entra ID)

The `azure` provider adds a new, expiring client secret (password) to an
existing Entra application and removes it when the lease ends. Consumers
authenticate as that application.

Set up the broker first: see [broker basics](broker-basics.md).

## Prerequisites

- Python 3 on the broker host. No Azure CLI is needed.
- An existing Entra application (app registration) whose permissions are what
  consumers should get. Note its object ID, client (application) ID and tenant ID.
- A Microsoft Graph access token (`admin_token`) allowed to manage this
  application's passwords.

## Create a least-privilege provisioning setup

- Create a dedicated application for each consumer group and give it only the
  role assignments it needs.
- The provisioning Graph token should be able to add and remove passwords on
  that application only. A common choice is an identity that owns the
  application and holds `Application.ReadWrite.OwnedBy`; check this against your
  tenant's policies.
- Graph access tokens are usually short-lived. The broker needs a valid
  `admin_token` whenever it issues or revokes, so plan how you renew it (see
  below).

## Profile

Merge this entry into the `profiles` object of the broker configuration:

```json
{
  "app-azure": {
    "provider": "azure",
    "ttl_seconds": 3600,
    "rotation_seconds": 2700,
    "overlap_seconds": 0,
    "config": {
      "application_object_id": "<APPLICATION_OBJECT_ID>",
      "client_id": "<CLIENT_ID>",
      "tenant_id": "<TENANT_ID>",
      "admin_token": "<GRAPH_ACCESS_TOKEN>"
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `application_object_id` | yes | Object ID of the application (GUID). |
| `client_id` | yes | Application (client) ID (GUID). |
| `tenant_id` | yes | Tenant ID (GUID). |
| `admin_token` | yes | Graph token that can manage this application's passwords. |

## Issued credentials

`GET /v1/credentials/<PROFILE>` returns these keys in `credentials`:

| Key | Value |
|---|---|
| `AZURE_CLIENT_ID` | Application client ID |
| `AZURE_CLIENT_SECRET` | New password |
| `AZURE_TENANT_ID` | Tenant ID |

## Lifetime, rotation and revocation

- The password's expiry is set to the lease deadline.
- Verification exchanges the new password at the tenant's v2 token endpoint.
- Revocation removes the matching password. OAuth access tokens already issued
  with it stay valid until their own expiry.
- Existing passwords on the application are never removed.

## Replacing the provisioning credential

Stop the broker. Create an owner-only JSON file containing only `admin_token`:

```json
{"admin_token": "<NEW_GRAPH_ACCESS_TOKEN>"}
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
- **Issuance fails after a while.** The Graph `admin_token` has probably expired.
  Replace it with `provider-auth`.
