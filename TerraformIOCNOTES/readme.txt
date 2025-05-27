Creating a clean and scalable Azure DevOps YAML pipeline for Terraform Infrastructure as Code (IaC) typically involves steps like:

Initializing Terraform

Validating configuration

Planning changes

Applying (conditionally)

Using Azure CLI or service connection authentication

✅ Recommended YAML Structure for Terraform IaC in Azure DevOps
Here’s a production-ready example that includes best practices like variable usage, approval gates, and conditional apply:

yaml
Copy
Edit
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - terraform/**

variables:
  TF_VERSION: '1.5.7'
  TF_WORKING_DIR: 'terraform'

stages:
  - stage: Terraform_Init_Validate
    displayName: 'Terraform Init & Validate'
    jobs:
      - job: InitValidate
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseTerraform@0
            inputs:
              terraformVersion: $(TF_VERSION)

          - script: |
              cd $(TF_WORKING_DIR)
              terraform init -input=false
              terraform validate
            displayName: 'Terraform Init & Validate'
            env:
              ARM_CLIENT_ID: $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
              ARM_TENANT_ID: $(ARM_TENANT_ID)

  - stage: Terraform_Plan
    displayName: 'Terraform Plan'
    dependsOn: Terraform_Init_Validate
    jobs:
      - job: Plan
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseTerraform@0
            inputs:
              terraformVersion: $(TF_VERSION)

          - script: |
              cd $(TF_WORKING_DIR)
              terraform plan -input=false -out=tfplan
            displayName: 'Terraform Plan'
            env:
              ARM_CLIENT_ID: $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
              ARM_TENANT_ID: $(ARM_TENANT_ID)

          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: '$(TF_WORKING_DIR)/tfplan'
              artifact: tfplan
              publishLocation: pipeline

  - stage: Terraform_Apply
    displayName: 'Terraform Apply'
    dependsOn: Terraform_Plan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: Apply
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - download: current
            artifact: tfplan

          - task: UseTerraform@0
            inputs:
              terraformVersion: $(TF_VERSION)

          - script: |
              cd $(TF_WORKING_DIR)
              terraform apply -input=false tfplan
            displayName: 'Terraform Apply'
            env:
              ARM_CLIENT_ID: $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
              ARM_TENANT_ID: $(ARM_TENANT_ID)
🧾 Notes & Best Practices
1. Secure Your Credentials
Use a Service Connection or set secrets in Azure DevOps pipeline variables or a secure variable group:

ARM_CLIENT_ID

ARM_CLIENT_SECRET

ARM_SUBSCRIPTION_ID

ARM_TENANT_ID

You can also use AzureCLI@2 task to log in instead of manually passing credentials.

2. Environment-Specific Configuration
For multi-env support (dev, test, prod), consider using:

TF_VAR_environment

terraform.workspace

Separate working directories or state files

3. Manual Approval for Apply
You can configure a pre-deployment approval in the Terraform_Apply stage using Azure DevOps environments for safety.

4. Folder Structure (suggested)
css
Copy
Edit
repo/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── backend.tf
├── azure-pipelines.yml
