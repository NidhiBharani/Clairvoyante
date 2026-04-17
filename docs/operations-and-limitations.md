# Operations and Limitations

## Runtime Environment

This repository is an older research codebase. The implementation assumptions are important to surface explicitly.

## Language and Framework Assumptions

- Primary implementation target: Python 2.7
- Deep learning stack: TensorFlow 1.x
- Fast preprocessing path: `pypy`

The root README and Dockerfile are aligned on the TensorFlow 1 generation, and the source still uses Python 2 syntax such as:

- `print >>`
- `xrange`
- implicit relative-import-era patterns

## Python 3 Support

Python 3 is not native in the repository. Instead, `port23.py` attempts an in-place conversion by:

1. running `2to3` on files in `clairvoyante/`
2. inserting the package path to help with Python-2-style imports
3. removing one specific relative-import artifact

This should be treated as a compatibility helper, not as a maintained Python 3 port.

## External Tool Dependencies

The code shells out to several external executables:

| Tool | Used for |
| --- | --- |
| `samtools` | FASTA indexing queries and BAM streaming. |
| `gzip` | Reading and writing almost all intermediate text files. |
| `tabix` | Optional indexed VCF region access in `GetTruth.py`. |
| `pypy` | High-performance preprocessing scripts. |
| `parallel` | Suggested by README for running chunked calling jobs. |
| `taskset` | Optional CPU affinity in `callVarBam.py`. |

If these are unavailable, parts of the workflow will fail or run significantly slower.

## Docker Image

The repository ships a simple Dockerfile based on:

- `tensorflow/tensorflow:1.12.0`

It installs:

- `pypy`
- `pypy-dev`
- `samtools`
- `wget`
- `parallel`

and then installs:

- `requirements.txt` under CPython
- `requirements_pypy.txt` under PyPy

This image is best interpreted as a reproducibility aid for the historical environment.

## Package Dependencies

From `requirements.txt`:

- `tensorflow==1.12.0`
- `numpy`
- `blosc`
- `intervaltree==2.0.1`

The README also calls out `intervaltree` and `blosc` as practical installation pain points on some systems.

## Performance Model

The repository is optimized around the idea that:

- preprocessing is CPU-bound and benefits substantially from `pypy`
- model inference/training depends on TensorFlow and standard Python
- direct BAM calling is bottlenecked more by tensor creation than by GPU inference

Operational consequences:

- `callVarBam.py` prefers CPU execution for parallel jobs
- `callVarBamParallel.py` is the intended whole-genome scaling mechanism
- training is the workflow that benefits most from a GPU

## Threading and Parallelism

There are three distinct sources of parallel behavior:

1. TensorFlow intra-op threads via `param.NUM_THREADS`
2. external process parallelism via `callVarBamParallel.py`
3. small overlap threads inside `train.py` and `callVar.py` to overlap compute and output/decompression

This makes total machine utilization sensitive to:

- the number of simultaneously launched chunk jobs
- TensorFlow thread counts per job
- whether other stages are also CPU-intensive

The random startup delay in `callVarBam.py` exists specifically to reduce simultaneous thread spikes.

## Format and Data Consistency Requirements

Several assumptions are enforced only implicitly:

- FASTA must have a valid `.fai`
- tensor geometry must match between `dataPrepScripts/param.py` and `clairvoyante/param.py`
- BED, BAM, VCF, and FASTA contig naming must be consistent
- checkpoints are referenced by prefix, but the corresponding `.meta` file must exist

If any of these assumptions break, failure often happens at runtime rather than at startup.

## Functional Limitations

The root README and implementation together point to several practical limitations.

### Multi-allelic variants

Version 3 effectively handles a single alternative allele per site. The README explicitly notes limited handling for `GT 1/2` style multi-allelic variants.

### Indel length modeling

The model predicts explicit short indel length classes and uses heuristics for longer events. Very long events may become symbolic structural variant alleles rather than fully reconstructed sequences.

### Legacy dependency surface

The TensorFlow 1 and Python 2 foundation makes the code harder to:

- install on modern systems
- integrate into current Python tooling
- extend with contemporary ML libraries

### Tight coupling through globals

Shared constants in `param.py` files act as a hidden contract across scripts. Changing tensor dimensions or flanking window sizes requires coordinated edits in multiple places.

## Risks for Modernization

If this repository is to be maintained or extended, the main technical risks are:

1. Python 2 specific syntax and behavior across the codebase.
2. TensorFlow 1 APIs such as `tf.Session`, `tf.placeholder`, `tf.layers`, and `tf.contrib`.
3. Process-based glue that depends on shell utilities instead of explicit Python interfaces.
4. Informal intermediate formats without schema validation.

## Practical Guidance for Maintainers

### For using the code as-is

- Prefer the Dockerfile or an isolated legacy environment.
- Keep both `pypy` and standard Python available.
- Verify `samtools`, FASTA indices, and contig naming before debugging model behavior.

### For documenting or refactoring the code

- Treat `callVarBam.py` as the real orchestration center.
- Treat `utils_v2.py` as the canonical source for dataset loading and label encoding.
- Treat `clairvoyante_v3.py` as the default model architecture.
- Keep tensor geometry changes synchronized across both `param.py` files.

### For future reimplementation

The cleanest modernization boundary is:

- preserve the tensor semantics and label encoding first
- then replace the TensorFlow 1 model/runtime independently
- then replace the shell-pipeline orchestration with explicit Python APIs if needed
