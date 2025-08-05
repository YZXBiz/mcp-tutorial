
Daily Dose of Data Science
Sponsor
Newsletter
More
Search 🔎
Zhuoxin Yang
Jun 1, 2025
The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part B)
Model context protocol crash course—Part 2.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar

Recap from Part A
Before we dive in, let’s briefly recap what we covered in Part 1 of this MCP crash course.

In Part 1, we laid the groundwork for understanding the Model Context Protocol (MCP). We began by exploring the evolution of context management in LLMs, from early techniques like static prompting and retrieval-augmented generation to more structured approaches like function calling and agent-based orchestration.


We then introduced the core motivation behind MCP: solving the M×N integration problem, where every new tool and model pairing previously required custom glue code.

Finally, we unpacked the MCP architecture, clarifying the roles of the Host (AI app), Client (protocol handler), and Server (tool provider) and explained how they interact through a standardized, modular system.


An illustration of MCP’s architecture: The Host (AI application) contains an MCP Client component for each connection. Each Client talks to an external MCP Server, which provides certain capabilities (tools, etc.). This modular design lets a single AI app interface with multiple external resources via MCP.
Now, in Part 2, we’ll build upon that foundation and get hands-on with MCP capabilities, communication protocols, and real-world implementation.

If you haven't read the previous part yet, we highly recommend doing so before reading ahead:

The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part A)
Model context protocol crash course—Part 1.

Daily Dose of Data Science
Avi Chawla

Let's dive in!

Capabilities in MCP
An MCP Server can expose one or more capabilities to the client.

Capabilities are essentially the features or functions that the server makes available. The MCP standard currently defines four major categories of capabilities:

Tools → Executable actions or functions that the AI (host/client) can invoke (often with side effects or external API calls).
Resources → Read-only data sources that the AI (host/client) can query for information (no side effects, just retrieval).
Prompts → Predefined prompt templates or workflows that the server can supply.
Sampling → A mechanism through which the server can request the AI (client/host) to perform an LLM operation, typically used to enable more advanced, multi-step interactions. This feature supports capabilities such as self-reflection and iterative reasoning. Sampling will be explored in greater depth in later parts of this course.
Let’s break down each in detail:

Tools (actions)
Tools are what they sound like: functions that do something on behalf of the AI model. These are typically operations that can have effects or require computation beyond the AI’s own capabilities.

Importantly, Tools are usually triggered by the AI model’s choice, which means the LLM (via the host) decides to call a tool when it determines it needs that functionality.

Because tools can change things or call external services, they often are subject to safety checks (the system may require the user to approve a tool invocation, especially if it’s something sensitive like sending an email or executing code).

Suppose we have a simple tool for weather. In an MCP server’s code, it might look like:


This Python function, registered with @mcp.tool(), can be invoked by the AI via MCP.

When the AI calls tools/call with name "get_weather" and {"location": "San Francisco"} as arguments, the server will execute get_weather("San Francisco") and return the dictionary result.

The client will get that JSON result and make it available to the AI. Notice the tool returns structured data (temperature, conditions), and the AI can then use or verbalize (generate a response) that info.

Since tools can do things like file I/O or network calls, an MCP implementation often requires that the user permit a tool call.


For example, Claude’s client might pop up “The AI wants to use the ‘get_weather’ tool, allow yes/no?” the first time, to avoid abuse. This ensures the human stays in control of powerful actions.

Tools are analogous to “functions” in classic function calling, but under MCP, they are used in a more flexible, dynamic context.

They are model-controlled (the model decides when to use them) but developer/governance-approved in execution.

Resources
Resources provide read-only data to the AI model.

These are like databases or knowledge bases that the AI can query to get information, but not modify.

Unlike tools, resources typically do not involve heavy computation or side effects, since they are often just information lookup.

Another key difference is that resources are usually accessed under the host application’s control (not spontaneously by the model). In practice, this might mean the Host knows when to fetch certain context for the model.

For instance, if a user says, “Use the company handbook to answer my question,” the Host might call a resource that retrieves relevant handbook sections and feeds them to the model.

Resources could include a local file’s contents, a snippet from a knowledge base or documentation, a database query result (read-only), or any static data like configuration info.

Essentially anything the AI might need to know as context. An AI research assistant could have resources like “ArXiv papers database,” where it can retrieve an abstract or reference when asked.

