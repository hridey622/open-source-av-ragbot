# A Low-latency Voice Bot built with Modal and Pipecat 

A real-time conversational AI bot powered by [Pipecat](https://github.com/pipecat-ai/pipecat) and deployed on [Modal](https://modal.com). This project features an interactive RAG (Retrieval-Augmented Generation) system with real-time speech-to-speech interaction.

[Blog post](https://modal.com/blog/low-latency-voice-bot)

## Installation

### 1. Clone the Repository

```bash
git clone git@github.com:modal-projects/open-source-av-ragbot.git
cd open-source-av-ragbot
```

### 2. Set Up Python Environment

This project uses [uv](https://github.com/astral-sh/uv) for Python package management:

```bash
# Install dependencies
uv sync

# Activate virtual environment
source .venv/bin/activate
```

### 3. Configure Modal

Go to [modal.com](modal.com) and make an account if you don't have one.

```bash
# Authenticate your Modal installation
modal setup
```

### 4. Set Up Client

## Install dependencies

```bash
cd client
npm i
```

## Build

```bash
npm run build

# return to root dir
cd ..
```

## Deployment

### Deploy All Services

The project consists of multiple Modal services that need to be deployed:

```bash
# From the root dir of the project

# Deploy an LLM Service

# etiher VLLM inference server for optimized TTFT
modal deploy -m server.llm.vllm_server

# or use SGLang server for 
# aster cold starts with GPU snapshots
modal deploy -m server.llm.sglang_server

# Deploy Parakeet STT service
modal deploy -m server.stt.parakeet_stt

# Deploy Kokoro TTS service
modal deploy -m server.tts.kokoro_tts

# Deploy main bot application with frontend
modal deploy -m app
```

### Use your own GPU services (instead of Modal-hosted inference)

If you want to run inference on your own GPU (for example an NVIDIA L40S), you can point the bot to self-hosted endpoints and skip deploying the Modal STT/TTS/LLM services.

Set these environment variables before deploying `app`:

```bash
# Required for self-hosted LLM (OpenAI-compatible API)
export RAGBOT_LLM_BASE_URL=http://<your-host>:8092/v1

# Required for self-hosted STT websocket service
export RAGBOT_STT_WS_URL=ws://<your-host>:<stt-port>/ws

# Required for self-hosted single-speaker TTS websocket service
export RAGBOT_TTS_WS_URL=ws://<your-host>:<tts-port>/ws

# Optional (only if Moe + Dal dual speaker mode is enabled)
export RAGBOT_TTS_MOE_WS_URL=ws://<your-host>:<tts-port>/ws
export RAGBOT_TTS_DAL_WS_URL=ws://<your-host>:<tts-port>/ws
```

Behavior:
- If a URL variable is set, the bot uses that self-hosted endpoint.
- If a URL variable is not set, it falls back to spawning the corresponding Modal service.

With all three primary variables set (`RAGBOT_LLM_BASE_URL`, `RAGBOT_STT_WS_URL`, `RAGBOT_TTS_WS_URL`), you only need to deploy:

```bash
modal deploy -m app
```

### Choose your Hugging Face / Pipecat models

You can run this project with different models (LLM/STT/TTS). The easiest way is to keep the pipeline logic and change model IDs in the service files.

#### 1) LLM (Hugging Face model)

This repo uses OpenAI-compatible serving (SGLang or vLLM) for the LLM.

- Update the model in **both** server configs:
  - `server/llm/sglang_server.py` → `MODEL_NAME`
  - `server/llm/vllm_server.py` → `MODEL_NAME`
- Keep the same model name in `server/bot/moe_and_dal_bot.py` where `ModalOpenAILLMService(model=...)` is created.

Example:

```python
MODEL_NAME = "meta-llama/Llama-3.1-8B-Instruct"
```

If the model is gated/private on Hugging Face, set a token before deploy:

```bash
export HF_TOKEN=<your_hf_token>
```

#### 2) STT (Hugging Face / NeMo model)

The STT server loads Parakeet in `server/stt/parakeet_stt.py`:

```python
self.model = nemo_asr.models.ASRModel.from_pretrained(
    model_name="nvidia/parakeet-tdt-0.6b-v3"
)
```

Replace `model_name` with another NeMo ASR model you want to use.

#### 3) TTS model/voice

TTS is implemented in `server/tts/kokoro_tts.py` using Kokoro. You can customize:
- voice in `server/bot/moe_and_dal_bot.py` (`voice="am_puck"`, `voice="am_fenrir"`)
- speed in the same place (`speed=...`)

#### 4) Pipecat service wiring

Pipeline orchestration is in `server/bot/moe_and_dal_bot.py` (STT → RAG → LLM → TTS). If you want a different Pipecat provider/service class, replace service construction there and keep compatible input/output frame behavior.

#### 5) Deploy after model changes

```bash
# If using Modal-hosted services
modal deploy -m server.llm.sglang_server
modal deploy -m server.stt.parakeet_stt
modal deploy -m server.tts.kokoro_tts
modal deploy -m app

# If using your own GPU endpoints
# (only app is needed once URLs are exported)
modal deploy -m app
```

### Warmup Snapshots
We can speed up the cold start time of our bot (this is more important) and our Parakeet and LLM service (if using SGLang) using snapshots. However this leads to extra start up time for the first few containers when the apps are (re-)deployed. To warmup snapshots, you can run these files as Python scripts.

```bash
python -m server.stt.parakeet_stt

python -m server.llm.sglang_server

python -m app
```

