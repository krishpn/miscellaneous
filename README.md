
[**TECHNICAL STEPS: Setup & Deployment Guide**](https://krishpn.github.io/projects/20260816/)

## Phase I: Bare-Metal Host Prerequisites & K3s Infrastructure Setup

This phase bootstraps the local host environment, configures NVIDIA GPU driver interfaces, and patches the K3s container runtime and CNI networking paths.

* **Directory Structure & Git Workspace Initialization:**
  * Create standard workspace directory tree (`apps/`, `deploy/`, `model_repository/`, `templates/`, `scripts/`).
  * Initialize Git repository, configure `.gitignore` for heavy model binaries (`.onnx`, `.engine`, `.safetensors`), and track empty directories with `.gitkeep`.


* **NVIDIA Container Toolkit Integration:**
  * Install NVIDIA repository GPG keys and `nvidia-container-toolkit` packages on the host.
  * Verify driver integration via `nvidia-ctk` CLI to expose underlying hardware capabilities to containers.


* **K3s CNI & Containerd Runtime Patching:**
  * Symlink Flannel CNI configuration paths to `/etc/cni/net.d` and resolve binaries under `/opt/cni/bin` to prevent network initialization failures across node restarts.
  * Symlink Kubelet device-plugin sockets to `/var/lib/kubelet/device-plugins`.
  * Override `/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl` to enforce `/usr/bin/nvidia-container-runtime` as the default CRI runtime.


* **K8s GPU Orchestration Verification:**
  * Deploy the NVIDIA K8s Device Plugin DaemonSet.
  * Confirm allocatable node capacity for `[nvidia.com/gpu](https://nvidia.com/gpu)` using `kubectl get nodes`.



## Phase II: Declarative Model Configuration Compiler Engine

This phase abstracts raw Protobuf (`config.pbtxt`) creation into a Python- and Jinja2-driven compilation pipeline managed by a central declarative model catalog.

  * **Dynamic Jinja2 Templating (`templates/config.pbtxt.j2`):**
  * Define master template rules for decoupled LLM streaming, dynamic batching delay queues, multi-instance GPU replication, and variable input/output tensor shapes.


* **Multi-Modal Model Catalog (`model_catalog.yaml`):**
  * Establish a single declarative specification defining model schemas across vLLM (`qwen_1b`), ONNX text embeddings (`bge_small_embedding`), and vision backends (`resnet50_vision`).


* **Engine-Specific Runtime Overrides:**
  * Pass backend execution parameters directly to C++ runtimes via engine configs (e.g., `model.json` capping vLLM VRAM utilization at 75% and maximum token context limits).


* **Automated Build Execution (`scripts/compile_configs.py`):**
  * Execute Python compilation via `uv` to parse the catalog, automatically generate required model version directories (`model_repository/<model>/1/`), and write compiled `config.pbtxt` specs.


## Phase III: Containerization & Local Deployment

This phase packages compiled model configurations and custom runtime dependencies into an OCI container and deploys/validates the workload locally on K3s.

* **Custom Triton OCI Image:**
  * Build a custom Triton container (`triton-custom:24.08`) integrating vLLM dependencies, PyTorch, and ONNX Runtime backends.


* **Local K3s Manifests (`deploy/overlays/local-k3s/`):**
  * Configure Kubernetes `Deployment` using explicit model control (`--load-model`) and `hostPath` mounts pointing to absolute host paths.
  * Define NodePort `Service` exposing ports `30800` (HTTP), `30801` (gRPC), and `30802` (Metrics).


* **Operational Verification:**
  * Validate pod startup logs via `kubectl logs`.
  * Query server repository state using `POST /v2/repository/index`.
  * Test inference pipelines using `curl` payloads for LLM streaming, text embeddings, and vision classification.




## Phase IV: Production Cloud Migration & GitOps (AWS EKS)

This phase automates the infrastructure transition from a single-node local setup to a highly available, cloud-native architecture on AWS EKS.

* **Infrastructure as Code (Terraform/Ansible):**
  * Provision AWS EKS clusters with auto-scaling multi-GPU node groups (`g5`/`p4d` instances) and S3 bucket storage for model weights.


* **Kustomize & Helm Stratification:**
  * Separate base deployment manifests (`deploy/base`) from environment-specific overrides (`deploy/overlays/aws-eks`).
  * Replace local `hostPath` mounts with S3 CSI driver persistent volume claims.


* **GitOps Continuous Deployment (ArgoCD):**
  * Configure ArgoCD application specs targeting the Git repository to continuously synchronize configuration changes, schema updates, and model deployments to EKS.






