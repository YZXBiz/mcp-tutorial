Daily Dose of Data Science
Sponsor
Newsletter
More
Search 🔎
Zhuoxin Yang
May 25, 2025
The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part A)
Model context protocol crash course—Part 1.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar

Introduction
Context management in LLMs
Pre-MCP techniques
The Model Context Protocol (MCP)
MCP architecture
In Part B
Introduction
Imagine you only know English. To get info from a person who only knows:

French, you must learn French.
German, you must learn German.
And so on.
In this setup, learning even 5 languages will be a nightmare for you!

But what if you add a translator that understands all languages?

You talk to the translator.
It infers the info you want.
It picks the person to talk to.
It gets you a response.
This is simple, isn't it?

The translator is like an MCP!

It lets you (Agents) talk to other people (tools or other capabilities) through a single interface.

To formalize, while LLMs possess impressive knowledge and reasoning skills, which allow them to perform many complex tasks, their knowledge is limited to their initial training data.

If they need to access real-time information, they must use external tools and resources on their own, which tells us how important tool calling is to LLMs if we need reliable inputs every time.

MCP acts as a universal connector for AI systems to capabilities (tools, etc.), similar to how USB-C standardizes connections between electronic devices.

It provides a secure, standardized way for AI models to communicate with external data sources and tools.

We are starting this MCP crash course to provide you with a thorough explanation and practical guide to MCPs, covering both the theory and the application.

Just as the RAG crash course and AI Agents crash course, each chapter will clearly explain MCP concepts, provide real-world examples, diagrams, and implementations, and practical lab assignments and projects.

As we progress, we will see how MCP dynamically supplies AI systems with relevant context, significantly enhancing their adaptability and utility.

By the end, you will fully understand:

What MCP is and why it’s essential
How MCP enhances context-awareness in AI systems
How to use MCP to dynamically provide AI systems with the necessary context.
How to connect models to tools and other capabilities via MCP
How to build your own modular AI workflows using MCP
In this part, we shall focus on laying the foundation by exploring what context means in the realm of LLMs, why traditional prompt engineering and context-management techniques fall short, and how MCP emerges as a robust solution to these limitations.

👉
Prerequisites: We assume that you have at least basic programming knowledge in Python and a high-level understanding of AI systems such as large language models (LLMs).
Let's begin!

Context management in LLMs
LLMs generate responses based on their "context window," which is the text input provided to the model.

This usually includes the user’s prompt, conversation history, instructions, or extra data given to the model. The model’s responses are completely determined by this context and its training.

However, there are a few important points to note:

Models have a limited maximum context length. If the required information exceeds this window, the model cannot directly “see” it during a single interaction.

Models come pre-trained on vast data up to a certain cut-off date, after which their knowledge is frozen in time. They lack awareness of any events or facts beyond their training data. Context management involves supplying updated or specialized information to augment the model’s static knowledge, which is exactly what we learned in the RAG crash course as well.
Early approaches to context management
Before advanced protocols like MCPs, developers used basic techniques to manage context in LLM applications:

Truncation and sliding windows: For multi-turn conversations (like chatbots), a common approach is to include as much recent dialogue/chats as fits in the window, while removing older messages or summarizing them. While this ensures the model sees the latest user query and some history, important older context might be lost.

Summarization: Another strategy is to summarize long documents or conversation histories into a shorter form that the model can handle. The summary is then given to the model as context. While this can capture key points, it introduces an additional layer where errors can occur. For instance, a poor summary can omit critical details or even introduce inaccuracies. It’s also quite intensive to create good summaries for every potential context on the fly.

Template-based prompts: Developers learned to craft prompts that include structured slots for context. For instance, a Q&A prompt might be:
Here is some information: [insert relevant info].
Using this, answer the question: [user question].
This ensures the model explicitly receives needed data. However, the burden is on the developer (or an automated pipeline) to retrieve or generate the [insert relevant info] part, which can be complex for large knowledge sources.

Despite these efforts, limitations persist.

The context window remains a hard cap; if the data doesn’t fit or isn’t fetched, the model won’t magically recall it.

👉
All these methods are essentially workarounds, they require significant prompt engineering or external preprocessing. There’s no standard mechanism in plain LLM APIs for the model to autonomously fetch more information during its reasoning process, it only passively receives whatever the prompt contains.
This leads us to consider prompt engineering more deeply and why it alone is often insufficient.

