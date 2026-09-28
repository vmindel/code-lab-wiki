# Local LLM on WEXAC with OpenCode

Run an open-weights LLM on a WEXAC GPU node as an LSF job, and use it as a
coding assistant through [OpenCode](https://opencode.ai). The model runs
inside our own cluster, so no code or data leaves WEXAC.

The setup has three pieces:

1. **`llm-start`** submits a GPU job to `short-gpu`.
2. **`llm-server-job.sh`** runs in that job. It starts llama.cpp's
   `llama-server` from a Singularity image and serves the model through an
   OpenAI-compatible API on port 8000.
3. **`llm-opencode`** launches OpenCode on the login node and points it at
   the running server.

## Current configuration

| Setting | Value |
| --- | --- |
| Model | `unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M` (4-bit GGUF, multimodal) |
| Runtime | llama.cpp `llama-server` in `llama-server-cuda.sif` |
| Context | 262,144 tokens (the model's native maximum), 1 parallel slot |
| LSF | `short-gpu`, 1 GPU with 48 GB, 4 cores, 6 h run limit |
| OpenCode model | `cluster-local/qwen3.8-27b` |

!!! warning "Login node"
    Model downloads and inference happen **only** inside the LSF job. On the
    login node you only submit (`bsub`), check (`bjobs`, `bpeek`) and run the
    OpenCode client.

## One-time setup

### 1. Install OpenCode

Install OpenCode following [its docs](https://opencode.ai/docs). The
installer puts the binary in `~/.opencode/bin/`. Make sure `opencode` is on
your `PATH`.

### 2. Get the llama.cpp container

WEXAC policy is that containers are pulled or built only on the `docker-gpu`
server, never on a login or compute node. We use the prebuilt image
`ghcr.io/ggml-org/llama.cpp:server-cuda`:

```bash
ssh docker-gpu.wexac.weizmann.ac.il
singularity pull llama-server-cuda.sif docker://ghcr.io/ggml-org/llama.cpp:server-cuda
scp llama-server-cuda.sif login1:~/.cache/cluster-local-llm/
```

The job expects the image at `~/.cache/cluster-local-llm/llama-server-cuda.sif`.
It only runs the image; it never pulls or builds one.

### 3. Install the scripts

Save the three scripts below in `~/.opencode/bin/` and make them executable
(`chmod 700 ~/.opencode/bin/llm-*`).

=== "llm-start"

    ```bash
    #!/usr/bin/env bash
    # Submit the local model server to LSF's short-gpu queue.
    set -euo pipefail
    ROOT="${HOME}/.cache/cluster-local-llm"
    mkdir -p "${ROOT}"
    exec bsub -q short-gpu -J local-llm -n 4 \
      -B -N \
      -R 'span[hosts=1] rusage[mem=4000]' \
      -gpu 'num=1:gmem=49152' \
      -W 06:00 \
      -oo "${ROOT}/server.%J.log" -eo "${ROOT}/server.%J.err" \
      "${HOME}/.opencode/bin/llm-server-job.sh"
    ```

=== "llm-server-job.sh"

    ```bash
    #!/usr/bin/env bash
    # Runs only as an LSF GPU job. Keep model downloads and inference off login nodes.
    set -euo pipefail

    ROOT="${HOME}/.cache/cluster-local-llm"
    IMAGE="${ROOT}/llama-server-cuda.sif"
    HF_CACHE="${ROOT}/huggingface"
    PORT="${LOCAL_LLM_PORT:-8000}"
    MODEL="unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M"

    mkdir -p "${ROOT}" "${HF_CACHE}"
    chmod 700 "${ROOT}"

    if [[ ! -s "${IMAGE}" ]]; then
      echo "Missing prebuilt SIF: ${IMAGE}" >&2
      echo "Pull/build it on docker-gpu and copy it here before submitting this job." >&2
      exit 2
    fi

    # Keep the per-job bearer token private; never put credentials in OpenCode config.
    umask 077
    od -An -N32 -tx1 /dev/urandom | tr -d ' \n' > "${ROOT}/api-key"
    KEY="$(cat "${ROOT}/api-key")"
    HOST="$(hostname -f)"
    printf '%s\n' "http://${HOST}:${PORT}" > "${ROOT}/endpoint"
    chmod 600 "${ROOT}/api-key" "${ROOT}/endpoint"

    echo "Local model endpoint: http://${HOST}:${PORT}/v1"
    echo "OpenCode launcher: ${HOME}/.opencode/bin/llm-opencode"

    export HF_HOME="${HF_CACHE}"
    exec singularity exec --nv \
      --env HF_HOME="${HF_CACHE}" \
      --env 'LD_LIBRARY_PATH=/app:/.singularity.d/libs:/usr/local/cuda/lib64:/usr/local/cuda/targets/x86_64-linux/lib:/usr/lib/x86_64-linux-gnu' \
      "${IMAGE}" /app/llama-server \
      --hf-repo "${MODEL}" \
      --host 0.0.0.0 \
      --port "${PORT}" \
      --ctx-size 262144 \
      --parallel 1 \
      --n-gpu-layers 99 \
      --api-key "${KEY}"
    ```

=== "llm-opencode"

    ```bash
    #!/usr/bin/env bash
    # Start a private OpenCode service with the currently submitted model endpoint.
    set -euo pipefail
    ROOT="${HOME}/.cache/cluster-local-llm"
    ENDPOINT="${ROOT}/endpoint"
    API_KEY="${ROOT}/api-key"
    if [[ ! -r "${ENDPOINT}" || ! -r "${API_KEY}" ]]; then
      echo "No local model endpoint/token found. Submit it with ~/.opencode/bin/llm-start first." >&2
      exit 1
    fi
    export LOCAL_LLM_BASE_URL="$(cat "${ENDPOINT}")/v1"
    export LOCAL_LLM_API_KEY="$(cat "${API_KEY}")"
    exec opencode --standalone "$@"
    ```

### 4. Register the provider in OpenCode

Add this to `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "providers": {
    "cluster-local": {
      "name": "Cluster Local LLM",
      "env": ["LOCAL_LLM_API_KEY"],
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": {
        "baseURL": "${LOCAL_LLM_BASE_URL}"
      },
      "models": {
        "qwen3.8-27b": {
          "name": "Qwen3.8 27B (GPU job)",
          "modelID": "unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M",
          "limit": {
            "context": 262144,
            "output": 2048
          }
        }
      }
    }
  }
}
```

The config holds no credentials. The URL and key come from environment
variables that `llm-opencode` sets.

## Daily use

### Start the server

```bash
~/.opencode/bin/llm-start
```

This only submits the job and prints its LSF job ID. Follow it with:

```bash
bjobs -l JOB_ID
bpeek JOB_ID
```

Full logs are at `~/.cache/cluster-local-llm/server.JOB_ID.{log,err}`. The
server is ready when the `.err` log shows `model loaded` and
`listening on http://0.0.0.0:8000` (or on your own port, see the port clash
note below). The first start downloads the weights
and takes a while. Later starts reuse the cache in
`~/.cache/cluster-local-llm/huggingface`.

`-B -N` makes LSF email you when the job starts and ends.

!!! warning "Port clash with other users"
    Everyone who uses this setup listens on port **8000** by default. If
    LSF puts two people's servers on the same GPU node, the second one
    can't open the port and exits right after starting. Its `.err` log
    shows an error that it couldn't bind the port.

    To avoid this, pick your own port and pass it when you submit:

    ```bash
    LOCAL_LLM_PORT=8123 ~/.opencode/bin/llm-start
    ```

    LSF copies your shell environment into the job, so
    `llm-server-job.sh` picks up `LOCAL_LLM_PORT`. It also writes the
    port into the `endpoint` file, so `llm-opencode` connects to the
    right place without any other changes. Pick a number between 1024 and
    65535 that's specific to you, and use the same one every time.

### Connect OpenCode

```bash
~/.opencode/bin/llm-opencode                    # works in the current directory
~/.opencode/bin/llm-opencode /path/to/project   # or pass a project directory
```

In OpenCode, pick `cluster-local/qwen3.8-27b` from `/models`.

Always launch OpenCode through `llm-opencode`, not plain `opencode`. Every
job gets a new host and a new API key, and the wrapper reads the current
ones. `--standalone` makes sure this OpenCode process uses the wrapper's
environment.

### Stop the server

```bash
bkill JOB_ID
```

Kill the job when you're done so you don't hold a 48 GB GPU for nothing.
`short-gpu` kills it after 6 hours anyway. Don't submit a second server
while one is still running: both would write the same endpoint and key
files.

!!! info "Security"
    The server listens on all interfaces of the compute node, so anyone on
    the cluster network can reach the port. The API key protects it. Each
    job writes a new random key to `~/.cache/cluster-local-llm/api-key`
    (mode 600). Never share it, commit it, or put it in `opencode.json`.

## Changing the model

Edit `MODEL` in `llm-server-job.sh`. The value is any llama.cpp-compatible
Hugging Face GGUF repo plus a quant tag:

```bash
MODEL="owner/model-GGUF:Q4_K_M"
```

`--hf-repo` downloads it when the job starts. Check that:

- the quant tag exists in the repo;
- the architecture is supported by the llama.cpp version in the SIF (if it
  isn't, pull a newer image, see step 2);
- for multimodal models, the repo includes the vision projector (`mmproj`)
  files.

Then update `opencode.json`:

1. Rename or add the entry under `providers.cluster-local.models` (this is
   the name you see in `/models`).
2. Set `modelID` to the exact name the server exposes, which is the `MODEL`
   string.
3. Adjust `limit.context` and `limit.output` if needed.
4. Restart the server job and relaunch `llm-opencode`.

!!! warning "GPU memory"
    Changing the model does not change the GPU request in `llm-start`.
    Weight memory depends on model size × quantization. KV-cache memory
    grows with the context size. On a GPU out-of-memory error, use a
    smaller quant, a shorter context, or raise `gmem`.

## Changing the context window

The context limit has to be set in **two places**, and they must match:

- `--ctx-size` in `llm-server-job.sh`
- `limit.context` for the model in `opencode.json`

Restart the job and OpenCode after changing them. The context covers the
system prompt, tool definitions, the whole conversation and the generated
output, not just your messages. The full 262k context uses a lot more GPU
memory than a short one.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Job stays `PEND` | `bjobs -l JOB_ID`. Usually it's waiting for a free GPU with 48 GB. |
| Job exits right away | `~/.cache/cluster-local-llm/server.JOB_ID.err` and `bpeek JOB_ID`. Exit code 2 means the SIF is missing. |
| Job exits and the log says the port is in use | Another user's server is on the same node. Resubmit with your own `LOCAL_LLM_PORT` (see [Start the server](#start-the-server)). |
| Context-length errors | `--ctx-size` and `limit.context` don't match. Fix them, then restart both. |
| OpenCode can't connect | Make sure you launched through `llm-opencode` and the job is still `RUN` (it may have hit the 6 h limit). |
| Model won't load / unknown architecture | The llama.cpp in the SIF is too old for the model. Pull a newer image on `docker-gpu`. |
