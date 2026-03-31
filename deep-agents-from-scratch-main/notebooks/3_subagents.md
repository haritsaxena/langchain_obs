# Subagents — chat notes (`notebooks/2_subagents.md`)

Consolidated notes from the discussion around `3_subagents.ipynb`, `task_tool.py`, tracing, and message display.

---

## 1. Renaming the `task` tool function

**Question:** Can the inner method named `task` be renamed?

**Answer:** Yes. LangChain’s `@tool` usually exposes the **Python function name** as the tool name the model must call. If you rename `def task` → e.g. `def delegate_to_subagent`, update the `return` and any prompts that say `task(...)`.

To keep the **external** tool name as `task` while using a different Python name, pass an explicit name to `@tool` (see LangChain docs for your version’s signature).

---

## 2. Why one `web_search` but four `gpt-4o-mini` calls?

**Observation:** Traces show `web_search` once but the chat model four times.

**Answer:** Tool calls are not LLM calls. A typical delegation looks like:

| Step | Who | What |
|------|-----|------|
| 1 | Supervisor | Model decides to call `task(...)` |
| 2 | Subagent | Model decides to call `web_search` |
| — | — | `web_search` runs **once** |
| 3 | Subagent | Model reads tool result and writes the final answer |
| 4 | Supervisor | Model sees `task` result and finishes (or calls another tool) |

That is **1 + 2 + 1 = 4** model calls and **one** search. The subagent alone often needs **two** LLM steps (choose tool → answer after tool), which is normal for a ReAct-style loop.

---

## 3. LangSmith: supervisor vs subagent in traces

**Goal:** See which agent (supervisor vs subagent) produced each run.

**Implementation (in `task_tool.py` and notebooks):**

- **`create_agent(..., name="supervisor")`** on the top-level agent.
- **`create_agent(..., name=f"subagent:{_agent['name']}")`** for each subagent graph.
- On **`sub_agent.invoke(state, config=...)`**, pass e.g.:
  - `run_name`: `subagent:<subagent_type>`
  - `tags`: `["subagent", subagent_type]`
  - `metadata`: `agent_role`, `subagent_type`

Optional: on `agent.invoke(...)` for the supervisor, add `tags` / `metadata` / `run_name` for the root run.

---

## 4. `@tool(description=TASK_DESCRIPTION_PREFIX.format(other_agents=other_agents_string))`

**What it does:**

- **`@tool`** registers the following function as a LangChain **tool** the LLM can call.
- **`description=`** is the long text in the **tool schema** the model reads when choosing tools.
- **`TASK_DESCRIPTION_PREFIX`** is a template with a `{other_agents}` placeholder.
- **`.format(other_agents=other_agents_string)`** fills in the list of configured subagents (names + descriptions) so the supervisor always sees up-to-date delegation options.

---

## 5. How is the inner `task` function invoked?

**Context:** `task` is defined inside `_create_task_tool` (closure) and returned.

**Answer:**

1. `@tool` wraps `task` as a **`BaseTool`**; `_create_task_tool` **returns** that tool.
2. The notebook passes it into **`create_agent(..., [task_tool], ...)`**.
3. The **supervisor model** may emit a tool call named `task` with arguments `description` and `subagent_type`.
4. The agent runtime’s **tools step** invokes that tool, supplying **injected** arguments (`state`, `tool_call_id`) that the model does not pass—via `InjectedState` / `InjectedToolCallId`.

The closure captures `agents`, descriptions, etc., so each `_create_task_tool(...)` build stays self-contained.

---

## 6. How are AI messages and tool messages printed in the notebook?

**Answer:** `agent.invoke` only returns state (including `messages`). Printing is **explicit** in the notebook:

```python
from utils import format_messages
result = agent.invoke(...)
format_messages(result["messages"])
```

**`notebooks/utils.py`:**

- **`format_message_content`:** For `AIMessage`, merges text with **tool call** lines (`🔧 Tool Call: ...`, args JSON). For `ToolMessage`, shows `.content`.
- **`format_messages`:** Iterates messages, picks a Rich **Panel** title by message class (`Human`, `Tool`, etc.), and **`console.print(Panel(...))`** so Jupyter displays the bordered output.

There is also **`stream_agent`** in `utils.py` for incremental printing via `astream`.

---

## Related files

| File | Role |
|------|------|
| `src/deep_agents_from_scratch/task_tool.py` | `_create_task_tool`, subagent `create_agent`, `task` tool, tracing `config` on invoke |
| `src/deep_agents_from_scratch/prompts.py` | `TASK_DESCRIPTION_PREFIX`, `SUBAGENT_USAGE_INSTRUCTIONS` |
| `notebooks/utils.py` | `format_messages`, `format_message_content`, `stream_agent` |
| `notebooks/3_subagents.ipynb` | Supervisor demo, `%%writefile` to `task_tool.py` |

---

*Generated as a consolidated chat history for subagents topics.*
