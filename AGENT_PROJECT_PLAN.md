# Agentic Bioinformatics Benchmark

## Overview

This project explores whether a large language model (LLM) agent can plan, execute, and evaluate a small multi-step bioinformatics analysis by using a predefined set of computational tools.

The initial testbed reuses components from the existing [`scvi-pca-benchmark`](https://github.com/Hwanseok-Jeong/scvi-pca-benchmark) project, which already contains reproducible workflows for PCA/scVI latent representations, downstream embeddings, and quantitative evaluation.

The goal is **not** to build a general autonomous scientist. The first version should stay deliberately small and interpretable so that agent behavior can be inspected and benchmarked.

---

## What does "giving tools to an agent" mean?

An LLM normally receives text and produces text.

An agentic system extends this by exposing a set of callable functions to the model. The model does not receive the entire source code as text on every turn. Instead, it receives a structured description of the available functions, including:

- tool name
- what the tool does
- required arguments
- argument types

For example, a Python function might be:

```python
def run_pca(n_components: int) -> dict:
    ...
```

The LLM can be told that a tool named `run_pca` exists and accepts an integer called `n_components`.

The interaction then becomes:

```text
User question
    ↓
LLM decides what to do
    ↓
Tool call: run_pca(n_components=50)
    ↓
Python executes the function
    ↓
Result is returned to the LLM
    ↓
LLM interprets the result and chooses the next action
```

Thus, "giving the agent a tool" usually means **wrapping existing code in callable functions and exposing their interfaces to the LLM**.

The existing scVI/PCA analysis code can therefore be reused rather than rewritten from scratch.

---

## Research Question

A first concrete question for this project is:

> Can an LLM agent select and execute an appropriate sequence of analysis tools to compare PCA and scVI representations, and can it adapt its analysis when intermediate results are unexpected?

This gives the project two layers:

1. **Scientific analysis layer**
   - PCA
   - scVI
   - dimensionality reduction
   - quantitative evaluation

2. **Agent layer**
   - planning
   - tool selection
   - parameter selection
   - interpreting intermediate results
   - deciding whether more analysis is needed
   - generating a final conclusion

---

## Minimal Architecture

```text
                         ┌─────────────────────┐
                         │    User question    │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │      LLM Agent      │
                         │                     │
                         │ decide next action  │
                         └──────────┬──────────┘
                                    │
                                    ▼
                    ┌──────────────────────────────┐
                    │       Available Tools        │
                    │                              │
                    │ inspect_dataset()            │
                    │ run_pca(...)                 │
                    │ run_scvi(...)                │
                    │ evaluate_representation(...) │
                    │ compare_representations()    │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                         ┌─────────────────────┐
                         │   Tool execution    │
                         │   (Python code)     │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Observation/result  │
                         └──────────┬──────────┘
                                    │
                                    └──────► back to LLM

The loop ends when the agent decides that enough evidence has been collected to answer the user's question.
```

---

## Phase 1 — Minimal Viable Agent

The first version should use only a small number of tools.

### Proposed tools

#### `inspect_dataset()`

Returns basic information about the dataset.

Example output:

```json
{
  "n_cells": 23822,
  "n_genes": 29713,
  "dataset": "Tasic 2018",
  "available_labels": ["cell_type", "region"]
}
```

---

#### `run_pca(n_components)`

Runs or retrieves a PCA representation.

Example:

```python
run_pca(n_components=50)
```

Possible return value:

```json
{
  "method": "PCA",
  "n_components": 50,
  "output_path": "results/embeddings/pca50.npy"
}
```

---

#### `run_scvi(n_latent)`

Runs or retrieves an scVI latent representation.

Example:

```python
run_scvi(n_latent=20)
```

---

#### `evaluate_representation(method, metric)`

Evaluates a representation or downstream embedding.

Possible metrics from the existing project include:

- KNN preservation
- neighborhood/class purity
- CPD / pairwise-distance correlation

Example:

```python
evaluate_representation(
    method="scvi20_tsne",
    metric="knn"
)
```

---

#### `compare_representations(method_a, method_b)`

Loads existing metrics and returns a concise comparison.

Example output:

```json
{
  "method_a": "pca50_tsne",
  "method_b": "scvi20_tsne",
  "knn": {
    "method_a": 0.383,
    "method_b": 0.424
  },
  "cpd": {
    "method_a": 0.578,
    "method_b": 0.377
  }
}
```

---

## Phase 2 — Basic Agent Loop

The first implementation should avoid a large agent framework so that the underlying mechanism remains understandable.

Conceptually:

```python
messages = [user_question]

while True:

    response = call_llm(
        messages=messages,
        tools=tool_definitions
    )

    if response.requests_tool:
        result = execute_tool(response.tool_call)

        messages.append(
            {
                "role": "tool",
                "content": result
            }
        )

    else:
        return response.final_answer
```

The important idea is:

```text
reason / decide
    ↓
use a tool
    ↓
observe result
    ↓
reason again
    ↓
choose another tool or finish
```

---

## First Test Case

### User task

> Compare PCA and scVI for neighborhood preservation on the Tasic dataset and determine which representation performs better.

A reasonable agent trajectory could be:

```text
1. inspect_dataset()

2. run_pca(n_components=50)

3. run_scvi(n_latent=20)

4. evaluate_representation("pca50", "knn")

5. evaluate_representation("scvi20", "knn")

6. compare_representations("pca50", "scvi20")

7. provide conclusion
```

This task is intentionally simple. The objective is initially to verify that:

- the model understands which tools exist
- arguments are valid
- tools are called in a sensible order
- tool outputs are interpreted correctly
- the final answer is grounded in actual tool outputs

---

## Phase 3 — Make the Workflow More Agentic

Once the basic loop works, the next version can allow the agent to investigate unexpected results.

Example question:

> Compare PCA and scVI and investigate possible reasons if scVI does not improve all metrics.

A possible trajectory becomes:

```text
PCA50
  ↓
scVI20
  ↓
compare metrics
  ↓
unexpected result
  ↓
agent proposes hypothesis:
"Could latent dimensionality explain the difference?"
  ↓
scVI10
scVI30
scVI50
  ↓
evaluate
  ↓
compare
  ↓
final interpretation
```

The important difference is that the additional analyses are **not fully predefined in advance**.

The agent chooses them based on intermediate observations.

---

## Phase 4 — Benchmark the Agent

After the agent works, the interesting research problem becomes:

> How reliably does the agent choose and execute an appropriate workflow?

Create a small benchmark set, for example 10–20 tasks.

Possible tasks:

1. Compare PCA50 with scVI20 using KNN preservation.
2. Find the scVI latent dimensionality with the highest neighborhood preservation.
3. Compare PCA and scVI under t-SNE.
4. Compare PCA and scVI under UMAP.
5. Determine whether conclusions change depending on the evaluation metric.
6. Investigate why one method has better local but worse global structure preservation.
7. Identify redundant analyses.
8. Recover from an invalid tool argument.
9. Decide whether further experiments are required before making a conclusion.
10. Summarize the evidence without claiming that one representation is universally superior.

---

## Possible Evaluation Metrics

### Task success

Did the agent answer the requested scientific question?

### Tool selection accuracy

Did it choose appropriate tools?

### Argument validity

Were function arguments valid and scientifically sensible?

### Grounded conclusion

Does the final answer agree with the actual tool outputs?

### Number of tool calls

How efficiently did the agent solve the task?

### Redundant tool calls

Did it unnecessarily repeat analyses?

### Error recovery

Can it recover when a tool fails or returns an invalid result?

### Reproducibility

Does the same task produce a similar workflow across repeated runs?

---

## Simple Experiment

Compare two agent instructions.

### Agent A — minimal instruction

```text
Solve the user's analysis problem using the available tools.
```

### Agent B — structured instruction

```text
Solve the user's analysis problem using the available tools.

Before choosing each action, identify what information is still required.
Use intermediate tool results to update the analysis plan.
Do not repeat a tool call unless the previous result provides a reason to do so.
Base the final conclusion only on results returned by the tools.
```

Then compare the agents on:

| Metric | Agent A | Agent B |
|---|---:|---:|
| Task success | | |
| Valid tool calls | | |
| Grounded conclusions | | |
| Mean number of calls | | |
| Redundant calls | | |
| Error recovery | | |

This turns the project from a simple LLM demo into a small **agent behavior benchmark**.

---

## Reusing the Existing scVI/PCA Repository

The current repository already separates major analysis stages into scripts such as:

```text
scripts/
├── preprocessing.py
├── latent_model.py
├── embedding.py
├── evaluation.py
└── visualization.py
```

The agent project should avoid modifying the scientific implementation unnecessarily.

Instead, add a thin wrapper layer around existing functions or command-line entry points.

Possible structure:

```text
agentic-bioinformatics-benchmark/
│
├── README.md
├── requirements.txt
│
├── agent/
│   ├── loop.py
│   ├── prompts.py
│   └── schemas.py
│
├── tools/
│   ├── dataset_tools.py
│   ├── latent_tools.py
│   └── evaluation_tools.py
│
├── benchmark/
│   ├── tasks.json
│   ├── evaluator.py
│   └── results/
│
└── tests/
    ├── test_tools.py
    └── test_agent.py
```

Another option is to add an `agent/` directory directly to `scvi-pca-benchmark`.

For an internship portfolio, a separate repository may make the agentic component easier to understand, while still linking explicitly to the original benchmark.

---

## Tool Wrapper Example

Suppose the existing analysis code contains:

```python
from sklearn.decomposition import PCA

def generate_pca(X, n_components):
    model = PCA(n_components=n_components)
    return model.fit_transform(X)
```

The agent-facing wrapper could be:

```python
def run_pca(n_components: int) -> dict:
    if n_components < 2 or n_components > 100:
        raise ValueError("n_components must be between 2 and 100")

    embedding = generate_pca(load_data(), n_components)

    output_path = save_embedding(
        embedding,
        f"pca_{n_components}.npy"
    )

    return {
        "status": "success",
        "method": "PCA",
        "n_components": n_components,
        "output_path": output_path,
    }
```

The LLM does not need to understand all implementation details.

It only needs to know:

```text
Tool: run_pca
Purpose: Generate a PCA latent representation.
Argument:
    n_components: integer
```

---

## Development Plan

### Session 1 — Understand tool calling

Goal:

- create two trivial Python tools
- connect an LLM
- allow it to select one tool
- return the result

Example tools:

```python
get_dataset_info()
get_existing_metrics()
```

Do not run scVI yet.

---

### Session 2 — Connect existing analysis code

Goal:

- wrap PCA
- wrap scVI or cached scVI results
- wrap evaluation
- make one complete analysis task work

At this stage, using **cached results is completely acceptable**.

The point is to learn agent orchestration, not spend every test run retraining scVI.

---

### Session 3 — Add multi-step decisions

Goal:

- let the model call several tools
- return every observation to the model
- stop when the model produces a final answer
- store a log of every action

Example log:

```json
[
  {
    "step": 1,
    "tool": "inspect_dataset",
    "arguments": {}
  },
  {
    "step": 2,
    "tool": "run_pca",
    "arguments": {"n_components": 50}
  }
]
```

---

### Session 4 — Create benchmark tasks

Goal:

- create 10–20 fixed questions
- run the agent on each
- record outputs
- calculate simple metrics

---

### Session 5 — Compare agent strategies

Goal:

- change system instructions or agent structure
- rerun the benchmark
- compare behavior quantitatively

This is the point at which the project becomes especially relevant to an agent R&D internship.

---

## What Not to Do Initially

Avoid adding too much complexity before the basic loop is understood.

Do not initially build:

- multiple collaborating agents
- autonomous web browsing
- vector databases / RAG
- a large frontend
- complex memory systems
- fully autonomous dataset preprocessing
- arbitrary shell access
- dozens of tools
- LangGraph before understanding the basic loop

These can be explored later if they solve a real limitation discovered during experimentation.

---

## Optional Next Steps

After the minimal implementation works:

- compare different LLM models
- compare prompts
- compare direct tool calling with a graph/state-machine architecture
- test repeated-run consistency
- intentionally inject tool failures
- allow the agent to formulate a hypothesis and select a follow-up experiment
- add human approval before expensive analyses
- add a simple UI showing tool execution and evidence
- reimplement a small part in TypeScript
- try an agent framework such as LangGraph after the basic loop is understood

---

## Relevance to BioLizard Internship

This project directly exercises several themes mentioned in the internship description:

- multi-step reasoning loops
- task execution pipelines
- autonomous tool integration
- agent behavior evaluation
- controlled benchmarking
- human-AI collaboration for scientific workflows

The project is intentionally framed as an exploratory learning project rather than as a claim of expertise in agentic AI.

A useful portfolio narrative would be:

> I had experience building reproducible bioinformatics workflows and benchmarking computational methods, but little direct experience with agentic AI. I therefore built a small tool-using scientific agent to understand the architecture concretely, then evaluated where and why its decisions succeeded or failed.

---

## Immediate Milestone

The first milestone is deliberately small:

> **Given a user question, can the LLM correctly choose among 3–5 Python tools, execute them in sequence, use their outputs, and produce a conclusion grounded in those outputs?**

Only after this works should the project expand into hypothesis generation and systematic benchmarking.
