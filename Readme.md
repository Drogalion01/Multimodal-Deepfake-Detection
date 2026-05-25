# Total Kaggle Training Report

## Executive Summary

This repository contains two generations of the multimodal deepfake detection pipeline. The original Kaggle pilot used ConvNeXt + Wav2Vec2 with a two-head fusion classifier and produced strong results on a 970-clip balanced subset. The present, improved approach replaces the video and audio backbones and the fusion strategy with a speaker-aware, cross-attention fusion stack (Video Swin Tiny + WavLM-Base+ + Cross-Attention Fusion) and a staged training workflow. Both approaches and their artifacts are preserved below.

### Key Outcomes (original pilot)

| Item | Result |
| --- | ---: |
| Sampled clips | 970 |
| Unique folders | 240 |
| Train / Val / Test | 726 / 120 / 124 |
| Video best val accuracy | 1.0000 |
| Audio best val accuracy | 0.9333 |
| Fusion test accuracy | 0.9677 |
| Fusion macro F1 | 0.9677 |

---

## Section A — Original / Previous Models (ConvNeXt + Wav2Vec2)

Summary: the first pilot used ConvNeXt-Tiny for video, Wav2Vec2-Base for audio, and a two-head multimodal classifier. It served as a compact baseline and produced the metrics above. Key files and artifacts are stored under `Outputs/all_deepfake_outputs_new`.

- Video backbone: ConvNeXt-Tiny
- Audio backbone: Wav2Vec2-Base
- Fusion: two-head classifier (video head + audio head → four-class aggregation)

Artifacts (original pilot):

- `Outputs/all_deepfake_outputs_new/outputs/convnext_tiny_deepfake_best.pt`
- `Outputs/all_deepfake_outputs_new/outputs/metrics.pt`
- `Outputs/all_deepfake_outputs_new/outputs/train_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs/val_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs/test_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/wav2vec2_base_deepfake_best.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/metrics.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/train_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/val_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/test_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs_fusion_two_head/best_checkpoint.pt`
- `Outputs/all_deepfake_outputs_new/outputs_fusion_two_head/last_checkpoint.pt`
- `Outputs/all_deepfake_outputs_new/outputs_fusion_two_head/test_metrics.json`
- `Outputs/all_deepfake_outputs_new/outputs_fusion_two_head/test_metrics.pt`
- `Outputs/all_deepfake_outputs_new/metadata/sample_manifest.json`

If you have additional artifacts from the original run, place them under `Outputs/all_deepfake_outputs_new/` and I'll add them to this list.

Present model outputs (placeholders)

When the new model finishes training, upload its artifacts under `Outputs/multimodal_videoswin_wavlm/` using the folder structure below. Leave the files here and I'll auto-link and summarize them in this README.

- `Outputs/multimodal_videoswin_wavlm/outputs_video/<checkpoint_files...>`
- `Outputs/multimodal_videoswin_wavlm/outputs_audio/<checkpoint_files...>`
- `Outputs/multimodal_videoswin_wavlm/outputs_fusion/best_checkpoint.pt`
- `Outputs/multimodal_videoswin_wavlm/outputs_fusion/final_metrics.json`
- `Outputs/multimodal_videoswin_wavlm/logs/training_log.txt`

When you upload, tell me the path to the main metrics file (e.g., `final_metrics.json`) and I'll insert the numeric results and a comparison table into this README.

### Kaggle saved notebook with outputs

The saved Kaggle notebook that contains the training outputs is here:

