Daily Dose of Data Science
Sponsor
Newsletter
More
Search 🔎
Zhuoxin Yang
Jun 8, 2025
The Full MCP Blueprint: Building a Custom MCP Client from Scratch
Model context protocol crash course—Part 3.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar

Recap
Building an MCP client
Integrating LLM into the client
Architectural overview
MCP versus API and function calling
Try out yourself
Conclusion and next steps
Recap
Before we dive further into this MCP course, let’s take a moment to reflect on what we have learned so far.

In part 1, we laid the conceptual foundation of the model context protocol (MCP). We discussed the core problem MCP solves: the M×N integration issue, where each new model-tool pairing would previously require custom glue code. MCP breaks that pattern with a unified, standardized interface.

We also thoroughly explored the MCP architecture: the Host, the Client, and the Server. This modular design lets AI assistants plug into multiple tools and data sources without manual rework.

In part 2, we got to know about capabilities, like tools, resources, and prompts, and also saw how AI models dynamically use these building blocks to reason, fetch data, perform actions, and adapt on the fly.

We understood how the client-server capability exchange works, and why MCP is a game-changer for dynamic tool integration, through hands-on examples.

If you haven’t yet gone through the previous parts, we strongly recommend doing that first before you read further. It’ll give you the necessary grounding to fully benefit from what’s coming next. Find them here:

The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part A)
Model context protocol crash course—Part 1.

Daily Dose of Data Science
Avi Chawla

The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part B)
Model context protocol crash course—Part 2.

Daily Dose of Data Science
Avi Chawla

In this part, we’ll shift our focus toward a more practical implementation and bring clarity to several key ideas and concepts covered so far in the first two parts.

By the end of this part, we will have a concrete understanding of:

How to build a custom MCP client, and not rely on prebuilt solutions like Cursor or Claude.
What the full MCP lifecycle looks like in action.
The true nature of MCP as a client-server architecture, as revealed through practical integration.
How MCP differs from traditional API and function calling, illustrated through hands-on implementations.
As always, everything will be assisted with intuitive examples and code.

Let's begin!

Building an MCP client
To truly understand how the model context protocol (MCP) works, and to integrate it into our own applications, we’re now going to build a custom MCP client.

To recap, an MCP Client is a component within the Host that handles the low-level communication with an MCP Server.

Think of the Client as the adapter or messenger. While the Host decides what to do, the Client knows how to speak MCP to actually carry out those instructions with the server.

Each MCP Client manages a 1:1 connection to a single MCP Server. If your Host app connects to multiple servers, it will instantiate multiple Client instances, one per server. The Client takes care of tasks like:

sending the appropriate MCP requests
listening for responses or notifications
handling errors or timeouts
ensuring the protocol rules are followed
It’s essentially the MCP protocol driver inside your app.

The Client role may sound a bit abstract, but a useful way to picture it is to compare it to a web browser’s networking layer.

The browser (host) wants to fetch data from various websites; it creates HTTP client connections behind the scenes to communicate with each website’s server.

👉
In many frameworks, this client functionality is provided by an SDK or library so you don’t have to implement stuff from scratch.
In our process of implementation, this client will:

Connect to an MCP server.
Forward user queries to an LLM.
Handle tool calls made by the model.
Return the final response to the user.
We’ll be using Python for our implementation, and we’ll walk through each step in detail.

👉
If you’ve completed Part 2, you’re already familiar with the setup process and have seen how an MCP server is implemented. We’ll build on that foundation here.
To implement a basic mockup MCP client (without involving an LLM yet), we’ll need to install a few core dependencies:

To keep things consistent, we’ll use the same sample server code from the previous part for our demonstration:

The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part B)
Model context protocol crash course—Part 2.

Daily Dose of Data Science
Avi Chawla

Here’s the server setup we’ll be working with (create a server.py file and add this code):

In the above code, we have created an MCP server with two tools:

