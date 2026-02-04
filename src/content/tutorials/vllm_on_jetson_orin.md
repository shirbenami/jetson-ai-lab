# Serving an LLM on Jetson with vLLM (Docker)

This README explains how to follow the **Jetson AI Lab – GenAI on Jetson (LLMs/VLMs)** tutorial and specifically how to:

1. Serve an LLM using **vLLM** inside a Docker container
2. Commit the container into an image
3. Re-run the container later
4. Verify the server works using a `curl` request

Reference tutorial:
https://www.jetson-ai-lab.com/tutorials/genai-on-jetson-llms-vlms/

---

## Prerequisites

- NVIDIA Jetson device (Orin / Xavier, etc.)
- JetPack installed (with CUDA working)
- Docker installed and NVIDIA runtime enabled
- Internet access (for Hugging Face model download)

Verify Docker GPU access:

```bash
sudo docker run --rm --runtime nvidia nvcr.io/nvidia/cuda:12.2.0-base nvidia-smi
```

---

## Step 1: Start the vLLM Container (Interactive)

Launch the official vLLM container as described in the Jetson AI Lab tutorial:

```bash
docker run --rm -it \
  --runtime nvidia \
  --network host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  nvcr.io/nvidia/vllm:latest-jetson-orin \
  /bin/bash
```

This gives you an interactive shell **inside the container**.

---

## Step 2: Serve the Model with vLLM

Inside the container, start the vLLM API server:

```bash
vllm serve espressor/meta-llama.Llama-3.2-3B-Instruct_W4A16 \
  --dtype float16 \
  --gpu-memory-utilization 0.5 \
  --max-model-len 512 \
  --max-num-batched-tokens 128 \
  --max-num-seqs 1 \
  --swap-space 0 \
  --enforce-eager
```

Notes:
- This configuration is tuned for **low-memory Jetson devices**
- `--enforce-eager` disables CUDA graphs for better stability
- Model weights will be cached under `~/.cache/huggingface`

Once loaded, the server will listen on:

```
http://0.0.0.0:8000
```

---

## Step 3: Commit the Container

In a **new terminal on the host**, find the running container:

```bash
docker ps
```

Commit it into a reusable image:

```bash
docker commit <CONTAINER_ID> vllm_llama_3b:latest
```

You can now stop the original container.

---

## Step 4: Run the Committed Image

Later, you can start the server again using:

```bash
docker run --rm -it \
  --runtime nvidia \
  --network host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm_llama_3b:latest
```

The vLLM server will start automatically if it was running at commit time.

---

## Step 5: Verify the Server Works

Run the following test request from the host:

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "espressor/meta-llama.Llama-3.2-3B-Instruct_W4A16",
    "messages": [{"role":"user","content":"Extract unique object names from the text. Return only a lowercase JSON array. No extra text. Remove colors, sizes, materials, and adjectives: A yellow box of cereal, a black metal fence, a white wall."}],
    "max_tokens": 32,
    "temperature": 0.2
  }' \
| python3 -c "import sys, json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
```

Expected output example:

```json
["box","fence","wall"]
```

---

## Useful Endpoints

- Health check:
  ```bash
  curl http://localhost:8000/health
  ```

- Metrics:
  ```bash
  curl http://localhost:8000/metrics
  ```

- OpenAPI docs:
  ```
  http://localhost:8000/docs
  ```

---

## Notes

- If you encounter GPU OOM errors, reduce:
  - `--gpu-memory-utilization`
  - `--max-model-len`
  - `--max-num-batched-tokens`
- For production, consider pinning model versions and Docker image tags

---

✅ You now have a reproducible workflow for serving LLMs on Jetson using vLLM.
