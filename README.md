# 🚀 GitOps Pipeline — VProfile Project

A production-grade **GitOps CI/CD pipeline** for the VProfile Java application, deploying on **AWS EKS** using **GitHub Actions**, **Helm**, and **ArgoCD**.

> Git is the single source of truth. Every change goes through a repository — infrastructure, application, and deployment config alike.

---

## 📐 Architecture Overview

![GitOps Architecture](GitOps_Architecture.pdf)

The pipeline spans **three GitHub repositories** and **two cloud zones**:

```
GitHub (Left Side)                         AWS (Right Side)
─────────────────────────────────────      ─────────────────────────
vprofile-infra  → Terraform Pipeline  ───► EKS Cluster
vprofile-helm   → Helm Charts + Argo  ───► ArgoCD Controller
vprofile-app    → App Source + CI/CD  ───► Amazon ECR + SonarQube EC2
```

---

## 🗂️ Repository Structure

| Repository | Purpose | Pipeline |
|---|---|---|
| [`vprofile-app`](https://github.com/eeshwardevops/vprofile-app) | Java Spring MVC source code | GitHub Actions CI/CD |
| [`vprofile-helm`](https://github.com/eeshwardevops/vprofile-helm) | Helm charts + ArgoCD manifests | ArgoCD watches & syncs |
| [`vprofile-infra`](https://github.com/eeshwardevops/vprofile-infra) | Terraform EKS infrastructure | GitHub Actions (manual apply) |

---

## 🔄 CI/CD Pipeline Flow

### App Pipeline (`vprofile-app`)

```
Developer commits to feature branch
        │
        ▼
[No pipeline runs on feature branch]
        │
        ▼  (Pull Request to main)
┌───────────────────────────────────┐
│  build-and-sonar job              │
│  ✔ Maven build + Unit Tests       │
│  ✔ Checkstyle code analysis       │
│  ✔ SonarQube scan (self-hosted)   │
│  ✔ Quality Gate check             │
│  ✘ Blocks merge if gate fails     │
└───────────────────────────────────┘
        │
        ▼  (Merge to main)
┌───────────────────────────────────┐
│  docker-build-push job            │
│  ✔ Build Docker image             │
│  ✔ Tag: commit SHA + latest       │
│  ✔ Push to Amazon ECR             │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│  update-helm job                  │
│  ✔ Clone vprofile-helm repo       │
│  ✔ Update app.image + app.tag     │
│    in helm/vprofile/values.yaml   │
│  ✔ Commit & push via PAT          │
└───────────────────────────────────┘
        │
        ▼  (ArgoCD detects change)
┌───────────────────────────────────┐
│  ArgoCD auto-sync                 │
│  ✔ Pulls latest Helm charts       │
│  ✔ Rolling update on EKS          │
│  ✔ Self-heals on drift            │
└───────────────────────────────────┘
```

### Infrastructure Pipeline (`vprofile-infra`)

```
Feature branch push  →  Validate only
Pull Request to main →  Validate + Plan
Merge to main        →  Manual APPLY button
Scheduled / drift    →  Detect drift + Notify (Slack)
```

---

## 🛠️ Prerequisites

### Tools (local machine)

**Windows (PowerShell as Admin):**
```powershell
choco install awscli terraform kubernetes-cli kubernetes-helm -y
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
irm get.scoop.sh | iex
scoop install eksctl
```

**macOS:**
```bash
brew install awscli terraform kubernetes-cli helm eksctl
```

### AWS Requirements
- Domain registered (e.g., GoDaddy) with a wildcard ACM certificate (`*.yourdomain.xyz`)
- IAM user with `AdministratorAccess` for Terraform runs
- IAM user with `AmazonEC2ContainerRegistryFullAccess` + `AmazonEKSClusterPolicy` for GitHub Actions

---

## 🏗️ Step-by-Step Setup

### Step 1 — Create GitHub Repositories

Create three **private** repositories on GitHub:
- `vprofile-app`
- `vprofile-helm`
- `vprofile-infra`

### Step 2 — Configure SSH Authentication

```bash
cd ~/.ssh
ssh-keygen   # name it after your GitHub account, e.g. devops4sure
```

Add to `~/.ssh/config`:
```
Host github.com-devops4sure
    HostName github.com
    User git
    IdentityFile ~/.ssh/devops4sure
    IdentitiesOnly yes
```

Copy the public key to **GitHub → Settings → SSH and GPG Keys**.

Clone all three repos:
```bash
mkdir ~/Desktop/gitops && cd ~/Desktop/gitops
git clone git@github.com-devops4sure:eeshwardevops/vprofile-infra.git
git clone git@github.com-devops4sure:eeshwardevops/vprofile-helm.git
git clone git@github.com-devops4sure:eeshwardevops/vprofile-app.git
```

---

### Step 3 — Set Up Helm Charts (`vprofile-helm`)

Copy Kubernetes manifests from the `kubedefs` folder into the helm repo, then use GitHub Copilot (or the provided prompt in `PromptHelmCharts.txt`) to generate Helm charts.

**Helm chart structure:**
```
helm/
└── vprofile/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── app-deployment.yaml
        ├── db-deployment.yaml
        ├── mc-deployment.yaml
        ├── rmq-deployment.yaml
        ├── services.yaml
        ├── ingress.yaml
        ├── secret.yaml
        ├── pvc.yaml
        └── dockerregistry-secret.yaml
```

Key `values.yaml` fields to verify:
```yaml
app:
  image: <your-ecr-uri>/vprofileappimg
  tag: latest
  replicas: 1
  containerPort: 8080
  servicePort: 8080

db:
  storageClass: gp2      # Required for AWS EKS EBS

ingress:
  enabled: true           # Must be true
  host: vprofile.yourdomain.xyz
```

Add ArgoCD ingress annotations to `ingress.yaml`:
```yaml
annotations:
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip
  alb.ingress.kubernetes.io/certificate-arn: <Your-ACM-ARN>
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
  alb.ingress.kubernetes.io/ssl-redirect: '443'
  alb.ingress.kubernetes.io/backend-protocol: HTTP
```

Commit and push to main:
```bash
git add . && git commit -m "generated helm charts" && git push origin main
```

---

### Step 4 — Initialize App Repo (`vprofile-app`)

Copy `src/`, `pom.xml`, and `Dockerfile` from the `kube-app` branch of the vprofile-project into `vprofile-app`.

Add `sonar-project.properties`:
```properties
sonar.projectKey=vprofile-app
sonar.projectName=vprofile-app
sonar.projectVersion=1.0
sonar.sources=src/
sonar.java.binaries=target/classes
sonar.junit.reportsPath=target/surefire-reports/
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
sonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
```

---

### Step 5 — Launch SonarQube EC2 Server

- **Instance type:** `t2.medium` (4 GB RAM minimum)
- **OS:** Ubuntu
- **Ports:** 22 (your IP), 80 (anywhere)
- Use the `sonar-setup.sh` userdata script
- After launch: log in at `http://<public-ip>`, reset admin password, and generate a **User Token**

---

### Step 6 — Create Amazon ECR Repository

```bash
# In AWS Console → ECR → Create Repository
# Name: vprofileappimg
```

Copy the URI (e.g., `123456789.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg`).

---

### Step 7 — Store GitHub Secrets & Variables

In **`vprofile-app` → Settings → Secrets and Variables → Actions**:

**Secrets:**
| Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `SONAR_TOKEN` | SonarQube user token |
| `HELM_REPO_USER` | Your GitHub username |
| `GITOPS_PAT` | GitHub Personal Access Token |
| `SLACK_WEBHOOK` | Slack incoming webhook URL |

**Variables:**
| Name | Value |
|---|---|
| `AWS_REGION` | `us-east-1` |
| `ECR_REPOSITORY` | `vprofileappimg` |
| `HELM_REPO_NAME` | `vprofile-helm` |
| `SONAR_HOST_URL` | `http://<sonar-ec2-public-ip>` |

---

### Step 8 — Create GitHub Actions Pipeline

Create `.github/workflows/ci.yml` in `vprofile-app` (use the provided `ci.yml`).

**Pipeline summary:**
- **On PR to main:** build + test + SonarQube quality gate
- **On merge to main:** build Docker image → push to ECR → update Helm values

Test the pipeline:
```bash
git checkout -b feature-x
# make a change
git add . && git commit -m "test pipeline" && git push origin feature-x
# Then raise a Pull Request on GitHub
```

---

### Step 9 — Provision EKS with Terraform (`vprofile-infra`)

Generate Terraform code using the prompt in `1_PromptTerraformEksSetup.txt`, then add EBS CSI driver support using `2_PromptEBSCSIDrivers.txt`.

```bash
aws configure   # Set admin IAM keys

terraform init
terraform plan
terraform apply
```

This creates:
- VPC `10.0.0.0/16` with 2 public subnets
- EKS cluster `vprofile-eks-cluster`
- Node group: 1× `t3.large` (scales to 2)
- EBS CSI driver addon (for PVC support)

---

### Step 10 — Install AWS Load Balancer Controller

```bash
# Download and create IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# Create IAM service account
eksctl create iamserviceaccount \
  --cluster vprofile-eks-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve --region us-east-1

# Fetch kubeconfig
aws eks update-kubeconfig --name vprofile-eks-cluster --region us-east-1

# Install cert-manager
kubectl apply --validate=false \
  -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.yaml

# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts && helm repo update

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=vprofile-eks-cluster \
  --set region=us-east-1 \
  --set vpcId=<YOUR_VPC_ID> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

### Step 11 — Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm upgrade argocd argo/argo-cd --version 9.5.2 --install --create-namespace -n argocd
```

Create `argocd-ingress.yaml` (see `ArgoCDSetup.txt`) and apply:
```bash
kubectl apply -f argocd-ingress.yaml
kubectl get ingress argocd-ingress -n argocd   # grab ALB DNS endpoint
```

Add a **CNAME record** in your domain registrar:
```
Type: CNAME
Name: argocd
Value: <ALB DNS endpoint>
```

Get admin password and login:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login via CLI
argocd login argocd.yourdomain.xyz --username admin
```

---

### Step 12 — Connect ArgoCD to Helm Repo

```bash
# Add the Helm repo to ArgoCD
argocd repo add git@github.com:eeshwardevops/vprofile-helm.git \
  --ssh-private-key-path ~/.ssh/devops4sure

# Grant EKS node group access to ECR
aws iam attach-role-policy \
  --role-name vprofile-eks-cluster-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess
```

---

### Step 13 — Deploy VProfile App via ArgoCD

Create ArgoCD project and application manifests in `vprofile-helm/argocd/`:

```bash
# In your vprofile-helm repo
mkdir -p argocd/projects argocd/apps
```

Apply the ArgoCD resources (see `ArgoVprofileAppSetup.txt` for full YAML):
```bash
kubectl apply -f argocd/projects/vprofile-project.yaml
kubectl apply -f argocd/apps/vprofile-app.yaml
```

Add CNAME for vprofile app in your domain registrar:
```
Type: CNAME
Name: vprofile
Value: <vprofile ALB DNS endpoint>
```

Access the app at `https://vprofile.yourdomain.xyz`
- Username: `admin_vp`
- Password: `admin_vp`

---

## 🔐 GitHub Secrets Reference

All secrets live in `vprofile-app` → Settings → Secrets and Variables → Actions.

```yaml
# Secrets (hidden)
SONAR_TOKEN          # SonarQube auth token
AWS_ACCESS_KEY_ID    # GitHub Actions IAM user
AWS_SECRET_ACCESS_KEY
HELM_REPO_USER       # GitHub username for helm repo commits
GITOPS_PAT           # GitHub Personal Access Token (repo scope)
SLACK_WEBHOOK        # Slack incoming webhook URL

# Variables (visible)
AWS_REGION           # us-east-1
ECR_REPOSITORY       # vprofileappimg
HELM_REPO_NAME       # vprofile-helm
SONAR_HOST_URL       # http://<sonar-ec2-ip>
```

---

## 🧹 Cleanup

Run in this exact order to avoid orphaned AWS resources:

```bash
# 1. Delete application ingress (removes load balancer)
kubectl delete ingress vproapp-ingress -n vprofile
kubectl delete ingress argocd-ingress -n argocd

# 2. Delete IAM service account
eksctl delete iamserviceaccount \
  --cluster vprofile-eks-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller

# 3. Destroy EKS cluster
cd ~/Desktop/gitops/vprofile-infra
terraform destroy

# 4. Stop (or terminate) SonarQube EC2 instance

# 5. Delete ECR images / IAM users created for the project
```

> ⚠️ Delete the ingress resources **before** running `terraform destroy`, otherwise the load balancers will remain and the destroy will fail.

---

## 📋 Assigned Tasks (Extensions)

Three improvements to implement on your own:

1. **Pull Request gate for Helm updates** — Instead of directly committing `values.yaml`, have the CI pipeline raise a PR on `vprofile-helm`. ArgoCD only syncs after the PR is approved.

2. **Terraform pipeline** (`vprofile-infra`) — Feature branch push → Validate; PR to main → Validate + Plan; Merge → manual Apply button; Scheduled → Drift detection.

3. **Slack notifications** — Add Slack notify steps (success/failure) to both the app and Terraform pipelines using the `SLACK_WEBHOOK` secret.

---

## 🧱 Tech Stack

| Layer | Technology |
|---|---|
| Cloud | AWS (EKS, ECR, ACM, IAM, VPC) |
| IaC | Terraform |
| CI/CD | GitHub Actions |
| GitOps Controller | ArgoCD |
| Packaging | Helm |
| Code Quality | SonarQube (self-hosted EC2) |
| Container Runtime | Docker |
| Application | Java Spring MVC (Maven, Tomcat WAR) |
| Notifications | Slack |

---

## 📎 Resources

- [vprofile-app](https://github.com/eeshwardevops/vprofile-app)
- [vprofile-helm](https://github.com/eeshwardevops/vprofile-helm)
- [vprofile-infra](https://github.com/eeshwardevops/vprofile-infra)
- [AWS Load Balancer Controller Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [ArgoCD Helm Chart](https://artifacthub.io/packages/helm/argo/argo-cd)
- [SonarQube Scan Action](https://github.com/SonarSource/sonarqube-scan-action)
