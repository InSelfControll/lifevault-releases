# Provider and vendor guides

Each guide covers one provider or tool: what Lifevault does with it, what you
need first, how to create a narrowly scoped credential, the exact commands, and
how updates and failures behave.

Guides use four kinds of integration:

- **Store credentials**: `import-provider` saves an existing credential bundle
  into the local vault. `import-provider PROVIDER --connect` opens the
  provider's token page, asks for the values and checks the token where
  supported.
- **Push secrets**: `target connect` (guided) or `target add` (scripted) mirror
  your projects, one way, into another secret manager. Bitwarden/Vaultwarden,
  HashiCorp Vault and Ansible Vault need nothing installed; 1Password needs `op`.
- **Import from**: copy secrets from another tool or file into the vault.
- **Broker dynamic credentials**: the optional standalone broker issues
  short-lived credentials from a provider adapter. Start with
  [broker basics](broker-basics.md).

| Guide | Used for | Summary |
|---|---|---|
| [Bitwarden / Vaultwarden](bitwarden.md) | Push secrets | Built-in client; guided `target connect bitwarden`, private collections with `--create-collection`, or scripted `target add --type bitwarden`. |
| [1Password](1password.md) | Push secrets | Push projects to one vault with a service account; guided `target connect onepassword`. Needs `op`. |
| [HashiCorp Vault](hashicorp-vault.md) | Push secrets, import from | Built-in client; push projects to KV v1/v2 with `target connect hashicorp` (OIDC or token); import KV fields with `import-hashicorp`. |
| [Ansible Vault](ansible-vault.md) | Push secrets, import from | Built-in client; write encrypted `project.yml` files; import fields with `import-ansible`. |
| [.env files](env-files.md) | Import from | Import `.env` files as projects; run apps with `run --project`. |
| [SSH keys](ssh.md) | Import from | Store private keys; load them into `ssh-agent` for one hour. |
| [Hetzner](hetzner.md) | Store credentials, broker dynamic credentials | Cloud API token (`--connect` verifies it) and Storage Box bundles; broker-managed read-only Storage Box subaccounts. |
| [OVHcloud](ovhcloud.md) | Store credentials, broker dynamic credentials | Legacy API keys (`ovh`) and OAuth (`ovh-oauth`) bundles; broker-managed OAuth service accounts. |
| [DigitalOcean](digitalocean.md) | Store credentials | Store a `DIGITALOCEAN_TOKEN`; `--connect` verifies it. |
| [Linode (Akamai)](linode.md) | Store credentials | Store a `LINODE_TOKEN`; `--connect` verifies it. |
| [Vultr](vultr.md) | Store credentials | Store a `VULTR_API_KEY`; `--connect` verifies it. |
| [Scaleway](scaleway.md) | Store credentials | Store an access/secret key pair with optional project, region and zone; `--connect` verifies it. |
| [Broker basics](broker-basics.md) | Broker dynamic credentials | Start the broker, enroll identities, fetch credentials, run the local agent. |
| [AWS](aws.md) | Broker dynamic credentials | STS AssumeRole sessions (`aws-sts`) or temporary IAM users (`aws-iam`). |
| [Azure](azure.md) | Broker dynamic credentials | Expiring passwords on an existing Entra application. |
| [Google Cloud](gcp.md) | Broker dynamic credentials | Short-lived service-account access tokens. |
| [PostgreSQL](postgresql.md) | Broker dynamic credentials | Temporary login roles that inherit one configured role. |
| [MySQL](mysql.md) | Broker dynamic credentials | Temporary TLS-only users with one default role. |
| [GitHub](github.md) | Broker dynamic credentials | GitHub App installation tokens with explicit permissions. |
| [GitLab](gitlab.md) | Broker dynamic credentials | Project access tokens with fixed scopes and access level. |
| [Cloudflare](cloudflare.md) | Broker dynamic credentials | Expiring API tokens with explicit policies. |

Run `lifevault --help`, `lifevault import-provider --list`, `lifevault target --help`,
`lifevault target connect --help` and `lifevault broker --help` for the built-in
reference. The full broker reference (`docs/broker.md` and
`docs/provider-capabilities.md`) ships inside each release archive.

## Conventions in these guides

- `<PLACEHOLDER>` marks a value you supply. Never paste real secrets into a
  command line; enter them at hidden prompts.
- To put a value into your environment without echoing it or saving it in shell
  history:

  ```sh
  read -rs HCLOUD_TOKEN && export HCLOUD_TOKEN   # paste, then press Enter
  ```

  Run `unset <NAME>` once Lifevault has stored it.
- Stored names that contain `__` belong to a project: `MY_APP__API_KEY` is key
  `API_KEY` of project `MY_APP`. Project secrets are pushed to any push target
  that covers the project.