A simple resource could be a function to read a file:


Here we use a decorator @mcp.resource to indicate a resource to the MCP server.

Notice that resources are usually identified by some identifier (like a URI or name) rather than being free-form functions.

They are also often application-controlled, meaning the app decides when to retrieve them (to avoid the model just reading everything arbitrarily).

From a safety standpoint, since resources are read-only, they are less dangerous, but still, one must consider privacy and permissions (the AI shouldn’t read files it’s not supposed to).

The Host can regulate which resource URIs it allows the AI to access, or the server might restrict access to certain data.

In summary, Resources give the AI knowledge without handing over the keys to change anything.

They’re the MCP equivalent of giving the model reference material when needed, which acts like a smarter, on-demand retrieval system integrated through the protocol.

Prompts
Prompts in the MCP context are a special concept: they are predefined prompt templates or conversation flows that can be injected to guide the AI’s behavior.

Essentially, a Prompt capability provides a canned set of instructions or an example dialogue that can help steer the model for certain tasks.

But why have prompts as a capability?

Think of recurring patterns: e.g., a prompt that sets up the system role as “You are a code reviewer” and the user’s code is inserted for analysis.

Rather than hardcoding that in the host application, the MCP server can supply it.

Prompts can also represent multi-turn workflows.

For instance, a prompt might define how to conduct a step-by-step diagnostic interview with a user. By exposing this via MCP, any client can retrieve and use these sophisticated prompts on demand.

As far as control is concerned, Prompts are usually user-controlled or developer-controlled.

The user might pick a prompt/template from a UI (e.g., “Summarize this document” template), which the host then fetches from the server.

The model doesn’t spontaneously decide to use prompts the way it does tools.

Rather the prompt sets the stage before the model starts generating. In that sense, prompts are often fetched at the beginning of an interaction or when the user chooses a specific “mode”.

Suppose we have a prompt template for code review. The MCP server might have:


This prompt function returns a list of message objects (in OpenAI format) that set up a code review scenario.

When the host invokes this prompt, it gets those messages and can insert the actual code to be reviewed into the user content.

Then it provides these messages to the model before the model’s own answer. Essentially, the server is helping to structure the conversation.

While we have personally not seen much applicability of this yet, common use cases for prompt capabilities include things like “brainstorming guide,” “step-by-step problem solver template,” or domain-specific system roles.

By having them on the server, they can be updated or improved without changing the client app, and different servers can offer different specialized prompts.

An important point to note here is that prompts as a capability blur the line between data and instructions.

They represent best practices or predefined strategies for the AI to use.

In a way, MCP prompts are similar to how ChatGPT plugins can suggest how to format a query, but here it’s standardized and discoverable via the protocol.

In the next section, we’ll get into the actual communication protocol that underpins all these interactions.

We’ve talked conceptually about requests, responses, etc., but now let’s see the format and transport details that make MCP communications possible.

The MCP communication protocol
To ensure all these components and capabilities interact smoothly, MCP defines a standard message format and conversation pattern.

It uses a tried-and-true foundation called JSON-RPC 2.0 for message structure.

👉
JSON-RPC is a lightweight remote procedure call protocol that encodes calls and responses in JSON format. It’s perfect for this scenario because it’s simple, human-readable, and language-agnostic.
Why JSON-RPC?
It provides just what we need: a way to send a request with a method name and parameters, and to get back a response (or an error), all expressed in a standard JSON format.

Many existing libraries and tools support JSON-RPC, and it’s much simpler than other stuff available out there. By building on JSON-RPC, MCP didn’t have to reinvent the wheel for basic messaging.

Message types
There are three types of messages in MCP’s JSON-RPC-based protocol.

Request
Sent by the Client to the Server to ask it to do something (e.g., call a tool, list resources). A Request includes:

A unique id (so we can match the response to this request).
A method name (a string like "tools/list" or "tools/call", and these method names are part of MCP’s spec).
params (a JSON object with any parameters needed for that method).
Suppose the client wants the server to execute a tool named “weather”. The request might look like this:


This asks the server: “Please call the tool named ‘weather’ with the argument location = San Francisco.” The id: 1 tags this call.

Response
Sent by the Server back to the Client as a reply to a Request.

A Response includes:

