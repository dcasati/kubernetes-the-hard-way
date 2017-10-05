# Prerequisites

This tutorial covers [Microsoft Azure](https://azure.microsoft.com). Most of the commands around Kubernets or Linux will work as-is on both cloud providers We will make a clear distiction when using examples for a specific cloud provider.

## Microsoft Azure

As a second way to illustrate how to setup a Kubernetes cluster from scratch, we will use [Microsoft Azure](https://azure.microsoft.com/). You can create your Azure account for [free](https://azure.microsoft.com/en-us/free/) with $200 worth in credits. 


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

Next: [Installing the Client Tools](02-client-tools.md)