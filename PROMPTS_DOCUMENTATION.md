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

---

## Instance/Task Prompts

Instance prompts define the specific task or problem the agent needs to solve. They provide context about the repository, the issue to fix, and the expected workflow.

### 2.1 Default Instance Prompt (PR Description Format)

**Location:** `config/default.yaml`

**Purpose:** Presents a GitHub PR-style problem statement to the agent, instructing it to implement changes to meet the requirements. This is the main prompt for SWE-bench style tasks.

**Usage Context:** Used when fixing bugs or implementing features based on issue descriptions.

**Key Features:**
- Uploaded files context with working directory
- PR description wrapped in XML tags
- Clear 5-step workflow (find code → reproduce → edit → verify → test edge cases)
- Explicit instruction not to modify tests
- Emphasizes minimal changes

**Prompt:**
```
<uploaded_files>
{{working_dir}}
</uploaded_files>
I've uploaded a python code repository in the directory {{working_dir}}. Consider the following PR description:

<pr_description>
{{problem_statement}}
</pr_description>

Can you help me implement the necessary changes to the repository so that the requirements specified in the <pr_description> are met?
I've already taken care of all changes to any of the test files described in the <pr_description>. This means you DON'T have to modify the testing logic or any of the tests in any way!
Your task is to make the minimal changes to non-tests files in the {{working_dir}} directory to ensure the <pr_description> is satisfied.
Follow these steps to resolve the issue:
1. As a first step, it might be a good idea to find and read code relevant to the <pr_description>
2. Create a script to reproduce the error and execute it with `python <filename.py>` using the bash tool, to confirm the error
3. Edit the sourcecode of the repo to resolve the issue
4. Rerun your reproduce script and confirm that the error is fixed!
5. Think about edgecases and make sure your fix handles them as well
Your thinking should be thorough and so it's fine if it's very long.
```

### 2.2 Coding Challenge Instance Prompt

**Location:** `config/coding_challenge.yaml`

**Purpose:** Presents a coding challenge (e.g., LeetCode problem) with specific instructions for solving it in a main.py file.

**Usage Context:** Used for algorithmic problem solving and coding competitions.

**Key Features:**
- ISSUE section with problem description
- Detailed INSTRUCTIONS section
- 5 important tips for efficient problem solving
- Shell prompt format showing open file and current directory
- Emphasis on testing before submission

**Prompt:**
```
We're currently attempting to solve the following problem:
ISSUE:
{{issue}}

INSTRUCTIONS:
Now, you're going to solve this issue on your own. Your terminal session has started and you're in the repository's root directory. You can use any bash commands or the special interface to help you. Edit all the files you need to and run any checks or tests that you want.
Remember, YOU CAN ONLY ENTER ONE COMMAND AT A TIME. You should always wait for feedback after every command.
When you're satisfied with all of the changes you've made, you can submit your changes to the code base by simply running the submit command.
Note however that you cannot use any interactive session commands (e.g. python, vim) in this environment, but you can write scripts and run them. E.g. you can write a python script and then run it with `python <script_name>.py`.

NOTE ABOUT THE EDIT COMMAND: Indentation really matters! When editing a file, make sure to insert appropriate indentation before each line!

IMPORTANT TIPS:
1. Write your solution in main.py. Always test your code thoroughly before submitting, and if any of the tests fail, try to fix the code before continuing.

2. If you run a command and it doesn't work, try running a different command. A command that did not work once will not work the second time unless you modify it!

3. If you open a file and need to get to an area around a specific line that is not in the first 100 lines, say line 583, don't just use the scroll_down command multiple times. Instead, use the goto 583 command. It's much quicker.

4. Always make sure to look at the currently open file and the current working directory (which appears right after the currently open file). The currently open file might be in a different directory than the working directory! Note that some commands, such as 'create', open files, so they might change the current  open file.

5. When editing files, it is easy to accidentally specify a wrong line number or to write code with incorrect indentation. Always check the code after you issue an edit to make sure that it reflects what you wanted to accomplish. If it didn't, issue another command to fix it.

(Open file: {{open_file}})
(Current directory: {{working_dir}})
bash-$
```

