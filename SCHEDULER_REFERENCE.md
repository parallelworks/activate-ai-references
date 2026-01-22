# HPC Scheduler Reference (SLURM & PBS)

Quick reference for SLURM and PBS scheduler directives commonly used in ACTIVATE workflows.

## SLURM Reference

### Common Flags

| Flag | Description | Example |
|------|-------------|---------|
| `--partition` / `-p` | Target partition/queue | `--partition=gpu` |
| `--time` / `-t` | Wall time limit | `--time=04:00:00` |
| `--nodes` / `-N` | Number of nodes | `--nodes=2` |
| `--ntasks` / `-n` | Number of tasks | `--ntasks=16` |
| `--ntasks-per-node` | Tasks per node | `--ntasks-per-node=8` |
| `--cpus-per-task` / `-c` | CPUs per task | `--cpus-per-task=4` |
| `--mem` | Memory per node | `--mem=64G` |
| `--mem-per-cpu` | Memory per CPU | `--mem-per-cpu=4G` |
| `--gres` | Generic resources (GPUs) | `--gres=gpu:4` |
| `--account` / `-A` | Billing account | `--account=myproject` |
| `--job-name` / `-J` | Job name | `--job-name=training` |
| `--output` / `-o` | Stdout file | `--output=job_%j.out` |
| `--error` / `-e` | Stderr file | `--error=job_%j.err` |
| `--exclusive` | Exclusive node access | `--exclusive` |
| `--constraint` / `-C` | Node features | `--constraint=a100` |

### GPU Examples

```bash
# Single GPU
--gres=gpu:1

# Multiple GPUs
--gres=gpu:4

# Specific GPU type
--gres=gpu:a100:2
--constraint=a100

# Full node with all GPUs
--exclusive --gres=gpu:8
```

### Time Formats

```bash
--time=30              # 30 minutes
--time=04:00:00        # 4 hours
--time=1-00:00:00      # 1 day
--time=7-12:00:00      # 7 days 12 hours
```

### Memory Formats

```bash
--mem=64G              # 64 GB per node
--mem=64000M           # 64000 MB per node
--mem-per-cpu=4G       # 4 GB per CPU
```

### SLURM in ACTIVATE Workflows

```yaml
jobs:
  provision:
    steps:
      - name: Get Compute
        id: agent
        uses: scheduler-agent
        with:
          scheduler-type: slurm
          scheduler-flags: |
            --partition=gpu
            --gres=gpu:2
            --time=04:00:00
            --mem=64G

  run:
    needs: [provision]
    ssh:
      remoteHost: "${{ needs.provision.steps.agent.outputs.ip }}"
    steps:
      - name: Execute
        run: ./run_training.sh
```

---

## PBS Reference

### Common Directives

| Directive | Description | Example |
|-----------|-------------|---------|
| `-q` | Queue name | `-q gpu` |
| `-l walltime` | Wall time limit | `-l walltime=04:00:00` |
| `-l select` | Resource selection | `-l select=1:ncpus=8:ngpus=2:mem=64gb` |
| `-l place` | Placement policy | `-l place=scatter` |
| `-A` | Account/project | `-A myproject` |
| `-N` | Job name | `-N training_job` |
| `-o` | Stdout file | `-o job.out` |
| `-e` | Stderr file | `-e job.err` |
| `-j oe` | Join stdout/stderr | `-j oe` |
| `-V` | Export environment | `-V` |

### Resource Selection Syntax

```bash
# Basic format
-l select=<chunks>:ncpus=<n>:mem=<size>

# 1 node, 8 CPUs, 64GB memory
-l select=1:ncpus=8:mem=64gb

# 2 nodes, 16 CPUs each, 2 GPUs each
-l select=2:ncpus=16:ngpus=2:mem=128gb

# With specific GPU type (site-dependent)
-l select=1:ncpus=8:ngpus=4:mem=64gb:gpu_model=a100
```

### Time Formats

