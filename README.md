# Howto-vault-cli

This repository serves as a handy reference for using the Vault CLI to perform common operations.

## Install Vault cli

Please read this `https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-install`

### On Mac

Add repository

```ssh
$ brew tap hashicorp/tap
Running `brew update --auto-update`...
==> Auto-updated Homebrew!
Updated 9 taps (trufflesecurity/trufflehog, shivammathur/php, hashicorp/tap, mongodb/brew, caskroom/cask, aws/tap, pulumi/tap, homebrew/core and homebrew/cask).
==> New Formulae
ast-grep                   hashicorp/tap/vlt          yamlfmt
clive                      libabigail
==> New Casks
smooze-pro                               smooze-pro

You have 59 outdated formulae and 7 outdated casks installed.
```

Install

```ssh
$ brew install hashicorp/tap/vault
==> Fetching hashicorp/tap/vault
==> Downloading https://releases.hashicorp.com/vault/1.13.2/vault_1.13.2_darwin_
######################################################################### 100.0%
==> Installing vault from hashicorp/tap
==> Caveats
To restart hashicorp/tap/vault after an upgrade:
  brew services restart hashicorp/tap/vault
Or, if you don't want/need a background service you can just run:
  /usr/local/opt/vault/bin/vault server -dev
==> Summary
🍺  /usr/local/Cellar/vault/1.13.2: 5 files, 194.4MB, built in 10 seconds
==> Running `brew cleanup vault`...
Disable this behaviour by setting HOMEBREW_NO_INSTALL_CLEANUP.
Hide these hints with HOMEBREW_NO_ENV_HINTS (see `man brew`).
```

## Local configuration

In my case profile I set the environment variables VAULT_ADDR and VAULT_TOKEN

```ssh
export VAULT_ADDR="https://vault.domain.com"
export VAULT_TOKEN="hvs.M1T0k3n"
```

Check vault status

```ssh
$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.12.0
Build Date               2022-10-10T18:14:33Z
Storage Type             dynamodb
Cluster Name             vault-cluster-09c4394f
Cluster ID               eaf6ec25-fb1f-4b45-b024-000a05c790e9
HA Enabled               true
HA Cluster               https://vault-1.vault-internal:8201
HA Mode                  active
Active Since             2023-05-19T11:27:23.160101101Z
```

View the secrets engines we have enabled

```ssh
$ vault secrets list
Path           Type         Accessor              Description
----           ----         --------              -----------
airflow/       kv           kv_65789bcb           n/a
cubbyhole/     cubbyhole    cubbyhole_d2048d9e    per-token private secret storage
develop/       kv           kv_2735dc82           n/a
identity/      identity     identity_cff424c2     identity store
production/    kv           kv_5ee7c9ba           n/a
quality/       kv           kv_39444b9c           n/a
secrets/       kv           kv_fa6b08b1           n/a
sys/           system       system_d99a4a2d       system endpoints used for control, policy and debugging
```

## Create a secret engine

Created an engine called devops
```ssh
vault secrets enable -path devops -
version 2 kv
Success! Enabled the kv secrets engine at: devops/
```

### Configure max_version

```ssh
$ vault path-help devops/config
$ vault write devops/config max_version=10
Success! Data written to: devops/config

$ vault read devops/config
```

### Configure ttl

```ssh
$ vault secrets tune -description "DevOps secrets" -default-lease-ttl=30m /devops
Success! Tuned the secrets engine at: devops/

$ vault secrets list -detailed -format=json
```

## How can I access a specific path?

```ssh
$ vault kv get devops/jenkins/credentials
================================== Secret Path ==================================
devops/data/jenkins/credentials

======= Metadata =======
Key                Value
---                -----
created_time       2023-05-09T12:16:36.140182117Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            6

=============== Data ===============
Key                            Value
---                            -----
dockerhub-password             ******
dockerhub-username             ******
jenkins-api-passphrase         ******
jenkins-api-password           ******
jenkins-api-username           ******
```

## How do I add a secret?

```ssh
$ vault kv put devops/jenkins/credentials "test=algo"
================================== Secret Path ==================================
devops/jenkins/credentials

======= Metadata =======
Key                Value
---                -----
created_time       2023-05-19T16:04:57.715297454Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            7
```

## Config Auth Methods

List all authentication methods

```ssh
$
ault auth list
Path        Type       Accessor                 Description                Version
----        ----       --------                 -----------                -------
approle/    approle    auth_approle_d83d0040    n/a                        n/a
github/     github     auth_github_c38c5455     n/a                        n/a
token/      token      auth_token_cc014220      token based credentials    n/a
```

### How Enable userpass method

```ssh
$ vault auth help userpass
$ vault auth enable userpass
Success! Enabled userpass auth method at: userpass/
```

### How create user with userpass

```ssh
$ vault path-help auth/userpass
$ vault write auth/userpass/users/joseph "password=d3m0x4"
Success! Data written to: auth/userpass/users/joseph
```

Login test

```ssh
$ vault login -method=userpass username=joseph
Password (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  hvs.hideFriend
token_accessor         FOeZXBuzIFBtSPwyAS7fulEc
token_duration         768h
token_renewable        true
token_policies         ["default"]
identity_policies      []
policies               ["default"]
token_meta_username    joseph
```

## Policy administration

### View policy list

```ssh
$ vault policy list
default
devops-admin
jenkins-cdktf
jenkins-policy-f31184f
root
```

### Read a policy

```ssh
$ vault policy read jenkins-cdktf
path "app_config/*/*/devops/*/application/*" {
                                capabilities = ["read", "list"]
                            }

                path "secrets/*/*/devops/*/services/test/*/*" {
                    capabilities = ["read", "list"]
                }
```

### Creating a policy

```ssh
$ vault policy write allow_kv kv_access.hcl
Success! Uploaded policy: allow_kv
```

### Attach the policy to user

```ssh
$ vault write auth/userpass/users/joseph "token_policies=allow_kv"
Success! Data written to: auth/userpass/users/joseph
```

## How list the app roles?

```ssh
$ vault list auth/approle/role
Keys
----
jenkins-approle
jenkins-cdk-role-test
jenkins-dev-role
```

```ssh
$ vault read auth/approle/role/jenkins-approle/role-id
Key        Value
---        -----
role_id    0a336a57-1945-772e-5fa3-69db2135080f
```

### How to view app-roles details

```ssh
$ vault read auth/approle/role/jenkins-approle
Key                        Value
---                        -----
bind_secret_id             true
local_secret_ids           false
policies                   [jenkins-policy]
secret_id_bound_cidrs      <nil>
secret_id_num_uses         0
secret_id_ttl              0s
token_bound_cidrs          []
token_explicit_max_ttl     0s
token_max_ttl              0s
token_no_default_policy    false
token_num_uses             0
token_period               0s
token_policies             [jenkins-policy]
token_ttl                  0s
token_type                 defaul
```

### How to view policy details

```ssh
$ vault read sys/policies/acl/jenkins-policy
Key       Value
---       -----
name      jenkins-policy-f31184f
policy    path "/secrets/*" {
        capabilities = ["read", "list"]
    }
    path "app_config/*/*/devops/*/application/*" {
        capabilities = ["read", "list"]
    }
```

## Move secrets

```ssh
$ vault secrets move path1/ path2/
```
