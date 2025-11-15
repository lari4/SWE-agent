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

---

## Retry Loop Pipelines

Retry loops allow the agent to make multiple attempts at solving a problem, with different strategies for selecting the best solution.

### 2.1 Score-Based Retry Loop Pipeline

This pipeline makes multiple attempts and uses an LLM reviewer to score each attempt, selecting the best one.

#### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│              SCORE RETRY LOOP INITIALIZATION                    │
├─────────────────────────────────────────────────────────────────┤
│  Configuration:                                                 │
│  ├─ max_attempts: Maximum number of attempts                    │
│  ├─ accept_score: Score threshold for accepting solution        │
│  ├─ max_accepts: Stop after N accepted solutions                │
│  ├─ cost_limit: Maximum total cost for all attempts+reviews     │
│  └─ reviewer_config: Reviewer LLM configuration                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ATTEMPT LOOP                                 │
│         for i in range(max_attempts):                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  Check Budget:                          │
        │  ├─ Total cost < cost_limit?            │
        │  └─ Enough budget for new attempt?      │
        └─────────────────────────────────────────┘
                              │
                    ┌─────────┴──────────┐
                    │                    │
                   NO                   YES
                    │                    │
                    ▼                    ▼
            ┌──────────────┐    ┌──────────────────┐
            │  STOP RETRY  │    │  START ATTEMPT i │
            └──────────────┘    └──────────────────┘
                                         │
                                         ▼
                    ┌─────────────────────────────────────┐
                    │  Reset Agent:                       │
                    │  ├─ Clear history                   │
                    │  ├─ Clear trajectory                │
                    │  ├─ Re-add system message           │
                    │  ├─ Re-add demonstrations           │
                    │  └─ Re-add instance message         │
                    └─────────────────────────────────────┘
                                         │
                                         ▼
                    ┌─────────────────────────────────────┐
                    │  Run Agent Until Submission         │
                    │  (Main Execution Loop)              │
                    └─────────────────────────────────────┘
                                         │
                                         ▼
                    ┌─────────────────────────────────────┐
                    │  REVIEW SUBMISSION                  │
                    └─────────────────────────────────────┘
                                         │
                                         ▼
        ┌─────────────────────────────────────────────────────────┐
        │           REVIEWER.REVIEW()                             │
        ├─────────────────────────────────────────────────────────┤
        │  1. Check exit_status:                                  │
        │     └─ If not "submitted", apply failure penalty        │
        │  2. Format reviewer messages:                           │
        │     ├─ System: reviewer_config.system_template          │
        │     └─ User: reviewer_config.instance_template          │
        │         ├─ Input: problem_statement                     │
        │         ├─ Input: trajectory (formatted)                │
        │         └─ Input: submission (patch)                    │
        │  3. Query reviewer LLM (n_sample times)                 │
        │  4. Extract scores from responses:                      │
        │     └─ Parse last number from each response             │
        │  5. Calculate final score:                              │
        │     ├─ Average of all samples                           │
        │     ├─ Subtract failure penalty (if not submitted)      │
        │     └─ Subtract std * reduce_by_std (if configured)     │
        │  6. Return ReviewerResult:                              │
        │     ├─ accept (score)                                   │
        │     ├─ outputs (reviewer responses)                     │
        │     └─ messages (reviewer conversation)                 │
        └─────────────────────────────────────────────────────────┘
                                         │
                                         ▼
                    ┌─────────────────────────────────────┐
                    │  Store Submission:                  │
                    │  ├─ trajectory                      │
                    │  ├─ info                            │
                    │  ├─ model_stats                     │
                    │  └─ review_score                    │
                    └─────────────────────────────────────┘
                                         │
                                         ▼
                    ┌─────────────────────────────────────┐
                    │  Check Termination Conditions:      │
                    │  ├─ Score >= accept_score?          │
                    │  ├─ N accepted >= max_accepts?      │
                    │  ├─ i >= max_attempts?              │
                    │  └─ Cost >= cost_limit?             │
                    └─────────────────────────────────────┘
                                         │
                              ┌──────────┴──────────┐
                              │                     │
                        CONTINUE              STOP RETRY
                              │                     │
                              └──────────┬──────────┘
                                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SELECT BEST ATTEMPT                          │
├─────────────────────────────────────────────────────────────────┤
│  1. Find attempt with highest review score                     │
│  2. Return that attempt's:                                      │
│     ├─ submission (patch)                                       │
│     ├─ trajectory                                               │
│     └─ info                                                     │
└─────────────────────────────────────────────────────────────────┘
```

#### Data Flow

**Inputs:**
- Problem statement
- Multiple agent attempts (trajectories)
- Reviewer configuration (system_template, instance_template)

**Data Passed to Reviewer:**
- `{{problem_statement}}`: Original issue
- `{{traj}}`: Formatted trajectory showing agent's actions
- `{{submission}}`: Git patch from the attempt
- `{{diff}}`: Git diff of changes

**Reviewer Output:**
- Numerical score (e.g., 0-100)
- Textual explanation

**Final Output:**
- Best attempt (highest score above threshold)

