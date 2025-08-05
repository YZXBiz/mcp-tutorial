---
title: "The Full MCP Blueprint: Part 2 - Capabilities, Protocols, and Implementation"
subtitle: "Model context protocol crash course—Part 2"
authors:
  - name: Avi Chawla
  - name: Akshay Pachaar
date: "June 1, 2025"
source: "Daily Dose of Data Science"
---

# The Full MCP Blueprint: Part 2 - Capabilities, Protocols, and Implementation

_Model context protocol crash course—Part 2._

**Authors:** Avi Chawla, Akshay Pachaar  
**Date:** June 1, 2025  
**Source:** Daily Dose of Data Science

---

## Table of Contents

```{contents}
:depth: 3
```

---

## Recap from Part A

Before we dive in, let's briefly recap what we covered in Part 1 of this MCP crash course.

In Part 1, we laid the groundwork for understanding the Model Context Protocol (MCP). We began by exploring the evolution of context management in LLMs, from early techniques like static prompting and retrieval-augmented generation to more structured approaches like function calling and agent-based orchestration.

We then introduced the core motivation behind MCP: solving the M×N integration problem, where every new tool and model pairing previously required custom glue code.

Finally, we unpacked the MCP architecture, clarifying the roles of the Host (AI app), Client (protocol handler), and Server (tool provider) and explained how they interact through a standardized, modular system.

```{mermaid}
graph TB
    subgraph "Host (AI Application)"
        A["LLM/AI Model"]
        B["MCP Client 1"]
        C["MCP Client 2"]
        D["MCP Client 3"]
        A --> B
        A --> C
        A --> D
    end

    subgraph "External Services"
        E["MCP Server 1<br/>(File System)"]
        F["MCP Server 2<br/>(Database)"]
        G["MCP Server 3<br/>(Web API)"]
    end

    B <--> E
    C <--> F
    D <--> G

    subgraph "Capabilities"
        H["Tools"]
        I["Resources"]
        J["Prompts"]
        K["Sampling"]
    end

    E --> H
    F --> I
    G --> J
    G --> K

    style A fill:#ff9999
    style B fill:#ffcc99
    style C fill:#ffcc99
    style D fill:#ffcc99
    style E fill:#99ccff
    style F fill:#99ccff
    style G fill:#99ccff
```

An illustration of MCP's architecture: The Host (AI application) contains an MCP Client component for each connection. Each Client talks to an external MCP Server, which provides certain capabilities (tools, etc.). This modular design lets a single AI app interface with multiple external resources via MCP.

Now, in Part 2, we'll build upon that foundation and get hands-on with MCP capabilities, communication protocols, and real-world implementation.

```{admonition} Prerequisites
:class: note
If you haven't read the previous part yet, we highly recommend doing so before reading ahead: [The Full MCP Blueprint: Part 1](../part-1/mcp-blueprint-part-1.md)
```

Let's dive in!

---

## Capabilities in MCP

An MCP Server can expose one or more capabilities to the client.

Capabilities are essentially the features or functions that the server makes available. The MCP standard currently defines four major categories of capabilities:

- **Tools** → Executable actions or functions that the AI (host/client) can invoke (often with side effects or external API calls).
- **Resources** → Read-only data sources that the AI (host/client) can query for information (no side effects, just retrieval).
- **Prompts** → Predefined prompt templates or workflows that the server can supply.
- **Sampling** → A mechanism through which the server can request the AI (client/host) to perform an LLM operation, typically used to enable more advanced, multi-step interactions. This feature supports capabilities such as self-reflection and iterative reasoning. Sampling will be explored in greater depth in later parts of this course.

Let's break down each in detail:

### Tools (Actions)

Tools are what they sound like: functions that do something on behalf of the AI model. These are typically operations that can have effects or require computation beyond the AI's own capabilities.

```{admonition} Key Point
:class: tip
Tools are usually triggered by the AI model's choice, which means the LLM (via the host) decides to call a tool when it determines it needs that functionality.
```

