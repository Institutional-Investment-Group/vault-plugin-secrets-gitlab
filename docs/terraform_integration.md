# Terraform Integration
_______________________

Here you can find an example of how to use the Terraform Vault provider to configure the plugin.

### Server Configuration

- The plugin is not part of the official Vault distribution, therefore you need to download it and place it in the
  plugins directory of your Vault server.
- For these examples, lets assume we have the plugin's binary in the directory `/opt/vault/plugins/` under the name
  `vault-plugin-secrets-gitlab_<version>` (e.g. `vault-plugin-secrets-gitlab_v0.2.4`).
- Make sure your Vault server configuration has the plugin directory set correctly:

```hcl
# config.hcl
ui = true

# this setting must be set to the absolute path of the plugin directory
plugin_directory = "/opt/vault/plugins" 

listener "tcp" {
  tls_disable = 1
  address = "[::]:8200"
  cluster_address = "[::]:8201"
}
```

### Input

For the sake of simplicity we will the following variable to store the plugin configuration.

```hcl
# variables.tf
variable "gitlab_secrets_config" {
  description = "Data required to configure GitLab as secrets backend"
  type        = object({
    gitlab_url        = string
    gitlab_init_token = string
    plugin_name       = string
    plugin_version    = string
    plugin_checksum   = string
  })
}
```

### Registration and enabling

Since the official [Vault provider][vault-provider] doesn't offer a resource definition for the `vault plugin register`
command; we will use the `vault_generic_endpoint` resource to perform all operations.

```hcl
# register the plugin
resource "vault_generic_endpoint" "vault_plugin_gitlab_secrets_registration" {
  path = "sys/plugins/catalog/secret/${var.gitlab_secrets_config.plugin_name}"

  data_json = jsonencode({
    command = "${var.gitlab_secrets_config.plugin_name}_${var.gitlab_secrets_config.plugin_version}"
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
  type = var.gitlab_secrets_config.plugin_name

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
# without this even after register a new version of the plugin vault will still have the old version in context
resource "vault_generic_endpoint" "gitlab_secrets_tune" {
  # since we are using output from the vault_mount there is no need to specify dependencies
  path      = "sys/mounts/${vault_mount.gitlab_secrets.path}/tune"
  data_json = jsonencode({
    plugin_version = var.gitlab_secrets_config.plugin_version
  })

  # omitting the following attributes will cause any destroy operation to fail
  disable_delete = true
}

# reload the plugin backend
resource "vault_generic_endpoint" "gitlab_secrets_reload" {
  # wait until the tuning operation is completed
  depends_on = [vault_generic_endpoint.gitlab_secrets_tune]

  path      = "sys/plugins/reload/backend"
  data_json = jsonencode({
    # set either "plugin" or "mounts" to execute the reload, we recommend using "plugin"
    # if "plugin" is set, all mounted paths that use that plugin backend will be reloaded
    plugin = var.gitlab_secrets_config.plugin_name

    # if "mounts" is set, only the specified paths will be reloaded
    # mounts = [vault_mount.gitlab_secrets.path]

    # if omitted, reloads the plugin or mounts on the current Vault instance
    # if 'global', will begin reloading the plugin on all instances of a cluster
    scope = "global"
  })

  # reload is a write operation without response, therefore all these attribute are required to be true
  ignore_absent_fields = true
  disable_read         = true
  disable_delete       = true
}
```

### Configure & Rotate

Here you can decide whether to have the GitLab token used for configuration in the Terraform state or provide it using
a different mechanism. In this example you can put it as input variable and immediately trigger a rotation. After that
only Vault will have a valid token.

```hcl
# configure the backend
resource "vault_generic_endpoint" "gitlab_secrets_config" {
  # wait until the reload operation is completed to make sure the latest version of the plugin is used
  depends_on = [vault_generic_endpoint.gitlab_secrets_reload]

  path = "${vault_mount.gitlab_secrets.path}/config"

  data_json = jsonencode({
    base_url                  = var.gitlab_secrets_config.gitlab_url
    token                     = var.gitlab_secrets_config.gitlab_init_token
    max_ttl                   = 60 * 24 * 7 # 7 days, adjust to your needs
    auto_rotate_token         = true
    revoke_auto_rotated_token = true
  })
}

resource "vault_generic_endpoint" "gitlab_secrets_config_rotate" {
  depends_on = [vault_generic_endpoint.gitlab_secrets_config]

  path = "${vault_mount.gitlab_secrets.path}/config/rotate"

  # this operation doesn't require any input but the attribute 'data_json' is required
  data_json = jsonencode(null)

  # "rotate" is a write operation without response, therefore all these attribute are required to be true
  ignore_absent_fields = true
  disable_delete       = true
  disable_read         = true
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