get_weather, which, for the sake of simplicity, just returns a dummy response but we can make it practical by integrating a weather API.
calculate, which calculates the result of a mathematical expression.
convert_currency, which converts a given amount from one currency to another.
To run this server under sse transport, simply change the mcp.run() line to:

This transport is used for remote or long-running services, often over a network or cloud.

On a side note, you can download the code for this whole article along with a full readme notebook with step-by-step instructions below.

It contains a few Python files and a notebook file:

Download below:

MCP-client-from-scratch
MCP-client-from-scratch.zip7 KB
Once the dependencies are installed and the server has been defined, go ahead and create a new file named client_stdio.py in the same directory as that of server.py.

This will be our minimal client implementation that interacts with the MCP server over a standard input/output (stdio) stream. Add the following code inside:

This code uses asyncio, for asynchronous programming.

👉
Asynchronous programming lets a program do multiple things at once without waiting for slow tasks to finish first. Instead of stopping completely, it starts the slow task, keeps working on other tasks, and later comes back when the slow task is done. This makes programs faster and more responsive.
The entire client logic is wrapped in the async function: async def main(), to enable non-blocking operations allowing the program to handle multiple tasks concurrently.

First, we start with some basic import statements:

Next, we define our server parameters:

This creates a configuration for launching our MCP server.

The executable to run (Python interpreter) is provided by command="python", and args=["server.py"] provides the arguments passed to the command (the server script).

When executed, this effectively runs python server.py to start the MCP server.

Once the configuration is set, we establish the connection to the server via stdio and initializes a session, using nested context managers.

The first line of code starts the server using the server parameters and sets up an asynchronous connection through standard input/output streams.

stdio_client() manages the communication.

The second line, establishes a session between the client and the server over input/output streams using ClientSession(). It manages the MCP protocol communication over those streams.

👉
The nested context managers ensure proper cleanup when the connection ends.
Then, after that comes the protocol initialization:

This sends a setup request to the server performing the MCP handshake, where exchanging initial request, response and a notification happens. This establishes the protocol connection between client and server.

The client queries the server for available tools and prints them:

The above code iterates through the returned tools, printing each tool’s details. Each tool has a name (or the tool identifier) and a description (or what the tool does). The description is nothing else but the docstrings.

Beyond name and description, there are several other things. Have a look at the code below, as used in the mcp package:

Moving further, we can invoke and perform tool execution:

In a realistic scenario, which tool will be invoked is decided by the LLM. Think of it this way. When we list all the available tools:

The LLM knows the query, which in the above code, is "2+3".
The LLM also knows all the available tools (we just listed them).
Then the LLM decides which tool would be the best to invoke.
We'll get into it shortly, but to keep things simple for now, we have hard-coded that with a specific query in the demonstration above.

Moving on, once everything is set up and in place, we can finally launch the client asynchronously by adding the following at the end:

Now our client is ready to use. To launch the client, simply run the following command in the terminal/command line:

You can also run this as follows:

Check the output below:

From this output image, we can see that the client performs dynamic discovery, a key feature of the MCP architecture and design.

So overall, when we see it, the picture looks like:

Launch: Client starts and configures server parameters
Connect: The Client launches the server process and establishes communication
Initialize: MCP handshake occurs.
Discover: Client queries available tools
Execute: Client calls a specific tool with arguments (the LLM is used to decide which tool would be the best)
Cleanup: Context managers handle connection teardown
Upon keen observation, we can see that this flow is nothing but the MCP lifecycle in action with the initialization, discovery, execution, and termination phases.

The MCP interaction lifecycle has already been covered in part 2 of the course. If you need a refresher, find it here:

The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part B)
Model context protocol crash course—Part 2.

Daily Dose of Data Science
Avi Chawla

Coming back to our discussion, here, if we want to use sse mechanism, we need to remove stdio imports and add sse imports, change server connection to sse and specify the url where our server is active.

Here's a complete code that demonstrates this in action:

👉
In case of sse, before we launch our client, we need to start our server by executing the server script in a separate terminal, using uv run server.py. Also, under the main method of server.py file, ensure to run this server as sse as discussed earlier (and shown in the code below). This makes our server active and available at the specified url.
To run the server under sse transport, change the mcp.run() line to:

Next, we run the server:

To launch the client, simply run uv run client_sse.py in a terminal, similar to how we did for the stdio based client.

You can also run this as follows:

Check the output below:

Integrating LLM into the client
Now that we understand the basic implementation of the MCP client, let’s take it a step further by reimplementing and integrating it with the core component, which is the large language model (LLM).

This gives our system a "brain" of its own, enabling it to interpret user input, decide when to invoke tools, and respond intelligently, all while remaining fully decoupled from the server logic.

Pipeline and flow in an MCP-enabled application
To get started, install the necessary packages into the existing project with the following command:

We use LiteLLM because it provides a unified interface to interact with multiple LLM providers, making it easy to switch between models with minimal code changes.

For this demonstration, we’ll use GPT-4o from OpenAI, but you're free to choose any other LLM that supports tool calling and is compatible with LiteLLM.

After this, create a .env file in your directory and store your OpenAI API key here:

Next up, we write the new client code.

To begin, create a new file client_stdio_with_llm.py we add the necessary imports and load our API key from the .env file using the dotenv package. This sets up the environment for securely accessing the LLM. Here's how to get started:

In the above code:

AsyncExitStack (from contextlib) is a helper that allows us to open multiple asynchronous context managers and then close them all in one call. We’ll use it to keep track of both the stdio_client stream (the transport connecting us to MCP), and the ClientSession itself. When we call exit_stack.aclose(), everything is torn down in reverse order.
Moving forward, we define a few global variables to help with managing session state throughout:

In the above code:

session (initially None): Stores the MCP client session for tool communication. Once we connect to the MCP server via connect_to_server(), we’ll assign a live ClientSession object here.
exit_stack: Manages the lifecycle of async resources.
Moving ahead, we define a function to connect to the server:

So now let’s break down what exactly is happening here:

In line 6, with server_params = StdioServerParameters(...), we implement the idea that when I create a stdio_client, it should run the command python server.py inside a subprocess. This subprocess is expected to run the MCP server and the file contains the implementation of the server.
In line 8, we actually launch the subprocess under the hood. The stdio_client(...) co-routine yields a pair (read_stream, write_stream) connected to that subprocess’s stdin/stdout. We saw that earlier. We store that pair in stdio_transport.
In line 10, we unpack stdio, write = stdio_transport. stdio is a reader interface (what we read from, i.e., what the MCP server prints out) and write is a writer interface (what we write into, i.e. what the MCP server receives).
Next, in line 12, we wrap the raw stdio streams into an mcp.ClientSession. Note that we register this ClientSession(...) with the same exit_stack. This way, when we eventually call await exit_stack.aclose(), it will close the session and then kill the underlying subprocess.
In line 14, as soon as everything is setup, we can complete the handshake (the MCP protocol requires an "initialize" RPC call right after opening.
Next, we perform tool discovery and format conversion to inspect the server’s capabilities and present them in a compatible format:

We begin by asking the server about the tools it offers via await session.list_tools() and print them out (similar to what we did earlier).

LiteLLM (and OpenAI’s “function-calling” API) expects each tool/function to be provided in a specific JSON format. Concretely, here we have specified that format by implementing a simple helper.

Here’s what this helper does:

For each tool in tools_result.tools, we create a new dictionary with keys "type": "function" and a nested "function": {...} object.
We copy over tool.name as the function’s "name", tool.description as the function’s "description", and the tool.inputSchema as the function’s "parameters".
We return a list of these formatted dictionaries.

Now comes the heart of the MCP workflow, where we must define the central query processing logic. This part that receives a user query, sends it to the LLM, observes any tool calls, asks for permission, invokes the corresponding server tool, and returns the final answer.

Have a look at the code below:

Given the complexity of the code, let’s break it into three phases.

In the first phase, we perform tool decision (first pass):

This section of code:

Gets all the tools using the get_mcp_tools method defined earlier.
Sends the user query to the language model while providing all the available tools as options.
Uses tool_choice="auto" to let the model decide if tools are even needed to answer the query.
Then we move to the second phase.

We gather the response of the LLM and create a messages object:

Then we check if the LLM wanted to invoke any tools:

If tools are required, the invocation logic goes inside the for-loop declared above (for call in calls_list):

Next up, recall from the MCP flow below:

We have just completed step 3 above and now we need to move to Step 4, which is user's approval. We do that below:

If the user permits the tool call, we invoke the tool, get the output and append it to the message history:

If the user declines the tool call, we append a "tool call denied" message to the message history:

This process runs for all the tools that the LLM wanted to invoke.

To summarize the logic outlined above for the second phase:

Extracts the response from the LLM.
Detects a tool-call request. We check calls = assistant_message.tool_calls or assistant_message.function_call, depending on the LiteLLM version. If the model did not request any tool, both of these will be None, so calls becomes None and we skip into the “no tools needed” branch.
Parses tool name and arguments.
Asks the user for permission. We ask: “Okay, the LLM wants to call tool_name with these arguments. May I actually run that tool?” This is a “human-in-the-loop” safety check.
Calls and executes the tool, if permission granted. Appends a “Tool” message to the history. By giving it role="tool", we signal to LLM that “This piece of text is the output of a tool we just ran.”
If permission denied, we simply append a dummy “tool” message saying “This tool call was denied.” Downstream (in the second pass), LiteLLM will see that the tool result is this denial string, and might respond accordingly.
Following the invocation of all tools, the LLM performs a second pass in the third phase to synthesize the final response:

Here we:

Send the complete conversation (including tool results) back to the model.
Use tool_choice="none" to ensure a final text response, without making any tool calls.
After this whole process has taken place, the function returns the response based on whether tools were invoked or not.

Now that we have our central logic in place let's give a final touch by ensuring that the session closes properly when the script exits:

This code:

Simply closes everything that was registered in the exit_stack:
First, it will close the ClientSession.
Then it will kill the underlying subprocess (the MCP server that was launched with python server.py).
Ensures that resources don’t linger.
Finally, we code our main() function where we test our client-server architecture:

In the above code:

We defined a query
Connected to the server while specifying the file which has all the tools available.
We used the process_query method defined above to generate a response.
Finally, finally closed the connection.
We complete our code with:

Putting it all together, here is what happens when we run the script:

connect_to_server("server.py"), launches python server.py and completes the MCP handshake.
With process_query(...), we make the first LiteLLM call to decide on tools, detect if the tools are needed or not, parse arguments, take permission, call the tools, add tool results to messages, make the second LiteLLM call for final response and return the results.
Perform cleanup using await cleanup().
Running the above file (using uv run client_stdio_with_llm.py or python client_stdio_with_llm.py), we get the following output:

💡
Make sure server.py has mcp.run() instead of mcp.run(transport="sse", host="127.0.0.1", port=8000).

Here, we can see the LLM in action, performing reasoning to decide upon multiple tools, then the client asks for user permission for each tool and finally a grounded response is returned based on tool results.

On a side note, this pattern, often referred to informally as the "two-pass conversation" workflow, is widely adopted in modern AI pipelines.

In the first pass, the model receives the user query and determines whether it needs to invoke a specific function or tool to generate a proper response. If so, it returns a formal function call rather than a direct answer.

In the second pass, once the tool is executed and its result is available, the model is prompted again, this time with the tool’s output, allowing it to generate a final, grounded response.

👉
This two-pass mechanism enables dynamic, tool-augmented intelligence while keeping the interaction modular and interpretable.
This above demonstration was for an stdio connection.

In case of sse transport, create a file client_sse_with_llm.py and start with some standard import statements:

The get_mcp_tools and the process_query methods will stay exactly the same.

Next up, we have the following:

We now switch from using the stdio_client to the sse_client for establishing a connection.

Unlike the previous setup, we no longer need AsyncExitStack.

Instead, we rely on nested asynchronous context managers, which, as discussed earlier, also handle proper cleanup automatically.

Apart from this change in transport mechanism and session management, the rest of the code largely remains the same.

Note that the implementations presented thus far have focused exclusively on tools. However, similar programmatic principles apply equally to prompts and resources.

The list_resources() and list_prompts() instance methods can be utilized in a manner analogous to list_tools().

Architectural overview
At its core, MCP operates as a client-server system, where:

The server implements and exposes a set of tools
The client discovers these tools and acts as a controller, coordinating LLMs and deciding when to invoke which tools.

Let's demonstrate via the stdio based server-client implementation, as a reference.

Here are the roles in the implementation

server.py:
Role: MCP Server
Description: Implements tools using @mcp.tool() decorators. Responds to tool calls from clients over transport.
ClientSession + stdio_client:
Role: MCP Client
Description: Connects to the server subprocess, initializes the session, lists available tools, and dispatches tool calls.
litellm.acompletion(...):
Role: LLM (brain/reasoning)
Description: Acts as the "brain" that decides what tool to call, using natural language and function-calling schema.
process_query(...):
Role: Client Orchestration Logic
Description: Implements the tool reasoning loop: sends queries to LLM, takes permission, executes tools if requested, and provides results back to the LLM.
This client-server interaction follows a typical request-response protocol, as discussed above and again shown below:

Simplified MCP communication flow
The client launches the server via subprocess:

Then wraps the server's I/O in a ClientSession. After that, await session.initialize() completes the handshake.

The client requests for available tools.
The server responds with a list of available tools, including their names, descriptions, and input schemas.

When the LLM decides a tool is needed, the client, post-permission, sends a tool call request to the server using call_tool instance method.
The server executes the corresponding tool, and replies with the result.

All requests and responses occur within a live ClientSession, which is gracefully closed using await exit_stack.aclose().

Note that we register this ClientSession(...) with the same exit_stack. That way, when we eventually call await exit_stack.aclose(), it will close the session and then kill the underlying subprocess.

Based on the above explanation, we can see that our implementations appropriately fit the definition of a typical client-server architecture.

MCP versus API and function calling
As also discussed in the previous part, while APIs and function calling have long served as mechanisms for integrating software systems, MCP introduces a new paradigm tailored for interoperability with tools, data, and prompts:

The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part B)
Model context protocol crash course—Part 2.