Because tools can change things or call external services, they often are subject to safety checks (the system may require the user to approve a tool invocation, especially if it's something sensitive like sending an email or executing code).

Suppose we have a simple tool for weather. In an MCP server's code, it might look like:

```python
@mcp.tool()
def get_weather(location: str) -> dict:
    """Get current weather for a location"""
    # This would normally call a real weather API
    return {
        "location": location,
        "temperature": "22°C",
        "conditions": "partly cloudy"
    }
```

When the AI calls `tools/call` with name "get_weather" and `{"location": "San Francisco"}` as arguments, the server will execute `get_weather("San Francisco")` and return the dictionary result.

The client will get that JSON result and make it available to the AI. Notice the tool returns structured data (temperature, conditions), and the AI can then use or verbalize (generate a response) that info.

```{admonition} Safety Considerations
:class: warning
Since tools can do things like file I/O or network calls, an MCP implementation often requires that the user permit a tool call. For example, Claude's client might pop up "The AI wants to use the 'get_weather' tool, allow yes/no?" the first time, to avoid abuse. This ensures the human stays in control of powerful actions.
```

Tools are analogous to "functions" in classic function calling, but under MCP, they are used in a more flexible, dynamic context. They are model-controlled (the model decides when to use them) but developer/governance-approved in execution.

### Resources

Resources provide read-only data to the AI model. These are like databases or knowledge bases that the AI can query to get information, but not modify.

Unlike tools, resources typically do not involve heavy computation or side effects, since they are often just information lookup.

```{admonition} Control Mechanism
:class: note
Another key difference is that resources are usually accessed under the host application's control (not spontaneously by the model). In practice, this might mean the Host knows when to fetch certain context for the model.
```

For instance, if a user says, "Use the company handbook to answer my question," the Host might call a resource that retrieves relevant handbook sections and feeds them to the model.

Resources could include:

- A local file's contents
- A snippet from a knowledge base or documentation
- A database query result (read-only)
- Any static data like configuration info

Essentially anything the AI might need to know as context. An AI research assistant could have resources like "ArXiv papers database," where it can retrieve an abstract or reference when asked.

A simple resource could be a function to read a file:

```python
@mcp.resource("file://{uri}")
def read_file_resource(uri: str) -> str:
    """Read contents of a file"""
    # Implementation would read and return file contents
    with open(uri, 'r') as f:
        return f.read()
```

Notice that resources are usually identified by some identifier (like a URI or name) rather than being free-form functions. They are also often application-controlled, meaning the app decides when to retrieve them (to avoid the model just reading everything arbitrarily).

```{admonition} Security Note
:class: warning
From a safety standpoint, since resources are read-only, they are less dangerous, but still, one must consider privacy and permissions (the AI shouldn't read files it's not supposed to). The Host can regulate which resource URIs it allows the AI to access, or the server might restrict access to certain data.
```

In summary, Resources give the AI knowledge without handing over the keys to change anything. They're the MCP equivalent of giving the model reference material when needed, which acts like a smarter, on-demand retrieval system integrated through the protocol.

### Prompts

Prompts in the MCP context are a special concept: they are predefined prompt templates or conversation flows that can be injected to guide the AI's behavior.

Essentially, a Prompt capability provides a canned set of instructions or an example dialogue that can help steer the model for certain tasks.

**But why have prompts as a capability?**

Think of recurring patterns: e.g., a prompt that sets up the system role as "You are a code reviewer" and the user's code is inserted for analysis. Rather than hardcoding that in the host application, the MCP server can supply it.

```{admonition} Advanced Use Cases
:class: tip
Prompts can also represent multi-turn workflows. For instance, a prompt might define how to conduct a step-by-step diagnostic interview with a user. By exposing this via MCP, any client can retrieve and use these sophisticated prompts on demand.
```

As far as control is concerned, Prompts are usually user-controlled or developer-controlled. The user might pick a prompt/template from a UI (e.g., "Summarize this document" template), which the host then fetches from the server.

The model doesn't spontaneously decide to use prompts the way it does tools. Rather the prompt sets the stage before the model starts generating. In that sense, prompts are often fetched at the beginning of an interaction or when the user chooses a specific "mode".

Suppose we have a prompt template for code review. The MCP server might have:

```python
@mcp.prompt("code_review")
def code_review_prompt() -> list:
    """Provides a code review prompt template"""
    return [
        {
            "role": "system",
            "content": "You are an expert code reviewer. Analyze the provided code for:"
                      "1. Logic errors\n2. Security vulnerabilities\n3. Performance issues\n4. Best practices"
        },
        {
            "role": "user",
            "content": "[CODE TO REVIEW WILL BE INSERTED HERE]"
        }
    ]
```

This prompt function returns a list of message objects (in OpenAI format) that set up a code review scenario. When the host invokes this prompt, it gets those messages and can insert the actual code to be reviewed into the user content.

Common use cases for prompt capabilities include things like "brainstorming guide," "step-by-step problem solver template," or domain-specific system roles. By having them on the server, they can be updated or improved without changing the client app, and different servers can offer different specialized prompts.

```{admonition} Important Insight
:class: note
An important point to note here is that prompts as a capability blur the line between data and instructions. They represent best practices or predefined strategies for the AI to use. In a way, MCP prompts are similar to how ChatGPT plugins can suggest how to format a query, but here it's standardized and discoverable via the protocol.
```

---

## The MCP Communication Protocol

To ensure all these components and capabilities interact smoothly, MCP defines a standard message format and conversation pattern. It uses a tried-and-true foundation called **JSON-RPC 2.0** for message structure.

```{admonition} Why JSON-RPC?
:class: tip
JSON-RPC is a lightweight remote procedure call protocol that encodes calls and responses in JSON format. It's perfect for this scenario because it's simple, human-readable, and language-agnostic.

It provides just what we need: a way to send a request with a method name and parameters, and to get back a response (or an error), all expressed in a standard JSON format. Many existing libraries and tools support JSON-RPC, and it's much simpler than other alternatives. By building on JSON-RPC, MCP didn't have to reinvent the wheel for basic messaging.
```

### Message Types

There are three types of messages in MCP's JSON-RPC-based protocol:

#### Request

Sent by the Client to the Server to ask it to do something (e.g., call a tool, list resources). A Request includes:

- A unique `id` (so we can match the response to this request)
- A `method` name (a string like "tools/list" or "tools/call", and these method names are part of MCP's spec)
- `params` (a JSON object with any parameters needed for that method)

Suppose the client wants the server to execute a tool named "weather". The request might look like this:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "weather",
    "arguments": {
      "location": "San Francisco"
    }
  }
}
```

This asks the server: "Please call the tool named 'weather' with the argument location = San Francisco." The `id: 1` tags this call.

#### Response

Sent by the Server back to the Client as a reply to a Request. A Response includes:

- The same `id` as the corresponding request (to match them)
- EITHER a `result` (if the request succeeded) OR an `error` (if something went wrong)

Here's a sample success response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Weather in San Francisco: 62°F, partly cloudy"
      }
    ]
  }
}
```

