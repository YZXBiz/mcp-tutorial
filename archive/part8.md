
Daily Dose of Data Science
Sponsor
Newsletter
More
Search 🔎
Zhuoxin Yang
Jul 13, 2025
The Full MCP Blueprint: Practical MCP Integration with 4 Popular Agentic Frameworks
Model context protocol crash course—Part 8.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar

Recap
What’s in this part
Recent developments
Framework overview
LangGraph integration with MCP
LlamaIndex Integration with MCP
CrewAI integration with MCP
PydanticAI integration with MCP
Recommendations
Conclusion
Recap
Before we dive into Part 8 of the MCP crash course, let’s briefly recap what we covered in the previous part of this course.


In Parts 6 and 7, we explored testing, security, and sandboxing in MCP-based systems. We began by examining MCP Inspector, a powerful tool that allows developers to visualize and debug interactions with MCP servers effectively.


Next, we discussed several key security vulnerabilities in MCP systems, including prompt injection, tool poisoning, and rug pull attacks, along with strategies for mitigating these risks.


We also explored MCP roots and how they can help limit unrestricted access, reducing potential security threats.


We then turned our focus to containerization and sandboxing techniques for MCP servers to enhance security and ensure reproducibility.

This included writing secure Dockerfiles for FastMCP servers, implementing hardened settings such as read-only filesystems, memory limits, and other container-level restrictions to reduce the attack surface.

Additionally, we covered how to integrate sandboxed servers with tools like Claude Desktop, Cursor IDE, and custom clients.


Finally, we demonstrated how to test these sandboxed servers using MCP Inspector, including connecting both STDIO and SSE-based sandboxed servers for debugging and inspection.


By the end of Part 7, we had a clear picture of how to examine and build secure environments for end-to-end cooperative intelligence systems developed with MCP.

If you haven’t studied Parts 6 and 7 yet, we strongly recommend going through them first. You can find them below:

The Full MCP Blueprint: Testing, Security and Sandboxing in MCPs (Part A)
Model context protocol crash course—Part 6.

Daily Dose of Data Science
Avi Chawla

The Full MCP Blueprint: Testing, Security, and Sandboxing in MCPs (Part B)
Model context protocol crash course—Part 7.

Daily Dose of Data Science
Avi Chawla

What’s in this part
In this chapter, we shift our focus to the integration of the model context protocol (MCP) with some of the most widely used agentic frameworks: LangGraph, LlamaIndex, CrewAI, and PydanticAI.


Previously, in our AI agents crash course, we explored how agents reason, invoke tools, and manage state:

AI Agent Crash Course - Daily Dose of Data Science

Daily Dose of Data Science
Avi Chawla

But one essential question remains: how can they interface with external systems and access dynamic capabilities in a standardized manner that MCP provides?

Thus, in this part, we'll explore how MCP enables clean, extensible, and reliable integration with frameworks that are powering modern AI agents.

MCP integration not only expands an Agent’s capabilities but also brings new responsibilities. It introduces interoperability, remote procedure access, and modular system composition. At the same time, it requires careful design to maintain isolation, trust, and correctness.

Here’s what we’ll explore:

Some recent advancements in the realm of MCP.
A clear and concise primer on each of the four frameworks.
Step-by-step guides for connecting MCP into each framework.
This chapter is practical, detailed, and grounded in the realities of building agentic systems.

Each topic will be accompanied by detailed implementations, ensuring that you not only grasp the idea behind it but can also perform integrations into your own stack.

Let’s begin.

Recent developments
Before getting into the core of this chapter, where we'll explore the integration of MCP in modern Agentic frameworks, we'd like to take some time to address some of the recent updates in the MCP world.

Ever since we started this crash course, several significant developments have emerged that are important to acknowledge.

So in this section, we will briefly highlight some of these key updates and provide useful links to help you stay informed as new information becomes available.

Streamable HTTP transport
Streamable HTTP is a modern, efficient transport for exposing your MCP server via HTTP. It is the recommended transport for web-based deployments.

To run a server using streamable HTTP, you can use the run() method with the transport argument set to "http". This will start the server on the default host (127.0.0.1), port (8000), and path (/mcp/).


👉
For backward compatibility, in FastMCP, wherever "http" is accepted as a transport name, you can also pass "streamable-http" as a fully supported alias.
We can also specify the host and port parameters similar to how we did for sse:


