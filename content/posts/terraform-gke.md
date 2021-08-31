---
title: "Provision GKE Cluster using Terraform and Github Actions"
date: 2021-08-31T19:05:00+02:00
draft: false
tags: [terraform,gke,github actions,gh-actions,github,gh,tf,google,cloud]
---

I started my career as a software developer writing small monolithic web applications and web services in Java.

Later the monoliths where replaced by microservices, nanoservice or functions. No matter what you are writing or calling it you will eventually need some servers to run it on.

Yes even serverless at some point require servers, it's just someone else's server.

As a developer at heart I strive to automate as much as possible through code, even the server.

In this post I will walk through how to provision a GKE cluster using terraform and Github Actions.

#### Terraform

Terraform is a tool for defining infrastructure as code for hundreds of cloud services. It provides a consistent way of defining infra structure and a single CLI tool to provision it all.

#### Github Actions

Github Actions is Githubs solution for defining workflows directly in your Github repository. The workflows can be triggered on events like pushes, pull requests, merges in Github.

### Prerequisites

1. [GCP Account](https://cloud.google.com/)
2. [Github Account](https://github.com/)
3. [gcloud cli](https://cloud.google.com/sdk/gcloud)
4. [gsutil cli](https://cloud.google.com/storage/docs/gsutil)

### Demo sources
If you want to fork the repository containing the code described in this post you can find it here: [418-cloud/terraform-actions-gcp](https://github.com/418-cloud/terraform-actions-gcp)

### Prepare credentials to GCP for Terraform

As we are going to run terraform outside of google cloud we need a way for terraform google cloud provider to authenticate.

#### Create service account

Following the GCP documentation [Creating and managing service accounts](https://cloud.google.com/iam/docs/creating-managing-service-accounts)

Authenticate with gcloud cli

```
gcloud auth login
```

Switch to your project with:

```
gcloud config set project <project-id>
```

After authenticating with gcloud commandline tool run this command to create a service account with the name _tf-gh-actions_

```shell
gcloud iam service-accounts create tf-gh-actions \
    --description="Used to provision resources with terraform from github actions" \
    --display-name="terraform github actions"
```

#### Create and download service account key

Following the GCP documentation [Creating and managing service account keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)

Create a service account key to authenticate as the service account you created in the previous step run replace \<project-id> with your project id

```shell
gcloud iam service-accounts keys create tf-gh-actions-key.json \
    --iam-account=tf-gh-actions@<project-id>.iam.gserviceaccount.com
```

Running this command will create a json file name tf-gh-actions-key.json in the folder from where you ran this command. Keep this as we need it later.

#### Create storage bucket to use as terraform backend

Create a bucket to use as backend for terraform with gsutil

```shell
gsutil mb -p <project-id> -b on -l <location> --pap enforced gs://<bucketname>
```

#### Grant the service account permissions

The service account terraform uses for its backend needs the Storage Object Admin role on the bucket you created.

Grant the role with gsutil

```shell
gsutil iam ch serviceAccount:tf-gh-actions@<project-id>.iam.gserviceaccount.com:roles/storage.objectAdmin gs://<bucketname>
```

To create different resources in your project the service account also needs enugh privileges to create those.

Here is an example of how you grant the role editor in your project. Please make your own assessment of what roles you should grant to the service account.

```shell
gcloud projects add-iam-policy-binding <project-id> --member=serviceAccount:tf-gh-actions@<project-id>.iam.gserviceaccount.com --role=roles/editor
```

### Setting up Github Actions workflow

Workflows in Actions are defined as yaml-fields (sweet, sweet yaml) under the `.github/workflows` folder in the repository.

As we push our pipeline and configuration to a git repository we need a safe place to store our credentials and other secrets. Naturally github knows this and has provided a place where you can store secrets for later consumption in a workflow.

These are located in the Secrets section under the Settings tab of your repository.

![Github Settings](/images/settings-secrets.png)

#### Adding the GCP credentials to secrets

We need to make the key we created for the service account earlier available to terraform.

Create a new secret for your repository by pressing `New repository secret`

Copy the content of the tf-gh-actions-key.json and create a secret with the name GOOGLE_CREDENTIALS and past the file content as value.

Create another secret for your project-id with the name GOOGLE_PROJECT this will be the default project where your resources will be created

#### Create terraform workflow

Clone your repository to your device and create a file named `main.yaml` under the folder `.github/workflows`

The content of main.yaml:
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
      GOOGLE_CREDENTIALS: ${{secrets.GOOGLE_CREDENTIALS}}
      GOOGLE_PROJECT: ${{ secrets.GOOGLE_PROJECT }}
    steps:
      - uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -out=gcp.tfplan
          
      - name: Terraform apply
        run: terraform apply -auto-approve gcp.tfplan

```

Let's walk through the workflow as this will create resources in GCP and start costing you money once the terraform fields are created.

```yaml
name: Apply infrastructure
on:
  push:
    branches: [main]
```

This tells github to run this workflow on every push to the main branch.

I have had my fair share of discussions if push to main should result in an automatic deploy/apply to production or if it should be a manual step/gate before it actually happens. In my opinion it should, once the main branch changes your environment should to.

The value of having your infrastructure as code in a git repository reduces if you have the search through pipelineruns and commits to see what commit your latest apply to production was.

As for the manual gate part of the argument: That's the purpose of the Pull Request you create against main.

This is my opinions, at the moment of writing, you might have a different one and the arguments or use case that makes me change my mind.

Next part, configuring the VM that runs our workflow steps:

```yaml
jobs:
  terraform:
    name: Run terraform apply
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform
    env:
      GOOGLE_CREDENTIALS: ${{secrets.GOOGLE_CREDENTIALS}}
      GOOGLE_PROJECT: ${{ secrets.GOOGLE_PROJECT }}
```

We define a job with the name `Run terraform apply` it should be executed on a agent running the latest ubuntu image from github.

We set the working directory in the repository to terraform as this is where we will place our terraform files.

Lastly, we define a environment variable `GOOGLE_CREDENTIALS` and `GOOGLE_PROJECT` and give them value from the secrets we created earlier.

Now for the actual work.

```yaml
    steps:
      - uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -out=gcp.tfplan
          
      - name: Terraform apply
        run: terraform apply -auto-approve gcp.tfplan

```

Steps defines unites of work.

Each step explained:

1. Checkout the repository
2. Install and setup the latest version of terraform
3. Initialize terraform, download and prepare providers and modules
4. Check terraform formatting. Terraform can handle wrong indent, but readability for humans is better with well formnatted code (run `terraform fmt` before you push your changes)
5. Validate the terraform scripts
6. Prepare and save the terraform execution plan
7. Apply the plan from step 6

### Defining GCP infrastructure with terraform

Up until now we have only prepared the tooling and automation. Finally it's time to define our GCP infrastructure.

Disclaimer this is mostly copy/paste from [GCP terraform provider documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs).

#### Defining providers and backend

From terraform.io:

Providers are a logical abstraction of an upstream API. They are responsible for understanding API interactions and exposing resources.

Create a file named `main.tf` the file can be named whatever as long has the extension `.tf` and not ends in _override (see: [Override Files](https://www.terraform.io/docs/language/files/override.html))

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "3.82.0"
    }
  }

  backend "gcs" {
    bucket = "gh-tf-state"
    prefix = "terraform/state"
  }
}

provider "google" {
  region = "europe-north1"
  zone   = "europe-north1-a"
}
```

The hcl block `terraform` defines the required providers and version, in this case only the google provider, and the backend used for state.

The `proverder "google"` block configures the provider we are going to use. In addition, we define the default project with the environment variable `GOOGLE_PROJECT`in our pipeline. You can also define it directly in the provider config.

#### Describing your GCP infrastructure

After all this we finally are getting down to defining the actual infrastructure.

Create a file called `gke.tf` again the file can be named almost anything, but the name should reflect what it creates as it makes life easier for the maintainer(s)

```hcl
resource "google_service_account" "gke_sa" {
  account_id   = "gke-service-account"
  display_name = "GKE Service Account"
}

resource "google_container_cluster" "demo-cluster" {
  name                     = "demo-gke-cluster"
  remove_default_node_pool = true
  initial_node_count       = 1
}

resource "google_container_node_pool" "primary_preemptible_nodes" {
  name       = "gke-node-pool"
  cluster    = google_container_cluster.demo.name
  node_count = 3

  node_config {
    preemptible  = true
    machine_type = "e2-medium"

    service_account = google_service_account.gke_sa.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}
```

After all this we are only going to create three resources:
1. service_account for the gke cluster
2. gke cluster where we remove the default nodepool after creation
3. nodepool with three preemptible nodes for our workloads

Resource blocks in terraform have three parameters
1. resource
2. name of the provider resource we are going to create e.g.: `"google_container_cluster"`
3. name of the resource in your statefile e.g.: `"demo-cluster"`

I will not go further into terraform in this post. As this was ment to connect the different tools to automate your infrastructure deployments.
The terraform documentation is really good so I urge you to read it: [terraform docs](https://www.terraform.io/docs/index.html)

### Reviewing infrastructure before they are applied

As I said earlier your infrastructure should change as soon as your main branch changes, but you still need to review your changes before you apply them.

Pull requests are in my opinion the perfect place to do this, you can see the changes made in your code with the resulting changes in your infrastructure.

Let's setup the workflow that displays changes terraform are going to do when a pull request is merged.

Create a file named `review.yaml` in the folder `.github/workflows`

```yaml
name: Review infrastructure changes
on:
  pull_request:
    branches: [main]

jobs:
  terraform:
    name: Run terraform apply
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform
    env:
      GOOGLE_CREDENTIALS: ${{secrets.GOOGLE_CREDENTIALS}}
      GOOGLE_PROJECT: ${{ secrets.GOOGLE_PROJECT }}
    steps:
      - uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan
```

This is very similar to our `main.yaml` workflow.

We have change the trigger to react to pull requests against the main branch
```yaml
on:
  pull_request:
    branches: [main]
```

And we have removed the apply step. This will now only output the changes terraform are going to apply after merge to `main`

Terraform will now output the changes it is going to make in the execution log:

```shell
Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # google_container_node_pool.primary_preemptible_nodes will be updated in-place
  ~ resource "google_container_node_pool" "primary_preemptible_nodes" {
        id                  = "projects/***/locations/europe-north1-a/clusters/demo-gke-cluster/nodePools/gke-node-pool"
        name                = "gke-node-pool"
      ~ node_count          = 3 -> 4
        # (7 unchanged attributes hidden)



        # (3 unchanged blocks hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Remember that if your merge another PR or otherwise change your infra structure the plan might change, so rerun the execution if in doubt.

### Closing notes

There are several other ways to automate your terraform setup like with [terraform cloud](https://learn.hashicorp.com/tutorials/terraform/github-actions)

And remember to teardown/destroy the resources you created, in my repository I have created a workflow that is executed every day at eight in case I forget to destroy it.

It is also possible to schedule the destroy workflow manually

```yaml
name: Destroy infrastructure
on:
  workflow_dispatch:
  schedule:
    - cron: 0 20 * * * #Schedule destroy every night at eight

jobs:
  terraform:
    name: Run terraform apply -destroy
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform
    env:
      GOOGLE_CREDENTIALS: ${{secrets.GOOGLE_CREDENTIALS}}
      GOOGLE_PROJECT: ${{ secrets.GOOGLE_PROJECT }}
    steps:
      - uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -destroy -out=gcp.tfplan
          
      - name: Terraform apply
        run: terraform apply -destroy -auto-approve gcp.tfplan
```

The goal of this post was to make a simple example with a few tools to automate your terraform setup.

I hope you found it interesting, learned something or inspired you. Reach out on twitter if you have any questions.