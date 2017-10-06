# Prerequisites

This tutorial covers the deployment and bootstraping of Kubernetes on [Microsoft Azure](https://azure.microsoft.com). You can create your Azure account for [free](https://azure.microsoft.com/en-us/free/) with $200 worth in credits. 

> The compute resources required for this tutorial exceed Azure's free tier.

## Microsoft Azure CLI 2.0

### Install the Microsoft Azure CLI 2.0

Follow the Azure CLI 2.0 [documentation](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) to install the `az` command line utility. You can install utility in various platforms such as macOS, Windows, Linux (various distros) and as a Docker container.

The examples here are based on the version 2.0.18 of the utility. You can verify the version of the tool by running:

```
az --version
```

> Note: If this is your first time using the Azure CLI tool, you can familiarize yourself with it's syntax and command options by running `az interactive` 

### First Things First

Before we can use Azure, your first step, aside from having an account, is to login. Once Azure CLI is installed, open up a terminal and run the following:

```
az login
```

This will prompt you to sign in using a web browser to https://aka.ms/devicelogin and to enter the displayed code. This single step is needed in order to allow `az` to talk back to Azure.

### Set a Default Location For the Resources

As we stand up the resources needed to support our Kubernetes cluster, Azure will created these  in one of the available locations throughout the world.

To fetch a list of all of the current available locations:

```
az account list-locations
```

With this list, you can set a default location that is, for example, closer to you:

```
az configure --default location=westus2
```

Now that we have selected a default location for our resources, let's go ahead and create a resource group. 

```
az group create --name kubernetes-the-hard-way
```

### Virtual Network

In this section a dedicated [Virtual Network](https://docs.microsoft.com/en-us/azure/virtual-network/) (VNet) network will be setup to host the Kubernetes cluster. We will carve out a subnet to be used as our management network while leaving enough IPs for Kubernetes.

Create the `kubernetes-the-hard-way` VNet network and a subnet named `kubernetes`:

```
az network vnet create \
  --name kubernetes-the-hard-way \
  --resource-group kubernetes-the-hard-way \
  --location westus2 \
  --address-prefix 10.240.0.0/16 \
  --subnet-name kubernetes-mgmt \
  --subnet-prefix 10.240.254.128/25
```
### Firewall Rules for the Management Subnet

Create a Network Security Group for out management network:

```
az network nsg create \
  --name kubernetes-the-hard-way-mgmt \
  --resource-group kubernetes-the-hard-way 
```

Create a firewall rule that allows external SSH:

```
az network nsg rule create \
  --name kubernetes-the-hard-way-allow-jumpbox \
  --nsg-name kubernetes-the-hard-way-mgmt \
  --resource-group kubernetes-the-hard-way \
  --priority 100 \
  --protocol Tcp \
  --destination-port-ranges 22 \
  --destination-address-prefixes VirtualNetwork
```

### Jumpbox host

With the VNet created and the firewall rules in place on our NSG, we are now ready to create a jumpbox. The jumpbox host will provide us access to our Kubernetes cluster that would otherwise be unaccessible to us as they do not have a public IP address. The jumpbox on the other hand will a public IP address.

### Create SSH keys

To login to our jumpbox we will use SSH keys. The examples here will use bash on Linux.

From a terminal run the following command to generate the key pair:

```
ssh-keygen -t rsa -b 2048 -f kubernetes-the-hard-way-jumpbox
```

If you need more assistance please refer to [Detailed walk through to create an SSH key pair and additional certificates for a Linux VM in Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/create-ssh-keys-detailed)

Create the jumpbox:

```
az vm create --name jumpbox \
  --no-wait \
  --resource-group kubernetes-the-hard-way \
  --image UbuntuLTS \
  --private-ip-address 10.240.254.250 \
  --size Basic_A0 \
  --vnet-name kubernetes-the-hard-way \
  --subnet kubernetes-mgmt \
  --nsg kubernetes-the-hard-way \
  --authentication-type ssh \
  --ssh-key-value ./kubernetes-the-hard-way-jumpbox.pub \
  --public-ip-address "kubernetes-the-hard-way-jumpbox" \
  --tags lab=kubernetes-the-hard-way type=jumpbox
```

### Verification

List the compute instances in the resource group and check that the jumpbox was created:

```
az vm list -g kubernetes-the-hard-way --show-details -o table
```

> output
```
Name     ResourceGroup            PowerState    PublicIps      Location
-------  -----------------------  ------------  -------------  ----------
jumpbox  kubernetes-the-hard-way  VM running    XX.XXX.XX.XXX  westus2
```

Next: [Installing the Client Tools](02-client-tools.md)