On the client side, to successfully connect to streamable HTTP servers, we can use the following approach:


With this, we’ve covered an overview of how to use the streamable HTTP transport mechanism.

It's important to note that the SSE transport is deprecated and may be removed in a future version. While we have used SSE in several places for demonstration purposes, and will continue to use it in this chapter as well, newer applications should prefer the streamable HTTP transport instead.

Finally, we’d like to mention one more important point: from now on, when using SSE (as long as it remains supported), the client should explicitly use the SSETransport to connect to the server:


👉
FastMCP will attempt to infer the appropriate transport from the provided configuration, but HTTP URLs are assumed to be Streamable HTTP (as of FastMCP 2.3.0).
We covered the core idea but for further exploration, you can find more details on this topic at the following link:

Running Your FastMCP Server - FastMCP
Learn how to run and deploy your FastMCP server using various transport protocols like STDIO, Streamable HTTP, and SSE.

FastMCP

Now, let us take a look at another very recent development: Elicitation.

Elicitation
This is a new feature in version 2.10.0 of FastMCP. User elicitation allows MCP servers to request structured input from users during tool execution.

Instead of requiring all inputs upfront, tools can interactively ask for missing parameters, clarification, or additional context as needed.

Elicitation enables tools to pause execution and request specific information from users. This is particularly useful for:

Missing parameters
Clarification requests
In order to utilize elicitation on the server-side, use the ctx.elicit() method within any tool function to request user input:


The elicitation result contains an action field indicating how the user responded:

accept: User provided valid input; data is available in the data field
decline: User chose not to provide the requested information and the data field is None
cancel: User cancelled the entire operation and the data field is None
Now let us take a look at how we can handle elicitation on the client-side.

Elicitation requires the client to implement an elicitation handler. If a client doesn’t support elicitation, any invocation of ctx.elicit() will raise an error indicating that elicitation is not supported.

Let us take a look at how we can implement a basic handler on the client-side for elicitation:


In the above code:

Line 7: We show the message from the server.
Line 9: We ask the user to provide input.
Line 12-14: If the user provides no input, we return an object of the ElicitResult class with a decline action.
Line 19: If the user provides an input, we return it after wrapping it in the right format.
Line 22-25: The Client integrates the server and runs it with an elicitation_handler to introduce support for Elicitation.
FastMCP automatically converts the server’s JSON schema into a Python dataclass type, making it easy to construct the response.

We can also observe that the implementation patterns for elicitation are quite similar to the ones we saw for sampling.

👉
The above code examples have been taken directly from the FastMCP docs for demonstration.
With this, we covered the core idea behind elicitation. For a deeper dive, check out the following links:

User Elicitation - FastMCP
Request structured input from users during tool execution through the MCP context.

FastMCP

User Elicitation - FastMCP
Handle server-initiated user input requests with structured schemas.

FastMCP

Claude Desktop support
Claude Desktop now supports MCP servers both through local STDIO connections and remote servers (beta), allowing you to extend Claude’s capabilities with custom tools, resources, and prompts from your MCP servers.

However, it is important to note that remote MCP server support is currently in beta and available for users on Claude Pro, Max, Team, and Enterprise plans (as of June 2025). Most users will still need to use local STDIO connections.

To stay up to date with the latest developments around MCP, we recommend regularly checking the following key resources:

Documentation:

Welcome to FastMCP 2.0! - FastMCP
The fast, Pythonic way to build MCP servers and clients.

FastMCP

Update pages:

FastMCP Updates - FastMCP

FastMCP

Example Clients - Model Context Protocol
A list of applications that support MCP integrations

Model Context Protocol

Now that we have discussed the latest updates in MCP, let's shift our focus to the primary objective of this chapter: integration of MCP into the most popular Agentic AI frameworks: LlamaIndex, CrewAI, Langgraph and PydanticAI.

Framework overview
Before diving into integration specifics, let’s briefly overview each framework and its role in the agentic AI ecosystem:

LangGraph

A framework for building stateful, graph-based AI workflows with multiple actors. LangGraph emphasizes a graph-of-nodes approach where each node can represent an LLM invocation or a tool call, allowing complex agent behaviors.

It is designed to orchestrate multi-step interactions and it also integrates with LangChain components wherever needed.

In essence, LangGraph provides the “wiring” for connecting LLMs to tools in a directed workflow graph.

