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
  - It shows the unseal key and root token.
  - Save the unseal key.
  - `export VAULT_DEV_ROOT_TOKEN_ID="<your_root_token>"`
  - `export VAULT_ADDR='http://127.0.0.1:8200'`
  - Verify it is up and running: `vault status`
* TBD
