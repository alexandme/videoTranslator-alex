# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Overview
- This is a Python 3.8+ CLI tool that translates videos end-to-end: transcribe (Whisper) → translate (Hugging Face M2M100/NLLB) → TTS (Edge TTS or gTTS) → optional RVC voice conversion → remux dubbed audio back into the original video via FFmpeg.
- Models and caches are stored locally under models/ and reused between runs. Outputs are written to output/<video_basename>/.
- No formal test suite or linter configuration is present in this repository.

Common commands
- macOS prerequisites (FFmpeg):
  - brew install ffmpeg
  - Verify: ffmpeg -version
- Create/activate venv and install dependencies:
  - python -m venv .venv
  - source .venv/bin/activate
  - pip install -r requirements.txt
  - If fairseq installation is problematic (RVC not needed), use: pip install -r requirements_no_fairseq.txt
- Basic run:
  - python main.py "path/to/video.mp4"
- Recommended examples from README:
  - Faster + good quality (English → Russian):
    - python main.py "path/to/video.webm" -s en -t ru -g male -w medium -tr m2m100_1.2B
  - Disable RVC:
    - python main.py "path/to/video.mp4" --no-rvc
  - Use a specific RVC model (ensure the model files exist; see Architecture):
    - python main.py "path/to/video.mp4" --rvc-model "models/rvc/male/ru/drevnyirus.pth" -s en -t ru -g male -w medium -tr m2m100_1.2B
  - Prefer GPU for Whisper (if CUDA available):
    - python main.py "path/to/video.webm" -s en -t ru -g male -w base --use-gpu
- CLI help:
  - python main.py --help
- Idempotent reruns: delete specific outputs to redo a step
  - Remove output/<video>/transcript.txt to re-transcribe
  - Remove output/<video>/translated.txt to re-translate
  - Remove output/<video>/tts-chunks/*.mp3 to regenerate TTS
  - Remove output/<video>/audio_dubbed.mp3 or <video>_dubbed.mp4 to rebuild

High-level architecture and flow
- Entry point: main.py (argparse CLI). Key stages and functions:
  1) Environment checks
     - check_ffmpeg(): ensures ffmpeg is available on PATH.
  2) Transcription (Whisper)
     - transcribe_video(video_path, transcript_path, source_lang, model_size, use_gpu)
     - Loads openai-whisper model (tiny/base/small/medium/large) with device selection (CPU/GPU). Downloads into models/.
     - Produces timestamped word-level segments; coalesces to ~≤5s chunks or sentence boundaries. Writes output/<video>/transcript.txt as [start - end] text.
  3) Translation (Hugging Face)
     - translate_text(texts, output_path, source_lang, target_lang, model_name)
     - Models configured via MODEL_CONFIG (facebook/m2m100_418M, facebook/m2m100_1.2B, facebook/nllb-200-distilled-600M, facebook/nllb-200-1.3B).
     - snapshot_download caches to models/; GPU used if available.
     - Writes output/<video>/translated.txt (line-per-chunk).
  4) TTS generation with timing
     - generate_tts_with_timing(texts, output_dir, base_name, segments, target_lang, use_rvc, voice_gender, rvc_model)
     - For each translated chunk: generate TTS then adjust duration to match original segment (speed up to 2x or pad silence) using ffmpeg.
     - TTS backend logic in generate_tts_audio():
       - Tries Edge TTS first (edge-tts). If not installed, the script installs it via pip and exits that run; rerun to use it. Falls back to gTTS otherwise.
       - If use_rvc and a model is provided/auto-selected, runs RVCConverter for voice conversion; else keeps TTS output.
     - Writes per-chunk mp3 files to output/<video>/tts-chunks/ as <base>_NNNN.mp3.
  5) Audio assembly
     - combine_audio_segments(audio_dir, final_audio_dir, output_filename)
     - Concatenates mp3 chunks using ffmpeg (re-encodes with consistent parameters) into output/<video>/audio_dubbed.mp3.
  6) Final muxing
     - replace_audio(video_path, audio_path, output_path)
     - Remuxes the dubbed audio track into the original video container via ffmpeg, producing output/<video>/<base>_dubbed.mp4.
- Language and voice configuration
  - SUPPORTED_LANGUAGES and LANGUAGE_CODES define recognized language codes for Whisper, M2M100, and gTTS.
  - Voice selection:
    - Edge TTS voice map is internal to generate_tts_audio (e.g., en-US-JennyNeural/GuyNeural, ru-RU-SvetlanaNeural/DmitryNeural, etc.).
    - gTTS uses VOICE_OPTIONS['gtts'][lang][gender] for language code; no real gender difference in gTTS voices.
  - RVC model paths (optional):
    - Auto-selection maps to models/rvc/<gender>/<lang>/
    - Expect a .pth and an index file in that folder (example names in README). If missing, script falls back to Edge TTS.
- Caching and idempotency
  - Hugging Face models and Whisper weights download into models/ under the repo to avoid re-downloading.
  - Each pipeline stage checks for its outputs first and skips if found.

Important repository notes
- Requirements
  - requirements.txt includes Whisper, Transformers, Torch, MoviePy, gTTS, Mutagen, SentencePiece, NumPy, tqdm, fairseq, torchaudio, soundfile, librosa, scipy.
  - requirements_no_fairseq.txt excludes fairseq (commented out). Use this when RVC is not needed or fairseq build is problematic.
- Supported languages (per README and code)
  - en, ru, es, fr, de, it, pt, ja, ko, zh, hi.
- Output structure (per README)
  - output/
    - <video_basename>/
      - tts-chunks/ (per-chunk mp3s)
      - transcript.txt
      - translated.txt
      - audio_dubbed.mp3
      - <video_basename>_dubbed.mp4
- Colab helpers present (optional)
  - COLAB_README.md and VideoTranslator_Hindi_Colab.ipynb provide a GPU-friendly flow; colab_setup.py can patch Hindi support (already present in code here) and prints CLI help; colab_audio_fix.py patches duration logic for older versions.
- Testing and linting
  - No tests/pytest or linter config (ruff/flake8/black) found in this repo.

Troubleshooting highlights
- "ffmpeg not found": install and add to PATH (brew install ffmpeg on macOS).
- Edge TTS not installed: first run attempts pip install edge-tts; rerun after installation completes.
- CUDA OOM: use smaller Whisper/translation models or omit --use-gpu; see README for guidance.

