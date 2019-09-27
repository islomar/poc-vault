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
  * The path prefix (e.g. `secret/`) tells Vault which secrets engine to which it should route traffic. By default, Vault enables a secrets engine called `kv` at the path `secret/`. Other possible secrets engines are `aws` or `database`.
* Enable a secrets engine in another path: `vault secrets enable -path=kv kv`
* List all the existing paths: `vault secrets list`
* List all the keys for the path `kv`: `vault list kv`
