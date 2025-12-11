# Flygtaxi CI/CD Infrastructure Setup

![Ursine Enterprises](https://avatars.githubusercontent.com/u/242738045?s=200&v=4)

**Author:** Mina Demian  
**Organization:** Ursine Enterprises  
**Last Updated:** December 9, 2025

---

## Overview

This document describes the complete infrastructure setup for deploying the Flygtaxi frontdoor service to AWS EKS using a GitOps approach. The stack combines Terraform for infrastructure provisioning, Docker for containerization, Helm for Kubernetes deployments, and GitHub Actions for CI/CD automation.

**Stack Components:**
- Terraform (IaC for AWS resources)
- Docker (containerization)
- Helm (Kubernetes package management)
- GitHub Actions (CI/CD pipeline)
- AWS EKS (Kubernetes orchestration)
- AWS ECR (container registry)

---

## Architecture

### Infrastructure Layers

```
┌─────────────────────────────────────────────────────┐
│  GitHub Actions (CI/CD)                             │
│  - Build Docker images                              │
│  - Push to ECR                                      │
│  - Deploy via Helm to EKS                           │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────┴───────────────────────────────────┐
│  AWS ECR (Container Registry)                       │
│  - Repository: eks-dev-eu-north-1                   │
│  - Images tagged with git commit SHA                │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────┴───────────────────────────────────┐
│  AWS EKS (dev-col-eu-north-1-eks)                   │
│  Namespace: flygtaxi-dev                            │
│  ├── Deployment (flygtaxi-dev-frontdoor)            │
│  ├── Service (LoadBalancer)                         │
│  ├── ServiceAccount (IRSA for AWS permissions)      │
│  ├── Ingress (ALB integration)                      │
│  └── PodDisruptionBudget                            │
└─────────────────────────────────────────────────────┘
```

### Request Flow

```
Internet → ALB Ingress → K8s Service → Pod (port 8080)
                                       ├── /health (readiness/liveness)
                                       ├── /status (full service info)
                                       └── /* (404)
```

---

## Terraform Configuration

### Directory Structure

```
terraform/
├── build/eu-north-1/         # Build-time resources
│   ├── ecr.tf                # ECR repository
│   ├── providers.tf
│   └── terraform.tf
└── dev/eu-north-1/           # Runtime resources
    ├── irsa.tf               # IAM Roles for Service Accounts
    ├── providers.tf
    └── terraform.tf
```

### ECR Repository

**File:** `terraform/build/eu-north-1/ecr.tf`

```terraform
module "dev_cabonline_flygtaxifrontdoordev_ecr" {
  source = "git@github.com:CabonlineTeam/Infra-Terraform-Resources.git//_modules/terraform-aws-cabonline-ecr"
  name   = "cabonline-flygtaxi-frontdoor-dev"
  env    = "dev"
}
```

**Provisions:**
- ECR repository: `eks-dev-eu-north-1`
- Lifecycle policies for image retention
- Cross-account pull permissions

### IRSA Configuration

**File:** `terraform/dev/eu-north-1/irsa.tf`

IAM Role for Service Account (IRSA) enables pods to assume AWS IAM roles without storing credentials.

```terraform
module "irsa_role" {
  source       = "git@github.com:CabonlineTeam/Infra-Terraform-Resources.git//_modules/terraform-aws-cabonline-irsa"
  eks_name     = "dev-col-eu-north-1-eks"
  namespace    = "flygtaxi-dev"
  policy       = aws_iam_policy.additional.arn
  name         = "cabonline-flygtaxi-frontdoor-dev"
  env          = "dev"
  sa_name      = "flygtaxi-dev-frontdoor"
}
```

**Grants permissions for:**
- RDS access
- EC2 management
- SSM Parameter Store (read/write)
- SQS message handling
- SNS publishing

**Role ARN:** `arn:aws:iam::369171354678:role/cabonline-service-role/dev-cabonline-flygtaxi-frontdoor-dev-eu-north-1`

### Terraform Workflow

```bash
# Initialize
cd terraform/build/eu-north-1
terraform init

# Plan and apply ECR
terraform plan
terraform apply

# Configure IRSA
cd ../../dev/eu-north-1
terraform init
terraform plan
terraform apply
```

---

## Docker Configuration

### Dockerfile

**Location:** `common-apps/frontdoor/Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY app.py .

ENV CONTAINER_PORT=8080

EXPOSE ${CONTAINER_PORT}

CMD ["python", "app.py"]
```

**Design Decisions:**
- `python:3.11-slim` base image for minimal footprint
- Single-file Python application (no dependencies required)
- Configurable port via `CONTAINER_PORT` environment variable
- Non-root user execution (inherited from base image)

### Application Structure

**Port Configuration:**
- Default: `8080` (configurable via `CONTAINER_PORT` env var)
- Health checks: `/`, `/health`, `/healthz`
- Status endpoint: `/status`
- Unknown paths: `404`

**Logging:**
- Output: stdout (captured by Kubernetes)
- Format: structured logging with levels (INFO, DEBUG, WARNING)
- Configurable via `LOG_LEVEL` environment variable

### Local Development

```bash
# Build
docker build -t flygtaxi-frontdoor:local .

# Run
docker run -p 8080:8080 -e LOG_LEVEL=debug flygtaxi-frontdoor:local

# Test
curl http://localhost:8080/health
curl http://localhost:8080/status
```

---

## Helm Configuration

### Chart Structure

```
helm/
├── Chart.yaml
├── values-dev.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── ingress.yaml
    └── frontdoor-pdb.yaml
```

### Chart Metadata

**File:** `helm/Chart.yaml`

```yaml
apiVersion: v2
name: frontdoor-fe-app
description: A Helm chart for deploying frontdoor-fe-app service
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### Values Configuration

**File:** `helm/values-dev.yaml`

Key configurations:

```yaml
name: flygtaxi-dev-frontdoor
namespace: flygtaxi-dev
replicaCount: 1
region: eu-north-1

image:
  repository: 165660931457.dkr.ecr.eu-north-1.amazonaws.com/eks-dev-eu-north-1:flygtaxi-dev-frontdoor-latest
  pullPolicy: Always

port: 8080

service:
  type: LoadBalancer
  port: 80

serviceAccount:
  enabled: true
  name: flygtaxi-dev-frontdoor
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::369171354678:role/cabonline-service-role/dev-cabonline-flygtaxi-frontdoor-dev-eu-north-1

resources:
  limits:
    cpu: 500m
    memory: 756Mi
  requests:
    cpu: 100m
    memory: 128Mi

availabilityZones:
  - eu-north-1a
  - eu-north-1b
  - eu-north-1c
```

### Deployment Template

**Key Features:**

**Rolling Update Strategy:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

**Health Probes:**
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
```

**Topology Spread:**
```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
```

Distributes pods across availability zones for high availability.

### ServiceAccount Template

**File:** `helm/templates/serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.serviceAccount.name }}
  namespace: {{ .Values.namespace }}
  annotations:
    eks.amazonaws.com/role-arn: {{ .Values.serviceAccount.annotations.eks.amazonaws.com/role-arn }}
```

Links Kubernetes ServiceAccount to AWS IAM Role via IRSA annotation.

### Service Template

**File:** `helm/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: {{ .Values.name }}
```

Exposes the application via AWS LoadBalancer on port 80, routing to container port 8080.

### Ingress Template

**File:** `helm/templates/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
    - host: api.dev.cabonline.io
      http:
        paths:
          - path: /flygtaxi-dev-frontdoor
            pathType: Prefix
            backend:
              service:
                name: flygtaxi-dev-frontdoor
                port:
                  number: 80
```

Configures AWS ALB for external access with automatic SSL redirect.

### Helm Commands

```bash
# Lint
helm lint ./helm

# Package
helm package ./helm --destination packaged

# Template (dry-run)
helm template test-release ./packaged/frontdoor-fe-app-0.1.0.tgz \
  --namespace flygtaxi-dev \
  --values ./helm/values-dev.yaml

# Install
helm upgrade --install flygtaxi-dev-frontdoor ./packaged/frontdoor-fe-app-0.1.0.tgz \
  --namespace flygtaxi-dev \
  --values ./helm/values-dev.yaml

# Verify
kubectl get all -n flygtaxi-dev
kubectl logs -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor -f
```

---

## GitHub Actions CI/CD

### Workflow Architecture

Three-stage pipeline with reusable workflows:

```
deploy-flygtaxi-dev-frontdoor.yaml (orchestrator)
  ├── build-push-ecr.yaml (stage 1: build & push)
  └── build-deploy-eks.yaml (stage 2: deploy)
```

### Main Workflow

**File:** `.github/workflows/deploy-flygtaxi-dev-frontdoor.yaml`

**Trigger:** Manual (`workflow_dispatch`)

**Inputs:**
- `environment`: Target environment (default: `dev`)
- `branch`: Git branch to deploy (default: `main`)
- `runner`: GitHub runner to use (default: `eks-dev`)

**Jobs:**

1. **build-and-push-frontdoor**
   - Builds Docker image
   - Tags with git commit SHA
   - Pushes to ECR
   - Outputs: `image_tag`, `ecr_registry`

2. **deploy-eks**
   - Waits for build completion
   - Authenticates to AWS
   - Connects to EKS cluster
   - Deploys via Helm with new image tag

### Build & Push Workflow

**File:** `.github/workflows/build-push-ecr.yaml`

**Key Steps:**

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::165660931457:role/GithubActions-GithubActionsManagementRole
    aws-region: eu-north-1

- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v2

- name: Build and push Docker image
  run: |
    IMAGE_TAG=${{ github.sha }}
    docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-${IMAGE_TAG} \
      -f ${DOCKERFILE_PATH} ${APP_DIR_PATH}
    docker push $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-${IMAGE_TAG}
    docker tag $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-${IMAGE_TAG} \
      $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-latest
    docker push $ECR_REGISTRY/$ECR_REPOSITORY:${IMAGE_NAME}-latest
```

**Image Tagging Strategy:**
- Primary tag: `flygtaxi-dev-frontdoor-<git-sha>`
- Latest tag: `flygtaxi-dev-frontdoor-latest`

### Deploy Workflow

**File:** `.github/workflows/build-deploy-eks.yaml`

**Key Steps:**

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::369171354678:role/GitHubActionsDeployRole
    aws-region: eu-north-1

- name: Connect to EKS cluster
  run: |
    aws eks update-kubeconfig \
      --region eu-north-1 \
      --name dev-col-eu-north-1-eks \
      --alias dev-cluster

- name: Deploy to EKS
  uses: ./.github/workflows/common-actions/deploy-to-eks-with-helm
  with:
    chart_dir: ./common-apps/frontdoor/helm
    release_name: flygtaxi-dev-frontdoor
    helm_namespace: flygtaxi-dev
    image_tag: ${{ inputs.image_tag }}
    ecr_registry: ${{ inputs.ecr_registry }}
    ecr_repository: eks-dev-eu-north-1
```

### Custom Action: Deploy to EKS with Helm

**Location:** `.github/workflows/common-actions/deploy-to-eks-with-helm/action.yaml`

**Steps:**

1. **Clone shared templates**
   - Pulls common templates from `cabonline-helm` repository
   - Only copies if files don't exist locally (preserves local customizations)

2. **Prepare values**
   - Copies `values-dev.yaml` to `values.yaml`

3. **Lint chart**
   - Runs `helm lint` to validate templates

4. **Package chart**
   - Creates `.tgz` artifact in `packaged/` directory

5. **Verify package contents**
   - Lists files in packaged chart
   - Checks for ServiceAccount and Service templates

6. **Dry-run template rendering**
   - Verifies ServiceAccount will be created
   - Validates template rendering

7. **Deploy**
   - Runs `helm upgrade --install` with packaged chart
   - Overrides image repository with new tag

**Critical Design Decision:**
Uses packaged `.tgz` file instead of chart directory to ensure all templates are included in the deployment.

### OIDC Authentication

GitHub Actions authenticates to AWS using OIDC (OpenID Connect), eliminating the need for long-lived credentials.

**Build Role:**
- ARN: `arn:aws:iam::165660931457:role/GithubActions-GithubActionsManagementRole`
- Permissions: ECR push, CloudWatch logs

**Deploy Role:**
- ARN: `arn:aws:iam::369171354678:role/GitHubActionsDeployRole`
- Permissions: EKS describe/update, assume IRSA roles

### Secrets Management

**Required Secrets:**

- `HELM_TOKEN`: GitHub personal access token for cloning `cabonline-helm` repository
  - Scope: `repo` (read access)
  - Used in: Template cloning step

### Running a Deployment

```bash
# Via GitHub UI
Actions → Build and deploy Flygtaxi Dev Frontdoor to EKS → Run workflow
  Environment: dev
  Branch: main
  Runner: eks-dev

# Via GitHub CLI
gh workflow run deploy-flygtaxi-dev-frontdoor.yaml \
  -f environment=dev \
  -f branch=main \
  -f runner=eks-dev
```

---

## Kubernetes Resources

### Namespace

```bash
kubectl create namespace flygtaxi-dev
```

### Deployed Resources

**Deployment:**
- Name: `flygtaxi-dev-frontdoor`
- Replicas: 1
- Image: `165660931457.dkr.ecr.eu-north-1.amazonaws.com/eks-dev-eu-north-1:flygtaxi-dev-frontdoor-<sha>`
- Port: 8080
- Health checks: Readiness and liveness probes on `/`

**Service:**
- Name: `flygtaxi-dev-frontdoor`
- Type: LoadBalancer
- Ports: 80 (external) → 8080 (container)
- Selector: `app: flygtaxi-dev-frontdoor`

**ServiceAccount:**
- Name: `flygtaxi-dev-frontdoor`
- Annotation: `eks.amazonaws.com/role-arn: arn:aws:iam::369171354678:role/cabonline-service-role/dev-cabonline-flygtaxi-frontdoor-dev-eu-north-1`
- Enables IRSA for AWS permissions

**Ingress:**
- ALB-backed ingress
- Host: `api.dev.cabonline.io`
- Path: `/flygtaxi-dev-frontdoor`
- SSL redirect enabled

**PodDisruptionBudget:**
- Name: `flygtaxi-dev-frontdoor-pdb`
- MinAvailable: 1
- Ensures availability during voluntary disruptions

### Verification Commands

```bash
# Check deployment status
kubectl get deployment -n flygtaxi-dev flygtaxi-dev-frontdoor

# Check pod health
kubectl get pods -n flygtaxi-dev -l app=flygtaxi-dev-frontdoor

# View logs
kubectl logs -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor -f

# Check service
kubectl get svc -n flygtaxi-dev flygtaxi-dev-frontdoor

# Check ingress
kubectl get ingress -n flygtaxi-dev

# Describe pod for troubleshooting
kubectl describe pod -n flygtaxi-dev -l app=flygtaxi-dev-frontdoor

# Check ServiceAccount
kubectl get sa -n flygtaxi-dev flygtaxi-dev-frontdoor -o yaml

# Verify IRSA annotation
kubectl get sa -n flygtaxi-dev flygtaxi-dev-frontdoor -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'
```

### Resource Limits

**Per Pod:**
- CPU: 100m (request), 500m (limit)
- Memory: 128Mi (request), 756Mi (limit)

**Scaling:**
- Manual: `kubectl scale deployment -n flygtaxi-dev flygtaxi-dev-frontdoor --replicas=3`
- Autoscaling: Disabled (can be enabled via `values.yaml`)

---

## Application Endpoints

### Health Check

**Path:** `/`, `/health`, `/healthz`

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-12-09T10:30:00.000000Z"
}
```

**Usage:**
- Kubernetes readiness probe
- Kubernetes liveness probe
- Load balancer health checks
- Minimal logging (DEBUG level)

### Status

**Path:** `/status`

**Response:**
```json
{
  "message": "Welcome to the Flygtaxi Dev Environment Frontdoor Service!",
  "timestamp": "2025-12-09T10:30:00.000000Z",
  "hostname": "flygtaxi-dev-frontdoor-c8c59c859-abc12",
  "environment": {
    "KUBERNETES_SERVICE_HOST": "10.100.0.1",
    "KUBERNETES_SERVICE_PORT": "443",
    "POD_NAME": "flygtaxi-dev-frontdoor-c8c59c859-abc12"
  },
  "status": "Integration test successful! ✓"
}
```

**Usage:**
- Manual testing
- Integration verification
- Monitoring dashboards

### Access URLs

**Internal (within cluster):**
```
http://flygtaxi-dev-frontdoor.flygtaxi-dev.svc.cluster.local:80
```

**External (via LoadBalancer):**
```
http://<loadbalancer-dns>:80
```

**External (via Ingress):**
```
https://api.dev.cabonline.io/flygtaxi-dev-frontdoor/status
```

---

## Monitoring and Logging

### Application Logs

**Format:**
```
2025-12-09 10:30:00,000 - __main__ - INFO - Flygtaxi Frontdoor Service - Starting Up
2025-12-09 10:30:00,001 - __main__ - INFO - Starting server on port 8080
2025-12-09 10:30:00,002 - __main__ - INFO - ✓ Running in Kubernetes cluster
2025-12-09 10:30:05,123 - __main__ - INFO - Status request - Status: 200, Path: /status
```

**Log Levels:**
- `DEBUG`: Health check requests, detailed response bodies
- `INFO`: Startup, status requests, shutdown
- `WARNING`: Non-Kubernetes environment, 404 errors
- `ERROR`: Application errors (if any)

**Configuration:**
Set `LOG_LEVEL` environment variable in `values-dev.yaml`:
```yaml
env:
  - name: LOG_LEVEL
    value: debug  # or info, warning, error
