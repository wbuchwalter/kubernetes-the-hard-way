# Cloud Infrastructure Provisioning - Azure
This lab will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 10 virtual machines will be created.

The guide assumes you'll be creating resources in the `West Us` region as a single Azure Resource Manager resource group.


> All machines will be provisioned with fixed private IP addresses to simplify the bootstrap process.

The cluster VMs are only accessible via a jump box (a VM with publicly accessible ssh endpoint). The workers machines are exposed via external load balancer that carries both an public IP and public FQDN. 


## Variables

Change the following values as needed. 

```
# dns for jumpbox is <jumpboxDnsLabel>.westus.cloudapp.azure.com
jumpboxDnsLabel="the-hard-way-jumpbox" 
```

```
# dns for workers is <workersDnsLabel>.westus.cloudapp.azure.com
workersDnsLabel="the-hard-way" 
```

```
#storage account used by jumpbox + controllers + Etcd VMs
controlPlaneStorageAccount="thehardwaycsa"
```

```
#storage account used by workers VMs
workersStorageAccount="thehardwaywsa"
```

```
# all vms are using ubunut 16.4 LTS 
imageUrn="Canonical:UbuntuServer:16.04.0-LTS:latest"
```

## Create Resource Group

```
azure group create \
	--name the-hard-way \
	--location "West Us"
```

## Networking

### Create Routing Table

```
azure network route-table create \
	--resource-group the-hard-way \
	--name the-hard-way-rtable \
	--location "West Us"
```

### Create Network Security Group (NSG)

```
azure network nsg create \
	--resource-group the-hard-way \
	--name the-hard-way-nsg \
	--location "West Us"
```


Create NSG rule allowing SSH to the jumpbox

```
azure network nsg rule create \
	--resource-group the-hard-way \
	--nsg-name the-hard-way-nsg \
	--name allow-ssh-jumpbox \
	--protocol tcp \
	--access allow  \
	--destination-address-prefix 10.0.0.5 \
	--destination-port-range 22 \
	--priority 100 \
	--direction inbound
```


### Create VNET + Subnet 

Cluster VNET
```
azure network vnet create \
	--resource-group the-hard-way \
	--name the-hard-way-net \
	--address-prefixes 10.0.0.0/8 \
	--location "West Us"
```

Create subnets

```
# Azure UDR "user defined routes" in custom routing tables 
# routes traffic leaving the subnet.
# Workers & pods (IPs) have to be in two separate subnets

azure network vnet subnet create \
	--resource-group the-hard-way \
	--vnet-name the-hard-way-net \
	--name kubernetes-mgmt \
	--address-prefix 10.0.0.0/16


azure network vnet subnet create \
	--resource-group the-hard-way \
	--vnet-name the-hard-way-net \
	--name kubernetes \
	--address-prefix 10.224.0.0/11


```

Associate the routing table and NSG to Kubernetes/-mgmt subnets

```
azure network vnet subnet set \
	--resource-group the-hard-way \
	--vnet-name the-hard-way-net \
	--name kubernetes-mgmt \
	--network-security-group-name the-hard-way-nsg \
	--route-table-name the-hard-way-rtable


azure network vnet subnet set \
	--resource-group the-hard-way \
	--vnet-name the-hard-way-net \
	--name kubernetes \
	--network-security-group-name the-hard-way-nsg \
	--route-table-name the-hard-way-rtable
```


Create public IP + DNS label for the jumpbox

```
azure network public-ip create \
	--resource-group the-hard-way \
	--name the-hard-way-jumpbox  \
	--allocation-method Dynamic \
	--domain-name-label $jumpboxDnsLabel \
	--location "West Us"
```

## Virtual Machines

Create SSH keys (Used by All VMs)

```
mkdir keys
ssh-keygen -t rsa -f ./keys/cluster
```

### Storage Accounts 

Create storage account for control plane VMs (Etcd & Controllers)

```
azure storage account create $controlPlaneStorageAccount \
	--resource-group the-hard-way \
	--kind storage \
	--sku-name LRS \
	--location "West Us"
```

Create storage account for workers VMs

```
azure storage account create $workersStorageAccount \
	--resource-group the-hard-way \
	--kind storage \
	--sku-name LRS \
	--location "West Us"
```



### Jump Box

#### Create NIC (Private IP + Public IP + FQDN)

```
azure network nic create \
	--resource-group the-hard-way \
	--name jumpbox-nic \
	--private-ip-address "10.0.0.5" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes-mgmt  \
	--public-ip-name the-hard-way-jumpbox \
	--location "West Us"
```

#### Create VM

```
azure vm create \
	--resource-group the-hard-way \
	--name jumpbox \
	--vm-size Standard_A1 \
	--nic-name jumpbox-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes-mgmt  \
	--os-type linux \
	--image-urn $imageUrn \
	--storage-account-name $controlPlaneStorageAccount \
	--storage-account-container-name vhds \
	--os-disk-vhd jumpbox.vhd \
	--admin-username thehardway \
	--ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
``` 