Pre-MCP techniques
To appreciate why MCP was developed, we need to look at how developers tried to extend AI capabilities with traditional prompting techniques, retrieval-based methods, and custom tool integrations, and where those approaches fell short that led us to develop MCP.

Static prompting (pre-tools)
Initially, using an LLM meant giving it all the necessary information in the prompt and hoping it would produce the answer.

If the model didn’t know something (e.g., today’s weather or a live database record), there was no straightforward way for it to find out.

👉
As mentioned earlier, the model's knowledge was effectively frozen at training time. Developers resorted to manually feeding data into the prompt (which has limits on size and becomes cumbersome) or just accepting that the model couldn’t perform certain tasks. This one-way paradigm constrained AI usefulness.
Retrieval-augmented generation

A big step was letting models leverage external data by retrieving documents or facts and adding them to the prompt.

👉
In a RAG setup, an external system (not the model itself) does a search or database lookup, and the results are then provided to the model as context.
This helped with up-to-date information and domain-specific knowledge, but it still treated the model as a passive consumer of data. The model itself wasn’t initiating these lookups.

Instead, the developer had to wire in the retrieval logic.

Also, RAG mainly addresses knowledge lookup; it doesn’t enable the model to perform actions or use tools (other than searching for text).

Prompt chaining and agents
Some advanced applications began using Agents (like those built with LangChain or custom scripts) where the model’s outputs could be interpreted as commands to perform actions.

For example, an LLM could be prompted in a way that it outputs something like, SEARCH: ‘weather in SF’ , which the system then recognizes and executes by calling a weather API, feeding the result back into the model.

This technique, often called the ReAct (Reasoning + Action) pattern or tool use via chain-of-thought prompting, was powerful but ad-hoc and relatively fragile. We implemented the ReAct pattern from scratch in the AI Agents crash course:

Implementing ReAct Agentic Pattern From Scratch
AI Agents Crash Course—Part 10 (with implementation).

Daily Dose of Data Science
Avi Chawla

In that discussion, we saw how the LLM's output indicated a function call whenever it required a function call:

Coming back to prompt chaining, here, every developer built their own chaining logic, and each LLM had its own style for how it might express a tool call, making it hard to generalize.

There was no common standard, which means integrating a new tool meant writing custom prompt logic and parsing for each case.

Function calling mechanisms
In 2023, OpenAI introduced function calling. This allowed developers to define structured functions that the LLM could invoke by name.

The model, when asked a question, could choose to call a function (for example, getWeather) and return a JSON object specifying that function and arguments, instead of a plain text answer.

This was much more structured than hoping the model outputs SEARCH: ... in plain text.

For instance, the model might respond with a JSON, like the one below, indicating that it needs to call the weather function.

The client code (outside the model) would then see this and actually call the getWeather API, and return the result to the model for the final answer.

Function calling bridged the gap between language and code by translating a natural language intent into an API call.

But that didn't make these easier logistically for developers.

Let's understand this below.

The M×N integration problem
In summary, before MCP, the landscape of connecting AI to external data and actions looked like a patchwork of one-off solutions.

Either you hard-coded logic for each tool, managed prompt chains that were not robust, or you used vendor-specific plugin frameworks.

This led to the infamous M×N integration problem.

Essentially, if you have M different AI applications and N different tools/data sources, you could end up needing M × N custom integrations.

The diagram below illustrates this complexity: each AI (each “Model”) might require unique code to connect to each external service (database, filesystem, calculator, etc.), leading to spaghetti-like interconnections.

Without a unifying protocol, integrating multiple AI models with multiple tools required bespoke connections for each pair, which is a highly complex, fragile approach.
This fragmentation not only made life hard for developers but also limited the AI systems themselves. An AI agent couldn’t easily share tools or context with another, and adding new capabilities was tedious.

Clearly, the community needed a better way, which would be a standard protocol so that any AI app could talk to any tool in a consistent way.

The Model Context Protocol (MCP)

Model context protocol (MCP) is a standardized interface and framework that allows AI models to seamlessly interact with external tools, resources, and environments.

It can be thought of as a universal adapter or language that both AI systems and external services speak, so they can understand each other without custom glue code.

