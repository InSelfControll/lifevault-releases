# OVHcloud

Lifevault can store two kinds of existing OVHcloud API credentials in the local
vault: legacy application keys (`ovh`) and OAuth service-account credentials
(`ovh-oauth`). The optional broker can also create temporary OAuth service
accounts with an exact, expiring IAM policy.

| Need | Use |
|---|---|
| Store an existing application key, secret and validated consumer key | [`ovh` bundle](#legacy-api-keys-ovh) |
| Store an existing OAuth client ID and secret | [`ovh-oauth` bundle](#oauth-service-account-ovh-oauth) |
| Hand out short-lived, narrowly scoped OAuth credentials | [Broker leases](#broker-oauth-leases) |

Note the naming: the bundle `ovh` holds **legacy** API keys, while the broker
provider `ovh` issues **OAuth** credentials.

## Legacy API keys (`ovh`)

### Create the credential

Create an application key and secret, then a consumer key, through OVHcloud's
API token pages for your region. Limit the consumer key to the HTTP methods and
paths your tool needs. Consumer-key enrollment requires human validation, so
Lifevault only stores an already validated key.

### Store it

```sh
read -rs OVH_APPLICATION_KEY && export OVH_APPLICATION_KEY
read -rs OVH_APPLICATION_SECRET && export OVH_APPLICATION_SECRET
read -rs OVH_CONSUMER_KEY && export OVH_CONSUMER_KEY
export OVH_ENDPOINT=ovh-eu        # optional
lifevault import-provider ovh
unset OVH_APPLICATION_KEY OVH_APPLICATION_SECRET OVH_CONSUMER_KEY OVH_ENDPOINT
```

| Variable | Required |
|---|---|
| `OVH_APPLICATION_KEY` | yes |
| `OVH_APPLICATION_SECRET` | yes |
| `OVH_CONSUMER_KEY` | yes |
| `OVH_ENDPOINT` | no (for example `ovh-eu`, `ovh-ca`, `ovh-us`) |

## OAuth service account (`ovh-oauth`)

### Create the credential

Create an OAuth service account (client-credentials flow) and attach an IAM
policy that grants only the resources and actions your tool needs. See OVHcloud's
[service account guide](https://docs.ovhcloud.com/en/guides/manage-and-operate/api/manage-service-account)
and [IAM policy guide](https://docs.ovhcloud.com/en/guides/account-and-service-management/account-information/iam-policies-api).

### Store it

```sh
read -rs OVH_CLIENT_ID && export OVH_CLIENT_ID
read -rs OVH_CLIENT_SECRET && export OVH_CLIENT_SECRET
export OVH_ENDPOINT=ovh-eu        # optional
lifevault import-provider ovh-oauth --prefix PROD_
unset OVH_CLIENT_ID OVH_CLIENT_SECRET OVH_ENDPOINT
```

| Variable | Required |
|---|---|
| `OVH_CLIENT_ID` | yes |
| `OVH_CLIENT_SECRET` | yes |
| `OVH_ENDPOINT` | no |

## Use the stored credentials

```sh
lifevault run OVH_APPLICATION_KEY OVH_APPLICATION_SECRET OVH_CONSUMER_KEY OVH_ENDPOINT -- <your-ovh-client>

# With a prefix, map names back to what the client reads:
lifevault run PROD_OVH_CLIENT_ID=OVH_CLIENT_ID \
  PROD_OVH_CLIENT_SECRET=OVH_CLIENT_SECRET -- <your-ovh-client>
```

Use a `PROJECT__` prefix (for example `--prefix MY_APP__`) to make the bundle
part of a project for `run --project` and push targets.

## Updating stored bundles

- Imports make no OVHcloud API calls and never create, rotate or refresh keys.
- After rotating at OVHcloud, export the new values and import with `--replace`.
- Omitting `OVH_ENDPOINT` on a re-import leaves any stored value unchanged. Use
  `lifevault remove <NAME>` to drop it.
- Validation or a name collision leaves the vault unchanged.

## Broker OAuth leases

The broker creates a dedicated client-credentials service account and an IAM
policy with your exact resources, actions and an `expiredAt` deadline. Set up the
broker first: see [broker basics](broker-basics.md).

### Prerequisites

- A provisioning OAuth service account whose permissions allow service-account
  and IAM-policy lifecycle and discovery in the target account.
- The resource URNs and action names for your product, found through OVHcloud IAM.
- An authenticated GET path that requires the issued permission (used to verify).
- Python 3 on the broker host. No OVHcloud CLI is needed.

### Profile

```json
"vps-read": {
  "provider": "ovh",
  "ttl_seconds": 3600,
  "rotation_seconds": 2700,
  "overlap_seconds": 0,
  "config": {
    "region": "eu",
    "account_id": "<ACCOUNT_ID>",
    "admin_client_id": "<PROVISIONER_CLIENT_ID>",
    "admin_client_secret": "<PROVISIONER_CLIENT_SECRET>",
    "resources": ["urn:v1:eu:resource:vps:<VPS_NAME>"],
    "actions": ["vps:apiovh:get"],
    "verify_path": "/1.0/vps/<VPS_NAME>"
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `account_id` | yes | Account the provisioner belongs to; checked before issuing. |
| `admin_client_id`, `admin_client_secret` | yes | Provisioning service account. |
| `resources` | yes | Exact URNs. Must match `region`. No wildcards or resource groups. |
| `actions` | yes | Exact actions. No wildcards or implicit permission groups. |
| `verify_path` | yes | Authenticated GET that needs the granted permission. A public endpoint proves nothing. |
| `region` | no | `eu` (default), `ca` or `us`. |

### Issued credentials

| Key | Value |
|---|---|
| `OVH_CLIENT_ID` | New service-account client ID |
| `OVH_CLIENT_SECRET` | Its secret |
| `OVH_ENDPOINT` | `ovh-eu`, `ovh-ca` or `ovh-us` |

Consumers exchange these for a bearer token at the normal regional OAuth endpoint.

### Lifetime and cleanup

- The IAM policy's `expiredAt` bounds authorization. The client secret itself
  does not expire; cleanup deletes the policy, then the service account, and
  confirms each with HTTP 404.
- Bearer tokens already minted can keep working during OVHcloud propagation.
- Do not attach other policies to managed accounts, and do not grant them
  identity-administration rights. Broader policies would override the deadline.
- Large accounts, rate limits and propagation can delay cleanup; the broker keeps
  the record and retries.

### Replacing the provisioning credential

Stop the broker. Create an owner-only JSON file with only the changed fields:

```json
{"admin_client_id": "<NEW_CLIENT_ID>", "admin_client_secret": "<NEW_CLIENT_SECRET>"}
```

```sh
lifevault --vault /secure/broker.vault broker provider-auth vps-read /secure/ovh-auth.json
```

Keep the same account, region, resources and policy scope. Outstanding leases
are kept; issued credentials are not rotated.
