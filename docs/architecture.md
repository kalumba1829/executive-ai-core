# Architecture

This document describes the technical architecture of ExecAI.

---

## Overview

ExecAI implements the simplest possible executive AI architecture : a linear pipeline with no memory, no branching, and no tool use. This is intentional. The goal is to make every part of the system visible and understandable.

---

## Pipeline

```
 User Input (string)
        |
        v
 +-------------+
 |   plan()    |   One API call.
 |             |   System prompt : "planner"
 |             |   Input  : the raw goal string
 |             |   Output : raw text (numbered list)
 |             |   Parsed : Python list of strings
 +-------------+
        |
        v
 List of steps  [step_1, step_2, ..., step_n]
        |
        v
 +-------------+
 | execute()   |   N API calls (one per step).
 |  (loop)     |   System prompt : "executor"
 |             |   Input  : one step string
 |             |   Output : result string
 +-------------+
        |
        v
 List of results  [result_1, result_2, ..., result_n]
        |
        v
 +-------------+
 |   run()     |   No API call.
 |  (output)   |   Prints steps and results to stdout.
 +-------------+
```

Total API calls for a goal that produces N steps : N + 1

---

## Module Breakdown

### `ask_ai(role, task) -> str`

This is the only function that communicates with the Anthropic API. All other functions call this one. It wraps `client.messages.create()` and returns the text content of the first response block.

Having a single point of contact with the API makes it easy to swap providers later (OpenAI, Mistral, a local Ollama model, etc.) by modifying only this function.

### `plan(goal) -> list`

Calls `ask_ai()` with a strict "planner" system prompt. The prompt instructs Claude to return only a numbered list, with no additional text. The raw string response is split on newlines and filtered for empty lines.

This parsing is naive. If Claude adds an introduction despite the instruction, the first line of that introduction will appear as a step. More robust implementations would use structured outputs (JSON) instead of plain text.

### `execute(step) -> str`

Calls `ask_ai()` with an "executor" system prompt. Each call is independent. The executor has no knowledge of the other steps, the original goal, or the previous results. This is a stateless design.

### `run(goal) -> list`

Orchestrates the full pipeline. Calls `plan()` once, then `execute()` in a loop, then prints the formatted results. Returns the list of results so that other scripts can import and use this function.

---

## Design Decisions

**One file, no classes**

The project uses plain functions instead of a class hierarchy. Classes would add abstraction without adding clarity for a project of this size. A single file means a beginner can read the whole codebase in five minutes.

**Stateless execution**

Each `execute()` call is independent. The executor does not know the goal or what previous steps produced. This makes each call simple and predictable, but limits the quality of results for complex tasks that depend on prior context.

**Two system prompts, same model**

Using the same model with two different system prompts demonstrates a fundamental concept : AI behavior is shaped by context, not by the model alone. The same Claude that plans in one call executes in the next.

---

## Possible Extensions

These are natural next steps for someone who wants to go further :

| Extension | What it adds | Complexity |
|---|---|---|
| Memory | Pass previous results as context to each `execute()` call | Low |
| Structured outputs | Use JSON instead of plain text for the plan | Low |
| CLI arguments | Accept the goal as a command-line argument with argparse | Low |
| File output | Save results to a Markdown file | Low |
| Tool use | Let the executor call external APIs (weather, search, etc.) | Medium |
| Error recovery | Retry a step if Claude returns an empty or malformed response | Medium |
| Multiple providers | Support OpenAI or Ollama via environment variable | Medium |
| Web interface | Expose the pipeline as a REST API with FastAPI | High |
