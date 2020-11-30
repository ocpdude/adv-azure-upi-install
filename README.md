# Azure UPI Advanced Install
## Customized Azure UPI Install as "Contributor" RBAC Only

### To follow along with this documentation see my YouTube video at https://youtu.be/h2QfP9IYzeY

This install is a Private Cluster without access from the Internet, a common build for Enterprises linked with Azure.

During this install, we will create a SPN and Managed Identity with `Contributor` rights, they will both be assigned to the Resource Group that we will be installing OpenShift 4.6 into. The SPN will also be assigned as `Network Contributor` to the VNET in a separate Resource Group. I identifed two areas that require elevated privileges:
1. Create the SPN & MI
2. Link the Private DNS Zone to the VNET

We will log in as the SPN to being the install once the setup has completed.


Download the openshift-installer, client tools (oc & kubectl), and your subscription pull-secret from `https://cloud.redhat.com`

Download the Azure CLI from `https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt`

Download 'jq' from your package installers or from `https://stedolan.github.io/jq/download/`

All of the customized ARM templates I used can be found under /json.

The script to follow is `dude_script.md`





