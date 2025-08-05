---
title: "The Full MCP Blueprint: Part C - Building a Custom MCP Client from Scratch"
subtitle: "Model context protocol crash course—Part 3"
authors:
  - name: Avi Chawla
  - name: Akshay Pachaar
date: "June 8, 2025"
source: "Daily Dose of Data Science"
---

# The Full MCP Blueprint: Part C - Building a Custom MCP Client from Scratch

_Model context protocol crash course—Part 3._

**Authors:** Avi Chawla, Akshay Pachaar  
**Date:** June 8, 2025  
**Source:** Daily Dose of Data Science

---

## Table of Contents

```{contents}
:depth: 3
```

---

## Recap

Before we dive further into this MCP course, let's take a moment to reflect on what we have learned so far.

In part 1, we laid the conceptual foundation of the model context protocol (MCP). We discussed the core problem MCP solves: the M×N integration issue, where each new model-tool pairing would previously require custom glue code. MCP breaks that pattern with a unified, standardized interface.

We also thoroughly explored the MCP architecture: the Host, the Client, and the Server. This modular design lets AI assistants plug into multiple tools and data sources without manual rework.

In part 2, we got to know about capabilities, like tools, resources, and prompts, and also saw how AI models dynamically use these building blocks to reason, fetch data, perform actions, and adapt on the fly.

We understood how the client-server capability exchange works, and why MCP is a game-changer for dynamic tool integration, through hands-on examples.

```{admonition} Prerequisites
:class: note
If you haven't yet gone through the previous parts, we strongly recommend doing that first before you read further. It'll give you the necessary grounding to fully benefit from what's coming next. Find them here:

- [The Full MCP Blueprint: Part A - Foundations & Architecture](../part-a/mcp-blueprint-part-a.md)
- [The Full MCP Blueprint: Part B - Capabilities & Protocols](../part-b/mcp-blueprint-part-b.md)
```

In this part, we'll shift our focus toward a more practical implementation and bring clarity to several key ideas and concepts covered so far in the first two parts.

By the end of this part, we will have a concrete understanding of:

- How to build a custom MCP client, and not rely on prebuilt solutions like Cursor or Claude.
- What the full MCP lifecycle looks like in action.
- The true nature of MCP as a client-server architecture, as revealed through practical integration.
- How MCP differs from traditional API and function calling, illustrated through hands-on implementations.

As always, everything will be assisted with intuitive examples and code.

Let's begin!

---

## Building an MCP Client

To truly understand how the model context protocol (MCP) works, and to integrate it into our own applications, we're now going to build a custom MCP client.

To recap, an MCP Client is a component within the Host that handles the low-level communication with an MCP Server.

Think of the Client as the adapter or messenger. While the Host decides what to do, the Client knows how to speak MCP to actually carry out those instructions with the server.

Each MCP Client manages a 1:1 connection to a single MCP Server. If your Host app connects to multiple servers, it will instantiate multiple Client instances, one per server. The Client takes care of tasks like:

- sending the appropriate MCP requests
- listening for responses or notifications
- handling errors or timeouts
- ensuring the protocol rules are followed

It's essentially the MCP protocol driver inside your app.

```{admonition} Useful Analogy
:class: tip
The Client role may sound a bit abstract, but a useful way to picture it is to compare it to a web browser's networking layer. The browser (host) wants to fetch data from various websites; it creates HTTP client connections behind the scenes to communicate with each website's server.

In many frameworks, this client functionality is provided by an SDK or library so you don't have to implement stuff from scratch.
```

In our process of implementation, this client will:

1. Connect to an MCP server
2. Forward user queries to an LLM
3. Handle tool calls made by the model
4. Return the final response to the user

We'll be using Python for our implementation, and we'll walk through each step in detail.

```{admonition} Building on Previous Knowledge
:class: tip
If you've completed Part 2, you're already familiar with the setup process and have seen how an MCP server is implemented. We'll build on that foundation here.
```

To implement a basic mockup MCP client (without involving an LLM yet), we'll need to install a few core dependencies:

```bash
uv add fastmcp mcp
```

To keep things consistent, we'll use the same sample server code from the previous part for our demonstration.

Here's the server setup we'll be working with (create a `server.py` file and add this code):