### 2.3 Bash-Only Instance Prompt

**Location:** `config/bash_only.yaml`

**Purpose:** Comprehensive instance prompt for bash-only mode with detailed REPL workflow instructions and example commands.

**Usage Context:** Used with minimal configurations that rely solely on bash commands without specialized tools.

**Key Features:**
- Extensive task instructions with workflow recommendations
- Important boundaries (what to modify vs. what not to modify)
- Command execution rules and format requirements
- Correct vs. incorrect response examples
- Useful command examples (sed, cat with heredoc, etc.)

**Prompt:**
```
<uploaded_files>
{{working_dir}}
</uploaded_files>

<pr_description>
I've uploaded a python code repository in the directory {{working_dir}}.
Consider the following PR description:
{{problem_statement}}
</pr_description>

<instructions>
# Task Instructions

## Overview
You're a software engineer interacting continuously with a computer shell in a REPL (Read-Eval-Print Loop) environment.
You'll be helping implement necessary changes to meet requirements in the PR description.
Your task is specifically to make changes to non-test files in the {{working_dir}} directory in order to fix the issue described in the PR description in a way that is general and consistent with the codebase.

IMPORTANT: This is an interactive process where you will think and issue ONE command, see its result, then think and issue your next command.

For each response:
1. Include a THOUGHT section explaining your reasoning and what you're trying to accomplish
2. Provide exactly ONE bash command to execute

## Important Boundaries
- MODIFY: Regular source code files in {{working_dir}}
- DO NOT MODIFY: Tests, configuration files (pyproject.toml, setup.cfg, etc.)

## Recommended Workflow
1. Analyze the codebase by finding and reading relevant files
2. Create a script to reproduce the issue
3. Edit the source code to resolve the issue
4. Verify your fix works by running your script again
5. Test edge cases to ensure your fix is robust

## Command Execution Rules
You are operating in a REPL (Read-Eval-Print Loop) environment where:
1. You write a single command
2. The system executes that command
3. You see the result
4. You write your next command

Each response should include:
1. A **THOUGHT** section where you explain your reasoning and plan
2. A single bash code block with your command

Format your responses like this:
```
THOUGHT: Here I explain my reasoning process, analysis of the current situation,
and what I'm trying to accomplish with the command below.

```bash
your_command_here
```
```

Commands must be specified in a single bash code block:

```bash
your_command_here
```

**CRITICAL REQUIREMENTS:**
- Your response SHOULD include a THOUGHT section explaining your reasoning
- Your response MUST include EXACTLY ONE bash code block
- This bash block MUST contain EXACTLY ONE command (or a set of commands connected with && or ||)
- If you include zero or multiple bash blocks, or no command at all, YOUR RESPONSE WILL FAIL
- Do NOT try to run multiple independent commands in separate blocks in one response

Example of a CORRECT response:
<example_response>
THOUGHT: I need to understand the structure of the repository first. Let me check what files are in the current directory to get a better understanding of the codebase.

```bash
ls -la
```
</example_response>

Example of an INCORRECT response:
<example_response>
THOUGHT: I need to examine the codebase and then look at a specific file. I'll run multiple commands to do this.

```bash
ls -la
```

Now I'll read the file:

```bash
cat file.txt
```
</example_response>

If you need to run multiple commands, either:
1. Combine them in one block using && or ||
```bash
command1 && command2 || echo "Error occurred"
```

2. Wait for the first command to complete, see its output, then issue the next command in your following response.

## Environment Details
- You have a full Linux shell environment
- Always use non-interactive flags (-y, -f) for commands
- Avoid interactive tools like vi, nano, or any that require user input
- If a command isn't available, you can install it

## Useful Command Examples

### Create a new file:
```bash
cat <<'EOF' > newfile.py
import numpy as np
hello = "world"
print(hello)
EOF
```

### Edit files with sed:
```bash
# Replace all occurrences
sed -i 's/old_string/new_string/g' filename.py

