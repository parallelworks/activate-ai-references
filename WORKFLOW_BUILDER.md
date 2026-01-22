# Parallel Works ACTIVATE Workflow Builder Guide

A comprehensive reference for building workflows on the ACTIVATE platform. This guide consolidates official documentation and community best practices.

## Table of Contents

- [Common Pitfalls](#common-pitfalls)
- [Overview](#overview)
- [Workflow Structure](#workflow-structure)
- [YAML Fields Reference](#yaml-fields-reference)
- [Input Types](#input-types)
- [Expressions](#expressions)
- [Actions](#actions)
- [Job Dependencies and Outputs](#job-dependencies-and-outputs)
- [Interactive Sessions](#interactive-sessions)
- [Best Practices](#best-practices)
- [Example Patterns](#example-patterns)
- [SDK and API](#sdk-and-api)
- [CLI Reference](#cli-reference)

---

## Common Pitfalls

**Read this section first to avoid common errors that produce unhelpful "Unknown Error" messages.**

### 1. Wrong Prefix for `uses` Field

The `uses` field references external workflows. The prefix matters:

| Prefix | When to Use | Example |
|--------|-------------|---------|
| `marketplace/` | Shared marketplace workflows | `uses: marketplace/script_submitter` |
| `workflow/` | Your own workflows in your account | `uses: workflow/my-custom-workflow` |
| *(no prefix)* | Built-in actions only | `uses: checkout`, `uses: update-session` |

**Common mistake:**
```yaml
# WRONG - will cause "Unknown Error"
- uses: workflow/script_submitter

# CORRECT - marketplace workflows need marketplace/ prefix
- uses: marketplace/script_submitter
```

### 2. Expression Whitespace

Expressions are whitespace-sensitive. Operators need spaces:

```yaml
# WRONG
hidden: ${{ inputs.mode!='advanced' }}

# CORRECT
hidden: ${{ inputs.mode != 'advanced' }}
```

### 3. Missing Required Fields

Every job needs `steps`. Every step needs either `run` or `uses` (not both):

```yaml
# WRONG - missing steps
jobs:
  my-job:
    env:
      FOO: bar

# CORRECT
jobs:
  my-job:
    env:
      FOO: bar
    steps:
      - name: Do something
        run: echo $FOO
```

### 4. Invalid Input Names

Input names can only contain alphanumeric characters, dashes, and underscores:

```yaml
# WRONG
inputs:
  my.input:     # periods not allowed
  my input:     # spaces not allowed

# CORRECT
inputs:
  my_input:
  my-input:
```

### 5. Referencing Grouped Inputs

When inputs are inside a `group`, you must include the group name in the path:

```yaml
on:
  execute:
    inputs:
      config:
        type: group
        items:
          - name: mode
            type: string

jobs:
  example:
    steps:
      # WRONG
      - run: echo "${{ inputs.mode }}"

      # CORRECT - include group name
      - run: echo "${{ inputs.config.mode }}"
```

---

## Overview

Workflows on the ACTIVATE platform are defined using YAML configuration files. These files control:
- Input parameters displayed in the Run Workflow tab
- Job definitions and execution order
- Environment variables and working directories
- Interactive session configuration

**Key concept**: The workflow name is derived from the repository/directory name, not from a `name:` field in the YAML.

---

## Workflow Structure

A complete workflow YAML has this general structure:

```yaml
# Workflow-level configuration
permissions: ["*"]           # Access control
env:                         # Global environment variables
  MY_VAR: "value"
timeout: 2h                  # Overall timeout

# Session definitions (for interactive workflows)
sessions:
  my-session:
    type: tunnel
    redirect: true

# Input form definition
on:
  execute:
    inputs:
      input_name:
        type: string
        label: "My Input"

# Job definitions
jobs:
  job-name:
    needs: [other-job]       # Dependencies
    steps:
      - name: Step Name
        run: echo "Hello"
```

---

## YAML Fields Reference

### Workflow-Level Fields

| Field | Description |
|-------|-------------|
| `permissions` | Access control. `["*"]` grants full user-equivalent capabilities |
| `env` | Global environment variables (lowest priority) |
| `timeout` | Maximum workflow duration (units: s, m, h, d) |
| `app` | Designates as an app; access cluster via `${{ app.target.ip }}` |
| `sessions` | Interactive session definitions |
| `configurations` | Pre-saved input sets for repeated execution |
| `on` | Trigger definition with `on.execute.inputs` for form parameters |

### Job-Level Fields

| Field | Description |
|-------|-------------|
| `jobs.<job>.needs` | Prerequisite jobs that must complete first |
| `jobs.<job>.steps` | Ordered list of execution commands |
| `jobs.<job>.if` | Conditional execution; supports `${{ always }}` |
| `jobs.<job>.env` | Job-scoped environment variables |
| `jobs.<job>.working-directory` | Custom execution directory |
| `jobs.<job>.timeout` | Job-specific timeout |
| `jobs.<job>.ssh` | Remote execution configuration |
| `jobs.<job>.outputs` | Explicit output mapping for dependent jobs |

### Step-Level Fields

| Field | Description |
|-------|-------------|
| `steps[*].name` | Human-readable identifier (shown in Runs tab) |
| `steps[*].run` | Shell command(s) to execute |
| `steps[*].uses` | Reference external workflow. **Use `marketplace/<slug>` for shared workflows, `workflow/<name>` for your own workflows, or bare name for built-in actions (checkout, update-session, scheduler-agent, wait-for-agent)** |
| `steps[*].with` | Input parameters for `uses` workflows |
| `steps[*].cleanup` | Commands run after step (always executes if step ran) |
| `steps[*].retry` | Retry configuration: `max-retries`, `interval`, `timeout` |
| `steps[*].ignore-errors` | Treat non-zero exit as non-fatal |
| `steps[*].early-cancel` | Cancel on condition (e.g., `any-job-failed`) |
| `steps[*].if` | Step-level conditional execution |
| `steps[*].env` | Step-specific environment variables (highest priority) |
| `steps[*].ssh` | Step-level SSH override; `null` executes in user workspace |

### SSH Configuration

```yaml
ssh:
  remoteHost: "${{ needs.provision.outputs.ip }}"
  remoteUser: auto-populated
  jumpNodeHost: optional-jump-ip
  jumpNodeUser: auto-populated
```

---

## Input Types

Inputs are defined under `on.execute.inputs`. Each input requires a `type` and supports optional attributes.

**Naming rule**: Only alphanumeric characters, dashes, and underscores.

### Basic Types

| Type | Description | Special Attributes |
|------|-------------|-------------------|
| `string` | Text input | `textarea: true` for multi-line |
| `number` | Numeric value | `min`, `max`, `step`, `slider: true` |
| `boolean` | Checkbox/toggle | - |
| `password` | Obfuscated text | - |

### Selection Types

| Type | Description |
|------|-------------|
| `dropdown` | Single selection from options |
| `multi-dropdown` | Multiple selections |
| `color-picker` | Color selection |

**Dropdown options example:**
```yaml
my_dropdown:
  type: dropdown
  label: "Select Option"
  default: "option1"
  options:
    - label: "Option 1"
      value: "option1"
    - label: "Option 2"
      value: "option2"
```

### Infrastructure Types

| Type | Description |
|------|-------------|
| `compute-clusters` | Cluster selection |
| `instance-type` | Instance type picker |
| `region` | Region selector |
| `zone` | Zone selector |
| `storage` | Storage selection |
| `storage-image` | Storage image picker |
| `organization-groups` | Org group selection |

### Kubernetes Types

`kubernetes-clusters`, `kubernetes-configmaps`, `kubernetes-deployments`, `kubernetes-namespaces`, `kubernetes-pods`, `kubernetes-pvc`, `kubernetes-secrets`, `kubernetes-services`, `kubernetes-statefulsets`, `kubernetes-workloads`

### SLURM Types

`slurm-accounts`, `slurm-partitions`, `slurm-qos`

### Structural Types

| Type | Description | Key Attributes |
|------|-------------|----------------|
| `group` | Collapsible container | `items: [...]` list of inputs |
| `header` | Display section | `text`, `bold`, `size` |
| `list` | Repeating input patterns | - |
| `editor` | Code input | `language` for syntax highlighting |

### Universal Input Attributes

| Attribute | Description |
|-----------|-------------|
| `default` | Initial value |
| `label` | Human-readable field name |
| `optional` | Whether value is required (boolean) |
| `disabled` | Read-only field (boolean) |
| `hidden` | Conditional visibility using expressions |
| `tooltip` | Hover help text |
| `ignore` | Exclude from backend payload |

---

## Expressions

Expressions are wrapped in `${{ }}` and enable dynamic behavior in forms and jobs.

**Critical**: Expressions are whitespace-sensitive, requiring spaces between operators and operands.

### Operators

| Category | Operators |
|----------|-----------|
| Logical | `&&`, `\|\|`, `!` |
| Comparison | `==`, `!=`, `===`, `!==`, `>`, `<`, `>=`, `<=` |
| Arithmetic | `+`, `-`, `*`, `/`, `**`, `//` (floor division) |
| Special | `??` (nullish coalescing), `get`, `in`, `!in`, `? :` (ternary) |

### Available Contexts

| Context | Description | Example |
|---------|-------------|---------|
| `inputs` | Access input values | `${{ inputs.my_input }}` |
| `inputs.group.field` | Grouped input access | `${{ inputs.config.mode }}` |
| `needs.<job>.outputs.<name>` | Job outputs | `${{ needs.build.outputs.version }}` |
| `needs.<job>.steps.<id>.outputs.<name>` | Step outputs | `${{ needs.job1.steps.get_port.outputs.PORT }}` |
| `app.target` | App cluster info | `${{ app.target.ip }}` |
| `sessions` | Session data | `${{ sessions.my_session }}` |
| `org.<variable>` | Org variables | `${{ org.api_key }}` |

### Common Expression Patterns

**Conditional visibility:**
```yaml
advanced_options:
  type: group
  hidden: ${{ !inputs.show_advanced }}
  items: [...]
```

**Dynamic defaults:**
```yaml
gpu_count:
  type: number
  default: ${{ inputs.model_size == 'large' ? 4 : 1 }}
```

**Value-based hiding:**
```yaml
huggingface_model:
  type: string
  hidden: ${{ inputs.model_source != 'huggingface' }}
```

---

## Actions

### checkout

Retrieves Git repositories for use in workflows.

```yaml
- name: Clone Repository
  uses: checkout
  with:
    repo: "https://github.com/org/repo.git"
    branch: "main"
    path: "/path/to/destination"        # Optional
    sparse_checkout:                     # Optional
      - "src/"
      - "scripts/"
```

### update-session

Configures session display and connectivity.

**Tunnel type:**
```yaml
- name: Configure Session
  uses: update-session
  with:
    name: my-session
    type: tunnel
    remotePort: 8080
    remoteHost: localhost              # Optional
    slug: "my-app"                     # Optional URL suffix
    target: "${{ inputs.cluster }}"   # Optional cluster ID
```

**Link type:**
```yaml
- name: External Link Session
  uses: update-session
  with:
    name: my-session
    type: link
    url: "https://external-service.com"
```

### scheduler-agent

Provisions compute nodes and establishes SSH access.

```yaml
- name: Provision Compute
  uses: scheduler-agent
  with:
    wait: true                    # Block until ready (default)
    scheduler-type: slurm         # or "pbs"
    scheduler-flags: |
      --partition=gpu
      --gres=gpu:1
      --time=04:00:00
```

### wait-for-agent

Waits for async agent provisioning.

```yaml
- name: Wait for Resources
  uses: wait-for-agent
  with:
    agentId: "${{ needs.provision.steps.agent.outputs.agentId }}"
```

---

## Job Dependencies and Outputs

### Defining Dependencies

```yaml
jobs:
  setup:
    steps:
      - name: Initialize
        run: echo "Setting up..."

  build:
    needs: [setup]    # Runs after setup completes
    steps:
      - name: Build
        run: make build

  deploy:
    needs: [setup, build]    # Runs after both complete
    steps:
      - name: Deploy
        run: ./deploy.sh
```

### Writing Outputs

Append `key=value` pairs to the `$OUTPUTS` environment variable:

```yaml
- name: Get Version
  id: get_version
  run: |
    VERSION=$(cat version.txt)
    echo "VERSION=$VERSION" >> $OUTPUTS
    echo "BUILD_DATE=$(date +%Y%m%d)" >> $OUTPUTS
```

### Consuming Outputs

```yaml
jobs:
  producer:
    outputs:
      version: ${{ needs.producer.steps.get_version.outputs.VERSION }}
    steps:
      - name: Get Version
        id: get_version
        run: echo "VERSION=1.0.0" >> $OUTPUTS

  consumer:
    needs: [producer]
    steps:
      - name: Use Version
        run: echo "Building version ${{ needs.producer.outputs.version }}"
```

---

## Interactive Sessions

### Session Definition

```yaml
sessions:
  my-session:
    type: tunnel              # or "link"
    redirect: true            # Redirect user after execution
    useTLS: true             # Enable HTTPS
    prompt-for-name: false   # User naming prompt
    openAI: false            # OpenAI API integration
```

### Two-Script Architecture

Interactive sessions typically use two scripts:

**setup.sh** (runs on controller/login node):
- Downloads software from GitHub, PyPI
- Pulls containers via Git LFS
- Generates passwords/tokens
- Writes `SETUP_COMPLETE` coordination file

**start.sh** (runs on compute node):
- Verifies setup completed
- Finds available port
- Writes `HOSTNAME` and `SESSION_PORT` files
- Starts service in background with logging

### Coordination Files

| File | Purpose |
|------|---------|
| `SETUP_COMPLETE` | Signals setup finished |
| `job.started` | Indicates job is running |
| `HOSTNAME` | Contains service hostname |
| `SESSION_PORT` | Contains service port |
| `job.ended` | Marks completion |

### Session Update Pattern

```yaml
jobs:
  wait_for_service:
    needs: [session_runner]
    steps:
      - name: Wait for Ready
        run: |
          source ./wait_for_service.sh
          # Reads HOSTNAME and SESSION_PORT

  update_session:
    needs: [wait_for_service]
    steps:
      - name: Configure Proxy
        uses: update-session
        with:
          name: session
          remoteHost: "${{ needs.wait_for_service.outputs.hostname }}"
          remotePort: "${{ needs.wait_for_service.outputs.port }}"
```

---

## Best Practices

### Input Design

1. **Group related inputs** using the `group` type for organization
2. **Provide sensible defaults** - make parameters optional when reasonable
3. **Use dropdowns** for fixed options rather than free text
4. **Hide advanced options** until needed with conditional visibility

```yaml
on:
  execute:
    inputs:
      basic_config:
        type: group
        label: "Basic Configuration"
        items:
          - name: mode
            type: dropdown
            default: "standard"
            options:
              - label: "Standard"
                value: "standard"
              - label: "Advanced"
                value: "advanced"

      advanced_config:
        type: group
        label: "Advanced Options"
        hidden: ${{ inputs.basic_config.mode != 'advanced' }}
        items:
          - name: custom_flags
            type: string
            optional: true
```

### Script Patterns

1. **Always quote bash variables**: `"${VARIABLE:?Required}"`
2. **Use absolute paths** instead of relative references
3. **Separate controller/compute logic** in interactive workflows
4. **Implement cleanup handlers** with trap

```bash
#!/bin/bash
set -euo pipefail

# Load environment
source "${JOB_DIR}/inputs.env"

# Cleanup on exit
trap 'cleanup_function' EXIT

# Use required variables with defaults
MODEL="${MODEL_NAME:?Model name required}"
OUTPUT_DIR="${OUTPUT_DIR:-./output}"
```

### Container Management

1. **Support auto-build** - allow both pre-built and on-demand containers
2. **Use Git LFS** for large container files
3. **Cache models/containers** to prevent redundant downloads

```yaml
container_source:
  type: dropdown
  label: "Container Source"
  options:
    - label: "Pre-built (faster)"
      value: "prebuilt"
    - label: "Build from definition"
      value: "build"

container_path:
  type: string
  hidden: ${{ inputs.container_source != 'prebuilt' }}
  label: "Container Path"
```

### Error Handling

1. Use `early-cancel: any-job-failed` for fast failure
2. Use `ignore-errors: true` sparingly for non-critical steps
3. Configure `retry` for transient failures

```yaml
steps:
  - name: Critical Step
    run: ./must-succeed.sh
    early-cancel: any-job-failed

  - name: Retry on Failure
    run: ./flaky-network-call.sh
    retry:
      max-retries: 3
      interval: 10s
      timeout: 60s

  - name: Optional Cleanup
    run: ./cleanup.sh
    ignore-errors: true
```

---

## Example Patterns

### Basic Workflow

```yaml
permissions: ["*"]

on:
  execute:
    inputs:
      message:
        type: string
        label: "Message"
        default: "Hello, World!"

jobs:
  greet:
    steps:
      - name: Print Message
        run: echo "${{ inputs.message }}"
```

### GPU Job with Scheduler

```yaml
permissions: ["*"]

on:
  execute:
    inputs:
      cluster:
        type: compute-clusters
        label: "GPU Cluster"

      scheduler_config:
        type: group
        label: "Scheduler Options"
        items:
          - name: partition
            type: slurm-partitions
            label: "Partition"
          - name: gpu_count
            type: number
            label: "GPUs"
            default: 1
            min: 1
            max: 8

jobs:
  provision:
    steps:
      - name: Get Compute Node
        id: agent
        uses: scheduler-agent
        with:
          scheduler-type: slurm
          scheduler-flags: |
            --partition=${{ inputs.scheduler_config.partition }}
            --gres=gpu:${{ inputs.scheduler_config.gpu_count }}

  run_training:
    needs: [provision]
    ssh:
      remoteHost: "${{ needs.provision.steps.agent.outputs.ip }}"
    steps:
      - name: Train Model
        run: python train.py
```

### Interactive Session Pattern

```yaml
permissions: ["*"]

sessions:
  desktop:
    type: tunnel
    redirect: true

on:
  execute:
    inputs:
      cluster:
        type: compute-clusters
        label: "Cluster"

jobs:
  preprocessing:
    steps:
      - name: Checkout Scripts
        uses: checkout
        with:
          repo: "https://github.com/org/session-scripts.git"
          branch: main

      - name: Setup Environment
        run: ./setup.sh

  session_runner:
    needs: [preprocessing]
    ssh:
      remoteHost: "${{ inputs.cluster }}"
    steps:
      - name: Start Service
        run: ./start.sh

  wait_for_service:
    needs: [session_runner]
    steps:
      - name: Wait for Ready
        id: wait
        run: |
          while [ ! -f HOSTNAME ]; do sleep 2; done
          echo "HOSTNAME=$(cat HOSTNAME)" >> $OUTPUTS
          echo "PORT=$(cat SESSION_PORT)" >> $OUTPUTS

  update_session:
    needs: [wait_for_service]
    steps:
      - name: Configure Proxy
        uses: update-session
        with:
          name: desktop
          remoteHost: "${{ needs.wait_for_service.steps.wait.outputs.HOSTNAME }}"
          remotePort: "${{ needs.wait_for_service.steps.wait.outputs.PORT }}"
```

---

## SDK and API

The Parallel Works SDK provides programmatic access to the ACTIVATE platform. Available in Python, TypeScript, and Go.

### Python SDK Quick Start

```python
import os
from parallelworks_client import Client

# Automatic credential detection (recommended)
with Client.from_credential(os.environ["PW_API_KEY"]).sync() as client:
    # List buckets
    response = client.get("/api/buckets")
    for bucket in response.json():
        print(f"Bucket: {bucket['name']}")

    # List clusters
    clusters = client.get("/api/clusters").json()

    # Get workflow runs
    runs = client.get("/api/workflows/my-workflow/runs").json()
```

### Async Python

```python
import asyncio
from parallelworks_client import Client

async def main():
    async with Client.from_credential(os.environ["PW_API_KEY"]) as client:
        buckets, clusters = await asyncio.gather(
            client.get("/api/buckets"),
            client.get("/api/clusters"),
        )
        print(buckets.json())

asyncio.run(main())
```

### Authentication Methods

| Method | Format | Use Case |
|--------|--------|----------|
| API Key | `pwt_[base64].[secret]` | Long-running integrations, CI/CD |
| JWT Token | `eyJhbGci...` (3 parts) | Scripts, expires in 24h |

The SDK auto-detects credential type and extracts the platform host from the credential.

### Key API Endpoints

| Endpoint | Description |
|----------|-------------|
| `/api/buckets` | Storage buckets |
| `/api/clusters` | Compute clusters |
| `/api/workflows` | Workflow management |
| `/api/workflows/{id}/runs` | Workflow executions |
| `/api/scheduler-jobs` | Job scheduling |
| `/api/auth/whoami` | Current user info |

### SDK Repository

- [parallelworks/sdk](https://github.com/parallelworks/sdk) - Official SDK (Python, TypeScript, Go)

---

## CLI Reference

The `pw` CLI enables command-line interaction with ACTIVATE resources.

### Installation

```bash
# Linux AMD64
mkdir -p ~/bin
wget "https://activate.parallel.works/cli/pw-linux-amd64" -O ~/bin/pw
chmod +x ~/bin/pw
export PATH="$HOME/bin:$PATH"
```

### Authentication

```bash
# Using API key
pw auth apikey

# Using token
pw auth token
```

### Common Commands

```bash
# List clusters
pw cluster list

# List buckets
pw buckets list

# Copy files to/from buckets
pw buckets cp local-file.txt pw://namespace/bucket/path/
pw buckets cp pw://namespace/bucket/file.txt ./local/

# List workflows
pw workflow list

# Run a workflow
pw workflow run my-workflow --input key=value
```

### URI Formats

| Provider | Format |
|----------|--------|
| Parallel Works | `pw://namespace/bucket` |
| Google Cloud | `gs://bucket-name` |
| AWS S3 | `s3://bucket-name` |
| Azure Blob | `https://name.blob.core.windows.net/container` |

---

## Additional Resources

- [Official Documentation](https://parallelworks.com/docs/run/workflows-and-apps/building-workflows)
- [API Reference](https://parallelworks.com/api)
- [CLI Documentation](https://parallelworks.com/docs/cli)
- [SDK Repository](https://github.com/parallelworks/sdk)

### Reference Workflow Repositories

- [activate-medical-finetuning](https://github.com/parallelworks/activate-medical-finetuning) - GPU training workflow
- [activate-sessions](https://github.com/parallelworks/activate-sessions) - Interactive sessions
- [activate-rag-vllm](https://github.com/parallelworks/activate-rag-vllm) - vLLM + RAG deployment
- [interactive_session](https://github.com/parallelworks/interactive_session) - VNC server patterns
