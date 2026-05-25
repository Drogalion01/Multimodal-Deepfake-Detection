# Dataset and Batch Model Notes

## Dataset Summary
- Project: FakeBDTeen deepfake dataset
- Root dataset path: `/kaggle/input/fakebdteen`
- Working Kaggle output path: `/kaggle/working`
- Total raw videos discovered in the local dataset tree: 7,628
- Sampled batch size for the Kaggle workflow: 970 videos
- Sampling strategy: balanced, distributed across folders instead of taking the same fixed count from each folder
- Sample manifest: saved as `sample_manifest.json` inside the sampled dataset folder

## Folder Structure
- The dataset is organized by label combinations such as `FF`, `FR`, `RF`, and `RR`
- Metadata is encoded in filenames
- The notebook keeps the sampled videos in a mirrored folder tree under a working directory
- The sampled root is named using the target clip count, for example `sampled_uniform_970_videos`

## Batch and Split Plan
- Training workflow now uses a 970-clip subset for the Kaggle run
- Split ratios used by the trainers:
  - Train: 75%
  - Validation: 12.5%
  - Test: 12.5%
- Splitting is done by `av_type` so all four combinations stay represented across splits
- Kaggle defaults were tuned for a faster run:
  - Video batch size: 8
  - Audio batch size: 12
  - Fusion batch size: 192
  - Number of workers: 2 for video, audio, and fusion

## Model Stack
- Video model: ConvNeXt-Tiny backbone with a trainable classifier head
- Audio model: Wav2Vec2-Base with a trainable classifier head
- Fusion model: two-head multimodal fusion on frozen video and audio embeddings
- Inference path: the same ConvNeXt-Tiny video extractor plus Wav2Vec2 audio extractor feeding the fusion checkpoint

## Current Working Behavior
- Notebook output is streamed live so long cells do not appear frozen
- Video, audio, fusion, and inference steps print progress while running
- Final outputs are packaged into separate and combined zip archives for download
- Final summary cell prints the best metrics and download paths in one place

## Output Artifacts
- Video outputs: `video_outputs.zip`
- Audio outputs: `audio_outputs.zip`
- Fusion outputs: `fusion_outputs.zip`
- Combined archive: `all_deepfake_outputs.zip`
- Download index: `download_index.json`

## Notes
- The video trainer has been upgraded away from the original MobileViT backbone
- The notebook and trainers are now aligned around the same 970-clip batch workflow
- This file is meant to be a quick record of the dataset setup and batch-model configuration used for the current run