```python
from fastmcp import FastMCP

# Create MCP server
mcp = FastMCP("Sample Server")

@mcp.tool()
def get_weather(location: str) -> str:
    """Get the current weather for a given location.

    Args:
        location: The location to get weather for

    Returns:
        Weather information as a string
    """
    # Mock weather data - in reality, you'd call a weather API
    return f"Weather in {location}: Sunny, 72°F"

@mcp.tool()
def calculate(expression: str) -> float:
    """Calculate the result of a mathematical expression.

    Args:
        expression: Mathematical expression to evaluate

    Returns:
        The result of the calculation
    """
    try:
        # CAUTION: eval() is dangerous in production
        result = eval(expression)
        return float(result)
    except Exception as e:
        return f"Error: {str(e)}"

@mcp.tool()
def convert_currency(amount: float, from_currency: str, to_currency: str) -> str:
    """Convert currency from one type to another.

    Args:
        amount: Amount to convert
        from_currency: Source currency code (e.g., 'USD')
        to_currency: Target currency code (e.g., 'EUR')

    Returns:
        Converted amount as a string
    """
    # Mock exchange rates
    exchange_rates = {
        "USD": {"EUR": 0.85, "GBP": 0.73, "INR": 83.0},
        "EUR": {"USD": 1.18, "GBP": 0.86, "INR": 97.6},
        "GBP": {"USD": 1.37, "EUR": 1.16, "INR": 113.5}
    }

    if from_currency == to_currency:
        return f"{amount} {from_currency}"

    if from_currency in exchange_rates and to_currency in exchange_rates[from_currency]:
        rate = exchange_rates[from_currency][to_currency]
        converted = amount * rate
        return f"{amount} {from_currency} = {converted:.2f} {to_currency}"
    else:
        return f"Exchange rate not available for {from_currency} to {to_currency}"

if __name__ == "__main__":
    mcp.run()  # stdio transport
```

In the above code, we have created an MCP server with three tools:

1. **get_weather** - which, for the sake of simplicity, just returns a dummy response but we can make it practical by integrating a weather API.
2. **calculate** - which calculates the result of a mathematical expression.
3. **convert_currency** - which converts a given amount from one currency to another.

To run this server under SSE transport, simply change the `mcp.run()` line to:

```python
mcp.run(transport="sse", host="127.0.0.1", port=8000)
```

This transport is used for remote or long-running services, often over a network or cloud.

```{admonition} Download Resources
:class: note
On a side note, you can download the code for this whole article along with a full readme notebook with step-by-step instructions. It contains a few Python files and a notebook file.
```

Once the dependencies are installed and the server has been defined, go ahead and create a new file named `client_stdio.py` in the same directory as that of `server.py`.

This will be our minimal client implementation that interacts with the MCP server over a standard input/output (stdio) stream. Add the following code inside:

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    # Server parameters for launching our MCP server
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize the connection
            await session.initialize()

            # List available tools
            tools_result = await session.list_tools()
            print("Available tools:")
            for tool in tools_result.tools:
                print(f"- {tool.name}: {tool.description}")

            # Call a specific tool
            result = await session.call_tool("calculate", {"expression": "2+3"})
            print(f"\nTool result: {result}")

if __name__ == "__main__":
    asyncio.run(main())
```

This code uses asyncio for asynchronous programming.

```{admonition} About Asynchronous Programming
:class: tip
Asynchronous programming lets a program do multiple things at once without waiting for slow tasks to finish first. Instead of stopping completely, it starts the slow task, keeps working on other tasks, and later comes back when the slow task is done. This makes programs faster and more responsive.
```

The entire client logic is wrapped in the async function: `async def main()`, to enable non-blocking operations allowing the program to handle multiple tasks concurrently.

First, we start with some basic import statements:

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
```

Next, we define our server parameters:

```python
server_params = StdioServerParameters(
    command="python",
    args=["server.py"]
)
```

This creates a configuration for launching our MCP server. The executable to run (Python interpreter) is provided by `command="python"`, and `args=["server.py"]` provides the arguments passed to the command (the server script).

When executed, this effectively runs `python server.py` to start the MCP server.

Once the configuration is set, we establish the connection to the server via stdio and initialize a session, using nested context managers.

```python
async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
```

The first line of code starts the server using the server parameters and sets up an asynchronous connection through standard input/output streams. `stdio_client()` manages the communication.

The second line establishes a session between the client and the server over input/output streams using `ClientSession()`. It manages the MCP protocol communication over those streams.

```{admonition} Context Manager Benefits
:class: tip
The nested context managers ensure proper cleanup when the connection ends.
```

Then, after that comes the protocol initialization:

```python
await session.initialize()
```

This sends a setup request to the server performing the MCP handshake, where exchanging initial request, response and a notification happens. This establishes the protocol connection between client and server.

The client queries the server for available tools and prints them:

```python
tools_result = await session.list_tools()
print("Available tools:")
for tool in tools_result.tools:
    print(f"- {tool.name}: {tool.description}")
```

The above code iterates through the returned tools, printing each tool's details. Each tool has a name (or the tool identifier) and a description (or what the tool does). The description is nothing else but the docstrings.

Beyond name and description, there are several other things. Have a look at the code below, as used in the mcp package:

```python
from mcp.types import Tool

# A tool object contains:
# - name: str
# - description: str
# - inputSchema: dict (JSON schema for parameters)
```

Moving further, we can invoke and perform tool execution:

```python
result = await session.call_tool("calculate", {"expression": "2+3"})
print(f"\nTool result: {result}")
```

In a realistic scenario, which tool will be invoked is decided by the LLM. Think of it this way. When we list all the available tools:

1. The LLM knows the query, which in the above code, is "2+3".
2. The LLM also knows all the available tools (we just listed them).
3. Then the LLM decides which tool would be the best to invoke.

We'll get into it shortly, but to keep things simple for now, we have hard-coded that with a specific query in the demonstration above.

Moving on, once everything is set up and in place, we can finally launch the client asynchronously by adding the following at the end:

```python
if __name__ == "__main__":
    asyncio.run(main())
```

Now our client is ready to use. To launch the client, simply run the following command in the terminal/command line:

```bash
uv run client_stdio.py
```

You can also run this as follows:

```bash
python client_stdio.py
```

**Check the output below:**

```
Available tools:
- get_weather: Get the current weather for a given location.
- calculate: Calculate the result of a mathematical expression.
- convert_currency: Convert currency from one type to another.

Tool result: [CallToolResult(content=[TextContent(type='text', text='5.0')], isError=False)]
```

From this output, we can see that the client performs dynamic discovery, a key feature of the MCP architecture and design.

So overall, when we see it, the picture looks like:

1. **Launch**: Client starts and configures server parameters
2. **Connect**: The Client launches the server process and establishes communication
3. **Initialize**: MCP handshake occurs
4. **Discover**: Client queries available tools
5. **Execute**: Client calls a specific tool with arguments (the LLM is used to decide which tool would be the best)
6. **Cleanup**: Context managers handle connection teardown

Upon keen observation, we can see that this flow is nothing but the MCP lifecycle in action with the initialization, discovery, execution, and termination phases.

```{admonition} MCP Lifecycle Reference
:class: note
The MCP interaction lifecycle has already been covered in part 2 of the course. If you need a refresher, find it [here](../part-b/mcp-blueprint-part-b.md#interaction-lifecycle).
```

Coming back to our discussion, here, if we want to use SSE mechanism, we need to remove stdio imports and add SSE imports, change server connection to SSE and specify the URL where our server is active.

Here's a complete code that demonstrates this in action:

```python
import asyncio
from mcp import ClientSession
from mcp.client.sse import sse_client

async def main():
    async with sse_client("http://localhost:8000/sse") as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize the connection
            await session.initialize()

            # List available tools
            tools_result = await session.list_tools()
            print("Available tools:")
            for tool in tools_result.tools:
                print(f"- {tool.name}: {tool.description}")

            # Call a specific tool
            result = await session.call_tool("calculate", {"expression": "2+3"})
            print(f"\nTool result: {result}")

if __name__ == "__main__":
    asyncio.run(main())
```

```{admonition} SSE Setup Requirements
:class: warning
In case of SSE, before we launch our client, we need to start our server by executing the server script in a separate terminal, using `uv run server.py`. Also, under the main method of `server.py` file, ensure to run this server as SSE as discussed earlier (and shown in the code below). This makes our server active and available at the specified URL.
```

To run the server under SSE transport, change the `mcp.run()` line to:

```python
mcp.run(transport="sse", host="127.0.0.1", port=8000)
```

Next, we run the server:

```bash
uv run server.py
```

To launch the client, simply run `uv run client_sse.py` in a terminal, similar to how we did for the stdio based client.

You can also run this as follows:

```bash
python client_sse.py
```

The output will be similar to the stdio version, demonstrating that both transport mechanisms work identically from the client's perspective.

---

## Integrating LLM into the Client

Now that we understand the basic implementation of the MCP client, let's take it a step further by reimplementing and integrating it with the core component, which is the large language model (LLM).

This gives our system a "brain" of its own, enabling it to interpret user input, decide when to invoke tools, and respond intelligently, all while remaining fully decoupled from the server logic.

### Pipeline and Flow in an MCP-enabled Application

To get started, install the necessary packages into the existing project with the following command:

```bash
uv add litellm python-dotenv
```

We use LiteLLM because it provides a unified interface to interact with multiple LLM providers, making it easy to switch between models with minimal code changes.

For this demonstration, we'll use GPT-4o from OpenAI, but you're free to choose any other LLM that supports tool calling and is compatible with LiteLLM.

After this, create a `.env` file in your directory and store your OpenAI API key here:

```env
OPENAI_API_KEY=your_openai_api_key_here
```

Next up, we write the new client code.

To begin, create a new file `client_stdio_with_llm.py` we add the necessary imports and load our API key from the `.env` file using the dotenv package. This sets up the environment for securely accessing the LLM. Here's how to get started:

```python
import asyncio
import os
from contextlib import AsyncExitStack
from dotenv import load_dotenv
import litellm
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

# Load environment variables
load_dotenv()

# Global variables for session management
session = None
exit_stack = AsyncExitStack()
```

In the above code:

**AsyncExitStack** (from contextlib) is a helper that allows us to open multiple asynchronous context managers and then close them all in one call. We'll use it to keep track of both the stdio_client stream (the transport connecting us to MCP), and the ClientSession itself. When we call `exit_stack.aclose()`, everything is torn down in reverse order.

