Daily Dose of Data Science
Sponsor
Newsletter
More
Search 🔎
Zhuoxin Yang
Jul 20, 2025
The Full MCP Blueprint: Building a Full-Fledged Research Assistant with MCP and LangGraph
Model context protocol crash course—Part 9.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar

Recap
In Part 8, we broadened our scope beyond core MCP and explored its integration into agentic frameworks. Specifically, we examined how MCP can be embedded into Agentic workflows built using LangGraph, CrewAI, LlamaIndex, and PydanticAI.

We evaluated the feature support and limitations of each integration, diving into both the practical implementation details and the underlying concepts.

Through in-depth code walkthroughs and theoretical discussions, we built a comprehensive understanding of how each framework interacts with MCP.

If you haven’t explored Part 8 yet, we strongly recommend reviewing it first, as it establishes the conceptual foundation essential for understanding the material we’re about to cover.

You can read it below:

The Full MCP Blueprint: Practical MCP Integration with 4 Popular Agentic Frameworks
Model context protocol crash course—Part 8.

Daily Dose of Data Science
Avi Chawla

In this part
In this chapter, we’ll focus exclusively on the LangGraph framework and its integration with MCP.

LangGraph is widely regarded as the leading choice for building production-grade agentic systems. We’ll take a deeper dive into LangGraph workflows and integration patterns for MCP by working through a comprehensive real-world use case: a Deep Research Assistant.

Before diving in, it’s important to ensure you have a solid understanding of the fundamentals of AI agents and RAG (Retrieval-Augmented Generation) systems. If you’d like to brush up on these concepts, we recommend exploring the resources below:

AI Agent Crash Course - Daily Dose of Data Science

Daily Dose of Data Science
Avi Chawla

RAG Crash Course - Daily Dose of Data Science

Daily Dose of Data Science
Avi Chawla

Like always, this chapter is entirely practical and hands-on, so it’s crucial that you actively follow along.

Each and every idea will be accompanied by detailed implementations, ensuring that you not only grasp the idea behind it but can also adopt it into your own projects.

Let’s begin.

Ideation
Before diving into the code, let's first focus on system-level thinking of what we are building, the design goals, and the modularity.

Our prime focus is to build a sophisticated yet modular architecture for a Deep Research Assistant, built using LangGraph and MCP.

The assistant must be designed such that it is an extensible, agentic system designed to reason, plan, and act across tools, resources, and prompts while maintaining stateful dialogues.

The system must combine LangGraph's agentic orchestration with MCP's structured tool, prompt, and resource management.

The assistant can leverage a graph-based reasoning, switch between multiple servers, and accept dynamic instructions from users who virtually retain control over the flow via queries and structured meta-commands.

Core capabilities and architecture
The Deep Research Assistant operates as a stateful LangGraph agent, powered by a dual-server MCP client. The system connects to:

💡
"Stateful" implies that the assistant remembers prior interactions and evolves its behavior based on them—crucial for multi-turn reasoning or complex workflows. Dual-server here refers to the MCP client being configured to communicate with two different MCP-compatible servers at once, mentioned below:
A custom research server, exposing vector storage tools, prompts, and FAISS-based semantic search.
The Firecrawl MCP server, capable of live web searching/scraping and data extraction from the internet.

At the core lies a StateGraph, which defines how messages flow, how and when tools are used, and how the conversation state evolves over time.

Here's a flowchart of the same:

Instead of a traditional linear chat loop, the system follows a graph of conditional logic, enabling flexible transitions based on whether tools are needed, whether follow-ups are requested, or whether a user invokes a specific prompt or resource.

If this is getting too much to digest, don't worry. The implementation details will make everything clear pretty soon.

User-guided orchestration with meta-commands
One of the defining aspects of this system must be user control.

Instead of hiding context management behind the scenes, users should be able to explicitly guide the assistant using intuitive commands like:

@prompt:<name> to load MCP prompts.
@resource:<uri> to load MCP resources.
@use_resource:<uri> <query> to use MCP resources.

👉
Note that the access control for prompts and resources defined above is consistent with MCP principles and similar to the manner how it is done in Claude Desktop.
RAG as a tool
Instead of building a fixed, all-in-one RAG pipeline, we use a more flexible approach.

We treat RAG as a set of tools that can be used whenever needed, like saving data, retrieving context, or answering questions.