# Replace only first occurrence
sed -i 's/old_string/new_string/' filename.py

# Replace first occurrence on line 1
sed -i '1s/old_string/new_string/' filename.py

# Replace all occurrences in lines 1-10
sed -i '1,10s/old_string/new_string/g' filename.py
```

### View file content:
```bash
# View specific lines with numbers
nl -ba filename.py | sed -n '10,20p'
```

### Any other command you want to run
```bash
anything
```

## Submission
When you've completed your changes or can't make further progress:
```bash
submit
```

We'll automatically save your work and have maintainers evaluate it.
</instructions>
```

---

## Observation & Feedback Prompts

These prompts format the system's feedback to the agent after command execution, including command output, error messages, and contextual information.

### 3.1 Default Next Step Template

**Location:** `config/default.yaml`

**Purpose:** Simple template that presents command output to the agent after each action.

**Usage Context:** Used in default configuration to show observation without additional context.

**Prompt:**
```
OBSERVATION:
{{observation}}
```

### 3.2 Default No Output Template

**Location:** `config/default.yaml`

**Purpose:** Informs the agent when a command executes successfully but produces no output.

**Usage Context:** Used to distinguish between empty output and command failure.

**Prompt:**
```
Your command ran successfully and did not produce any output.
```

### 3.3 Coding Challenge Next Step Template

**Location:** `config/coding_challenge.yaml`

**Purpose:** Shows command output along with current file and directory context for coding challenges.

**Usage Context:** Used with windowed file editor to help agent track which file is open and current working directory.

**Key Features:**
- Displays observation output
- Shows currently open file
- Shows current working directory
- Shell prompt format

**Prompt:**
```
{{observation}}
(Open file: {{open_file}})
(Current directory: {{working_dir}})
bash-$
```

### 3.4 Coding Challenge No Output Template

**Location:** `config/coding_challenge.yaml`

**Purpose:** No output message with file and directory context.

**Prompt:**
```
Your command ran successfully and did not produce any output.
(Open file: {{open_file}})
(Current directory: {{working_dir}})
bash-$
```

### 3.5 Bash-Only Observation Template

**Location:** `config/bash_only.yaml`

**Purpose:** XML-formatted observation for bash-only mode.

**Usage Context:** Uses XML tags for clear structure in minimal configurations.

**Prompt:**
```
<observation>
{{observation}}
</observation>
```

### 3.6 Bash-Only No Output Template

**Location:** `config/bash_only.yaml`

**Purpose:** Warning-formatted message when command produces no output.

**Prompt:**
```
<warning>
Your last command ran successfully and did not produce any output.
</warning>
```

### 3.7 Truncated Observation Template (Default)

**Location:** `sweagent/agent/agents.py` (TemplateConfig class)

**Purpose:** Warns the agent when command output exceeds the maximum observation length and has been truncated.

**Usage Context:** Automatically used when observation exceeds `max_observation_length` (default: 100,000 characters).

**Key Features:**
- Shows truncated observation
- Provides character count of elided content
- Suggests alternative commands (head/tail/grep/redirect)
- Warns against interactive pagers

**Prompt:**
```
Observation: {{observation[:max_observation_length]}}<response clipped>
<NOTE>Observations should not exceeded {{max_observation_length}} characters. {{elided_chars}} characters were elided. Please try a different command that produces less output or use head/tail/grep/redirect the output to a file. Do not use interactive pagers.</NOTE>
```

### 3.8 Truncated Observation Template (Bash-Only)

**Location:** `config/bash_only.yaml`

**Purpose:** More sophisticated truncation handling that shows both head and tail of output.

**Usage Context:** Bash-only mode with 10,000 character limit.

**Key Features:**
- Shows first half of observation
- Shows character count of elided middle section
- Shows last half of observation
- Provides specific command suggestions

