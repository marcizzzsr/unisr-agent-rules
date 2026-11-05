# UniSR Cluster — Operating Rules

You are running on the login node of the UniSR GPU cluster
(2 nodes: workstation-ai-01, workstation-ai-02, 2 GPUs each).

## Golden rule
Never run compute on the login node. Every workload runs on a compute
node allocated through SLURM, by default inside an Apptainer container
(SLURM + venv is an alternative when needed, see below).
The login node is only for: editing files, submitting/monitoring jobs,
light file operations on /mnt/data and $HOME.
Execution time on the login node is limited: anything that takes more
than a few seconds (downloads, installs, data copies) goes in a SLURM job.

## Running jobs
- Default to `sbatch` with a script. Use `srun --pty` only for short
  interactive checks (e.g. testing that a container starts).
- Partitions (same nodes, same resources, only time limit differs):
  - `interactive`: max 12:00:00
  - `cuda`: max 5-00:00:00
  Pick by expected runtime. A `--time` over the limit is rejected.
- Always set: `--job-name`, `--partition`, `--time`, `--cpus-per-task`,
  `--mem`, `--output=logs/%x_%j.out` (output log name is just an example, could also be structured in subdirs)
- GPUs: `--gres=gpu:<n>` (never `--gpus`). Request GPUs only if the
  workload uses them.
- Request close to real needs: start small, you don't have access to seff.
- One job = one step. Chain preprocessing/training/eval with
  `--dependency=afterok:<jobid>`.
- Container invocation inside the job:
  `apptainer exec --nv --bind <host>:<container> <image.sif> <command>`
  Bind code and data explicitly; never rely on default mounts.
- SLURM + venv: for tasks where a container is overkill, a Python venv
  managed with `uv` (or similar) can be used instead of Apptainer.
  Keep the venv in the project folder (e.g. `$HOME/projects/<repo>/.venv`).
  Creating/syncing the venv downloads packages, so run it inside a SLURM
  job too (`srun ... uv sync`), and run the workload with
  `srun`/`sbatch` + `uv run python ...`. Prefer Apptainer when the task
  needs system libraries, CUDA toolchains or must be reproducible elsewhere.
- Downloads (datasets, model weights, `wget`/`curl`/`huggingface-cli`):
  always in a dedicated SLURM job, never on the login node.

Template:
```bash
#!/usr/bin/env bash
#SBATCH --job-name=<task>
#SBATCH --partition=cuda
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=1-00:00:00
#SBATCH --output=logs/%x_%j.out
set -euo pipefail
apptainer exec --nv \
  --bind "$REPO_DIR:/workspace" \
  --bind /mnt/beegfs/nvme:/mnt/beegfs/nvme \
  "$IMAGE" python -u /workspace/train.py ...
```

## Monitoring and cancelling
- `squeue -u $USER`, `tail -f logs/<name>_<jobid>.out` (or whichever output log name you chose)
- Cancel failed or mistaken jobs immediately with `scancel <jobid>`.
- Only ever cancel jobs you submitted. Never `scancel -u $USER`
  unless the user explicitly asks.

## Apptainer images
- Never build images on the cluster (`apptainer build` is forbidden here).
  If a new image is needed, write the .def/Dockerfile and ask the user
  to build it off-cluster.
- Apptainer is available only inside SLURM context, calling Apptainer on the login node will result in an error.
- Shared images live in:
  `/mnt/data/unisr-data/apptainer_images/images/<name>/image.sif`
  Each folder also contains the build files (Dockerfile, .def,
  requirements.txt...): read them to check what the image contains. Whenever you need to run something and don't already know which image to use start from here.
- Before choosing an image, list that directory and pick one whose
  purpose matches the task. Do not reuse an image built for an
  unrelated domain just because it has Python.
- Code, data and configs are bound at runtime, never baked into images.


## Experiment tracking (MLflow)
Logging experiments to the on-premise MLflow server is the recommended
practice: when writing or launching training code, propose to log
params, metrics and artifacts to MLflow, and tell the user where to view
them.
- Server: `https://mlflow-server.ihsr.ai-hub.it/`
- Auth via environment variables (Apptainer passes host env vars into
  the container by default):
  - `MLFLOW_TRACKING_URI=https://mlflow-server.ihsr.ai-hub.it/`
  - `MLFLOW_TRACKING_USERNAME`: must match exactly the Username shown on
    the user's MLflow token page (mismatch -> `Authentication required`)
  - `MLFLOW_TRACKING_PASSWORD`: the user's personal access token
- Credentials are the user's: ask them to export these variables (e.g. in
  `~/.bashrc` or a private env file). Never hardcode, print, log or commit
  the token.
- The user must have logged in once via browser (KeyCloak -> EntraID HSR)
  and created the token from the Permissions page. Each user has only one
  token and creating a new one invalidates the old: never ask the user to
  regenerate it unless authentication actually fails.
- In code: `mlflow.set_experiment(...)`, `mlflow.start_run()`, then the
  usual `log_param`/`log_metric`/`log_artifact` calls.

## Data and storage
| Path | Use |
|---|---|
| `$HOME/projects/<repo>` | code (git repos) |
| `/mnt/data/unisr-data/datasets/` | raw datasets, landing zone (slow) |
| `/mnt/data/unisr-data/models/` | shared models |
| `/mnt/beegfs/nvme/` (~6TB, shared) | active training data, checkpoints, logs |
| `/mnt/beegfs/hdd/` (going to be deleted soon) | large datasets, intermediate results |

- Never train reading from /mnt/data: stage to
  `/mnt/beegfs/nvme/datasets/<name>` first.
- Never access /mnt/beegfs from the login node (it hangs): do staging
  copies and any BeeGFS operation inside a SLURM job.
- Storage is shared: only delete files you created. Never delete or
  overwrite datasets, images or other users' data.

## Always inform the user
State clearly what you are doing and why when you:
- Write to shared folders (anything under /mnt/data or /mnt/beegfs,
  e.g. apptainer_images, datasets, models).
- Launch long runs (longer than ~12h or using more than 1 GPU).
- Use a pre-built image: say which one and why it fits the task.

## Ask the user and wait for confirmation before
- Moving or copying data between slow storage (/mnt/data) and fast
  storage (/mnt/beegfs/nvme), in either direction. State source,
  destination and approximate size.
- Deleting anything in shared folders (anything under /mnt/data or
  /mnt/beegfs), including files you created yourself. List exactly what
  will be deleted. This applies in every permission mode, including
  "auto"/autonomous modes: never delete in shared folders without an
  explicit confirmation from the user for that specific deletion.
