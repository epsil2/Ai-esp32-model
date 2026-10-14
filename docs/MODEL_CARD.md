# Model and artifact formats

PocketGPT: decoder-only Transformer; 12 layers; hidden size 448; 8 query heads and 2 KV heads; FFN 1,344; RoPE; RMSNorm; 6,144-token SentencePiece vocabulary. Runtime context is 64 tokens and generation is limited to 24 tokens.

The tied training architecture has **30,460,864 independent trained parameters**. The saved checkpoint contains **33,213,376 tensor elements**, because embedding/head values were serialized twice. Equal values in this checkpoint do not mean the two saved tensors share storage.

Deployment formats differ:

- Original mixed Q4/FP16 export: **25,746,576 bytes**; duplicated FP16 embedding/head storage.
- DevKit deduplicated export: **20,241,616 bytes** including a 64-byte validation footer. Transformer Q4 with FP16 scales; FP16 shared embedding/head. `lm_head.weight` aliases the embedding and adds zero payload bytes. `tools/verify_pack.py` verifies hashes, sizes, footer and alias metadata.

### XIAO storage layout

The XIAO package manifest and firmware layout show **21,717,088 bytes of model-weight files**:

- **Flash model-partition image:** 7,602,176 bytes. It also contains model regions beyond the transformer matrices.
- **Loaded into PSRAM at boot:** 6,877,248 bytes, copied from `transformer_psram_boot.bin` on the microSD card.
- **Remains on microSD:** 7,237,664 bytes: the 5,505,024-byte FP16 embedding table and 1,732,640 bytes of residual transformer matrices, read during inference.

The firmware layout places 20 transformer matrices (6,104,224 bytes) in Flash, 38 (6,877,248 bytes) in PSRAM and 26 (1,732,640 bytes) on microSD. The Flash model-partition image is larger than its transformer-matrix portion because it also contains other model regions.

Separate support files are not included in the 21,717,088-byte weight total: the two Flash cache components total **186,592 bytes**, and the three microSD tokenizer files total **358,523 bytes**. Do not interchange these XIAO assets with the DevKit pack.

The DevKit builder derives a deduplicated pack from a specific already-quantized mixed pack; it does not train or export a raw checkpoint. Pass both `--source` and `--out-dir`. Fixed binary input hashes prevent accidental use of another model. The original manifest's private path is not required or reproduced.

Training provenance is documented in [DATA_PROVENANCE.md](DATA_PROVENANCE.md), with inspected-source hashes in PROVENANCE_EVIDENCE.json. Four conversational revisions are recorded. Historical general-corpus versions and the exact tokenizer mix remain unresolved. The released weights and tokenizer are available through the [download and setup guide](ASSETS.md). Source attribution is in the [model release notes](MODEL_RELEASE_NOTES.md).

Recorded generation rates are approximately 1.41 tokens/s on the specified DevKit and 0.742 tokens/s on XIAO. These are prior single-run measurements, not a controlled cross-platform benchmark or a promise for another build.
