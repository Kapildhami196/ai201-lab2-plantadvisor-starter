# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an *agent* rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's current message |
| `history` | `list` | Gradio conversation history — list of `[user_msg, assistant_msg]` pairs |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. Gradio history is a list of `[user, assistant]`
pairs — convert each pair to two API-format dicts:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for user_msg, assistant_msg in history:
    messages.append({"role": "user", "content": user_msg})
    if assistant_msg:
        messages.append({"role": "assistant", "content": assistant_msg})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

*The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case.*

```
[### Loop termination conditions

The loop should stop in two cases:

1. If the LLM response has no `tool_calls`, that means the model has enough information and has produced the final answer. I detect this with:

```python
if not assistant_message.tool_calls:
    return assistant_message.content or ""
```

2. If the loop reaches `MAX_TOOL_ROUNDS`, the agent should stop to prevent an infinite tool-calling loop. In that case, return a user-readable fallback message such as:

```python
return "I wasn't able to fully process your request. Please try again."
```

This protects the app from getting stuck if the model keeps requesting tools repeatedly.

````

### Extracting the final text response

When the LLM stops requesting tools, the final text answer is stored in the assistant message's `content` field.

```python
assistant_message = response.choices[0].message
return assistant_message.content or ""
````

The `response.choices[0].message` object contains the assistant's final message. The `content` field is the string that should be returned to Gradio and shown to the user.

````

## Implementation Notes

**Trace of a working agent turn (what tools were called and in what order):**

```text
Query: "How should I care for my snake plant in winter?"
Round 1 tool call: lookup_plant({"plant_name": "snake plant"})
Round 1 tool call: get_seasonal_conditions({"season": "winter"})
Final response: The agent combined snake plant care data with winter seasonal advice. It recommended watering less often, avoiding cold drafts, keeping the plant in indirect light, and reducing winter care intensity.
````

**What happens when you ask about a plant that isn't in the database?**

```text
When I asked about bird of paradise, the lookup tool could not find it in the local plant database. The agent clearly said it did not have specific care data for that plant, then gave general care guidance instead of pretending the plant existed in the database.
```

**One thing about the tool call API that surprised you:**

```text
One thing that surprised me is that the assistant tool-call message must be appended to the messages list before adding the tool result messages. The tool result also needs a matching tool_call_id so the API can connect the result back to the exact tool call that requested it.
```
]
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
[your answer here]
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: [tool name, args]
Round 2 tool call: [tool name, args] (if any)
Final response: [brief description]
```

**What happens when you ask about a plant that isn't in the database?**

```
[describe the behavior you observed]
```

**One thing about the tool call API that surprised you:**

```
[your answer here]
```
