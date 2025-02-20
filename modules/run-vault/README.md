# Vault Run Script

This folder contains a script for configuring and running Vault on an [Azure](https://azure.microsoft.com/) server. This script has been tested on Ubuntu 16.04. There is a good chance it will work on other flavors of Debian as well.

## Quick start

This script assumes you installed it, plus all of its dependencies (including Vault itself), using the [install-vault module](./modules/install-vault). The default install path is `/opt/vault/bin`, so to start Vault in server mode, you run:

```shell
/opt/vault/bin/run-vault -azure-account-name my-account-name --azure-account-key [REDACTED] --azure-container "Vault" --tls-cert-file /opt/vault/tls/vault.crt.pem --tls-key-file /opt/vault/tls/vault.key.pem
```

This will:

1. Generate a Vault configuration file called `default.hcl` in the Vault config dir (default: `/opt/vault/config`).
   See [Vault configuration](#vault-configuration) for details on what this configuration file will contain and how
   to override it with your own configuration.

1. Generate a [Supervisor](http://supervisord.org/) configuration file called `run-vault.conf` in the Supervisor
   config dir (default: `/etc/supervisor/conf.d`) with a command that will run Vault:  
   `vault server -config=/opt/vault/config`.

1. Tell Supervisor to load the new configuration file, thereby starting Vault.

We recommend using the `run-vault` command as part of [Custom Data](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/classic/inject-custom-data), so that it executes
when the Azure Instance is first booting. After running `run-vault` on that initial boot, the `supervisord` configuration will automatically restart Vault if it crashes or the Azure instance reboots.

See the [main example](./MAIN.md) for fully-working sample code.

## Command line Arguments

The `run-vault` script accepts the following arguments:

* `--azure-account-name` (required): Specifies the Azure storage account to use to store Vault data.
* `--azure-account-key` (required): Specifies the Azure account key for the `--azure-account-name`.
* `--azure-container` (required): Specifies the Azure container to use to store Vault data.
* `--tls-cert-file` (required): Specifies the path to the certificate for TLS. To configure the listener to use a CA certificate, concatenate the primary certificate and the CA certificate together. The primary certificate should appear first in the combined file. See [How do you handle encryption?](#how-do-you_handle-encryption) for more info.
* `--tls-key-file` (required): Specifies the path to the private key for the certificate. See [How do you handle encryption?](#how-do-you_handle-encryption) for more info.
* `--port` (optional): The port Vault should listen on. Default is `8200`.
* `--log-level` (optional): The log verbosity to use with Vault. Default is `info`.
* `--cluster-port` (optional): The port Vault should listen on for server-to-server communication. Default is `--port + 1`.
* `config-dir` (optional): The path to the Vault config folder. Default is to take the absolute path of `../config`, relative to the `run-vault` script itself.
* `user` (optional): The user to run Vault as. Default is to use the owner of `config-dir`.
* `skip-vault-config`: If this flag is set, don't generate a Vault configuration file. This is useful if you have
  a custom configuration file and don't want to use any of of the default settings from `run-vault`.

Example:

```shell
/opt/vault/bin/run-vault --azure-account-name my-account-name --azure-account-key [REDACTED] --azure-container my-container-name --tls-cert-file /opt/vault/tls/vault.crt.pem --tls-key-file /opt/vault/tls/vault.key.pem
```

## Vault configuration

`run-vault` generates a configuration file for Vault called `default.hcl` that tries to figure out reasonable defaults for a Vault cluster in Azure. Check out the [Vault  Configuration Files documentation](https://www.vaultproject.io/docs/configuration/index.html) for what configuration settings are available.
  
### Default configuration

`run-vault` sets the following configuration values by default:

* [storage](https://www.vaultproject.io/docs/configuration/index.html#storage): Configure Azure Container as the storage backend
  with the following settings:
  * [accountName](https://www.vaultproject.io/docs/configuration/storage/azure.html#accountname): Set to the `--azure-account-name` parameter.
  * [accountKey](https://www.vaultproject.io/docs/configuration/storage/azure.html#accountkey): Set to the `--azure-account-key` parameter.
  * [container](https://www.vaultproject.io/docs/configuration/storage/azure.html#container): Set to the `--azure-container` parameter.

* [ha_storage](https://www.vaultproject.io/docs/configuration/index.html#ha_storage): Configure Consul as the [high availability](https://www.vaultproject.io/docs/concepts/ha.html) storage backend with the following settings:
  * [address](https://www.vaultproject.io/docs/configuration/storage/consul.html#address): Set the address to `127.0.0.1:8500`. This is based on the assumption that the Consul agent is running on the same server.
  * [scheme](https://www.vaultproject.io/docs/configuration/storage/consul.html#scheme): Set to `http` since our connection is to a Consul agent running on the same server.
  * [path](https://www.vaultproject.io/docs/configuration/storage/consul.html#path): Set to `vault/`.
  * [service](https://www.vaultproject.io/docs/configuration/storage/consul.html#service): Set to `vault`.  
  * [redirect_addr](https://www.vaultproject.io/docs/configuration/storage/consul.html#redirect_addr): Set to `https://<PRIVATE_IP>:<CLUSTER_PORT>` where `PRIVATE_IP` is the Instance's private IP and `CLUSTER_PORT` is the value passed to `--cluster-port`.  
  * [cluster_addr](https://www.vaultproject.io/docs/configuration/storage/consul.html#cluster_addr): Set to `https://<PRIVATE_IP>:<CLUSTER_PORT>` where `PRIVATE_IP` is the Instance's private IP and `CLUSTER_PORT` is the value passed to `--cluster-port`.
* [listener](https://www.vaultproject.io/docs/configuration/index.html#listener): Configure a [TCP listener](https://www.vaultproject.io/docs/configuration/listener/tcp.html) with the following settings:
  * [address](https://www.vaultproject.io/docs/configuration/listener/tcp.html#address): Bind to `0.0.0.0:<PORT>` where `PORT` is the value passed to `--port`.
  * [cluster_address](https://www.vaultproject.io/docs/configuration/listener/tcp.html#cluster_address): Bind to `0.0.0.0:<CLUSTER_PORT>` where `CLUSTER` is the value passed to `--cluster-port`.
  * [tls_cert_file](https://www.vaultproject.io/docs/configuration/listener/tcp.html#tls_cert_file): Set to the `--tls-cert-file` parameter.
  * [tls_key_file](https://www.vaultproject.io/docs/configuration/listener/tcp.html#tls_key_file): Set to the `--tls-key-file` parameter.

### Overriding the configuration

To override the default configuration, simply put your own configuration file in the Vault config folder (default: `/opt/vault/config`), but with a name that comes later in the alphabet than `default.hcl` (e.g. `my-custom-config.hcl`). Vault will load all the `.hcl` configuration files in the config dir and merge them together in alphabetical order, so that settings in files that come later in the alphabet will override the earlier ones.

For example, to set a custom `cluster_name` setting, you could create a file called `name.hcl` with the contents:

```hcl
cluster_name = "my-custom-name"
```

If you want to override *all* the default settings, you can tell `run-vault` not to generate a default config file
at all using the `--skip-vault-config` flag:

```shell
/opt/vault/bin/run-vault --azure-account-name my-account-name --azure-account-key [REDACTED] --azure-container my-container-name --tls-cert-file /opt/vault/tls/vault.crt.pem --tls-key-file /opt/vault/tls/vault.key.pem --skip-vault-config
```

## How to handle encryption

Vault uses TLS to encrypt all data in transit. To configure encryption, you must do the following:

1. [Provide TLS certificates](#provide-tls-certificates)
1. [Consul encryption](#consul-encryption)

### Provide TLS certificates

When you execute the `run-vault` script, you need to provide the paths to the public and private keys of a TLS certificate:

```shell
/opt/vault/bin/run-vault --azure-account-name my-account-name --azure-account-key [REDACTED] --azure-container my-container-name --tls-cert-file /opt/vault/tls/vault.crt.pem --tls-key-file /opt/vault/tls/vault.key.pem
```

See the [private-tls-cert module](./modules/private-tls-cert) for information on how to generate a TLS certificate.

### Consul encryption

Since this Vault Module uses Consul as a high availability storage backend, you may want to enable encryption for Consul too. Note that Vault encrypts any data *before* sending it to a storage backend, so this isn't strictly necessary, but may be a good extra layer of security.

By default, the Vault server nodes communicate with a local Consul agent running on the same server over (unencrypted) HTTP. However, you can configure those agents to talk to the Consul servers using TLS. Check out the [official Consul encryption docs](https://www.consul.io/docs/agent/encryption.html) and the Consul Azure Module [How do you handle encryption docs](.//modules/run-consul#how-do-you-handle-encryption) for more info.