The same id as the corresponding request (to match them).
EITHER a result (if the request succeeded) OR an error (if something went wrong).
Here's a sample success response:


This indicates request id 1 (our weather call) succeeded, and here is the result data (62°F and partly cloudy).

Here's what a sample error response could look like:


Here, perhaps the tool didn’t recognize the location given, so it returns a standard JSON-RPC error with a code and message.

The client would see this and handle it (maybe by telling the user the location was invalid).

Notification
A message that does not expect a response (thus no id).

Notifications are typically sent from the Server to Client to convey asynchronous events or updates.

For example, progress updates, or a signal that something happened on the server side.

Here's a sample notification:


This could be sent by the server while working on a long task, telling the client “I’m 50% done processing data…”. The client doesn’t reply to notifications; it may just show this info to the user or log it.

These three message types cover all the needed patterns. The methods (tools/call, tools/list, resources/list, shutdown, etc.) are part of MCP’s defined API. The combination of Requests, Responses, and Notifications orchestrates the interaction.

Transport mechanisms

Now, how do these JSON messages actually travel between client and server?

MCP supports two primary transport mechanisms:

Stdio (standard input/output)
This transport is used when the Server runs as a local subprocess on the same machine as the Client/Host.

Essentially, when the host launches the server, it connects to the server’s stdin and stdout streams. The client writes JSON lines into the server’s stdin (Requests) and reads JSON lines from the server’s stdout (Responses/Notifications).

This is a simple and effective method for local tools.

It doesn’t require any network setup or ports, and the operating system ensures isolation (the server can be run with limited permissions, etc.).

Many MCP servers (especially those that act like local plugins, e.g., file system tools) use stdio by default.

HTTP + SSE (server-sent events)
This transport is used for remote or long-running services, often over a network or cloud.

It operates via HTTP: the Client sends Requests as HTTP POST requests to the server’s URL, and the Server replies using Server-Sent Events (SSE) to stream responses and notifications back over a persistent HTTP connection.

👉
SSE is a mechanism that keeps an HTTP response open and allows the server to push multiple messages (it’s great for streaming partial results or progress without needing WebSockets).
In MCP, the SSE channel is typically used to send any number of notifications and eventually the final response.

MCP’s design even allows servers to upgrade to SSE when needed (called streamable HTTP). For instance, a server might start sending a normal HTTP response, but if it sees that it will have multiple messages, it can switch to SSE mode mid-connection to keep the stream alive.

Interaction lifecycle
We’ve covered message types and transports; now let’s outline the typical lifecycle of an MCP connection from start to finish, incorporating some of the handshake details mentioned earlier:


The MCP lifecycle
Initialization phase

When a client first connects to a server, there’s an initial handshake. The client sends an initialize request, which includes information like the protocol version it supports, maybe the client’s identity or preferences.

The server responds (response to initialize) confirming connection and stating its supported protocol version and maybe server info.


Then the client often sends an initialized notification to signal that it’s ready. This ensures both sides agree on basics like protocol version.

It’s also where a server might send over any “welcome” info or require authentication tokens if needed (depending on implementation).

Discovery phase

Next, the client typically inquires about capabilities. It may call methods like tools/list, resources/list, prompts/list sequentially. Each of those is a request that the server answers with a list of what it has.

For example, tools/list might return a list of tool names along with descriptions and argument schemas. This allows the Host to know what’s available.

To learn the functionality of every available capability, the docstring and function signature is utilized, so that the client LLM knows how exactly everything works and how and when to utilize them. They basically serve as tool description and argument schema.

Below is a basic example of how one can obtain function signatures programmatically using the inspect package:


In some cases, the Host might skip explicit discovery and directly attempt to call a known tool, but general discovery is useful, especially if the host wants to automatically enable the AI to choose tools. The client might cache these results for the session.

Execution phase

Now the client (on behalf of the AI’s reasoning) makes use of the capabilities. It sends specific requests like tools/call (to invoke a tool), or if using resources, maybe resources/read with a resource identifier, etc., depending on the action. The server executes and returns responses accordingly.

During this phase, the server might also send notification messages back (e.g., progress updates as shown earlier, or other events). The client continues to listen and process those (for example, updating a progress bar in the UI).

If the AI triggers multiple actions in a sequence, this could be a series of calls. For long-running actions, the SSE transport shines here: the server can send multiple progress notifications (like every 10% completion) and then a final result.