```

### CloudWatch Integration

Logs are automatically streamed to AWS CloudWatch Logs via Fluent Bit DaemonSet (cluster-level configuration).

**Log Group:** `/aws/eks/dev-col-eu-north-1-eks/application`

**Query Example:**
```
fields @timestamp, @message
| filter @logStream like /flygtaxi-dev-frontdoor/
| sort @timestamp desc
| limit 100
```

### Metrics

**Kubernetes Metrics:**
- CPU usage: `kubectl top pod -n flygtaxi-dev`
- Memory usage: `kubectl top pod -n flygtaxi-dev`

**AWS Metrics:**
- ALB target health
- EKS node metrics
- CloudWatch Container Insights

---

## Troubleshooting

### Common Issues

**1. ServiceAccount not found**

**Symptom:**
```
Error: pods "flygtaxi-dev-frontdoor-xxx" is forbidden: error looking up service account
```

**Solution:**
Verify ServiceAccount template is included in packaged chart:
```bash
tar -tzf packaged/frontdoor-fe-app-0.1.0.tgz | grep serviceaccount.yaml
helm template test-release packaged/frontdoor-fe-app-0.1.0.tgz | grep "kind: ServiceAccount"
```

**2. Image pull errors**

**Symptom:**
```
Failed to pull image: unauthorized or not found
```

**Solution:**
- Verify ECR permissions for EKS node role
- Check image tag exists: `aws ecr describe-images --repository-name eks-dev-eu-north-1`
- Verify image pull policy: Should be `Always` for latest tags

**3. Pod CrashLoopBackOff**

**Symptom:**
```
CrashLoopBackOff or Error state
```

**Solution:**
```bash
kubectl logs -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor --previous
kubectl describe pod -n flygtaxi-dev -l app=flygtaxi-dev-frontdoor
```

Check for:
- Port conflicts
- Missing environment variables
- Application startup errors

**4. Health check failures**

**Symptom:**
```
Readiness probe failed: Get "http://10.x.x.x:8080/": context deadline exceeded
```

**Solution:**
- Verify application is listening on correct port (8080)
- Check `CONTAINER_PORT` environment variable
- Increase `initialDelaySeconds` if app has slow startup
- Test endpoint manually: `kubectl exec -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor -- curl localhost:8080/health`

**5. Helm deployment fails**

**Symptom:**
```
Error: UPGRADE FAILED: invalid value for serviceAccountName
```

**Solution:**
- Verify template references: `{{ .Values.serviceAccount.name }}` (not `{{ .Values.serviceAccount }}`)
- Lint chart: `helm lint ./helm`
- Dry-run: `helm template test-release ./helm --values ./helm/values-dev.yaml`

### Debug Commands

```bash
# Get all resources
kubectl get all -n flygtaxi-dev

