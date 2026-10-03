# Broker basics

The optional standalone broker issues short-lived, verified credentials from
cloud, database and code-hosting providers to scoped machine (M2M) and user
(U2M) identities. It rotates and revokes them, and can run a local credential
proxy. It does not need HashiCorp Vault. This page covers the steps every
provider guide relies on. The full reference, `docs/broker.md` and
`docs/provider-capabilities.md`, ships in each release archive.

## Prerequisites

- The `lifevault` binary on the broker host.
- Python 3 and the provider adapter `adapters/lifevault-provider.py` from the
  release archive. Install it at an absolute path, owned by the service user or
  root, executable, and not group- or world-writable. Its parent directories must
  be controlled by the same administrator.
- Provider CLIs only where a guide says so: `aws` (AWS), `psql` (PostgreSQL),
  `mysql` (MySQL), `openssl` (GitHub).
- A dedicated OS service account, encrypted swap, and a **dedicated state file**
  separate from your personal vault.

```sh
lifevault broker providers
# ["aws-sts","aws-iam","azure","gcp","postgres","mysql","github","gitlab","cloudflare","hetzner-storagebox","ovh"]
```

## 1. Write the configuration

Create an owner-only JSON file (`umask 077` first). Each profile names one
provider and its admin-controlled config. Clients only pick a profile; they never
supply provider configuration.

```json
{
  "helper": "/opt/lifevault/adapters/lifevault-provider.py",
  "profiles": {
    "<PROFILE>": {
      "provider": "<PROVIDER_ID>",
      "ttl_seconds": 3600,
      "rotation_seconds": 2700,
      "overlap_seconds": 0,
      "config": { }
    }
  },
  "issuers": []
}
```

| Field | Meaning |
|---|---|
| `helper` | Absolute path of the adapter. |
| `provider` | One ID from `broker providers`. |
| `ttl_seconds` | Lease lifetime, 60 to 43,200. AWS STS minimum 900; GCP and GitHub maximum 3,600. |
| `rotation_seconds` | `0` disables automatic replacement; otherwise 60 to TTL minus one. |
| `overlap_seconds` | 0 to 300, below TTL. Keeps the old verified credential during handoff. |
| `config` | Provider fields. See the provider guide. |
| `issuers` | OIDC issuers for user identities (see below). |

## 2. Initialize and serve

```sh
umask 077
lifevault --vault /secure/broker.vault broker init /secure/broker-config.json > /secure/admin.json
lifevault --vault /secure/broker.vault broker serve --listen 127.0.0.1:8719
```

- `init` asks for a master password and prints a 12-hour administrator token
  once. Copy the `token` string from `admin.json` into an owner-only file such as
  `/secure/admin.token`. Never put tokens in command arguments or shell history.
- The configuration is encrypted into broker state. Delete the plaintext config
  and `admin.json` when no longer needed.
- For a non-loopback listener, TLS is mandatory:
  `--listen <IP>:<PORT> --cert /secure/server.pem --key /secure/server-key.pem`.
- `serve` asks for the password once at start (or uses an unlock session, or
  `LIFEVAULT_PASSWORD` from a service supervisor) and keeps state in memory so
  its worker can rotate and clean up unattended.
- Lost the admin token? Stop the broker and run `lifevault --vault /secure/broker.vault broker admin-token`.

## 3. Enroll an identity

Machine identity, in an owner-only body file:

```json
{"id":"<IDENTITY>","kind":"machine","profiles":["<PROFILE>"],"ttl_seconds":43200}
```

```sh
lifevault broker request https://<broker-host> POST /v1/identities /secure/admin.token /secure/machine.json
```

The response contains the machine token once. Store it in an owner-only file.
Machine and admin tokens last at most 12 hours; renew by enrolling again.

User identities (U2M) bind an OIDC issuer and subject. The issuer is configured
under `issuers` with a dedicated API audience and pinned RSA JWKS; Lifevault
validates access tokens but has no login page. See `docs/broker.md` in the
release archive.

