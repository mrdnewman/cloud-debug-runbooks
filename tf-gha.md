

repo/
 ├── terraform/
 │    ├── main.tf
 │    ├── variables.tf
 │    ├── outputs.tf
 │    └── backend.tf
 │
 ├── environments/
 │    ├── staging/
 │    │     └── terraform.tfvars
 │    └── prod/
 │          └── terraform.tfvars
 │
 └── .github/
      └── workflows/
           └── terraform.yml

# GitHub Actions CI Workflow

name: terraform-ci

on:
  pull_request:
  push:
    branches:
      - feature/*

permissions:
  id-token: write
  contents: read

jobs:
  terraform-ci:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: terraform

    steps:

    - name: Checkout
      uses: actions/checkout@v4

    - name: Configure AWS Credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-terraform
        aws-region: us-east-1

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3

    - name: Terraform Format
      run: terraform fmt -check -recursive

    - name: Terraform Init
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      run: terraform plan


# Deploy to Staging on Merge
# Staging Deployment

name: terraform-deploy

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: terraform

    steps:

    - name: Checkout
      uses: actions/checkout@v4

    - name: Configure AWS Credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-terraform
        aws-region: us-east-1

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan (Staging)
      run: terraform plan -var-file=environments/staging.tfvars

    - name: Terraform Apply
      run: terraform apply -auto-approve -var-file=environments/staging.tfvars