LlamaIndex

An open-source framework, formerly known as GPT Index, originally built for connecting LLMs to external data via indexes, which has evolved to include an agent toolkit.

LlamaIndex enables LLMs to use tools and external knowledge through a structured interface. Recently, it added an MCP connector that allows LlamaIndex agents to call MCP-compliant capabilities.

This lets developers deploy LlamaIndex agents that can invoke any tool exposed by an MCP server.

CrewAI

A lightweight, high-performance and almost no-code agent framework to create “AI agent teams” (called crews) where each agent has a specialized role. CrewAI emphasizes autonomous collaboration among agents and fine-grained control over their workflows.

It provides constructs like Crew, Agent, Task, and Process to structure complex AI solutions. CrewAI’s design focuses on native support for the integration of external tools.

Through the crewai-tools library, CrewAI can treat MCP servers as tool providers.


PydanticAI

A Python agent framework aimed at production-grade AI applications. PydanticAI uses Pydantic’s data validation strengths to enforce type-safe interactions with LLMs and tools.

Importantly, PydanticAI has first-class support for MCP, this means, an agent can act as an MCP client and agents can be embedded as MCP server tools.

In practice, PydanticAI’s MCP integration is very comprehensive, covering multiple transport protocols and features like sampling. We’ll explore these capabilities in detail.

With these introductions in mind, let’s examine how each framework integrates with MCP. We’ll start with LangGraph, move through LlamaIndex and CrewAI, and conclude with Pydantic’s MCP support. Each section includes code snippets and explanations wherever necessary to illustrate the integration patterns.

LangGraph integration with MCP

LangGraph, by design, enables complex AI agent workflows using a graph of nodes that manage state and decisions. Integrating MCP into LangGraph allows these graph-based agents to utilize capabilities exposed by MCP servers.

In practical terms, a LangGraph agent can list and invoke MCP prompts, resources, and tools as part of its workflow.

This combination gives a structured, stateful control (via LangGraph) over a rich ecosystem of MCP-provided capabilities.

Project setup
The code and project setup we are using here are attached below as a zip file. You can simply extract it and run uv sync command to get going. It is recommended to follow the instructions in the README file to get started.


Download the zip file below:

langgraph_mcp
langgraph_mcp.zip86 KB
For details about the dependencies, you can check the pyproject.toml file.

Setting up MCP servers
To ground the discussion, let’s consider two simple MCP servers that our LangGraph Agent will utilize tools from.

Let's create our first server as server_one.py:


Then let's create our second server and save it as server_two.py:


These MCP servers are STDIO transport-based and define one tool each (add and multiply).

When run, it listens (here via standard I/O) for client requests. Any MCP-compliant client can connect to list available capabilities or to invoke them.

Now that the MCP servers are ready, let us see how we can integrate them into LangGraph.

LangGraph provides adapter functions to load MCP-provided tools, resources, and prompts into its environment.

These are available via the langchain_mcp_adapters package, which bridges the MCP client and LangChain’s tool interface.

Server integration with LangGraph
LangGraph uses its own adapter, but under the hood it relies on the standard MCP client from the MCP Python SDK.

One of MCP’s strengths is that it is an open protocol, so an agent can connect to multiple independent MCP servers simultaneously.

LangGraph supports this via a MultiServerMCPClient utility, which initiates multiple client sessions to manage each of the multiple server connections and aggregate their tools.

👉
So basically our LangGraph script works as an MCP Host, that can connect to multiple servers via multiple managed client sessions, that is, one client session per server.
This is useful if different capabilities are hosted on different MCP servers (for modularity or scaling reasons).

Hence, using MultiServerMCPClient, we can configure connections to both servers and retrieve a combined tool set.

Now to get started with our integration code, firstly, we'll add the necessary imports:


After this, we'll configure our server parameters:


Since both our servers are STDIO-based, we use the above format. In case of SSE or streamable HTTP, we would use something like:


In the above code, we demonstrate the SSE server on port 8000 and the streamable HTTP one on port 8001.

👉
In case of sse or streamable-http, make sure that before trying to connect to the server via the client, the servers are running, that is, the server scripts are executed with python or uv run in the command line.
Once we are done with the configuration part, let us take a look at the following piece of code:


This code block defines an asynchronous function create_graph that sets up a LangGraph-based agent system. Here's a step-by-step explanation:

Line 1: It's an async function, meaning it returns a coroutine and supports await. It takes in two MCP client sessions (first_session and second_session), linked to separate MCP servers.
Line 3: The LLM is setup as gpt-4o. An API key is directly provided (or could be loaded from .env).
Lines 4-5: Loaded MCP tools using load_mcp_tools(session), we fetched all tools from the connected MCP server and got them as LangChain tool objects. These include the add and multiply tools.
Line 6: Combines tools from both servers into a single list.
Line 7: Wraps the LLM with tool usage capability.
Line 8-11: Defines a chat prompt template. A system message and a placeholder for prior chat messages.
Line 12: chat_llm is a prompt-bound LLM capable of tool usage.
Line 14-16: Declares the graph state as a dict with a single key messages. The messages list is automatically updated via add_messages.
Line 18-21: We define a function for chat nodes. This node uses the LLM (with tools bound) to generate a new message based on conversation history (state["messages"]). The output which could include tool calls is appended back into state["messages"].
Line 24: Initializes a LangGraph stateful flow with state schema defined earlier.
Line 25-26: Adds two nodes -
chat_node: where reasoning happens.
tool_node: a built-in node that automatically executes tool calls from messages.
Line 27: The graph starts at chat_node.
Line 28-29: After chat_node, the graph checks whether the next step involves tools.
If yes → go to tool_node.
If no tool needed → end the graph.
Line 30: After tool execution, go back to chat_node for further reasoning or message generation.
Line 31: Compiles the stateful workflow into an executable graph. MemorySaver() ensures that intermediate states are saved in memory (ephemeral in nature).
The below diagram effectively summarizes the workflow in form of a flowchart:


Next up, we have our asynchronous main function:


This is the function that runs the LangGraph-based agent (defined using create_graph) in a live command-line chat loop.

Here, two MCP client sessions are opened to two different servers: "first_server" and "second_server". These are passed to the create_graph() function to load tools from each and set up the agent.

With this, we complete our implementation of MCP with LangGraph. We can simply run the script with python or uv run in a terminal. Have a look at the following example results:


User query: "what tools do you have"
In the above screenshot, we can see the MCP tools available to our client via the server.

💡
"Parallel Tool Use" is an internal LangGraph mechanism for running tools in parallel. It’s not defined in our server code, and when testing the script on your end, you may or may not see this listed. There's no need for concern either way.

User query: "multiply 897 and 789"
In this screenshot, we can see the workflow utilizing our multiplication tool.

This multi-server integration demonstrates how LangGraph plus MCP can scale an agent’s tool use modularly. Under the hood, each tool invocation goes to the appropriate MCP server, but LangGraph abstracts that away.

The MultiServerMCPClient manages separate ClientSession instances to each server, either launching subprocesses (STDIO) or maintaining connections (SSE or streamable HTTP) as needed.

At this point, we would like to make a point that LangGraph also has support for MCP resources and prompts. The usage is quite similar in syntax to tools, take a look here:


To print and check the content, we can use print(example_prompt) for prompts and for resources print(example_resources[0].as_string()) to check the content of the first item/resource in the list, since resources are present as a list of Blob in LangGraph.

In summary, LangGraph’s integration with MCP allows an AI agent to become a true polymath: it can draw on a network of specialized MCP servers for different tasks, all within a controlled, stateful workflow.

The MCP adapter functions and the MultiServerMCPClient make it straightforward to plug these external skills into the LangGraph pipeline.

Next, let's take a look at how we can use MCP with LlamaIndex.

LlamaIndex Integration with MCP

LlamaIndex is widely used for enabling LLMs to access external data sources, such as documents and databases, via indexes.

We used it throughout our RAG crash course as well:

RAG Crash Course - Daily Dose of Data Science

Daily Dose of Data Science
Avi Chawla

Recently, LlamaIndex introduced support for tools and agents, including an MCP connector that allows an agent built on LlamaIndex to call tools hosted on an MCP server.

This means we can augment a LlamaIndex agent with functionalities that live outside its local environment, accessible via the MCP.

The integration is relatively lightweight: LlamaIndex provides an MCP client class and a tool specification to bridge to MCP servers.

Let’s explore how to build a LlamaIndex agent that uses MCP.

MCP connector components in LlamaIndex
LlamaIndex’s MCP integration centers around two main classes, found in the llama_index.tools.mcp module:

BasicMCPClient: A client that knows how to connect to an MCP server (over a given transport like SSE) and implement required MCP operations.
McpToolSpec: A ToolSpec implementation that, given a BasicMCPClient, fetches all the tools from the server and presents them as LlamaIndex FunctionTool objects for an agent to use.
In simpler terms:

BasicMCPClient handles the low-level communication with the server.
McpToolSpec translates the server’s offerings (tool names, descriptions, and interfaces) into a form that LlamaIndex can consume.
Project setup
The codes and project setup we are using are attached below as a zip file. You can simply extract it and run uv sync command to get going. It is recommended to follow the instructions in the README file to get started.


Download the zip file below:

llamaindex_mcp
llamaindex_mcp.zip89 KB
For details about the dependencies, you can check the pyproject.toml file.

Server setup
Imagine we have an MCP server with three tools:

add(a, b): Adds two numbers
multiply(a, b): Multiplies two numbers
weather(loc): Returns weather information for a location
This is precisely what our MCP server looks like:


The server runs on http://127.0.0.1:8000/sse using SSE as the transport mechanism.

Server integration with LlamaIndex
To get started with our integration code, firstly, we'll add the necessary imports:


Apart from imports, we also load our environment variables, basically the LLM's API key, from the .env file.

Then we build a LlamaIndex agent with MCP tools. We define a system prompt and construct the agent using FunctionAgent and LiteLLM:


This agent will invoke the appropriate tool (add, multiply, or weather) depending on the user’s message.

Next up, we'll define a function for handling user input and tool usage:


The handle_user_message function is an async utility designed to handle a single user message, run it through a LlamaIndex agent, and optionally log tool usage.

Let's break it down step by step:

Line 1-6: It's the function signature where:
message_content: The user’s input (a natural language query).
agent: The LlamaIndex FunctionAgent instance.
agent_context: A Context object that tracks the ongoing conversation and tool state.
verbose: If True, logs intermediate tool calls/results for debugging or inspection.
Line 8: This triggers the agent's reasoning loop using the LLM and tools.
Line 10: This line asynchronously listens for tool-related events as they happen in real-time. It uses an async for loop to process each event emitted during the agent’s execution.
Line 12-17: ToolCall is triggered when the agent decides to use a tool. This logs which tool was chosen and with what arguments. ToolCallResult is triggered when the tool returns its result. This logs the tool's output.
Line 19-21: After all events have streamed, the handler resolves into the final LLM-generated response. This is returned as a string for display/output.
Once we are at this stage, we'll establish the MCP connection using LlamaIndex’s provided classes for MCP:


This setup prepares LlamaIndex to fetch and use tools defined in the MCP server, including add, multiply, and weather.

Note that we're using SSE transport over here. For other transports, you can do this:


Finally, we write our main function responsible for running the agent loop:


The main() function initializes a LlamaIndex agent with MCP tool access and sets up conversation context.

It then enters an interactive loop where it accepts user input, processes it through the agent using handle_user_message(), and prints the agent's response.

The diagram below effectively summarizes the crux of the entire workflow. Have a look:


Note that before running the client-side script, make sure the server is running since we are using SSE:


Then we run the client script and ask queries:


User query: "what tools do you have"
In the above screenshot, we see the available tools via MCP. Similar to LangGraph, we can observe that LLamaIndex also mentions the parallel tool execution ability.

In the following screenshot, we demonstrate the use of the weather tool:


User query: "what is the weather in Delhi"
At this point, it is worth making a note that the BasicMCPClient provides comprehensive access to MCP server capabilities beyond just tools:


In the above code, we can see that alongside tools, LlamaIndex also has very streamlined support for prompts and resources.

💡
The above piece of code, has directly been referenced from the LlamaIndex documentation.
With this, we have also seen how we would perform MCP integration with LlamaIndex. LlamaIndex’s MCP connector is a straightforward way to extend LLM agent functionality by connecting to external systems using the MCP protocol.

In summary, we just point a BasicMCPClient to an MCP server, wrap it with McpToolSpec, and plug it into a FunctionAgent.

This unlocks powerful use cases like connecting to microservices, triggering remote workflows, etc., all via a consistent interface, decoupling tool logic from the agent’s core behavior.

Next up, let's see how we can leverage the powers of MCP in CrewAI.

CrewAI integration with MCP

