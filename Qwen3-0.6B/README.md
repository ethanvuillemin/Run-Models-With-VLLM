# Run Qwen3 0.6B with vLLM and Docker

A containerized setup to run the Qwen3 0.6B language model using vLLM with OpenAI-compatible API.

## Prerequisites

- Docker and Docker Compose installed
- NVIDIA GPU with CUDA support
- NVIDIA Container Toolkit installed
- Git and Git LFS

## Quick Start

### 1. Download the Model

Clone the model from Hugging Face:
```bash
git clone https://huggingface.co/Qwen/Qwen3-0.6B
```

**Note:** Update the volume path in `docker-compose.yml` accordingly.

### 2. Start the Container
```bash
docker compose -f qwen3_0.6b.yml up -d
```

The API will be available at `http://localhost:8888`

### 3. Test the API
```bash
curl http://localhost:8888/v1/models
```

## Configuration

The setup includes the following optimizations:

- **Port:** 8888 (mapped from container's 8000)
- **GPU Memory:** 80% utilization
- **Max Context Length:** 5140 tokens
- **Tensor Parallel Size:** 1 (single GPU)
- **Tool Calling:** Enabled with Hermes parser
- **Offline Mode:** Enabled (no internet required after model download)

## Usage Examples

### Python
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8888/v1",
    api_key="dummy"  # vLLM doesn't require a real key
)

response = client.chat.completions.create(
    model="/qwen3_0.6b/",
    messages=[
        {"role": "user", "content": "Hello, how are you?"}
    ]
)

print(response.choices[0].message.content)
```

### cURL
```bash
curl http://localhost:8888/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/qwen3_0.6b/",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ]
  }'
```

## Managing the Container

Stop the container:
```bash
docker compose down
```

View logs:
```bash
docker compose logs -f
```

Restart the container:
```bash
docker compose restart
```

## Troubleshooting

### GPU not detected
Ensure NVIDIA Container Toolkit is installed:
```bash
docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
```

### Out of memory errors
Reduce `--gpu-memory-utilization` or `--max-model-len` in the docker-compose.yml

### Model not found
Verify the volume path matches your local model directory

## Resources

- [vLLM Documentation](https://docs.vllm.ai/)
- [Qwen3 Model Card](https://huggingface.co/Qwen/Qwen3-0.6B)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)