### Etcd 

#### Etcd 0

Create Nic
```
azure network nic create \
	--resource-group the-hard-way \
	--name etcd-0-nic \
	--private-ip-address "10.240.0.10" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--location "West Us"
```
Create VM

```
azure vm create \
	--resource-group the-hard-way \
	--name etcd0 \
	--vm-size Standard_A1 \
	--nic-name etcd-0-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
	--os-type linux \
	--image-urn $imageUrn \
	--storage-account-name $controlPlaneStorageAccount \
	--storage-account-container-name vhds \
	--os-disk-vhd etcd-0.vhd \
	--admin-username thehardway \
	--ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```

#### Etcd 1

Create Nic

```
azure network nic create \
	--resource-group the-hard-way \
	--name etcd-1-nic \
	--private-ip-address "10.240.0.11" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--location "West Us"
```

Create VM

```
azure vm create \
	--resource-group the-hard-way \
	--name etcd1 \
	--vm-size Standard_A1 \
	--nic-name etcd-1-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
	--os-type linux \
	--image-urn $imageUrn \
	--storage-account-name $controlPlaneStorageAccount \
	--storage-account-container-name vhds \
	--os-disk-vhd etcd-1.vhd \
	--admin-username thehardway \
	--ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```

#### Etcd 2

Create Nic

```
azure network nic create \
	--resource-group the-hard-way \
	--name etcd-2-nic \
	--private-ip-address "10.240.0.12" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--location "West Us"
```

Create VM

```
azure vm create \
	--resource-group the-hard-way \
	--name etcd2 \
	--vm-size Standard_A1 \
	--nic-name etcd-2-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
	--os-type linux \
	--image-urn $imageUrn \
	--storage-account-name $controlPlaneStorageAccount \
	--storage-account-container-name vhds \
	--os-disk-vhd etcd-2.vhd \
	--admin-username thehardway \
	--ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```


### Kubernetes Controllers


#### Controllers Internal Load Balancer 

Create load balancer 

```
azure network lb create \
	--resource-group the-hard-way \
	--name the-hard-way-clb \
	--location "West Us"
```

Create & assign the front-end private IP to the internal load balancer

```
azure network lb frontend-ip create \
	--resource-group the-hard-way \
	--name the-hard-way-cfe  \
	--lb-name the-hard-way-clb \
	--private-ip-address "10.240.0.4" \
	--subnet-vnet-name the-hard-way-net \
	--subnet-name kubernetes
```

Create a backend address pool for the load balancer

```
clbbackendPoolId=$(azure network lb address-pool create \
	--resource-group the-hard-way \
	--lb-name the-hard-way-clb \
	--name backend-pool \
	--json | jq -r '.id')
```

#### Create controllers availability set

```
azure availset create \
	--resource-group the-hard-way \
	--name controllers-availset \
	--location "West Us"
```


#### Controller 0 

Create Nic

```
azure network nic create \
	--resource-group the-hard-way \
	--name controller-0-nic \
	--private-ip-address "10.240.0.20" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--lb-address-pool-ids $clbbackendPoolId \
	--location "West Us"
```

Create VM

```
azure vm create \
	--resource-group the-hard-way \
    --name controller0 \
    --vm-size Standard_A1 \
    --nic-name controller-0-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
	--availset-name controllers-availset \
    --os-type linux \
    --image-urn $imageUrn \
    --storage-account-name $controlPlaneStorageAccount \
    --storage-account-container-name vhds \
    --os-disk-vhd controller-0.vhd \
    --admin-username thehardway \
    --ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```

#### Controller 1 

Create Nic

```
azure network nic create \
	--resource-group the-hard-way \
	--name controller-1-nic \
	--private-ip-address "10.240.0.21" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--lb-address-pool-ids $clbbackendPoolId \
	--location "West Us"
```

Create VM

```
azure vm create \
	--resource-group the-hard-way \
    --name controller1 \
    --vm-size Standard_A1 \
    --nic-name controller-1-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
	--availset-name controllers-availset \
    --os-type linux \
    --image-urn $imageUrn \
    --storage-account-name $controlPlaneStorageAccount \
    --storage-account-container-name vhds \
    --os-disk-vhd controller-1.vhd \
    --admin-username thehardway \
    --ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```

#### Controller 2

Create Nic

```
azure network nic create \
	--resource-group the-hard-way \
	--name controller-2-nic \
	--private-ip-address "10.240.0.22" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--lb-address-pool-ids $clbbackendPoolId \
	--location "West Us"
```

Create VM

```
azure vm create \
	--resource-group the-hard-way \
    --name controller2 \
    --vm-size Standard_A1 \
    --nic-names controller-2-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
	--availset-name controllers-availset \
    --os-type linux \
    --image-urn $imageUrn \
    --storage-account-name $controlPlaneStorageAccount \
    --storage-account-container-name vhds \
    --os-disk-vhd controller-2.vhd \
    --admin-username thehardway \
    --ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```


### Kubernetes Workers

