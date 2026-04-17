# CLI and Script Reference

## Top-Level Commands

The main convenience entrypoint is `clairvoyante.py`. It dispatches to named submodules.

### Exposed Clairvoyante submodules

| Name | Underlying module | Purpose |
| --- | --- | --- |
| `callVarBamParallel` | `clairvoyante/callVarBamParallel.py` | Emit chunked `callVarBam.py` commands for parallel calling. |
| `callVarBam` | `clairvoyante/callVarBam.py` | Call variants directly from BAM through a three-stage pipeline. |
| `callVar` | `clairvoyante/callVar.py` | Call variants from prepared tensors. |
| `calTrainDevDiff` | `clairvoyante/calTrainDevDiff.py` | Compare training and development loss on a checkpoint. |
| `demoRun` | `clairvoyante/demoRun.py` | Train and test with the demo dataset. |
| `evaluateListOfModels` | `clairvoyante/evaluateListOfModels.py` | Evaluate many checkpoints on one dataset. |
| `evaluate` | `clairvoyante/evaluate.py` | Evaluate one checkpoint on labeled tensors. |
| `getEmbedding` | `clairvoyante/getEmbedding.py` | Generate TensorBoard projector embeddings. |
| `getTensorAndLayerPNG` | `clairvoyante/getTensorAndLayerPNG.py` | Render tensors and layer activations as PNGs. |
| `tensor2Bin` | `clairvoyante/tensor2Bin.py` | Convert text tensors into a binary pickled dataset. |
| `trainNonstop` | `clairvoyante/trainNonstop.py` | Continuous training with per-epoch checkpoints. |
| `train` | `clairvoyante/train.py` | Main adaptive training entrypoint. |
| `trainWithoutValidationNonstop` | `clairvoyante/trainWithoutValidationNonstop.py` | Continuous training without validation split. |

### Exposed data-prep submodules

| Name | Underlying module | Purpose |
| --- | --- | --- |
| `ChooseItemInBed` | `dataPrepScripts/ChooseItemInBed.py` | Filter records by BED overlap. |
| `CombineMultipleDatasetsForTraining` | `dataPrepScripts/CombineMultipleDatasetsForTraining.py` | Helper for assembling multi-dataset training inputs. |
| `CountNumInBed` | `dataPrepScripts/CountNumInBed.py` | Count BED-overlapping records. |
| `CreateTensor` | `dataPrepScripts/CreateTensor.py` | Build alignment tensors around candidate sites. |
| `ExtractVariantCandidates` | `dataPrepScripts/ExtractVariantCandidates.py` | Generate candidate loci from BAM alignments. |
| `GetTruth` | `dataPrepScripts/GetTruth.py` | Extract simplified truth calls from VCF. |
| `PairWithNonVariants` | `dataPrepScripts/PairWithNonVariants.py` | Mix truth-variant tensors with sampled non-variant tensors. |
| `RandomSampling` | `dataPrepScripts/RandomSampling.py` | Helper sampling script. |

## Primary Operational Scripts

## `callVarBam.py`

Purpose:

- direct BAM-to-VCF calling
- orchestration of preprocessing and inference subprocesses

Key inputs:

- `--chkpnt_fn`
- `--bam_fn`
- `--ref_fn`
- `--ctgName`
- optional `--bed_fn`
- optional `--vcf_fn`

Operational notes:

- requires `pypy`
- requires `samtools`
- defaults to enabling `--considerleftedge`
- chooses a random startup delay to reduce thread oversubscription when many jobs start at once

## `callVarBamParallel.py`

Purpose:

- split calling jobs by contig and genomic chunk
- print shell commands instead of executing them

Key inputs:

- `--ref_fn` with `.fai`
- `--bam_fn`
- `--chkpnt_fn`
- `--output_prefix`
- optional BED and candidate VCF

Operational notes:

- uses BED overlap to avoid emitting useless chunks
- defaults to `--tensorflowThreads 4`
- intended to be paired with GNU Parallel or similar tooling

## `callVar.py`

Purpose:

- score prepared tensors with a trained checkpoint
- render predictions as VCF

Key inputs:

- `--tensor_fn`
- `--chkpnt_fn`
- `--call_fn`
- optional `--ref_fn`
- optional `--qual`

Operational notes:

- can read tensors from `PIPE`
- supports v2/v3 and slim/full network selection
- writes VCF header with contigs if a reference `.fai` is available

## `train.py`

Purpose:

- main model training entrypoint

Key inputs:

- `--bin_fn` or raw `--tensor_fn` plus `--var_fn`
- optional `--bed_fn`
- optional `--chkpnt_fn`
- optional `--ochk_prefix`
- optional `--olog_dir`

Operational notes:

- defaults to version 3
- can resume from a checkpoint
- saves checkpoints only if `--ochk_prefix` is provided

## `tensor2Bin.py`

Purpose:

- precompute a binary training/evaluation dataset from text tensors

Why it matters:

- avoids reloading and reparsing large gzip text tensors repeatedly
- stores BLosc-compressed blocks suitable for `utils_v2.py`

## Data Preparation Script Details

## `ExtractVariantCandidates.py`

Important flags:

| Flag | Meaning |
| --- | --- |
| `--threshold` | Minimum allele fraction for a candidate site. |
| `--minCoverage` | Minimum read depth for a candidate site. |
| `--minMQ` | Minimum mapping quality. |
| `--gen4Training` | Broad candidate generation for training data creation. |
| `--ctgName`, `--ctgStart`, `--ctgEnd` | Region restriction. |

## `CreateTensor.py`

Important flags:

| Flag | Meaning |
| --- | --- |
| `--can_fn` | Candidate or truth-site input. |
| `--considerleftedge` | Expand candidate activation logic to left-edge overlapping reads. |
| `--dcov` | Depth cap per position. |
| `--minCoverage` | Minimum depth to emit a tensor. |

## `GetTruth.py`

Important flags:

| Flag | Meaning |
| --- | --- |
| `--vcf_fn` | Truth VCF input, usually gzipped. |
| `--var_fn` | Output truth record file or `PIPE`. |
| `--ctgName`, `--ctgStart`, `--ctgEnd` | Region restriction. |

Implementation note:

- if the VCF is tabix-indexed and `tabix` exists, the script uses region queries
- otherwise it scans the decompressed file

## `PairWithNonVariants.py`

Important flags:

| Flag | Meaning |
| --- | --- |
| `--tensor_can_fn` | Candidate/non-variant tensors. |
| `--tensor_var_fn` | Truth-variant tensors. |
| `--bed_fn` | Optional filter for usable regions. |
| `--amp` | Ratio multiplier for non-variant sampling. |

## Secondary Utility Scripts

These are useful but not central to the main execution path:

| Script | Summary |
| --- | --- |
| `calTrainDevDiff.py` | Compare training and development loss on an existing checkpoint. |
| `CombineMultipleDatasetsForTraining.py` | Merge multiple datasets before later pairing/binning steps. |
| `ChooseItemInBed.py` | BED-based filter helper. |
| `CountNumInBed.py` | BED-based counting helper. |
| `RandomSampling.py` | Input subsampling helper. |

## Example Workflow Matrix

| Goal | Minimal script chain |
| --- | --- |
| Call variants from BAM | `callVarBam.py` |
| Parallelize BAM calling | `callVarBamParallel.py` + external parallel runner |
| Call variants from tensors | `callVar.py` |
| Train a model from prepared tensors | `tensor2Bin.py` -> `train.py` |
| Evaluate a checkpoint | `evaluate.py` |
| Visualize learned representations | `getEmbedding.py` or `getTensorAndLayerPNG.py` |
