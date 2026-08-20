# Tool Use With Claude — Key Concepts

---

## 1. Why Tools Exist

By default, Claude only knows what it learned during training. That means it has no access to real-time information (current date, live prices, news), cannot interact with external systems (databases, APIs, file systems), and sometimes struggles with precise computations (like date arithmetic 25 years in the future).

Tools solve this. By registering a set of functions Claude can call, you extend its capabilities beyond its training data and give it real agency in the world.

> **Quiz anchor:** *"Claude can only access information from its training data by default. What allows Claude to get current, real-time information?"* — **Using tools to access external information.**

---

## 2. The Tool Use Workflow

The interaction between your app and Claude when tools are involved always follows the same four-step sequence:

```
Initial Request → Tool Request → Data Retrieval → Final Response
```

1. **Initial Request** — Your user sends a message. You forward it to Claude along with the list of available tools.
2. **Tool Request** — Claude decides a tool is needed and responds with a `tool_use` block describing which tool to call and with what arguments.
3. **Data Retrieval** — *Your code* actually runs the function and gets the result. Claude never runs code itself.
4. **Final Response** — You send the tool result back to Claude, and it produces the final answer for the user.

This cycle can repeat multiple times in one conversation — Claude may need to call several tools in sequence before it has everything it needs.

> **Quiz anchor:** *"What is the correct sequence of steps in the tool use workflow?"* — **Initial Request → Tool Request → Data Retrieval → Final Response.**

---

## 3. Defining Tools: The JSON Schema

Claude learns about your tools through **JSON schemas** — structured descriptions you pass alongside the request. Each schema tells Claude:

- The **name** of the tool (used to match the call back to your function)
- A **description** of what the tool does and when to use it (Claude reads this to decide whether to use it)
- The **input_schema**: what arguments the function expects, their types, and which are required

Here is an example from the notebooks — a schema for getting the current date and time:

```python
get_current_datetime_schema = {
    "name": "get_current_datetime",
    "description": "Returns the current date and time formatted according to the specified format string...",
    "input_schema": {
        "type": "object",
        "properties": {
            "date_format": {
                "type": "string",
                "description": "A string specifying the format...",
                "default": "%Y-%m-%d %H:%M:%S",
            }
        },
        "required": [],
    },
}
```

The description is not just documentation — it is how Claude understands *when* to reach for the tool. A vague description produces unreliable tool use. A precise one makes Claude predictable.

> **Quiz anchor:** *"What is the main purpose of a JSON schema when working with Claude tools?"* — **To tell Claude what arguments your function expects and how to use it.**

---

## 4. What Claude Returns When It Wants to Use a Tool

When Claude decides to call a tool, its response is not a plain text string. It is a **multi-block message** that can contain both a text block (optional reasoning) and one or more `tool_use` blocks. A `tool_use` block looks like this:

```python
ToolUseBlock(
    id='toolu_01XfWce6tvAzJnZF1pmdQ3RY',
    name='add_duration_to_datetime',
    input={'datetime_str': '2050-01-01', 'duration': 177, 'unit': 'days'},
    type='tool_use'
)
```

The `id` is critical — you need to reference it when you send the result back, so Claude knows which call is being answered.

> **Quiz anchor:** *"When Claude uses a tool, what type of message structure does it return?"* — **Multi-block messages with text and tool use blocks.**

---

## 5. Detecting When Claude Wants a Tool Call

After each response from Claude, you check the `stop_reason` field of the message object:

- `"end_turn"` → Claude is done. Return the response to the user.
- `"tool_use"` → Claude wants to call a tool. Run it and loop back.

This is the condition that drives the conversation loop. If you miss it, the conversation will simply stop at the wrong point.

```python
def run_conversation(messages):
    while True:
        response = chat(messages, tools=[...])
        add_assistant_message(messages, response)

        if response.stop_reason != "tool_use":  # ← the key check
            break

        tool_results = run_tools(response)
        add_user_message(messages, tool_results)
```

> **Quiz anchor:** *"How can you tell if Claude wants to make another tool call in a conversation?"* — **Look at the `stop_reason` field for `"tool_use"`.**

---

## 6. Running Tools and Sending Results Back

When `stop_reason == "tool_use"`, your code extracts the `tool_use` blocks from Claude's response, runs the corresponding Python functions, and packages the results as `tool_result` blocks:

