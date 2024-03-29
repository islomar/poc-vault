# Vault

* https://hub.docker.com/_/vault 
* https://www.hashicorp.com/blog/enabling-cloud-based-auto-unseal-in-vault-open-source 


## Tutorials
* https://learn.hashicorp.com/vault/getting-started/first-secret 
* https://www.katacoda.com/search?q=vault&hPP=12&idx=scenarios&p=0&is_v=1
  - https://www.katacoda.com/courses/docker-production/vault-secrets 


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
* `vault path-help aws/creds/my-non-existent-role`

### Authentication
* Vault has pluggable auth methods.
* Authentication is the process by which user or machine-supplied information is verified and converted into a Vault token with matching policies attached. 
* Authentication is simply the process by which a user or machine gets a Vault token.
* When you start a dev server with vault server -dev, it prints your root token. The root token is the initial access token to configure Vault. It has root privileges, so it can perform any operation within Vault.
* You can create more tokens: `vault token create`
  - this will create a child token of your current token that inherits all the same policies.
  - tokens always have a parent, and when that parent token is revoked, children can also be revoked all in one operation.
* To authenticate with a token: `vault login <your_token>`
* After a token is created, you can revoke it: `vault token revoke <your_token>`
* Log back with root token: `vault login $VAULT_DEV_ROOT_TOKEN_ID`
* In practice, operators should not use the `token create` command to generate Vault tokens for users or machines. Instead, those users or machines should authenticate to Vault using any of Vault's configured auth methods such as GitHub, LDAP, AppRole, etc.
* Auth methods: Vault supports many auth methods, but they must be enabled before use. 
  - E.g. enable GitHub auth method: `vault auth enable github`
* Unlike secrets engines which are enabled at the root router, auth methods are always prefixed with `auth/` in their path. So the GitHub auth method we just enabled is accessible at `auth/github`.
* `vault auth list`
* You can ask for help information about any CLI auth method, even if it is not enabled: `vault auth help github`
* Authenticate to GitHub: `vault login -method=github`
* You can revoke any logins from an auth method using: `vault token revoke -mode path auth/github`
* Disable GitHub auth method: `vault auth disable github`

### Authorization
* Policies in Vault control what a user can access.
  - The `default` policy provides a common set of permissions and is included on all tokens by default. 
  - The `root` policy gives a token super admin permissions, similar to a root user on a linux machine.
* To format a policy file: `vault policy fmt my-policy.hcl`
* To write a policy: `vault policy write my-policy my-policy.hcl`
* `vault policy list`
* To view the contents of a policy: `vault policy read my-policy`
* Create a token and assign it to a policy: `vault token create -policy=my-policy`
* Use the `vault path-help` system with your auth method to determine how the mapping is done since it is specific to each auth method. For example, with GitHub, it is done per team using the `map/teams/<team>` path:
* Auth methods all must map to the central policy system.

### Configuring Vault
* Install Consul and start an instance: `consul agent -dev`
* `vault server -config=config.hcl`
  - It might show a warning: https://www.vaultproject.io/docs/configuration/index.html#disable_mlock
