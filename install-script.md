# OCP UPI Advanced Installation Script for Azure 

Ref: YouTube Video - https://youtu.be/h2QfP9IYzeY


This script expects the SPN to have Contributor rights assigned to the Resource Group. This script and accompanying customized templates are provided to help install OpenShift 4.6+ on Azure within a Private Network with Restricted “Contributor” roles to the SPN and Managed Identity. 
```
Note: * Elevated privileges required for this step, please have your administrator / owner build these in advance along with the Resource Group, VNET and all DNS entries.
```

## PRE INSTALL SETUP
### Environment Variables Used 
```
export AZURE_REGION=<location>
export CLUSTER_NAME=<cluster_name>
export BASE_DOMAIN=<base_domain>.<com>
export PRVT_DNS="${CLUSTER_NAME}.${BASE_DOMAIN}"
export RESOURCE_GROUP=<resource_group_name>
export VNET_RG=<virtual_network_resource_group_name>
export VNET=<virtual_network_name>
export INSTALL=<INSTALL_DIRECTORY>
export KUBECONFIG=${INSTALL}/auth/kubeconfig
export SSH_KEY=`cat </path/to/.ssh/id_rsa.pub>`
export STORAGE_SA=<storageaccount>
export MAN_ID=<managed_identity_name>
export MASTER_SUB=<master_subnet_name>
export WORKER_SUB=<worker_subnet_name>
export INT_LB=<internal_lb_ip>
export INT_APPS_LB=<internal_apps_lb_ip>
```
### DNS Requirements
Resolves to ${INT_LB} 
```
	api.<cluster_name>.<base_domain>.<com>
	api-int.<cluster_name>.<base_domain>.<com>
```
Resolves to ${INT_APPS_LB}
```
	*.apps.<cluster_name>.<base_domain>.<com>
```

Note: If wildcard resolution is not permitted add this prior to installation, pointing to ${INT_APPS_LB}
```
oauth-openshift.apps.<cluster_name>.<base_domain>.<com>
console-openshift-console.apps.<cluster_name>.<base_domain>.<com>
downloads-openshift-console.apps.<cluster_name>.<base_domain>.<com>
default-route-openshift-image-registry.apps.<cluster_name>.<base_domain>.<com>
alertmanager-main-openshift-monitoring.apps.<cluster_name>.<base_domain>.<com>
grafana-openshift-monitoring.apps.<cluster_name>.<base_domain>.<com>
prometheus-k8s-openshift-monitoring.apps.<cluster_name>.<base_domain>.<com>
thanos-querier-openshift-monitoring.apps.<cluster_name>.<base_domain>.<com>
```

1. *Create Identity Acct and assign "Contributor" role to RG 
```
az identity create -g ${RESOURCE_GROUP} -n ${MAN_ID}
export PRINCIPAL_ID=`az identity show -g ${RESOURCE_GROUP} -n ${MAN_ID} --query principalId --out tsv` 
export RESOURCE_GROUP_ID=`az group show -g ${RESOURCE_GROUP} --query id --out tsv`
az role assignment create --assignee "${PRINCIPAL_ID}" --role 'Contributor' --scope "${RESOURCE_GROUP_ID}"
```
2. Create private dns zone = "<cluster_name>.<base_domain>.<com>" for local lookups within the RG
```
az network private-dns zone create -g ${RESOURCE_GROUP} -n ${PRVT_DNS}
```
3. *Link Private DNS Zone to VNET
```
az network private-dns link vnet create -g ${RESOURCE_GROUP} -z ${PRVT_DNS} -n ${CLUSTER_NAME}-network-link -v ${VNET} -e false
```
4. Create StorageSA account
```
az storage account create -g ${RESOURCE_GROUP} --location ${AZURE_REGION} --name ${STORAGE_SA} --kind Storage --sku Standard_LRS
```

## INSTALLATION PROCESS

Login with your SPN \
`az login --service-principal --username APP_ID --password PASSWORD --tenant TENANT_ID`

1. Create install-config (Optional, use included customized "install-config.yaml") \
`openshift-install create install-config —dir=$INSTALL`

2. Create manifests \
`openshift-install create manifests --dir=$INSTALL`

3. Backup worker machine-config \
`cp $INSTALL/openshift/99_openshift-cluster-api_worker-machineset-0.yaml ~/`

4. Set IngressController to "HostNetwork" / replicas: 3 \
`vi $INSTALL/manifests/cluster-ingress-default-ingresscontroller.yaml`
    ```
    apiVersion: operator.openshift.io/v1
    kind: IngressController
    metadata:
    name: default
    namespace: openshift-ingress-operator
    spec:
    endpointPublishingStrategy:
        type: HostNetwork
        replicas: 3
    ```
5. Delete machine-configs \
`rm -f $INSTALL/openshift/99_openshift-cluster-api_master-machines-*.yaml` \
`rm -f $INSTALL/openshift/99_openshift-cluster-api_worker-machineset-*.yaml`

6. Record resource group name for $INFRA_ID \
    export INFRA_ID=`grep infrastructureName $INSTALL/manifests/cluster-infrastructure-02-config.yml | awk -F":" '{gsub (" ", "", $0); print $2}'`
    
7. Create ignition files \
`openshift-install create ignition-configs --dir=${INSTALL}`

