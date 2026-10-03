# SSH keys

Lifevault stores SSH private keys in the encrypted vault and loads them into your
`ssh-agent` for one hour, without writing a plaintext key file.

## Prerequisites

- OpenSSH `ssh-add` on `PATH`.
- A running agent reachable through `SSH_AUTH_SOCK`.
- The private-key file must be a regular, owner-only file (`chmod 600`).

## Store a key

```sh
lifevault import-ssh DEPLOY_SSH_KEY ~/.ssh/id_ed25519
lifevault import-ssh DEPLOY_SSH_KEY ~/.ssh/id_ed25519 --replace   # overwrite intentionally
```

- The file must carry OpenSSH or PEM private-key markers. Public-key files are
  rejected. Lifevault checks the envelope only; OpenSSH validates the key when it
  loads it.
- The source file is left untouched. Delete or keep it as you see fit.
- Passphrase-protected keys keep their passphrase.

## Load it into the agent

```sh
eval "$(ssh-agent -s)"            # only if no agent is running
lifevault ssh-add DEPLOY_SSH_KEY
ssh <user>@<host>
```

- The key is piped to OpenSSH `ssh-add` with a one-hour lifetime. No temporary
  key file is written, and the Lifevault master password is removed from the
  child's environment.
- A key passphrase can still prompt through your terminal or askpass.
- `ssh-add` does not open an SSH session or remove other agent identities.
- `ssh-add -T <key>.pub` can check that the agent signs with the key without
  connecting anywhere.

## Notes

- A stored key is an ordinary multiline secret. `lifevault get DEPLOY_SSH_KEY`
  prints it; avoid that unless you mean to.
- A key stored as `PROJECT__NAME` belongs to that project and is pushed to its
  push targets. Use a name without `__` to keep it local.
