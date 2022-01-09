---
title: "Provision AKS Cluster using Terraform and Github Actions"
date: 2022-01-08T13:21:57+01:00
draft: false
toc: true
images:
tags: [terraform,aks,github actions,gh-actions,gh,tf,azure,cloud]
---

In my previous post [Provision GKE Cluster using Terraform and Github Actions]({{< ref "terraform-gke.md" >}}) I tried to show how you could use Github actions and terraform to provision a [GKE](https://cloud.google.com/kubernetes-engine) cluster in [GCP](https://cloud.google.com/).

In this post I will do the same only this time the goal is to create a [AKS](https://azure.microsoft.com/services/kubernetes-service/) cluster in [Azure](https://azure.microsoft.com/) and what changes needed from the scripts used to provision a GKE cluster.

### Prerequisites

1. [Azure Account](https://azure.microsoft.com/en-gb/free/)
2. [Github Account](https://github.com/)
3. [Azure cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

### Demo sources
If you want to fork the repository containing the code described in this post you can find it here: [418-cloud/terraform-actions-azure](https://github.com/418-cloud/terraform-actions-azure)

### Prepare credentials and state backend in Azure for Terraform

Terraform needs credentials to authenticate to Azure and a place to store the state.

#### Create a service principal

Following the Azure documentation [Create an Azure service principal with the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli)

Authenticate with azure cli

```shell
az login
```

Switch to the Subscription you are going to use

```shell
az account set --subscription "<subscription-uuid/name>"
```

After authenticating and switching to the desired Subscription run this command to create a Service Principal with the name _tf-gh-actions_ and grant it Contributor permissions to the Subscription
This command will output the credentials for the Service Principal we will need these later, but keeps these safe and secret

```shell
az ad sp create-for-rbac --name tf-gh-actions --role Contributor
```

#### Create a StorageAccount and container for terraform state

Create a ResourceGroup for the StorageAccount we are going to create

The following command will create a ResourceGroup with the name _tf-gh-actions-storage_ in _westeurope_

```shell
az group create \
  --name tf-gh-actions-storage \
  --location westeurope
```

Create the StorageAccount using azure CLI named _tfghdemostorage_ in _westeurope_ 

```shell
az storage account create \
  --name tfghdemostorage \
  --resource-group tf-gh-actions-storage \
  --location westeurope \
  --sku Standard_LRS \
  --kind StorageV2
```

Create a blob container named _infrastate_ in the StorageAccount where we will tell terraform to store the state file.

```shell
az storage container create \
  --name infrastate \
  --account-name tfghdemostorage
```

If you created the storageaccount in another Subscription or did not grant the ServicePrincipal Contributor you might need to grant permissions to the ServicePrincipal.

### Setting up Github Actions workflow

This section is very similar to what I did in my previous post. If you want a detailed walkthrough of the workflows in Github actions please read the [previous post]({{< ref "terraform-gke.md" >}}).

Here I will try to focus on the differences as this will point out the benefits of using Terraform as a tool to provision infrastructure.

#### Add the Azure credentials as secrets for Github actions

Credentials should never be commited to a git repository so we will add them as secrets and use these from our Github actions.

Secrets are located under the _Settings_ tab in your repository.

We are going to create four secrets: 
* __ARM_CLIENT_ID__ 
    UUID of the service principal (appId from the create service principal command output)
* __ARM_CLIENT_SECRET__ password for the service principal (password from the create service principal command output)
* __ARM_SUBSCRIPTION_ID__ UUID of the subscription where we are going to create the AKS cluster
* __ARM_TENANT_ID__ UUID of the tenant where the service principal was created (tenant from the create service principal command output)

### Create terraform workflows

We are going the copy/past the workflows from the [418-cloud/terraform-actions-gcp](https://github.com/418-cloud/terraform-actions-gcp) repository and make some small changes.

We are going to remove the two environment variables for GCP and add four environment variables for the secrets we defined in the previous step:

.github/workflows/main.yaml
```yaml
name: Apply infrastructure
on:
  push:
    branches: [main]

jobs:
  terraform:
    name: Run terraform apply
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform
    env:
      ARM_CLIENT_ID: ${{secrets.ARM_CLIENT_ID}}
      ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
```

We do the same changes to the other workflows
* .github/workflows/review.yaml
* .github/workflows/destroy.yaml

### Defining Azure infrastructure with terraform

We have now made all the preparations and all the changes we need for the automation to work. Now it's time to define the infrastructure in Azure.

In my last post we used the "google" provider, in this post we are going to substitute the provider and resources with the "azurerm" provider.

Terraform providers can be viewed as plugins, we are going to remove the google plugin and add the azurerm plugin.

Resources are still defined as hcl but the resources have different name and attributes.

#### Defining provider and backend

Once again, I will copy/past most of these example from terraforms great documentation for its providers. In this case the [azurerm provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)

Create a file named `main.tf` this will hold the configuration and requirements for our provider

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "2.91.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "tf-gh-actions-storage"
    storage_account_name = "tfghdemostorage"
    container_name       = "infrastate"
    key                  = "demo.tfstate"
  }
}

provider "azurerm" {
  features {}
}
```

The hcl block `terraform` defines the providers and version we require and the backend where we are going to store the state.

The hcl block `provider "azurerm"` block defines the actual provider with config, we have no additional config so our is empty (we need to define a empty features block as it is required).

#### Defining your Azure infrastructure

Finally, we are again coming to the steps where we actually can define our infrastructure.

We create a file named "azure.tf" where we will define the hcl describing the azure infrastructure. The name of the file is not important, but as with all software files it should reflect what you are creating in it.

```hcl
resource "azurerm_resource_group" "teapot_demo" {
  name     = "teapot-demo"
  location = "westeurope"
}

resource "azurerm_kubernetes_cluster" "teapot_demo" {
  name                = "teapot-demo"
  location            = azurerm_resource_group.teapot_demo.location
  resource_group_name = azurerm_resource_group.teapot_demo.name
  dns_prefix          = "teapotaks"

  default_node_pool {
    name       = "default"
    node_count = 1
    vm_size    = "Standard_D2_v2"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = {
    Environment = "Demo"
  }
}
```

This script will create a resourcegroup name `418-demo` and a AKS cluster named `418-demo`in this resourcegroup.

#### Making changes to your infrastructure

Given that you have committed the files over to your main branch and no errors occurred you should now have a 1 node AKS cluster.

At some point you will need to make changes to your infrastructure, and this is where terraform really shines in my mind. To make changes we can now make a commit and if we want to, we can review the changes in the configuration and the changes this is going to change in the cloud.

The review pipeline we create earlier will output what will change in the cloud if we make changes to the config in a PR.

Let's change the number of nodes in the AKS cluster commit it to a new branch and issue a PR to main and see what terraform and the review pipeline outputs.

We change [line 14 in terraform/azure.tf](https://github.com/418-cloud/terraform-actions-azure/blob/main/terraform/azure.tf#L14) file from 1 to 2, create a new branch, commit and push it to Github and issue a PR to main.

As soon as we create the PR the review workflow will start and output something similar to this:

```shell
Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # azurerm_kubernetes_cluster.teapot_demo will be updated in-place
  ~ resource "azurerm_kubernetes_cluster" "teapot_demo" {
        id                                  = "/subscriptions/***/resourceGroups/teapot-demo/providers/Microsoft.ContainerService/managedClusters/teapot-demo"
        name                                = "teapot-demo"
        tags                                = {
            "Environment" = "Demo"
        }
        # (18 unchanged attributes hidden)


      ~ default_node_pool {
            name                         = "default"
          ~ node_count                   = 1 -> 2
            tags                         = {}
            # (19 unchanged attributes hidden)
        }




        # (5 unchanged blocks hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

The output will also notify you of changes made outside of terraform, tags added and other changes. These will be updated in your statefile next time you run apply.

You can view the PR and output from the workflow [here](https://github.com/418-cloud/terraform-actions-azure/pull/1).

### Closing notes

The terraform configuration for Azure is different from my GCP example, but we could combine the two repos and manage both infrastructures from one repository.

This along with the declarative nature of terraform is the real value with it. I can clearly define my desired state in multiple providers in one repository and make changes to them as I need.

There is still many other ways to automate your terraform setup like [terraform cloud](https://learn.hashicorp.com/tutorials/terraform/github-actions)

To save money you need to remember to destroy all the resources you have created in this demo. I have added a destroy pipeline in the example repository that runs every evening, this pipeline can also be triggered manually.

Once again I hope you found it interesting, learned something and/or inspired you. Reach out on twitter or Github if you have any questions.