8. Create private api load balancing
    ```
    az deployment group create -g ${RESOURCE_GROUP} \
    --template-file "adv_infra.json" \
    --parameters privateDNSZoneName="${PRVT_DNS}" \
    --parameters baseName="${INFRA_ID}" \
    --parameters virtualNetworkResourceGroup="${VNET_RG}" \
    --parameters virtualNetworkName="${VNET}" \
    --parameters masterSubnetName="${MASTER_SUB}" \
    --parameters serviceSubnetName="${WORKER_SUB}" \
    --parameters internalLoadBalancerIP="${INT_LB}" \
    --parameters internalAppsLoadBalancerIP="${INT_APPS_LB}"
    ```
9. Export storage account key and vhd_url \
export ACCOUNT_KEY=`az storage account keys list -g ${RESOURCE_GROUP} --account-name ${STORAGE_SA} --query "[0].value" -o tsv` \
export VHD_URL=`curl -s https://raw.githubusercontent.com/openshift/installer/release-4.6/data/data/rhcos.json | jq -r .azure.url`

10. Create a storage container for the rhcos.vhd \
`az storage container create --name vhd --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY}`

11. Copy rhcos.vhd into the container \
`az storage blob copy start --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} --destination-blob "rhcos.vhd" --destination-container vhd --source-uri "${VHD_URL}"`
12. While Loop, wait until completed before proceeding
    ```
    status="unknown"
    while [ "$status" != "success" ]
    do
    status=`az storage blob show --container-name vhd --name "rhcos.vhd" --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} -o tsv --query properties.copy.status`
    echo $status
    done
    ```
13. Create the storage container to host the bootstrap.ign file \
`az storage container create --name files --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} --public-access blob`

14. Copy bootstrap.ign into the container \
`az storage blob upload --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} -c "files" -f "$INSTALL/bootstrap.ign" -n "bootstrap.ign"`

15. Export the RHCOS VHD \
export VHD_BLOB_URL=`az storage blob url --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} -c vhd -n "rhcos.vhd" -o tsv`

16. Deploy rhcos image def_storage.json \
`az deployment group create -g ${RESOURCE_GROUP} --template-file "def_storage.json" --parameters vhdBlobURL="${VHD_BLOB_URL}" --parameters baseName="${INFRA_ID}"`

17. Creating the bootstrap \
export BOOTSTRAP_URL=`az storage blob url --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} -c "files" -n "bootstrap.ign" -o tsv` \
export BOOTSTRAP_IGNITION=`jq -rcnM --arg v "3.1.0" --arg url ${BOOTSTRAP_URL} '{ignition:{version:$v,config:{replace:{source:$url}}}}' | base64 | tr -d '\n'`
    ```
    az deployment group create -g ${RESOURCE_GROUP} --template-file "adv_bootstrap.json" \
    --parameters bootstrapIgnition="${BOOTSTRAP_IGNITION}" \
    --parameters sshKeyData="${SSH_KEY}" \
    --parameters baseName="${INFRA_ID}" \
    --parameters virtualNetworkResourceGroup="${VNET_RG}" \
    --parameters virtualNetworkName="${VNET}" \
    --parameters masterSubnetName="${MASTER_SUB}" \
    --parameters identityName="${MAN_ID}"
    ```

18. Creating the masters \
export MASTER_IGNITION=`cat $INSTALL/master.ign | base64 -w0 | tr -d '\n'`
    ```
    az deployment group create -g ${RESOURCE_GROUP} --template-file "adv_masters.json" \
    --parameters masterIgnition="${MASTER_IGNITION}" \
    --parameters sshKeyData="${SSH_KEY}" \
    --parameters privateDNSZoneName="{PRVT_DNS}" \
    --parameters baseName="${INFRA_ID}" \
    --parameters virtualNetworkResourceGroup="${VNET_RG}" \
    --parameters virtualNetworkName="${VNET}" \
    --parameters masterSubnetName="${MASTER_SUB}" \
    --parameters identityName="${MAN_ID}"
    ```
19. Monitor bootstrap process \
`openshift-install wait-for bootstrap-complete --dir=$INSTALL  --log-level info`

20. Creating workers \
export WORKER_IGNITION=`cat $INSTALL/worker.ign | base64 -w0 | tr -d '\n'`
    ```
    az deployment group create -g ${RESOURCE_GROUP} --template-file "adv_workers.json" \
    --parameters workerIgnition="${WORKER_IGNITION}" \
    --parameters sshKeyData="${SSH_KEY}" \
    --parameters baseName="${INFRA_ID}" \
    --parameters virtualNetworkResourceGroup="${VNET_RG}" \
    --parameters virtualNetworkName="${VNET}" \
    --parameters identityName="${MAN_ID}" \
    --parameters serviceSubnetName="${WORKER_SUB}"
    ```
21. Approve csr's \
`watch "oc get csr | grep Pending"` \
`oc adm certificate approve ...... `

22. Remove Bootstrap Resources
    ```
    az vm stop -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap
    az vm deallocate -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap
    az vm delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap --yes
    az disk delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap_OSDisk --no-wait --yes
    az network nic delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap-nic --no-wait
    az storage blob delete --account-key ${ACCOUNT_KEY} --account-name ${STORAGE_SA} --container-name files --name bootstrap.ign
    ```
23. Add workers to both backend load balancer pools and wait for services to come online. \
`watch oc get co`