This indicates request id 1 (our weather call) succeeded, and here is the result data (62°F and partly cloudy).

Here's what a sample error response could look like:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -1,
    "message": "Unknown location: San Francisco"
  }
}
```

Here, perhaps the tool didn't recognize the location given, so it returns a standard JSON-RPC error with a code and message. The client would see this and handle it (maybe by telling the user the location was invalid).

#### Notification

A message that does not expect a response (thus no `id`). Notifications are typically sent from the Server to Client to convey asynchronous events or updates. For example, progress updates, or a signal that something happened on the server side.

Here's a sample notification:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progress": 0.5,
    "message": "Processing data... 50% complete"
  }
}
```

This could be sent by the server while working on a long task, telling the client "I'm 50% done processing data…". The client doesn't reply to notifications; it may just show this info to the user or log it.

These three message types cover all the needed patterns. The methods (`tools/call`, `tools/list`, `resources/list`, `shutdown`, etc.) are part of MCP's defined API. The combination of Requests, Responses, and Notifications orchestrates the interaction.

### Transport Mechanisms

Now, how do these JSON messages actually travel between client and server? MCP supports two primary transport mechanisms:

#### Stdio (Standard Input/Output)

This transport is used when the Server runs as a local subprocess on the same machine as the Client/Host.

Essentially, when the host launches the server, it connects to the server's stdin and stdout streams. The client writes JSON lines into the server's stdin (Requests) and reads JSON lines from the server's stdout (Responses/Notifications).

This is a simple and effective method for local tools. It doesn't require any network setup or ports, and the operating system ensures isolation (the server can be run with limited permissions, etc.).

