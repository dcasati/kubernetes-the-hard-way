# Provisioning Resources on Microsoft Azure

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [region](https://azure.microsoft.com/regions/).

> Ensure a default region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Network](https://docs.microsoft.com/en-us/azure/virtual-network/) (VNet) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` VNet network and a subnet named `kubernetes`:

```
az network vnet create \
--name kubernetes-the-hard-way \
--resource-group kubernetes-the-hard-way \
--location westus2 \
--address-prefix 10.240.0.0/16 \
--subnet-name kubernetes \
--subnet-prefix 10.240.0.0/24
```

> Please note that Azure reserves a handful of IP address on each subnet for it's internal services. Hence, a `10.240.0.0/24` subnet will provide 251 IP address. For more information please refer to the [Azure Virtual Network FAQ](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq)

### Firewall Rules

Create a Network Security Group to accomodate our rules:

```
az network nsg create \
  --name kubernetes-the-hard-way \
  --resource-group kubernetes-the-hard-way 
```

<!--  THIS MIGHT NOT BE NEEDED

Create a firewall rule that allows internal communication across all protocols:

```
az network nsg rule create \
  --resource-group kubernetes-the-hard-way \
  --nsg-name kubernetes-the-hard-way \
  --name allow-internal \
  --access Allow \
  --protocol "*" \
  --direction Inbound \
  --priority 100 \
  --source-address-prefix 10.240.0.0/24,10.200.0.0/16 \
  --source-port-range "*" \
  --destination-address-prefix "*" \
  --destination-port-range "*"
``` -->

Create a firewall rule that allows external SSH, ICMP, and HTTPS:

```
az network nsg rule create \
  --name kubernetes-the-hard-way-allow-external \
  --nsg-name kubernetes-the-hard-way \
  --resource-group kubernetes-the-hard-way \
  --priority 100 \
  --protocol Tcp \
  --destination-port-ranges 22 6443 \
  --destination-address-prefixes VirtualNetwork
```

> An [Internet facing load balancer](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-internet-overview) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VNet network:

```
az network nsg rule list \
  --nsg-name kubernetes-the-hard-way \
  -g kubernetes-the-hard-way \
  -o table
```

> output

```
Access    DestinationAddressPrefix    Direction    Name                                      Priority  Protocol    ProvisioningState    ResourceGroup            SourceAddressPrefix    SourcePortRange
--------  --------------------------  -----------  --------------------------------------  ----------  ----------  -------------------  -----------------------  ---------------------  -----------------
Allow     VirtualNetwork              Inbound      kubernetes-the-hard-way-allow-external         100  Tcp         Succeeded            kubernetes-the-hard-way  *                      *
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
az network public-ip create \
  --name kubernetes-the-hard-way \
  --resource-group kubernetes-the-hard-way \
  --location westus2 \
  --allocation-method Static 
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
az network public-ip list --resource-group kubernetes-the-hard-way  -o jsonc
```

> output

```
[
  {
    "dnsSettings": null,
    "etag": "W/\"d6b67561-eebe-4064-b071-f1f938059a08\"",
    "id": "/subscriptions/XYZ/resourceGroups/kubernetes-the-hard-way/providers/Microsoft.Network/publicIPAddresses/kubernetes-the-hard-way",
    "idleTimeoutInMinutes": 4,
    "ipAddress": "XXX.XXX.XXX.XXX",
    "ipConfiguration": null,
    "location": "westus2",
    "name": "kubernetes-the-hard-way",
    "provisioningState": "Succeeded",
    "publicIpAddressVersion": "IPv4",
    "publicIpAllocationMethod": "Static",
    "resourceGroup": "kubernetes-the-hard-way",
    "resourceGuid": "XYZ",
    "sku": {
      "name": "Basic"
    },
    "tags": null,
    "type": "Microsoft.Network/publicIPAddresses",
    "zones": null
  }
]

```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 16.04, which has good support for the [cri-containerd container runtime](https://github.com/kubernetes-incubator/cri-containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Create SSH keys

To login to our VMs we will use SSH keys. The examples here will use bash on Linux.

From a terminal run the following command to generate the key pair:

```
ssh-keygen -t rsa -b 2048 -f kubernetes-the-hard-way 

```

If you need more assistance please refer to [Detailed walk through to create an SSH key pair and additional certificates for a Linux VM in Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/create-ssh-keys-detailed). 

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  az vm create --name controller-${i} \
  --no-wait \
  --resource-group kubernetes-the-hard-way \
  --image UbuntuLTS \
  --private-ip-address 10.240.0.1${i} \
  --size Basic_A1 \
  --vnet-name kubernetes-the-hard-way \
  --subnet kubernetes \
  --nsg kubernetes-the-hard-way \
  --authentication-type ssh \
  --ssh-key-value ~/kubernetes-the-hard-way.pub \
  --public-ip-address "" \
  --tags lab=kubernetes-the-hard-way type=controller
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  az vm create --name worker-${i} \
  --no-wait \
  --resource-group kubernetes-the-hard-way \
  --image UbuntuLTS \
  --private-ip-address 10.240.0.2${i} \
  --size Basic_A1 \
  --vnet-name kubernetes-the-hard-way \
  --subnet kubernetes \
  --nsg kubernetes-the-hard-way \
  --authentication-type ssh \
  --ssh-key-value ~/kubernetes-the-hard-way.pub \
  --public-ip-address "" \
  --tags lab=kubernetes-the-hard-way type=worker pod-cidr=10.200.${i}.0/24
done
```
### Verification

List the compute instances in your default compute zone:

```
az vm list -g kubernetes-the-hard-way --show-details -o table
```

> output

```
Name          ResourceGroup            PowerState    Location
------------  -----------------------  ------------  ----------
controller-0  kubernetes-the-hard-way  VM running    westus2
controller-1  kubernetes-the-hard-way  VM running    westus2
controller-2  kubernetes-the-hard-way  VM running    westus2
worker-0      kubernetes-the-hard-way  VM running    westus2
worker-1      kubernetes-the-hard-way  VM running    westus2
worker-2      kubernetes-the-hard-way  VM running    westus2

```
Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
