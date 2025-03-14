# LILO

LILO: Learning Interpretable Libraries by Compressing and Documenting Code ([arxiv/2310.19791](https://arxiv.org/abs/2310.19791))

![LILO Splash Figure](_assets/lilo_splash.png)

# Installation

To run LILO, you'll need an environment with:

- Python 3.7 (for backwards compatibility with DreamCoder)
- OCaml (required by DreamCoder)
- Rust (required by Stitch)

The easiest way to get this environment is to use the Docker container provided in this repository. If you'd like to build a local environment, see the instructions below.

## LILO Docker container (Recommended)

First, install [Docker](https://docs.docker.com/get-docker/) on your local machine. Then, pull the Docker container from Docker Hub:

```
docker pull gabegrand/lilo
```

Alternatively, you can clone the repo and build the Docker container locally:

```
git clone --recurse-submodules https://github.com/gabegrand/lilo
cd lilo
docker build -t lilo .
```

## Building a local LILO environment

LILO is compatible with both Linux (tested on Debian 12 "bookworm" and Ubuntu 20.04 "Focal Fossa") and MacOS (tested on M1 Mac running MacOS 12 Monterey).

### Prerequisites
- Conda / Miniconda: https://docs.conda.io/en/latest/miniconda.html
- OCaml: https://ocaml.org/install
- Rust: https://www.rust-lang.org/tools/install

### Building the Python environment

To build a local environment, first clone the repo:
```
git clone --recurse-submodules https://github.com/gabegrand/lilo
cd lilo
```

Next, install the `lilo` conda environment.
```
conda env create -f environment.yml
```
> [!NOTE]
> If this step is taking a long time, you may wish to install the [libmamba](https://conda.github.io/conda-libmamba-solver/getting-started/) solver, which significantly speeds up environment resolution.

### Building the OCaml binaries

```
cd dreamcoder
make setup-ocaml
make
```

## Testing the environment

If the environment is installed correctly, you should be able to activate it:
```
conda activate lilo
```
Next, from a Python shell, try constructing a simple `Program` object and performing type inference with `.infer()` and evaluation via `.evaluate()`:
```
>>> from run_experiment import *
>>> p = Program.parse("(_rconcat _x _y)")
>>> p.infer()
tsubstr
>>> p.evaluate([])
'xy'
```



# Running experiments

The general entry point for running experiments is `run_iterative_experiment.py`.

> [!NOTE]
> To run experiments with the OpenAI API, you'll need to set the `OPENAI_API_KEY` environment variable to your API key. You can find your API key at https://platform.openai.com/account/api-keys.
> ```
> export OPENAI_API_KEY=<sk-...a123>
> ```

## Models

Each model type is specified in a template file. For example, the template for the `lilo` model is `experiments_iterative/templates/template_lilo.json`. The `run_iterative_experiment.py` script takes an `--experiment_type` argument that specifies which template to use.

Below, we provide demo commands for running experiments with each model type on the REGEX domain. Note that these commands are designed to be runnable on a consumer-scale machine (e.g. a laptop). See the section below on full-scale experiments for replicating our experiments on a high-performance computing cluster.

### DreamCoder

[experiments_iterative/templates/template_dreamcoder.json](experiments_iterative/templates/template_dreamcoder.json)

```
python run_iterative_experiment.py \
  --experiment_name test_runs \
  --experiment_type dreamcoder \
  --domain re2 \
  --encoder re2 \
  --iterations 1 \
  --global_batch_sizes 32 \
  --enumeration_timeout 5 \
  --recognition_train_steps 100 \
  --verbose \
```

### LLM Solver

[experiments_iterative/templates/llm_solver.json](experiments_iterative/templates/llm_solver.json)

```
python run_iterative_experiment.py \
  --experiment_name test_runs \
  --experiment_type llm_solver \
  --domain re2 \
  --iterations 1 \
  --global_batch_sizes 32 \
  --random_seeds 111 \
  --init_frontiers_from_checkpoint \
  --resume_checkpoint_directory experiments_iterative/outputs/test_runs/domains/re2/dreamcoder/seed_111/dreamcoder_32 \
  --verbose \
```
> [!IMPORTANT]
> Requires running `python run_iterative_experiment.py` with `--experiment_type dreamcoder` for at least one iteration to generate an initial `frontiers.json` file.

### LILO

[experiments_iterative/templates/lilo.json](experiments_iterative/templates/lilo.json)

```
python run_iterative_experiment.py \
  --experiment_name test_runs \
  --experiment_type lilo \
  --domain re2 \
  --encoder re2 \
  --iterations 1 \
  --global_batch_sizes 32 \
  --enumeration_timeout 5 \
  --recognition_train_steps 100 \
  --random_seeds 111 \
  --init_frontiers_from_checkpoint \
  --resume_checkpoint_directory experiments_iterative/outputs/test_runs/domains/re2/dreamcoder/seed_111/dreamcoder_32 \
  --verbose \
```
> [!IMPORTANT]
> Requires running `python run_iterative_experiment.py` with `--experiment_type dreamcoder` for at least one iteration to generate an initial `frontiers.json` file.

## Resuming from checkpoints

There are two main use cases for resuming from a checkpoint:
- Resuming a run that was interrupted (e.g. due to a crash or a timeout)
- Initializing a new run with the frontiers from a prior run or model. This is used above for the LLM Solver and LILO models to provide an initial seed set of programs for few-shot prompting.

To resume from a prior run, use the `--resume_checkpoint_directory` flag:
```
--resume_checkpoint_directory experiments_iterative/outputs/<experiment_name>/domains/<domain>/<experiment_type>/seed_<seed>/<experiment_name>_<batch_size>
```

Note that you will also need to pass `--init_frontiers_from_checkpoint` to load the `frontiers.json` file, which contains the set of program solutions from the checkpoint.

### Resuming from checkpoint at every iteration

By default, resuming from checkpoint will only occur at the first iteration. This is useful in the case where you want to load from an initial `frontiers.json` and then continue learning from there. To resume from checkpoint at every iteration, use the `--init_frontiers_every_iteration` flag. This is usually only used for debugging or to resume a run that was interrupted.

## Domains

The `run_iterative_experiment.py` script takes a `--domain` argument that specifies which domain to use. The following domains are supported:

### REGEX

```
--domain re2 \
--encoder re2 \
--enumeration_timeout 1000 \
--iterations 16 \
```

### CLEVR

```
--domain clevr \
--encoder clevr \
--enumeration_timeout 600 \
--iterations 10 \
```

### LOGO

```
--domain logo \
--encoder LOGO \
--enumeration_timeout 1800 \
--iterations 10 \
```

## Full-scale experiments

Below, we include flags for replicating our experiments on a high-performance computing cluster. We ran our experiments on c5.24xlarge machines on AWS with 96 CPUs.

```
  --global_batch_sizes 96 \
  --recognition_train_steps 10000 \
  --random_seeds 111 222 333 \
```

# Data release

We make all data from our experiments public: `s3://lilo-experiments-release/lilo_arxiv_v1/`

To view this data in your browser, you can use the AWS CLI (requires AWS account sign-up): https://s3.console.aws.amazon.com/s3/buckets/lilo-experiments-release

# Citations

Thanks for citing this work! Please use the following archival citation from ICLR 2024.

```
@inproceedings{
  grand2024lilo,
  title={{LILO}: Learning Interpretable Libraries by Compressing and Documenting Code},
  author={Gabriel Grand and Lionel Wong and Maddy Bowers and Theo X. Olausson and Muxin Liu and Joshua B. Tenenbaum and Jacob Andreas},
  booktitle={The Twelfth International Conference on Learning Representations},
  year={2024},
  url={https://openreview.net/forum?id=TqYbAWKMIe}
}
```

# Acknowledgements

LILO inherits from the LAPS codebase (github.com/CatherineWong/laps). We gratefully acknowledge Lionel Wong for the use of their codebase as a foundation and for technical support.

# License

MIT License Copyright (c) 2023 Gabriel Grand

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
