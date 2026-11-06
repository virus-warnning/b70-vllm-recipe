# b70-vllm-recipe

Build a local inference environment with Intel Arc Pro B70 and vLLM.

> English | [正體中文](README.zh-TW.md)

## About This Recipe

Since vLLM 0.31.0, it has been quite friendly to Intel Arc B series products. With minimal configuration, you can spin up a local inference engine.

Current open-source models also deliver a decent level of capability and performance.

B70 owners can simply copy and paste to quickly experience inference, saving you the hassle of tinkering.

## compose.yaml

```yaml
services:
  vllm-xpu:
    image: intel/llm-scaler-vllm:0.31.0-b1-experimental
    # image: vllm/vllm-openai-xpu:v0.33.0
    container_name: vllm-xpu-qwen38-prod
    ports:
      - "127.0.0.1:55850:5850"
    ipc: host
    devices:
      # Share Intel Arc Pro B70 with the container
      - /dev/dri:/dev/dri
    volumes:
      # Share Intel Arc Pro B70 with the container
      - /dev/dri/by-path:/dev/dri/by-path:ro
      # Share Hugging Face model cache
      - ${HOME_DIR}/.cache/huggingface:/root/.cache/huggingface
    environment:
      # Prevent container from accessing Hugging Face
      - HF_HUB_OFFLINE=1
      # Disable deferred memory allocation to prevent OOM during allocation (!!critical, will fail to start without this)
      - NEOReadDebugKeys=1
      - EnableDeferBacking=0
    command:
      - "${MODEL}"
      - "--port=5850"
      - "--served-model-name=develop"
      - "--gpu-memory-utilization=${VRAM_RESERVATION}"
      - "--dtype=bfloat16"
      - "--max-model-len=${CONTEXT_LENGTH}"
      - "--kv-cache-dtype=fp8"
      - "--tensor-parallel-size=${TP}"
      - "--max-num-seqs=8"
      - "--max-num-batched-tokens=4224"
      - "--enable-auto-tool-choice"
      - "--enable-prefix-caching"
      - "--reasoning-parser=qwen3"
      - "--tool-call-parser=qwen3_xml"
      - '--speculative-config={"method":"mtp","num_speculative_tokens":${SPEC_TOKENS}}'
      - '--compilation-config={"cudagraph_mode":"FULL_DECODE_ONLY"}'
    restart: "no"
```

## .env

```ini
# Models suitable for Intel Arc Pro B70
# - Launch80/Swift-1.5-Qwen3.8-27B-Uncensored-Int4-baked
# - davetha/Huihui-Qwen3.8-27B-abliterated-AutoRound-Int4-baked-embed-int8
# - Wrapzii/Swift-1.5-Qwen3.8-27b-GPTQ-Int4-baked-v1-embed-int8
MODEL=Wrapzii/Swift-1.5-Qwen3.8-27b-GPTQ-Int4-baked-v1-embed-int8

# Speculative decoding token length
SPEC_TOKENS=3

# Context length
# - 65536   64K   Minimum
# - 131072  128K  Suitable for daily development
# - 163840  160K  Likely the limit for Qwen 3.8 27B
# - 262144  256K  Prone to issues
CONTEXT_LENGTH=163840

# VRAM pre-allocation ratio
# When switching models, start at 0.8, then decrease incrementally
VRAM_RESERVATION=0.55

# Number of GPUs to use
# - 2 (two GPUs, daily use, may encounter oneCCL errors during upgrades)
# - 1 (one GPU, for debugging)
TP=2

# Home directory, so the container can read the models
HOME_DIR=/home/username
```

## Benchmark Data

| MTP | Intel PP512 | Intel TG128 | vLLM PP512 | vLLM TG128 |
|:---:|:---:|:---:|:---:|:---:|
| 6 | 3017 | 419 | 2994 | 392 |
| 5 | 3052 | 439 | 3048 | 449 |
| 4 | 3065 | 440 | 3027 | 435 |
| 3 | 3035 | 470 | 3009 | 457 |

The numbers above are from the second run of the script below; the first run is treated as warm-up.

```bash
HF_HUB_OFFLINE=1 llama-benchy --base-url http://localhost:55850/v1 \
  --model develop \
  --pp 512 --tg 128 --runs 5 --concurrency 8 \
  2>/dev/null | grep "^|" -B1 -A3
```

## Test Environment

* GPU: ASRock Intel Arc Pro B70
* Bus: PCIe 5.0 x8/x8
* Tensor Parallelism: 2
* Context Length: 163840 (160K)
* Concurrent: 8
* Engine Image:
  * Intel: intel/llm-scaler-vllm:0.31.0-b1-experimental
  * vLLM: vllm/vllm-openai-xpu:v0.33.0
* CPU: AMD Ryzen 5 9600X
* Mainboard: ASUS ProArt B850-CREATOR WIFI NEO
* RAM: TEAM T-CREATE EXPERT 16G x2

## If You're Lost

This recipe assumes you have already completed the following:

If something doesn't make sense, it means you need to set up these prerequisites first.

- Install Ubuntu 26.04
- Install Intel Arc B series drivers
- Install Python and uv
- Install hf for Hugging Face model management
- Install Docker and learn Docker Compose
