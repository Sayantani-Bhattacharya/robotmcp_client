# robot-mcp-client: Bug Fixes & Improvements

This document describes the bugs found and changes made to get the client working correctly with a live ROS 2 system. It is intended both as a developer reference and as a guide for maintainers reviewing or extending this project.

---

## Summary of Changes

| File | Change | Reason |
|------|--------|--------|
| `clients/baseclient.py` | Pass `env` to `StdioServerParameters` | MCP server couldn't see ROS topics |
| `clients/baseclient.py` | Add bounded conversation history | Agent forgot context between turns; unbounded growth fixed |
| `clients/baseclient.py` | Add system prompt | Agent asked unnecessary clarifying questions |
| `clients/baseclient.py` | Fix structured content response rendering | Raw JSON was printed instead of text |
| `clients/baseclient.py` | Handle `KeyboardInterrupt` cleanly | Ctrl+C printed a full traceback |
| `clients/llm_store.py` | Upgrade Gemini model to `gemini-2.5-flash` | `flash-lite` was too weak for tool use |
| `.env` | Switch provider to `gemini`, then `llama-8b`, then `gemini` | Anthropic had no credits; Groq free tier token limit too low |
| `ros_mcp/main.py` (ros-mcp-server) | Change rosbridge port from `9090` to `9091` | VS Code occupies port 9090 |

---

## Change 1 — Pass ROS environment to the MCP server subprocess

**File:** `clients/baseclient.py`

**What was wrong:**
`StdioServerParameters` spawns the MCP server (`uv run server.py`) as a child process. By default, Python's subprocess inherits the parent's environment — but `uv` may reset or filter it. In practice, the subprocess was not receiving the ROS 2 environment variables (`ROS_DISTRO`, `AMENT_PREFIX_PATH`, etc.), so the MCP server could not discover any ROS topics.

```python
# Before (broken)
self.server_params = StdioServerParameters(
    command="uv",
    args=["--directory", f"{server_root}", "run", "server.py"],
)
```

**Fix:**
Explicitly pass a copy of the current process environment:
```python
# After (correct)
self.server_params = StdioServerParameters(
    command="uv",
    args=["--directory", f"{server_root}", "run", "server.py"],
    env=os.environ.copy(),
)
```

**Result:** The MCP server can now see all ROS 2 topics, services, and nodes that are active in the terminal where the client is launched.

**Note for maintainers:** The client must be launched from a shell where ROS 2 has been sourced (i.e., `source /opt/ros/humble/setup.bash` or equivalent). If you launch from an unsourced shell, topics will still be invisible even with this fix.

---

## Change 2 — Add bounded conversation history across turns

**File:** `clients/baseclient.py`

**What was wrong:**
`serve_query()` was passing only the current user message to the agent on every invocation:

```python
# Before (no memory)
response = await self.agent.ainvoke(
    {"messages": [("user", query)]},
    config={"recursion_limit": 50},
)
```

This meant the agent had no memory of previous exchanges. Every turn started from scratch, so the agent would ask for information the user had already provided (e.g., topic name, message type, duration) multiple times in the same session.

**Fix:**
Added a `self.history` list to `MCPClient` that accumulates the full message thread. Each turn appends the new user message, invokes the agent with the full history, then updates history with the agent's complete response (including any intermediate tool call/result messages):

```python
self.history = []  # initialized in __init__

# In serve_query():
from langchain_core.messages import HumanMessage
self.history.append(HumanMessage(content=query))
response = await self.agent.ainvoke(
    {"messages": self.history},
    config={"recursion_limit": 50},
)
all_messages = response.get("messages", [])
self.history = list(all_messages)  # store raw message objects, not tuples
```

**Why raw message objects matter:**
An earlier attempt stored messages as `(type, content)` tuples. This silently dropped the `tool_call_id` field from `ToolMessage` objects. On the next turn, the agent would see orphaned tool results with no matching call ID and raise a `KeyError: 'tool_call_id'`. Storing the actual LangChain message objects preserves all fields.

**History trimming — why a naive slice is not enough:**