Moving forward, we define a few global variables to help with managing session state throughout:

```python
# Global variables for session management
session = None
exit_stack = AsyncExitStack()
```

In the above code:

- **session** (initially None): Stores the MCP client session for tool communication. Once we connect to the MCP server via `connect_to_server()`, we'll assign a live ClientSession object here.
- **exit_stack**: Manages the lifecycle of async resources.

Moving ahead, we define a function to connect to the server:

```python
async def connect_to_server(server_script: str):
    """Connect to MCP server and initialize session"""
    global session, exit_stack

    # Server parameters
    server_params = StdioServerParameters(
        command="python", args=[server_script]
    )

    # Create stdio transport
    stdio_transport = await exit_stack.enter_async_context(
        stdio_client(server_params)
    )
    read, write = stdio_transport

    # Create and initialize session
    session = await exit_stack.enter_async_context(
        ClientSession(read, write)
    )
    await session.initialize()
```

So now let's break down what exactly is happening here:

1. In line 6, with `server_params = StdioServerParameters(...)`, we implement the idea that when I create a stdio_client, it should run the command `python server.py` inside a subprocess. This subprocess is expected to run the MCP server and the file contains the implementation of the server.

2. In line 8, we actually launch the subprocess under the hood. The `stdio_client(...)` co-routine yields a pair `(read_stream, write_stream)` connected to that subprocess's stdin/stdout. We saw that earlier. We store that pair in `stdio_transport`.

3. In line 10, we unpack `stdio, write = stdio_transport`. `stdio` is a reader interface (what we read from, i.e., what the MCP server prints out) and `write` is a writer interface (what we write into, i.e. what the MCP server receives).

4. Next, in line 12, we wrap the raw stdio streams into an `mcp.ClientSession`. Note that we register this `ClientSession(...)` with the same exit_stack. This way, when we eventually call `await exit_stack.aclose()`, it will close the session and then kill the underlying subprocess.

