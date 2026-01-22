# ACTIVATE Platform - AI Code Assistant Reference

Reference documentation for AI code assistants (Claude, Codex, Copilot, etc.) working with the Parallel Works ACTIVATE platform.

**Point your AI assistant to this repository** for context when developing ACTIVATE workflows, using the SDK, or working with HPC schedulers.

---

## What is ACTIVATE?

ACTIVATE is Parallel Works' HPC (High-Performance Computing) and cloud computing platform. It enables users to:

- Run computational workflows on remote clusters (SLURM, PBS, Kubernetes)
- Deploy interactive sessions (Jupyter, VS Code, desktops, custom services)
- Manage cloud and on-prem compute resources
- Transfer and process data across storage systems

---

## Quick Reference Guides

| Guide | Description |
|-------|-------------|
| [WORKFLOW_BUILDER.md](WORKFLOW_BUILDER.md) | Complete workflow YAML reference, input types, actions, expressions, SDK, CLI |
| [SCHEDULER_REFERENCE.md](SCHEDULER_REFERENCE.md) | SLURM and PBS flags, resource requests, GPU configuration |
| [ENVIRONMENT_VARIABLES.md](ENVIRONMENT_VARIABLES.md) | Platform variables, accessing inputs, writing outputs |

---

## Critical Syntax Rules

**Read this first to avoid common errors that produce unhelpful "Unknown Error" messages.**

### The `uses:` Field (Most Common Error)

```yaml
# Marketplace workflows - use marketplace/ prefix
- uses: marketplace/script_submitter    # CORRECT
- uses: workflow/script_submitter       # WRONG - causes "Unknown Error"

# Your own workflows - use workflow/ prefix
- uses: workflow/my-custom-workflow     # CORRECT

# Built-in actions - no prefix
- uses: checkout                        # CORRECT
- uses: update-session                  # CORRECT
- uses: scheduler-agent                 # CORRECT
- uses: wait-for-agent                  # CORRECT
```

### Expressions Need Spaces

```yaml
hidden: ${{ inputs.mode != 'advanced' }}     # CORRECT
hidden: ${{ inputs.mode!='advanced' }}       # WRONG
```

### Grouped Input References

```yaml
# If input is in a group, include group name
${{ inputs.scheduler_config.partition }}     # CORRECT
${{ inputs.partition }}                      # WRONG (if partition is in a group)
```

---

## Before Generating Workflow Code

**Always clarify these before writing a workflow:**

1. **Target infrastructure**: What cluster/resource will this run on?
2. **Scheduler type**: SLURM, PBS, or direct SSH?
3. **Container requirements**: Does this need Singularity/Apptainer?
4. **Interactive or batch?**: Does the user need a session (Jupyter, desktop) or a batch job?
5. **Existing patterns**: Is there a similar workflow in the reference repos to build from?

---

## Workflow Patterns by Use Case

### Pattern 1: Simple Script Execution

```yaml
permissions: ["*"]

on:
  execute:
    inputs:
      cluster:
        type: compute-clusters
        label: "Cluster"

jobs:
  run:
    ssh:
      remoteHost: "${{ inputs.cluster }}"
    steps:
      - name: Run Script
        run: ./my_script.sh
```

### Pattern 2: SLURM Job Submission

```yaml
jobs:
  provision:
    steps:
      - name: Get Compute Node
        id: agent
        uses: scheduler-agent
        with:
          scheduler-type: slurm
          scheduler-flags: |
            --partition=gpu
            --gres=gpu:2
            --time=04:00:00

  run:
    needs: [provision]
    ssh:
      remoteHost: "${{ needs.provision.steps.agent.outputs.ip }}"
    steps:
      - name: Train Model
        run: python train.py
```

### Pattern 3: Interactive Session

```yaml
sessions:
  my-service:
    type: tunnel
    redirect: true

jobs:
  start:
    steps:
      - name: Start Service
        run: ./start_service.sh

  update:
    needs: [start]
    steps:
      - uses: update-session
        with:
          name: my-service
          remotePort: 8080
```

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `PW_USER` | Current username |
| `PW_JOB_ID` | Workflow run ID |
| `PW_WORKFLOW_NAME` | Name of the workflow |
| `PW_PLATFORM_HOST` | Platform URL |
| `JOB_DIR` | Working directory for the job |
| `OUTPUTS` | File to append outputs (`echo "KEY=value" >> $OUTPUTS`) |

See [ENVIRONMENT_VARIABLES.md](ENVIRONMENT_VARIABLES.md) for complete reference.

---

## Reference Workflow Repositories

When generating workflows, reference these proven patterns:

| Repository | Use Case |
|------------|----------|
| [activate-medical-finetuning](https://github.com/parallelworks/activate-medical-finetuning) | GPU training, container management, model handling |
| [activate-sessions](https://github.com/parallelworks/activate-sessions) | Interactive sessions, two-script architecture |
| [activate-rag-vllm](https://github.com/parallelworks/activate-rag-vllm) | LLM deployment, vLLM, RAG |
| [interactive_session](https://github.com/parallelworks/interactive_session) | VNC, desktop environments |

---

## Testing Recommendations

Suggest users test workflows incrementally:

1. Start with a minimal workflow that just echoes inputs
2. Add one job/feature at a time
3. Use the Build tab's Ctrl+Space for field suggestions
4. Check the Runs tab for detailed logs when debugging

---

## Official Resources

- [Official Documentation](https://parallelworks.com/docs)
- [API Reference](https://parallelworks.com/api)
- [CLI Documentation](https://parallelworks.com/docs/cli)
- [SDK Repository](https://github.com/parallelworks/sdk) - Python, TypeScript, Go
