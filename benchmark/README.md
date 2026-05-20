# PhysX-Omni Benchmark

This folder contains the metric-first benchmark pipeline for evaluating generated 3D objects with VLM judges.

Run commands from the repository root:

```bash
cd PhysX-Omni
```

The pipeline is:

```text
physx_result/ model outputs
  -> generate metric evidence assets
  -> build one manifest per metric
  -> run the VLM judge with the corresponding prompt
  -> aggregate raw JSON outputs into CSV summaries
```

## Installation

See [INSTALL.md](INSTALL.md) for CUDA, Blender, MuJoCo, Genesis, EGL/OpenGL,
ffmpeg, and VLM environment notes.

Quick start:

```bash
conda env create -f benchmark/environment.yml
conda activate physx-omni-benchmark

# Optional: choose a Hugging Face cache or local model path.
export HF_HOME=hf_cache
export MODEL_ID=Qwen/Qwen3.5-122B-A10B

# Headless rendering defaults.
export MUJOCO_GL=egl
export PYOPENGL_PLATFORM=egl
export PYGLET_HEADLESS=true
```

Before a large run, execute the tiny smoke test. It does not require a VLM or
GPU; it checks manifest building, aggregation, and denominator validation:

```bash
bash benchmark/scripts/run_tiny_smoke_test.sh
```

## Configuration

Copy and edit the path config:

```bash
cp benchmark/configs/paths.example.yaml benchmark/configs/paths.yaml
```

All full-run scripts read `CONFIG`, `METHODS`, and `DATASETS` from the
environment:

```bash
CONFIG=benchmark/configs/paths.yaml \
METHODS="ours physxanything physxgen" \
DATASETS="mobility verse inthewild" \
bash benchmark/scripts/run_aps.sh
```

Useful switches:

```bash
RUN_VLM=0          # build assets/manifests only, skip model inference
RUN_AGGREGATE=0   # skip aggregation
RUN_VALIDATE=0    # skip denominator validation
LIMIT=5           # smoke-test only for scripts whose manifest builder supports it
```

## Metrics

| Metric | Meaning | Required evidence |
|---|---|---|
| RQS | Render Quality Score | sampled rendered views + quality reference image |
| MCS | Multi-view Consistency Score | sampled rendered views |
| DCS | Description Consistency Score | one rendered color view + same-view black/white part mask + reference description |
| DQS | Dimension Quality Score | condition image + generated dimension metadata |
| APS | Affordance Plausibility Score | condition image + affordance heatmap views |
| KPS | Kinematic Plausibility Score | condition image + standardized articulation video |
| MPS | Material Plausibility Score | condition image + water video + floor video + generated material parameters |

## Directory Layout

```text
benchmark/
├── assets/
│   ├── quality_reference/image.png
│   └── description/descript_*.json
├── configs/paths.example.yaml
├── prompts/
├── code/
│   ├── vlm/multi.py
│   ├── manifests/
│   ├── aggregation/
│   ├── scoring/
│   ├── assets/
│   └── render/
│       ├── views/
│       ├── kinematic/
│       └── material_batch/
└── scripts/run_live_aggregation.sh
```

Generated files are written under ignored folders:

```text
benchmark/benchmark_assets/
benchmark/benchmark_manifests/
benchmark/benchmark_results/
benchmark/logs/
```

## Expected Input

Place model outputs under `physx_result/`:

```text
physx_result/
├── demo_mobility/
├── demo_verse/
├── demo_inthewild/
├── ours_mobility_181500/
├── ours_verse_181500/
├── ours_inthewild_181500/
├── output_physxanything_mobility/
├── output_physxanything_verse/
├── output_physxanything_inthewild/
├── outputs_physxgen_mobility/
├── outputs_physxgen_verse/
├── outputs_physxgen_inthewild/
├── output_articulateanything_mobility/
├── output_articulateanything_verse/
└── output_articulateanything_inthewild/
```

Method conventions are encoded in the manifest builders:

- `ours` and `physxanything`: usually provide `basic.xml`, `sample.glb`, `basic_info.json`, or `basic_info.txt`.
- `physxgen`: usually provides `texture.glb`, `scale.npy`, and optionally `mesh/basic.urdf`.
- `articulateanything`: KPS should be rendered from URDF evidence using our renderer.
- APS masks are expected under object-level `affordance/`.
- DCS masks are expected under object-level `description/`.

## Full-run Scripts

Each script performs the metric-specific manifest build, VLM judging, aggregation,
and denominator validation. Some expensive asset-generation stages are opt-in so
that users can resume safely on large clusters.

