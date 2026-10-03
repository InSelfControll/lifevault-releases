# AWS

The broker has two AWS providers:

- `aws-sts` assumes an IAM role and returns a temporary session.
- `aws-iam` creates a dedicated IAM user with an access key and one
  customer-managed policy, and deletes it when the lease ends.

Prefer `aws-sts`: credentials expire on their own and nothing persistent is created.

Set up the broker first: see [broker basics](broker-basics.md).

## Prerequisites

- AWS CLI v2 (`aws`) and Python 3 on the broker host.
- Provisioning credentials: either an access key pair in the profile
  (`credentials`) or a named AWS CLI `profile` of the broker's service user.
  Instance metadata lookup is disabled.

## Create a least-privilege provisioning setup

**`aws-sts`.** Create a role that holds only the permissions consumers need.
Its trust policy should allow only the provisioning principal to assume it.
Give the provisioning principal `sts:AssumeRole` on that role only. Optionally
require an `external_id`, and narrow each session further with `session_policy`.

**`aws-iam`.** Create a customer-managed policy with the consumer's permissions.
Grant the provisioning principal only what it needs to manage users under the
path `/lifevault/` with Lifevault ownership tags: create, tag, inspect and delete
those users, attach and detach the configured policy, and manage their access
keys. It also needs `iam:PutUserPolicy`, `iam:ListUserPolicies` and
`iam:DeleteUserPolicy` for a fixed inline guard. That guard,
`lifevault-identity-guard`, denies `iam:*`, `sts:*`, `organizations:*` and
`account:*` to the temporary user even if the policy allows them. Your policy
must still avoid indirect escalation paths, such as changing privileged compute
workloads or reading other administrators' credentials.

## Profile

Merge this entry into the `profiles` object of the broker configuration:

```json
{
  "build-aws": {
    "provider": "aws-sts",
    "ttl_seconds": 3600,
    "rotation_seconds": 2700,
    "overlap_seconds": 0,
    "config": {
      "role_arn": "arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>",
      "profile": "<PROVISIONER_PROFILE>",
      "region": "us-east-1"
    }
  },
  "deploy-iam": {
    "provider": "aws-iam",
    "ttl_seconds": 3600,
    "rotation_seconds": 0,
    "overlap_seconds": 0,
    "config": {
      "policy_arn": "arn:aws:iam::<ACCOUNT_ID>:policy/<POLICY_NAME>",
      "credentials": {
        "AWS_ACCESS_KEY_ID": "<PROVISIONER_KEY_ID>",
        "AWS_SECRET_ACCESS_KEY": "<PROVISIONER_SECRET>"
      }
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `role_arn` | `aws-sts` | Role to assume. |
| `policy_arn` | `aws-iam` | Customer-managed policy in the same account. |
| `credentials` or `profile` | yes | `credentials`: object with `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, optional `AWS_SESSION_TOKEN`. `profile`: AWS CLI profile of the service user (resolved through its `HOME`). |
| `region` | no | Default `us-east-1`. |
| `external_id` | no | `aws-sts` only. |
| `session_policy` | no | `aws-sts` only. JSON policy object that narrows the session. |

## Issued credentials

`GET /v1/credentials/<PROFILE>` returns these keys in `credentials`:

| Key | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key ID |
| `AWS_SECRET_ACCESS_KEY` | Secret access key |
| `AWS_SESSION_TOKEN` | `aws-sts` only |

## Lifetime, rotation and revocation

- **`aws-sts`**: 900 to 43,200 seconds, bounded by the role's maximum session
  duration; role chaining caps it at 3,600. Revocation is `expiry_only`: the broker
  stops handing the session out immediately, but it stays valid at AWS until it
  expires.
- **`aws-iam`**: no native expiry. The broker worker deletes the access key,
  detaches the policy, removes the guard and deletes the user. If the broker is
  down, the user stays active. AWS propagation delays apply.
- Rotation issues and verifies a new credential before switching. A failed
  verification keeps the previous lease.

## Replacing the provisioning credential

Stop the broker. Create an owner-only JSON file containing only `credentials` (a profile-based setup refreshes through its own credential source):

```json
{"credentials": {"AWS_ACCESS_KEY_ID": "<NEW_KEY_ID>", "AWS_SECRET_ACCESS_KEY": "<NEW_SECRET>"}}
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
- **`aws` not found.** The adapter uses a sanitized `PATH` (standard system paths
  and `/run/current-system/sw/bin`). Install AWS CLI v2 there.
- **TTL rejected for `aws-sts`.** The minimum is 900 seconds, and the role's
  maximum session duration must allow the requested TTL.
- **`profile` not found.** The broker passes only its own absolute `HOME`; the
  profile must exist in that user's AWS configuration.