Many MCP servers (especially those that act like local plugins, e.g., file system tools) use stdio by default.

#### HTTP + SSE (Server-Sent Events)

This transport is used for remote or long-running services, often over a network or cloud.

It operates via HTTP: the Client sends Requests as HTTP POST requests to the server's URL, and the Server replies using Server-Sent Events (SSE) to stream responses and notifications back over a persistent HTTP connection.

```{admonition} About SSE
:class: tip
SSE is a mechanism that keeps an HTTP response open and allows the server to push multiple messages (it's great for streaming partial results or progress without needing WebSockets). In MCP, the SSE channel is typically used to send any number of notifications and eventually the final response.
```

MCP's design even allows servers to upgrade to SSE when needed (called streamable HTTP). For instance, a server might start sending a normal HTTP response, but if it sees that it will have multiple messages, it can switch to SSE mode mid-connection to keep the stream alive.

### Interaction Lifecycle

We've covered message types and transports; now let's outline the typical lifecycle of an MCP connection from start to finish:

#### The MCP Lifecycle

**1. Initialization Phase**

When a client first connects to a server, there's an initial handshake. The client sends an `initialize` request, which includes information like the protocol version it supports, maybe the client's identity or preferences.

The server responds (response to `initialize`) confirming connection and stating its supported protocol version and maybe server info.

Then the client often sends an `initialized` notification to signal that it's ready. This ensures both sides agree on basics like protocol version.

It's also where a server might send over any "welcome" info or require authentication tokens if needed (depending on implementation).

**2. Discovery Phase**

Next, the client typically inquires about capabilities. It may call methods like `tools/list`, `resources/list`, `prompts/list` sequentially. Each of those is a request that the server answers with a list of what it has.

For example, `tools/list` might return a list of tool names along with descriptions and argument schemas. This allows the Host to know what's available.

```{admonition} Function Signatures
:class: note
To learn the functionality of every available capability, the docstring and function signature is utilized, so that the client LLM knows how exactly everything works and how and when to utilize them. They basically serve as tool description and argument schema.
```

Below is a basic example of how one can obtain function signatures programmatically using the inspect package:

```python
import inspect

def get_function_signature(func):
    """Extract function signature and docstring"""
    signature = inspect.signature(func)
    docstring = inspect.getdoc(func)

    return {
        "name": func.__name__,
        "signature": str(signature),
        "description": docstring or "No description available",
        "parameters": {
            name: {
                "type": str(param.annotation) if param.annotation != param.empty else "Any",
                "default": param.default if param.default != param.empty else None
            }
            for name, param in signature.parameters.items()
        }
    }
```

In some cases, the Host might skip explicit discovery and directly attempt to call a known tool, but general discovery is useful, especially if the host wants to automatically enable the AI to choose tools. The client might cache these results for the session.

**3. Execution Phase**

