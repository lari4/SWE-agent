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