Termination phase

When the session or usage is over, the client will close things down gracefully. This typically involves a shutdown request from client to server, which the server responds to acknowledging.

Then the client may send an exit notification as a final signal, and then actually terminate the connection or subprocess. Proper termination ensures no processes are left hanging and resources (like file handles or network sockets) are cleaned up.

If the user simply closes the application or aborts, the Host should try to perform this shutdown handshake, but if it can’t, most servers will also handle sudden disconnections gracefully.

Overall, throughout this lifecycle, MCP’s protocol is designed to be extensible and robust. The initialization’s version negotiation means as MCP evolves, older clients/servers can still talk if they both agree on a common version.

The discovery mechanism means a smart client can adapt to different servers (some might have advanced tools, some only basic ones). The request/response pattern ensures synchronous actions get results, while notifications allow async updates.

It’s a well-thought-out flow that balances structure with flexibility.

Hands-on examples
To solidify understanding, let’s walk through two concrete examples of MCP in action with Python code, using FastMCP.

We’ll create a minimal MCP server that offers a simple tool and resource, with Cursor and Claude desktop as host/client.

👉
FastMCP is a Pythonic framework that simplifies the creation of MCP servers and clients. It offers features like full client support, server composition, and integration with FastAPI.
Setup
Firstly, you needs to download and set up Cursor and Claude desktop on your system and ensure your python version is 3.8 or higher.

https://www.cursor.com/
https://claude.ai/
Then setup uv package manager:


Moving on, the next step is project initialization:


Then move inside the project directory:


Initialize a virtual environment:


Activate the virtual environment:


Install the FastMCP package:


Done!

Server program
Now create a file server.py and add the code below:


This Python script sets up a sample MCP server using the FastMCP library. It defines a server called Sample Server and exposes three tools that an AI (like Claude or Cursor) can invoke through the MCP protocol.

Let's look a it step by step.


,

This creates an MCP server named "Sample Server".
The server will handle incoming tool calls over MCP.

A simple mock tool that returns hardcoded weather data for any location.
Example usage: get_weather("Delhi") returns "Weather in Delhi: Sunny, 72°F"
Similarly, the calculate and convert_currency tools have been defined.
calculate evaluates a math expression using Python's eval().
Example: calculate("2 + 3 * 4") → 14.0
👉
Caution: eval() is dangerous in production; it should be sandboxed or replaced with a parser like ast.literal_eval.
convert_currency, also a sample tool, converts amount between two currencies using hardcoded exchange rates.
Example: convert_currency(100, "USD", "INR") → "100 USD = 8300.00 INR"

Launches the MCP server and starts listening for tool invocation requests (from Claude, Cursor, or a custom client).
Uses stdio transport mode. For sse simply change the mcp.run() line to:


Connect to Cursor/Claude

For Cursor go to settings and then MCP tab, then click on "Add new global MCP server" and put the below configuration into the mcp.json file:

For stdio:


For sse:


Note: For sse transport, one would need to run the server with uv run server.py before one could start interacting with it.

Once this is done, the server successfully starts and gets connected to Cursor as client and host.


In the chat area, we can now interact with our MCP tools via relevant queries.


Similarly, for Claude Desktop, go to settings and then "Developer" tab. Click on "Edit Config" and then inside claude_desktop_config.json add the same stdio configuration as shown earlier.

As of now, Claude does not support MCP servers via sse mechanism.

Then, one can see the MCP tools listed inside the available tools button present on the chat interface of Claude Desktop.


An interesting example
Even with all the above implementations, we can still make things easier as developers who are using MCP servers.

See, the issue is that when you try to handle a multi-step request like “get the latest flight prices from X to Y on Z,” you often end up hardcoding everything into a single function, which includes parsing, validation, fallback defaults, and more.

This bloats your logic and makes the tool brittle.

But with an LLM on the client side and MCP’s tool chaining, you can break the task into smaller, well-documented tools, like one for parsing, one for validation, one for lookup, and let the LLM orchestrate the sequence intelligently.

This provides cleaner tools, better reuse, and zero need for procedural glue logic.

The below server example demonstrates the ability to leverage LLM on the client's end, interdependence of tools, importance of a clear and explicit docstring, and elimination of loads of explicit code by smart tool use with MCP.


