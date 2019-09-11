---
layout: episode
title: "Recording computational steps for OSeMOSYS"
teaching: 15
exercises: 25
questions:
  - How can we create a reproducible workflow?
  - When to use scientific workflow management systems.
objectives:
  - Discuss pros/cons of GUI vs. manual steps vs. scripted vs. workflow tools.
  - Get familiar with Snakemake.
keypoints:
  - Preserve the steps for re-generating published results.
  - Hundreds of workflow management tools exist.
  - Make and Snakemake are a comparatively simple and lightweight options to create transferable and scalable data analyses.
  - Sometimes a script is enough.
---

## One problem solved in 5 different ways

> The following material is adapted from a [HPC Carpentry lesson](https://hpc-carpentry.github.io/hpc-python/)

Let's look at an example project which follows the project structure guidelines given in the previous episode.
The project runs OSeMOSYS and plots a couple of charts.

To follow along, clone this [repository](https://github.com/KTH-dESA/osemosys_workflow):
```shell
$ git clone https://github.com/KTH-dESA/osemosys_workflow.git
```

The example project directory listing is:
```
.
├── README.md
├── data
│   ├── README.md
│   └── simplicity.txt
├── env
│   ├── dag.yaml
│   ├── osemosys.yaml
│   └── plot.yaml
├── model
│   ├── LICENSE
│   └── osemosys.txt
├── processed_data
├── scripts
│   ├── osemosys_run.sh
│   └── plot_results.py
└── snakefile
```

In this example we wish to:
1. Run a model run
2. Extract some csv files from the `SelectedResults.csv` file
3. Plot those results

Ideally, we would go on to:
4. Create environments for each of the key tasks
5. Run model runs in parallel and scale them up for running on a cluster


## The Problem

Example (for one model run only) - let us test this out:

```
$ glpsol -d data/simplicity.txt -m model/osemosys.txt -o process_data/results.sol
$ ???
$ python scripts/plot_results.py processed_data/total_annual_capacity.csv results/total_annual_capacity.pdf
$ ???
$ python scripts/plot_results.py processed_data/tid_demand.csv results/tid_demand.pdf
```

Can you relate? Are you using similar setups in your research?

This was for one model run - how about 3 model runs? How about 3000 model runs?

> ## Discussion
>
> Discuss the pros and cons of this approach. Is it reproducible? Does it scale to hundreds of model runs? Can it be automated?
> What if you modify only one data file and do not wish to rerun the pipeline for all datafiles again?
{: .task}

> ## Exercise
>
> What commands are required to replace the `???` in the second line?
> Think back to the first workshop on the shell...
{: .task}

---

## Solution 1: Script

Let's express it more compactly with a shell script (Bash). Let's call it `run_analysis.sh`:
```bash
#!/usr/bin/env bash
glpsol -d data/simplicity.txt -m model/osemosys.txt -o process_data/results.sol
head -n 326 processed_data/SelectedResults.csv | tail -n 29 > processed_data/total_annual_capacity.csv
python scripts/plot_results.py processed_data/total_annual_capacity.csv results/total_annual_capacity.pdf
head -n 33 processed_data/SelectedResults.csv | tail -n 22 > processed_data/tid_demand.csv
python scripts/plot_results.py processed_data/tid_demand.csv results/tid_demand.pdf
```

We can run it with:
```
$ run_analysis.sh
```

This is still **imperative style**: we tell the script to run these steps in precisely this order.

> ## Exercise/discussion
>
> Discuss the pros and cons of this approach. Is it reproducible? Does it scale to hundreds of model runs? Can it be automated?
> What if you modify only one data file and do not wish to rerun the pipeline for all datafiles again?
{: .task}

---

## Solution 2: Using [GNU Make](https://www.gnu.org/software/make/)

- A tool from the 70s often used to build software.
- Uses specific syntax that the user writes in a Makefile.
- Makefile specifies how to build targets from their dependencies.
- Observe that we use wildcards instead of explicit model run names.

It contains rules that relate targets to dependencies and commands:

```makefile
# rule (mind the tab)
target: dependencies
	command(s)
```

We can think of it as follows:
```makefile
outputs: inputs
	command(s)
```

Make uses **declarative style**: we describe dependencies but we let Make
figure out the series of steps to produce results (targets). Fun fact: Excel is also
declarative, not imperative.

---

## Solution 3: Using [Snakemake](https://snakemake.readthedocs.io/en/stable/index.html)

First study the `Snakefile`:

```python
# a list of all the model runs we are analyzing
DATA = glob_wildcards('data/{model run}.txt').model run

# this is for running on HPC resources
localrules: all, clean, make_archive

# the default rule
rule all:
    input:
        'zipf_analysis.tar.gz'

# delete everything so we can re-run things
rule clean:
    shell:
        '''
        rm -rf source/__pycache__
        rm -f zipf_analysis.tar.gz processed_data/* results/*
        '''

# count words in one of our model runs
# logfiles from each run are put in .log files"
rule count_words:
    input:
        wc='source/wordcount.py',
        model run='data/{file}.txt'
    output: 'processed_data/{file}.dat'
    log: 'processed_data/{file}.log'
    shell:
        '''
        echo "Running {input.wc} on {input.model run}." &> {log} &&
            python {input.wc} {input.model run} {output} >> {log} 2>&1
        '''

# create a plot for each model run
rule make_plot:
    input:
        plotcount='source/plotcount.py',
        model run='processed_data/{file}.dat'
    output: 'results/{file}.png'
    shell: 'python {input.plotcount} {input.model run} {output}'

# generate summary table
rule zipf_test:
    input:
        zipf='source/zipf_test.py',
        model runs=expand('processed_data/{model run}.dat', model run=DATA)
    output: 'results/results.txt'
    shell:  'python {input.zipf} {input.model runs} > {output}'

# create an archive with all of our results
rule make_archive:
    input:
        expand('results/{model run}.png', model run=DATA),
        expand('processed_data/{model run}.dat', model run=DATA),
        'results/results.txt'
    output: 'zipf_analysis.tar.gz'
    shell: 'tar -czvf {output} {input}'
```

Also Snakemake uses **declarative style**:

<img src="{{ site.baseurl }}/img/snakemake.png" style="height: 250px;"/>

Try it out:
```
$ snakemake clean
$ snakemake
```

Try running `snakemake` again and observe that and discuss why it refused to rerun all steps:
```
$ snakemake

Building DAG of jobs...
Nothing to be done.
```

Make a modification to a txt or a dat file and run `snakemake` again and discuss
what you see. One way to modify files is to use the `touch` command which will
only update its timestamp:

```
$ touch data/sierra.txt
$ snakemake
```

Finally try to run the pipeline on several cores in parallel (here we will try 4):

```
$ snakemake clean
$ snakemake -j 4
```


### Why Snakemake?

- Gentle learning curve.
- Free, open-source, and installs easily via conda or pip.
- Cross-platform (Windows, MacOS, Linux) and compatible with all HPC schedulers
  - same workflow works without modification and scales appropriately whether on a laptop or cluster.
- [Heavily used in bioinformatics](https://twitter.com/carl_witt/status/1103951128046301185), but is completely general.


### Snakemake vs. Make

- Workflows defined in Python scripts extended by declarative code to define rules:
  - anything that can be done in Python can be done with Snakemake
  - rules work much like in GNU Make (can call any commands/executables)
- Possible to define isolated software environments per rule.
- Also possible to run workflows in Docker or Singularity containers.
- Workflows can be pushed out to run on a cluster or in the cloud without modifications to scale up.


### Visualizing the workflow

We can visualize the directed acyclic graph (DAG) of our current Snakefile
using the `--dag` option, which will output the DAG in `dot` language (a
plain-text format for describing graphs used by [Graphviz software](https://www.graphviz.org/),
which can be installed by `conda install graphviz`)

```bash
$ snakemake --dag | dot -Tpng > dag.png
```
Rules that have yet to be completed are indicated with solid outlines, while already completed rules are indicated with dashed outlines.

<img src="{{ site.baseurl }}/img/snakemake_dag.png" style="height: 300px;"/>

> ## Exercise/discussion
>
> Discuss the pros and cons of this approach. Is it reproducible? Does it scale to hundreds of model runs? Can it be automated?
{: .task}



### More Snakemake goodies

- Archive a workflow into a tarball:
```
$ snakemake --archive my-workflow.tar.gz
```
- Snakemake has an experimental GUI feature which can be invoked by:
```
$ snakemake --gui
```
- Isolated software environments per rule using conda. Invoke by `snakemake --use-conda`. Example:
```python
rule NAME:
    input:
        "table.txt"
    output:
        "plots/myplot.pdf"
    conda:
        "envs/ggplot.yaml"
    script:
        "scripts/plot-stuff.R"
```
- It is possible to address and offload to non-CPU resources:
```
$ snakemake clean
$ snakemake -j 4 --resources gpu=1
```
- Transferring your workflow to a cluster:
```
$ snakemake --archive myworkflow.tar.gz
$ scp myworkflow.tar.gz <some-cluster>
$ ssh <some-cluster>
$ tar zxf myworkflow.tar.gz
$ cd myworkflow
$ snakemake -n --use-conda
```
- Interoperability with Slurm:
```json
{
    "__default__":
    {
        "account": "a_slurm_submission_account",
        "mem": "1G",
        "time": "0:5:0"
    },
    "count_words":
    {
        "time": "0:10:0",
        "mem": "2G"
    }
}
```
  The workflow can now be executed by:
```bash
$ snakemake -j 100 --cluster-config cluster.json --cluster "sbatch -A {cluster.account} --mem={cluster.mem} -t {cluster.time} -c {threads}"
```
  Note that in this case `-j` does not correspond to the number of cores used, instead it represents the maximum
  number of jobs that Snakemake is allowed to have submitted at the same time.
  The `--cluster-config` flag specifies the config file for the particular cluster, and the `--cluster` flag specifies
  the command used to submit jobs on the particular cluster.
- Jobs can be run in containers. Execute with `snakemake --use-singularity`. Example:
```python
rule NAME:
    input:
        "table.txt"
    output:
        "plots/myplot.pdf"
    singularity:
        "docker://joseespinosa/docker-r-ggplot2"
    script:
        "scripts/plot-stuff.R"
```
- There is a lot more: [snakemake.readthedocs.io](https://snakemake.readthedocs.io/en/stable/).

---

## Comparison and summary

- GUIs may or may not be reproducible.
- Some GUIs can be automated, many cannot.
- Typing the same series of commands for 100 similar inputs is tedious and error prone.
- Imperative scripts are reproducible and great for automation.
- Declarative workflows such as Snakemake are great for longer multi-step analyses.
- Declarative workflows are often easy to parallelize without you changing anything.
- With declarative workflows it is no problem to add/change things late in the project.
- Many [specialized frameworks](https://github.com/common-workflow-language/common-workflow-language/wiki/Existing-Workflow-systems) exist.
