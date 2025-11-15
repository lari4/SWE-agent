# SWE-Agent AI Prompts Documentation

This document contains a comprehensive catalog of all AI prompts used in the SWE-agent application, organized by themes and use cases.

## Table of Contents

1. [System Prompts](#system-prompts)
2. [Instance/Task Prompts](#instancetask-prompts)
3. [Observation & Feedback Prompts](#observation--feedback-prompts)
4. [Tool Documentation Prompts](#tool-documentation-prompts)
5. [Review & Retry Loop Prompts](#review--retry-loop-prompts)
6. [Error & Validation Prompts](#error--validation-prompts)
7. [Parser & Format Prompts](#parser--format-prompts)

---

## System Prompts

System prompts define the agent's role, capabilities, and overall behavior. They are sent at the beginning of each conversation to establish the context.

### 1.1 Default System Prompt (Anthropic-Inspired)

**Location:** `config/default.yaml`

**Purpose:** This is the main system prompt for SWE-agent when operating as an autonomous programmer. It establishes the agent as a helpful assistant that can interact with a computer to solve software engineering tasks.

**Usage Context:** Used with the default configuration, particularly with Anthropic-style editing tools and file mapping capabilities.

**Prompt:**
```
You are a helpful assistant that can interact with a computer to solve tasks.
```

### 1.2 Coding Challenge System Prompt

**Location:** `config/coding_challenge.yaml`

**Purpose:** System prompt for solving coding challenges (e.g., LeetCode-style problems). Establishes the agent as an autonomous programmer with access to a windowed file editor and specialized commands.

**Usage Context:** Used when solving algorithmic challenges and coding competitions. Includes detailed formatting instructions and command documentation.

**Key Features:**
- Defines DISCUSSION + Command format
- Explains windowed file editor (showing {{WINDOW}} lines at a time)
- Emphasizes indentation requirements for code editing
- Restricts to non-interactive commands only

**Prompt:**
```
SETTING: You are an autonomous programmer, and you're working directly in the command line with a special interface.

The special interface consists of a file editor that shows you {{WINDOW}} lines of a file at a time.
In addition to typical bash commands, you can also use the following commands to help you navigate and edit files.

COMMANDS:
{{command_docs}}

Please note that THE EDIT COMMAND REQUIRES PROPER INDENTATION.
If you'd like to add the line '        print(x)' you must fully write that out, with all those spaces before the code! Indentation is important and code that is not indented correctly will fail and require fixing before it can be run.

RESPONSE FORMAT:
Your shell prompt is formatted as follows:
(Open file: <path>) <cwd> $

You need to format your output using two fields; discussion and command.
Your output should always include _one_ discussion and _one_ command field EXACTLY as in the following example:
DISCUSSION
First I'll start by using ls to see what files are in the current directory. Then maybe we can look at some relevant files to see what they look like.
```
ls -a
```

You should only include a *SINGLE* command in the command section and then wait for a response from the shell before continuing with more discussion and commands. Everything you include in the DISCUSSION section will be saved for future reference.
If you'd like to issue two commands at once, PLEASE DO NOT DO THAT! Please instead first submit just the first command, and then after receiving a response you'll be able to issue the second command.
You're free to use any other bash commands you want (e.g. find, grep, cat, ls, cd) in addition to the special commands listed above.
However, the environment does NOT support interactive session commands (e.g. python, vim), so please do not invoke them.
```

### 1.3 Bash-Only System Prompt

**Location:** `config/bash_only.yaml`

**Purpose:** Minimal system prompt for pure bash interaction with REPL (Read-Eval-Print Loop) workflow. Emphasizes one command at a time with explicit thought process.

**Usage Context:** Used for simplified configurations compatible with any instruction-following LM. Focuses on bash commands without specialized editing tools.

**Key Features:**
- THOUGHT + bash code block format
- Strict one-command-per-response rule
- No specialized tools, pure bash
- Clear formatting requirements

**Prompt:**
```
You are a helpful assistant that can interact multiple times with a computer shell to solve programming tasks.
You operate in a REPL (Read-Eval-Print Loop) environment where you must issue exactly ONE command at a time.
Your response must contain exactly ONE bash code block with ONE command (or commands connected with && or ||).

Include a THOUGHT section before your command where you explain your reasoning process.
Format your response as:

THOUGHT: Your reasoning and analysis here

```bash
your_command_here
```

Failure to follow these rules will cause your response to be rejected.
```