Daily Dose of Data Science
Avi Chawla

Here, in this part, we'll concretize the differences on the basis of our implementations of client and server.

Below is a comparative overview of how MCP differs fundamentally from both approaches.

MCP versus API
In terms of flexibility:
APIs offer Rigid contracts (hardcoded parameters).
MCPs enable dynamic discovery and adaptation to server-side changes.
In terms of change management:
With APIs, client code breaks on schema changes.
MCPs let the clients auto-adapt.
To elaborate further, traditional APIs require clients to hardcode request formats.

Any change in API parameters (e.g., adding a new required field) breaks compatibility unless the client updates its code.

In contrast, MCP clients query the server for available tools and their schemas dynamically, enabling zero-code adaptation.

After updates to parameters:

MCP versus function calling
In terms of Integration:
Traditional function calling is also hardcoded and app-specific.
MCPs are decoupled and protocol-driven.
In terms of scalability:
Traditional function calling involves M×N manual mappings.
MCPs offer M+N; which is scalable through dynamic discovery and standardization
In terms of code maintenance:
Traditional function calling demands updates per function instance.
MCPs allows tool updates to be automatically propagated to the client.
In terms of reusability:
Traditional function calling offer Low reusability since the logic is tightly coupled to app context.
MCPs offer High reusability since ttools are abstracted from usage.

