# KUNAS Umbrel Store

Community Umbrel app store for KUNAS apps.

## Add this store to Umbrel

Use this URL in umbrelOS:

```text
https://github.com/9vibes/KNS-Umbrel
```

## Apps

- **Qobuz Sync** (`kunas-qobuz-sync`)  
  Sync purchased Qobuz music to `Downloads/QobuzSync`.  
  App source: [`9vibes/QobuzSync`](https://github.com/9vibes/QobuzSync)

- **Kokoro GPU** (`kunas-kokoro-fastapi`)  
  GPU-enabled Kokoro FastAPI text-to-speech server for Umbrel/Open WebUI.

- **ComfyUI GPU** (`kunas-comfyui`)
  Node-based AI image, video, and audio workflows with NVIDIA GPU acceleration.

- **InvokeAI GPU** (`kunas-invokeai`)
  Professional creative AI image generation with NVIDIA GPU acceleration and persistent models/outputs.

- **OpenCode Git** (`kunas-opencode-git`)
  OpenCode AI coding agent with Git and GitHub CLI installed inside the container, plus first-run local Ollama config for `qwen3.6:35b`.

- **KUNAS/Labs** (`kunas-steamlab`)
  OBS stream monitoring, manual recording, and opt-in face grouping for x86-64 NVIDIA hosts with CUDA 12.4 support. No CPU fallback.
  App source: [`9vibes/SteamLab`](https://github.com/9vibes/SteamLab). [Setup and safety notes](kunas-steamlab/README.md).

## Repository purpose

This repo contains only Umbrel app-store metadata and installable app packages.
Application source code lives in each app's own repo.
