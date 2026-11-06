# b70-vllm-recipe

使用 Intel Arc Pro B70 與 vLLM 建立本地推論環境。

> [English](README.md) | 正體中文

## 關於這個食譜

自從 vLLM 0.31.0 開始, 對 Intel Arc B 系列產品已經相當友善, 少量的設定就能啟動本地推理引擎

而且目前的開源模型也具備一定程度的工作能力, 也有不錯的性能,

B70 的主人只需要複製貼上就可以快速體驗, 節省折騰時間

## compose.yaml

```yaml
services:
  vllm-xpu:
    image: vllm/vllm-openai-xpu:v0.33.0
    container_name: vllm-xpu-qwen38-prod
    ports:
      - "127.0.0.1:55850:5850"
    ipc: host
    devices:
      # 分享 Intel Arc Pro B70 給容器端
      - /dev/dri:/dev/dri
    volumes:
      # 分享 Intel Arc Pro B70 給容器端
      - /dev/dri/by-path:/dev/dri/by-path:ro
      # 分享 Hugging Face 模型快取
      - ${HOME_DIR}/.cache/huggingface:/root/.cache/huggingface
    environment:
      # 避免 container 存取 Hugging Face
      - HF_HUB_OFFLINE=1
      # 停用記憶體延遲分配, 防止分配期間 OOM (!!非常重要, 沒設定會啟動失敗)
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
# Intel Arc Pro B70 適合模型
# - Launch80/Swift-1.5-Qwen3.8-27B-Uncensored-Int4-baked
# - davetha/Huihui-Qwen3.8-27B-abliterated-AutoRound-Int4-baked-embed-int8
# - Wrapzii/Swift-1.5-Qwen3.8-27b-GPTQ-Int4-baked-v1-embed-int8
MODEL=Wrapzii/Swift-1.5-Qwen3.8-27b-GPTQ-Int4-baked-v1-embed-int8

# 投機解碼的 token 長度
SPEC_TOKENS=3

# context 長度
# - 65536   64K   最低值
# - 131072  128K  適合日常開發
# - 163840  160K  可能是 Qwen 3.8 27B 極限
# - 262144  256K  容易出問題
CONTEXT_LENGTH=163840

# VRAM 預先分配比例
# 換模型時先調 0.8, 再遞減測試
VRAM_RESERVATION=0.55

# 用幾張顯卡去跑
# - 2 (用兩張顯卡, 日常用, 升級時可能會遇到 oneCCL 錯誤)
# - 1 (用一張顯卡, 除錯用)
TP=2

# 家目錄, 讓容器讀得到模型
HOME_DIR=/home/username
```

## 效能數據

| MTP | Intel PP512 | Intel TG128 | vLLM PP512 | vLLM TG128 |
|:---:|:---:|:---:|:---:|:---:|
| 6 | 3017 | 419 | 2994 | 392 |
| 5 | 3052 | 439 | 3048 | 449 |
| 4 | 3065 | 440 | 3027 | 435 |
| 3 | 3035 | 470 | 3009 | 457 |

以上數據是下列腳本執行第二次的結果, 第一次執行視為 warm-up

```bash
HF_HUB_OFFLINE=1 llama-benchy --base-url http://localhost:55850/v1 \
  --model develop \
  --pp 512 --tg 128 --runs 5 --concurrency 8 \
  2>/dev/null | grep "^|" -B1 -A3
```

## 實驗環境

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

## 如果看不懂

這食譜的背景條件, 是假設您已經完成這些事項,

如果看不懂, 表示您需要先完成這些軟體設定

- 安裝 Ubuntu 26.04
- 安裝 Intel Arc B 系列驅動程式
- 安裝 python 與 uv
- 安裝 hf 管理 Hugging Face 模型
- 安裝 docker 與操作 docker compose
