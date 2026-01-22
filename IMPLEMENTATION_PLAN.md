# ACTIVATE AI Reference Repository - Implementation Plan

## Purpose

Create a centralized reference repository that AI code assistants (Claude, Codex, etc.) can use to quickly understand and assist with Parallel Works ACTIVATE platform development.

## Proposed Repository Structure

```
activate-ai-reference/
├── README.md                    # Repository overview and quick navigation
├── WORKFLOW_BUILDER.md          # Comprehensive workflow YAML guide
├── INTERACTIVE_SESSIONS.md      # Interactive session development guide
├── API_REFERENCE.md             # SDK/API quick reference
├── CLI_REFERENCE.md             # CLI commands and usage
└── examples/
    ├── basic-workflow.yaml      # Minimal workflow example
    ├── gpu-job-workflow.yaml    # GPU/scheduler workflow example
    └── interactive-session.yaml # Interactive session example
```

## Document Descriptions

### 1. WORKFLOW_BUILDER.md (Priority: High)
Consolidated guide covering:
- Workflow YAML structure and all fields
- Input types (37 types) with attributes
- Expressions syntax and operators
- Actions (checkout, update-session, scheduler-agent, wait-for-agent)
- Job dependencies and execution flow
- Conditional logic patterns
- Best practices and common patterns
- Annotated examples from real workflows

**Sources consolidated:**
- WORKFLOW_BUILDER_GUIDE.md (activate-medical-finetuning)
- DEVELOPER_GUIDE.md (activate-sessions)
- Official docs: yaml-fields, inputs-and-expressions, actions

### 2. INTERACTIVE_SESSIONS.md (Priority: Medium)
Guide for building interactive session workflows:
- Two-script architecture (setup.sh / start.sh)
- Controller vs compute node responsibilities
- Session coordination files
- Port forwarding and proxy configuration
- Service health checks
- Cleanup and error handling

### 3. API_REFERENCE.md (Priority: Medium)
Quick reference for the ACTIVATE API:
- Authentication methods (API key, Bearer token)
- Common endpoints by category
- Request/response examples
- Python SDK usage patterns

### 4. CLI_REFERENCE.md (Priority: Low)
CLI command reference:
- Installation across platforms
- Authentication setup
- Common commands (cluster, workflow, storage)
- URI formats for different providers

### 5. examples/ directory (Priority: Medium)
Annotated example workflows:
- Basic "hello world" workflow
- GPU job with SLURM/PBS
- Interactive session with VNC/desktop

## Implementation Order

1. **WORKFLOW_BUILDER.md** - Most comprehensive and frequently needed
2. **examples/** - Concrete references for AI assistants
3. **INTERACTIVE_SESSIONS.md** - Common use case
4. **API_REFERENCE.md** - For programmatic integrations
5. **CLI_REFERENCE.md** - For command-line operations
6. **README.md** - Update with final navigation

## Key Design Principles

1. **AI-Optimized Structure**: Use clear headings, code blocks, and tables that AI assistants can easily parse

2. **Self-Contained**: Each document should be usable standalone without requiring full context

3. **Example-Rich**: Include annotated examples showing common patterns

4. **Quick Reference Tables**: Provide lookup tables for input types, expression operators, etc.

5. **Link to Official Docs**: Include links to official documentation for deeper exploration

## Next Steps

1. Review and approve this plan
2. I will create WORKFLOW_BUILDER.md as the first deliverable
3. Iterate based on feedback before proceeding to other documents