#### Workers External Load Balancer 

Create public IP + DNS label for workers ingestion external load balancer

```
azure network public-ip create \
	--resource-group the-hard-way \
	--name the-hard-way-workers  \
	--allocation-method Dynamic \
	--domain-name-label $workersDnsLabel \
	--location "West Us"
```

Create load balancer 

```
azure network lb create \
	--resource-group the-hard-way \
	--name the-hard-way-lb \
	--location "West Us"
```

Assign the front-end public IP + FQDN to the load balancer

```
azure network lb frontend-ip create \
	--resource-group the-hard-way \
	--name the-hard-way-fe  \
	--lb-name the-hard-way-lb \
	--public-ip-name the-hard-way-workers
```

Create a backend address pool for the load balancer

```
wlbbackendPoolId=$(azure network lb address-pool create \
	--resource-group the-hard-way \
	--lb-name the-hard-way-lb \
	--name backend-pool \
	--json | jq -r '.id')
```

#### Create Workers Availablity set

```
azure availset create \
	--resource-group the-hard-way \
	--name workers-availset \
	--location "West Us"
```

#### Worker 0

Create Nic

```
azure network nic create \
	--resource-group the-hard-way \
	--name worker-0-nic \
	--private-ip-address "10.240.0.30" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--enable-ip-forwarding "true" \
	--lb-address-pool-ids $wlbbackendPoolId \
	--location "West Us"
```

Create VM

```
azure vm create \
	--resource-group the-hard-way \
    --name worker0 \
    --vm-size Standard_D4 \
    --nic-name worker-0-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
    --availset-name workers-availset \
	--os-type linux \
    --image-urn $imageUrn \
    --storage-account-name $workersStorageAccount \
    --storage-account-container-name vhds \
    --os-disk-vhd worker-0.vhd \
    --admin-username thehardway \
    --ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```

#### Worker 1

Create Nic

```
azure network nic create \
	--resource-group the-hard-way \
	--name worker-1-nic \
	--private-ip-address "10.240.0.31" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--enable-ip-forwarding "true" \
	--lb-address-pool-ids $wlbbackendPoolId \
	--location "West Us"
```

Create VM

```
azure vm create \
	--resource-group the-hard-way \
    --name worker1 \
    --vm-size Standard_D4 \
    --nic-name worker-1-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
    --availset-name workers-availset \
	--os-type linux \
    --image-urn $imageUrn \
    --storage-account-name $workersStorageAccount \
    --storage-account-container-name vhds \
    --os-disk-vhd worker-1.vhd \
    --admin-username thehardway \
    --ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```

#### Worker 2

Create Nic

```
azure network nic create \
	--resource-group the-hard-way \
	--name worker-2-nic \
	--private-ip-address "10.240.0.32" \
	--subnet-vnet-name the-hard-way-net  \
	--subnet-name kubernetes  \
	--enable-ip-forwarding "true" \
	--lb-address-pool-ids $wlbbackendPoolId \
	--location "West Us"
```

Create VM

```
azure vm create \
	--resource-group the-hard-way \
    --name worker2 \
    --vm-size Standard_D4 \
    --nic-name worker-2-nic \
	--vnet-name the-hard-way-net  \
	--vnet-subnet-name kubernetes  \
    --availset-name workers-availset \
	--os-type linux \
    --image-urn $imageUrn \
    --storage-account-name $workersStorageAccount \
    --storage-account-container-name vhds \
    --os-disk-vhd worker-2.vhd \
    --admin-username thehardway \
    --ssh-publickey-file ./keys/cluster.pub \
	--location "West US"
```

## Verify 

```
azure vm list --resource-group the-hard-way
```

Expected Output 
```
info:    Executing command vm list
+ Getting virtual machines                                                     
data:    ResourceGroupName  Name          ProvisioningState  PowerState  Location  Size       
data:    -----------------  ------------  -----------------  ----------  --------  -----------
data:    the-hard-way       controller-0  Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       controller-1  Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       controller-2  Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       etcd-0        Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       etcd-1        Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       etcd-2        Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       jumpbox       Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       worker-0      Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       worker-1      Succeeded          VM running  westus    Standard_A1
data:    the-hard-way       worker-2      Succeeded          VM running  westus    Standard_A1
info:    vm list command OK
```


## Using The Jumpbox

> The Jumpbox does not have Azure CLI installed. All further Azure CLI commands should be executed on a machine with CLI installed & configured.   

### Connect to Jumpbox 

```
ssh -i ./keys/cluster \
	thehardway@$jumpboxDnsLabel.westus.cloudapp.azure.com
```

### Copy the cluster private key to jumpbox 

```
scp -i ./keys/cluster \
	./keys/cluster \
	thehardway@$jumpboxDnsLabel.westus.cloudapp.azure.com:~/cluster
```

### Connecting to Other VMs

```
#from the jumpbox 
#connect to the second controller

ssh -i ./cluster \
	thehardway@10.240.0.31  

#or

ssh -i ./cluster \
	thehardway@controller-1  

```
