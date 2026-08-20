
Phase III packages the compiled model_repository and custom Triton/vLLM dependencies into a `self-contained` OCI container image and `deploys/validates` the workload locally on K3s using Helm.

**Multi-Stage OCI Image Construction (`Dockerfile`)**

Build a multi-stage container image that bundles `vLLM` (`0.5.4`), ONNX Runtime GPU support, and bakes the local model repository (`model_repository/`) directly into the final image target (`/opt/model_repository`).


```bash
# Dockerfile
# Stage 1: Base image with Triton pre-installed
FROM nvcr.io/nvidia/tritonserver:24.08-py3 AS base

# Stage 2: Install Python dependencies (vLLM, ONNX Runtime)
# Note: Triton 24.08 ships with PyTorch and CUDA 12.x pre-configured.
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir vllm==0.5.4 onnxruntime-gpu

# Stage 3: Final Runtime Image
FROM base AS final

WORKDIR /opt/tritonserver

# Copy the local compiled model repository into the container image
COPY model_repository /opt/model_repository

EXPOSE 8000 8001 8002

ENTRYPOINT ["tritonserver"]

# Default model repository loading path
CMD ["--model-repository=/opt/model_repository"]
```


**Build & Import Commands**

```bash
# 1. Build the multi-stage OCI image from repository root
docker build -t triton-custom:24.08 -f Dockerfile .

# 2. Export and import directly into the K3s Containerd runtime engine
docker save triton-custom:24.08 | sudo k3s ctr images import -
```

**Helm Chart Configuration** (`helm/triton-inference-server/`)

Manage deployment parameterization, explicit model loading, shared memory IPC volume mounts, and NodePort routing via Helm values.

Values Configuration (`helm/triton-inference-server/values.yaml`)

```yaml
replicaCount: 1

image:
  repository: triton-custom
  tag: "24.08"
  pullPolicy: Never

service:
  type: NodePort
  ports:
    http: 30800
    grpc: 30801
    metrics: 30802

resources:
  limits:
    nvidia.com/gpu: "1"
    memory: 12Gi
  requests:
    cpu: "2"
    memory: 6Gi

triton:
  modelRepository: "/opt/model_repository"
  modelControlMode: "explicit"
  loadModels:
    - "simple"

shmVolume:
  enabled: true
  sizeLimit: "4Gi"
```


**Chart Templates**

_Deployment_ (`helm/triton-inference-server/templates/deployment.yaml`):

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: triton-server
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          args:
            - "--model-repository={{ .Values.triton.modelRepository }}"
            - "--model-control-mode={{ .Values.triton.modelControlMode }}"
            {{- range .Values.triton.loadModels }}
            - "--load-model={{ . }}"
            {{- end }}
          ports:
            - containerPort: 8000
              name: http
            - containerPort: 8001
              name: grpc
            - containerPort: 8002
              name: metrics
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            {{- if .Values.shmVolume.enabled }}
            - name: dshm
              mountPath: /dev/shm
            {{- end }}
      volumes:
        {{- if .Values.shmVolume.enabled }}
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: {{ .Values.shmVolume.sizeLimit }}
        {{- end }}
```

  
_Service_ (`helm/triton-inference-server/templates/service.yaml`):

```YAML
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: {{ .Values.service.ports.http }}
      name: http
    - port: 8001
      targetPort: 8001
      nodePort: {{ .Values.service.ports.grpc }}
      name: grpc
    - port: 8002
      targetPort: 8002
      nodePort: {{ .Values.service.ports.metrics }}
      name: metrics
  selector:
    app: {{ .Release.Name }}
```

**Deployment Execution**

```bash
# Install or upgrade chart in the triton namespace
helm upgrade --install triton-server ./helm/triton-inference-server \
  --namespace triton \
  --create-namespace

# Verify deployment rollout
kubectl rollout status deployment/triton-server -n triton
```

**Operational Verification & Integration Testing** (`test.py`)

Validate runtime readiness, explicit model loading, and inference via python execution.

_Functional Test Script_ (`test.py`)

```python
import json
import urllib.request

TRITON_HTTP_URL = "http://localhost:30800"

def check_health():
    url = f"{TRITON_HTTP_URL}/v2/health/ready"
    req = urllib.request.Request(url)
    try:
        with urllib.request.urlopen(req) as resp:
            print(f"[+] Triton Health Ready Check Status: {resp.status}")
            return resp.status == 200
    except Exception as e:
        print(f"[-] Health check failed: {e}")
        return False

def get_repository_index():
    url = f"{TRITON_HTTP_URL}/v2/repository/index"
    req = urllib.request.Request(url, method="POST")
    with urllib.request.urlopen(req) as resp:
        data = json.loads(resp.read().decode())
        print(f"[+] Loaded Model Index:\n{json.dumps(data, indent=2)}")

def infer_simple():
    url = f"{TRITON_HTTP_URL}/v2/models/simple/infer"
    payload = {
        "inputs": [
            {"name": "INPUT0", "shape": [1, 16], "datatype": "INT32", "data": [i for i in range(16)]},
            {"name": "INPUT1", "shape": [1, 16], "datatype": "INT32", "data": [i for i in range(16)]}
        ]
    }
    headers = {"Content-Type": "application/json"}
    req = urllib.request.Request(url, data=json.dumps(payload).encode(), headers=headers, method="POST")
    try:
        with urllib.request.urlopen(req) as resp:
            res = json.loads(resp.read().decode())
            print(f"[+] Infer Simple Output Status: {resp.status}")
            print(f"[+] Payload Response Keys: {list(res.keys())}")
    except Exception as e:
        print(f"[-] Inference failed: {e}")

if __name__ == "__main__":
    if check_health():
        get_repository_index()
        infer_simple()
```

**Running Tests**

```bash
python3 test.py
```

**Troubleshooting & Runtime Fixes**

_Issue 1: Shared Memory (`/dev/shm`) Bus Error_

Symptom: vLLM workers or ONNX Runtime backends fail during dynamic batching or tensor initialization with Bus error (core dumped).

Cause: Default Containerd shared memory allocation (`/dev/shm`) defaults to 64MB, restricting high-throughput IPC.

Fix: Enforce shmVolume.enabled: true in `helm/triton-inference-server/values.yaml` to back `/dev/shm` with a 4Gi emptyDir RAM volume.

_Issue 2: ErrImageNeverPull / Image Miss on K3s_

Symptom: Pod fails to initialize with `ErrImageNeverPull` or `ImagePullBackOff`.

Cause: Docker daemon images are not automatically accessible to the K3s Containerd CRI.

Fix: Export the Docker image into the K3s ctr engine store:

```bash
docker save triton-custom:24.08 | sudo k3s ctr images import -
```

_Issue 3: CUDA OOM Under Multi-Backend Load_

Symptom: Triton fails to load additional ONNX or vLLM backends dynamically.

Cause: Unconstrained GPU memory reservation by engine backends on a single GPU (e.g., `RTX 3070 Ti 8GB`).

Fix: Pass precise VRAM allocation settings in `model_repository/simple/config.pbtxt` or vLLM runtime configuration files:


```json
parameters: {
  key: "gpu_memory_utilization"
  value: { string_value: "0.50" }
}
```