This makes the system more modular. RAG isn’t something that runs every time by default. It’s just one of the tools the agent can use when it makes sense.

Modular tooling and extensibility
Each MCP server would act as a capability provider, exposing tools, prompts, and resources independently. This makes the entire ecosystem swappable and extensible.

To add a new tool, just add it to the server, and it becomes readily available in the Agent’s toolset. The system supports scaling horizontally across domains.

LangGraph as the orchestrator
LangGraph would play a vital role in giving this system structured memory and decision-making control.

Unlike stateless LLM chains, LangGraph allows the assistant to maintain a chain-of-thought, conditionally branch based on whether a tool call is required, and reintegrate tool outputs into future decisions.

This is essential for such applications that require research and exploration, where context from earlier conversations, resources, or tool outputs must inform current decisions. It also opens up possibilities for multi-step research flows.

Here is the complete flow summarized as a diagram:

Now that we have a clear blueprint on how to proceed, let's get started with setting up the project.

Project setup
We'll be implementing one custom MCP server and also using the prebuilt Firecrawl MCP server.

We would be requiring npm for installing the Firecrawl MCP server. For that, we need to make sure we have Node.js (v22 or later) on our machine.

To download and set up Node.js on your system based on your operating system, check out the official installation guide linked below:

Node.js — Download Node.js®
Node.js® is a free, open-source, cross-platform JavaScript runtime environment that lets developers create servers, web apps, command line tools and scripts.

Or you can watch video tutorials as per your OS:

Windows →
MacOS →
Linux/Ubuntu →
Once we have Node.js up and running, we'll set up our Firecrawl MCP server with:

Note: We'll be focusing our implementation on STDIO transport. Thus, we are installing the Firecrawl MCP server on our machine.

However, if you do not wish to set up the server on your machine, you may also use the remotely hosted URL of the Firecrawl MCP server:

👉
Sign up on Firecrawl to get your API key.
After this, we'll set up a Python project. The codes and project setup we are using are attached below as a zip file. You can simply extract it and run uv sync command to get going. It is recommended to follow the instructions in the README file to get started.

Download the zip file below:

deep-researcher
deep-researcher.zip96 KB
For details about the dependencies, you can check the pyproject.toml file.

Setting up MCP servers
Our first server would be responsible for FAISS-based vector storage and semantic retrieval operations.

Broadly, since it is meant for operations on the information we get or have, let's create it as op_server.py.

Firstly, we'll add all the necessary imports:

These imports help us utilize vector storage and semantic retrieval capabilities. Once this is done, we'll create a server instance and specify the root location for all our vector DBs:

Now, let's move forward and define an MCP tool for saving content inside a FAISS database:

This MCP tool saves a list of text documents (docs) to a FAISS vector store at a specified subdirectory. This tool accepts:

docs: a list of plain text strings.
path: the folder under vector_dbs/ to save the index (default is "default").
Let's understand this tool's logic step-by-step:

Lines 7-8: We start by defining the target path for storage. The subdirectory is created if it doesn't exist.
Line 10: Here we initialize embeddings. The tool uses OpenAI’s "text-embedding-3-small" model to convert text into dense vectors (make sure to specify your OpenAI API key).
Line 12: Next, we wrap raw strings into LangChain Document objects.
Line 13: Points to the FAISS vector index file.
Lines 15-20: If the index already exists, we load it, update and save it again with the appended information.
👉
The allow_dangerous_deserialization=True flag is required to load the index.pkl metadata file. Since all indexes are created internally by this tool, it is relatively safe to enable this.
Lines 21-23: If no existing index, then creates a new FAISS index using the given documents and saves it to the target directory with index.faiss + index.pkl.
Line 26: Returns a status message with the number of documents saved and the target location.
Overall, the tool automatically creates or updates the store at a specified location by:

Creating a new index if one doesn't exist
Appending to an existing index if one is already saved
Here's a diagram summarizing the flow:

Next, we need a tool to perform a semantic search over a FAISS database to retrieve information.

This semantic_search tool is designed to perform vector-based semantic search over a FAISS index stored under a given path (vector_dbs/{path}/), using OpenAI embeddings.

Similar to the previous tool, we first locate the vector store path, then check for the index's existence, load the embeddings, and then the FAISS index.