# Describe pod for events
kubectl describe pod -n flygtaxi-dev -l app=flygtaxi-dev-frontdoor

# Check resource usage
kubectl top pod -n flygtaxi-dev

# Port forward for local testing
kubectl port-forward -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor 8080:8080

# Execute command in pod
kubectl exec -it -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor -- /bin/bash

# Check service endpoints
kubectl get endpoints -n flygtaxi-dev flygtaxi-dev-frontdoor

# View Helm release
helm list -n flygtaxi-dev
helm get values -n flygtaxi-dev flygtaxi-dev-frontdoor

# Check IRSA token
kubectl exec -n flygtaxi-dev deployment/flygtaxi-dev-frontdoor -- env | grep AWS
```

---

## Security Considerations

### IAM Roles and Permissions

**IRSA (IAM Roles for Service Accounts):**
- Pods assume IAM roles via ServiceAccount annotations
- No AWS credentials stored in pods
- Role assumption is scoped to namespace and ServiceAccount name
- Trust policy validates OIDC provider and subject claim

**GitHub Actions OIDC:**
- Short-lived tokens (1 hour)
- Scoped to repository and workflow
- No long-lived credentials in GitHub secrets

### Network Security

**Ingress:**
- ALB enforces SSL/TLS
- Automatic redirect from HTTP to HTTPS
- WAF integration available (not configured in this setup)

**Service:**
- LoadBalancer type creates AWS NLB
- Security groups restrict access to specific ports
- Internal service communication uses ClusterIP

### Container Security

**Base Image:**
- Official Python slim image
- Regularly updated for security patches
- Minimal attack surface (no unnecessary packages)

**Runtime:**
- Non-root user execution
- Read-only filesystem where possible
- Resource limits prevent resource exhaustion

### Secrets Management

**Current State:**
- `HELM_TOKEN` stored in GitHub Secrets
- No application secrets in this deployment

**Recommendations:**
- Migrate to AWS Secrets Manager for application secrets
- Use External Secrets Operator for Kubernetes integration
- Rotate credentials regularly

---

## Performance Optimization

### Resource Allocation

**Current Settings:**
- CPU: 100m request, 500m limit
- Memory: 128Mi request, 756Mi limit

**Tuning Recommendations:**
- Monitor actual usage: `kubectl top pod -n flygtaxi-dev`
- Set requests to P95 usage
- Set limits to max observed + 20% headroom
- Adjust based on load testing results

### Scaling

**Horizontal Pod Autoscaling (disabled):**

To enable, update `values-dev.yaml`:
```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