In the above example we have implemented a mockup application that uses MCP for flight booking. Now, before proceeding further, let's take a look at the output of the above code.

Given Query: "I want to book a flight from bangalore to delhi on 25th june, 2025. name avi chawla and email avi123@gmail.com book the cheapest... use mcp tools"

The first tool called is search_flights:


👉
Notice, how the LLM automatically converts information from our query into the intended format, and we did not need to implement any logic for that.
Then, sequentially, book_flight is called:


👉
The LLM automatically takes relevant portions of the previous result and passes as input to the book_flight tool, without us having to implement the parser.
Finally, the send_confirmation tool is invoked:


Finally, the result is given by the client LLM:


Overall, the entire flow was:


From the code and the result we obtained, we can see that we had not written any explicit parser or used regex to obtain relevant portions of query or intermediate tool results to be passed on over to the next tool.

The LLM on the client's end automatically did that based on its own reasoning, which is extremely powerful yet not as obvious as it appears.

Neither the sequence of tool calling was determined beforehand.

The LLM, on its own discretion, decided the tool sequence on the fly.

In this case, it logically followed the workflow: search_flights → book_flight → send_confirmation. Additionally, we can see, based on the instructions in the prompt and its own intelligence, the LLM automatically selected the flight with the lowest price from the available options, without requiring us to write any logic for that.

Overall, we were saved from implementing explicit logics, which otherwise would have been necessary. This is a very key area where we can see how smart tool use with MCP is saving us from a lot of effort.

Although this might appear trivial, it underscores two significant points.

First, if we had combined all functionality into a single monolithic tool, we would have needed to implement parsing and sequencing logic within that tool's code. According to MCP design principles, the LLM at the client side can only provide inputs and receive tool outputs so it cannot intervene during a tool's execution. This means all conditional logic and data processing would need to be coded within the tool itself.

Second, if we had used tool calling approaches such as custom API integrations, hardcoded tool chains, or non-standardized implementations, we might still have needed to explicitly handle all parsing and sequencing logic internally, depending on the implementation pattern.
In either of these situations, the system would have been rigid in addressing varying user needs. Providing preferences or instructions through natural language prompting, as effectively done with MCP's standardized multi-tool approach, would not have been sufficient to change the system's behavior.

👉
Breaking functionality into smaller, composable tools allows the client-side LLM to manage intermediate outputs and make decisions dynamically. This approach enables the system to adapt to different user preferences expressed through natural language, without requiring code changes for each variation.
The granular tool design allows for much greater flexibility.

For instance, a user could request to book the flight of a specific airline or something like "find flights under $300" and the LLM would adapt its tool selection and sequencing criteria accordingly, all without modifying the underlying tools.

Also, observe that we specified formats in the docstrings of the tools. As discussed earlier, the client becomes aware of the server's capabilities based on the function signature and the corresponding docstring.

We can see the LLM formed inputs to tools as per specifications in the docstrings. Hence, it is quite important to write clear and explicit docstrings.

👉
For larger, more capable models like GPT-4 or Claude Sonnet, concise docstrings with good function signatures are often sufficient. However, explicit docstrings with detailed examples become crucial when working with smaller local models that may need more guidance to understand expected formats and behaviors. As a best practice, always write clear, detailed docstrings with examples, regardless of which LLM will be used as the client.
Miscellaneous concepts
Below is a concise yet important explanation of certain common doubts one faces while learning about MCPs.

API versus MCP
APIs (Application Programming Interfaces) and MCPs are both used for communication between software systems, but they have distinct purposes and are designed for different use cases.

APIs are general-purpose interfaces for software-to-software communication, while MCPs are specifically designed for AI agents to interact with external tools and data. 

Key differences:

MCPs aim to standardize how AI agents interact with tools, while APIs can vary greatly in their implementation. 
MCPs are designed to manage dynamic, evolving context, including data resources, executable tools, and prompts for workflows. 
MCPs are particularly well-suited for AI agents that need to adapt to new capabilities and tools without pre-programming. 
In a traditional API setup:

If your API initially requires two parameters (e.g., location and date for a weather service), users integrate their applications to send requests with those exact parameters.

Later, if you decide to add a third required parameter (e.g., unit for temperature units like Celsius or Fahrenheit), the API’s contract changes.