Then we perform a semantic search with vectorstore.similarity_search(query, k=5). This line basically:

Converts the query into a vector.
Performs nearest neighbor search to retrieve the top 5 documents whose vectors are most similar to the query.
Finally, we extract the text content (r.page_content) from the Document objects returned by FAISS and return a plain list of strings.

Here's a diagram summarizing the flow:

Let's proceed further and define another tool for listing all the available prompts in the server:

In the above implementation, we retrieve the available prompt details by using the get_prompts() method on our server instance (mcp).

👉
We'll understand the role and significance of this available_prompts tool when we'll implement the client-side of this workflow.
Now we are done with all the required tools on our server. So let's move forward and define an MCP resource that serves as a lightweight list for all available vector databases stored on disk:

This resource, when loaded, provides a list of all available databases to the client.

👉
The usefulness of this resource will become clear once we complete our client code and run the full end-to-end workflow.
Finally, let's also define an MCP prompt to perform deep research on a user-specified topic:

This prompt basically helps the user to get a thorough and detailed report on a topic of choice, without diving into the complexity of writing structured and detailed queries themselves.

With this, we have defined all the capabilities of our server, and we end the server script with:

Note that we just use mcp.run() because we are using the STDIO transport mechanism.

Any MCP-compliant client can connect to list available capabilities or to invoke them.

Our second MCP server is the Firecrawl MCP server. We have already seen how to install it, so now we only need to configure it in our client. Hence, let's take a look at how we can integrate them into a LangGraph multi-server client/host application.

Client-side implementation
For the client side, we'll be implementing a Python script that sets up an interactive CLI-based AI assistant primarily using the LangGraph framework.

It connects to our two servers (operations server and Firecrawl server) and uses tools and other capabilities from those servers to help answer user queries.

👉
Our client-side script basically works as an MCP Host, that can connect to multiple servers via multiple managed client sessions, that is, one client session per server.
To get started with our implementation, we import all the required modules:

Note that the colorama import is used for printing colored text in the console (to distinguish user input, AI output, and instructions with distinct colors for better readability).

After imports, colorama is initialized:

Next, in this script, we create a MultiServerMCPClient instance:

The MultiServerMCPClient helps establish multiple client sessions to manage each of the server connections separately and aggregate their capabilities. This is similar to what we saw in the previous parts of the MCP crash course.

After this, we define a global dictionary to keep track of loaded resources:

This dictionary will store the content of resources that the user loads via special commands defined ahead. Keeping it global ensures that once a resource is loaded, it remains available for subsequent queries until the program ends or the resource is reloaded.

Utility functions
The script defines several helper functions to process user input and manage the special commands.

Firstly, we have the extract_meta_command function, which checks if the user's input starts with a special command prefix like @resource:, @prompt:, or @use_resource:.

Let's break down how this works:

It takes the user message string and checks prefixes:
Lines 3-4: If the message begins with "@resource:", it will treat it as a command to load a resource. It returns the tuple ("resource", <resource_uri>).
Lines 6-7: If the message begins with "@prompt:", similarly it returns ("prompt", <prompt_name>) after extracting the prompt name.
Lines 9-22: If the message begins with "@use_resource:", the logic is a bit more involved:
It splits off the part after @use_resource: and then tries to separate it into the resource URI and the user’s actual question.
It finds the first space in the remaining string. If found, it splits into resource_uri (everything before that space) and user_query (everything after the space).
If no space is found, it means the user didn’t provide a query after the resource name. In that case, it returns the resource name and an empty query string.
Ultimately, it returns ("use_resource", (resource_uri, user_query)).
Line 24: If the message doesn’t match any of these special patterns, the function returns (None, None) indicating it's a normal message, that is, not a special command.
This function is key to routing the user input: the main loop will use it to decide how to handle the input (as we will see later).

After this, we define a parse_arguments function that parses a string of arguments into a dictionary. It's used when the user supplies arguments for prompts.

It tries to interpret the input string in several ways:

JSON format: It attempts json.loads(raw_input_str). If the input is a valid JSON string, it will parse it into a dictionary directly. If this succeeds, it returns the dictionary.
Comma-separated key:value pairs: If JSON parsing fails, it next splits the string by commas. Then for each part, it splits by the first colon to separate key and value. Each key and value is stripped of whitespace. This allows inputs like key1:value1, key2:value2.
Single key:value pair: If the comma-separated parsing doesn't produce anything (or if no comma was present), it tries one more time to split the string by a colon into a single key and value.
If all parsing attempts fail, it returns an empty dictionary.
This flexible parser means the user can input arguments for a prompt in various convenient formats, and the code will handle it.