5. In line 14, as soon as everything is setup, we can complete the handshake (the MCP protocol requires an "initialize" RPC call right after opening.

Next, we perform tool discovery and format conversion to inspect the server's capabilities and present them in a compatible format:

```python
async def get_mcp_tools():
    """Get available tools from MCP server in LiteLLM format"""
    tools_result = await session.list_tools()

    # Print available tools
    print("Available tools:")
    for tool in tools_result.tools:
        print(f"- {tool.name}: {tool.description}")

    # Convert to LiteLLM format
    def convert_to_openai_tool(tool):
        return {
            "type": "function",
            "function": {
                "name": tool.name,
                "description": tool.description,
                "parameters": tool.inputSchema
            }
        }

    return [convert_to_openai_tool(tool) for tool in tools_result.tools]
```

We begin by asking the server about the tools it offers via `await session.list_tools()` and print them out (similar to what we did earlier).

LiteLLM (and OpenAI's "function-calling" API) expects each tool/function to be provided in a specific JSON format. Concretely, here we have specified that format by implementing a simple helper.

Here's what this helper does:

- For each tool in `tools_result.tools`, we create a new dictionary with keys `"type": "function"` and a nested `"function": {...}` object.
- We copy over `tool.name` as the function's "name", `tool.description` as the function's "description", and the `tool.inputSchema` as the function's "parameters".
- We return a list of these formatted dictionaries.

Now comes the heart of the MCP workflow, where we must define the central query processing logic. This part receives a user query, sends it to the LLM, observes any tool calls, asks for permission, invokes the corresponding server tool, and returns the final answer.

Have a look at the code below:

```python
async def process_query(query: str) -> str:
    """Process user query with LLM and MCP tools"""

    # Phase 1: Tool decision (first pass)
    tools = await get_mcp_tools()

    # Send query to LLM with available tools
    response = await litellm.acompletion(
        model="gpt-4o",
        messages=[{"role": "user", "content": query}],
        tools=tools,
        tool_choice="auto"
    )

    # Phase 2: Handle tool calls
    messages = [
        {"role": "user", "content": query},
        response.choices[0].message.dict()
    ]

    # Check if model wants to use tools
    assistant_message = response.choices[0].message
    calls = getattr(assistant_message, 'tool_calls', None) or getattr(assistant_message, 'function_call', None)

    if calls:
        if isinstance(calls, list):
            calls_list = calls
        else:
            calls_list = [calls]

        # Process each tool call
        for call in calls_list:
            tool_name = call.function.name
            tool_args = eval(call.function.arguments)  # In production, use json.loads

            # Ask for user permission
            print(f"\n🤖 AI wants to call tool '{tool_name}' with arguments: {tool_args}")
            permission = input("Allow this tool call? (y/n): ").lower().strip()

            if permission == 'y':
                # Execute the tool
                try:
                    result = await session.call_tool(tool_name, tool_args)
                    tool_response = str(result.content[0].text) if result.content else "No response"

                    # Add tool result to conversation
                    messages.append({
                        "role": "tool",
                        "tool_call_id": call.id,
                        "content": tool_response
                    })
                    print(f"✅ Tool executed successfully: {tool_response}")
                except Exception as e:
                    messages.append({
                        "role": "tool",
                        "tool_call_id": call.id,
                        "content": f"Error: {str(e)}"
                    })
                    print(f"❌ Tool execution failed: {str(e)}")
            else:
                # Tool call denied
                messages.append({
                    "role": "tool",
                    "tool_call_id": call.id,
                    "content": "Tool call was denied by user."
                })
                print("🚫 Tool call denied")

        # Phase 3: Final response (second pass)
        final_response = await litellm.acompletion(
            model="gpt-4o",
            messages=messages,
            tool_choice="none"  # Force text response
        )

        return final_response.choices[0].message.content
    else:
        # No tools needed
        return assistant_message.content
```

Given the complexity of the code, let's break it into three phases.

**In the first phase, we perform tool decision (first pass):**

```python
# Phase 1: Tool decision (first pass)
tools = await get_mcp_tools()

# Send query to LLM with available tools
response = await litellm.acompletion(
    model="gpt-4o",
    messages=[{"role": "user", "content": query}],
    tools=tools,
    tool_choice="auto"
)
```

This section of code:

- Gets all the tools using the `get_mcp_tools` method defined earlier.
- Sends the user query to the language model while providing all the available tools as options.
- Uses `tool_choice="auto"` to let the model decide if tools are even needed to answer the query.

**Then we move to the second phase.**

We gather the response of the LLM and create a messages object:

```python
# Phase 2: Handle tool calls
messages = [
    {"role": "user", "content": query},
    response.choices[0].message.dict()
]
```

Then we check if the LLM wanted to invoke any tools:

```python
# Check if model wants to use tools
assistant_message = response.choices[0].message
calls = getattr(assistant_message, 'tool_calls', None) or getattr(assistant_message, 'function_call', None)

if calls:
```

If tools are required, the invocation logic goes inside the for-loop declared above (`for call in calls_list`):

```{figure} /content/resources/images/mcp-flow-diagram.png
:name: mcp-flow-diagram
:alt: MCP Flow Diagram
:align: center

Next up, recall from the MCP flow above:
```

We have just completed step 3 above and now we need to move to Step 4, which is user's approval. We do that below:

```python
# Ask for user permission
print(f"\n🤖 AI wants to call tool '{tool_name}' with arguments: {tool_args}")
permission = input("Allow this tool call? (y/n): ").lower().strip()
```

If the user permits the tool call, we invoke the tool, get the output and append it to the message history:

```python
if permission == 'y':
    # Execute the tool
    try:
        result = await session.call_tool(tool_name, tool_args)
        tool_response = str(result.content[0].text) if result.content else "No response"

        # Add tool result to conversation
        messages.append({
            "role": "tool",
            "tool_call_id": call.id,
            "content": tool_response
        })
        print(f"✅ Tool executed successfully: {tool_response}")
```

If the user declines the tool call, we append a "tool call denied" message to the message history:

```python
else:
    # Tool call denied
    messages.append({
        "role": "tool",
        "tool_call_id": call.id,
        "content": "Tool call was denied by user."
    })
    print("🚫 Tool call denied")
```

This process runs for all the tools that the LLM wanted to invoke.

To summarize the logic outlined above for the second phase:

1. Extracts the response from the LLM.
2. Detects a tool-call request. We check `calls = assistant_message.tool_calls or assistant_message.function_call`, depending on the LiteLLM version. If the model did not request any tool, both of these will be None, so calls becomes None and we skip into the "no tools needed" branch.
3. Parses tool name and arguments.
4. Asks the user for permission. We ask: "Okay, the LLM wants to call tool_name with these arguments. May I actually run that tool?" This is a "human-in-the-loop" safety check.
5. Calls and executes the tool, if permission granted. Appends a "Tool" message to the history. By giving it `role="tool"`, we signal to LLM that "This piece of text is the output of a tool we just ran."
6. If permission denied, we simply append a dummy "tool" message saying "This tool call was denied." Downstream (in the second pass), LiteLLM will see that the tool result is this denial string, and might respond accordingly.

**Following the invocation of all tools, the LLM performs a second pass in the third phase to synthesize the final response:**

```python
# Phase 3: Final response (second pass)
final_response = await litellm.acompletion(
    model="gpt-4o",
    messages=messages,
    tool_choice="none"  # Force text response
)

return final_response.choices[0].message.content
```

Here we:

- Send the complete conversation (including tool results) back to the model.
- Use `tool_choice="none"` to ensure a final text response, without making any tool calls.

After this whole process has taken place, the function returns the response based on whether tools were invoked or not.

Now that we have our central logic in place let's give a final touch by ensuring that the session closes properly when the script exits:

```python
async def cleanup():
    """Clean up resources"""
    global exit_stack
    if exit_stack:
        await exit_stack.aclose()
```

This code:

- Simply closes everything that was registered in the exit_stack:
  - First, it will close the ClientSession.
  - Then it will kill the underlying subprocess (the MCP server that was launched with `python server.py`).
- Ensures that resources don't linger.

Finally, we code our `main()` function where we test our client-server architecture:

```python
async def main():
    """Main function to test the MCP client with LLM integration"""
    try:
        # Define a query
        query = "What's 15 * 8 + 22? Also convert 100 USD to EUR."

        # Connect to server
        await connect_to_server("server.py")

        # Process query with LLM and tools
        response = await process_query(query)
        print(f"\n🎯 Final response: {response}")

    finally:
        # Clean up
        await cleanup()

if __name__ == "__main__":
    asyncio.run(main())
```

In the above code:

- We defined a query
- Connected to the server while specifying the file which has all the tools available.
- We used the `process_query` method defined above to generate a response.
- Finally, finally closed the connection.

We complete our code with:

```python
if __name__ == "__main__":
    asyncio.run(main())
```

Putting it all together, here is what happens when we run the script:

1. `connect_to_server("server.py")`, launches `python server.py` and completes the MCP handshake.
2. With `process_query(...)`, we make the first LiteLLM call to decide on tools, detect if the tools are needed or not, parse arguments, take permission, call the tools, add tool results to messages, make the second LiteLLM call for final response and return the results.
3. Perform cleanup using `await cleanup()`.

Running the above file (using `uv run client_stdio_with_llm.py` or `python client_stdio_with_llm.py`), we get the following output:

```
Available tools:
- get_weather: Get the current weather for a given location.
- calculate: Calculate the result of a mathematical expression.
- convert_currency: Convert currency from one type to another.

🤖 AI wants to call tool 'calculate' with arguments: {'expression': '15 * 8 + 22'}
Allow this tool call? (y/n): y
✅ Tool executed successfully: 142.0

🤖 AI wants to call tool 'convert_currency' with arguments: {'amount': 100, 'from_currency': 'USD', 'to_currency': 'EUR'}
Allow this tool call? (y/n): y
✅ Tool executed successfully: 100 USD = 85.00 EUR

🎯 Final response: The calculation of 15 * 8 + 22 equals 142. Also, 100 USD converts to 85.00 EUR.
```

```{admonition} Server Configuration Note
:class: warning
Make sure `server.py` has `mcp.run()` instead of `mcp.run(transport="sse", host="127.0.0.1", port=8000)`.
```

Here, we can see the LLM in action, performing reasoning to decide upon multiple tools, then the client asks for user permission for each tool and finally a grounded response is returned based on tool results.

```{admonition} Two-Pass Conversation Workflow
:class: note
On a side note, this pattern, often referred to informally as the "two-pass conversation" workflow, is widely adopted in modern AI pipelines.

In the first pass, the model receives the user query and determines whether it needs to invoke a specific function or tool to generate a proper response. If so, it returns a formal function call rather than a direct answer.

In the second pass, once the tool is executed and its result is available, the model is prompted again, this time with the tool's output, allowing it to generate a final, grounded response.

This two-pass mechanism enables dynamic, tool-augmented intelligence while keeping the interaction modular and interpretable.
```

This above demonstration was for an stdio connection.

**In case of SSE transport,** create a file `client_sse_with_llm.py` and start with some standard import statements:

```python
import asyncio
import os
from dotenv import load_dotenv
import litellm
from mcp import ClientSession
from mcp.client.sse import sse_client

# Load environment variables
load_dotenv()

# Global variables
session = None
```

The `get_mcp_tools` and the `process_query` methods will stay exactly the same.

Next up, we have the following:

```python
async def connect_to_server(url: str):
    """Connect to MCP server via SSE and initialize session"""
    global session

    async with sse_client(url) as (read, write):
        async with ClientSession(read, write) as client_session:
            session = client_session
            await session.initialize()

            # Your main logic here
            await main_logic()

async def main_logic():
    """Main application logic"""
    query = "What's 15 * 8 + 22? Also convert 100 USD to EUR."
    response = await process_query(query)
    print(f"\n🎯 Final response: {response}")

async def main():
    """Main function"""
    await connect_to_server("http://localhost:8000/sse")

if __name__ == "__main__":
    asyncio.run(main())
```

We now switch from using the `stdio_client` to the `sse_client` for establishing a connection.

Unlike the previous setup, we no longer need `AsyncExitStack`. Instead, we rely on nested asynchronous context managers, which, as discussed earlier, also handle proper cleanup automatically.

Apart from this change in transport mechanism and session management, the rest of the code largely remains the same.

```{admonition} Resources and Prompts
:class: note
Note that the implementations presented thus far have focused exclusively on tools. However, similar programmatic principles apply equally to prompts and resources.

The `list_resources()` and `list_prompts()` instance methods can be utilized in a manner analogous to `list_tools()`.
```

---

## Architectural Overview

At its core, MCP operates as a client-server system, where:

- The **server** implements and exposes a set of tools
- The **client** discovers these tools and acts as a controller, coordinating LLMs and deciding when to invoke which tools.

Let's demonstrate via the stdio based server-client implementation, as a reference.

### Here are the roles in the implementation:

**server.py:**

- **Role**: MCP Server
- **Description**: Implements tools using `@mcp.tool()` decorators. Responds to tool calls from clients over transport.

**ClientSession + stdio_client:**

- **Role**: MCP Client
- **Description**: Connects to the server subprocess, initializes the session, lists available tools, and dispatches tool calls.

**litellm.acompletion(...):**

- **Role**: LLM (brain/reasoning)
- **Description**: Acts as the "brain" that decides what tool to call, using natural language and function-calling schema.

**process_query(...):**

- **Role**: Client Orchestration Logic
- **Description**: Implements the tool reasoning loop: sends queries to LLM, takes permission, executes tools if requested, and provides results back to the LLM.

This client-server interaction follows a typical request-response protocol, as discussed above and again shown below:

### Simplified MCP Communication Flow

1. **The client launches the server via subprocess:**

   ```python
   server_params = StdioServerParameters(command="python", args=["server.py"])
   ```

2. **Then wraps the server's I/O in a ClientSession.** After that, `await session.initialize()` completes the handshake.

3. **The client requests for available tools.**

   ```python
   tools_result = await session.list_tools()
   ```

4. **The server responds with a list of available tools,** including their names, descriptions, and input schemas.

5. **When the LLM decides a tool is needed,** the client, post-permission, sends a tool call request to the server using `call_tool` instance method.

6. **The server executes the corresponding tool,** and replies with the result.

All requests and responses occur within a live ClientSession, which is gracefully closed using `await exit_stack.aclose()`.

Note that we register this `ClientSession(...)` with the same exit_stack. That way, when we eventually call `await exit_stack.aclose()`, it will close the session and then kill the underlying subprocess.

Based on the above explanation, we can see that our implementations appropriately fit the definition of a typical client-server architecture.

---

## MCP versus API and Function Calling

As also discussed in the previous part, while APIs and function calling have long served as mechanisms for integrating software systems, MCP introduces a new paradigm tailored for interoperability with tools, data, and prompts.

Here, in this part, we'll concretize the differences on the basis of our implementations of client and server.

Below is a comparative overview of how MCP differs fundamentally from both approaches.

### MCP versus API

**In terms of flexibility:**

- **APIs** offer Rigid contracts (hardcoded parameters).
- **MCPs** enable dynamic discovery and adaptation to server-side changes.

**In terms of change management:**

- With **APIs**, client code breaks on schema changes.
- **MCPs** let the clients auto-adapt.

To elaborate further, traditional APIs require clients to hardcode request formats. Any change in API parameters (e.g., adding a new required field) breaks compatibility unless the client updates its code.

In contrast, MCP clients query the server for available tools and their schemas dynamically, enabling zero-code adaptation.

**After updates to parameters:**

```python
# Traditional API approach
def call_weather_api(location, date):  # Fixed parameters
    # If API adds 'unit' parameter, this breaks
    return requests.get(f"/weather?location={location}&date={date}")

# MCP approach
tools = await session.list_tools()  # Dynamic discovery
weather_tool = next(t for t in tools if t.name == "get_weather")
# Tool schema automatically includes any new parameters
```

### MCP versus Function Calling

**In terms of Integration:**

- **Traditional function calling** is also hardcoded and app-specific.
- **MCPs** are decoupled and protocol-driven.

**In terms of scalability:**

- **Traditional function calling** involves M×N manual mappings.
- **MCPs** offer M+N; which is scalable through dynamic discovery and standardization

**In terms of code maintenance:**

- **Traditional function calling** demands updates per function instance.
- **MCPs** allows tool updates to be automatically propagated to the client.

**In terms of reusability:**

- **Traditional function calling** offer Low reusability since the logic is tightly coupled to app context.
- **MCPs** offer High reusability since tools are abstracted from usage.

Function calling enables LLMs to invoke specific functions based on prompts, but lacks a standard for how those functions are defined, described, or discovered.

MCP solves this by standardizing the lifecycle and interface, enabling seamless, modular integration of evolving tools.

### Demonstrating via Code

Let's solidify the above with a code demonstration.

If we go ahead and modify, replace, add or remove a tool in our `server.py` file, the client-side logic remains untouched.

Without changing a single line of client-side logic, we can query the updated system and receive accurate responses.

The client script is designed to dynamically adapt to the available tools.

To add a new tool, simply annotate it with `@mcp.tool()` in `server.py`, re-execute the server/client programs, and the client's `get_mcp_tools()` method will automatically discover and reflect the updated toolset. Similarly we can modify and remove tools.

For instance, let's add a random new tool called `web_search` (mockup) and modify the `get_weather` tool to additionally accept a time argument:

```python
@mcp.tool()
def web_search(query: str, max_results: int = 5) -> str:
    """Search the web for information about a query.

    Args:
        query: Search query string
        max_results: Maximum number of results to return

    Returns:
        Search results as formatted text
    """
    # Mock search results
    results = [
        f"Result {i+1}: Information about '{query}'"
        for i in range(min(max_results, 3))
    ]
    return f"Search results for '{query}':\n" + "\n".join(results)

@mcp.tool()
def get_weather(location: str, time: str = "current") -> str:
    """Get weather information for a location at a specific time.

    Args:
        location: The location to get weather for
        time: Time period ('current', 'hourly', 'daily')

    Returns:
        Weather information as a string
    """
    return f"Weather in {location} ({time}): Partly cloudy, 72°F"
```

Now update the query, then re-execute the scripts and observe the output. You'll observe that everything works perfectly without any issues.

This is something that MCP has achieved, which APIs and traditional function calling setups lack.

**Check out the output screenshot below:**

```
Available tools:
- get_weather: Get weather information for a location at a specific time.
- calculate: Calculate the result of a mathematical expression.
- convert_currency: Convert currency from one type to another.
- web_search: Search the web for information about a query.

🤖 AI wants to call tool 'web_search' with arguments: {'query': 'latest AI developments', 'max_results': 3}
Allow this tool call? (y/n): y
✅ Tool executed successfully: Search results for 'latest AI developments':
Result 1: Information about 'latest AI developments'
Result 2: Information about 'latest AI developments'
Result 3: Information about 'latest AI developments'

🎯 Final response: Based on the search results, here are the latest AI developments: [results would be processed here]
```

Here we can observe that although we did not change any logic on the client, the client automatically fetches the updated tool definitions and our MCP system is also working perfectly fine.

```{admonition} Dynamic Extensibility Advantage
:class: tip
This dynamic extensibility is a core strength of MCP. It enables modular, declarative integration that decouples system evolution from client logic.

This results in a more maintainable, scalable, and developer-friendly workflow for building robust AI-agent systems.
```

---

## Try Out Yourself

1. **Modify the existing implementations** of clients (both stdio and SSE) to accept user queries dynamically from the terminal using `input()`, instead of hardcoding them.

2. **Implement a chat loop** so that after answering one query, the user can continue asking more without the script exiting. The loop should persist until explicitly terminated by a predefined keyboard key/shortcut.

3. **Add a keyboard shortcut** (e.g., Ctrl + F) that enables the user to refresh the connection and restart the entire MCP lifecycle at any point during script execution, without needing to stop or re-run the script. This feature mirrors Cursor's refresh or restart functionality and provides a seamless way to reinitialize the client, especially when server-side tools or capabilities have been modified. A quick refresh ensures the client is immediately synchronized with the updated server state and reflects the latest available toolset.

4. **(Optional)** While we have used the mcp Python package to implement our client so far, now that we have a solid understanding of how MCP orchestrates interactions, try building a minimal MCP client, supporting only tools and excluding LLM integration, without relying on the mcp package. This exercise will help you appreciate the finer details of the MCP architecture and lifecycle, including the request structure, session handling, and tool invocation logic.

```{admonition} Hint for Optional Exercise
:class: tip
You'll primarily use the `requests` python module for this.
```

---

## Conclusion and Next Steps

With Part 3, we've moved one step ahead and built upon some of the key theoretical concepts by actually implementing them into constructing a fully functional custom MCP client of our own.

This practical walkthrough not only demystified the client-server nature of MCP but also showcased its power as a standardized interface for seamless tool integration.

By practically observing the full lifecycle, from connection initialization to tool discovery and execution, we've seen how MCP enables dynamic and modular AI interactions without the overhead of tightly coupled APIs.

This architecture allows developers to scale, extend, or replace tools effortlessly, making AI systems more adaptable and robust.

This hands-on walkthrough demonstrated the technical simplicity and modularity MCP offers. With decoupled tool management, clean extensibility, and a developer-friendly workflow, MCP significantly reduces integration friction.

If Part 1 introduced MCP and the motivation behind MCP, and Part 2 explored the lifecycle, protocol design and server-side design, then Part 3 solidified things by showing what it takes to build and run a client system that speaks MCP.

Together, these chapters provide a fundamental and comprehensive overview of MCP, not just as a protocol, but as a transformative design pattern for AI systems that need to reason, act, and interface with the world around them.

**Looking ahead, upcoming chapters will dive into:**

- Using prompts and resources alongside tools
- Advanced sampling workflows
- Sandboxing
- Testing and debugging
- Real-world deployments and integrations

As always, thanks for reading!

---

## Discussion

Any questions? Feel free to post them in the comments or connect via chat for private discussions.

```{admonition} Tags
:class: note
Topics: Agents, MCP, MCP Crash Course
```