👉
A popular analogy is that “MCP is the USB-C for AI applications”. Just as a USB-C port standardized how we connect devices (so you don’t need different cables for different peripherals anymore), MCP standardizes how AI models connect to all sorts of external capabilities.
This benefits everyone in the AI ecosystem. For instance:

For developers, instead of writing custom integrations for every tool, you can implement MCP once in your application. After that, a wide range of tools and data sources become plug-and-play. This dramatically reduces integration effort and maintenance.
For API providers, if there's a useful service or library (e.g. a database, a cloud API, a calculator, etc.), it can be exposed through MCP and instantly made available to any AI app that speaks MCP. You implement the protocol once on your side, and you’re potentially compatible with many AI systems. This one-to-many distribution amplifies the tool’s reach.
For end users, they get AI applications that are much more powerful and flexible. Because MCP-enabled AI can fetch real-time information, use specialized functions, or control devices, users enjoy more relevant and accurate assistance. The AI is no longer limited to its training data. Instead, it can truly act as an agent on the user’s behalf.

Formally, MCP is an open protocol (not tied to a single vendor) for structured communication between an AI Host (the AI app or agent environment) and an external Server (which provides tools/data).

By unifying interfaces, MCP breaks down data silos and interoperability barriers. For example, under MCP, whether the AI needs to query a SQL database, call a web API, or execute a Python script, it does so using the same protocol and message format (we shall learn this shortly in more detail).

Crucially, MCP was inspired by the success of the language server protocol (LSP) in the developer tools world.

LSP did for code editors and programming languages what MCP aims to do for AI and tools: it defined a standard so that an editor (like VSCode) could talk to any programming language’s analysis engine in a uniform way (providing autocompletion, error checking, etc.), instead of each editor needing a custom plugin for each language.

Similarly, MCP sets up AI assistants to talk to any tool that implements the MCP server standard.

The result is flexibility and dynamic discovery: an AI agent can autonomously discover, select, and orchestrate tools based on the task context, rather than being limited to a fixed toolbox wired in at design time.

To sum up, MCP is the answer to the question: “How can we let AI systems use external tools and information as easily and uniformly as possible?”

It provides the language and rules for that interaction.

In the next sections, we’ll dive into how MCP achieves this:

its design principles,
the components,
and the mechanics of how an AI uses MCP to get things done.
Why MCP?
The motivation for MCP becomes clear when examining the challenges of earlier approaches. We already touched on the M×N integration problem.

Without a unifying protocol, integrating multiple AI models with multiple tools required bespoke connections for each pair, which is a highly complex, fragile approach.
Let’s expand on that and see how MCP solves it.

Without MCP, adding a new tool or integrating a new model was a headache.

If you had three AI applications and three external tools, you might end up writing nine different integration modules (each AI x each tool) because there was no common standard. This doesn’t scale.

Developers of AI apps were essentially reinventing the wheel each time, and tool providers had to support multiple incompatible APIs to reach different AI platforms.

MCP tackles this by introducing a standard interface in the middle. Instead of M × N direct integrations, we get M + N implementations: each of the M AI applications implements the MCP client side once, and each of the N tools implements an MCP server once.

Now everyone speaks the same “language”, so to speak, and a new pairing doesn’t require custom code since they already understand each other via MCP.

The following diagram illustrates this shift.

MCP transforms integration complexity from M×N to M+N by acting as a universal interface between AI applications and tools. An AI Host only needs to implement MCP once to access many tools, and tool providers implement MCP once to serve many AI clients. The result is standardization and scalability (think “unified APIs” instead of bespoke integrations).
On the left (pre-MCP), every model had to wire into every tool.
On the right (with MCP), each model and tool connects to the MCP layer, drastically simplifying connections. You can also relate this to the translator example we discussed earlier.
Beyond solving the integration combinatorial problem, MCP brings qualitative improvements that go beyond what basic function calling offered:

Dynamic tool discovery:
MCP allows an AI to query a server for what capabilities are available at runtime. In other words, tools don’t need to be known ahead of time if an MCP server offers a new function.
Instead, the client can discover it on the fly. This makes AI systems much more adaptable.
For example, you could plug a new “Stock Market” tool into an MCP server and your AI assistant could start using it immediately if needed, without its core code changing.
Relating to the translator example, if your translator learns a new language, your connection with the translator does not change. You can simply ask if they can help you communicate with a person who knows that new language.
Stateful interactions and memory:
MCP is built with the idea of maintaining context across a session. Because the AI Host (client side) manages the conversation with the tool, it can preserve state, cache results, or reuse context in ways that a stateless API call cannot.
This means an agent can have a form of working low-level chat memory.
Orchestration of multi-step operations:
MCP is designed for tool orchestration, meaning the AI can perform a sequence of actions, possibly involving multiple tools, and even incorporate conditional logic or loops.
For example, consider a travel planning assistant: it might use an MCP tool to get flight options, another tool to check your calendar availability, maybe a payment tool to book the flight, and so on...all threaded together.
MCP provides a structured way for the AI to manage these workflows and even coordinate with multiple agents if needed.
Separation of concerns (client-server model):
With MCP’s client-server architecture, the AI model (client side) and the tools (server side) are decoupled and modular.
The AI's focus remains on understanding user intent and deciding what needs to be done, whereas the MCP server focuses on how to do it (executing the tool).
Standardized execution and interoperability:
Perhaps the most immediate benefit is simply compatibility.
If both an AI application and a tool speak MCP, they can work together out of the box.
Relating to the translator example again, if both you and the person you want to talk to can talk to the translator, it makes things simpler.
Safety and human control:
Because MCP centralizes how tools are invoked, it also provides a point to implement safety checks and require user confirmation for sensitive actions.

For example, an MCP server might decline or ask for approval if the model tries to call a tool that deletes files or sends messages on behalf of the user.
Traditional prompting had no such guardrails (the model could output a dangerous instruction and the system might execute it blindly).
👉
In a nutshell, MCP was needed to standardize and supercharge how AI models use tools. It addresses the fragmentation of early solutions by introducing a common protocol (solving the integration problem), and it expands what’s possible by enabling dynamic, stateful, multi-step interactions with a wide world of resources.
Now that we understand why MCP is important, let’s dive into how it actually works, which is the architecture and components that make up an MCP system.

MCP architecture
At its heart, MCP follows a client-server architecture (much like the web or other network protocols).

However, the terminology is tailored to the AI context. There are three main roles to understand: the Host, the Client, and the Server.

An illustration of MCP’s architecture: The Host (AI application) contains an MCP Client component for each connection. Each Client talks to an external MCP Server, which provides certain capabilities (tools, etc.). This modular design lets a single AI app interface with multiple external resources via MCP.
Let's understand them one by one in detail since we have seen people get confused at times especially between the host and the client.

Host
The Host is the user-facing AI application, the environment where the AI model lives and interacts with the user.

This could be a chat application (like OpenAI’s ChatGPT interface or Anthropic’s Claude desktop app), an AI-enhanced IDE (like Cursor), or any custom app that embeds an AI assistant like Chainlit.

The Host is what the end-user directly sees and uses.

Its responsibilities include managing the user’s input and output (chat UI, or editor, etc.), maintaining the overall session state or conversation, and orchestrating the workflow of calls between the AI model and any external tools.

Importantly, the Host is the one that initiates connections to the available MCP servers when the system needs them.

You can think of the Host as the “brain” of the application that decides when and which external capability to invoke for a given task, based on the AI’s intents.

👉
Examples: Claude Desktop app, Cursor (an AI code editor) or a custom Python program using LangChain or smolagents to build an assistant would also be acting as the Host in MCP terms.
Client
The MCP Client is a component within the Host that handles the low-level communication with an MCP Server.

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
Server
The MCP Server is the external program or service that actually provides the capabilities (tools, data, etc.) to the application.

An MCP Server can be thought of as a wrapper around some functionality, which exposes a set of actions or resources in a standardized way so that any MCP Client can invoke them.

