
Phase IV automates the infrastructure transition from the local single-node K3s deployment to an auto-scaling, production-grade AWS EKS cluster driven by Terraform, Ansible, Helm, and GitOps via ArgoCD.

**Host Bootstrapping & Control Plane Setup** (`ansible/setup_host.yml`)

Automate control-node provisioning, tooling installation (`kubectl, helm, terraform, aws-cli`), and AWS authentication setup before applying infrastructure.


Ansible Playbook (`ansible/setup_host.yml`)

```yaml
- name: Prepare Operator Control Plane & Host Tooling
  hosts: localhost
  connection: local
  gather_facts: true

  tasks:
    - name: Ensure dependencies are installed
      apt:
        name:
          - curl
          - unzip
          - jq
          - python3-pip
        state: present
      sudo: true

    - name: Install kubectl
      shell: |
        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
      args:
        creates: /usr/local/bin/kubectl

    - name: Install Helm CLI
      shell: |
        curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
      args:
        creates: /usr/local/bin/helm

    - name: Verify AWS CLI installation
      command: aws --version
      register: aws_cli_check
      ignore_errors: true

    - name: Display environment status
      debug:
        msg: "Control plane tools (kubectl, helm, aws-cli) operational."
```
    
**Playbook Execution**

```bash
ansible-playbook ansible/setup_host.yml
```

**Infrastructure as Code** (```terraform/main.tf```)

Provision an AWS EKS cluster equipped with dedicated GPU node groups (```g5.xlarge```), AWS S3 bucket storage for model registry weights, and the AWS EBS / S3 CSI drivers.

Terraform Manifest (```terraform/main.t```f)

```bash
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "cluster_name" {
  type    = string
  default = "triton-eks-cluster"
}

# 1. S3 Bucket for Centralized Model Registry
resource "aws_s3_bucket" "model_registry" {
  bucket        = "triton-model-registry-production-weights"
  force_destroy = true
}

# 2. VPC Module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name                 = "triton-vpc"
  cidr                 = "10.0.0.0/16"
  azs                  = ["${var.aws_region}a", "${var.aws_region}b"]
  public_subnets       = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets      = ["10.0.10.0/24", "10.0.11.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = true
}

# 3. EKS Cluster with GPU Node Group
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.30"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    gpu_nodes = {
      min_size       = 1
      max_size       = 3
      desired_size   = 1
      instance_types = ["g5.xlarge"] # NVIDIA A10G (24GB VRAM)

      ami_type = "AL2_x86_64_GPU"

      pre_bootstrap_user_data = <<-EOT
        #!/bin/bash
        /usr/bin/nvidia-smi
      EOT
    }
  }
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "s3_bucket_name" {
  value = aws_s3_bucket.model_registry.bucket
}
```

**Provisioning Commands**

```bash
cd terraform
terraform init
terraform apply -auto-approve

# Connect local kubectl to the target EKS cluster
aws eks update-kubeconfig --region us-east-1 --name triton-eks-cluster
3. GitOps Continuous Delivery (gitops/triton-app.yaml)
Declaratively manage the lifecycle of the Triton Inference Server Helm deployment using ArgoCD targeting your GitHub repository.

ArgoCD Application Manifest (gitops/triton-app.yaml)
YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: triton-inference-server
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: 'https://github.com/krishpn/cicdtest.git'
    targetRevision: HEAD
    path: helm/triton-inference-server
    helm:
      valueFiles:
        - values.yaml
      parameters:
        - name: service.type
          value: LoadBalancer
        - name: image.repository
          value: "<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/triton-custom"
        - name: image.tag
          value: "24.08"
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: triton
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**GitOps Sync Execution**

```bash
# Apply ArgoCD Application spec
kubectl apply -f gitops/triton-app.yaml

# Check ArgoCD sync status
argocd app get triton-inference-server
```


**Troubleshooting & Operational Fixes**

_Issue 1: ArgoCD Helm Sync Failure (RPC Failed: Out of Memory during Image Pull)_

Symptom: Nodes fail to pull `triton-custom:24.08` from AWS ECR during automated deployment, leading to ImagePullBackOff or disk exhaustion on EKS root volumes.

Cause: The default AWS EKS node root disk size (20GB) is insufficient for extracting heavy Triton/CUDA images (15GB+ uncompressed).

Fix: Update terraform/main.tf node group spec to explicitly allocate larger EBS volume sizes:

```bash
gpu_nodes = {
  block_device_mappings = {
    xvda = {
      device_name = "/dev/xvda"
      ebs = {
        volume_size           = 100
        volume_type           = "gp3"
        delete_on_termination = true
      }
    }
  }
}
```

_Issue 2: NVIDIA Device Plugin Not Allocating GPUs in EKS_

Symptom: Pods remain in Pending state with reason 0/1 nodes are available: 1 Insufficient [nvidia.com/gpu](https://nvidia.com/gpu).

Cause: The EKS cluster lacks the DaemonSet required to expose hardware devices to the Kubernetes scheduler.

Fix: Deploy the official NVIDIA K8s Device Plugin Helm chart to EKS via helm:

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm upgrade --install nvdp nvdp/nvidia-device-plugin \
  --namespace kube-system
```

_Issue 3: S3 Model Synchronization Timeout / IAM Permission Denied_

Symptom: Triton pods crash with AccessDenied when fetching model weights from S3 or mounting via S3 CSI.

Cause: The EKS service account lacks an AWS IAM OIDC Role binding permitted to perform s3:GetObject and s3:ListBucket.

Fix: Associate an IRSA (IAM Roles for Service Accounts) policy via Terraform/AWS CLI:

```bash
aws iam attach-role-policy \
  --role-name triton-eks-cluster-gpu-nodes \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```