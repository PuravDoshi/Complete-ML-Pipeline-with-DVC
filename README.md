## Building and Tracking an ML Pipeline with DVC

This project creates a spam detector and follows the full workflow: create the project, automate the
pipeline, use parameters, track experiments, and store DVC-managed files in AWS S3.

---

## 1. Build the project

### A. Create a GitHub repository and clone it locally

Create an empty repository on GitHub, then clone it to your computer. Keep your code and pipeline definition in this repository. Add an experiments directory consisting of the original dataset and the Jupyter notebook.

``` bash
git clone <your-repository-url>
cd <your-repository-name>
```

**What this does:** Git tracks your code and configuration. DVC will help track larger data and model artifacts.

### B. Add a `src/` folder and run each component individually

Put your pipeline scripts in `src/` (for example: data_ingestion, data_preprocessing, feature_engineering, model_building, and model_evaluation). Run each script on its own first to confirm it works before connecting the scripts into an automated pipeline.

### C. Add generated files to `.gitignore`

Add directories such as data, models, and reports if you do not want Git to track their contents directly. DVC can manage appropriate data/model outputs instead.

**Important:** Do not ignore files that you need Git to track, such as source code, `dvc.yaml`, `params.yaml`, or other configuration files. If Git is already tracking a file, adding it to `.gitignore` alone does not untrack it.

### D. Commit and push the initial project

``` bash
git add .
git commit -m "Add initial ML project"
git push
```

**What this does:** Saves your initial code and project structure in GitHub.

---

## 2. Set up a DVC pipeline without parameters

### A. Create `dvc.yaml` and define pipeline stages

Create a `dvc.yaml` file that describes the stages of your pipeline, their commands, dependencies, and outputs. A stage might prepare data, another might train a model, and another might evaluate it.

A stage definition generally tells DVC:
**`cmd`** --- which command to run.
**`deps`** --- which files the stage depends on.
**`outs`** --- which files the stage produces.
**`metrics`** --- optional files containing metrics to track.

**What this does:** Describes the workflow so DVC can determine which stages need to run and in what order. Use your actual script and file paths.

### B. Initialize DVC, run the pipeline, and inspect the DAG

Initialize DVC in the repository (if you have not already done so), then run the pipeline:

``` bash
dvc init
dvc repro
dvc dag
```

**What each command does:**
`dvc init` sets up DVC tracking in the repository. Run it only once per repository.
`dvc repro` runs the stages defined in `dvc.yaml`, in dependency order. On later runs, DVC
can skip stages whose dependencies have not changed.
`dvc dag` displays the pipeline as a directed acyclic graph, helping you understand the order and dependencies of stages.

### C. Commit and push the pipeline definition

``` bash
git add dvc.yaml .dvc .gitignore
git commit -m "Add DVC pipeline"
git push
```

Include any other relevant source/configuration files that changed. DVC metadata and pipeline definitions belong in Git; large DVC-managed data/model contents are stored separately through DVC remotes.

---

## 3. Set up the DVC pipeline with parameters

### A. Add `params.yaml`

Create a `params.yaml` file to keep changeable settings separate from your Python code. For example:

``` yaml
n_estimators: 100
random_state: 42
```

These are example parameters. Use the parameters that your own model actually needs.

**What this does:** Makes it easy to adjust settings for different runs without editing the training script each time.

### B. Connect the parameters to `dvc.yaml` and your code

In the relevant stage of `dvc.yaml`, list the parameters that the stage uses. For example:

``` yaml
stages:
    model_building:
        cmd: python src/model_building.py
        deps:
        - data/processed
        - src/model_building.py
        params:
        - model_building.n_estimators
        - model_building.random_state
        outs:
        - models/model.pkl
```

This is only an example stage; adapt its command, dependencies, and outputs to your pipeline. The parameter names listed under `params` must match the keys in `params.yaml`.

### C. Run the pipeline again

``` bash
dvc repro
```

**What this does:** DVC checks whether the parameters or other dependencies have changed. If a change affects a stage, DVC reruns that stage and any downstream stages that need updating.

### D. Commit and push the parameterized pipeline

``` bash
git add dvc.yaml params.yaml src/
git commit -m "Add configurable pipeline parameters"
git push
```

Include any other changed files needed for the pipeline to run.

---

## 4. Run and compare experiments with DVC Live

### A. Install DVCLive

Activate your project environment, then install DVCLive:

``` bash
pip install dvclive
```

**What this does:** DVCLive provides functions for logging metrics, parameters, and other run information so that experiment results can be inspected and compared.

### B. Add the DVCLive logging code

In your evaluation or training script, log the metrics that were actually calculated from the model's predictions. For example:

