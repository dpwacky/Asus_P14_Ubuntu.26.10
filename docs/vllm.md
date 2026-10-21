# A local model server

One vLLM container serves `nvidia/Qwen3.6-35B-A3B-NVFP4` on the installed
laptop. It is a post-install step. It is not in the ISO, and it is not in
`overlay/`.

The checkpoint is Qwen3.6-35B-A3B in NVIDIA NVFP4: about 35 billion
parameters in total, about 3 billion active per token. Qwen3.8-Flash-Next
is the larger sibling (about 176 billion parameters, 6 billion active) and
fills almost all of a 128 GB GB10-class machine. This one leaves the
desktop room. Several agents share the one server. Each agent is a client.
They do not each load a model.

Measured on this laptop on 21 October 2026, kernel
`7.3.0-rc6-n1x-64m.2+`, NVIDIA userspace 615.78.08, container
`vllm/vllm-openai:v0.32.0-aarch64` (vLLM 0.32.0). One stream, after a
warmup. Speculative decoding was off.

| Run | Prompt tokens | Completion tokens | Time to first token | Elapsed | Decode |
| --- | --- | --- | --- | --- | --- |
| Warmup, cap 64 | 17 | 64 | 148 ms | 0.901 s | 84.9 tok/s |
| Measure, cap 512 | 35 | 512 | 167 ms | 6.213 s | 84.7 tok/s |

Decode is completion tokens divided by the time after the first token.
Both runs stopped because they hit the cap (`finish_reason=length`). All
512 measured tokens were reasoning tokens: thinking was left on, and the
cap arrived before the answer. The figure is the decode rate.

The scheduler admits at most 4 sequences. That cap was not load-tested
here. The engine reported a KV cache of 2,026,867 tokens, which it
described as 15.46 times a full 128K request. The running limit is still
4.

## What was installed

The laptop had no Docker. Ubuntu 26.10 (`stonking`) already had the
packages:

- `docker.io` 29.7.2
- `nvidia-container-toolkit` 1.19.1
- `pigz`, pulled in with them

`/etc/apt/preferences.d/n1x-kernel` pins every Ubuntu package named
`libnvidia-*` to priority -1, so the archive cannot replace the 615.78.08
driver. That glob also hides `libnvidia-container1` and
`libnvidia-container-tools`, which are the container runtime, not the GPU
driver. A file that sorts first adds an exception for those two names
only:

`/etc/apt/preferences.d/00-allow-nvidia-container`

```
# The image pin blocks every Ubuntu package named libnvidia-*, which also
# hides the container runtime libraries. These two are not the GPU driver.
Package: libnvidia-container1
Pin: release o=Ubuntu
Pin-Priority: 500

Package: libnvidia-container-tools
Pin: release o=Ubuntu
Pin-Priority: 500
```

Apt uses the first matching pin. `00-allow-nvidia-container` is read
before `n1x-kernel`. `nvidia-driver-*`, `nvidia-utils-*` and the other
`libnvidia-*` packages stay pinned. Do not delete `n1x-kernel`.

Then:

```sh
sudo apt-get install -y docker.io nvidia-container-toolkit pigz
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

A check container confirmed the GPU. `docker run --rm --gpus all -e NVIDIA_DISABLE_REQUIRE=1 ubuntu:24.04 nvidia-smi -L` printed `GPU 0: NVIDIA RTX Spark N1X`.

`NVIDIA_DISABLE_REQUIRE=1` is there because the host driver is 615.78.08.
The image's CUDA requirement list does not have to name that exact driver
for the devices to work. The same variable is passed to the vLLM
container.

## The image and the container

```sh
sudo docker pull vllm/vllm-openai:v0.32.0-aarch64
```

The official `v0.32.0` tag is amd64. This laptop is aarch64, so the tag
has to be `v0.32.0-aarch64`.

The Hugging Face cache is on the host, so a later start does not download
the weights again. The checkpoint is public (about 23 GB). No token was
set.

```sh
mkdir -p "$HOME/.cache/huggingface"
sudo docker run -d --name qwen36-nvfp4 \
  --gpus all --ipc=host \
  -p 127.0.0.1:8000:8000 \
  -e NVIDIA_DISABLE_REQUIRE=1 \
  -v "$HOME/.cache/huggingface":/root/.cache/huggingface \
  vllm/vllm-openai:v0.32.0-aarch64 \
  nvidia/Qwen3.6-35B-A3B-NVFP4 \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 1 \
  --trust-remote-code \
  --kv-cache-dtype fp8 \
  --attention-backend flashinfer \
  --moe-backend marlin \
  --gpu-memory-utilization 0.4 \
  --max-model-len 131072 \
  --max-num-seqs 4 \
  --max-num-batched-tokens 8192 \
  --enable-chunked-prefill \
  --async-scheduling \
  --enable-prefix-caching \
  --load-format fastsafetensors \
  --reasoning-parser qwen3 \
  --tool-call-parser qwen3_xml \
  --enable-auto-tool-choice
sudo docker update --restart unless-stopped qwen36-nvfp4
```

There is no `--speculative-config`. The engine log recorded
`speculative_config=None`. Context per request is 131072 tokens.
`max-num-seqs 4` is the concurrency cap. `--gpu-memory-utilization 0.4`
keeps the unified memory shared with the desktop. The port is bound to
`127.0.0.1` only.

Attention is FlashInfer. The MoE path is Marlin NVFP4, which vLLM selected
on this GPU (`Using 'MARLIN' NvFp4 MoE backend`). The reasoning parser and
the `qwen3_xml` tool parser are on so a client can separate the thinking
channel and tool calls. Prefix caching stays on. Speculative decoding
stays off: on this hybrid model, MTP together with the prefix cache has
produced malformed tool calls on GB10-class machines.

## What the server reported

The first start downloaded the weights (the load line includes that wait:
20.28 GiB, 225 s) and then finished engine init in 71.6 s, of which 19.0 s
was compilation. CUDA graphs took a few seconds and 0.04 GiB.

| | |
| --- | --- |
| Weights and other consumed memory | 24.05 GiB |
| Peak activation | 1.87 GiB |
| KV cache | 21.35 GiB, 2,026,867 tokens |
| Requested cap, 0.4 of the device | 47.26 GiB of 118.14 GiB |
| Free on device at startup, as vLLM printed it | 109.47 GiB |
| Host `MemAvailable` after the server was up | 62.7 GiB |

`/v1/models` lists `nvidia/Qwen3.6-35B-A3B-NVFP4` with `max_model_len`
131072. A short completion before the timed run answered `12 * 17` with
204.

## The token test

The laptop has no `curl`. The run was a streaming
`/v1/chat/completions` request from Python, `temperature` 0. The warmup
asked for the single word `ready`, capped at 64 tokens. The measured
request asked for a numbered list of 80 short facts about trains, capped
at 512 tokens. Stream deltas for this model put thinking text in
`delta.reasoning`. Time to first token is the first such chunk. The table
at the top is that run.

Stop the server with `sudo docker stop qwen36-nvfp4`. Follow it with
`sudo docker logs -f qwen36-nvfp4`. Docker starts it again on boot
(`unless-stopped`). The first start after a reboot loads the weights from
the cache. It does not download them again.
