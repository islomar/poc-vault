# Vault

* https://hub.docker.com/_/vault 
* https://www.hashicorp.com/blog/enabling-cloud-based-auto-unseal-in-vault-open-source 

## Tutorials
* https://learn.hashicorp.com/vault/getting-started/first-secret 
* https://www.katacoda.com/courses/docker-production/vault-secrets 


## Getting started (official Vault tutorial)
* Install Vault
* `vault -autocomplete-install && exec $SHELL`
* Start the server: `vault server -dev`
  - The dev server stores all its data in-memory (but still encrypted) and starts unsealed with a single unseal key. The root token is already authenticated to the CLI.
  - It shows the unseal key and root token.
  - Save the unseal key.
  - `export VAULT_DEV_ROOT_TOKEN_ID="<your_root_token>"`
  - `export VAULT_ADDR='http://127.0.0.1:8200'`
  - Verify it is up and running: `vault status`
* CRUD secrets:
  * Write a secret: `vault kv put secret/hello foo=world`
  * Read a secret: `vault kv get secret/hello`
  * To print only the value of a given field: 
    - `vault kv get -field=excited secret/hello`
    - `vault kv get -format=json secret/hello | jq -r .data.data.excited`
  * Delete a secret: `vault kv delete secret/hello`

### Secrets engine  
* The path prefix (e.g. `secret/`) tells Vault which secrets engine to which it should route traffic. By default, Vault enables a secrets engine called `kv` at the path `secret/`. Other possible secrets engines are `aws` or `database`.
* Enable a secrets engine in another path: `vault secrets enable -path=kv kv`
* List all the existing paths: `vault secrets list`
* List all the keys for the path `kv`: `vault list kv`
* Disable a secrets engine: `vault secrets disable kv/`. When a secrets engine is disabled, all secrets are revoked and the corresponding Vault data and configuration is removed.

### Dynamic secrets
* https://learn.hashicorp.com/vault/getting-started/dynamic-secrets
* dynamic secrets are generated when they are accessed. Dynamic secrets do not exist until they are read, so there is no risk of someone stealing them or another client using the same secrets.
* Enable `aws`: `vault secrets enable -path=aws aws`. The AWS secrets engine generates dynamic, on-demand AWS access credentials.
* `vault write aws/config/root access_key=<your_access_key> secret_key=<your_secret_key> region=<YOUR_REGION>`
*  A role in Vault is a human-friendly identifier to an action.
  - File `create_iam_role.sh`: When I ask for a credential for "my-role", create it and attach the IAM policy `{ "Version": "2012..." }`.
* Generate an access key pair for that role: `vault read aws/creds/my-role`
  - Each time you read from `aws/creds/:name`, Vault will connect to AWS and generate a new IAM user and key pair.
  - Take careful note of the `lease_id` field in the output. This value is used for renewal, revocation, and inspection. Copy this lease_id to your clipboard. Note that the lease_id is the full path, not just the UUID at the end.
  - Revoke the secret: `vault lease revoke <lease_id>` , it will remove the IAM user.

### Built-in Help
* `vault path-help aws`