```python
def run_tools(message):
    tool_requests = [block for block in message.content if block.type == "tool_use"]
    tool_result_blocks = []

    for tool_request in tool_requests:
        try:
            tool_output = run_tool(tool_request.name, tool_request.input)
            tool_result_block = {
                "type": "tool_result",
                "tool_use_id": tool_request.id,   # ← must match the request id
                "content": json.dumps(tool_output),
                "is_error": False,
            }
        except Exception as e:
            tool_result_block = {
                "type": "tool_result",
                "tool_use_id": tool_request.id,
                "content": f"Error: {e}",
                "is_error": True,   # ← signal errors clearly so Claude can handle them
            }

        tool_result_blocks.append(tool_result_block)

    return tool_result_blocks
```

These result blocks are sent back as a **user message**, and then Claude continues the conversation from there.

---

## 7. The Batch Tool Pattern

When a task requires multiple independent tool calls, the naive approach forces Claude to do them one at a time — each requiring its own round-trip through the API. This creates unnecessary latency.

The **batch tool** is a meta-tool you can define that lets Claude pack multiple tool calls into a single request:

```python
batch_tool_schema = {
    "name": "batch_tool",
    "description": "Invoke multiple other tool calls simultaneously",
    "input_schema": {
        "type": "object",
        "properties": {
            "invocations": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string"},
                        "arguments": {"type": "string"},  # JSON-encoded args
                    },
                    "required": ["name", "arguments"],
                },
            }
        },
        "required": ["invocations"],
    },
}
```

Instead of Claude calling `get_current_datetime` then waiting for a response before calling `set_reminder`, it can call both through `batch_tool` in one shot — your code executes them in parallel and returns both results together.

> **Quiz anchor:** *"What problem does the batch tool solve?"* — **It reduces the number of back-and-forth communications when multiple tools are needed.**

---

## 8. Built-in Tools: Text Editor and Web Search

Some tools are **built into the Anthropic API**. You don't need to write a schema for them because Anthropic provides it — but you still need to implement the actual functionality on your side (for the text editor) or accept that Anthropic handles execution (for web search).

### Text Editor Tool

The text editor tool (`str_replace_editor`) gives Claude the ability to create and modify files. The API provides the schema — Claude knows how to call `view`, `str_replace`, `create`, `insert`, and `undo_edit` commands. You register the tool with a type identifier and implement a handler that maps each command to real filesystem operations:

```python
def get_text_edit_schema(model):
    return {
        "type": "text_editor_20250728",
        "name": "str_replace_based_edit_tool",
    }
```

Your `run_tool` function then dispatches to a `TextEditorTool` class that actually reads and writes files.

### Web Search Tool

The web search tool (`web_search_20250305`) lets Claude search the internet in real time. Here both the schema and the execution are handled by Anthropic — you simply include it in your tools list:

```python
web_search_schema = {
    "type": "web_search_20250305",
    "name": "web_search",
    "max_uses": 5,
    "allowed_domains": ["nih.gov"],  # optional domain filtering
}
```

Note that you can restrict which domains Claude is allowed to search, giving you control over information sources.

> **Quiz anchor:** *"What makes Claude's built-in text editor and web search tools different from custom tools?"* — **Claude provides the schema, but you may still need to implement some functionality** (for the text editor, you implement the file operations; for web search, Anthropic handles everything).

---

## 9. Streaming With Tools

For better user experience, you can stream Claude's responses as they arrive instead of waiting for the full reply. This works with tools too, using `client.beta.messages.stream(...)`.

During streaming, you handle different event types:

```python
with chat_stream(messages, tools=tools) as stream:
    for chunk in stream:
        if chunk.type == "text":
            print(chunk.text, end="")   # stream text as it arrives

        if chunk.type == "content_block_start":
            if chunk.content_block.type == "tool_use":
                print(f'\n>>> Tool Call: "{chunk.content_block.name}"')

        if chunk.type == "input_json" and chunk.partial_json:
            print(chunk.partial_json, end="")   # stream the tool arguments too

    response = stream.get_final_message()   # get the full message when done
```

The conversation loop stays identical — you still check `stop_reason` on the final message and run tools the same way.

---

## Summary Table

| Concept | Key Point |
|---|---|
| Why tools exist | Claude is limited to training data; tools give it real-world reach |
| Workflow sequence | Initial Request → Tool Request → Data Retrieval → Final Response |
| JSON Schema | Tells Claude what the tool does, what args it expects, and when to use it |
| Response structure | Multi-block: can contain both text blocks and `tool_use` blocks |
| Detecting tool calls | Check `response.stop_reason == "tool_use"` after every API call |
| Sending results back | Use `tool_result` blocks with the matching `tool_use_id` |
| Batch tool | A meta-tool that lets Claude bundle multiple calls into one round-trip |
| Built-in tools | Anthropic provides the schema; you may still implement the execution logic |
| Streaming | Works the same way — handle extra event types, check `stop_reason` at the end |