Servers can run locally on the same machine as the Host or remotely on some cloud service since MCP is designed to support both scenarios seamlessly (we'll see that in the demos in the upcoming parts).

The key is that the Server advertises what it can do in a standard format (so the client can query and understand available tools) and will execute requests coming from the client, then return results.

👉
Examples: There are MCP servers that interface with OS operations (like reading/writing files), servers that connect to third-party APIs (e.g. a Slack MCP server that lets the AI read/send Slack messages), servers that provide utilities (like a calculator, or a code execution sandbox), and so on.
Often, these servers are small programs or scripts themselves, while many open-source MCP servers are just GitHub repositories in Python, Node.js, etc., that anyone can run.

For instance, the “Sequential Thinking” server (a community-contributed MCP server) simply guides a model’s reasoning process and can be launched via Node.js.

Here's a demo of it:

Another example: an MCP server could expose a SQLite database with tools like run_query or create_table to let an AI interact with a database directly, instead of dumping the data into the prompt.

👉
Essentially, if you have a capability that an AI might want (data or action), you can MCP-enable it by creating a server for it.
Operational mechanics
In a full MCP-enabled application, these components work together in a pipeline whenever an AI needs to use an external capability. Here’s the typical communication flow step-by-step, combining the roles we just defined:

Pipeline and flow in an MCP-enabled application
User interaction: A user makes a request or asks a question to the AI (via the Host app’s UI). For example, “Hey, what’s the weather in San Francisco?”.
Host decides to use a tool: The Host application processes this input. Perhaps it uses the LLM to parse the request and determines that an external tool is needed, e.g., a weather lookup tool. The Host then decides which MCP Server can provide that tool.
Client connects to server: The Host directs its MCP Client component to establish a connection to the chosen server (if not already connected). This could involve launching the server process (for a local server) or opening a network connection (for a remote server).
Capability discovery: Once connected, the Client queries the Server for its available capabilities. This is typically done with standardized requests like tools/list, resources/list, etc., which ask the server “what tools do you have?”, “what data resources do you have?”, etc. The server replies with descriptions (names, parameters, descriptions of each tool). The Host can then ensure the AI model is aware of these.
Tool invocation: The Host (or the AI model via the Host) decides on a specific action. For our weather example, the Host might instruct the Client: “Call the get_weather tool on the Weather Server with parameter location='San Francisco'.” The MCP Client then sends a Request message to the Server to invoke that tool (we’ll see the exact message format soon).
Server executes and responds: The MCP Server receives the request, executes the underlying function (perhaps it calls an external Weather API or database), and then returns the result back to the client in a response message.
Results integrated: The MCP Client passes the result data back to the Host application. The Host can then incorporate this into the AI model’s context, often by appending it to the conversation transcript or memory so the model can use it to formulate an answer. In our example, the Host might add a message in the conversation like: Assistant (tool): The current weather in San Francisco is 65°F, partly cloudy. The model then continues and replies to the user: “It’s 65°F and partly cloudy in San Francisco.” The Host finally displays this answer to the user.
(Optional) Iteration or more tools: The process can repeat. Maybe the user’s follow-up question triggers another tool use, or the AI decides to call a different tool next. MCP supports multiple rounds of such interactions in one session, and even multiple servers can be in play concurrently.
Termination: When the Host no longer needs the server (e.g., the session ends or the tool is not needed anymore), it will gracefully close the connection. There are MCP messages for a clean shutdown handshake between the Client and Server. If the Host application itself closes, it also shuts down any local servers it spawned.
One of the key advantages of this architecture is modularity.

If a new tool or service is developed, you don’t have to alter the AI application at all. Instead, just run the new MCP server, and the existing Host can discover it.

This decoupling is exactly how MCP turns the integration nightmare into a manageable ecosystem.

Now that we know the roles of Host, Client, and Server, and how they collaborate, we should examine what exactly these Servers provide, i.e., what kinds of capabilities can be exposed via MCP.

In Part B
This is a natural place to conclude Part 1 of our MCP crash course.

By now, you’ve seen the why and what behind MCP.

You understand how context management has evolved, the problems MCP solves, and the high-level architecture that powers it. We’ve also demystified concepts like host-client-server roles, and why the M×N explosion in integration logic called for a protocol-level solution.

But knowing the architecture alone isn’t enough.

To truly use MCP effectively, you need to understand its capabilities, how tools and resources are structured, how messages flow, and how to build real applications that take advantage of it all.

In Part 2, we’ll bring theory to life:

You’ll see how MCP servers expose tools, prompts, and resources.
Learn how JSON-RPC powers the communication layer.
Explore the full lifecycle of an MCP interaction.
And walk through complete hands-on examples with Cursor and Claude.
If Part 1 was about understanding why MCP matters, Part 2 is where we show how to use it to build powerful, modular, AI-native systems.

You can read it below:

The Full MCP Blueprint: Background, Foundations, Architecture, and Practical Usage (Part B)
Model context protocol crash course—Part 2.

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
