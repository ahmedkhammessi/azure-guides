# AKS Node Public IP
This guide demonstrates how to create an AKS cluster with a public IP assigned to each node. You can find the Azure doc [here](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#assign-a-public-ip-per-node-in-a-node-pool).

## Setup
Cluster creation requires a few values be created and set in the azuredeploy.parameters.json.

### Deployment Parameters
- *clusterName:* Uniquely identifies your cluster within a resource group
- *dnsprefix:* unique name used for the kubernetes api server FQDN. Should be as unique as possible, lowercase and alphanumeric.
- *agentCount:* number of nodes that should be deployed
- *agentVMSize:* Virtual machine size used for the nodes
- *linuxAdminUsername:* admin user name assigned to the nodes used if you ssh into the nodes for diagnostics
- *sshRSAPublicKey:* ssh key used if you need to connect to the nodes via ssh for diagnostic reasons. You can generate a public key with ssh-keygen
- *servicePrincipalClientId:* App ID for the cluster service principal. Generated via steps below.
- *servicePrincipalClientSecret:* Password for the service principal. See steps below

1. Register for the preview feature
```bash
az feature register --name NodePublicIPPreview --namespace Microsoft.ContainerService

# Check registration status (may take up to 30min to register)
az feature show --namespace Microsoft.ContainerService --name NodePublicIPPreview

# Once registered you need to refresh the provider
az provider register --namespace Microsoft.ContainerService
```

1. Generate ssh key
    ```bash
    # Run ssh-keygen to generate an ssh key for the cluster
    ssh-keyget

    # Get the public key from the generated key
    cat ~/.ssh/id_rsa.pub
    ```
1. Generate service principal for the cluster
    ```bash
    az ad sp create-for-rbac --skip-assignment -o json

    # Example output
    {
    "appId": "sdfsdsd-ac68-4a3b-a936-sdfsdfsdfds",
    "displayName": "azure-cli-2020-03-25-02-54-39",
    "name": "http://azure-cli-2020-03-25-02-54-39",
    "password": "sdffsddd-042c-4d36-9b0c-sdfsdfdsfsdf",
    "tenant": "4fdsf4-86f1-41af-91ab-asdfsdfsdf"
    }
    ```

1. Update the azuredeploy.parameters.json with the ssh key, app id, password and any other values you want to adjust.

1. Deploy the cluster
    ```bash
    chmod +x clustercreate.sh
    ./clustercreate.sh
    ```
1. After creation completes, check the nodes have external ips
    ```bash
    # Get your AKS cluster credentials
    az aks get-credentials -g <Resrouce Group Name> -n <Cluster Name>

    # Check the nodes have 'External IPs' assigned
    kubectl get nodes -o wide
    NAME                                STATUS   ROLES   AGE     VERSION    INTERNAL-IP   EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
    aks-nodepool1-37394874-vmss000000   Ready    agent   3m3s    v1.15.10   10.240.0.4    40.121.148.106   Ubuntu 16.04.6 LTS   4.15.0-1071-azure   docker://3.0.10+azure
    aks-nodepool1-37394874-vmss000001   Ready    agent   3m7s    v1.15.10   10.240.0.5    40.121.149.70    Ubuntu 16.04.6 LTS   4.15.0-1071-azure   docker://3.0.10+azure
    aks-nodepool1-37394874-vmss000002   Ready    agent   3m28s   v1.15.10   10.240.0.6    40.121.144.56    Ubuntu 16.04.6 LTS   4.15.0-1071-azure   docker://3.0.10+azure
    ```

1. To deploy a pod that leverages the public IP you'll need to take advantage of the hostNetwork of the pod specification. 

    ```bash
    # Deploy the pod
    kubectl apply -f nginx-node-pubip.yaml

    # Get the pod public IP
    kubectl get node $(kubectl get pods -l run=nginx -o jsonpath='{.items[0].spec.nodeName}') -o jsonpath='{.status.addresses[?(@.type=="ExternalIP")].address}'

    # For this demo you can use the following to get the URL to the pod
    echo http://$(kubectl get node $(kubectl get pods -l run=nginx -o jsonpath='{.items[0].spec.nodeName}') -o jsonpath='{.status.addresses[?(@.type=="ExternalIP")].address}')
    ```

1. Finally, you will need to open up the Azure Network Security group that is created for the cluster to allow traffic to the nodes on port 80 where Nginx is hosted.

    ```bash
    RG=<Insert the Resource Group Name for your Cluster>
    CLUSTERNAME=<Insert your cluster name>

    # Get the 'Managed Cluster' (MC) resource group for your cluster
    MCRG=$(az aks show -g $RG -n $CLUSTERNAME -o tsv --query 'nodeResourceGroup')

    # Get the Network Security Group Name for your cluster
    NSG=$(az network nsg list -g $MCRG -o tsv --query '[0].name')

    # Create the rule to allow traffic to port 80 for the subnet
    az network nsg rule create -g $MCRG --nsg-name $NSG -n Allow80 --priority 100 \
        --destination-address-prefixes '*' --destination-port-ranges 80 --access Allow \
        --protocol Tcp --description "Allow inbound traffic to port 80 on all nodes."
    ```

1. You should now be able to access the Nginx URL you retrieved above.