**Manual Scaling:**
```bash
kubectl scale deployment -n flygtaxi-dev flygtaxi-dev-frontdoor --replicas=3
```

### Caching

**Docker Image Layers:**
- Base image cached across builds
- Multi-stage builds reduce final image size
- Use `.dockerignore` to exclude unnecessary files

**Helm Charts:**
- Packaged charts cached in `packaged/` directory
- Reusable across deployments

---

## Maintenance Procedures

### Updating the Application

**Code Changes:**
1. Commit changes to git
2. Push to target branch
3. Run GitHub Actions workflow
4. Verify deployment via logs and health checks

**Configuration Changes:**
1. Update `values-dev.yaml`
2. Commit and push
3. Run deployment workflow
4. Verify with: `helm get values -n flygtaxi-dev flygtaxi-dev-frontdoor`

### Terraform Updates

**ECR Changes:**
```bash
cd terraform/build/eu-north-1
terraform plan
terraform apply
```

**IRSA Changes:**
```bash
cd terraform/dev/eu-north-1
terraform plan
terraform apply

# Update ServiceAccount annotation in values-dev.yaml
# Redeploy via GitHub Actions or helm upgrade
```

### Rollback Procedures

**Helm Rollback:**
```bash
# List revisions
helm history -n flygtaxi-dev flygtaxi-dev-frontdoor

# Rollback to previous revision
helm rollback -n flygtaxi-dev flygtaxi-dev-frontdoor

# Rollback to specific revision
helm rollback -n flygtaxi-dev flygtaxi-dev-frontdoor 3
```