Without trimming, `self.history` grows forever and will eventually exceed the LLM's context window or cause memory issues. The fix caps history at `MAX_HISTORY` messages (default 20, configurable via `.env`). However, slicing blindly by count is dangerous.

After a few turns, the history looks like this:

```
index  type           content
──────────────────────────────────────────────────────
  0    HumanMessage   "move forward 2 seconds"
  1    AIMessage      tool_calls=[{id: "call_abc", name: "publish_to_topic"}]
  2    ToolMessage    tool_call_id="call_abc", content="Published successfully"
  3    AIMessage      "I moved the robot forward"
  4    HumanMessage   "now turn left"
  5    AIMessage      tool_calls=[{id: "call_def", name: "publish_to_topic"}]
  6    ToolMessage    tool_call_id="call_def", content="Published successfully"
  7    AIMessage      "I turned the robot left"
  8    HumanMessage   "stop"
  9    AIMessage      tool_calls=[{id: "call_ghi"}]
 10    ToolMessage    tool_call_id="call_ghi", content="Published successfully"
 11    AIMessage      "Robot stopped"
```

If `MAX_HISTORY=6`, a raw slice of the last 6 gives:

```
  6    ToolMessage    tool_call_id="call_def"   ← ORPHANED — no matching AIMessage above it
  7    AIMessage      "I turned the robot left"
  8    HumanMessage   "stop"
  9    AIMessage      tool_calls=[{id: "call_ghi"}]
 10    ToolMessage    tool_call_id="call_ghi"   ← valid, matched
 11    AIMessage      "Robot stopped"
```

The LLM receives a `ToolMessage` at the top of the conversation referencing `call_def`, but there is no `AIMessage` above it that issued that call. The LLM API rejects this with `KeyError: 'tool_call_id'` because the conversation is structurally broken. Tool calls always come in **pairs**: the `AIMessage` that requests the call and the `ToolMessage` that returns the result. You can never split a pair.

**The fix — snap to the nearest `HumanMessage`:**

```python
trimmed = list(all_messages)[-MAX_HISTORY:]
for i, msg in enumerate(trimmed):
    if isinstance(msg, HumanMessage):
        trimmed = trimmed[i:]
        break
self.history = trimmed
```

After slicing, the code scans forward until it finds a `HumanMessage` and discards everything before it. In the example above, index 6 (`ToolMessage`) — skip. Index 7 (`AIMessage`) — skip. Index 8 (`HumanMessage`) — start here. The resulting history is:

```
  8    HumanMessage   "stop"           ← clean boundary
  9    AIMessage      tool_calls=[{id: "call_ghi"}]
 10    ToolMessage    tool_call_id="call_ghi"   ← matched, valid
 11    AIMessage      "Robot stopped"
```

Every `ToolMessage` now has its issuing `AIMessage` directly above it. The LLM accepts the conversation.

**`MAX_HISTORY` is configurable** via `.env`:
```
MAX_HISTORY=20
```
If absent, defaults to 20. Tune it based on how much context you need versus token cost per request.

**Result:** The agent remembers context within a session, history never grows unbounded, and the message thread is always structurally valid for the LLM API.

---

## Change 3 — Add a system prompt for ROS-aware behaviour

**File:** `clients/baseclient.py`

**What was wrong:**
Without a system prompt, the agent had no domain knowledge. It would ask unnecessary clarifying questions for every command (e.g., "What is the topic name?", "What message type?", "What is the duration?") even for standard ROS movement commands where sensible defaults are obvious.

**Fix:**
A system prompt is passed to `create_agent()` via the `system_prompt` parameter:

```python
system_prompt = (
    "You are a ROS 2 robot operator assistant. "
    "You have access to MCP tools that communicate with a live ROS 2 system via rosbridge. "
    "When the user asks you to move the robot or interact with it, act immediately using the available tools — "
    "do not ask for clarification unless a required parameter is genuinely missing. "
    "For movement commands, use /cmd_vel with geometry_msgs/msg/Twist: set linear.x for forward/backward, "
    "angular.z for turning. Reasonable default values: linear.x=0.2 m/s, angular.z=0.5 rad/s. "
    "Always confirm what action you took after calling a tool."
)
self.agent = create_agent(self.llm, tools, system_prompt=system_prompt)
```

