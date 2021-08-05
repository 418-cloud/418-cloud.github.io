---
title: "Provision GKE Cluster using Terraform adn Github Actions"
date: 2021-08-05T20:51:00+02:00
draft: true
tags: [terraform,gke,github actions,gh-actions,github,gh,tf,google,cloud]
---

I started my career as a software developer writing small monolithic web applications and web services in Java.

Later the monoliths where replaced by microservices, nanoservice or functions. No matter what you are writing or calling it you will eventually need some servers to run it on.

Yes even serverless at some point require servers, it's just someone elses server.

As a developer at heart I stribe to automate as much as possible through code, even the server.

In this post I will walk through how to provision a GKE cluster using terraform and Github Actions.

#### Terraform

Terraform is a tool for definining infrastructure as code for hundreds of cloud services. It provides a consistent way of defining infra structure and a singel CLI tool to provision it all.

#### Github Actions

Github Actions is Githubs solution for defining workflows directly in your Github repository. The workflows can be triggered on events like pushes, pull requests, merges in Github.

### Prerequisites

1. [GCP Account](https://cloud.google.com/)
2. [Github Account](https://github.com/)
3. [gcloud cli](https://cloud.google.com/sdk/gcloud)

### Demo sources
If you want to fork the repository containing the code described in this post you can find it here: [418-cloud/terraform-actions-gcp](https://github.com/418-cloud/terraform-actions-gcp)

### Prepare credentials to GCP for Terraform

As we are going to run terraform outside of google cloud we need a way for terraform google cloud provider to authenticate.

#### Create service account

Following the GCP documentation [Creating and managing service accounts](https://cloud.google.com/iam/docs/creating-managing-service-accounts)

After authenticating with gcloud comandline tool run this command to create a service account with the name _tf-gh-actions_

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

### Setting up Github Actions workflow

Workflows in Actions are defined as yaml-fiels (sweet, sweet yaml) in the fold .github/workflows folder in the repository.