This means all users of your API must update their code to include the new parameter. If they don’t update, their requests might fail, return errors, or provide incomplete results.

MCP’s design solves this as follows:

For instance, when a client (e.g., an AI application like Claude Desktop) connects to an MCP server (e.g., your weather service), it sends an initial request to learn the server’s capabilities.
The server responds with details about its available tools, resources, prompts, and parameters. For example, if your weather API initially supports location and date, the server communicates these as part of its capabilities.

If you later add a unit parameter, the MCP server can dynamically update its capability description during the next exchange. The client doesn’t need to hardcode or predefine the parameters since it simply queries the server’s current capabilities and adapts accordingly.

This way, the client can then adjust its behavior on-the-fly, using the updated capabilities (e.g., including unit in its requests) without needing to rewrite or redeploy code.
We’ll understand this topic better in Part 3 of this course, when we build a custom client and see how it communicates with the server.

MCP versus function calling
Before MCPs became mainstream (or popular like they are right now), most AI workflows relied on traditional function calling for tools.

Here’s a visual that explains Function calling & MCP:


Function calling enables LLMs to execute predefined functions based on user inputs. In this approach, developers define specific functions, and the LLM determines which function to invoke by analyzing the user's prompt. The process involves:

Developers create functions with clear input and output parameters.
The LLM interprets the user's input to identify the appropriate function to call.
The application executes the identified function, processes the result, and returns the response to the user.
But there are limitation as well:

As the number of functions grows, managing and integrating them becomes complex. Requires M×N integrations.
Functions are closely tied to specific applications, making reuse across different systems challenging.
Any changes require manual updates across all instances where the function is used.
MCP, offers a standardized protocol for integrating LLMs with external tools and data sources. It decouples tool implementation from their consumption, allowing for more modular and scalable AI systems.

Comparative overview
Aspect	Traditional Function Calling	Model Context Protocol (MCP)
Integration	Manual and application-specific	Standardized and reusable across platforms
Scalability	Limited by manual management	High, due to dynamic discovery and standardization
Maintenance	Requires manual updates in each application	Centralized updates propagate across systems
Flexibility	Rigid, with tight coupling	Flexible, with decoupled architecture
Security & Compliance	Basic, depends on individual implementations	Built-in support for audit and approval workflows
While traditional function calling provides a straightforward method for integrating LLMs with specific functions, it falls short in scalability and flexibility. MCP addresses these challenges by offering a standardized, decoupled approach that promotes interoperability and maintainability.

The dynamic discovery of capabilities ensures that any changes (in available capabilities or schemas of existing capabilities) at the server end are automatically reflected at the client side, without requiring any changes in codes at the client's end.

Adopting MCP can lead to more robust and adaptable AI systems, especially as the complexity and number of integrated tools grow.


Conclusion and next steps
With Part 2, we’ve completed our deep dive into the Model Context Protocol, not just in theory, but in practice.

We started with a close look at MCP’s capabilities: how tools, resources, and prompts can be exposed by a server and used by AI models through a structured, standardized interface.

We then walked through the MCP communication protocol, why it’s built on JSON-RPC, the different message types it supports (requests, responses, notifications), and how communication flows through transport mechanisms like stdio and HTTP + SSE.

We also explored the complete interaction lifecycle of MCP, such as initialization, discovery, execution, and termination, so you can reason about how end-to-end orchestration works.

Finally, we brought it all together with hands-on examples, connecting everything from a local Python server to Claude and Cursor, and even exploring modular server design with tool interdependence.

If Part 1 was about understanding MCP’s purpose and architecture, Part 2 showed how to build with it, turning theory into working systems.

Together, these chapters offer a full “theory of operation” for AI-tool interoperability. MCP shifts the paradigm from prompt engineering to systems engineering for AI, where models interact intelligently with a growing ecosystem of tools, data, and interfaces.

This sets the stage for what’s next: building secure, scalable, and production-grade MCP applications. In future chapters, we’ll cover:

Advanced sampling workflows
Permissioning and sandboxing
Real-world deployments and integrations
Read part 3 here, where you will learn how to build a custom MCP client from scratch:

The Full MCP Blueprint: Building a Custom MCP Client from Scratch
Model context protocol crash course—Part 3.

Daily Dose of Data Science
Avi Chawla

Thanks for reading!

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