**Prompt:**
```
<warning>
The output of your last command was too long.
Please try a different command that produces less output.
If you're looking at a file you can try use head, tail or sed to view a smaller number of lines selectively.
If you're using grep or find and it produced too much output, you can use a more selective search pattern.
If you really need to see something from the full command's output, you can redirect output to a file and then search in that file.
</warning>

<observation_head>
{{observation[ : max_observation_length // 2]}}
</observation_head>

<elided_chars>
{{elided_chars}} characters elided
</elided_chars>

<observation_tail>
{{observation[- max_observation_length // 2:]}}
</observation_tail>
```

### 3.9 Command Timeout Template

**Location:** `config/bash_only.yaml`

**Purpose:** Informs the agent when a command is cancelled due to timeout.

**Usage Context:** Commands that exceed execution timeout (default: 60 seconds in bash_only.yaml).

**Prompt:**
```
<warning>
The command '{{command}}' was cancelled because it took more than {{timeout}} seconds to complete.
It may have been waiting for user input or otherwise blocked.
Please try a different command.
</warning>
```

### 3.10 Demonstration Template

**Location:** `config/coding_challenge.yaml`

**Purpose:** Introduces example demonstrations to show the agent how to use the interface correctly.

**Usage Context:** Shown at the beginning when demonstration trajectories are included in the configuration.

**Prompt:**
```
Here is a demonstration of how to correctly accomplish this task.
It is included to show you how to correctly use the interface.
You do not need to follow exactly what is done in the demonstration.
--- DEMONSTRATION ---
{{demonstration}}
--- END OF DEMONSTRATION ---
```

---

## Tool Documentation Prompts

Tool documentation prompts are dynamically generated from tool configuration files and injected into the system prompt via the `{{command_docs}}` variable. Each tool has a signature and docstring that teaches the agent how to use it.

### 4.1 str_replace_editor (Anthropic-Style Editor)

**Location:** `tools/edit_anthropic/config.yaml`

**Purpose:** Advanced file editing tool inspired by Anthropic's computer use demo. Provides view, create, str_replace, insert, and undo operations.

**Usage Context:** Used in default configuration for modern LLMs with function calling capabilities.

**Key Features:**
- Persistent state across commands
- Directory listing and file viewing
- String replacement with exact matching
- Insert operations at specific lines
- Undo functionality

**Tool Signature:**
```
str_replace_editor <command> <path> [<file_text>] [<view_range>] [<old_str>] [<new_str>] [<insert_line>]
```

**Tool Documentation:**
```
Custom editing tool for viewing, creating and editing files
* State is persistent across command calls and discussions with the user
* If `path` is a file, `view` displays the result of applying `cat -n`. If `path` is a directory, `view` lists non-hidden files and directories up to 2 levels deep
* The `create` command cannot be used if the specified `path` already exists as a file
* If a `command` generates a long output, it will be truncated and marked with `<response clipped>`
* The `undo_edit` command will revert the last edit made to the file at `path`

Notes for using the `str_replace` command:
* The `old_str` parameter should match EXACTLY one or more consecutive lines from the original file. Be mindful of whitespaces!
* If the `old_str` parameter is not unique in the file, the replacement will not be performed. Make sure to include enough context in `old_str` to make it unique
* The `new_str` parameter should contain the edited lines that should replace the `old_str`
```

### 4.2 Windowed File Navigation Tools

**Location:** `tools/windowed/config.yaml`

**Purpose:** Windowed file viewer that shows a limited number of lines at a time, with navigation commands.

**Usage Context:** Used in coding_challenge.yaml and other windowed editing configurations.

**Tool Signatures and Documentation:**

#### goto
```
goto <line_number>
```
**Description:** moves the window to show <line_number>

#### open
```
open "<path>" [<line_number>]
```
**Description:** opens the file at the given path in the editor. If line_number is provided, the window will be move to include that line

#### create
```
create <filename>
```
**Description:** creates and opens a new file with the given name

#### scroll_up
```
scroll_up
```
**Description:** moves the window up {WINDOW} lines