```bash
# Shared condition-image preparation.
bash benchmark/scripts/prepare_condition_images.sh

# Static / functional metrics.
bash benchmark/scripts/run_rqs.sh
bash benchmark/scripts/run_mcs.sh
bash benchmark/scripts/run_dcs.sh
bash benchmark/scripts/run_dqs.sh
bash benchmark/scripts/run_aps.sh

# Dynamic / material metrics.
bash benchmark/scripts/run_kps.sh
bash benchmark/scripts/run_mps.sh
```

Optional expensive stages:

```bash
RUN_RENDER=1 bash benchmark/scripts/run_rqs.sh       # Blender rendered views
RUN_RENDER=1 bash benchmark/scripts/run_mcs.sh
RENDER_KPS=1 bash benchmark/scripts/run_kps.sh       # MuJoCo KPS videos
RUN_WATERTIGHT=1 RENDER_MATERIAL=1 bash benchmark/scripts/run_mps.sh
```

To validate counts after any run:

```bash
bash benchmark/scripts/run_denominator_validation.sh
```

## Shared Step: Condition Images

Condition images are shared by DQS, APS, KPS, and MPS. It can be downloaded from [PhysX-Bench](https://huggingface.co/datasets/PhysX-Omni/PhysX-Bench)

Input:

```text
physx_result/demo_<dataset>/<object_id>.png
```

Command:

```bash
python3 benchmark/code/assets/prepare_demo_condition_images.py \
  --input-dir physx_result/demo_mobility \
  --dataset mobility \
  --output-root benchmark/benchmark_assets/condition_images \
  --symlink --skip-existing

python3 benchmark/code/assets/prepare_demo_condition_images.py \
  --input-dir physx_result/demo_verse \
  --dataset verse \
  --output-root benchmark/benchmark_assets/condition_images \
  --symlink --skip-existing

python3 benchmark/code/assets/prepare_demo_condition_images.py \
  --input-dir physx_result/demo_inthewild \
  --dataset inthewild \
  --output-root benchmark/benchmark_assets/condition_images \
  --symlink --skip-existing
```

Output:

```text
benchmark/benchmark_assets/condition_images/<dataset>/<object_id>/first_frame.png
```

## RQS: Render Quality

Asset input:

```text
physx_result/<method_result_folder>/<object_id>/
```

Asset generation:

```bash
export BLENDER_BIN=blender
export BLENDER_DEVICE=GPU

python3 benchmark/code/render/views/render_adaptive.py \
  --dataset_roots physx_result/ours_mobility_181500 \
  --output_root benchmark/benchmark_assets/rendered_views/description \
  --gpus 0 1 2 3 \
  --view_mode orbit \
  --num_views 30
```

Asset output:

```text
benchmark/benchmark_assets/rendered_views/description/<source_folder>/<object_id>/000.png ... 029.png
```

Manifest:

```bash
python3 benchmark/code/manifests/build_render_view_manifest.py \
  --root benchmark/benchmark_assets/rendered_views/description \
  --metric rqs \
  --source ours_mobility_181500:ours:mobility \
  --output-jsonl benchmark/benchmark_manifests/rqs.jsonl \
  --output-csv benchmark/benchmark_manifests/rqs.csv
```

VLM:

```bash
python3 benchmark/code/vlm/multi.py \
  --model-id Qwen/Qwen3.5-122B-A10B \
  --pairs-manifest benchmark/benchmark_manifests/rqs.jsonl \
  --prompts-file benchmark/prompts/prompts_quality.yaml \
  --output-root benchmark/benchmark_results/raw_vlm_outputs
```

## MCS: Multi-view Consistency

MCS uses the same rendered view assets as RQS.

Manifest:

```bash
python3 benchmark/code/manifests/build_render_view_manifest.py \
  --root benchmark/benchmark_assets/rendered_views/description \
  --metric mcs \
  --source ours_mobility_181500:ours:mobility \
  --output-jsonl benchmark/benchmark_manifests/mcs.jsonl \
  --output-csv benchmark/benchmark_manifests/mcs.csv
```

VLM:

```bash
python3 benchmark/code/vlm/multi.py \
  --model-id Qwen/Qwen3.5-122B-A10B \
  --pairs-manifest benchmark/benchmark_manifests/mcs.jsonl \
  --prompts-file benchmark/prompts/prompts_consistency.yaml \
  --output-root benchmark/benchmark_results/raw_vlm_outputs
```

## DCS: Description Consistency

DCS uses exactly one color render and one same-view black/white mask per object. The mask selects the generated region to compare against the reference description.

Inputs:

```text
benchmark/benchmark_assets/rendered_views/description/<source_folder>/<object_id>/<view>.png
physx_result/<source_folder>/<object_id>/description/<view>.png
benchmark/assets/description/descript_<dataset>.json
```

Manifest:

```bash
python3 benchmark/code/manifests/build_description_mask_manifest.py \
  --render-root benchmark/benchmark_assets/rendered_views/description \
  --result-root physx_result \
  --description-root benchmark/assets/description \
  --source ours_mobility_181500:ours_mobility_181500:ours:mobility \
  --output-jsonl benchmark/benchmark_manifests/dcs.jsonl \
  --output-csv benchmark/benchmark_manifests/dcs.csv
```

Output manifest rows include `view_image_paths`, `mask_image_paths`, and `reference_description`. Missing color images, masks, or descriptions are retained and later counted as zero.

VLM:

```bash
python3 benchmark/code/vlm/multi.py \
  --model-id Qwen/Qwen3.5-122B-A10B \
  --pairs-manifest benchmark/benchmark_manifests/dcs.jsonl \
  --prompts-file benchmark/prompts/prompts_description.yaml \
  --output-root benchmark/benchmark_results/raw_vlm_outputs
```

## DQS: Dimension Quality

DQS first asks the VLM for an image-based size prior, then computes the final score deterministically.

Inputs:

```text
benchmark/benchmark_assets/condition_images/<dataset>/<object_id>/first_frame.png
physx_result/<source_folder>/<object_id>/basic_info.json or basic_info.txt
physx_result/<source_folder>/<object_id>/scale.npy for physxgen
```

Manifest:

```bash
python3 benchmark/code/manifests/build_dimension_manifest.py \
  --physx-result-root physx_result \
  --condition-image-root benchmark/benchmark_assets/condition_images \
  --output-jsonl benchmark/benchmark_manifests/dqs_prior.jsonl \
  --output-csv benchmark/benchmark_manifests/dqs_prior.csv \
  --methods ours physxanything physxgen \
  --datasets mobility verse inthewild
```

VLM prior:

```bash
python3 benchmark/code/vlm/multi.py \
  --model-id Qwen/Qwen3.5-122B-A10B \
  --pairs-manifest benchmark/benchmark_manifests/dqs_prior.jsonl \
  --prompts-file benchmark/prompts/prompts_dimension.yaml \
  --output-root benchmark/benchmark_results/raw_vlm_outputs
```

Deterministic scoring:

```bash
python3 benchmark/code/scoring/score_dimension_results.py \
  --manifest benchmark/benchmark_manifests/dqs_prior.jsonl \
  --prior-results-root benchmark/benchmark_results/raw_vlm_outputs \
  --output-root benchmark/benchmark_results/raw_vlm_outputs/dimension_scoring \
  --object-csv benchmark/benchmark_results/object_level_scores/dimension_scores.csv \
  --summary-csv benchmark/benchmark_results/dataset_level_scores/dimension_metric_summary.csv
```

## APS: Affordance Plausibility

Inputs:

```text
physx_result/<source_folder>/<object_id>/affordance/*.png
benchmark/benchmark_assets/condition_images/<dataset>/<object_id>/first_frame.png
```

Asset generation:

```bash
python3 benchmark/code/assets/prepare_affordance_heatmaps.py \
  --physx-result-root physx_result \
  --output-root benchmark/benchmark_assets/affordance_heatmaps \
  --methods ours physxanything physxgen \
  --datasets mobility verse inthewild \
  --skip-existing
```

Asset output:

```text
benchmark/benchmark_assets/affordance_heatmaps/<method>/<dataset>/<object_id>/
```

Manifest:

```bash
python3 benchmark/code/manifests/build_affordance_manifest.py \
  --physx-result-root physx_result \
  --condition-image-root benchmark/benchmark_assets/condition_images \
  --heatmap-root benchmark/benchmark_assets/affordance_heatmaps \
  --output-jsonl benchmark/benchmark_manifests/aps.jsonl \
  --output-csv benchmark/benchmark_manifests/aps.csv \
  --methods ours physxanything physxgen \
  --datasets mobility verse inthewild
```

VLM:

```bash
python3 benchmark/code/vlm/multi.py \
  --model-id Qwen/Qwen3.5-122B-A10B \
  --pairs-manifest benchmark/benchmark_manifests/aps.jsonl \
  --prompts-file benchmark/prompts/prompts_affordance.yaml \
  --output-root benchmark/benchmark_results/raw_vlm_outputs
```

## KPS: Kinematic Plausibility

KPS should use standardized videos rendered by this repository. Use
`benchmark/code/render/kinematic/kinematic_articulation_demo.py` for all KPS
video rendering. The script supports MJCF XML directly and can convert URDF
visual meshes into a temporary MJCF with `--urdf-visual-only`, so XML-based and
URDF-based methods share one rendering entry point.

By default, the renderer conservatively lifts visible non-plane object geometry
above the ground plane before rendering. This avoids half-buried generated
assets. Disable this only for debugging with `--no-lift-above-ground`.

Inputs:

```text
benchmark/benchmark_assets/condition_images/<dataset>/<object_id>/first_frame.png
physx_result/<source_folder>/<object_id>/basic.xml or basic.urdf
physx_result/<source_folder>/<object_id>/mesh/basic.urdf for physxgen
physx_result/<source_folder>/<object_id>/joint_actor/.../*.urdf for articulateanything
```

Build a source manifest:

```bash
python3 benchmark/code/manifests/build_kinematic_manifest.py \
  --physx-result-root physx_result \
  --condition-image-root benchmark/benchmark_assets/condition_images \
  --video-root benchmark/benchmark_assets/kinematic_videos \
  --output-jsonl benchmark/benchmark_manifests/kps_source.jsonl \
  --output-csv benchmark/benchmark_manifests/kps_source.csv \
  --methods ours physxanything physxgen articulateanything \
  --datasets mobility verse inthewild
```

Render XML rows, for example `ours` on Mobility:

```bash
python3 benchmark/code/render/kinematic/kinematic_articulation_demo.py \
  --batch-input-dir physx_result/ours_mobility_181500 \
  --batch-xml-name basic.xml \
  --output-root benchmark/benchmark_assets/kinematic_videos/ours/mobility \
  --input-root physx_result/ours_mobility_181500 \
  --return-mode hold \
  --include-root-free \
  --scene-preset ours_xml \
  --view 135,-18 \
  --view 45,-18 \
  --skip-existing
```

Render PhysXGen URDF rows:

```bash
python3 benchmark/code/render/kinematic/kinematic_articulation_demo.py \
  --batch-input-dir physx_result/outputs_physxgen_mobility \
  --batch-xml-name mesh/basic.urdf \
  --output-root benchmark/benchmark_assets/kinematic_videos/physxgen/mobility \
  --input-root physx_result/outputs_physxgen_mobility \
  --drop-xml-parent-levels 1 \
  --return-mode hold \
  --include-root-free \
  --urdf-visual-only \
  --asset-dir-from-xml-parent \
  --scene-preset ours_xml \
  --view 135,-18 \
  --view 45,-18 \
  --skip-existing
```

Render ArticulateAnything URDF rows:

```bash
python3 benchmark/code/render/kinematic/kinematic_articulation_demo.py \
  --batch-input-dir physx_result/output_articulateanything_mobility \
  --batch-xml-name joint_actor/iter_0/seed_0/mobility.urdf \
  --output-root benchmark/benchmark_assets/kinematic_videos/articulateanything/mobility \
  --input-root physx_result/output_articulateanything_mobility \
  --drop-xml-parent-levels 3 \
  --return-mode hold \
  --include-root-free \
  --urdf-visual-only \
  --asset-dir-from-sample-objs \
  --scene-preset ours_xml \
  --view 135,-18 \
  --view 45,-18 \
  --skip-existing
```

Use the same pattern for `verse` and `inthewild` by changing the source folder,
method/dataset output folder, and `--input-root`.

Rebuild the final manifest after rendering:

```bash
python3 benchmark/code/manifests/build_kinematic_manifest.py \
  --physx-result-root physx_result \
  --condition-image-root benchmark/benchmark_assets/condition_images \
  --video-root benchmark/benchmark_assets/kinematic_videos \
  --output-jsonl benchmark/benchmark_manifests/kps.jsonl \
  --output-csv benchmark/benchmark_manifests/kps.csv \
  --methods ours physxanything physxgen articulateanything \
  --datasets mobility verse inthewild
```

VLM:

```bash
python3 benchmark/code/vlm/multi.py \
  --model-id Qwen/Qwen3.5-122B-A10B \
  --pairs-manifest benchmark/benchmark_manifests/kps.jsonl \
  --prompts-file benchmark/prompts/prompts_vaps_english.yaml \
  --output-root benchmark/benchmark_results/raw_vlm_outputs
```

## MPS: Material Plausibility

MPS uses water and floor simulations.

Inputs:

```text
physx_result/watertightFix_max3000/<source_folder>/<object_id>/
physx_result/material_metric_json/origin/<source_folder>/<object_id>.json
physx_result/<source_folder>/<object_id>/basic_info.json or basic_info.txt
benchmark/benchmark_assets/condition_images/<dataset>/<object_id>/first_frame.png
```

If your material metric JSON files come from another preprocessing folder, copy
or symlink them into `physx_result/material_metric_json/origin/<source_folder>/`.

Generate watertight FEM proxy meshes:

```bash
python3 benchmark/code/render/material_batch/utils/voxel_remesh.py \
  --watertight-batch \
  --output-root physx_result/watertightFix_max3000 \
  --max-faces 3000 \
  --workers 64 \
  --skip-existing
```

Generate floor videos:

```bash
python3 benchmark/code/render/material_batch/floor/multi_gpu_batch.py \
  --config benchmark/code/render/material_batch/floor/config.toml \
  --pairs ours-mobility,physxanything-mobility \
  --gpus 0,1,2,3 \
  --resume --skip-existing
```

Generate water videos:

```bash
python3 benchmark/code/render/material_batch/water/multi_gpu_batch.py \
  --config benchmark/code/render/material_batch/water/config.toml \
  --pairs ours-mobility,physxanything-mobility \
  --gpus 0,1,2,3 \
  --resume --skip-existing
```

Build manifest:

```bash
python3 benchmark/code/manifests/build_material_manifest.py \
  --physx-result-root physx_result \
  --water-root benchmark/benchmark_assets/material_videos/water \
  --floor-root benchmark/benchmark_assets/material_videos_v2/floor \
  --output-jsonl benchmark/benchmark_manifests/mps.jsonl \
  --output-csv benchmark/benchmark_manifests/mps.csv \
  --methods ours physxanything \
  --datasets mobility verse inthewild
```

VLM:

```bash
python3 benchmark/code/vlm/multi.py \
  --model-id Qwen/Qwen3.5-122B-A10B \
  --pairs-manifest benchmark/benchmark_manifests/mps.jsonl \
  --prompts-file benchmark/prompts/prompts_material.yaml \
  --output-root benchmark/benchmark_results/raw_vlm_outputs \
  --video-num-frames 20
```

## Aggregation

Run after one or more metric runs finish:

```bash
python3 benchmark/code/aggregation/aggregate_vlm_results.py \
  --results-root benchmark/benchmark_results/raw_vlm_outputs \
  --object-csv benchmark/benchmark_results/object_level_scores/object_scores_long.csv \
  --summary-csv benchmark/benchmark_results/dataset_level_scores/dataset_metric_summary.csv \
  --submetric-csv benchmark/benchmark_results/dataset_level_scores/dataset_submetric_summary.csv \
  --errors-jsonl benchmark/benchmark_results/logs/aggregate_errors.jsonl
```

Live aggregation:

```bash
LIVE_AGG_INTERVAL_SEC=600 bash benchmark/scripts/run_live_aggregation.sh
```

## Denominator Validation

Run denominator validation before reporting benchmark numbers:

```bash
python3 benchmark/code/validation/validate_denominators.py \
  --manifest-root benchmark/benchmark_manifests \
  --results-root benchmark/benchmark_results/raw_vlm_outputs \
  --object-csv benchmark/benchmark_results/object_level_scores/object_scores_long.csv \
  --summary-csv benchmark/benchmark_results/dataset_level_scores/dataset_metric_summary.csv \
  --output-csv benchmark/benchmark_results/logs/denominator_validation.csv \
  --errors-jsonl benchmark/benchmark_results/logs/denominator_validation_errors.jsonl \
  --strict
```

The validation CSV reports, for each `method / dataset / metric`:

- expected manifest count;
- ready count;
- missing-zero count;
- parsed raw result count before and after deduplication;
- auto-scored zero count;
- final object-level and summary-level counts;
- duplicate or extra raw outputs;
- sample missing object ids when counts do not match.

If `--strict` is set, the script exits non-zero when count mismatches are found.

## Missing Evidence Policy

Missing required evidence remains in the benchmark denominator:

- missing RQS/MCS rendered views become score 0;
- missing DCS color/mask/description evidence becomes DCS 0;
- malformed or missing DQS generated dimensions become DQS 0;
- missing APS heatmaps become APS 0;
- missing KPS videos become KPS 0 for active methods;
- missing MPS water/floor video pairs become MPS 0.

This keeps model/dataset comparisons denominator-consistent.