Finally, we define our last utility, inject_resource_into_message function, which is used to incorporate a loaded resource's content into a user query when the user uses the @use_resource command.

It takes the user_query and a resource_uri and checks if the resource is already loaded in the loaded_resources dictionary:

If yes, it retrieves the resource_content from the dictionary. It then constructs a combined message string that includes:
A marker line [USING RESOURCE: resource_uri].
The full content of the resource.
Then it appends User query: {user_query} at the end.
This combined string essentially prepends the resource material to the user's actual question.
If the resource is not found in loaded_resources, it returns a message indicating an error and listing available resources, followed by the user query. This ensures the AI still sees the user’s question, but is informed that the resource wasn’t available.
The returned string will later be fed into the AI agent as if it were the conversation. In essence, this allows the user to ask a question with a specific resource’s content in context.

With our utilities defined, let's move forward and create the workflow graph.

Building the workflow
The core of the AI assistant is constructed in the following create_graph asynchronous function. This is where we set up the large language model, tools, and the conversation flow logic (graph) that will be followed:

Let's break this down step-by-step:

Line 3: Initialize the LLM.
Lines 6-7: Loaded MCP tools using load_mcp_tools(session), we fetched all tools from the connected MCP server and got them as LangChain tool objects.
Line 8: Combines tools from both servers into a single list.
Line 11: Wraps the LLM with tool usage capability.
Lines 13-17: Defines a chat prompt template. A system message and a placeholder for prior chat messages.
Lines 19-21: Declares the graph state as a dict with a single key messages. The messages list is automatically updated via add_messages.
Lines 23-27: We define a function for chat nodes. It combines the prompt template with the LLM+tools. This node uses the chat_llm to generate a new message based on conversation history (state["messages"]). Returns a new state with the response as the only message in the list.
👉
This might seem odd (since we likely want to accumulate messages), but remember the add_messages annotation: under the hood, LangGraph will merge this with the prior state's messages, effectively appending the new response to the conversation. So the conversation grows with each turn.

Line 30: Initializes a LangGraph stateful flow with the state schema defined earlier.
Lines 31-32: Adds two nodes -
chat_node: where reasoning happens.
tool_node: a built-in node that automatically executes tool calls from messages.
Line 34: The graph starts at chat_node.
Lines 35-37: After chat_node, the graph checks whether the next step involves tools:
If yes → go to tool_node.
If no tool is needed → end the graph.
Line 38: After tool execution, go back to chat_node for further reasoning or message generation.
Line 40: Returns the compiled stateful workflow into an executable graph along with the tool list. MemorySaver() ensures that intermediate states are saved in memory.
Together, these edges define a loop where:

The AI gives an output.
If the output is a tool request, execute the tool, and get the result.
Feed the result back into the AI to get a refined answer.
Repeat until the AI's output does not request more tools (then go to END).
Here's a diagram summarizing the flow:

Now, we've created an AI agent that is configured to:

Use an LLM that can call tools.
Start by reading user input and generating a response.
Automatically use tools if needed and incorporate their results.
Keep track of the conversation context (stateful memory).
Next, let's take a look at main() function.

Main program execution
After defining all the components (utilities and the agent graph), we define the main() function, which orchestrates the actual runtime behavior. This function manages user interaction and ties everything together.

Let's walk through the main() function step by step.

Starting multi-server sessions
The code uses an async with context to start sessions for both servers:

Here client is our MultiServerMCPClient from earlier, which knows how to manage the two configured servers.

Creating the agent and listing tools and resources
Within the session context, the code creates the agent:

This calls the create_graph function we discussed above, providing the active sessions. It yields:

agent: the compiled conversation agent ready to accept messages.
tools: the list of tool names/functions loaded from both servers.
Immediately after, the code prints the available tools to the user.

Then it loads available resources:

The load_mcp_resources(first_session) invocation fetches a list of resources from the first server. These could be documents or data sources that the user can load by name. Each resource likely has metadata including a 'URI' or identifier.

We do not specify the second session here since we are primarily interested in the tools provided by the Firecrawl server.