**Manual Rollback:**
```bash
# Deploy previous image tag
helm upgrade flygtaxi-dev-frontdoor ./packaged/frontdoor-fe-app-0.1.0.tgz \
  --namespace flygtaxi-dev \
  --values ./helm/values-dev.yaml \
  --set image.repository=165660931457.dkr.ecr.eu-north-1.amazonaws.com/eks-dev-eu-north-1:flygtaxi-dev-frontdoor-<previous-sha>
```

### Disaster Recovery

**Backup Resources:**
```bash
# Export all resources
kubectl get all -n flygtaxi-dev -o yaml > backup-$(date +%Y%m%d).yaml

# Export Helm values
helm get values -n flygtaxi-dev flygtaxi-dev-frontdoor > values-backup.yaml
```

**Restore:**
```bash
# Recreate namespace if needed
kubectl create namespace flygtaxi-dev

# Redeploy via Helm
helm upgrade --install flygtaxi-dev-frontdoor ./packaged/frontdoor-fe-app-0.1.0.tgz \
  --namespace flygtaxi-dev \
  --values values-backup.yaml
```

---

## Cost Optimization

### Current Infrastructure Costs

**Monthly Estimates (dev environment):**
- EKS cluster: ~$73 (control plane)
- EC2 nodes: Variable (shared across workloads)
- ALB: ~$22 + data transfer
- ECR storage: ~$0.10/GB/month
- CloudWatch logs: ~$0.50/GB ingested

