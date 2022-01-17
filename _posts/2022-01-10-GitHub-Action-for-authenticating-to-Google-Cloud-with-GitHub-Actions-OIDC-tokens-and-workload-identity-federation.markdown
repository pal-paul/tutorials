---
layout: post
title: "GitHub Action for authenticating to Google Cloud with GitHub Actions OIDC tokens and Workload Identity Federation"
date: 2022-01-10 13:44:45 +0200
categories: Terraform
permalink: /:title/
---

## Auth
GitHub Action exchanges a GitHub Actions OIDC token into a Google Cloud access token using [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation). This obviates the need to export a long-lived Google Cloud service account key and establishes a trust delegation relationship between a particular GitHub Actions workflow invocation and permissions on Google Cloud

### Previously
* Create a Google Cloud service account and grant IAM permissions
* Export the long-lived JSON service account key
* Upload the JSON service account key to a GitHub secret

### With Workload Identity Federation
* Create a Google Cloud service account and grant IAM permissions
* Create and configure a Workload Identity Provider for GitHub
* Exchange the GitHub Actions OIDC token for a short-lived Google Cloud access token


## Prerequisites
This action requires you to create and configure a Google Cloud Workload Identity Provider

## Setup Workload Identity Provider

To exchange a GitHub Actions OIDC token for a Google Cloud access token, you must create and configure a Workload Identity Provider. 

Here terraform example how to create/configure Workload Identity Provider for Github

```
data "google_project" "project" {
  project_id = var.project
}

resource "google_iam_workload_identity_pool" "ugc_wip_gh_pool" {
  project                   = var.project
  provider                  = google-beta
  workload_identity_pool_id = "wpi-github-pool"
}

resource "google_iam_workload_identity_pool_provider" "ugc_wip_provider" {
  provider                           = google-beta
  project                            = var.project
  workload_identity_pool_id          = google_iam_workload_identity_pool.ugc_wip_gh_pool.workload_identity_pool_id
  workload_identity_pool_provider_id = "wpi-github-provider"
  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
  }
  oidc {
    allowed_audiences = [var.wip_allowed_audiences]
    issuer_uri        = "https://token.actions.githubusercontent.com"
  }
}
```
See [documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/iam_workload_identity_pool_provider)

project is your google project id
The audience is the key that gcp allows to connect from Github

The attribute mappings map claims in the GitHub Actions JWT to assertions you can make about the request (like the repository or GitHub username of the principal invoking the GitHub Action). These can be used to further restrict the authentication using attribute.repository flags.

For example, you can map the attribute repository values (which can be used later to restrict the authentication to specific repositories):

```
"google.subject"       = "assertion.sub"
"attribute.repository" = "assertion.repository"
```

The full ID for the Workload Identity Provider will be like below format:

```
projects/123456789/locations/global/workloadIdentityPools/<${var.env}-wpi-github-pool>/providers/<${var.env}-wpi-github-provider>
```

## Setup Workload Identity Provider to impersonate the Service Account

Allow authentications from the Workload Identity Provider to impersonate the Service Account created above
*Warning*: This grants access to any resource in the pool (all GitHub repos). It's strongly recommended that you map to a specific attribute such as the owner/organization or repository name instead. 

See mapping [external identities](https://cloud.google.com/iam/docs/configuring-workload-identity-federation#oidc) for more information.

Here terraform example how to create/configure service account for the Github action

```
# Create service account for ugc-support service
module "service_accounts_ugc_support" {
  source        = "terraform-google-modules/service-accounts/google"
  version       = "~> 3.0"
  project_id    = var.project
  prefix        = var.env
  names         = ["tf-ugc-support"]
  generate_keys = false
  project_roles = [
    "${var.project}=>roles/iam.serviceAccountTokenCreator",
    "${var.project}=>roles/secretmanager.secretAccessor",
    "${var.project}=>roles/secretmanager.viewer",
  ]
}

data "google_iam_policy" "ugc_support_wlif_gh_policy" {
  binding {
    role = "roles/iam.workloadIdentityUser"
    members = [
      "principalSet://iam.googleapis.com/projects/${data.google_project.project.number}/locations/global/workloadIdentityPools/${google_iam_workload_identity_pool.ugc_wip_gh_pool.workload_identity_pool_id}/attribute.repository/${var.github_owner}/${var.repo}",
    ]
  }
}
```

* *roles/iam.serviceAccountTokenCreator* role is must to create token to validate
* *roles/iam.workloadIdentityUser* role is must to use as workload identity user
* *policy* should add to connect from specific Github owner/organization or repository name

The full ID of pilicy should look like 
```
"principalSet://iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/${var.env}-wpi-github-pool/attribute.repository/${var.github_owner}/${var.repo}"
```

## Github Action using Access Token (OAuth 2.0)
Use this GitHub Action with the Workload Identity Provider ID and Service Account email. The GitHub Action will mint a GitHub OIDC token and exchange the GitHub token for a Google Cloud access token (assuming the authorization is correct). This all happens without exporting a Google Cloud service account key JSON!

```
name: Deploy - Dev
on:
  push:
    branches:
      - master

concurrency:
  group: dev-deploy-${{ github.ref }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write'
    steps:
      - name: Checkout repo
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Authenticate to Google Cloud
        uses: 'google-github-actions/auth@v0.4.4'
        with:
          token_format: 'access_token'
          workload_identity_provider: ${{ secrets.DEV_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.DEV_TF_GCP_SERVICE_ACCOUNT }}
          audience: ${{ secrets.DEV_AUDIENCES }} 

      - name: Build and deploy to dev
        run: /bin/bash ./build/package/build.bash
```

* DEV_WORKLOAD_IDENTITY_PROVIDER=projects/123456789/locations/global/workloadIdentityPools/wpi-github-pool/providers/wpi-github-provider
* DEV_TF_GCP_SERVICE_ACCOUNT=dev-tf-ugc-support@<your-project-id>.iam.gserviceaccount.com
* DEV_AUDIENCES=var.wip_allowed_audiences

See [documentation](https://github.com/google-github-actions/auth) and more [example](https://github.com/google-github-actions/auth#examples) of google-github-actions/auth