#### scroll_down
```
scroll_down
```
**Description:** moves the window down {WINDOW} lines

### 4.3 Search Tools

**Location:** `tools/search/config.yaml`

**Purpose:** File and directory search tools for finding files and searching content.

**Usage Context:** Used in coding challenges and configurations that need search capabilities.

**Tool Signatures and Documentation:**

#### find_file
```
find_file <file_name> [<dir>]
```
**Description:** finds all files with the given name or pattern in dir. If dir is not provided, searches in the current directory. Supports shell-style wildcards (e.g. *.py)

#### search_dir
```
search_dir <search_term> [<dir>]
```
**Description:** searches for search_term in all files in dir. If dir is not provided, searches in the current directory

#### search_file
```
search_file <search_term> [<file>]
```
**Description:** searches for search_term in file. If file is not provided, searches in the current open file

### 4.4 Submit Tool

**Location:** `tools/submit/config.yaml`

**Purpose:** Allows the agent to submit its solution when complete.

**Usage Context:** Used in all configurations to signal task completion.

**Tool Signature:**
```
submit
```
**Description:** submits the current file

---

## Review & Retry Loop Prompts

These prompts are used in the review and retry loop system, which allows the agent to make multiple attempts at solving a problem and selecting the best solution.

### 5.1 Submit Review Messages

**Location:** `config/default.yaml` (registry_variables → SUBMIT_REVIEW_MESSAGES)

**Purpose:** Multi-stage submission validation messages that guide the agent to review their changes before final submission.

**Usage Context:** Shown after the agent runs the `submit` command to ensure quality control.

**Key Features:**
- Requests re-running reproduction script
- Asks to remove temporary scripts
- Warns about reverting test file modifications
- Shows git diff of all changes

**Prompt:**
```
Thank you for your work on this issue. Please carefully follow the steps below to help review your changes.

1. If you made any changes to your code after running the reproduction script, please run the reproduction script again.
  If the reproduction script is failing, please revisit your changes and make sure they are correct.
  If you have already removed your reproduction script, please ignore this step.
2. Remove your reproduction script (if you haven't done so already).
3. If you have modified any TEST files, please revert them to the state they had before you started fixing the issue.
  You can do this with `git checkout -- /path/to/test/file.py`. Use below <diff> to find the files you need to revert.
4. Run the submit command again to confirm.

Here is a list of all of your changes:

<diff>
{{diff}}
</diff>
```

### 5.2 Colleague Discussion Prompts (Ask Colleagues Sampler)

**Location:** `sweagent/agent/action_sampler.py` (AskColleagues class)

**Purpose:** Creates a collaborative discussion between multiple AI "colleagues" to select the best action.

**Usage Context:** Used when action_sampler is configured with type "ask_colleagues". The agent samples multiple actions and synthesizes them.

**How it works:**
1. Model generates N sample actions
2. System formats them as colleague suggestions
3. Agent compares and selects best action

**Prompt:**
```
Your colleagues had the following ideas:

Thought (colleague 0): {thought_0}
Proposed Action (colleague 0): {action_0}

Thought (colleague 1): {thought_1}
Proposed Action (colleague 1): {action_1}

Please summarize and compare the ideas and propose and action to take. Finally choose one action to perform and explain it in detail and include it as a tool call. <important>You must include a thought and action (as a tool/function call). Do not try to invoke commands with triple backticks, use function calls instead.</important>
```

### 5.3 Binary Trajectory Comparison Prompts

**Location:** `sweagent/agent/action_sampler.py` (BinaryTrajectoryComparisonConfig class)

**Purpose:** Expert overseer system that compares two proposed actions and selects the better one through tournament-style elimination.

**Usage Context:** Advanced action sampling that uses pairwise comparisons to find the best action among multiple candidates.

#### System Template
```
<setting>You are an expert software engineer overseeing junior developers. They suggest actions to take to solve a problem. You must choose the best action to take. </setting>
```

