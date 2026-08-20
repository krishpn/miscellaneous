

This template dynamically handles decoupled streaming (vLLM), dynamic batching queues, GPU instance duplication, and variable input/output tensor shapes.

**Master Jinja2 Config Template (`templates/config.pbtxt.j2`)**

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

{% if instance_count is defined %}
instance_group [
  {
    count: {{ instance_count }}
    kind: KIND_GPU
  }
]
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

**Declarative Model Catalog (`model_catalog.yaml`)**

Acts as the single source of truth for all serving workloads across LLM, embedding, and vision backends.

```yaml

models:
  # 1. Large Language Model (vLLM C++ Backend)
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
    instance_count: 2
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
    instance_count: 2
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


**Engine-Specific Runtime Overrides (`model_repository/qwen_1b/1/model.json`)**

Directly overrides internal vLLM execution limits and memory allocation without polluting global server parameters.

```json 
{
  "model": "Qwen/Qwen2.5-1.5B-Instruct",
  "disable_log_requests": true,
  "gpu_memory_utilization": 0.75,
  "max_model_len": 1024
}
```


**Compiler Script (`scripts/compile_configs.py`)**
   
Parses the declarative YAML specification and writes concrete config.pbtxt Protobuf files into each target model repository path.

```python
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


**Execution Pipeline**

```bash
# Setup virtual environment and dependencies using uv
uv venv
uv pip install jinja2 pyyaml
```

**Expected Output Artifact Tree**
   
Executing Phase II generates the following file layout, ready for downstream OCI container baking (Phase III) or Helm mounting (Phase IV):

```text
model_repository/
├── bge_small_embedding/
│   ├── 1/                        # Weights/Artifact directory
│   └── config.pbtxt              # Compiled Protobuf configuration
├── qwen_1b/
│   ├── 1/
│   │   └── model.json            # Engine runtime overrides
│   └── config.pbtxt              # Compiled Protobuf configuration
└── resnet50_vision/
    ├── 1/                        # Weights/Artifact directory
    └── config.pbtxt              # Compiled Protobuf configuration
```

**Compile model configurations**

uv run python scripts/compile_configs.py