CrewAI approaches the agentic AI paradigm by orchestrating teams of agents (called crew members) to collaborate on tasks.

Integrating MCP into CrewAI allows these agents to use external tools provided by MCP servers as part of their workflow.

Because CrewAI is built with an emphasis on tooling and modularity, the integration is quite seamless. MCP servers are essentially treated as tool providers for CrewAI agents.

CrewAI provides a dedicated utility class MCPServerAdapter (in the crewai_tools package) to handle MCP integration. This adapter abstracts away the details of connecting to an MCP server (including managing the process or network connection) and yields a set of tool objects that an agent can use.

Let’s examine how to use MCPServerAdapter and highlight CrewAI’s current capabilities and limitations with MCP.

Project setup
The codes and project setup we are using are attached below as a zip file. You can simply extract it and run uv sync command to get going. It is recommended to follow the instructions in the README file to get started.


Download the zip file below:

crewai_mcp
crewai_mcp.zip168 KB
For details about the dependencies, you can check the pyproject.toml file.

On a side note, if you're on Windows and during project setup, you get an error similar to the one shown below:


Then your system lacks the required C++ build tools. To resolve this, you must install Microsoft C++ Build Tools from here.

Once downloaded, run the installer and from the installer download "Desktop development with C++".

Check out this reference:

GitHub - bycloudai/InstallVSBuildToolsWindows: Tutorial on how to install Microsoft C++ Build Tools
Tutorial on how to install Microsoft C++ Build Tools - bycloudai/InstallVSBuildToolsWindows

GitHub
bycloudai

Server setup
Let's use the same server setup as the one we used above with LlamaIndex:


The server runs on http://127.0.0.1:8000/sse using SSE as the transport mechanism.

Integration
CrewAI’s MCPServerAdapter can connect to MCP servers over different transports like STDIO, SSE, or streamable HTTP.

In our setup, we'll be using SSE. The integration involves defining a server configuration and then using a context manager to establish a connection and retrieve tool definitions from the MCP server.

Firstly, let us add the necessary imports and also load the LLM:


Now, since in our example we have a simple FastMCP server (server.py) exposing tools like add, multiply, and weather, and it runs with transport='sse' at localhost:8000. In crew_mcp.py, we connect to this server using the SSE configuration:


Note that in the case of streamable HTTP, our configuration would be quite similar to SSE:


But for the STDIO transport mode, our configuration would differ. We would need to use StdioServerParameters from mcp to define server_params:


Now, whatever the transport may be, this server_params is passed into MCPServerAdapter. Within the with block, it yields access to the remote tools provided by the server:


This list-like adapter is then directly injected into a CrewAI agent. We create an agent configured to use these tools:


This agent is embedded into an interactive CLI loop that repeatedly accepts user queries, creates a corresponding Task, runs it using Crew.kickoff(), and displays the result:


With this, our integration is basically complete. We can simply run the script with python or uv run in a terminal. But since our server is of the SSE type, make sure that you run it first.

Below are some screenshots of sample results:


Appears on running the 'crew_mcp.py' file
Now, suppose we give a query for the weather tool, we'll see the crew execution start:


User query: "what is the weather in Jaipur?"
Then, midway somewhere, we'll see our tool call results:


Then, consequently, we'll get a final response to our query:


👉
If on running the codes, some sort of deprecation warnings appear, you can safely ignore them.
Current limitations
It’s important to mention that, as of now, CrewAI’s MCP integration focuses only on tools. The adapter does not directly surface MCP prompts or resources to CrewAI agents.

So, an MCP server’s prompts and resource files are not automatically integrated. Agents still have their prompts/goals defined in CrewAI’s way.

If an MCP server has critical context in a prompt or resource, you’d need to fetch it via a separate mechanism and incorporate it into the agent’s prompt manually.

Additionally, the adapter mainly deals with text-based tool outputs (content text). Complex outputs or non-text data might need custom handling, because by default, the adapter looks at the primary text content of the tool result.

To summarize
Despite the present limitations, overall, we can't deny that CrewAI’s integration makes using MCP servers as easy as using any other tool. By wrapping the connection in MCPServerAdapter, a CrewAI agent (or crew of agents) can leverage a huge range of capabilities.

The design cleanly separates the concerns: CrewAI agents focus on when to use a tool, and the MCP server provides how the tool executes. This keeps the Agents lean while giving them expansive abilities furnished by the MCP ecosystem.