#### Instance Template
```
We're solving the following problem

<problem_statement>
{{problem_statement}}
</problem_statement>

So far, we've performed the following actions:

<trajectory>
{{traj}}
</trajectory>
```

#### Comparison Template
```
Two junior developers suggested the following actions:

<thought1>
{{thought1}}
</thought1>

<action1>
{{action1}}
</action1>

<thought2>
{{thought2}}
</thought2>

<action2>
{{action2}}
</action2>

Please compare the two actions in detail.

Which action should we take?

If you think the first action is better, respond with "first".
If you think the second action is better, respond with "second".

The last line of your response MUST be "first" or "second".
```

### 5.4 Reviewer System Prompts

**Location:** `sweagent/agent/reviewer.py` (ReviewerConfig)

**Purpose:** Evaluates completed solutions and assigns quality scores to help select the best attempt.

**Usage Context:** Used in score-based retry loops where the agent makes multiple attempts and the reviewer scores each attempt.

**Key Features:**
- Scores submissions on a configurable range
- Can apply penalties for non-submitted solutions
- Supports multiple samples for robust scoring
- Uses trajectory formatting to show agent's work

**Template Structure:**
- `system_template`: Defines the reviewer's role
- `instance_template`: Provides problem statement and solution trajectory
- Score is extracted from the last number in the response

### 5.5 Chooser System Prompts

**Location:** `sweagent/agent/reviewer.py` (ChooserConfig)

**Purpose:** Selects the best solution from multiple attempts without scoring, using direct comparison.

**Usage Context:** Alternative to score-based review, directly chooses which submission is best.

**Template Structure:**
- `system_template`: Defines the chooser's role
- `instance_template`: Provides problem statement
- `submission_template`: Formats each submission for comparison
- Extracts chosen index from the last number in response

### 5.6 Preselector System Prompts

**Location:** `sweagent/agent/reviewer.py` (PreselectorConfig)

**Purpose:** Pre-filters submissions before final selection to reduce the candidate pool.

**Usage Context:** Used as a first-pass filter when there are many submissions (>2), selecting promising candidates for final review.

**Template Structure:**
- Similar to Chooser but returns multiple indices
- Used before Chooser to narrow down options

---

## Error & Validation Prompts

These prompts are shown to the agent when errors occur or when the agent's output doesn't meet expected format requirements.

### 6.1 Bash Syntax Error Template

**Location:** `sweagent/agent/agents.py` (TemplateConfig class)

**Purpose:** Informs the agent when a bash command has incorrect syntax.

**Usage Context:** Triggered when the command fails bash syntax validation before execution.

**Default Template:**
```
<NOTE>The command you provided has incorrect syntax. Please correct it and try again.</NOTE>
```

### 6.2 Format Error Messages

**Location:** Throughout codebase, raised by FormatError exceptions

**Purpose:** Generic error messages when the agent's response doesn't match expected format.

**Usage Context:** Used by parsers when they cannot extract thought/action from model response.

---

## Parser & Format Prompts

Parser prompts define the expected output format for the agent and provide error messages when the format is incorrect.

### 7.1 ThoughtActionParser Format

**Location:** `sweagent/tools/parsing.py` (ThoughtActionParser class)

**Purpose:** Expects the agent to provide a discussion followed by a command in backticks.

**Usage Context:** Default parser for many configurations, especially coding_challenge.yaml.

**Expected Format:**
```
DISCUSSION
Discuss here with yourself about what your planning and what you're going to do in this step.

```
command(s) that you're going to run
```
```

**Error Message:**
```
Your output was not formatted correctly. You must always include one discussion and one command as part of your response. Make sure you do not have multiple discussion/command tags.
Please make sure your output precisely matches the following format:
DISCUSSION
Discuss here with yourself about what your planning and what you're going to do in this step.

```
command(s) that you're going to run
```
```

### 7.2 XMLThoughtActionParser Format

**Location:** `sweagent/tools/parsing.py` (XMLThoughtActionParser class)