## 4. Fetch credentials

```sh
lifevault broker request https://<broker-host> GET /v1/profiles /secure/machine.token
lifevault broker request https://<broker-host> GET /v1/credentials/<PROFILE> /secure/machine.token
```

The response is JSON with a `lease` object (ID, profile, `expires_at`,
`provider_expires_at`, status, revocation) and a `credentials` object whose keys
are listed in each provider guide. It is printed to stdout on purpose; send it
only to the consumer. `provider_expires_at: null` means the provider sets no
expiry, not an unlimited lease.

A shell consumer could read one response and export the keys without putting
values in argv (this uses `jq`, which Lifevault does not ship):

```sh
resp=$(lifevault broker request https://<broker-host> GET /v1/credentials/<PROFILE> /secure/machine.token)
export GITHUB_TOKEN=$(printf %s "$resp" | jq -r .credentials.GITHUB_TOKEN)
unset resp
```

Consumers must fetch again after rotation. Environment variables and
connection pools are not updated automatically.

## API summary

| Method and path | Purpose |
|---|---|
| `GET /v1/profiles` | Profiles allowed for this identity |
| `GET /v1/credentials/NAME` | Issue and verify, or return the active lease |
| `POST /v1/credentials/NAME/rotate` | Verify a replacement, then switch (5-second cooldown) |
| `GET /v1/leases` | Lease metadata, no secret values |
| `DELETE /v1/leases/ID` | Deny retrieval and request provider cleanup |
| `POST /v1/identities` | Enroll (administrator) |
| `DELETE /v1/identities/ID` | Revoke and clean up owned leases (administrator) |
| `GET /v1/audit` | Secret-free lifecycle history (administrator) |

## Local agent proxy

The agent keeps the upstream identity token out of application configuration.
Applications call `GET /v1/credentials/<PROFILE>` on loopback with a local token.

```sh
umask 077
lifevault broker agent-token > /secure/local-agent.token
lifevault broker agent /secure/agent.json
```

```json
{
  "upstream": "https://<broker-host>",
  "upstream_token_file": "/secure/machine.token",
  "local_token_file": "/secure/local-agent.token",
  "listen": "127.0.0.1:8720",
  "profiles": ["<PROFILE>"]
}
```

Both token files are re-read on each request, so you can replace them without a
restart. The agent does not cache credentials. Token and config files must be
regular, owned by you, mode 600, and not symlinks.

## Changing configuration

| Task | Command (broker stopped) |
|---|---|
| Replace an expired provisioning credential, keep leases | `lifevault --vault /secure/broker.vault broker provider-auth <PROFILE> /secure/auth.json` |
| Replace the whole configuration | `lifevault --vault /secure/broker.vault broker configure /secure/broker-config.json` (needs every lease revoked or expired with cleanup finished) |

The `provider-auth` file may contain only the provisioning credential field for
that provider: `admin_token` (Azure, GCP, GitLab, Cloudflare, Hetzner Storage
Box), `admin_password` (PostgreSQL, MySQL), `private_key_pem` (GitHub),
`credentials` (AWS), or `admin_client_id`/`admin_client_secret` (OVHcloud).

## Operating notes

- Rotation is **issue, verify, switch, revoke**. A failed verification keeps the
  previous lease.
- The worker must keep running for leases without native expiry (AWS IAM,
  MySQL, Hetzner Storage Box) to be deleted on time.
- Monitor `revoke_pending` leases and provisioning-authentication failures.
- HTTP 503 means the broker is busy; retry with backoff. HTTP 429 means an
  identity holds too many concurrent requests.
- `lifevault update` replaces only the CLI. Update the Python adapter from the
  full release package yourself.
- Before production, test issuance, rotation, expiry and revocation against a
  narrowly scoped test account.
