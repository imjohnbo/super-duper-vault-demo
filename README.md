# Super Duper Vault Demo 🙃

Vault and GitHub Actions and OIDC! It's super!

Refer to [Configuring OpenID Connect in HashiCorp Vault - GitHub Docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-hashicorp-vault) for more fun.

## Setup

### Configure Vault

This demo uses the [open source, self hosted version of Vault](https://www.vaultproject.io/downloads). Vault Enterprise could be used with slight modifications.

Start Vault locally:
```
vault server -dev
```

Enable JWT auth mechanism:
```
vault auth enable jwt
```

Configure JWT auth with OIDC discovery URL. Note: `default_role` must match the name of the role created later.
```
vault write auth/jwt/config \
	oidc_discovery_url="https://token.actions.githubusercontent.com" \
	bound_issuer="https://token.actions.githubusercontent.com" \
	default_role="github-action"
```

Make a policy to allow read access to the path where the secret will be kept. Note: in the [KV2](https://www.vaultproject.io/docs/secrets/kv/kv-v2) engine, [`data` is used](https://www.vaultproject.io/docs/secrets/kv/kv-v2#acl-rules) in the path.
```hcl
# ci-policy.hcl
path "secret/data/ci" {
  capabilities = [ "read" ]
}
```

Write the policy to Vault:
```
vault policy write ci ci-policy.hcl
```

Create a role for JWT authentication. Note: the `bound_subject` and `bound_audiences` are configurable according to your security preferences. Read more in the [GitHub](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-hashicorp-vault#overview) and [HashiCorp](https://www.vaultproject.io/docs/auth/jwt#oidc-configuration-troubleshooting) documentation.
```
vault write auth/jwt/role/github-action \
	bound_subject="repo:imjohnbo/super-duper-vault-demo:ref:refs/heads/main" \
	bound_audiences="https://github.com/imjohnbo" \
	user_claim="sub" \
	policies="ci" \
	ttl=10m \
	role_type="jwt"
```

Add secret:
```
vault kv put secret/ci npmToken=imjohnbo
```

### Configure GitHub Actions self hosted runner

Configure and start a [self hosted runner](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners):
```bash
# ...configuration according to instructions...

./run.sh
```

### Retrive the secret from an Actions workflow

See `.github/workflows/vault.yml` for a working example.