**Purpose:** Expects the agent to provide a discussion followed by a command wrapped in XML `<command>` tags.

**Usage Context:** Used in some configurations that prefer XML-style formatting.

**Expected Format:**
```
Discussion text here
<command>
ls -l
</command>
```

**Error Message:**
```
Your output was not formatted correctly. You must always include one discussion and one command as part of your response. Make sure you do not have multiple discussion/command tags.
Please make sure your output precisely matches the following format:
```

### 7.3 XMLFunctionCallingParser Format

**Location:** `sweagent/tools/parsing.py` (XMLFunctionCallingParser class)

**Purpose:** Expects function/tool calls in XML format with parameters.

**Usage Context:** Used for structured tool calling with multiple parameters.

**Expected Format:**
```
Discussion text here
<function=bash>
<parameter=command>find /testbed -type f -name "_discovery.py"</parameter>
</function>
```

**Error Messages:**

When no function is found:
```
Your last output did not use any tool calls!
Please make sure your output includes exactly _ONE_ function call!
If you think you have already resolved the issue, please submit your changes by running the `submit` command.
If you think you cannot solve the problem, please run `submit`.
Else, please continue with a new tool call!
```

When multiple functions are found:
```
Your last output included multiple tool calls!
Please make sure your output includes a thought and exactly _ONE_ function call.
```

When unexpected arguments are present:
```
Your action could not be parsed properly: {{exception_message}}.
Make sure your function call doesn't include any extra arguments that are not in the allowed arguments, and only use the allowed commands.
```

### 7.4 ActionParser Format

**Location:** `sweagent/tools/parsing.py` (ActionParser class)

**Purpose:** Expects only a command name, no thought or discussion.

**Usage Context:** Minimal parser for simple command-only responses.

**Expected Format:**
```
ls -l
```

**Error Message:**
```
The command you provided was not recognized. Please specify one of the commands (+ any necessary arguments) from the following list in your response. Do not include any other text.

COMMANDS:
{command_docs}
```

### 7.5 EditFormat Parser

**Location:** `sweagent/tools/parsing.py` (EditFormat class)

**Purpose:** Specialized parser for file editing that expects replacement text in backticks.

**Usage Context:** Used with windowed editing tools that replace entire window contents.

**Expected Format:**
```
Comments about what you're going to do

```
New window contents.
Make sure you copy the entire contents of the window here, with the required indentation.
```
```

**Error Message:**
```
Your output was not formatted correctly. You must wrap the replacement text in backticks (```).
Please make sure your output precisely matches the following format:
COMMENTS
You can write comments here about what you're going to do if you want.

```
New window contents.
Make sure you copy the entire contents of the window here, with the required indentation.
Make the changes to the window above directly in this window.
Remember that all of the window's contents will be replaced with the contents of this window.
Don't include line numbers in your response.
```
```

### 7.6 FunctionCallingParser Format

**Location:** `sweagent/tools/parsing.py` (FunctionCallingParser class)

**Purpose:** Native function calling format for models that support tool use API.

**Usage Context:** Default for modern LLMs (GPT-4, Claude 3.5+) that support native function calling.

**Expected Format:**
Uses the model's native function calling API format (JSON-based tool calls).

**Key Features:**
- No need for `{{command_docs}}` in system prompt
- Automatic tool schema generation
- Structured parameter validation
- Native model support for tool calling

---

## Summary

This documentation covers all major AI prompts used in SWE-agent across seven categories:

1. **System Prompts**: Define agent role and capabilities
2. **Instance/Task Prompts**: Provide specific problem context
3. **Observation & Feedback Prompts**: Format command output and feedback
4. **Tool Documentation Prompts**: Teach agent how to use available tools
5. **Review & Retry Loop Prompts**: Enable multi-attempt problem solving
6. **Error & Validation Prompts**: Handle errors and edge cases
7. **Parser & Format Prompts**: Enforce output structure

These prompts work together to create a comprehensive autonomous coding agent system that can understand problems, use tools, handle errors, and iterate toward solutions.