PydanticAI integration with MCP

Finally, we turn to PydanticAI, which arguably offers the most streamlined integration with MCP. As outlined earlier, PydanticAI was built with MCP in mind from the start.

In PydanticAI, agents can be implemented as MCP Clients. A PydanticAI agent can directly connect to MCP servers to use their tools, similar to the scenarios we’ve seen with LangGraph, LlamaIndex, and CrewAI.

Pydantic provides convenient classes for each MCP transport and integrates the tool-calling into the agent’s run loop.

Project setup
The codes and project setup we are using are attached below as a zip file. You can simply extract it and run uv sync command to get going. It is recommended to follow the instructions in the README file to get started.


Download the zip file below:

pydantic_mcp
pydantic_mcp.zip49 KB
For details about the dependencies, you can check the pyproject.toml file.

Server setup
We'll use the same server setup as the one for LlamaIndex and CrewAI:


The server runs on http://127.0.0.1:8000/sse using SSE as the transport mechanism.

PydanticAI agent as an MCP client
PydanticAI supports connecting agents to external tool servers using the MCP protocol.

It provides connector classes like MCPServerSSE, MCPServerStreamableHTTP, and MCPServerStdio, each corresponding to a specific MCP transport. We instantiate these with the appropriate server information, either a URL or a command, and pass them to the Agent.

The agent can then invoke tools provided by the server automatically.

In our case, we'll be using the MCPServerSSE class. Firstly, we'll add the necessary imports and set our environment variables:


Then we set up our connection configuration and agent as depicted below:


This sets up an agent connected to our MCP server.

Then we define our main function with the agent as an async context manager with agent.run_mcp_servers().

Let’s see how exactly to do that:


So what’s happening under the hood?

Once you enter the async with agent.run_mcp_servers() block, the connection is established and stays active for the duration of the block.

Then you input a query: "What is the weather at Delhi?" The agent processes this prompt. The model correctly identifies that the weather tool is appropriate, and it generates a tool call.

Before running the client script, make sure the server is running since we are using the SSE transport. Then we can run the client script and observe the output:


PydanticAI takes care of the full lifecycle. The model decides to use a tool, PydanticAI routes that request through the MCP client, which sends it over sse to the FastMCP server.

The tool runs, produces a result and that response is then returned back to the agent. Finally, result.output contains the full generated answer.

Note that if you want to switch from SSE to STDIO or HTTP, you can do that with:


So to summarize our setup uses PydanticAI to run a FastMCP server with tool capabilities. The agent knows about these tools, decides when to use them, and returns synthesized answers, all without us manually orchestrating tool calls.

It’s a clean and powerful way to extend LLM reasoning with local tool execution.

👉
If on running the server/client codes, some sort of deprecation warnings appear, you can safely ignore them.
Sampling
In PydanticAI, MCP sampling is supported. Let's walk through a quick example demonstrating the use of sampling in PydanticAI.

First, let us create a server setup similar to how we did in Part 5 when we first learnt sampling:


This server provides a poem-writing tool facility. Now, let us take a look at how we would be implementing the client-side with the help of PydanticAI:


In the above code, we can see that our client-side does not really change. This is because sampling is automatically supported by PydanticAI agents when they act as a client.

👉
If on running the server/client codes, some sort of deprecation warnings appear, you can safely ignore them.
We can also disallow sampling by setting allow_sampling=False when creating the server reference:


With this, we have also covered support for MCP sampling in PydanticAI SDK.

Hence, to summarize, PydanticAI’s SDK integration exemplifies a streamlined embrace of the MCP concept.

The code we’ve gone through shows that with only a few lines of configuration, a Pydantic agent can gain a host of new abilities provided by MCP tools.

Recommendations
Based on the context set by this chapter, we'd like to recommend some knowledge-enhancing tasks for you:

Recently, the concept of Authorization in MCP has been introduced, providing authorization capabilities at the transport level. This enables MCP clients to securely make requests to restricted MCP servers. The specification outlines the authorization flow for HTTP-based transports, marking a significant advancement in MCP security architecture. As an exercise, we encourage all readers to go through this concept and familiarize themselves with this approach to transport-level authorization. You can refer to the following links for more details:
Authorization - Model Context Protocol

Model Context Protocol

OAuth Authentication - FastMCP
Authenticate your FastMCP client via OAuth 2.1.