Next, if you check the codes attached, the code prints a help menu of commands and argument formats. This is basically a user guide that mentions:

@resource:<uri>: to load a resource by its URI (as listed in available resources).
@prompt:<name>: to load a predefined prompt by name.
@use_resource:<uri> <question>: to ask a question with a previously loaded resource's content.
exit or quit: to exit the program.
At this point, the program has informed the user about the environment (tools, resources) and how to interact with it.

Next, let's see how exactly the user inputs would be taken and processed.

The user input loop
The main loop waits for user input and processes it:

It continuously prompts the user (printing "User:" in green to signal it's waiting for input).
The input is read as message (stripped of whitespace).
If the user types "exit" or "quit" (case-insensitive), the loop breaks, ending the program.
Otherwise, it invokes our earlier extract_meta_command(message) to see if the input is one of the special commands. It returns a cmd_type and a value:
cmd_type could be "resource", "prompt", "use_resource", or None.
value is the associated value (like the resource URI, prompt name, or a tuple of resource_uri and user_query).
The code then uses a series of if/elif to handle each case:

Handling @use_resource command:

Here is a clear explanation of the case when cmd_type is "use_resource":

Line 2: The value from extract_meta_command is a tuple (resource_uri, user_query).
Lines 3-11: First, it checks if user_query is empty. If so, it prints an error in red and a usage hint in yellow, then uses continue to skip the rest of the loop (go back to prompting user again).
Lines 12-19: Next, it checks if the resource specified has been loaded into loaded_resources. If not loaded:
It prints an error saying the resource hasn't been loaded and a hint to load it first with @resource:<uri>.
Then continue to skip processing (since we can't proceed without the resource content).
Lines 21-31: If the resource is loaded and a query is provided:
It calls inject_resource_into_message(user_query, resource_uri) to create the combined message that includes the resource content and the user’s question.
This combined enhanced_message is then passed to the agent to get a suitable response.
response i.e. the final AI answer to the user's query (using the resource) is printed.
Line 32: continue at the end of this block, ensures that after handling @use_resource, the loop goes back to the prompt for new input.
Handling @prompt command:

Here is a clear explanation of the case when cmd_type is "prompt":

Line 4: Prompt is loaded, where value is the prompt name the user provided after @prompt:, meta command.
Lines 5-16: If prompt is truthy (found):
Prepares prompt_message.
Next, we invoke the agent similarly as before and pass the prompt content as the message to the agent. This means we're sending this entire prompt to the agent as if the user just provided it.
The response is awaited and then printed.
If an exception occurs during agent invocation, it prints an agent error.
Lines 17-18: If prompt is falsy (no prompt found with that name), it prints "No prompt messages found" in red.
Lines 20-59: If an exception is caught at this point, it is due to the fact that the prompt needed arguments, because agent response related exceptions are already caught earlier. So here is how we handle this:
It prints a message indicating that the prompt requires arguments, along with the error for more details.
It then prompts the user to enter arguments (with an input).
The input for arguments is then parsed with parse_arguments (the function we described earlier).
If parsing fails (args is an empty dict), it prints an error and uses continue to go back, skipping further processing in this iteration.
If parsing succeeds, it tries load_mcp_prompt again, but this time with the arguments=args.
Then, if the prompt is loaded, it again prepares the prompt_message.
Then calls agent.ainvoke with the prompt message and prints the results.
If any error happens during invocation, or if the prompt still isn't found, it handles those by printing error messages.
Line 60: After handling the prompt, it continues to the next loop iteration.
This block allows the user to run predefined prompt scenarios. If a prompt needs input, the user will be prompted to provide those.

Handling @resource command:

First, if the resource is already in loaded_resources, it means the user might be reloading it. Hence deletes the old entry from the dictionary. This ensures we fetch a fresh copy.

Then we invoke load_mcp_resources(first_session, uris=[value]). If blob is not empty (resource found):

It stores the content string into loaded_resources dictionary with the key as the resource name.
If blob is empty (no such resource):

Prints an error.
If any exception occurs during loading, it prints the "Failed to load resource" message.

This block of code helps the user load a resource from the server by name. Once loaded, its content is stored and can be used in queries via @use_resource.

Now that we have handled all the meta commands, let's take a look at a relatively straightforward code snippet for handling queries without meta commands.

Handling general queries
Finally, the last part of the loop handles regular user inputs that are not special commands:

If extract_meta_command returned (None, None), the code ends up in this else branch, meaning the user input is a normal question or statement.

It calls agent.ainvoke({"messages": message}, config=config):

It sends the user's message to the LangGraph agent to get a response.
It waits for the agent to return a response.
If there's an error during the agent call, it prints an error message.
If successful, it prints the response.
In blue text. It extracts the content of the last message from the response (which should be the assistant's answer).

After printing the AI's answer, the loop goes back to prompt the user again.

This loop continues until the user types "exit" or "quit", at which point the loop breaks and the program (and the async with context, thus the server sessions) ends.

Outside the loop and main function
We complete our client script with:

This is the typical way to start the asynchronous main() function when the script is executed.

When the user eventually exits, the async with block will close the connections to both servers gracefully, and asyncio.run() will finish.

To summarize, this script sets up an AI assistant that:

Connects to two servers to obtain capabilities (tools and data).
Uses an LLM that can utilize those tools through a LangGraph state machine.
Provides a command-line interface where a user can:
Directly ask questions.
Load external resources into memory and then ask questions about them.
Execute predefined complex prompts or scenarios.
The code is structured to handle different user inputs gracefully, providing help messages and ensuring the AI has the needed context (like injecting resource content into queries).
Through the use of LangGraph, the assistant can decide to use tools in the middle of answering questions (for example, doing a web search) and then continue the conversation with the results of those tools.
Here's a diagram encapsulating the complete client application flow:

By breaking down the code into setup, utilities, agent creation, and main loop, we can see how all the pieces work together to form a powerful interactive research assistant system.

Below are some screenshots of sample results:

View on running the client script
In the above screenshot, we can see a view of how things will look as soon as we run our client code with uv run client.py. We get lists of available tools, resources, and guidelines for meta command usage and argument formats.

👉
If you're on Windows, the application might take a few seconds to start after you run the command.

Query: "what prompts do you have"
Here we ask for available prompts. Note that this uses the available_prompts tool for this. There is no direct functionality like load_mcp_tools or load_mcp_resources in LangGraph, that would help us know all the available prompts. Hence, this tool finds its use in our application setup.

Using the 'research_prompt' to get a detailed study on LLMOps
In the above screenshot, we can see how prompts and arguments to prompts are getting handled. After supplying arguments, once our prompt is successfully loaded, it is automatically executed. Then the agent would use the needed tools and finally generate a report like the sample below:

👉
This is not a complete length screenshot of the generated results. It's basically attached to demonstrate how things are working.
At this stage, most probably, the Firecrawl tools will be used. So you can navigate to your Firecrwal account dashboard and under "Activity Logs" option, see the details:

Next, let's try to save the report that we got:

Query: "save this at location LLMOps_report"
Here we prompt it to save the generated results at a specified location.

Here we can see our vector DB created with our results stored within.

Command: "@resource:vector://list"
In the above screenshot, we load the resource with URI vector://list. Now let's see how to use it exactly:

Command: @use_resource:vector://list what are the available databases
Writing this command gives us the list of existing databases, using the resource as context. As shown in the screenshot, this resource helps us get information about the existing vector databases available.

This information as context could help the LLM specifically answer availability-related queries like "Does a DB by name 'X' exists or not?", or queries pointed specifically at the content of a given database.

👉
We have designed such an access mechanism for resources and prompts, since as per MCP principles, they are not meant to be model controlled (check Part 4).

Query: 'from the LLMOps_report database can you help me with content regarding "current state of research"'
Here in the above screenshot, we are asking the mentioned query in the caption to get results from our saved database LLMOps_report.

👉
This is not a complete length screenshot. It's basically attached to demonstrate how things are working.
Similarly, various other possibilities and combinations can be tried out by the use of a mix of prompts, resources, and tools with various different queries.

👉
Note that these screenshots are attached to provide an idea of things, and do not show the complete responses. In your usage, the responses will differ based on your data and the LLM's understanding of the same. However, the behavior will be grounded and predictable.
We encourage you to try and experiment with more queries and combinations, and observe how the capabilities work differently yet complement each other.

Try out yourself
Based on the context set by this chapter and previous knowledge from the series, we'd like to recommend some knowledge-enhancing tasks for you:

Test out the servers (Operations server and Firecrawl server) with MCP Inspector.
In the Operations server (op_server.py), instead of directly hardcoding the LLM API key or loading from .env file, check how you could provide the API key as part of the server parameter configuration on the client side and access it in the server, similar to how it is done for the Firecrawl.
Earlier, we saw, we are using allow_dangerous_deserialization and setting it to be True. Although it is relatively safe since all indexes are created internally, it would be better to Dockerize it. Hence, try setting the op_server.py file inside a Docker container with limited permissions and use it from there. For reference, you could use our sandboxing article:
The Full MCP Blueprint: Testing, Security, and Sandboxing in MCPs (Part B)
Model context protocol crash course—Part 7.

Daily Dose of Data Science
Avi Chawla

Explore multi-agent collaboration, where sub-agents specialize in specific tasks. For conceptual clarity on this, we recommend referring to the AI Agents crash course:
AI Agent Crash Course - Daily Dose of Data Science

Daily Dose of Data Science
Avi Chawla

Fun task
With this part, we conclude our MCP Crash Course. But don’t go too far, a brand new series is just around the corner, and it’s going to be just as interesting. Can you guess the topic?

Hint: A subtle clue is hidden somewhere in this article.

Conclusion
With this final chapter, we bring the MCP Crash Course to a close by not just discussing what’s possible, but by building it.

In this part, we brought together everything we’ve learned throughout the MCP course to build a fully functional system: the Deep Research Assistant.

We didn’t just add LangGraph support, we centered the entire architecture around it. LangGraph served as the orchestration backbone, powering structured memory, dynamic tool calls, and conditional execution paths.

By combining it with the modularity of the Model Context Protocol (MCP), we created a sophisticated yet intuitive agent that could reason, plan and act.

Let’s recap what we accomplished:

We began with system-level design thinking, defining key goals such as modularity, extensibility, and structured orchestration. We treated tools, prompts, and resources as swappable, composable components governed by clear MCP principles.
We implemented a custom MCP server that offered vector-based storage and retrieval capabilities via FAISS. Through tools like we broke free from rigid RAG pipelines and instead created a plugin-based, query-specific retrieval system.
We integrated a second server: Firecrawl, which brought real-time web capabilities like scraping and search. This dual-server setup illustrated how multiple MCP services could coexist and contribute to an agent’s workflow. More importantly, it showed that capability is not tied to a monolith, it’s distributed, modular, and controllable.
On the client side, we built a LangGraph host application that dynamically connected to both servers. Users weren’t left guessing how the agent was reasoning, instead, they actively guided it via meta-commands.
We built a memory-driven LangGraph state machine, using tool nodes and conditional transitions to manage multi-turn tool interactions.
We implemented a well-structured command loop, complete with error handling, usage guidance, and support for prompt argument injection.
The agent gracefully handled invalid inputs, missing resources, and prompts requiring dynamic arguments.
But most importantly, we demonstrated how LangGraph and MCP together redefine the agentic development paradigm:

MCP provides a clean, universal interface to tools, prompts, and resources, i.e., everything that an agent needs to think and act.
LangGraph brings memory, conditional logic, and decision orchestration to that environment, elevating it from a simple LLM wrapper to an autonomous reasoning entity.
Together, they create a blueprint for building intelligent, interactive systems that are:

Extensible: New tools and capabilities can be added without refactoring the entire agent.
Modular: Components (servers, tools, prompts, etc.) are loosely coupled and swappable.
Reproducible: Context and behavior are structured and observable.
This chapter has also shown that MCP isn't just a protocol for calling tools, it's a design philosophy. It prioritizes modularity, user control, safe execution, and cross-framework interoperability.

Throughout this series, we observed that, when implemented correctly, MCP lets you build agents that are:

Interoperable, using shared standards for communication.
Decoupled, able to plug into different environments.
If you’ve followed the MCP Crash Course from the beginning, you’re no longer just a learner; you’re now a practitioner.

With this, we conclude our MCP Crash Course.

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
The Full MCP Blueprint: Practical MCP Integration with 4 Popular Agentic Frameworks
MCP
Jul 13, 2025
•
28 min read
The Full MCP Blueprint: Practical MCP Integration with 4 Popular Agentic Frameworks
Model context protocol crash course—Part 8.

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