* `vault operator init`: Initializing the Vault: Initialization is the process configuring the Vault. This only happens once when the server is started against a new backend that has never been used with Vault before. When running in HA mode, this happens once per cluster, not per server.
  - It returns 5 Unseal keys and an initial root token.
  - In a real deployment scenario, you would never save these keys together. Instead, you would likely use [Vault's PGP and Keybase.io support](https://www.vaultproject.io/docs/concepts/pgp-gpg-keybase.html) to encrypt each of these keys with the users' PGP keys. This prevents one single person from having all the unseal keys. 
* Every initialized Vault server starts in the sealed state. From the configuration, Vault can access the physical storage, but it can't read any of it because it doesn't know how to decrypt it. The process of teaching Vault how to decrypt the data is known as unsealing the Vault.
* Unsealing has to happen every time Vault starts.
* Vault uses an algorithm known as [Shamir's Secret Sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing) to split the master key into shards. 
* Unseal the Vault: `vault operator unseal`
* As a root user, you can reseal the Vault with `vault operator seal`
  - When the Vault is sealed again, it clears all of its state (including the encryption key) from memory. The Vault is secure and locked down from access.

### Using the HTTP APIs with Authentication
* Check initialization: `curl http://127.0.0.1:8200/v1/sys/init`
* Machines that need access to information stored in Vault will most likely access Vault via its REST API. The application would first authenticate to Vault which would return a Vault API token. The application would use that token for future communication with Vault.
* Initialize Vault:
  ```
    curl \
      --request POST \
      --data '{"secret_shares": 1, "secret_threshold": 1}' \
      http://127.0.0.1:8200/v1/sys/init | jq
    ```
   - This response contains our initial root token. It also includes the unseal key (aka *recovery key*). You can use the unseal key to unseal the Vault and use the root token perform other requests in Vault that require authentication.
* Unseal the vault:
  ```
  curl \
      --request POST \
      --data '{"key": "<UNSEAL_KEY>"}' \
      http://127.0.0.1:8200/v1/sys/unseal | jq
  ```
  - Now any of the available auth methods can be enabled and configured.
* The `approle` auth method allows machines or apps to authenticate with Vault-defined roles.
  - Enable the `approle` auth method.
  - Create a role `my-role` and get its `id`.
  - Create a secret_id under `my-role`
  - fetch a new Vault token providing the role id and the secret_id


## Seal/unseal
* https://www.vaultproject.io/docs/concepts/seal.html
* Unsealing is the process of constructing the master key necessary to decrypt the data encryption key.
* Vault never stores the master key, therefore, the only way to retrieve the master key is to have a quorum of unseal keys re-generate it.
* [Unseal key vs Recovery key](https://www.vaultproject.io/docs/enterprise/hsm/behavior.html#key-split-between-unseal-keys-and-recovery-keys)\
  - Unseal key = Master key
  - Recovery key = created when using HSM.
  - But in some places, unseal key = recovery keys: https://www.hashicorp.com/blog/enabling-cloud-based-auto-unseal-in-vault-open-source
* https://github.com/hashicorp/vault-guides/tree/master/operations/aws-kms-unseal/terraform-aws
* The data stored by Vault is stored encrypted. Vault needs the encryption key in order to decrypt the data. **The encryption key is also stored with the data**, but encrypted with another encryption key known as the **master key**. The master key isn't stored anywhere.
* Instead of distributing this master key as a single key to an operator, Vault uses an algorithm known as Shamir's Secret Sharing to split the key into shards. A certain threshold of shards is required to reconstruct the master key.

### Auto-unseal with AWS KMS
* https://learn.hashicorp.com/vault/day-one/ops-autounseal-aws-kms
* https://www.vaultproject.io/docs/concepts/seal.html#auto-unseal
* Auto Unseal was developed to aid in reducing the operational complexity of keeping the master key secure. This feature delegates the responsibility of securing the master key from users to a trusted device or service. Instead of only constructing the key in memory, the master key is encrypted with one of these services or devices and then stored in the storage backend allowing Vault to decrypt the master key at startup and unseal automatically.
* Configure `seal "awskms"` in the configuration file `/etc/vault.d/vault.hcl`.

### Seal migration
* https://www.vaultproject.io/docs/concepts/seal.html#seal-migration
* The seal can be migrated from Shamir keys to Auto Unseal and vice versa.
* To migrate from Shamir keys to Auto Unseal, take your server cluster offline and update the seal configuration with the appropriate seal configuration. 
* When you bring your server back up, run the unseal process with the -migrate flag. All unseal commands must specify the -migrate flag. Once the required threshold of unseal keys are entered, the unseal keys will be migrated to recovery keys.
* `vault operator unseal -migrate`
* To migrate from Auto Unseal to Shamir keys, take your server cluster offline and update the seal configuration and add `disabled = "true"` to the seal block. 


### Web UI
* http://127.0.0.1:8200/ui 


## Operations and Development Tracks
https://learn.hashicorp.com/vault#operations-and-development




## Useful info
* Restart Vault: `sudo systemctl restart vault`
* Vault configuration: `/etc/vault.d/vault.hcl`
