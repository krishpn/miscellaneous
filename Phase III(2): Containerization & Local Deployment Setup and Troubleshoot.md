NVIDIA Triton Inference Server Operations Runbook

A unified guide for operating Triton Inference Server locally on k3s / Docker and preparing deployments for AWS EKS & CI/CD environments.

Workspace & Environment Setup

Set up the project environment using dynamic, relative paths rather than static directory references:

```bash
  # 1. Define project paths relative to active workspace
  export PROJECT_ROOT="$(pwd)"
  export MODEL_REPO_DIR="${PROJECT_ROOT}/model_repository"
  export MANIFEST_PATH="${PROJECT_ROOT}/deploy/overlays/local-k3s/triton-deployment.yaml"
```

```bash
  # 2. Grant permissions across model repository files
  chmod -R 755 "${MODEL_REPO_DIR}"
```

**AWS EKS / CI/CD Note:**

In production, model files are stored in an Amazon `S3` bucket (e.g., `s3://my-company-triton-models/model_repository`) rather than local filesystems. *Triton streams model weights directly from S3 across multiple Availability Zones without mounting host storage.*

**Dynamic Kubernetes Manifest Template**

This template uses environment variables (`${MODEL_REPO_PATH}`) to dynamically inject model paths during deployment.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: triton
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triton-inference-server
  namespace: triton
  labels:
    app: triton
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: triton
  template:
    metadata:
      labels:
        app: triton
    spec:
      containers:
      - name: triton-server
        image: localhost:5000/triton-custom:24.08
        imagePullPolicy: IfNotPresent
        command: ["tritonserver"]
        args:
          - "--model-repository=/models"
          - "--model-control-mode=explicit"
          - "--load-model=resnet50_vision"
          - "--load-model=bge_small_embedding"
        ports:
          - containerPort: 8000
            name: http
          - containerPort: 8001
            name: grpc
          - containerPort: 8002
            name: metrics
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: 12Gi
          requests:
            cpu: "2"
            memory: 6Gi
            nvidia.com/gpu: 1
        volumeMounts:
          - name: model-repo
            mountPath: /models
            readOnly: true
          - name: dshm
            mountPath: /dev/shm
      volumes:
        - name: model-repo
          hostPath:
            path: ${MODEL_REPO_PATH}
            type: Directory
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: triton-service
  namespace: triton
spec:
  type: NodePort
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: 30810
      name: http
    - port: 8001
      targetPort: 8001
      nodePort: 30811
      name: grpc
    - port: 8002
      targetPort: 8002
      nodePort: 30812
      name: metrics
  selector:
    app: triton
```

**AWS EKS / CI/CD Notes:**

*Storage*: Remove `hostPath` volumes. Update container args to `--model-repository=s3://<your-bucket-name>/model_repository`.

*Permissions*: Attach an IAM Role for Service Accounts (`IRSA`) via `serviceAccountName`: `triton-sa` to securely authenticate with S3.

*Networking*: Replace type: NodePort with type: `ClusterIP` and attach an AWS Load Balancer Controller (`NLB` or `ALB Ingress`) for production traffic routing.

**Local Standalone Docker Workflows**

Run Triton locally to validate model configurations before pushing updates:

```bash
# 1. Launch container with local volume mount
docker run --gpus all -d \
  --name tritonserver \
  --restart unless-stopped \
  -p 8000:8000 -p 8001:8001 -p 8002:8002 \
  -v "${MODEL_REPO_DIR}":/opt/model_repository \
  nvcr.io/nvidia/tritonserver:24.08-py3 \
  tritonserver --model-repository=/opt/model_repository --model-control-mode=explicit

# 2. Verify container restart policy
docker inspect tritonserver --format '{{ .HostConfig.RestartPolicy.Name }}'

# 3. Explicitly load models via HTTP API
curl -X POST http://localhost:8000/v2/repository/models/bge_small_embedding/load
curl -X POST http://localhost:8000/v2/repository/models/resnet50_vision/load

# 4. Query repository index
curl -s -X POST http://localhost:8000/v2/repository/index | jq .
```

**AWS EKS / CI/CD Note:**

Running docker run `--gpus` all directly is omitted in pipeline execution unless using self-hosted GPU CI runners (e.g., `AWS EC2 g4dn runners`). Standard CI pipelines perform static configuration linting or CPU-based smoke tests instead.

**Local Kubernetes (k3s) Operations**

Step 1: Image Push or Import

```bash 
# Option A: Push to local registry
docker push localhost:5000/triton-custom:24.08

# Option B: Import directly into k3s containerd cache
docker save localhost:5000/triton-custom:24.08 | sudo k3s ctr images import -
```

**AWS EKS / CI/CD Note:**

CI/CD pipelines build images and push directly to Amazon ECR (Elastic Container Registry):

```bash 
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com
docker build -t <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/triton-custom:v1.0 .
docker push <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/triton-custom:v1.0
```

Step 2: Clear Conflicts & Apply Manifest

```bash
# 1. Clear stale services or deployments
kubectl delete service triton-triton -n triton --ignore-not-found
kubectl delete service triton-service -n triton --ignore-not-found
kubectl delete deployment triton-inference-server -n triton --ignore-not-found


# 2. Inject local path variable and apply manifest
export MODEL_REPO_PATH="${MODEL_REPO_DIR}"
envsubst < "${MANIFEST_PATH}" | kubectl apply -f -
```

