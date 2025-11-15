# SWE-Agent Pipelines Documentation

This document describes all the execution pipelines and workflows in the SWE-agent system, showing how different components interact and what data flows between them.

## Table of Contents

1. [Main Agent Execution Pipeline](#main-agent-execution-pipeline)
2. [Retry Loop Pipelines](#retry-loop-pipelines)
3. [Action Sampling Pipelines](#action-sampling-pipelines)
4. [Review & Selection Pipelines](#review--selection-pipelines)
5. [Tool Execution Pipeline](#tool-execution-pipeline)
6. [History Processing Pipeline](#history-processing-pipeline)

---

## Main Agent Execution Pipeline

This is the core pipeline that executes when you run the agent on a problem instance.

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        AGENT.RUN()                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SETUP PHASE                                 │
├─────────────────────────────────────────────────────────────────┤
│  1. Install tools in environment                                │
│  2. Format system message from system_template                  │
│     ├─ Input: problem_statement, working_dir, command_docs      │
│     └─ Output: system message                                   │
│  3. Add system message to history                               │
│  4. Load and add demonstrations (if configured)                 │
│     ├─ Format with demonstration_template                       │
│     └─ Add to history as single message or step-by-step         │
│  5. Format instance message from instance_template              │
│     ├─ Input: problem_statement, working_dir, state             │
│     └─ Output: instance message                                 │
│  6. Add instance message to history                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   MAIN EXECUTION LOOP                           │
│                   while not done:                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │         STEP() - Single Iteration       │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │    FORWARD_WITH_HANDLING()              │
        │    (See Action Execution Pipeline)      │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │    Process Step Output:                 │
        │    ├─ thought (string)                  │
        │    ├─ action (string)                   │
        │    ├─ observation (string)              │
        │    ├─ exit_status (submitted/error/etc) │
        │    └─ submission (patch string)         │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  Add to History:                        │
        │  ├─ Assistant message (thought+action)  │
        │  └─ User message (observation)          │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  Add to Trajectory:                     │
        │  └─ Complete step data                  │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  Update Info:                           │
        │  ├─ submission                          │
        │  ├─ exit_status                         │
        │  ├─ edited_files                        │
        │  └─ model_stats (cost, tokens, etc)     │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  Save Trajectory to Disk                │
        └─────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  Check if Done:                         │
        │  ├─ Agent submitted?                    │
        │  ├─ Cost limit exceeded?                │
        │  ├─ Max steps reached?                  │
        │  └─ Error occurred?                     │
        └─────────────────────────────────────────┘
                              │
                              ▼
                   ┌─────────┴──────────┐
                   │                    │
                  YES                  NO
                   │                    │
                   ▼                    │
           ┌──────────────┐             │
           │  END LOOP    │             │
           └──────────────┘             │
                   │                    │
                   │                    │
                   │◄───────────────────┘
                   │       CONTINUE
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FINALIZATION PHASE                           │
├─────────────────────────────────────────────────────────────────┤
│  1. Save final trajectory                                       │
│  2. Return AgentRunResult:                                      │
│     ├─ info (all metadata)                                      │
│     └─ trajectory (complete action/observation history)         │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

**Inputs to Run:**
- `problem_statement`: ProblemStatement object containing issue description
- `env`: SWEEnv environment instance
- `output_dir`: Path for trajectory storage

**Outputs from Run:**
- `AgentRunResult`:
  - `info`: Dictionary with submission, exit_status, model_stats, edited_files
  - `trajectory`: List of trajectory steps with actions and observations

**Key Template Variables Used:**
- `{{problem_statement}}`: Issue description
- `{{working_dir}}`: Repository directory path
- `{{command_docs}}`: Auto-generated tool documentation
- `{{open_file}}`: Currently open file (if using windowed editor)
- `{{observation}}`: Output from previous command

