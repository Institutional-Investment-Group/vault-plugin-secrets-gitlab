# Terraform Integration
_______________________

Here you can find an example of how to use the Terraform Vault provider to configure the plugin.

### Input

For the sake of simplicity we will the following variable to store the plugin configuration.

```hcl
# variables.tf
variable "gitlab_secrets_config" {
  description = "Data required to configure GitLab as secrets backend"
  type        = object({
    gitlab_url        = string
    gitlab_init_token = string
    plugin_version    = string
    plugin_checksum   = string
    plugin_command    = string
  })
}
```

### Registration and enabling

Since the official [Vault provider][vault-provider] doesn't offer a resource definition for the `vault plugin register` 
command; we will use the `vault_generic_endpoint` resource to perform all operations.

```hcl
# register the plugin
resource "vault_generic_endpoint" "vault_plugin_gitlab_secrets_registration" {
  path = "sys/plugins/catalog/secret/${var.gitlab_secrets_config.plugin_command}_${var.gitlab_secrets_config.plugin_version}"

  data_json = jsonencode({
    command = var.gitlab_secrets_config.plugin_command
    sha256  = var.gitlab_secrets_config.plugin_checksum
    version = var.gitlab_secrets_config.plugin_version
  })

  ignore_absent_fields = true
  # omitting the following attributes will cause the apply to fail
  disable_delete       = true
  disable_read         = true
}

# mount the gitlab secrets backend
resource "vault_mount" "gitlab_secrets" {
  # explicit define dependency to control the execution order 
  depends_on = [vault_generic_endpoint.vault_plugin_gitlab_secrets]

  path = "gitlab"
  type = "gitlab"

  # feel free to change with your desired values
  options = {
    default_lease_ttl_seconds = 60 * 24     # 24 hours
    max_lease_ttl_seconds     = 60 * 24 * 7 # 7 days
  }
}
```

### Tune & Reload

This operation is vital to ensure a proper version upgrade of the plugin, you can read in detail about it in the 
[official documentation][vault-plugin-upgrade].

```hcl
# this operation will make sure to switch to a new version of the plugin
resource "vault_generic_endpoint" "gitlab_secrets_tune" {
  # since we are using output from the vault_mount there is no need to specify dependencies
  path      = "sys/mounts/${vault_mount.gitlab_secrets.path}/tune"
  data_json = jsonencode({
    plugin_version = var.gitlab_secrets_config.plugin_version
  })
}

# reload the plugin backend
resource "vault_generic_endpoint" "gitlab_secrets_reload" {
  # explicit define dependency to make sure both operations are executed upfront
  depends_on = [vault_mount.gitlab_secrets, vault_generic_endpoint.gitlab_secrets_tune]
  
  path      = "sys/plugins/reload/backend"
  data_json = jsonencode({
    mount = [vault_mount.gitlab_secrets.path]
    
    # if omitted, reloads the plugin or mounts on the current Vault instance
    # if 'global', will begin reloading the plugin on all instances of a cluster
    scope = "global"
  })
}
```

### Configure & Rotate

Here you can decide whether to have the GitLab token used for configuration in the Terraform state or provide it using
a different mechanism. In this example you can put it as input variable and immediately trigger a rotation. After that
only Vault will have a valid token.

```hcl
# configure the backend
resource "vault_generic_endpoint" "gitlab_secrets_config" {
  path = "${vault_mount.gitlab_secrets.path}/config"

  data_json = jsonencode({
    base_url                  = var.gitlab_secrets_config.gitlab_url
    token                     = var.gitlab_secrets_config.gitlab_init_token
    max_ttl                   = 60 * 24 * 7 # 7 days
    auto_rotate_token         = true
    revoke_auto_rotated_token = true
  })

  disable_delete       = true
  ignore_absent_fields = true
}

resource "vault_generic_endpoint" "gitlab_secrets_config_rotate" {
  depends_on = [vault_generic_endpoint.gitlab_secrets_config]

  path = "${vault_mount.gitlab_secrets.path}/config/rotate"
  
  # this operation doesn't require any input but the attribute 'data_json' is required
  data_json = jsonencode(null)

  # omitting the following attributes will cause the apply to fail
  disable_delete       = true
  ignore_absent_fields = true
}
```

### Roles

Here is just an example of how to create a role for a project token.

```hcl
resource "vault_generic_endpoint" "gitlab_secrets_project_guest" {
  path      = "${vault_mount.gitlab_secrets.path}/roles/project"
  data_json = jsonencode({
    access_level = "guest"
    name         = "project-token-name"
    path         = "my_group/my_project"
    scopes       = "read_api"
    token_ttl    = "24h"
    token_type   = "project"
  })
}
```

[vault-provider]: https://registry.terraform.io/providers/hashicorp/vault
[vault-plugin-upgrade]: https://developer.hashicorp.com/vault/docs/upgrading/plugins