```bash
-l walltime=04:00:00      # 4 hours
-l walltime=24:00:00      # 1 day
-l walltime=168:00:00     # 7 days
```

### PBS in ACTIVATE Workflows

```yaml
jobs:
  provision:
    steps:
      - name: Get Compute
        id: agent
        uses: scheduler-agent
        with:
          scheduler-type: pbs
          scheduler-flags: |
            -q gpu
            -l select=1:ncpus=8:ngpus=2:mem=64gb
            -l walltime=04:00:00

  run:
    needs: [provision]
    ssh:
      remoteHost: "${{ needs.provision.steps.agent.outputs.ip }}"
    steps:
      - name: Execute
        run: ./run_training.sh
```

---

## SLURM vs PBS Comparison

| Feature | SLURM | PBS |
|---------|-------|-----|
| Partition/Queue | `--partition=gpu` | `-q gpu` |
| Time limit | `--time=04:00:00` | `-l walltime=04:00:00` |
| Nodes | `--nodes=2` | `-l select=2:...` |
| CPUs | `--ntasks=16` or `--cpus-per-task=4` | `-l select=1:ncpus=16` |
| Memory | `--mem=64G` | `-l select=1:mem=64gb` |
| GPUs | `--gres=gpu:2` | `-l select=1:ngpus=2` |
| Account | `--account=proj` | `-A proj` |
| Job name | `--job-name=test` | `-N test` |

---

## Workflow Input Patterns for Schedulers

### Dynamic Scheduler Selection

```yaml
on:
  execute:
    inputs:
      scheduler_type:
        type: dropdown
        label: "Scheduler"
        default: "slurm"
        options:
          - label: "SLURM"
            value: "slurm"
          - label: "PBS"
            value: "pbs"
          - label: "None (SSH only)"
            value: "none"

      slurm_options:
        type: group
        label: "SLURM Options"
        hidden: ${{ inputs.scheduler_type != 'slurm' }}
        items:
          - name: partition
            type: slurm-partitions
            label: "Partition"
          - name: time
            type: string
            label: "Time Limit"
            default: "04:00:00"
          - name: gpus
            type: number
            label: "GPUs"
            default: 1
            min: 0
            max: 8

      pbs_options:
        type: group
        label: "PBS Options"
        hidden: ${{ inputs.scheduler_type != 'pbs' }}
        items:
          - name: queue
            type: string
            label: "Queue"
          - name: walltime
            type: string
            label: "Walltime"
            default: "04:00:00"
          - name: select
            type: string
            label: "Resource Selection"
            default: "1:ncpus=8:mem=64gb"
```

### Common Partition/Queue Patterns

```yaml
partition:
  type: dropdown
  label: "Partition"
  options:
    - label: "CPU Compute"
      value: "compute"
    - label: "GPU (A100)"
      value: "gpu-a100"
    - label: "GPU (V100)"
      value: "gpu-v100"
    - label: "High Memory"
      value: "highmem"
    - label: "Debug (short jobs)"
      value: "debug"
```

---

## Useful Environment Variables

### SLURM

| Variable | Description |
|----------|-------------|
| `$SLURM_JOB_ID` | Job ID |
| `$SLURM_JOB_NAME` | Job name |
| `$SLURM_NODELIST` | Allocated nodes |
| `$SLURM_NNODES` | Number of nodes |
| `$SLURM_NTASKS` | Number of tasks |
| `$SLURM_CPUS_PER_TASK` | CPUs per task |
| `$SLURM_GPUS` | Number of GPUs |
| `$SLURM_SUBMIT_DIR` | Submission directory |

### PBS

| Variable | Description |
|----------|-------------|
| `$PBS_JOBID` | Job ID |
| `$PBS_JOBNAME` | Job name |
| `$PBS_NODEFILE` | File with node list |
| `$PBS_O_WORKDIR` | Submission directory |
| `$NCPUS` | Number of CPUs |
| `$NGPUS` | Number of GPUs (if set) |