``` python
from dvclive import Live

with Live(save_dvc_exp=True) as live:
    live.log_metric("accuracy", metrics["accuracy"])
    live.log_metric("precision", metrics["precision"])
    live.log_metric("recall", metrics["recall"])
    live.log_metric("auc", metrics["auc"])
    live.log_params(params)
```

Here, `metrics` and `params` are dictionaries that your code has already created. Change the metric names to match the metrics your project calculates.

### C. Run an experiment

``` bash
dvc exp run
```

To set a parameter for a named experiment, you can run:

``` bash
dvc exp run --set-param n_estimators=50 --name exp-50
```

**What this does:** Runs the pipeline and records the experiment's parameter values and logged metrics. DVC uses the existing `dvc.yaml`; `dvc exp run` does not normally create a new `dvc.yaml`. DVCLive
commonly writes run outputs under a `dvclive/` directory, depending on its configuration.

### D. View and compare experiments

``` bash
dvc exp show
```

You can also use the DVC extension in VS Code to view experiments if it is installed and configured.

**What this does:** Shows recorded experiment parameters and metrics so you can compare runs. If different parameter values show identical metrics, check that: 
1. The parameters are listed under the correct stage in `dvc.yaml`. 
2. The Python code loads and actually uses those parameters. 
3. The experiment reruns the training and evaluation stages.
4. Evaluation loads the newly trained model and calculates metrics using predictions, not the true labels twice.

### E. Remove or apply an experiment (optional)

Remove an experiment you no longer need:

``` bash
dvc exp remove <experiment-name>
```

Apply an experiment's changes to your current workspace:

``` bash
dvc exp apply <experiment-name>
```

**What this does:** - `dvc exp remove` removes the selected experiment. - `dvc exp apply` applies that experiment's changes to the workspace. Review the resulting Git status before committing.

### F. Commit and push your Git changes

``` bash
git status
git add dvc.yaml params.yaml src/ reports/ dvclive/
git commit -m "Add DVC experiment tracking"
git push
```

---

## 5. Configure AWS S3 as a DVC remote

### A. Log in to the AWS Console

Open the AWS Management Console and sign in to the AWS account where you want to store the DVC artifacts.

**What this does:** Gives you access to create and configure the AWS resources needed for storage.

### B. Create/configure an IAM user or role with S3 permissions

Set up an IAM identity with only the permissions required for your S3 bucket. Follow your organization's security practices; an IAM role or another temporary-credential approach is preferable when available.

**What this does:** IAM controls who can access the bucket and what they are allowed to do. Do not commit AWS access keys or secret keys to GitHub or put them in `params.yaml`.

### C. Create an S3 bucket

Create an S3 bucket in the AWS Console. Bucket names must be globally unique. Note the bucket name and region.

**What this does:** The bucket stores data and model artifacts that DVC uploads. The bucket is separate from your GitHub repository.

### D. Install the DVC S3 support and AWS CLI

In your active Python environment:

``` bash
pip install "dvc[s3]"
pip install awscli
```

**What this does:** `dvc[s3]` adds S3 remote support to DVC. The AWS CLI lets you configure and test AWS credentials from your terminal.

### E. Configure AWS credentials

Run:

``` bash
aws configure
```

Enter the access key, secret key, default region, and output format when prompted, if you are using access keys. Use the credential method approved for your environment.

**What this does:** Configures credentials the AWS CLI and DVC can use. Keep credentials out of your repository. If your environment already supplies credentials through an IAM role or another provider, you may not need to enter access keys.

### F. Add the S3 bucket as a DVC remote

Run this command, replacing `bucketname` with your actual bucket name:

``` bash
dvc remote add -d storage s3://bucketname
```

**What this does:** Registers an S3 location named `storage` as DVC's default remote (`-d`). DVC can use it to store and retrieve DVC-managed artifacts.

Check the configured remotes with:

``` bash
dvc remote list
```

### G. Push DVC-managed artifacts to S3

After running the pipeline and ensuring the outputs are tracked by DVC, run:

``` bash
dvc push
```

**What this does:** Uploads the DVC-managed cache objects required by the project to the configured remote. It is the normal command for sending DVC data/model artifacts to S3.

If you want to share a DVC experiment itself with a Git remote, use the experiment-push workflow supported by your DVC version, for example:

``` bash
dvc exp push origin <experiment-name>
```

`dvc push` and `dvc exp push` serve different purposes: the first pushes DVC-tracked data/artifacts to the DVC remote; the second shares an experiment through a Git remote.

### H. Commit and push Git-tracked project files

``` bash
git add .dvc/config dvc.yaml params.yaml .gitignore src/
git commit -m "Configure S3 DVC remote"
git push
```

Include other changed files that should be version-controlled. Do not commit credentials. The `.dvc/config` file normally records the remote URL, not secret access keys.

---