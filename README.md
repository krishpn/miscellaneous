
[**TECHNICAL STEPS: Setup & Deployment Guide**](https://krishpn.github.io/projects/20260816/)


```bash
  ## 1. Directory Structure Setup
  mkdir -p <DIRECTORY>/.github/workflows \
         <DIRECTORY>/apps/gateway \
         <DIRECTORY>/model_repository/qwen_1b/1 \
         <DIRECTORY>/model_repository/bge_small_embedding/1 \
         <DIRECTORY>/model_repository/resnet50_vision/1 \
         <DIRECTORY>/deploy/base \
         <DIRECTORY>/deploy/overlays/local-k3s \
         <DIRECTORY>/deploy/overlays/aws-eks \
         <DIRECTORY>/templates \
         <DIRECTORY>/scripts
```


```text
├── .github/
│   └── workflows/               # CI/CD pipelines
├── apps/
│   └── gateway/                # API Gateway / Router service
├── deploy/
│   ├── base/                    # Kustomize base deployment manifests
│   └── overlays/
│       ├── local-k3s/          # Local K3s overrides (hostPath, single GPU, NodePort)
│       └── aws-eks/            # AWS EKS overrides (S3 storage, autoscaling)
├── model_repository/            # Triton model repository
│   ├── bge_small_embedding/
│   │   └── 1/                  # ONNX model artifact
│   ├── qwen_1b/
│   │   └── 1/
│   │       └── model.json      # vLLM engine runtime parameters
│   └── resnet50_vision/
│       └── 1/                  # Vision ONNX/TensorRT artifact
├── scripts/
│   └── compile_configs.py      # Dynamic Jinja2 -> pbtxt compiler
├── templates/
│   └── config.pbtxt.j2          # Master Triton pbtxt Jinja2 template
└── model_catalog.yaml          # Declarative multi-modal model spec
```

**Host Prerequisites & GPU Setup**

```bash
  # 1. Add GPG Key
  curl -fsSL [https://nvidia.github.io/libnvidia-container/gpgkey](https://nvidia.github.io/libnvidia-container/gpgkey) | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

  # 2. Add repository to apt sources
  curl -s -L [https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list](https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list) | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

  # 3. Install toolkit package
  sudo apt-get update
  sudo apt-get install -y nvidia-container-toolkit

  # 4. Confirm installation
  nvidia-ctk --version
```

**K3s Infrastructure Provisioning & Fixes**

_Fix CNI Configuration Pathing:_

```bash
  sudo mkdir -p /etc/cni/net.d
  sudo cp /var/lib/rancher/k3s/agent/etc/cni/net.d/10-flannel.conflist /etc/cni/net.d/10-flannel.conflist
  sudo ln -sf /var/lib/rancher/k3s/agent/etc/cni/net.d/10-flannel.conflist /etc/cni/net.d/10-flannel.conflist
  sudo systemctl restart k3s
```

_Fix CNI Binary Resolution (/opt/cni/bin):_

```bash
  sudo mkdir -p /opt/cni/bin
  sudo cp -f /var/lib/rancher/k3s/data/current/bin/* /opt/cni/bin/
  kubectl delete pods --all -n kube-system
```

_Configure Persistent Containerd NVIDIA Runtime:_

```bash 
  sudo mkdir -p /var/lib/rancher/k3s/agent/kubelet/device-plugins
  sudo mkdir -p /var/lib/kubelet
  sudo rm -rf /var/lib/kubelet/device-plugins
  sudo ln -s /var/lib/rancher/k3s/agent/kubelet/device-plugins /var/lib/kubelet/device-plugins

  sudo bash -c 'cat << "EOF" > /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
  version = 2

  [plugins."io.containerd.grpc.v1.cri"]
    enable_selinux = false
    stream_server_address = "127.0.0.1"
    stream_server_port = "10010"

  [plugins."io.containerd.grpc.v1.cri".containerd]
    default_runtime_name = "nvidia"
    snapshotter = "overlayfs"

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
      runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
        BinaryName = "/usr/bin/nvidia-container-runtime"
  EOF'

  sudo nvidia-ctk runtime configure --runtime=containerd \
    --config=/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl \
    --set-as-default

  sudo systemctl restart k3s
```


_Deploy NVIDIA Device Plugin:_

```bash
  kubectl delete daemonset -n kube-system nvidia-device-plugin-daemonset --ignore-not-found
  kubectl apply -f [https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml](https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml)
```

_Verify GPU Allocatable Capacity:_

```bash
  kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUs:.status.allocatable."nvidia\.com/gpu"
```

**Git Initialization**

```bash

  git init
  git branch -M main

  cat << 'EOF' > .gitignore
  # Python
  __pycache__/
  *.pyc
  .venv/
  env/

  # OS / Editors
  .DS_Store
  .idea/
  .vscode/

  # Model Weights & Artifacts
  *.onnx
  *.engine
  *.bin
  *.pt
  *.safetensors
  EOF

  find . -type d -not -path '*/.*' -exec touch {}/.gitkeep \;
  git add .
  git commit -m "initial commit: project structure and gitignore"
```

**Config Compilation Engine**

_Jinja2 Template (`templates/config.pbtxt.j2`)_

```bash 
  {% if model_name %}
  name: "{{ model_name }}"
  {% endif %}
  backend: "{{ backend }}"
  max_batch_size: {{ max_batch_size | default(0) }}

  {% if model_transaction_policy is defined and model_transaction_policy.decoupled %}
  model_transaction_policy {
    decoupled: true
  }
  {% endif %}

  {% if dynamic_batching is defined %}
  dynamic_batching {
    max_queue_delay_microseconds: {{ dynamic_batching.max_queue_delay_microseconds | default(5000) }}
  }
  {% endif %}

  input [
  {% for inp in inputs %}
    {
      name: "{{ inp.name }}"
      data_type: {{ inp.data_type }}
      dims: [ {{ inp.dims | join(', ') }} ]
      {% if inp.optional %}optional: true{% endif %}
    }{% if not loop.last %},{% endif %}
  {% endfor %}
  ]

  output [
  {% for out in outputs %}
    {
      name: "{{ out.name }}"
      data_type: {{ out.data_type }}
      dims: [ {{ out.dims | join(', ') }} ]
    }{% if not loop.last %},{% endif %}
  {% endfor %}
  ]
```


**Model Catalog** (`model_catalog.yaml`)

```bash

  models:
  # 1. Large Language Model (vLLM Backend)
  - model_name: "qwen_1b"
    backend: "vllm"
    max_batch_size: 0
    model_transaction_policy:
      decoupled: true
    inputs:
      - name: "text_input"
        data_type: "TYPE_STRING"
        dims: [1]
      - name: "sampling_parameters"
        data_type: "TYPE_STRING"
        dims: [1]
        optional: true
    outputs:
      - name: "text_output"
        data_type: "TYPE_STRING"
        dims: [-1]

  # 2. Text Embedding Model (ONNX Runtime Backend)
  - model_name: "bge_small_embedding"
    backend: "onnxruntime"
    max_batch_size: 64
    dynamic_batching:
      max_queue_delay_microseconds: 5000
    inputs:
      - name: "input_ids"
        data_type: "TYPE_INT64"
        dims: [-1]
      - name: "attention_mask"
        data_type: "TYPE_INT64"
        dims: [-1]
    outputs:
      - name: "embedding"
        data_type: "TYPE_FP32"
        dims: [384]

  # 3. Vision Classifier (ONNX Runtime / TensorRT Backend)
  - model_name: "resnet50_vision"
    backend: "onnxruntime"
    max_batch_size: 32
    dynamic_batching:
      max_queue_delay_microseconds: 2000
    inputs:
      - name: "input"
        data_type: "TYPE_FP32"
        dims: [3, 224, 224]
    outputs:
      - name: "output"
        data_type: "TYPE_FP32"
        dims: [1000]
```


**vLLM Configuration** (`model_repository/qwen_1b/1/model.json`)

```bash
  {
  "model": "Qwen/Qwen2.5-1.5B-Instruct",
  "disable_log_requests": true,
  "gpu_memory_utilization": 0.75,
  "max_model_len": 1024
}
```


**Compilation Script** (`scripts/compile_configs.py`)

```bash
#!/usr/bin/env python3
import os
import yaml
from jinja2 import Environment, FileSystemLoader

def compile_triton_configs():
    catalog_path = "model_catalog.yaml"
    if not os.path.exists(catalog_path):
        raise FileNotFoundError(f"Missing catalog at {catalog_path}")

    with open(catalog_path, "r") as f:
        data = yaml.safe_load(f)

    env = Environment(loader=FileSystemLoader("templates"))
    template = env.get_template("config.pbtxt.j2")

    for model_cfg in data.get("models", []):
        model_name = model_cfg["model_name"]
        target_dir = os.path.join("model_repository", model_name)
        os.makedirs(os.path.join(target_dir, "1"), exist_ok=True)

        rendered_pbtxt = template.render(**model_cfg)
        output_file = os.path.join(target_dir, "config.pbtxt")

        with open(output_file, "w") as f:
            f.write(rendered_pbtxt)

        print(f"--> [Compiled] {output_file}")

if __name__ == "__main__":
    compile_triton_configs()
```

**Compile Configurations with** `uv`

```bash 
uv venv
uv pip install jinja2 pyyaml
uv run python scripts/compile_configs.py
```

**Local K3s Deployment Manifest**

Save to `deploy/overlays/local-k3s/triton-deployment.yaml`:

```bash
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
        imagePullPolicy: Never
        command: ["tritonserver"]
        args:
          - "--model-repository=/models"
          - "--model-control-mode=explicit"
          - "--load-model=resnet50_vision"
          - "--load-model=bge_small_embedding"
        ports:
          - containerPort: 8000
            hostPort: 8000
            name: http
          - containerPort: 8001
            hostPort: 8001
            name: grpc
          - containerPort: 8002
            hostPort: 8002
            name: metrics
        resources:
          limits:
            [nvidia.com/gpu](https://nvidia.com/gpu): 1
            memory: 12Gi
          requests:
            cpu: "2"
            memory: 6Gi
        volumeMounts:
          - name: model-repo
            mountPath: /models
      volumes:
        - name: model-repo
          hostPath:
            path: ./model_repository
            type: Directory
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
      nodePort: 30800
      name: http
    - port: 8001
      targetPort: 8001
      nodePort: 30801
      name: grpc
    - port: 8002
      targetPort: 8002
      nodePort: 30802
      name: metrics
  selector:
    app: triton
```

**Deployment, Validation & Operational Verification**

__Step 1: Deploy Manifest__

Apply the local K3s deployment configuration:

```bash
kubectl apply -f ./deploy/overlays/local-k3s/triton-deployment.yaml
```

__Stream Container Logs__

Verify pod startup and monitor loaded backends:

```bash
kubectl logs -n triton -l app=triton -c triton-server -f
```

__Query Repository Index__

Verify which models are dynamically ready:

```bash
curl -s -X POST http://localhost:8000/v2/repository/index | jq
```

**Test Dynamic LLM Load & Unload**

```bash
curl -X POST http://localhost:8000/v2/models/bge_small_embedding/infer \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": [
      { "name": "input_ids", "shape": [1, 4], "datatype": "INT64", "data": [101, 2054, 2003, 102] },
      { "name": "attention_mask", "shape": [1, 4], "datatype": "INT64", "data": [1, 1, 1, 1] }
    ]
  }'
```


