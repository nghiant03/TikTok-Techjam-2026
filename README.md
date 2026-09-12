# Autonomous Recommender Research System

Entry for **TikTok TechJam 2026**, Track #2: **Autonomous Machine Learning Research Agent for Recommender Systems**. 

## Setup 

Install the project from the repository root:

```bash
uv sync --dev

export REPO_ROOT="$PWD"
export RUNTIME_ROOT="/external/tiktok2026-runtime"
export TIKTOK2026_RUNTIME_ROOT="$RUNTIME_ROOT"
export TIKTOK2026_KUAIRAND_PURE_DATA="/external/read-only/KuaiRand-Pure"

uv run tiktok2026 runtime-init \
  --runtime-root "$RUNTIME_ROOT" --repository-root "$REPO_ROOT"
uv run tiktok2026 migrate \
  --runtime-root "$RUNTIME_ROOT" --repository-root "$REPO_ROOT"
uv run tiktok2026 verify-manifests --repository-root "$REPO_ROOT"
```

Production uses `config/budgets/judged.toml` by default. Use `--profile-path` for another profile and `--operator-config` for an external TOML file. The operator file may provide `dataset_root`, an immutable `docker_image` containing `@sha256:...`, budget values, and a `[models.<role>]` table for each of `orchestration`, `research`, `implementor`, and `validator`.

For example, `/external/tiktok2026-operator.toml` can contain the following settings:

```toml
dataset_root = "/external/read-only/KuaiRand-Pure"
docker_image = "registry.example/tiktok2026@sha256:REPLACE_WITH_IMAGE_DIGEST"

[execution]
timeout_seconds = 900
memory_bytes = 4294967296
cpus = 1.0
gpu_count = 0

[budget]
gpu_hours = 1.0
wall_clock_seconds = 7200
tokens = 200000
disk_bytes = 21474836480
reserved_final_gpu_hours = 0.25
frontier_capacity = 4
max_repairs = 3

[models.orchestration]
base_url = "https://provider.example/v1"
model = "operator-approved-model"
api_key_env = "TIKTOK2026_ORCHESTRATION_API_KEY"
temperature = 0.0
max_tokens = 4096
timeout_seconds = 120.0
```

`execution.gpu_count` defaults to `0`. Set it to a positive count only when the host Docker daemon has the required GPU runtime and the pinned image supports that accelerator.

### LiteLLM gateway 

The four runtime agents can share a local LiteLLM OpenAI-compatible gateway. The
checked-in configuration in `config/litellm/config.yaml` maps the
`tiktok2026-chatgpt` alias to LiteLLM's documented provider,
and `config/litellm/operator-models.toml` configures that gateway for
orchestration, research, implementor, and validator. 

Install and start the gateway in one terminal:

```bash
uv sync --dev --group gateway
export LITELLM_MASTER_KEY="$(openssl rand -hex 32)"
export LITELLM_API_KEY="$LITELLM_MASTER_KEY"
uv run --group gateway litellm \
  --config "$REPO_ROOT/config/litellm/config.yaml" \
  --host 127.0.0.1 --port 4000
```

With the gateway running, start the controller using an operator TOML containing
the four tables from `config/litellm/operator-models.toml`:

```bash
uv run tiktok2026 run \
  --runtime-root "$RUNTIME_ROOT" --repository-root "$REPO_ROOT" \
  --profile-path "$REPO_ROOT/config/budgets/judged.toml" \
  --operator-config /external/tiktok2026-operator.toml
```

## CLI

The executable is `uv run tiktok2026`. The commands and their actual options are:

```text
runtime-init   --runtime-root PATH [--repository-root PATH]
migrate        --runtime-root PATH [--repository-root PATH]
verify-manifests [--repository-root PATH]
synthetic-run  [--iterations INTEGER] [--runtime-root PATH]
calibrate-baseline --runtime-root PATH [--repository-root PATH]
                   [--profile-path PATH]
run            --runtime-root PATH [--repository-root PATH]
               [--profile-path PATH] [--operator-config PATH] [--synthetic]
resume         --runtime-root PATH --run-id TEXT [--repository-root PATH]
recover-source-registration --runtime-root PATH --run-id TEXT [--repository-root PATH]
               [--profile-path PATH] [--operator-config PATH] [--synthetic]
inspect        --runtime-root PATH --run-id TEXT
finalize       --runtime-root PATH --run-id TEXT [--repository-root PATH] [--synthetic]
export         --runtime-root PATH --run-id TEXT [--repository-root PATH] [--synthetic]
diagnostics    [--repository-root PATH]
```

For example, a configured production run and its later operations are:

```bash
uv run tiktok2026 calibrate-baseline \
  --runtime-root "$RUNTIME_ROOT" --repository-root "$REPO_ROOT" \
  --profile-path "$REPO_ROOT/config/budgets/judged.toml"

uv run tiktok2026 run \
  --runtime-root "$RUNTIME_ROOT" --repository-root "$REPO_ROOT" \
  --profile-path "$REPO_ROOT/config/budgets/judged.toml" \
  --operator-config /external/tiktok2026-operator.toml

uv run tiktok2026 resume --runtime-root "$RUNTIME_ROOT" --run-id RUN_ID \
  --repository-root "$REPO_ROOT" \
  --operator-config /external/tiktok2026-operator.toml
uv run tiktok2026 inspect --runtime-root "$RUNTIME_ROOT" --run-id RUN_ID
uv run tiktok2026 finalize --runtime-root "$RUNTIME_ROOT" --run-id RUN_ID
uv run tiktok2026 export --runtime-root "$RUNTIME_ROOT" --run-id RUN_ID
```
