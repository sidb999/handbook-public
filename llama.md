# llama host (LLM inference)

Dedicated LLM inference box on the home LAN. Serves local models via
`llama-server` (OpenAI-compatible API) which the Hermes agent VM consumes.

## Facts

- IP: `10.0.0.2` (home LAN, same subnet as the Talos cluster)
- Hardware: X99 MR9A Machinist, Xeon E5-2630 v4 (10c/20t), 48GB RAM,
  RTX 3090 24GB, LITEON LCH-256V2S 240GB system disk
- OS: Arch Linux
- Inference: `llama-server` from llama.cpp, GPU offload (`--ngl all`),
  flash attention, MTP speculative decoding, q8_0 KV cache
- Default model: `unsloth/Qwen3.8-27B-GGUF:Q4_K_S` (fits in 24GB VRAM)
- API: `http://10.0.0.2:8080/v1` (host 0.0.0.0, port 8080)
- Extras enabled: `--webui-mcp-proxy --tools all` (MCP proxy + tool calling)

## Models

- Managed in `~/models/` (git repo, `*.gguf` gitignored — only the
  Makefile and notes are tracked).
- `make qwen3.8` is the current run target (sets GPU power limit 280W,
  starts llama-server with the tuned flags). Other targets: 9B and
  35B variants.
- Models are pulled from HuggingFace with `hfdownloader` /
  `huggingface-cli` (unsloth GGUF quants).

## Access

```bash
ssh <user>@10.0.0.2
curl http://10.0.0.2:8080/v1/models
```

## Hermes agent VM

The Hermes agent runs in a Lima VM on the user's machine (not on llama
itself), provisioned and driven by a Makefile:
a separate config repo (branch
`vm-only`).

- `vm-setup` — `limactl start` from the `archlinux` template
  (qemu on Linux, vz on macOS; 4c/8GB default, 12c/16GB with
  `VM_SIZE=big`)
- `vm-packages` — pacman/yay base: go, docker, chromium, neovim, etc.
- `vm-hermes` — installs `hermes-agent` (yay), seeds `~/.hermes`
  (config, env, skills), installs agent-browser
- `hermes-create-profiles` — profiles: `default`, `yolodev` (yolo
  mode), `work-secure` (smart approvals, no browser/delegation)
- `hermes-run-it-*` — interactive entry points per profile
- `hermes-backup-profiles` / `hermes-restore-profiles` — profile
  backup/restore across VM reinstalls (`vm-reinstall` chains them)
- Config lives in `hermes/config.lima.yaml` (uploaded to
  `~/.hermes/config.yaml`); model provider is `custom` pointing at
  llama's `/v1` endpoint.

The VM reaches llama directly on the home LAN
(`http://10.0.0.2:8080/v1` verified). Note: the committed
`config.lima.yaml` still has `base_url: http://<stale-ip>:8080/v1` —
stale IP, update it to `10.0.0.2`.

The repo also contains a self-hosted
[Firecrawl](https://github.com/firecrawl/firecrawl) docker-compose
(playwright + API + redis + postgres) used as the agent's web-scraping
backend.
