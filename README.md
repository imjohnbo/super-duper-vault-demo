# Super Duper Vault Demo 🙃

Vault and GitHub Actions and OIDC! It's super!

Refer to [Configuring OpenID Connect in HashiCorp Vault - GitHub Docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-hashicorp-vault) for more fun.

## Setup

### Configure Vault

Start Vault:
```
vault server -dev
```

Enable JWT auth mechanism:
```
vault auth enable jwt
```

Configure jwt auth with oidc discovery URL:
```
vault write auth/jwt/config \
	oidc_discovery_url="https://token.actions.githubusercontent.com" \
	bound_issuer="https://token.actions.githubusercontent.com" \
	default_role="github-action"
```

Edit `ci` policy to allow access to CI:
```hcl
# ci-policy.hcl
path "secret/data/ci" {
  capabilities = [ "read" ]
}
```

Write the policy:
```
vault policy write ci ci-policy.hcl
```

Create a role for jwt authentication:
```
vault write auth/jwt/role/github-action \
	bound_subject="repo:imjohnbo/super-duper-vault-demo:ref:refs/heads/main" \
	bound_audiences="https://github.com/imjohnbo" \
	user_claim="sub" \
	policies="ci" \
	ttl=10m \
	role_type="jwt"
```
Note: ttl defines the validity of client_token.Change this if longer validity for token is needed.

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

```yml
on:
  workflow_dispatch:

name: Retrieve Vault Secret

jobs:
  build:
    runs-on: self-hosted
    permissions:
      id-token: write
      contents: read

    steps:

    # Use official HashiCorp Vault action, directing it to retrieve the `npmToken` secret from
    # the local endpoint using the role configured previously.
    - uses: hashicorp/vault-action@v2.4.0
      with:
        url: 'http://127.0.0.1:8200'
        method: jwt
        role: github-action
        secrets: secret/data/ci npmToken

    # Use the secret. By default, the secret is written to an 
    # environment variable with the same name as the secret. 
    # https://github.com/hashicorp/vault-action#set-output-variable-name
```
