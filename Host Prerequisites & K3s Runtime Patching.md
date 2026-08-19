
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