**AWS EKS / CI/CD Note:**

Production pipelines replace envsubst scripts with Helm or Kustomize templates managed by GitOps controllers like ArgoCD or FluxCD.

**Pod & GPU Troubleshooting**

In single-GPU local setups (e.g., `RTX 3070 Ti`), new pods may stay Pending with 1 Insufficient [nvidia.com/gpu](https://nvidia.com/gpu) if a previous pod holds an active GPU resource lock.


```bash 
# 1. Target active pod dynamically
export POD_NAME=$(kubectl get pods -n triton -l app=triton -o jsonpath='{.items[0].metadata.name}')

# 2. Force-delete pods locking GPU resources
kubectl delete pod -n triton --force --grace-period=0 -l app=triton

# 3. Trigger deployment rollout restart
kubectl rollout restart deployment/triton-inference-server -n triton

# 4. Stream pod status transitions
kubectl get pods -n triton -w
```

**AWS EKS / CI/CD Note:**

Production EKS clusters utilize Karpenter or Cluster Autoscaler paired with EC2 GPU Node Groups (g4dn.xlarge, g5.xlarge). Karpenter provisions additional GPU instances dynamically as replica demand scales, avoiding resource deadlocks.

Dynamic In-Pod Model Copy (Debug Workflow)
Transfer model updates directly into a running pod during local development:

```bash
export POD_NAME=$(kubectl get pods -n triton -l app=triton -o jsonpath='{.items[0].metadata.name}')

# Loop through model directories and copy into pod
for model_dir in "${MODEL_REPO_DIR}"/*/; do
  model_name=$(basename "${model_dir}")
  echo "Copying ${model_name} to pod ${POD_NAME}..."
  kubectl cp "${model_dir}" "triton/${POD_NAME}:/models/${model_name}"
done
```

**AWS EKS / CI/CD Note:**

Manual kubectl cp commands are prohibited in production environments. Updates follow standard GitOps flows: upload model artifacts to S3, then trigger a model reload via Triton's Explicit Control API or execute a zero-downtime rolling deployment (kubectl rollout restart).

__Health Verification & Endpoint Auditing__

Audit model readiness and endpoint health using target variables:

```bash
export TRITON_HOST="localhost"
export TRITON_PORT="30810"

# 1. Check Server Readiness
curl -i "http://${TRITON_HOST}:${TRITON_PORT}/v2/health/ready"

# 2. Stream execution logs
kubectl logs -n triton -l app=triton --tail=100 -f

# 3. Query status for all active models in repository index
curl -s -X POST "http://${TRITON_HOST}:${TRITON_PORT}/v2/repository/index" | jq -r '.[].name' | while read model; do
  echo "Checking model readiness: ${model}"
  curl -i "http://${TRITON_HOST}:${TRITON_PORT}/v2/models/${model}/ready"
done

```



### Comparative Matrix: Local (nkepsX) vs. AWS EKS Cloud (cicdtest)

| Operational Layer | Local Environment (nkepsX / k3s) | Production Cloud (cicdtest / AWS EKS) | Impact & Migration Strategy |
| :--- | :--- | :--- | :--- |
| **Model Storage** | **Local File System**<br>`hostPath` mounted from `/models` | **Amazon S3**<br>`s3://<bucket>/model_repository` | Move models out of Git/containers into S3. Triton streams weights directly at startup. |
| **Authentication & IAM** | **Local Permissions**<br>POSIX `chmod 755` file mounts | **AWS IRSA**<br>IAM Roles for Service Accounts | Attach ServiceAccounts with IAM roles to pods so Triton authenticates to S3 without static keys. |
| **Container Registry** | **Local / Private**<br>`localhost:5000` or local `containerd` | **GitHub Container Registry / ECR**<br>`ghcr.io/krishpn/cicdtest` | Push images in CI/CD pipeline using GitHub Actions with `packages: write` permissions. |
| **Manifest Deployment** | **Manual Shell Scripts**<br>Raw `kubectl apply` + `envsubst` | **Automated GitOps / Helm**<br>ArgoCD / Flux syncing `helm/` | Replace raw `.yaml` templates with parameterized Helm charts managed via GitOps triggers. |
| **Networking & Ingress** | **NodePort**<br>`30810` / `30811` static host ports | **AWS Load Balancer Controller**<br>ClusterIP + AWS ALB/NLB | Use standard K8s Ingress resources with annotations targeting AWS Application/Network Load Balancers. |
| **Node Scheduling & GPUs** | **Fixed Single Host**<br>Manual pod deletion to clear GPU locks | **Karpenter / Autoscaler**<br>EC2 GPU Node Groups (`g4dn`, `g5`) | Karpenter automatically provisions/terminates GPU EC2 instances based on pending pod requests. |
| **Verification & Testing** | **Manual Terminal Commands**<br>`curl` endpoints directly | **Automated Integration Testing**<br>`test.py` triggered in CI/CD | Execute `test.py` post-deployment in GitHub Actions to validate model readiness automatically. |