**Optimization Opportunities:**
- Use Fargate for ephemeral workloads
- Enable ECR lifecycle policies (already configured via Terraform module)
- Reduce log verbosity in production
- Use reserved instances for predictable workloads

---

## Future Enhancements

### Planned Improvements

**GitOps Integration:**
- Migrate to ArgoCD or Flux for declarative deployments
- Automated sync from git repository
- Visual deployment status

**Observability:**
- Integrate with Datadog or Prometheus
- Custom metrics export from application
- Distributed tracing with OpenTelemetry

**Security Hardening:**
- Pod Security Standards enforcement
- Network policies for pod-to-pod communication
- Automated vulnerability scanning in CI pipeline

**CI/CD Enhancements:**
- Automated testing before deployment
- Blue-green deployment strategy
- Canary releases with gradual traffic shift

---

## References

### Internal Documentation

- Terraform modules: `git@github.com:CabonlineTeam/Infra-Terraform-Resources.git`
- Helm templates: `git@github.com:CabonlineTeam/cabonline-helm.git`
- GitHub Actions workflows: `.github/workflows/`

### External Resources

- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)

### Contact

For questions or issues, contact the platform team or file an issue in the repository.

---

**Document Version:** 1.0  
**Last Updated:** December 9, 2025  
**Author:** Mina Demian  
**Organization:** Ursine Enterprises