FastMCP

Bearer Token Authentication - FastMCP
Authenticate your FastMCP client with a Bearer token.

FastMCP

In both LangGraph and LlamaIndex, we’ve seen that MCP prompts and resources are well-supported. As an exercise, we suggest you to explore how exactly you could use MCP prompts and resources inside agentic workflows built with LangGraph and LlamaIndex, while adhering to MCP principles.
We also strongly recommend that you consult the detailed documentation for the Agentic AI frameworks discussed in this article, especially the sections related to MCP integration. While we have covered the core concepts and integration patterns, the official documentation will offer a deeper understanding, including nuances that may not have been addressed here. Once you have reviewed the documentation and along with this chapter, we invite you to put your thoughts in the comments section on which framework would be best in the context of MCP and why. This would also enable us to understand your preference, helping us decide on the design and flow of topics to be covered in the future. Here are some reference links for the same:
Overview
Build reliable, stateful AI systems, without giving up control

logo

LlamaIndex + MCP Usage - LlamaIndex

logo

MCP Servers as Tools in CrewAI - CrewAI
Learn how to integrate MCP servers as tools in your CrewAI agents using the 
c
r
e
w
a
i
−
→
o
l
s
 library.

CrewAI

Model Context Protocol (MCP) - PydanticAI
Agent Framework / shim to use Pydantic with LLMs

logo

Conclusion
With this chapter, we moved from sandboxing and threat containment into the next layer of real-world readiness: integration.

Specifically, we explored how an MCP layer can be added within some of the most widely adopted agentic AI frameworks.

We examined four powerful frameworks: LangGraph, LlamaIndex, CrewAI, and PydanticAI, each with its own vision for how agents should think, act, and collaborate.

Despite their differences in design, each of these frameworks shares one common point: external access to structured capabilities.

Here’s a quick recap of what we covered in this part:

Before exploring the frameworks, we quickly went through some recent advancements in the realm of MCP.
We then began with a high-level comparison of the four frameworks, unpacking how each structures agents and orchestrates execution.
After that, we explored how each can integrate MCP capabilities, with real code examples and usage patterns.
We demonstrated integration across multiple transports, including HTTP, SSE, and STDIO.
Beyond wiring things together, we examined architectural implications and highlighted how MCP serves not just as a tool-calling protocol, but as a foundation for modularity and reuse in the age of intelligent agents.
Finally, we highlighted some recommendations for you to solidify your understanding.
Throughout this series, we observed that, when implemented correctly, MCP lets you build agents that are:

Interoperable, using shared standards for communication.
Decoupled, able to plug into different environments.
By using MCP as the backbone and layering frameworks like LangGraph on top, we get the best of both worlds: the flexibility of modern orchestration with the standardization and precision of a protocol.

In this series, now that we’ve covered the core principles of MCP, alongside advanced topics like sampling, security, sandboxing, and framework integration, you’re equipped to build robust, high-quality MCP-driven systems.

As always, thanks for reading!

Any questions?

Feel free to post them in the comments.

Or

If you wish to connect privately, feel free to initiate a chat here:


Connect via chat
MCP
Agents
MCP Crash Course
Share this article
Tweet
Share
Share
Email

Copy

Read next
The Full MLOps Blueprint: The Machine Learning System Lifecycle
MLOps
Aug 3, 2025
•
29 min read
The Full MLOps Blueprint: The Machine Learning System Lifecycle
MLOps and LLMOps Crash Course—Part 2.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar
The Full MLOps Blueprint: Background and Foundations for ML in Production
MLOps
Jul 27, 2025
•
22 min read
The Full MLOps Blueprint: Background and Foundations for ML in Production
MLOps and LLMOps Crash Course—Part 1.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar
The Full MCP Blueprint: Building a Full-Fledged Research Assistant with MCP and LangGraph
MCP
Jul 20, 2025
•
25 min read
The Full MCP Blueprint: Building a Full-Fledged Research Assistant with MCP and LangGraph
Model context protocol crash course—Part 9.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar
Daily Dose of Data Science
A daily column with insights, observations, tutorials and best practices on python and data science. Read by industry professionals at big tech, startups, and engineering students, across:
Navigation
Sponsor
Newsletter
More
Contact
FAQs
About
Search 🔎
©2025 Daily Dose of Data Science. All rights reserved.