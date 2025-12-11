# Step-by-Step Guide: Deploying a Service to AWS EKS

Authors: [Wishwa Hettige](https://github.com/wishwa14), Mina Demian

**Estimated Time**
- 🕐 First time: 2-3 hours
- ⚡ Subsequent deploys: 30 minutes

**Quick Start**
- 🆕 Need to deploy a new service? → [Start at Part 1](#part-1-set-up-aws-infrastructure-with-terraform)
- 🔄 Updating existing service? → [Jump to Part 5](#step-52-run-github-actions-workflow)
- ⚠️ Deployment failed? → [Go to Part 7 Troubleshooting](#part-7-troubleshooting-common-issues)

---

## Prerequisites

Before starting, ensure you have:
- A repository ready
- EKS namespace created for deployment
- AWS account with appropriate permissions (EKS, ECR, IAM)
- GitHub repository access with Actions enabled
- Local development environment with:
  - Docker installed
  - AWS CLI configured (`aws configure`)
  - `kubectl` installed
  - Helm 3.x installed
  - Terraform installed
  - GitHub CLI (`gh`) installed

---

## Architecture Overview

We'll build a complete CI/CD pipeline that:

1. **Terraform** provisions AWS infrastructure (ECR, IAM roles)
2. **Docker** packages the Python application
3. **GitHub Actions** automates build and deployment
4. **Helm** deploys to Kubernetes
5. **EKS** runs the containerized application

**Data Flow:**
```
Developer → Git Push → GitHub Actions → Docker Build → ECR → Helm Deploy → EKS Pod
```

---

## Prerequisite: Create Your EKS Namespace

> If you haven't already, [set up your CLI environment](https://cabonline.atlassian.net/wiki/spaces/CID/pages/4093116423/Working+with+AWS+Kubernetes).

```bash
kubectl create namespace flygtaxi-dev
```

**Verify:**
```bash
kubectl get namespaces | grep flygtaxi-dev
```

## Part 1: Set Up AWS Infrastructure with Terraform

We provision AWS infrastructure resources with [Atlantis](https://www.runatlantis.io/) to automate Terraform workflows. This is done by connecting your GitHub repository with the Atlantis webhook.

**Workflow**
```
PR with Terraform resources → Git Push → Webhook fired -> GitHub Actions → Manual review and approval by Platform team → Run Atlantis commands in PR comment → Helm Deploy
```
### Step 1.1: Prepare the Infrastructure Resource Bundle

1. Identify to which environments your service needs to be deployed. In this case, we will deploy to the `dev` environment in `eu-north-1` region.
2. Create a new directory to set up the ECR registry::
```bash
$ mkdir -p terraform/build/eu-north-1
$ cd terraform/build/eu-north-1
```

3. Create the following Terraform files:

**Create `ecr.tf`:**
```terraform
module "dev_cabonline_serviceasonestring_ecr" {
  source = "git@github.com:CabonlineTeam/Infra-Terraform-Resources.git//_modules/terraform-aws-cabonline-ecr"
  name   = "cabonline-service-in-kebab-case"
  env    = "dev"
}
```

**Create `irsa.tf`:**
```terraform
resource "aws_iam_policy" "additional" {
  name        = "dev-cabonline-service-additional"
  description = "Additional test policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "rds:*",
          "ec2:*",
          "ssm:GetParameter",
          "ssm:GetParameters",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sns:Publish"
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]
  })
}

module "irsa_role" {
  source       = "git@github.com:CabonlineTeam/Infra-Terraform-Resources.git//_modules/terraform-aws-cabonline-irsa"
  eks_name     = "dev-col-eu-north-1-eks"
  namespace    = "<EKS namespace>"
  policy       = aws_iam_policy.additional.arn
  name         = "cabonline-service"
  env          = "dev"
  sa_name      = "service-account-name"
}
```

**Create `providers.tf`**
```terraform
provider "aws" {
  region = "eu-north-1"

  assume_role {
    role_arn = "arn:aws:iam::165660931457:role/TerraformAdminRole"
  }

  default_tags {
    tags = {
      BackupPolicy = "Default"
      Owner        = "Cabonline"
      Repository   = "CabonlineTeam/service"
      Service      = "ServiceInCapitalCase"
      map-migrated = "migEP5EHDU6ZD"
    }
  }
}
```

4. Create a new directory to provision the service in the intended environment. (Repeat the process for any other environments as needed.):
```bash
$ mkdir -p terraform/dev/eu-north-1
$ cd terraform/dev/eu-north-1
```

Create `irsa.tf`
```terraform
resource "aws_iam_policy" "additional" {
  name        = "dev-cabonline-service-in-kebab-case-additional"
  description = "Additional test policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "rds:*",
          "ec2:*",
          "rds-db:*",
          "ssm:PutParameter",
          "ssm:LabelParameterVersion",
          "ssm:UnlabelParameterVersion",
          "ssm:DescribeParameters",
          "ssm:GetParameterHistory",
          "sqs:receivemessage",
          "ssm:DescribeDocumentParameters",
          "ssm:GetParametersByPath",
          "ssm:GetParameters",
          "ssm:GetParameter",
          "sqs:deletemessage",
          "sns:Publish"
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]
  })
}

module "irsa_role" {
  source          = "git@github.com:CabonlineTeam/Infra-Terraform-Resources.git//_modules/terraform-aws-cabonline-irsa"
  eks_name              = "dev-col-eu-north-1-eks"
  namespace             = "flygtaxi-dev"
  policy                = aws_iam_policy.additional.arn
  name                  = "cabonline-service-in-kebab-case"
  env                   = "dev"
  sa_name               = "service"
}

```

**Create `providers.tf`:**
```terraform
provider "aws" {
  region = "eu-north-1"

  assume_role {
    role_arn = "arn:aws:iam::369171354678:role/TerraformAdminRole"
  }

  default_tags {
    tags = {
      BackupPolicy = "Default"
      Owner        = "Cabonline"
      Repository   = "CabonlineTeam/repository"
      Service      = "ServiceInCapitalCase"
      map-migrated = "migEP5EHDU6ZD"
    }
  }
}
```

**Create `terraform.tf`:**
```terraform
terraform {
  required_version = "~> 1.9"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.93.0, < 6.1.0"
    }
  }

  backend "s3" {
    bucket = "cabonline-tf-state-369171354678-eu-north-1"
    key    = "eu-north-1/flygtaxi-frontdoor-dev/terraform.tfstate"
    region = "eu-north-1"

    assume_role = {
      role_arn = "arn:aws:iam::369171354678:role/TerraformAdminRole"
    }
  }
}
```

### Step 1.2: Integrate Your Repository with Atlantis

1. Create `atlantis.yaml` in the root of your repository if it doesn't exist:
```yaml
version: 3
projects:
  - name: dev-eu-north-1
    dir: terraform/dev/eu-north-1
  - name: build-eu-north-1
    dir: terraform/build/eu-north-1
```
2. Prepare and create a Pull Request with the new Terraform files.
1. Ask a Platform team member to create the webhook in Atlantis for your repository if not already done.
1. Once the PR is created, Atlantis will pick it up. A Platform team member will review and approve the changes.
1. After approval, run the following commands in the PR comments to apply the changes:
2. 
```bash
atlantis apply
```

## Part 2: Configure Helm In Your Repository

### Step 2.1: Create Chart Structure

```bash
mkdir -p helm/templates
cd helm/templates
```

### Step 2.2: Create Chart.yaml

**Create `helm/Chart.yaml`:**
```yaml
apiVersion: v2
name: <service>
description: A Helm chart for deploying <service>
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### Step 2.3: Create Values File

**Create `helm/values-dev.yaml`:**
```yaml
name: <service name>
namespace: <EKS namespace you'll be deploying to>
replicaCount: 1
region: eu-north-1

domain: dev-internaledge.cabonline.io
suburl: <service name>
externaldomain:
  enabled: true
  domain: api.dev.cabonline.io

image:
  repository: 165660931457.dkr.ecr.eu-north-1.amazonaws.com/<ECR repository>:<service name>-latest
  pullPolicy: Always

port: 8080

service:
  type: LoadBalancer
  port: 80

serviceAccount:
  enabled: true
  name: <service name>
  create: true
  annotations:
    eks.amazonaws.com/role-arn: <IAM role ARN from Terraform output>*

availabilityZones:
  - eu-north-1a
  - eu-north-1b
  - eu-north-1c

resources:
  limits:
    cpu: 500m
    memory: 756Mi
  requests:
    cpu: 100m
    memory: 128Mi

env: # add your own environment variables here
  - name: ENV_VAR_NAME
    value: VALUE
```

> * Replace the IAM role ARN with the one from your Terraform output in the Atlantis PR you merged earlier.

### Step 2.4: Create Deployment Template

> If you're deploying a Java application, you can reuse the standard templates in the [cabonline-helm](https://github.com/CabonlineTeam/cabonline-helm) repository. Otherwise, you'll have to override the deployment template as shown in the [Appendix](#Appendix). Same applies to other templates like service, service account, etc.

### Step 2.5: Validate Helm Chart

If you want to validate locally the Helm chart, you can run the following commands:

```bash
# Lint the chart
helm lint ./helm

# Template to see rendered YAML
helm template test-release ./helm \
  --values ./helm/values-dev.yaml

# Check for ServiceAccount
helm template test-release ./helm \
  --values ./helm/values-dev.yaml | grep -A 5 "kind: ServiceAccount"
```

**Expected Output:**
```yaml
kind: ServiceAccount
metadata:
  name: flygtaxi-dev-frontdoor
  namespace: flygtaxi-dev
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::369171354678:role/...
```

---

## Part 3: Set Up GitHub Actions

### Step 3.1: Configure AWS OIDC for GitHub Actions

GitHub Actions needs permission to push to ECR and deploy to EKS. Ask a Platform team member to set up OIDC trust between GitHub and AWS for your repository.

### Step 3.2: Create Build & Push Workflow

> The following sections are suggested only. Adapt as needed for your repository structure and naming conventions.

**Create `.github/workflows/build-push-ecr.yaml`:**
```yaml
name: Build and Push to ECR

on:
  workflow_call:
    inputs:
      image:
        required: true
        type: string
      app_dir_path:
        required: true
        type: string
      environment:
        required: true
        type: string
      branch:
        required: true
        type: string
      aws_region:
        required: true
        type: string
      ecr_repository:
        required: true
        type: string
      runner:
        required: false
        type: string
        default: "ubuntu-latest"
    outputs:
      image_tag:
        value: ${{ jobs.build-and-push.outputs.image_tag }}
      ecr_registry:
        value: ${{ jobs.build-and-push.outputs.ecr_registry }}

permissions:
  contents: read
  id-token: write

jobs:
  build-and-push:
    runs-on: ${{ inputs.runner }}
    outputs:
      image_tag: ${{ steps.set-outputs.outputs.image_tag }}
      ecr_registry: ${{ steps.login-ecr.outputs.registry }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ inputs.branch }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::165660931457:role/GithubActions-GithubActionsManagementRole
          aws-region: ${{ inputs.aws_region }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{ inputs.ecr_repository }}
          IMAGE_NAME: ${{ inputs.image }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          echo "Building image: $IMAGE_NAME"
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-${IMAGE_TAG} \
            -f ${{ inputs.app_dir_path }}/Dockerfile \
            ${{ inputs.app_dir_path }}
          
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-${IMAGE_TAG}
          
          # Tag as latest
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-${IMAGE_TAG} \
            $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-latest
          
          echo "Successfully pushed: $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-${IMAGE_TAG}"

      - name: Set outputs
        id: set-outputs
        run: echo "image_tag=${{ github.sha }}" >> $GITHUB_OUTPUT
```

### Step 3.3: Create Deploy Workflow

**Create `.github/workflows/build-deploy-eks.yaml`:**
```yaml
name: Deploy to EKS

on:
  workflow_call:
    inputs:
      image:
        required: true
        type: string
      image_tag:
        required: true
        type: string
      ecr_registry:
        required: true
        type: string
      ecr_repository:
        required: true
        type: string
      environment:
        required: true
        type: string
      branch:
        required: true
        type: string
      aws_region:
        required: true
        type: string
      deploy_role_arn:
        required: true
        type: string
      eks_cluster_name:
        required: true
        type: string
      eks_cluster_alias:
        required: true
        type: string
      eks_cluster_arn:
        required: true
        type: string
      eks_namespace:
        required: true
        type: string
      helm_namespace:
        required: true
        type: string
      chart_dir:
        required: true
        type: string
      runner:
        required: false
        type: string
        default: "eks-dev"
    secrets:
      HELM_TOKEN:
        required: true

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ${{ inputs.runner }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ inputs.branch }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ inputs.deploy_role_arn }}
          aws-region: ${{ inputs.aws_region }}

      - name: Connect to EKS cluster
        run: |
          aws eks update-kubeconfig \
            --region ${{ inputs.aws_region }} \
            --name ${{ inputs.eks_cluster_name }} \
            --alias ${{ inputs.eks_cluster_alias }}
          
          kubectl cluster-info
          kubectl get nodes

      - name: Deploy with Helm
        uses: ./.github/workflows/common-actions/deploy-to-eks-with-helm
        with:
          chart_dir: ${{ inputs.chart_dir }}
          release_name: ${{ inputs.image }}
          helm_namespace: ${{ inputs.helm_namespace }}
          image_tag: ${{ inputs.image_tag }}
          ecr_registry: ${{ inputs.ecr_registry }}
          ecr_repository: ${{ inputs.ecr_repository }}
          image: ${{ inputs.image }}
          environment: ${{ inputs.environment }}
          token: ${{ secrets.HELM_TOKEN }}
```

### Step 3.4: Create Custom Helm Deploy Action

**Create `.github/workflows/common-actions/deploy-to-eks-with-helm/action.yaml`:**
```yaml
name: Deploy to EKS with Helm
description: Deploy application to EKS using Helm

inputs:
  chart_dir:
    required: true
  release_name:
    required: true
  helm_namespace:
    required: true
  image_tag:
    required: false
  ecr_registry:
    required: false
  ecr_repository:
    required: false
  image:
    required: true
  environment:
    required: true
  token:
    required: true

runs:
  using: composite
  steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Install Helm
      uses: azure/setup-helm@v3
    
    - name: Clone templates from repo
      shell: bash
      env:
        GITHUB_TOKEN: ${{ inputs.token }}
      run: |
        git clone https://x-access-token:${GITHUB_TOKEN}@github.com/CabonlineTeam/cabonline-helm cabonline-helm
        # Only copy if files don't exist locally
        [ ! -f "${{ inputs.chart_dir }}/templates/service.yaml" ] && cp cabonline-helm/templates/service.yaml ${{ inputs.chart_dir }}/templates/ || echo "service.yaml already exists, skipping"
        [ ! -f "${{ inputs.chart_dir }}/templates/serviceaccount.yaml" ] && cp cabonline-helm/templates/serviceaccount.yaml ${{ inputs.chart_dir }}/templates/ || echo "serviceaccount.yaml already exists, skipping"
    
    - name: Ensure all YAML files were copied over
      shell: bash
      run: |
        ls ${{ inputs.chart_dir }}/templates/*.yaml && echo "[SUCCESS]: YAML files exist in helm/templates/"
    
    - name: Prep values.yaml
      shell: bash
      run: |
        cp ${{ inputs.chart_dir }}/values-dev.yaml ${{ inputs.chart_dir }}/values.yaml
    
    - name: Package Helm chart
      shell: bash
      run: |
        echo "Templates before packaging:"
        ls -la ${{ inputs.chart_dir }}/templates/
        helm lint ${{ inputs.chart_dir }} --set image.repository=${{ inputs.image }}
        helm package ${{ inputs.chart_dir }} --destination packaged
    
    - name: Verify packaged chart contents
      shell: bash
      run: |
        PACKAGED_CHART=$(ls packaged/*.tgz | head -n 1)
        echo "Packaged chart: ${PACKAGED_CHART}"
        echo "Contents of packaged chart:"
        tar -tzf ${PACKAGED_CHART} | grep templates/
    
    - name: Dry-run helm template to verify resources
      shell: bash
      run: |
        PACKAGED_CHART=$(ls packaged/*.tgz | head -n 1)
        echo "===== Rendering templates to verify ServiceAccount ====="
        helm template test-release ${PACKAGED_CHART} \
          --namespace ${{ inputs.helm_namespace }} \
          --values ${{ inputs.chart_dir }}/values.yaml | grep -A 10 "kind: ServiceAccount" || echo "WARNING: ServiceAccount not found in rendered templates!"
    
    - name: Deploy to ${{ inputs.environment }}
      shell: bash
      run: |
        # Find the packaged chart
        PACKAGED_CHART=$(ls packaged/*.tgz | head -n 1)
        echo "Using packaged chart: ${PACKAGED_CHART}"
        
        HELM_CMD="helm upgrade --install ${{ inputs.release_name }} ${PACKAGED_CHART} \
          --namespace ${{ inputs.helm_namespace }} \
          --values ${{ inputs.chart_dir }}/values.yaml"
        
        # If specific image tag is provided, override the image repository
        if [ -n "${{ inputs.image_tag }}" ] && [ -n "${{ inputs.ecr_registry }}" ] && [ -n "${{ inputs.ecr_repository }}" ]; then
          IMAGE_FULL="${{ inputs.ecr_registry }}/${{ inputs.ecr_repository }}:${{ inputs.image }}-${{ inputs.image_tag }}"
          echo "Deploying with specific image: ${IMAGE_FULL}"
          HELM_CMD="${HELM_CMD} --set image.repository=${IMAGE_FULL}"
        fi
        
        eval $HELM_CMD
```

### Step 3.5: Create Main Orchestrator Workflow

**Create `.github/workflows/deploy-my-service.yaml`:**
```yaml
name: Build and deploy Flygtaxi Dev Frontdoor to EKS

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "dev"
        type: string
      branch:
        description: "Git branch to deploy"
        required: true
        default: "main"
        type: string
      runner:
        description: "Runner to use"
        required: false
        default: "eks-dev"
        type: string

permissions:
  contents: read
  id-token: write

jobs:
  build-and-push-frontdoor:
    uses: "./.github/workflows/build-push-ecr.yaml"
    with:
      image: "flygtaxi-dev-frontdoor"
      app_dir_path: "./common-apps/frontdoor/"
      environment: ${{ github.event.inputs.environment }}
      branch: ${{ github.event.inputs.branch || github.ref_name }}
      aws_region: "eu-north-1"
      ecr_repository: "eks-dev-eu-north-1"
      runner: ${{ github.event.inputs.runner }}
  
  deploy-eks:
    uses: "./.github/workflows/build-deploy-eks.yaml"
    needs: build-and-push-frontdoor
    with:
      image: "<service name>"
      image_tag: ${{ needs.build-and-push-frontdoor.outputs.image_tag }}
      ecr_registry: ${{ needs.build-and-push-frontdoor.outputs.ecr_registry }}
      ecr_repository: "<ecr repository>"
      environment: ${{ github.event.inputs.environment }}
      branch: ${{ github.event.inputs.branch || github.ref_name }}
      aws_region: "eu-north-1"
      deploy_role_arn: "<your AWS role ARN for deployment>"
      eks_cluster_name: "<EKS cluster name>"
      eks_cluster_alias: "<EKS cluster alias>"
      eks_cluster_arn: "<EKS cluster ARN>"
      eks_namespace: "<EKS namespace>"
      helm_namespace: "<EKS namespace>"
      chart_dir: "path/to/helm/chart"
      runner: ${{ github.event.inputs.runner }}
    secrets:
      HELM_TOKEN: ${{ secrets.HELM_TOKEN }}
```

### Step 3.6: Configure GitHub Secrets

In your GitHub repository:

1. Go to Settings → Secrets and variables → Actions
2. Add `HELM_TOKEN`:
   - Create a GitHub Personal Access Token with `repo` scope
   - Add as repository secret named `HELM_TOKEN`

---

### Step 5.2: Run GitHub Actions Workflow

**Option 1: Via GitHub UI**
1. Go to your repository on GitHub
2. Click "Actions" tab
3. Select "Build and deploy Flygtaxi Dev Frontdoor to EKS"
4. Click "Run workflow"
5. Fill in inputs:
   - Environment: `dev`
   - Branch: `main`
   - Runner: `eks-dev`
6. Click "Run workflow"

**Option 2: Via GitHub CLI**
```bash
gh workflow run deploy-my-service.yaml \
  -f environment=dev \
  -f branch=main \
  -f runner=eks-dev
```

**Watch the workflow:**
```bash
gh run watch
```

### Step 5.3: Verify Deployment

**Check deployment status:**
```bash
kubectl get deployment -n flygtaxi-dev flygtaxi-dev-frontdoor
```

**Expected Output:**
```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
<service name>             1/1     1            1           2m
```

**Check pods:**
```bash
kubectl get pods -n <EKS namespace> -l app=<service name> 
```

**Expected Output:**
```
NAME                                      READY   STATUS    RESTARTS   AGE
<service name>-c8c59c859-abc12             1/1     Running   0          2m
```

**Check logs:**
```bash
kubectl logs -n <EKS namespace> deployment/<service name>
```

**Check service:**
```bash
kubectl get svc -n <EKS namespace> <service name>
```

**Check ServiceAccount:**
```bash
kubectl get sa -n <EKS namespace> <service name> -o yaml
```

**Verify IRSA annotation exists:**
```bash
kubectl get sa -n <EKS namespace> <service name> -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'
```

**Expected Output:**
```
arn:aws:iam::369171354678:role/cabonline-service-role/dev-cabonline-flygtaxi-frontdoor-dev-eu-north-1
```

---

## Part 7: Troubleshooting Common Issues

### Issue 1: Pod Not Starting

**Symptom:**
```bash
kubectl get pods -n flygtaxi-dev
NAME                                      READY   STATUS             RESTARTS   AGE
flygtaxi-dev-frontdoor-c8c59c859-abc12   0/1     CrashLoopBackOff   3          2m
```

**Diagnosis:**
```bash
# Check logs
kubectl logs -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor

# Check events
kubectl describe pod -n flygtaxi-dev -l app=flygtaxi-dev-frontdoor
```

**Common Causes:**
- Port mismatch: Check `CONTAINER_PORT` env var matches Dockerfile `EXPOSE`
- Missing dependencies: Verify Dockerfile has all required packages
- Startup timeout: Increase `initialDelaySeconds` in readiness probe

### Issue 2: ServiceAccount Not Found

**Symptom:**
```
Error: pods "flygtaxi-dev-frontdoor-xxx" is forbidden: error looking up service account
```

**Solution:**
```bash
# Verify ServiceAccount exists
kubectl get sa -n flygtaxi-dev flygtaxi-dev-frontdoor

# If missing, check Helm release
helm get manifest -n flygtaxi-dev flygtaxi-dev-frontdoor | grep -A 10 "kind: ServiceAccount"

# Manually create if needed
kubectl create serviceaccount flygtaxi-dev-frontdoor -n flygtaxi-dev
kubectl annotate serviceaccount flygtaxi-dev-frontdoor -n flygtaxi-dev \
  eks.amazonaws.com/role-arn=arn:aws:iam::369171354678:role/cabonline-service-role/dev-cabonline-flygtaxi-frontdoor-dev-eu-north-1
```

### Issue 3: Image Pull Errors

**Symptom:**
```
Failed to pull image: unauthorized or not found
```

**Solution:**
```bash
# Verify image exists in ECR
aws ecr describe-images \
  --repository-name eks-dev-eu-north-1 \
  --region eu-north-1 \
  --query 'imageDetails[*].[imageTags[0]]' \
  --output table

# Check EKS node role has ECR pull permissions
aws iam list-attached-role-policies \
  --role-name <eks-node-role-name>

# Verify image pull policy
kubectl get deployment -n flygtaxi-dev flygtaxi-dev-frontdoor -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'
```

### Issue 4: Health Check Failures

**Symptom:**
```
Readiness probe failed: Get "http://10.x.x.x:8080/": context deadline exceeded
```

**Solution:**
```bash
# Check if app is listening on correct port
kubectl exec -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor -- netstat -tlnp

# Test endpoint manually
kubectl exec -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor -- curl localhost:8080/health

# Check logs for errors
kubectl logs -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor | grep -i error
```

### Issue 5: Deployment Updates Not Reflecting

**Symptom:**
Old code still running after deployment

**Solution:**
```bash
# Force pod restart
kubectl rollout restart deployment -n flygtaxi-dev flygtaxi-dev-frontdoor

# Check rollout status
kubectl rollout status deployment -n flygtaxi-dev flygtaxi-dev-frontdoor

# Verify image tag
kubectl get deployment -n flygtaxi-dev flygtaxi-dev-frontdoor -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---
## Appendix: Custom Deployment Template Example
**Create `helm/templates/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.name }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: {{ .Values.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.name }}
    spec:
      serviceAccountName: {{ .Values.serviceAccount.name }}
      {{- if .Values.availabilityZones }}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: {{ .Values.name }}
      {{- end }}
      containers:
        - name: {{ .Values.name }}
          image: "{{ .Values.image.repository }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.port }}
              protocol: TCP
          readinessProbe:
            httpGet:
              path: /
              port: {{ .Values.port }}
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: {{ .Values.port }}
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          env:
            - name: HOME
              value: "/tmp"
            - name: AWS_REGION
              value: {{ .Values.region }}
            - name: CONTAINER_PORT
              value: {{ .Values.port | quote }}
          {{- if .Values.env }}
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
          {{- end }}
          {{- if .Values.resources }}
          resources:
            requests:
              cpu: {{ .Values.resources.requests.cpu | default "100m" | quote }}
              memory: {{ .Values.resources.requests.memory | default "128Mi" | quote }}
            limits:
              cpu: {{ .Values.resources.limits.cpu | default "500m" | quote }}
              memory: {{ .Values.resources.limits.memory | default "756Mi" | quote }}
          {{- end }}
```

**Create `helm/templates/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.port }}
      protocol: TCP
  selector:
    app: {{ .Values.name }}
```
**Create `helm/templates/serviceaccount.yaml`:**
```yaml
{{- if .Values.serviceAccount.enabled }}
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.serviceAccount.name }}
  namespace: {{ .Values.namespace }}
  {{- if .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml .Values.serviceAccount.annotations | nindent 4 }}
  {{- end }}
{{- end }}
{{- end }}
```

**Create `helm/templates/frontdoor-pdb.yaml`:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ .Values.name }}-pdb
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.name }}
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: {{ .Values.name }}
```

## Additional Resources

- [AWS EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