**What the prompt encodes:**
- The agent's role and context (ROS 2 operator via rosbridge)
- A directive to act immediately rather than stall with questions
- Default topic (`/cmd_vel`), message type (`geometry_msgs/msg/Twist`), and velocity values
- A requirement to confirm actions taken

**Result:** Natural language commands like "move forward for 2 seconds" or "turn left" are executed directly without the agent asking follow-up questions.

---

## Change 4 — Fix rendering of structured content responses

**File:** `clients/baseclient.py`

**What was wrong:**
Gemini sometimes returns message content as a list of typed content blocks rather than a plain string:

```python
# Example of what Gemini returns
[{'type': 'text', 'text': 'I moved the robot to the left.', 'extras': {'signature': '...'}}]
```

The original code assumed `.content` was always a string and printed it directly, resulting in the raw list being shown to the user.

**Fix:**
Check the type of the response content and extract text blocks if it is a list:

```python
raw = all_messages[-1].content
if isinstance(raw, list):
    final_answer = " ".join(
        b["text"] for b in raw if isinstance(b, dict) and b.get("type") == "text"
    )
else:
    final_answer = raw
```

**Result:** Responses always display as clean readable text regardless of whether the model returns a string or a structured content block list.

---

## Change 5 — Clean Ctrl+C exit

**File:** `clients/baseclient.py`

**What was wrong:**
Pressing Ctrl+C to exit the client printed a full Python traceback ending in `KeyboardInterrupt`, which was confusing and noisy.

**Fix:**
Wrap `asyncio.run(main())` in a try/except:

```python
if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        pass
```

**Result:** Ctrl+C exits silently after printing "MCP Client closed."

---

## Change 6 — Upgrade Gemini model from `flash-lite` to `flash`

**File:** `clients/llm_store.py`

**What was wrong:**
`gemini-2.5-flash-lite` is a lightweight model not optimised for agentic/tool-use tasks. In practice it failed to decide when to call tools and instead kept asking the user for information the tools themselves could provide.

**Fix:**
```python
# Before
"gemini": "google_genai:gemini-2.5-flash-lite",

# After
"gemini": "google_genai:gemini-2.5-flash",
```

**Result:** The agent reliably decides to call the correct MCP tool and interprets the result without further prompting from the user.

---

## Change 7 — Change rosbridge port to 9091

**File:** `ros_mcp/main.py` (in the `ros-mcp-server` repository)

**What was wrong:**
The MCP server was configured to connect to rosbridge on port `9090` (the default). On this machine, VS Code occupies port 9090, so rosbridge could not bind to it and failed to start with `OSError: [Errno 98] Address already in use`.

**Fix:**
```python
# Before
ROSBRIDGE_PORT = 9090

# After
ROSBRIDGE_PORT = 9091
```

Rosbridge must be launched on the matching port:
```bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml port:=9091
```

**Result:** Rosbridge starts successfully and the MCP server can establish a WebSocket connection to it.

**Note for maintainers:** If you are not running VS Code or your port 9090 is free, you can revert this to `9090` and launch rosbridge without the `port:=` argument.

---

## Runtime Prerequisites

Before running `uv run clients/baseclient.py`, ensure the following are in place:

1. **ROS 2 sourced** in the terminal:
   ```bash
   source /opt/ros/humble/setup.bash
   ```

2. **Rosbridge running** (in a separate terminal, also with ROS sourced):
   ```bash
   ros2 launch rosbridge_server rosbridge_websocket_launch.xml port:=9091
   ```

3. **`.env` configured** with a valid API key matching `LLM_PROVIDER`. Recommended free option:
   ```
   LLM_PROVIDER=gemini
   GOOGLE_API_KEY=<your key from aistudio.google.com>
   ```

4. **`ROS_MCP_SERVER_PATH`** set to the absolute path of the `ros-mcp-server` folder.
