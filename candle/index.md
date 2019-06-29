---
bigimg: "/img/FNL_ATRF_Pano_4x10.jpg"
title: How to run CANDLE on Biowulf
---

# Introduction

CANDLE is open source software for hyperparameter optimization that scales very efficiently on the world's fastest supercomputers.  The [SDSI team](https://cbiit.github.com/sdsi/team) at the [Frederick National Laboratory for Cancer Research](https://frederick.cancer.gov), sponsored by NCI, has recently installed CANDLE on NIH's [Biowulf](https://hpc.nih.gov) supercomputer.

In a machine/deep learning model, "hyperparameters" refer to any variables that define the model aside from the model's "weights."  Typically, for a given set of hyperparameters (typically 5-20), the corresponding model's weights (typically tens of thousands) are iteratively optimized using algorithms such as gradient descent.  Such optimization of the model's weights starting from random values -- a process called "training" -- is typically run very efficiently on graphics processing units (GPUs) and typically takes 30 min. to a couple days.

If a measure of accuracy is assigned to each model trained on the same set of data, we would like to ultimately choose the model (i.e., set of hyperparameters) that best fits that dataset by maximizing the accuracy (or minimizing a loss).  The process of choosing the best set of hyperparameters is referred to as "[hyperparameter optimization](https://en.wikipedia.org/wiki/Hyperparameter_optimization)" (HPO).  The most common, brute-force way of determining the optimal set of hyperparameters is to run one training job for every desired combination of hyperparameters and choose that which produces the highest accuracy.  Such a workflow is labeled in CANDLE by "UPF," which stands for Unrolled Parameter File, which is simply a text file specifying all the combinations of hyperparameters to be run.

A more elegant way of determining the optimal set of hyperparameters is to use a Bayesian approach in which information about how well prior sets of hyperparameters performed is used to select the next sets of hyperparameters to try.  This type of workflow is labeled in CANDLE by "[mlrMBO](https://mlrmbo.mlr-org.com)."

HPO need not be used for only machine/deep learning applications; it can be applied to any computational pipeline that can be parametrized by a number of settings.  With ever-increasing amounts of data, applications like these, in addition to machine/deep learning applications, are growing at NCI and in the greater NIH community.  If HPO is performed, better models for describing relationships between data can be found, and the better the model, the more accurate predictions can be determined given new sets of data.  CANDLE is here to help with this!

# Quick start

These steps will get you running a sample CANDLE job on Biowulf right away!

## Step 1: Set up your environment

Once [logged in to Biowulf](https://hpc.nih.gov/docs/connect.html), set up your environment by creating and entering a working directory on your data partition and loading the CANDLE module:

```bash
mkdir /data/$USER/candle
cd /data/$USER/candle
module load candle
```

## Step 2: Copy the template submission script to the working directory

Copy the template CANDLE submission script `submit_candle_job.sh` to the working directory:

```bash
copy_candle_template-new
```

This will also create an empty `experiments` directory in the working directory.

## Step 3: Run the job

Submit the job by running

```bash
./submit_candle_job.sh
```

(No, there really is no need for `sbatch`.)

# Modifying the template for your use case

You only need to modify the six settings inside the `submit_candle_job.sh` script.  All variables should be preceded by an `export` command, as they are in the template submission script.  Please examine the sample settings below to better understand their meaning.

## Required variables

* **`MODEL_SCRIPT`**: This should point to the Python or R script that you would like to run
  * E.g., `$CANDLE_WRAPPERS/templates/models/wrapper_compliant/mnist_mlp.py`
  * This script must have been adapted to work with CANDLE (see the following section)
  * The filename extension will automatically determine whether Python or R will be used to run the model
* **`DEFAULT_PARAMS_FILE`**: Default settings for the hyperparameters defined in the model (following section)
  * E.g., `$CANDLE_WRAPPERS/templates/model_params/mnist1.txt`
  * These values will be overwritten by those defined in the `WORKFLOW_SETTINGS_FILE`, below
* **`WORKFLOW_SETTINGS_FILE`**: This file contains the settings parametrizing the workflow you would like to run
  * E.g., `$CANDLE_WRAPPERS/templates/workflow_settings/upf_workflow-3.txt`
  * These settings will assign values to the hyperparameters that will override their default values defined by `DEFAULT_PARAMS_FILE`, above
  * The hyperparameters specified in this file should be a subset of those in `DEFAULT_PARAMS_FILE`, indicating only the hyperparameters that will be changed during the workflow
  * The filename MUST begin with `<WORKFLOW_TYPE>_workflow-`, where `<WORKFLOW_TYPE>` is `upf`
  * Run `module load python/3.6; python $CANDLE_WRAPPERS/templates/scripts/generate_hyperparameter_grid.py` for help with generating an Unrolled Parameter File if running the UPF workflow
  * Note: Python's `False`, `True`, and `None`, should be replaced by JSON's `false`, `true`, and `null` in `WORKFLOW_SETTINGS_FILE`
* **`NGPUS`**: Number of GPUs you would like to use for the CANDLE job
  * E.g., `2`
  * Note: One or two extra nodes will be allocated in order run background processes
* **`GPU_TYPE`**: Type of GPU you would like to use
  * E.g., `k80`
  * The choices on Biowulf are `k20x`, `k80`, `p100`, `v100`
* **`WALLTIME`**: How long you would like your job to run
  * E.g., `00:20:00`
  * Format is `HH:MM:SS`

## Optional variables

### Python models only

* **`PYTHON_BIN_PATH`**: If you don't want to use the Python version with which CANDLE was built (currently `python/3.6`), you can set this to the location of the Python binary you would like to use
  * E.g., `$CONDA_PREFIX/envs/<YOUR_CONDA_ENVIRONMENT_NAME>/bin`
  * E.g., `/data/BIDS-HPC/public/software/conda/envs/main3.6/bin`
  * If set, it will override the setting of `EXEC_PYTHON_MODULE`, below
* **`EXEC_PYTHON_MODULE`**: If you'd prefer loading a module rather than specifying the path to the Python binary (above), set this to the name of the Python module you would like to load
  * E.g., `python/2.7`
  * This setting will have no effect if `PYTHON_BIN_PATH` (above) is set
  * If neither `PYTHON_BIN_PATH` nor `EXEC_PYTHON_MODULE` is set, then the version of Python with which CANDLE was built (currently `python/3.6`) will be used
* **`SUPP_PYTHONPATH`**: This is a supplementary setting of the PYTHONPATH variable that will be searched for libraries that can't otherwise be found
  * E.g., `/data/$USER/conda/envs/my_conda_env/lib/python3.6/site-packages`
  * E.g., `/data/BIDS-HPC/public/software/conda/envs/main3.6/lib/python3.6/site-packages`

### R models only

* **`EXEC_R_MODULE`**: If you don't want to use the R version with which CANDLE was built (currently `R/3.5.0`), set this to the name of the R module you would like to load
  * E.g., `R/3.6`

### Models written in either language

* **`SUPP_MODULES`**: Modules you would like to have loaded while your model is run
  * E.g., `"CUDA/10.0 cuDNN/7.5/CUDA-10.0"` (these particular example settings are necessary for running TensorFlow when using a custom Conda installation)
* **`EXTRA_SCRIPT_ARGS`**: Command-line arguments you'd like to include when invoking Python or Rscript
  * E.g., `--max-ppsize=100000` if the model is written in R
  * In other words, the model will ultimately be run like `python $EXTRA_SCRIPT_ARGS my_model.py` or `Rscript $EXTRA_SCRIPT_ARGS my_model.R`
* **`RESTART_FROM_EXP`**: If a UPF job was run previously but for whatever reason did not complete, here you can specify the name of the experiment from which to resume
  * E.g., `X002`

# Adapting your model to work with CANDLE

Prior to adapting your model script, it should run standalone on a Biowulf compute node.  (This can be tested by requesting an interactive GPU node [e.g., `sinteractive --constraint=gpuk20x --mem=20G --gres=gpu:k20x:1`] and then running the model like, e.g., `python my_model.py` or `Rscript my_model.R`; don't forget to use the correct version of Python or R, if required!)  Otherwise, you script needs to be modified in two simple ways in order to work with CANDLE.

## Specify the hyperparameters

Specify the hyperparameters in your code using the dictionary (Python) or data.frame (R) datatypes.  E.g., in Python, if your model script `my_model.py` contains

```python
n_convolutional_layers = 4
batch_size = 128
```

but you'd like to vary these settings during the CANDLE workflow, you should change those lines to

```python
n_convolutional_layers = hyperparams['nconv_layers']
batch_size = hyperparams['batch_size']
```

Note that the "key" in the `hyperparams` dictionary should match the variable names in the `DEFAULT_PARAMS_FILE` and `WORKFLOW_SETTINGS_FILE` files, whereas the variables to which they are assigned in the model script should obviously match the names used in the rest of the script.

Likewise, in R, if your model script `my_model.R` contains

```R
n_convolutional_layers <- 4
batch_size <- 128
```

you should change those lines to

```R
n_convolutional_layers <- hyperparams[["nconv_layers"]]
batch_size <- hyperparams[["batch_size"]]
```

## Define the metric you would like to minimize

If your model is written in Python, either define a history object named `history` (as in, e.g., the return value of a model.fit() method), e.g.,

```python
history = model.fit(x_train, y_train, ...)
```

or define a single number named `val_to_return` that contains the metric you would like to minimize, e.g.,

```python
score = model.evaluate(x_test, y_test)
val_to_return = score[0]
```

If your model is written in R, define a single number named `val_to_return` that contains the metric you would like to minimize, e.g.,

```R
val_to_return <- my_validation_loss
```

---

Feel free to email [Andrew Weisman](mailto:andrew.weisman@nih.gov) with any questions.