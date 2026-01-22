# ACTIVATE Environment Variables Reference

Environment variables available during workflow execution on the ACTIVATE platform.

## Platform Variables

These are automatically set by the ACTIVATE platform:

| Variable | Description | Example |
|----------|-------------|---------|
| `PW_USER` | Current username | `jsmith` |
| `PW_JOB_ID` | Unique workflow run ID | `abc123` |
| `PW_WORKFLOW_NAME` | Name of the workflow | `my-training-job` |
| `PW_PLATFORM_HOST` | Platform URL | `https://activate.parallel.works` |
| `JOB_DIR` | Working directory for the job | `~/pw/jobs/workflow/123/` |
| `OUTPUTS` | File path for writing outputs | `/path/to/outputs` |

## Accessing User Inputs

User inputs defined in `on.execute.inputs` are accessible in two ways:

### 1. In YAML Expressions

```yaml
on:
  execute:
    inputs:
      model_name:
        type: string
        label: "Model"

jobs:
  train:
    steps:
      - name: Train
        run: python train.py --model "${{ inputs.model_name }}"
```

### 2. As Environment Variables

Inputs are also available as `PW_INPUT_*` environment variables in scripts:

```bash
#!/bin/bash
# Access input named "model_name"
echo "Training model: ${PW_INPUT_model_name}"
```

### Grouped Inputs

For inputs inside a group:

```yaml
on:
  execute:
    inputs:
      config:
        type: group
        items:
          - name: epochs
            type: number
```

Access as:
- YAML: `${{ inputs.config.epochs }}`
- Script: `${PW_INPUT_config_epochs}` (dots become underscores)

## Writing Outputs

To pass values between jobs, write to the `$OUTPUTS` file:

```yaml
jobs:
  producer:
    steps:
      - name: Generate Values
        id: gen
        run: |
          PORT=$(find_available_port)
          HOSTNAME=$(hostname)
          echo "PORT=$PORT" >> $OUTPUTS
          echo "HOSTNAME=$HOSTNAME" >> $OUTPUTS

  consumer:
    needs: [producer]
    steps:
      - name: Use Values
        run: |
          echo "Connecting to ${{ needs.producer.steps.gen.outputs.HOSTNAME }}:${{ needs.producer.steps.gen.outputs.PORT }}"
```

## Job-Level Environment Variables

Set variables at different scopes:

```yaml
# Workflow-level (lowest priority)
env:
  GLOBAL_VAR: "available everywhere"

jobs:
  example:
    # Job-level (overrides workflow)
    env:
      JOB_VAR: "available in this job"

    steps:
      - name: Step 1
        # Step-level (highest priority)
        env:
          STEP_VAR: "only in this step"
        run: echo "$GLOBAL_VAR $JOB_VAR $STEP_VAR"
```

Priority order (highest to lowest):
1. Step-level `env`
2. Job-level `env`
3. Workflow-level `env`

## Common Patterns

### Pass All Inputs to a Script

```yaml
steps:
  - name: Setup Environment
    run: |
      # Write all inputs to an env file
      cat > inputs.env << 'EOF'
      MODEL_NAME="${{ inputs.model_name }}"
      EPOCHS=${{ inputs.epochs }}
      LEARNING_RATE=${{ inputs.learning_rate }}
      EOF

  - name: Run Training
    run: |
      source inputs.env
      python train.py
```

### Dynamic Environment Based on Input

```yaml
on:
  execute:
    inputs:
      environment:
        type: dropdown
        options:
          - label: "Development"
            value: "dev"
          - label: "Production"
            value: "prod"

jobs:
  deploy:
    env:
      CONFIG_PATH: ${{ inputs.environment == 'prod' ? '/etc/prod.conf' : '/etc/dev.conf' }}
    steps:
      - run: ./deploy.sh --config $CONFIG_PATH
```

### Secrets and Sensitive Values

For sensitive values, use organization variables:

```yaml
steps:
  - name: Authenticate
    env:
      API_KEY: ${{ org.my_api_key }}
    run: ./authenticate.sh
```

Organization variables are configured in the ACTIVATE platform settings and referenced via `${{ org.variable_name }}`.

## Scheduler Environment Variables

When running on HPC schedulers, additional variables are available:

### SLURM

| Variable | Description |
|----------|-------------|
| `SLURM_JOB_ID` | Job ID |
| `SLURM_NODELIST` | Allocated nodes |
| `SLURM_NNODES` | Number of nodes |
| `SLURM_NTASKS` | Number of tasks |
| `SLURM_CPUS_PER_TASK` | CPUs per task |
| `SLURM_GPUS` | GPUs allocated |

### PBS

| Variable | Description |
|----------|-------------|
| `PBS_JOBID` | Job ID |
| `PBS_NODEFILE` | File listing nodes |
| `PBS_O_WORKDIR` | Submission directory |
| `NCPUS` | CPUs allocated |

## Directory Structure

Default working directory structure:

```
~/pw/jobs/{workflow-name}/{job-id}/
├── inputs.env          # User inputs as env vars
├── outputs/            # Job outputs
└── logs/               # Execution logs
```

Access via `$JOB_DIR` or `${{ jobs.<job>.working-directory }}`.

## Tips

1. **Always quote variables** in bash to handle spaces:
   ```bash
   python train.py --name "${MODEL_NAME}"
   ```

2. **Use defaults** for optional inputs:
   ```bash
   EPOCHS="${PW_INPUT_epochs:-10}"
   ```

3. **Validate required variables**:
   ```bash
   : "${MODEL_NAME:?ERROR: MODEL_NAME is required}"
   ```

4. **Debug by printing variables**:
   ```yaml
   steps:
     - name: Debug
       run: |
         echo "JOB_DIR: $JOB_DIR"
         echo "PW_USER: $PW_USER"
         env | grep PW_
   ```
