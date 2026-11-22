# fastBEST
fastBE supplementary tools (fastBEST) is a [Snakemake](https://snakemake.readthedocs.io/en/stable/) workflow that bundles the generic workflow from variant calling with Mutect2, to clonal denvolution with [fastBE](https://github.com/raphael-group/fastBE). A clonal tree was inferred with fasBE with it's search method, and the variants were clustered with fastBE's cluster method, and the optimal number of clones were inferred with the [kneedle algorithim](https://kneed.readthedocs.io/en/stable/).

------------

# Overview

1. [Setup](#setup)
2. [Overview](#overview)
3. [Usage](#usage)
4. [Outputs](#workflow-output)
5. [Workflow DAG](#workflowDAG)
6. [Citation](#citation)

--------------

# Dependencies

* [uv](https://docs.astral.sh/uv/), installs Snakemake and its SLURM plugin from `pyproject.toml`
* [conda](https://github.com/conda-forge/miniforge), version >24.1.2, builds the per-rule tool environments in `workflow/envs/`
* Python 3.12 or 3.13 (uv can install this for you). Python 3.14+ is not supported yet because Snakemake's `immutables` dependency has no wheels for it.

---------------

# Setup

Snakemake itself is installed with uv, while the bioinformatics tools each rule uses (bwa-mem2, GATK, bcftools, fastBE, ...) are installed by Snakemake into conda environments on first run.

1. Install [uv](https://docs.astral.sh/uv/getting-started/installation/):

```
curl -LsSf https://astral.sh/uv/install.sh | sh
```

2. Install `conda` through [miniforge](https://github.com/conda-forge/miniforge#install), and add the bioconda and conda-forge channels:

```
conda config --add channels bioconda
conda config --add channels conda-forge
```

3. Clone this repository and install Snakemake. uv reads `.python-version`, downloads Python 3.13 if needed, and creates `.venv` from `uv.lock`:

```
git clone <repository-url> fastBEST
cd fastBEST
uv python install 3.13
uv sync
uv run snakemake --version
```

---------------

# Overview

Our Snakemake pipeline's main function is to automate variant calling with Mutect2, phylogenetic reconstruction with fastBE, and subsequent analysis with custom-made python scripts to generate cancer cell fraction (CCF) stacked bar chart of the clonal composition of a sample, a phylogenetic reconstructed tree from fastBE outputs, and a fishplot to visualize the evolutionary dynamics of a cohort overtime.

One major advantage Snakemake worklows, along with other workflow management systems is the utilization of a configuration file to generalize a pipeline. We split the workflow to use two configuration files: one config file for samples. This is in the format;

1. `Samples` - the samples dictionary where you will place the path to your input BAM files.
2. `sample_name` - The name of the sample you are analyzing. This will be used to label newly created folders where your outputs will be placed.
3. `sample rows` - First row contains path to `tumor bam files`. Second row holds the `name of normal sample`. Final row is the path to `normal bam file`.

> [!IMPORTANT]
> This workflow is intended for use with tumor-normal samples. If you wish to use a panel of normals, or tumor only: modify rules/mutect2.smk to alter the variant calling shell command.

We used a second configuration file to store other necessary files for this workflow;

1. `Reference Genome`
2. `Intervals List`
3. `Known Polymorphic Sites`
4. `Germline Resources`

The default parameters of this workflow strictly filters the called variants by removig indels, and restricting to only one called variant per loci. Additionally, all C > T, and G > A variants are removed. We strictly filtered the calls due to time complexity we experienced with clonal deconvolution with fastBE. Our initial testing found that the number variants exceeding 2500 typically led to unreasonably long compute times, hence justifying the filtering steps. 

> [!TIP]
> You can manually alter filtering or any other steps in the workflow by modifying or removing specific rules in the workflow/rules folder.



---------------

# Usage

All commands are run from the repository root. Prefix them with `uv run`, or activate the environment once with `source .venv/bin/activate`.

1. Run a dry-run to test if workflow produces correct outputs

```
uv run snakemake --use-conda --cores <num-of-cpu-cores> -n
```

2. Run the pipeline

```
uv run snakemake --use-conda --cores <num-of-cpu-cores>
```


# Snakemake with SLURM

On a SLURM cluster, Snakemake can submit each rule as its own job using [snakemake-executor-plugin-slurm](https://snakemake.github.io/snakemake-plugin-catalog/plugins/executor/slurm.html), which `uv sync` already installs. Memory, runtime and threads are set per rule in `workflow/rules/`; settings specific to your cluster live in a profile.

1. Edit `workflow/profiles/slurm/config.yaml` and replace every value marked `CHANGE ME` (account and partitions). To find your cluster's values:

```
sinfo -s                                                  # partitions and time limits
sacctmgr show assoc user=$USER format=account,partition   # your accounts
scontrol show partition <name>                            # max memory / walltime
```

2. From a login node (inside `tmux` or `screen` so it survives disconnects), dry-run and then launch:

```
uv run snakemake --profile workflow/profiles/slurm -n
uv run snakemake --profile workflow/profiles/slurm
```

> [!TIP]
> If your cluster does not allow long-running processes on login nodes, submit the Snakemake controller itself as a small job (1 CPU, ~2 GB, walltime longer than the whole workflow) that runs `uv run snakemake --profile workflow/profiles/slurm`.

For other schedulers (PBS/Torque, SGE, LSF), copy the profile and switch to the [cluster-generic executor](https://snakemake.github.io/snakemake-plugin-catalog/plugins/executor/cluster-generic.html).


----------------



# Workflow DAG

![workflow-DAG](Snakemake-DAG.png)


---------------

# Citation

For fastbe, please cite the original authors:

```
@article{schmidt2024regression,
  title={A regression based approach to phylogenetic reconstruction from multi-sample bulk DNA sequencing of tumors},
  author={Schmidt, Henri and Raphael, Benjamin J},
  journal={PLOS Computational Biology},
  volume={20},
  number={12},
  pages={e1012631},
  year={2024},
  publisher={Public Library of Science San Francisco, CA USA}
}
```
