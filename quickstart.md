# Quickstart

The instructions included with this Blueprint arent the clearest or easiest to understand.
This Quickstart will fix that.

## Assumptions

This quick start assumes the following:

* Terraform (TF) 0.12.x is used
* Terraform is installed and being run locally (no TF Cloud (TFC), TF Enterprise (TFE) or other automation)
* Packer is installed and being run locally
* Shared State is not used
* You have access to an Azure Subscription

## Prepare your local machine

1. Configure an Azure AD (AAD) Service Principle (SP) for your local TF to connect to AAD. [Hashicorp Docs](https://www.terraform.io/docs/providers/azurerm/auth/service_principal_client_secret.html)
2. `git clone` the repo to your local machine
3. Create a `terraform.tfvars` file or copy the example one (`terraform.tfvars.example`) that is included and fill in the values you will use.

## Build your images

Assuming you're in the root of the repo you cloned.

### Build the Consule image

1. TODO

### Build the Vault image

1. Modify the `./examples/vault-consul-image/vault-consul.json` to suit your needs
1. `cd ./examples/vault-consul-image`
1. `packer build ./vault-consul.json`

Packer will build the Azure image used by the Vault ScaleSet.
The URI of the image will be output at completion. Copy this URI to the `image_uri` variable in your `terraform.tfvars` file.

## Build your Cluster

With the images ready for use you can now deploy the Cluster.

1. Run `terraform init`
1. Run `terraform plan`
1. If the plan looks good, run `terraform apply`
1. Run the [vault-examples-helper.sh script](./examples/vault-examples-helper/vault-examples-helper.sh) to print out the IP addresses of the Vault servers and some example commands you can run to interact with the cluster: `../vault-examples-helper/vault-examples-helper.sh`.

## Author

Name:   Andrew Best
Email:  abest@diaxion.com
GitHub: ausfestivus
