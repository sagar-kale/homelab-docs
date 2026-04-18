# AI Services

All AI services run locally on **ocean-strix** (128GB unified memory). No cloud API calls — everything from speech recognition to LLM inference to TTS runs on-device.

## Stack Overview

```mermaid
graph TD
    USER["User"]
    MIC["Microphone\n(PCM audio)"]
    CAM["Camera / Image"]

    WHISPER["Faster-Whisper STT\nport 8090\nlarge-v3-turbo-ct2"]
    YOLO["YOLO11n Detection\n80 COCO classes"]
    VISIONAGENT["Vision Agent\nport 8881\nFastAPI + React"]

    LLMSWAP["llama-swap\nport 11434\nOpenAI-compatible"]
    GEMMA_E4B["gemma4:e4b\nfast VLM"]
    GEMMA_26B["gemma4:26b\nhigh-quality"]
    NOMIC["nomic-embed-text\n768-dim embeddings"]

    KOKORO["Kokoro TTS\nport 8880 · ~0.5s\naf_bella · ONNX CPU"]
    CHATTERBOX["Chatterbox Turbo\nport 8882 · ~2.2s\nVoice clone · ROCm GPU"]

    USER --> MIC --> WHISPER --> VISIONAGENT
    USER --> CAM --> YOLO --> VISIONAGENT
    CAM --> VISIONAGENT
    VISIONAGENT --> LLMSWAP
    LLMSWAP --- GEMMA_E4B
    LLMSWAP --- GEMMA_26B
    LLMSWAP --- NOMIC
    LLMSWAP --> VISIONAGENT
    VISIONAGENT --> KOKORO
    VISIONAGENT --> CHATTERBOX
```

## Services

| Service | Port | Description |
|---------|------|-------------|
| [Vision Agent](vision-agent.md) | 8881 | Full-stack voice + vision app |
| [LLM Inference](llm-inference.md) | 11434 | llama-swap multi-model proxy |
| [TTS](tts.md) | 8880 / 8882 | Kokoro (fast) + Chatterbox (expressive) |
| Whisper STT | 8090 | Speech-to-text, OpenAI-compatible |
| Open WebUI | 3000 | Chat UI connected to llama-swap |