- [Kaggle notebook: FakeBDTeenTesting](https://www.kaggle.com/code/mahirashab/fakebdteentesting/notebook?scriptVersionId=322055664)

Use that notebook page to open the saved outputs, then look for the files listed below. This README keeps the local file names so anyone can match the notebook output to the artifact on disk.

Output file index:

- `Outputs/all_deepfake_outputs_new/outputs/convnext_tiny_deepfake_best.pt`
- `Outputs/all_deepfake_outputs_new/outputs/metrics.pt`
- `Outputs/all_deepfake_outputs_new/outputs/train_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs/val_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs/test_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/wav2vec2_base_deepfake_best.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/metrics.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/train_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/val_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs_audio/test_embeddings.pt`
- `Outputs/all_deepfake_outputs_new/outputs_fusion_two_head/best_checkpoint.pt`
- `Outputs/all_deepfake_outputs_new/outputs_fusion_two_head/last_checkpoint.pt`
- `Outputs/all_deepfake_outputs_new/outputs_fusion_two_head/test_metrics.json`
- `Outputs/all_deepfake_outputs_new/outputs_fusion_two_head/test_metrics.pt`
- `Outputs/all_deepfake_outputs_new/metadata/sample_manifest.json`

Notes & known behaviours:

- Video-only branch achieved perfect validation accuracy on the held-out split (likely indicates overfitting on this pilot split; use larger validation or speaker-held splits for production).
- FF (Fake Video + Fake Audio) samples remain the most confused class in some folds; targeted augmentation can help.

---

## Section B — Present Model Approach (Video Swin Tiny + WavLM-Base+ + Cross-Attention Fusion)

Overview: the current implementation moves to stronger backbones and a fusion strategy designed to let audio and video interact via cross-attention. The training is staged to stabilize learning and reduce catastrophic forgetting.

Architecture summary:

- Video backbone: Video Swin Tiny (from `timm.create_model("swinv2_tiny_window...")` or equivalent) producing spatio-temporal video embeddings.
- Audio backbone: `microsoft/wavlm-base-plus` (WavLM-Base+) using the `transformers` WavLMModel to produce per-frame audio embeddings.
- Fusion: `CrossAttentionFusion` module — projected audio queries attend to video keys/values (or vice-versa) using `nn.MultiheadAttention` (batch_first=True), followed by layer norms and projection heads.
- Classification: two-head fusion with separate video/audio heads and a combined 4-class head (FF/FR/RF/RR) used during final evaluation.

Training strategy (staged):

1. Stage A — Train video backbone only (frozen audio + fusion). Stabilizes video features.
2. Stage B — Fine-tune audio backbone separately (frozen video). Ensures good audio features without cross-modal interference.
3. Stage C — Train fusion layers with both backbones frozen (learn cross-modal parameters).
4. Stage D — End-to-end fine-tune (optional, low LR) to squeeze final gains.

Dataset & sampling:

- Speaker-aware pilot construction: pilot manifests are built with speaker-stratified sampling to avoid speaker leakage between train/val/test where possible.
- Class-balanced sampling is used in DataLoader via `WeightedRandomSampler` to handle class imbalance.
- Robust I/O: ffmpeg-based audio extraction with `torchaudio` and `scipy` fallbacks; video frames read with `cv2`. Files with unusual names are normalized before class inference.

Training reliability measures:

- Mixed precision training using `torch.cuda.amp` (autocast + GradScaler).
- Gradient accumulation and adjustable batch sizes for GPU memory control.
- Safe split logic: falls back to deterministic shuffle splits if speaker-stratified splitting is impossible for a given subset.

Expected artifacts (present approach):

- `Outputs/multimodal_videoswin_wavlm/outputs_video/*` (video-only checkpoints)
- `Outputs/multimodal_videoswin_wavlm/outputs_audio/*` (audio-only checkpoints)
- `Outputs/multimodal_videoswin_wavlm/outputs_fusion/*` (fusion checkpoints and final metrics)
- pilot manifests: `Outputs/pilot_manifests/pilot_manifest_970.json`

Practical notes and reproducibility

- To reproduce Stage A quickly, set `RUN_TRAINING=False` except the Stage A cell(s) and run for 1–2 epochs to sanity-check shapes and numerical stability.
- If you see shape errors in the WavLM backbone, ensure audio tensors are `[batch, time]` (squeeze channel dims introduced by some load pipelines).
- For large-scale runs, keep a snapshot of the `pilot_manifest_970.json` and the exact `transformers` / `timm` versions used.

---

## Comparison: Previous vs Present

- Strengths (previous): simpler, faster to train for quick pilots; ConvNeXt + Wav2Vec2 is a solid baseline and useful for ablation.
- Strengths (present): better temporal modelling (Swin), stronger audio representations (WavLM-Base+), and explicit cross-modal interactions via attention — typically improves robustness to subtle multimodal manipulations.
- Tradeoffs: the present approach is heavier (more parameters), requires careful AMP/accumulation setup and longer tuning. Use smaller pilot runs to confirm setup before a full sweep.

---

## Where to look next

- See `kaggle_videoswin_wavlm_multimodal_training.ipynb` for the full present-model training notebook and staged-run cells.
- Sample manifest and pilot utilities are in `Notebooks` (search for `pilot_manifest_970.json` and `discover_clip_records`).
- If you want, I can: run a one-batch dry-run locally (forward pass) to validate dataloaders and model shapes; or create a small `README_quickstart.md` with the exact commands to run Stage A.

---

## Contact / Repro tips

- Recommended packages: `torch>=2.0`, `timm`, `transformers`, `torchaudio`, `opencv-python`, `scipy` (for wav fallback), and `ffmpeg` available on PATH.
- Keep a copy of the pilot manifest and the `requirements.txt` used for the run to reproduce exact results.

---

Credits: dataset assembly and sampling scripts by the project team. Updated README prepared to document the transition to the cross-attention fusion stack.