Function calling enables LLMs to invoke specific functions based on prompts, but lacks a standard for how those functions are defined, described, or discovered.

MCP solves this by standardizing the lifecycle and interface, enabling seamless, modular integration of evolving tools.

Demonstrating via code
Let's solidify the above with a code demonstration.

If we go ahead and modify, replace, add or remove a tool in our server.py file, the client-side logic remains untouched.

Without changing a single line of client-side logic, we can query the updated system and receive accurate responses.

The client script is designed to dynamically adapt to the available tools.

To add a new tool, simply annotate it with @mcp.tool() in server.py, re-execute the server/client programs, and the client’s get_mcp_tools() method will automatically discover and reflect the updated toolset. Similarly we can modify and remove tools.

For instance, let's add a random new tool called web_search (mockup) and modify the get_weather tool to additionally accept a time argument:

Now update the query, then re-execute the scripts and observe the output. You'll observe that everything works perfectly without any issues.

This is something that MCP has achieved, which APIs and traditional function calling setups lack.

Check out the output screenshot below:

Here we can observe that although we did not change any logic on the client, the client automatically fetches the updated tool definitions and our MCP system is also working perfectly fine.

This dynamic extensibility is a core strength of MCP. It enables modular, declarative integration that decouples system evolution from client logic.

This results in a more maintainable, scalable, and developer-friendly workflow for building robust AI-agent systems.

Try out yourself
Modify the existing implementations of clients (both stdio and sse) to accept user queries dynamically from the terminal using input(), instead of hardcoding them.
Implement a chat loop so that after answering one query, the user can continue asking more without the script exiting. The loop should persist until explicitly terminated by a predefined keyboard key/shortcut.
Add a keyboard shortcut (e.g., Ctrl + F) that enables the user to refresh the connection and restart the entire MCP lifecycle at any point during script execution, without needing to stop or re-run the script. This feature mirrors Cursor’s refresh or restart functionality and provides a seamless way to reinitialize the client, especially when server-side tools or capabilities have been modified. A quick refresh ensures the client is immediately synchronized with the updated server state and reflects the latest available toolset.
(Optional) While we have used the mcp Python package to implement our client so far, now that we have a solid understanding of how MCP orchestrates interactions, try building a minimal MCP client, supporting only tools and excluding LLM integration, without relying on the mcp package. This exercise will help you appreciate the finer details of the MCP architecture and lifecycle, including the request structure, session handling, and tool invocation logic.
Hint: You’ll primarily use the requests python module for this.
Conclusion and next steps
With Part 3, we’ve moved one step ahead and built upon some of the key theoretical concepts by actually implementing them into constructing a fully functional custom MCP client of our own.

This practical walkthrough not only demystified the client-server nature of MCP but also showcased its power as a standardized interface for seamless tool integration.

By practically observing the full lifecycle, from connection initialization to tool discovery and execution, we've seen how MCP enables dynamic and modular AI interactions without the overhead of tightly coupled APIs.

This architecture allows developers to scale, extend, or replace tools effortlessly, making AI systems more adaptable and robust.

This hands-on walkthrough demonstrated the technical simplicity and modularity MCP offers. With decoupled tool management, clean extensibility, and a developer-friendly workflow, MCP significantly reduces integration friction.

If Part 1 introduced MCP and the motivation behind MCP, and Part 2 explored the lifecycle, protocol design and server-side design, then Part 3 solidified things by showing what it takes to build and run a client system that speaks MCP.

Together, these chapters provide a fundamental and comprehensive overview of MCP, not just as a protocol, but as a transformative design pattern for AI systems that need to reason, act, and interface with the world around them.

Looking ahead, upcoming chapters will dive into:

Using prompts and resources alongside tools
Advanced sampling workflows
Sandboxing
Testing and debugging
Real-world deployments and integrations
As always, thanks for reading!

Any questions?

Feel free to post them in the comments.

Or

If you wish to connect privately, feel free to initiate a chat here:

Connect via chat
Agents
MCP
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
