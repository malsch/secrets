# secrets

A (typically private) folder for central, [SOPS](https://github.com/getsops/sops)-encrypted files with secrets shared across projects. Encrypted to post-quantum [age](https://github.com/FiloSottile/age) keys (`age1pq1…`, post-quantum
hybrid ML-KEM-768 + X25519 key pair). Only recipients who have at least one of the private keys can use it to decrypt the secrets.

This is a local setup, and cloud options have not been systematically explored. Key management services (KMS) in the cloud would likely be even safer. SOPS supports KMS-backed keys as well.

Compared with a potentially more standard approach using the unix password manager [pass](https://www.passwordstore.org/), which encrypts one file
per secret with GPG, this setup avoids GPG altogether: GPG is a general-purpose
identity and trust system with a history dating back to 1991, when PGP was first published. Given current developments in quantum computing, it feels better to have a small quantum-resistant program that aims to do just one thing well: asymetric encryption. More on the [PGP vs age](https://words.filippo.io/age-authentication/) debate, by the age developer ...

## Files

| File               | Scope                                          | Recipients             |
|--------------------|------------------------------------------------|------------------------|
| `dev.enc.yaml`     | development (**default**)                      | local_key + backup_key |
| `staging.enc.yaml` | staging                                        | local_key + backup_key |
| `prod.enc.yaml`    | production                                     | prod_key + backup_key  |

(This is configured via `.sops.yaml`.)

Layout: **flat, env-var-style keys** (mise and `sops exec-env` reject
nested maps). Same names across environments, different values:

```yaml
DB_URL: postgres://...
DB_URL__RETIRES: "2027-03-31"     # optional, stays plaintext
GITHUB_TOKEN: ghp_...
GITHUB_TOKEN__RETIRES: "2027-01-15"
```

## Retirement dates

- **Secrets:** add `<NAME>__RETIRES: "YYYY-MM-DD"` next to the secret. It is
  left unencrypted (`unencrypted_regex`) but covered by the sops MAC, so
  editing it outside `sops edit` breaks decryption. Note: mise also exports these as env vars (harmless).
- The script `bin/retiring [days]`
  lists overdue/soon-due entries without decrypting (exit 1 if any; default
  30 days).

## Installation

```bash
mise use -g sops age
```

More on using [SOPS with mise](https://mise.jdx.dev/environments/secrets/sops.html) (though we use a slighthly different setup).

## First-time setup

1. `local_key`: `age-keygen -pq -o ~/.config/sops/age/keys.txt && chmod 600 ~/.config/sops/age/keys.txt`
2. If additional keys are needed: Generate `prod_key` (or reuse `local_key`) and `backup_key` offline. To use
   `prod_key` locally, append its identity to `keys.txt` (one file can hold several).
3. Put the public keys in `.sops.yaml` and note where each private key lives (replace the placeholders).
4. `sops edit dev.enc.yaml` (and `staging`/`prod`) creates each file encrypted.
5. On a fresh clone: `git config core.hooksPath hooks`

### mise

Needed only for loading whole files with `_.file` (see below); the
single-value `exec(...)` form works without it.

In `~/.config/mise/config.toml`:

```toml
[settings]
sops.rops = false   # mise's built-in decryptor (rops) can't read PQ age keys; use the sops CLI
```

### Keep AI coding agents out (Claude Code)

Deny the agent the private keys and the sops CLI in `~/.claude/settings.json`
(user-level, applies to every project):

```json
{
  "permissions": {
    "deny": [
      "Read(~/.config/age/**)",
      "Read(~/.config/sops/age/**)",
      "Bash(sops:*)"
    ]
  }
}
```

Limits: `Read(...)` rules cover Claude's file tools, not every shell command
(e.g. `cat` on the key, `age -d`, or `mise env` in a project that loads a
`*.enc.yaml`). For an OS-level block, enable Claude Code's sandbox
(`"sandbox": {"enabled": true}`); `Read(...)` deny rules then also apply to
sandboxed commands.

## Edit an encrypted file

(or add a new file)

```shell
sops edit <filename>
```

## Use from a project

**mise, single value** (only this variable, not the whole file):

```toml
[env]
OPENAI_API_KEY = '''{{ exec(command=`sops decrypt --extract '["OPENAI_API_KEY"]' ~/secrets/dev.enc.yaml`) }}'''
```

**mise** (`mise.toml` in the project; needs the one-time [mise setup](#mise)):

```toml
# mise.toml (default = dev)
[env]
_.file = "~/secrets/dev.enc.yaml"
```

```toml
# mise.prod.toml, active with MISE_ENV=prod (same pattern for staging)
[env]
_.file = "~/secrets/prod.enc.yaml"
```

**Single value** (direnv `.envrc`, scripts):

```bash
export OPENAI_API_KEY="$(sops decrypt --extract '["OPENAI_API_KEY"]' ~/secrets/dev.enc.yaml)"
```

**One command with all values in its environment:**

```bash
sops exec-env ~/secrets/dev.enc.yaml 'uv run python main.py'
```

## Maintenance

- Change recipients: edit `.sops.yaml`, then `sops updatekeys <file>`
- Rotate data key: `sops rotate -i <file>` (rotate the real credentials at the provider too)
- Guard: `hooks/pre-commit` rejects unencrypted `*.enc.*` files and age private keys.

## For the future

- It would be nice to encrypt to SSH public keys. `age` supports `ssh-rsa` and `ssh-ed25519` keys, but neither is quantum-resistant. OpenSSH's new post-quantum keys (`ssh-mldsa44-ed25519`) are signature-only, so they can't be used for encryption; that would need a key type based on ML-KEM, which SSH doesn't have yet.
- It may be worth checking back on the cryptographic right answers blog post series ([here](https://www.latacora.com/blog/post-quantum-cryptographic-right-answers/) or [here](https://words.filippo.io/right-answers/)) for latest developments.