Now the client (on behalf of the AI's reasoning) makes use of the capabilities. It sends specific requests like `tools/call` (to invoke a tool), or if using resources, maybe `resources/read` with a resource identifier, etc., depending on the action. The server executes and returns responses accordingly.

During this phase, the server might also send notification messages back (e.g., progress updates as shown earlier, or other events). The client continues to listen and process those (for example, updating a progress bar in the UI).

If the AI triggers multiple actions in a sequence, this could be a series of calls. For long-running actions, the SSE transport shines here: the server can send multiple progress notifications (like every 10% completion) and then a final result.

**4. Termination Phase**

When the session or usage is over, the client will close things down gracefully. This typically involves a `shutdown` request from client to server, which the server responds to acknowledging.

Then the client may send an `exit` notification as a final signal, and then actually terminate the connection or subprocess. Proper termination ensures no processes are left hanging and resources (like file handles or network sockets) are cleaned up.

If the user simply closes the application or aborts, the Host should try to perform this shutdown handshake, but if it can't, most servers will also handle sudden disconnections gracefully.

```{admonition} Protocol Design Benefits
:class: tip
Overall, throughout this lifecycle, MCP's protocol is designed to be extensible and robust. The initialization's version negotiation means as MCP evolves, older clients/servers can still talk if they both agree on a common version.

The discovery mechanism means a smart client can adapt to different servers (some might have advanced tools, some only basic ones). The request/response pattern ensures synchronous actions get results, while notifications allow async updates.

It's a well-thought-out flow that balances structure with flexibility.
```

---

## Hands-on Examples

To solidify understanding, let's walk through two concrete examples of MCP in action with Python code, using FastMCP.

We'll create a minimal MCP server that offers a simple tool and resource, with Cursor and Claude desktop as host/client.

```{admonition} About FastMCP
:class: tip
FastMCP is a Pythonic framework that simplifies the creation of MCP servers and clients. It offers features like full client support, server composition, and integration with FastAPI.
```

### Setup

Firstly, you need to download and set up Cursor and Claude desktop on your system and ensure your python version is 3.8 or higher.

- https://www.cursor.com/
- https://claude.ai/

Then setup uv package manager:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Moving on, the next step is project initialization:

```bash
uv init mcp-tutorial-project
```

Then move inside the project directory:

```bash
cd mcp-tutorial-project
```

Initialize a virtual environment:

```bash
uv venv
```

Activate the virtual environment:

```bash
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

Install the FastMCP package:

```bash
uv add fastmcp
```

Done!

### Server Program

Now create a file `server.py` and add the code below:

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
        # Consider using ast.literal_eval or a proper math parser
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
    # Mock exchange rates - in reality, you'd use a real API
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

This Python script sets up a sample MCP server using the FastMCP library. It defines a server called "Sample Server" and exposes three tools that an AI (like Claude or Cursor) can invoke through the MCP protocol.

Let's look at it step by step:

```python
mcp = FastMCP("Sample Server")
```

This creates an MCP server named "Sample Server". The server will handle incoming tool calls over MCP.

**Tools Defined:**

1. **get_weather**: A simple mock tool that returns hardcoded weather data for any location.

   - Example usage: `get_weather("Delhi")` returns "Weather in Delhi: Sunny, 72°F"

2. **calculate**: Evaluates a math expression using Python's `eval()`.
   - Example: `calculate("2 + 3 * 4")` → 14.0

```{admonition} Security Warning
:class: warning
Caution: `eval()` is dangerous in production; it should be sandboxed or replaced with a parser like `ast.literal_eval`.
```

3. **convert_currency**: A sample tool that converts amount between two currencies using hardcoded exchange rates.
   - Example: `convert_currency(100, "USD", "INR")` → "100 USD = 8300.00 INR"

```python
mcp.run()
```

Launches the MCP server and starts listening for tool invocation requests (from Claude, Cursor, or a custom client). Uses stdio transport mode. For SSE simply change the `mcp.run()` line to:

```python
mcp.run(transport="sse", host="127.0.0.1", port=8000)
```

### Connect to Cursor/Claude

**For Cursor:** Go to settings and then MCP tab, then click on "Add new global MCP server" and put the below configuration into the `mcp.json` file:

For stdio:

```json
{
  "mcpServers": {
    "sample-server": {
      "command": "python",
      "args": ["server.py"]
    }
  }
}
```

For SSE:

```json
{
  "mcpServers": {
    "sample-server": {
      "url": "http://127.0.0.1:8000/sse"
    }
  }
}
```

```{admonition} SSE Transport Note
:class: note
For SSE transport, one would need to run the server with `uv run server.py` before one could start interacting with it.
```

Once this is done, the server successfully starts and gets connected to Cursor as client and host. In the chat area, we can now interact with our MCP tools via relevant queries.

**Similarly, for Claude Desktop:** Go to settings and then "Developer" tab. Click on "Edit Config" and then inside `claude_desktop_config.json` add the same stdio configuration as shown earlier.

```{admonition} Claude SSE Limitation
:class: warning
As of now, Claude does not support MCP servers via SSE mechanism.
```

Then, one can see the MCP tools listed inside the available tools button present on the chat interface of Claude Desktop.

### An Interesting Example

Even with all the above implementations, we can still make things easier as developers who are using MCP servers.

See, the issue is that when you try to handle a multi-step request like "get the latest flight prices from X to Y on Z," you often end up hardcoding everything into a single function, which includes parsing, validation, fallback defaults, and more.

This bloats your logic and makes the tool brittle.

But with an LLM on the client side and MCP's tool chaining, you can break the task into smaller, well-documented tools, like one for parsing, one for validation, one for lookup, and let the LLM orchestrate the sequence intelligently.

This provides cleaner tools, better reuse, and zero need for procedural glue logic.

The below server example demonstrates the ability to leverage LLM on the client's end, interdependence of tools, importance of a clear and explicit docstring, and elimination of loads of explicit code by smart tool use with MCP.

```python
from fastmcp import FastMCP
from datetime import datetime

# Create MCP server for flight booking
mcp = FastMCP("Flight Booking Server")

@mcp.tool()
def search_flights(origin: str, destination: str, date: str) -> str:
    """Search for available flights between two cities on a specific date.

    Args:
        origin: Departure city
        destination: Arrival city
        date: Travel date in YYYY-MM-DD format

    Returns:
        JSON string with available flights including prices and times
    """
    # Mock flight data
    flights = [
        {"flight": "AI101", "price": 299, "departure": "08:00", "arrival": "11:30"},
        {"flight": "6E203", "price": 245, "departure": "14:20", "arrival": "17:45"},
        {"flight": "UK505", "price": 320, "departure": "19:15", "arrival": "22:40"}
    ]

    return f"Available flights from {origin} to {destination} on {date}: {flights}"

@mcp.tool()
def book_flight(flight_info: str, passenger_name: str, email: str) -> str:
    """Book a specific flight for a passenger.

    Args:
        flight_info: Flight details (flight number, price, etc.)
        passenger_name: Full name of the passenger
        email: Contact email for booking confirmation

    Returns:
        Booking confirmation with reference number
    """
    import random
    booking_ref = f"BK{random.randint(100000, 999999)}"

    return f"Flight booked successfully! Booking reference: {booking_ref}. " \
           f"Details: {flight_info}. Passenger: {passenger_name}. " \
           f"Confirmation will be sent to {email}"

@mcp.tool()
def send_confirmation(booking_details: str, email: str) -> str:
    """Send booking confirmation email to passenger.

    Args:
        booking_details: Complete booking information
        email: Recipient email address

    Returns:
        Confirmation of email sent
    """
    return f"Confirmation email sent to {email} with booking details: {booking_details}"

if __name__ == "__main__":
    mcp.run()
```

In the above example we have implemented a mockup application that uses MCP for flight booking. Now, before proceeding further, let's take a look at the output of the above code.

**Given Query:** "I want to book a flight from bangalore to delhi on 25th june, 2025. name avi chawla and email avi123@gmail.com book the cheapest... use mcp tools"

The first tool called is `search_flights`:

```{admonition} Automatic Data Conversion
:class: tip
Notice, how the LLM automatically converts information from our query into the intended format, and we did not need to implement any logic for that.
```

Then, sequentially, `book_flight` is called:

```{admonition} Intelligent Data Flow
:class: tip
The LLM automatically takes relevant portions of the previous result and passes as input to the `book_flight` tool, without us having to implement the parser.
```

Finally, the `send_confirmation` tool is invoked, and the result is given by the client LLM.

**Overall, the entire flow was:**

1. Search flights → Find available options
2. Book flight → Select cheapest option automatically
3. Send confirmation → Complete the booking process

From the code and the result we obtained, we can see that we had not written any explicit parser or used regex to obtain relevant portions of query or intermediate tool results to be passed on over to the next tool.

The LLM on the client's end automatically did that based on its own reasoning, which is extremely powerful yet not as obvious as it appears.

Neither the sequence of tool calling was determined beforehand. The LLM, on its own discretion, decided the tool sequence on the fly.

In this case, it logically followed the workflow: `search_flights` → `book_flight` → `send_confirmation`. Additionally, we can see, based on the instructions in the prompt and its own intelligence, the LLM automatically selected the flight with the lowest price from the available options, without requiring us to write any logic for that.

Overall, we were saved from implementing explicit logics, which otherwise would have been necessary. This is a very key area where we can see how smart tool use with MCP is saving us from a lot of effort.

Although this might appear trivial, it underscores two significant points:

**First,** if we had combined all functionality into a single monolithic tool, we would have needed to implement parsing and sequencing logic within that tool's code. According to MCP design principles, the LLM at the client side can only provide inputs and receive tool outputs so it cannot intervene during a tool's execution. This means all conditional logic and data processing would need to be coded within the tool itself.

**Second,** if we had used tool calling approaches such as custom API integrations, hardcoded tool chains, or non-standardized implementations, we might still have needed to explicitly handle all parsing and sequencing logic internally, depending on the implementation pattern.

In either of these situations, the system would have been rigid in addressing varying user needs. Providing preferences or instructions through natural language prompting, as effectively done with MCP's standardized multi-tool approach, would not have been sufficient to change the system's behavior.

```{admonition} Key Insight: Modular Design Benefits
:class: tip
Breaking functionality into smaller, composable tools allows the client-side LLM to manage intermediate outputs and make decisions dynamically. This approach enables the system to adapt to different user preferences expressed through natural language, without requiring code changes for each variation.
```

The granular tool design allows for much greater flexibility. For instance, a user could request to book the flight of a specific airline or something like "find flights under $300" and the LLM would adapt its tool selection and sequencing criteria accordingly, all without modifying the underlying tools.

Also, observe that we specified formats in the docstrings of the tools. As discussed earlier, the client becomes aware of the server's capabilities based on the function signature and the corresponding docstring.

We can see the LLM formed inputs to tools as per specifications in the docstrings. Hence, it is quite important to write clear and explicit docstrings.

```{admonition} Docstring Best Practices
:class: tip
For larger, more capable models like GPT-4 or Claude Sonnet, concise docstrings with good function signatures are often sufficient. However, explicit docstrings with detailed examples become crucial when working with smaller local models that may need more guidance to understand expected formats and behaviors. As a best practice, always write clear, detailed docstrings with examples, regardless of which LLM will be used as the client.
```

---

## Miscellaneous Concepts

Below is a concise yet important explanation of certain common doubts one faces while learning about MCPs.

### API versus MCP

APIs (Application Programming Interfaces) and MCPs are both used for communication between software systems, but they have distinct purposes and are designed for different use cases.

APIs are general-purpose interfaces for software-to-software communication, while MCPs are specifically designed for AI agents to interact with external tools and data.

**Key differences:**

- MCPs aim to standardize how AI agents interact with tools, while APIs can vary greatly in their implementation.
- MCPs are designed to manage dynamic, evolving context, including data resources, executable tools, and prompts for workflows.
- MCPs are particularly well-suited for AI agents that need to adapt to new capabilities and tools without pre-programming.

**In a traditional API setup:**

If your API initially requires two parameters (e.g., location and date for a weather service), users integrate their applications to send requests with those exact parameters.

Later, if you decide to add a third required parameter (e.g., unit for temperature units like Celsius or Fahrenheit), the API's contract changes.

This means all users of your API must update their code to include the new parameter. If they don't update, their requests might fail, return errors, or provide incomplete results.

**MCP's design solves this as follows:**

For instance, when a client (e.g., an AI application like Claude Desktop) connects to an MCP server (e.g., your weather service), it sends an initial request to learn the server's capabilities.

The server responds with details about its available tools, resources, prompts, and parameters. For example, if your weather API initially supports location and date, the server communicates these as part of its capabilities.

If you later add a unit parameter, the MCP server can dynamically update its capability description during the next exchange. The client doesn't need to hardcode or predefine the parameters since it simply queries the server's current capabilities and adapts accordingly.

This way, the client can then adjust its behavior on-the-fly, using the updated capabilities (e.g., including unit in its requests) without needing to rewrite or redeploy code.

We'll understand this topic better in Part 3 of this course, when we build a custom client and see how it communicates with the server.

### MCP versus Function Calling

Before MCPs became mainstream (or popular like they are right now), most AI workflows relied on traditional function calling for tools.

```{mermaid}
graph TB
    subgraph "Traditional Function Calling"
        A["AI Application"] --> B["Function Registry"]
        B --> C["Function 1<br/>(Weather)"]
        B --> D["Function 2<br/>(Calculator)"]
        B --> E["Function 3<br/>(Email)"]
        F["Changes require<br/>app updates"]
        G["Tightly coupled<br/>to application"]
    end

    subgraph "MCP Approach"
        H["AI Host"] --> I["MCP Client"]
        I <--> J["MCP Server 1<br/>(Weather Service)"]
        I <--> K["MCP Server 2<br/>(Math Tools)"]
        I <--> L["MCP Server 3<br/>(Email Service)"]

        J --> M["Dynamic Discovery"]
        K --> N["Protocol Standard"]
        L --> O["Modular Design"]
    end

    style A fill:#ff9999
    style H fill:#99ff99
    style B fill:#ffcccc
    style I fill:#ccffcc
    style C fill:#ccccff
    style D fill:#ccccff
    style E fill:#ccccff
    style J fill:#99ccff
    style K fill:#99ccff
    style L fill:#99ccff
```

Here's a visual comparison between traditional Function calling and MCP approaches

Function calling enables LLMs to execute predefined functions based on user inputs. In this approach, developers define specific functions, and the LLM determines which function to invoke by analyzing the user's prompt. The process involves:

1. Developers create functions with clear input and output parameters.
2. The LLM interprets the user's input to identify the appropriate function to call.
3. The application executes the identified function, processes the result, and returns the response to the user.

**But there are limitations as well:**

- As the number of functions grows, managing and integrating them becomes complex. Requires M×N integrations.
- Functions are closely tied to specific applications, making reuse across different systems challenging.
- Any changes require manual updates across all instances where the function is used.

MCP offers a standardized protocol for integrating LLMs with external tools and data sources. It decouples tool implementation from their consumption, allowing for more modular and scalable AI systems.

#### Comparative Overview

| Aspect                | Traditional Function Calling                 | Model Context Protocol (MCP)                       |
| --------------------- | -------------------------------------------- | -------------------------------------------------- |
| Integration           | Manual and application-specific              | Standardized and reusable across platforms         |
| Scalability           | Limited by manual management                 | High, due to dynamic discovery and standardization |
| Maintenance           | Requires manual updates in each application  | Centralized updates propagate across systems       |
| Flexibility           | Rigid, with tight coupling                   | Flexible, with decoupled architecture              |
| Security & Compliance | Basic, depends on individual implementations | Built-in support for audit and approval workflows  |

While traditional function calling provides a straightforward method for integrating LLMs with specific functions, it falls short in scalability and flexibility. MCP addresses these challenges by offering a standardized, decoupled approach that promotes interoperability and maintainability.

The dynamic discovery of capabilities ensures that any changes (in available capabilities or schemas of existing capabilities) at the server end are automatically reflected at the client side, without requiring any changes in codes at the client's end.

Adopting MCP can lead to more robust and adaptable AI systems, especially as the complexity and number of integrated tools grow.

---

## Conclusion and Next Steps

With Part 2, we've completed our deep dive into the Model Context Protocol, not just in theory, but in practice.

We started with a close look at MCP's capabilities: how tools, resources, and prompts can be exposed by a server and used by AI models through a structured, standardized interface.

We then walked through the MCP communication protocol, why it's built on JSON-RPC, the different message types it supports (requests, responses, notifications), and how communication flows through transport mechanisms like stdio and HTTP + SSE.

We also explored the complete interaction lifecycle of MCP, such as initialization, discovery, execution, and termination, so you can reason about how end-to-end orchestration works.

Finally, we brought it all together with hands-on examples, connecting everything from a local Python server to Claude and Cursor, and even exploring modular server design with tool interdependence.

If Part 1 was about understanding MCP's purpose and architecture, Part 2 showed how to build with it, turning theory into working systems.

Together, these chapters offer a full "theory of operation" for AI-tool interoperability. MCP shifts the paradigm from prompt engineering to systems engineering for AI, where models interact intelligently with a growing ecosystem of tools, data, and interfaces.

This sets the stage for what's next: building secure, scalable, and production-grade MCP applications. In future chapters, we'll cover:

- Advanced sampling workflows
- Permissioning and sandboxing
- Real-world deployments and integrations

```{admonition} Next Chapter
:class: note
Read part 3 here, where you will learn how to build a custom MCP client from scratch: [The Full MCP Blueprint: Part C - Building a Custom MCP Client](../part-c/mcp-blueprint-part-c.md)
```

Thanks for reading!

---

## Discussion

Any questions? Feel free to post them in the comments or connect via chat for private discussions.

```{admonition} Tags
:class: note
Topics: Agents, MCP, MCP Crash Course
```
