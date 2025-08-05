
Daily Dose of Data Science
Sponsor
Newsletter
More
Search 🔎
Zhuoxin Yang
Jun 22, 2025
The Full MCP Blueprint: Integrating Sampling into MCP Workflows
Model context protocol crash course—Part 5.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar

Recap
In this part
Introduction
Concept and motivation
Architecture
Context object and sampling mechanism
Server-side implementation
Client-side implementation
Model preferences
Sampling use cases
Advanced use cases and patterns
Error handling and best practices
Conclusion
Recap
In Part 4 of this MCP crash course, we expanded our focus beyond tools and took a deep dive into the other two foundational pillars of MCP: resources and prompts.


We learned how resources allow servers to expose data, anything from static documents to dynamic query-backed endpoints.


Prompts, on the other hand, enable natural language-based behaviors that can dynamically shape responses based on input, roles, and instructions.

Through theory and hands-on walkthroughs, we explored how each capability differs in execution flow, when to use tools vs. resources vs. prompts, and how they can be combined to design the server logic.

We also discussed how Claude Desktop can leverage these primitives to orchestrate rich user interactions powered by model-controlled, application-controlled, and user-initiated patterns.


Overall, in Part 4, we learned everything that it takes to build full-fledged MCP servers that don’t just execute code, but also expose data and template-based reasoning, allowing for cleaner separation of logic and language.

If you haven’t explored Part 4 yet, we strongly recommend going through it first since it lays the conceptual scaffolding that’ll help you better understand what we’re about to dive into here.

You can read it below:

The Full MCP Blueprint: Building a Full-Fledged MCP Workflow using Tools, Resources, and Prompts
Model context protocol crash course—Part 4.

Daily Dose of Data Science
Avi Chawla

In this part
In this chapter, we move a step ahead and introduce what we think is one of the most powerful features in the MCP protocol: sampling.

So far, we’ve seen how clients invoke server-side logic, but what if the server wants to delegate part of the work back to the client’s LLM?

That’s where sampling comes in.

We’ll explore how MCP enables server-initiated completions to request a response from the client’s LLM during execution.

We’ll explore:

What is sampling and why is it useful?
Sampling support in FastMCP
How does it work on the server side?
How to write a sampling handler on the client side?
Model preferences
Use cases for sampling
Error handling and some best practices
As always, every notion will be explained through clear examples and walkthroughs to develop a solid understanding.

Let’s begin!

Introduction
In a typical MCP setup, an MCP server exposes functions (tools), data (via resources), and prompts that an LLM client can use.

But what happens when the server itself needs to tap into the intelligence of an LLM? This is exactly what MCP Sampling is designed to solve.


LLM sampling enables the server to delegate tasks to the language model available on the Client's side. In simpler terms, sampling in MCP allows a server to ask the client’s AI model to generate text (a completion) and return the result.

We will explore the concept, implementation, and applications of sampling with the FastMCP framework.

Concept and motivation
LLM sampling creates an inverted or bidirectional architecture.

Typically, an LLM client invokes the server’s tools, but with sampling, the server can also call back to the client’s model.

Why is this useful?

Consider the scenario of building an AI-driven server that needs to perform language understanding or generation as part of its functionality.

For example, summarizing a document or composing a message. Traditionally, the server would have to integrate an LLM API itself or run a model locally, which can be costly and hard to scale.

MCP sampling flips this around:

Delegation of LLM work: The server offloads the language generation task to the client’s model. The server can ask, “Please summarize this text”, and the client’s LLM does the heavy lifting.
Here's the step-by-step flow:

The server’s tool function determines it needs an LLM’s help (e.g. to analyze or generate text).
The server sends a prompt/request to the client via the sampling mechanism.
The client receives this request and invokes its own LLM (local model or an API) to generate the completion.
The client returns the generated result back to the server.
The server’s tool function then resumes execution, using the LLM-generated result as needed, as if the server had produced it itself.
This provides several key advantages:

Scalability
The server avoids doing the computationally intensive inference.

This means the server can handle more concurrent requests since it’s not bogged down generating large chunks of text itself. Each client essentially handles its own LLM workload.

Cost efficiency
Any API costs or computational load associated with the LLM are borne by the client. If using a paid API, the client’s account is charged for the completion, not the server.

This distributes costs across users and frees the server owner from maintaining expensive infrastructure.

Flexibility in LLM choice
Clients can choose which model to use for a given request.

One client might use OpenAI’s GPT-4o, another might use an open-source model, and the server doesn’t need to change at all.

The server can even suggest a model preference (we’ll discuss model preferences later), but the client ultimately controls execution.

Avoiding bottlenecks
In a traditional setup, if the server had to handle all AI generation, it could become a bottleneck (imagine hundreds of users triggering text generation at once on one server).

With sampling, each user’s own environment handles their request, preventing server-side queue build-up.

In summary, sampling in MCP allows distributed AI computing.

The MCP server can incorporate powerful LLM capabilities without embedding a model or invoking external APIs itself.

It’s a bridge between the usually deterministic server logic and the dynamic text generation, achieved through a standardized protocol call.

Next, we’ll see how this fits into the MCP architecture.

Architecture
From previous parts, we know that MCP uses a client-server architecture.

The server exposes an interface of tools, resources, and prompts, and the client connects to the server to use those capabilities.

Apart from the server providing tools, resources, and prompts, it can also request the client’s LLM to generate completions (i.e., sampling).

The LLM’s outputs are returned to the server, enabling AI-driven behaviors on the server side.

In a normal interaction, the client sends a request. The server does the necessary execution and returns the result to the client. Sampling introduces a reverse interaction: now the server, while handling a request, can ask the client to perform an AI task.

👉
Under the hood, this is supported by the MCP protocol as a special type of message, often called a completion request or sampling request, that the client can fulfill.

MCP Server (blue)

A function/tool is running as part of an agent workflow on the server.
Inside this function, ctx.sample() is called to invoke an LLM for generating a response or make a decision.
This call does not execute the sampling locally. Instead, it packages the request and sends it to the MCP client.
MCP Client (green)

The client listens for sampling requests from the server and receives this one.
A user-defined sampling_handler() is triggered. This function defines how to process the request. E.g., format the prompt, handle retries, etc.
The client uses either an external LLM API (like OpenAI) or a local model (like LLaMA or Mistral) to complete the request.
The client sends back the generated text as a response to the server.
Once done, we get back on the MCP Server (blue) and resume execution along with LLM's result. The server receives the result from the client and resumes the tool function execution using the LLM-generated output.

In practical terms, FastMCP’s client library helps us explicitly provide a sampling handler (a callback function) to deal with these requests. We’ll cover how to implement this shortly in this article.

Transport and execution
The MCP protocol ensures that these requests are transported reliably. FastMCP supports multiple transports (like stdio and sse).

No matter the transport, the flow is the same. The server invokes the sampling method on an instance/object of the Context class (like ctx.sample(...)), packages it into an MCP message, sends it to the client, and waits for a response.

The client side invokes the sampling handler, which in turn invokes the actual LLM, and sends back the completion. All of this is asynchronous, that is, the server’s tool coroutine will suspend until the result comes back, avoiding blocking other tasks.

The entire flow is depicted in the diagram below:


Sampling requests are just another type of structured request, so all the usual MCP validation applies.

For example, the server can’t force the client to run code; it can only request a text completion with certain parameters. And from the client’s perspective, it only ever runs its own LLM on prompts, and it doesn’t execute server code.

Context object and sampling mechanism
To use sampling in FastMCP, the server provides a Context object to its functions.

The Context is a powerful handle that, apart from requesting LLM sampling, can give server code access to logging, sending updates, etc.

FastMCP automatically injects this context into tool functions when we include it as a parameter. For example:


In the above snippet, because we annotated ctx with type Context, FastMCP will inject the context when analyze_data is invoked by an MCP client.

On a side note, you can download the code for this article below. Specifically open the open-me.ipynb notebook for instructions.


Download below:

sampling-mcp-code
sampling-mcp-code.zip6 KB
👉
We can name this parameter however we like (ctx, context, etc.); FastMCP recognizes it by type hint. But it is important to specify the type hint since the type is picked by the MCP server.
One important point to note is that the context is only valid during the request, which means we can’t store it as a global variable and use it later outside the tool call (if we do, it would raise an error).

If we need to debug or invoke context methods outside of a request, we can type the variable as Context | None=None to avoid missing argument errors.

Once we have access to ctx inside a tool, we gain the ability to invoke ctx.sample() method to request a completion from the client’s LLM.

Since interacting with the client or server internals may involve I/O (and in the case of sampling, waiting for a remote model), all methods on FastMCP’s Context object are asynchronous.

That’s why our tool functions are defined as async def. Using await ctx.sample(...) will suspend that function until the result arrives.

👉
Under the hood, ctx.sample packages up the provided prompt and parameters into a message that conforms to the MCP specification for sampling.
To summarize, the Context object is the gateway to advanced MCP capabilities.

Just add a Context parameter, and FastMCP will provide it when a client invokes the tool. Moreover, it is only valid during the call, which ensures isolation and safety.

Next, let’s dive into how to actually implement sampling on both server and client sides.

Server-side implementation
On the server side, implementing sampling in FastMCP is straightforward.

We simply invoke the ctx.sample() method from within a tool function to request a completion from the client’s LLM.

👉
Although it is not very intuitive, but you should note that sampling can also be used within resources and prompts. However, this is relatively uncommon in practice, as actionable logic usually resides within tool functions.
Let’s walk through an example of a server tool that uses sampling to summarize a document:


We have defined a tool summarize_document that takes in some text and a Context. Inside the function, we invoke ctx.sample(...) with a prompt and some parameters:

messages: Here we passed a single string. When we provide a plain string, FastMCP treats it as a user message to the LLM.
system_prompt: We provided a system instruction to set the context and rules for the LLM’s behavior. In this case, we tell the model it is an expert summarizer. The system prompt is a special, optional parameter that the client’s LLM will receive as a high-level directive.
temperature: Controls the randomness of the output. Lower values (e.g. 0 or 0.2) yield more deterministic outputs, while higher values (0.7 or 1.0) yield more varied and creative text. We set temperature=0.7, which is a moderate value to allow some creativity but still keep the summary focused.
max_tokens: We capped the generation at 300 tokens, to ensure the summary isn’t too long. If not specified, a default value may be used.
The return value of ctx.sample is a content object representing the LLM’s response. In FastMCP, this is typically a TextContent object with a .text attribute containing the generated text. We access response.text to get the actual summary string.

We can use ctx.sample(), as shown above, in any server function that has a Context object, whether it’s a tool, a resource handler, or even a prompt template function.

It’s most common in tools (since tools perform actions and might need AI help), but one could imagine a resource that dynamically generates text via the LLM, or a prompt that itself uses the LLM to fill in some template.

Just remember that each ctx.sample call will incur a round-trip to the client’s model, so use it when it makes sense and be mindful of performance.

After setting everything up, we can start the server using mcp.run() (as discussed in earlier parts) by passing the necessary arguments to configure the desired transport and other behaviors.

We’ve seen how to write the server side for sampling. Now let's implement the client side to actually handle this sampling request.

Client-side implementation
On the client side, we need to define a sampling handler function that will be invoked whenever the server requests an LLM completion.

On a side note:

At the time of publishing this part, neither Claude Desktop nor Cursor supports sampling as MCP clients. Therefore, we’ll implement a minimal custom handler using FastMCP’s built-in support for writing client-side logic.

However, if you prefer a prebuilt solution with full MCP capabilities, including tools, resources, prompts, and sampling, you may want to explore the classic VS Code + GitHub Copilot setup, which offers complete sampling support. You can find more details at the links below:

The Complete MCP Experience: Full Specification Support in VS Code
VS Code now supports the complete Model Context Protocol specification, including authorization, prompts, resources, and sampling.

Microsoft
Microsoft

Use MCP servers in VS Code (Preview)
Learn how to configure and use Model Context Protocol (MCP) servers with GitHub Copilot in Visual Studio Code.

Microsoft
Microsoft

Coming back to the client-side implementation, a sampling handler is simply an async function that takes the prompt from the server and returns the generated text.

Let’s implement one:


In this example, our sampling_handler has the following:

It receives a list of SamplingMessage objects, which contain the content and role of each message the server provided.
SamplingParams object with additional parameters (like systemPrompt, temperature, maxTokens, etc.).
It also gets a RequestContext. Though not used in here, it carries metadata about the request like request_id, session, etc.
In our handler, we construct a list chat_messages in the format required by LiteLLM.
If a system prompt was provided, we add it as the first message with role "system".
Then we iterate over the messages list (which typically contains the user prompt or a conversation history) and append each message’s text with the appropriate role (likely "user" or "assistant").
We then invoke the LLM. We could also decide based on preferences (more on model selection ahead). We pass the prepared messages. We handle this in a try/except to catch any errors from the API (network issues, etc.), and return either the generated text or an error message string.
Finally, we return the generated_text. The string we return is what gets sent back to the server as the completion. The server’s ctx.sample() call will receive this text encapsulated in a TextContent object.
A few things to note:

In our simple snippet, we used the system prompt. But we did not explicitly use params.temperature or params.maxTokens in the LLM call (though we could pass them, e.g. temperature=params.temperature etc.). A complete and robust handler would map all relevant parameters.
The RequestContext (ctx in parameters) can provide info about the request. In most cases for sampling, we won’t need it.
With the sampling_handler defined, the last step is to attach it to the MCP client. We will use FastMCP’s Client class to connect to the server, we can pass the handler like:


Above we use sse as transport, to use stdio simply replace the URL with the file name or path of the server code, as shown below:


If it's in the same folder-level location as the client, just the file name is enough, else the complete absolute path to the script needs to be specified.

When the server’s summarize_document tool invokes ctx.sample(...), the FastMCP client library will invoke our sampling_handler to service the request.

The model’s answer will be sent back to the server, which then continues execution and ultimately returns the summary.

In the snippet above, result would contain the tool’s output (which includes the summary text). We could then print or use that result as needed on the client side.

At this point, we have covered both sides: the server calls ctx.sample(), and the client provides a sampling_handler to generate the completion.


To summarize the entire flow of the MCP sampling mechanism, with the help of the above diagram:

Step 1: The client invokes a tool on the server.
Step 2: Within the tool logic, the server calls ctx.sample(...) to request an LLM-generated response.
Step 3: This sampling request is then packaged into a standardized MCP message.
Step 4: The request is transmitted over the transport layer (stdio or sse) from the server to the client.
Step 5: Upon receiving the request, the client invokes its sampling_handler() function.
Step 6: The sampling_handler() prepares the prompt and sends it to the client’s LLM.
Step 7: The LLM processes the input and returns the generated text.
Step 8 and 9: The sampling_handler() returns this generated result to the server, via the client, as a completion result.
Step 10: The server resumes the coroutine where the ctx.sample(...) was awaited, now continuing with the sampled output.
Step 11: The rest of the tool logic executes using the result provided by the sampling process.
Step 12: Finally, the server sends the final response back to the client, completing the interaction.
Now let's discuss how to craft the prompts and responses effectively, and how to leverage model preferences.

Model preferences
FastMCP’s sampling API includes a parameter called model_preferences which allows the server to hint or request a particular model (or models) for the client to use.

This is an optional feature, but it can be valuable in certain situations.

The idea is that the server might know that a certain task is best handled by a certain model, or the user might have specified a model choice, etc., and the server can pass that information along.

Ultimately, the client, based on its implementation, decides how to interpret these preferences. In an ideal setup, the client might honor it if possible, or ignore it if it does not have access to that specific model. However it may be, it provides a standardized way to convey model selection.

What can model_preferences be?

A string representing a model hint (for example, "gpt-4o" or any identifier the client would recognize).
A list of strings to suggest fallbacks or acceptable models in order of preference (e.g., ["gpt-4o", "claude-3-sonnet"] meaning “GPT-4o if you have it, otherwise Claude Sonnet”). Fallback logic for this must be implemented on the client side.
A ModelPreferences object (a more structured way, not often needed unless the MCP client has a complex selection mechanism).
👉
FastMCP converts any of the above formats into a ModelPreferences object before any further processing.
Using model preferences is straightforward. Here’s an example:


In this hypothetical analyze_sentiment tool, we provided model_preferences=["custom_model", "gpt-4o"]. Where "custom_model" is the name of a fine-tuned model the client might have for sentiment classification.

The client must be such that, if the client doesn’t support or recognize the custom model, it could fall back to "gpt-4o". Also, if neither is available, the client must just use its default model.

The client’s sampling_handler could check params.modelPreferences (in SamplingParams) and decide how to route the request. In our earlier sampling_handler example, we didn’t utilize params.modelPreferences; but a handler could do something like as shown below:


This code tries to extract the first model if a list is specified as preferences, or simply the only model if preference specified as a string.

The preferred_model got from here can be used to select a model API endpoint, something else like a local model, or trigger the fallback logic.

Usage guide
This feature is most useful when the server has knowledge that different models might produce better results for different tasks.

For example, maybe you have a tool that writes poetry, so you might prefer a model known for creative writing (and you might know the client has access to it).

Or you have a tool that requires very accurate, factual output then you might request a larger model if available.

Another use case is if the tool accepts a parameter from the user for model choice (imagine the user says “use GPT-4o for this query”); the server could pass that through via model_preferences. This gives advanced users more control while still funneling through the standard MCP interface.

👉
Keep in mind that clients must be implemented in a way such that they are not obligated to follow the preference. This is partially helped with FastMCP supporting model_preferences as optional parameter, but explicit fallback logics must be implemented on the client-side for building robust systems.
A well-behaved client will try, but if it does not have access to the specified model, it will just use its default or move ahead to check for a supported preferred model, in case multiple preferences were specified as a list.

Hence, the server logic should not assume the response quality or style is guaranteed just because the preference is set.

Model preferences are more of a hint or request. In practice, using model preferences is optional and if we don’t specify any, the client just uses its default model.

Sampling use cases
Sampling opens up many possibilities. Here are some common use cases where sampling shines:

Text summarization
As demonstrated above, a server can summarize documents or long texts by leveraging the client’s LLM. For instance, an MCP server might have access to some data and a tool for summarization.


The tool could fetch the data and then call ctx.sample to summarize it. The user gets a summary without the server itself needing a dedicated summarization model.

Complex Q&A or analysis
Sometimes a tool will involve logic + LLM reasoning. Suppose you have a sales data analysis tool that first pulls some numbers from a database, and then needs an explanation or insight from those numbers.

Data classification
The server might have tools to extract certain information from text or to classify text (sentiment, topic, etc.).

While trivial classification could be done with code, using the LLM might boost accuracy for nuanced cases.

Natural language to code
If a server allows users to ask for data in natural language, you could have a tool that uses the LLM to translate a request into a SQL query or code, and then executes it.


For instance, the tool might call ctx.sample with a prompt like “Translate this request into an SQL query: {query}” and then run the query on a database. Essentially, using the LLM as an on-demand translator.

Content transformation
Perhaps there is a resource that returns raw data or text, but you want to offer a tool that gives a more user-friendly version.

Using sampling, the server can take the raw output and ask the LLM to, say, “Put this into a bulleted list of key points.” The server then returns the transformed content. This way, the MCP server can present data in a variety of formats without hardcoding all those transformations.

Multi-turn assistants
In some advanced setups, the server might maintain a conversational state and use sampling to generate the next response in a dialogue.

However, note that usually the client is the primary conversational agent (the server just provides capabilities like tools). But if we implement an MCP server that itself acts like an agent (using an LLM via sampling internally), we could do that.

For example, a tool for chat replies keeps a log of past messages and calls ctx.sample with the conversation history to generate a reply.

This is somewhat meta (the client’s LLM generating a response that the client is going to show to the user, effectively turning the server into an AI echo), but it could be used if one wants the server to inject some controlled AI persona separate from the client’s main AI. It’s an unusual use case, so treat it as experimental.

In all these scenarios, the common theme is: whenever natural language understanding or generation is required as part of fulfilling a request, use sampling to leverage the client’s AI.

This can drastically reduce the complexity on the server side. Instead of writing complex logic or integrating separate ML models, we rely on the client’s general AI capabilities.

Advanced use cases and patterns
Beyond the typical patterns, there are advanced ways to use sampling in MCP to build more sophisticated workflows:

Chained reasoning calls
One may design a tool that calls ctx.sample multiple times in a sequence to break down a problem.

For instance, consider a math solver tool. One could first call ctx.sample("Think step by step about how to solve: {problem}") to get the LLM’s reasoning.

Then parse or validate that reasoning, and invoke ctx.sample again to get the final answer or explanation. Each call to ctx.sample could use the result of the previous call. This effectively uses the LLM in an iterative way.

Agentic behavior
We can combine tool usage with sampling to create agent-like behavior. For example, the server might have a researcher tool that internally:

Uses ctx.sample to brainstorm a list of questions about the topic.
For each question, invoke some API (via a normal tool call) to get information.
Collate the information and call ctx.sample again to ask the LLM to compile as a report.
Finally add persistence logic within the same or another tool to save it at a specified location.
Finally, ensure that advanced behaviors do not violate safety or privacy. For instance, if sampling is being used to summarize sensitive data from a secure source, know that the data is being sent to the client’s LLM. If the client’s LLM is an external service, that might have privacy implications.

One best practice is to inform the user (client side) when such things happen or give them control like, maybe the client only connects if they are okay with those data being processed by the model. MCP is all about clear contracts so we need to make sure that whatever we do follows the defined standards.

Error handling and best practices
Building robust MCP servers with sampling requires careful error handling and adherence to best practices. Let’s discuss some guidelines:

Handling exceptions
Although ctx.sample is simple to use, things can go wrong.

For instance, if the client’s sampling_handler throws an exception (maybe the LLM API is down, or an invalid parameter was given), the server will receive an error.


In FastMCP, if a tool function raises an exception (unhandled), the client will get an error response for that tool call. To avoid crashing and giving a poor user experience, we should catch exceptions around ctx.sample. For instance:


In this snippet, if ctx.sample fails, we catch the exception. We use ctx.error() to send an error-level log message back to the client, and then we return a fallback string.

By returning a normal string, our tool completes gracefully with a result (indicating failure). This is often better than letting the exception propagate, which would make FastMCP return an error object and a stack trace to the client.

Customize the fallback, as appropriate, might return an error code, or an apology message, etc., depending on the use case.

Validation of inputs
Before calling ctx.sample, validate any user input or data getting including in the prompt. Since the prompt might go to an external API, we don’t want to accidentally send something we shouldn’t.

Also, a malformed prompt (e.g., extremely large or empty when it shouldn’t be) could cause issues. For example, if a tool takes a URL and then does ctx.sample("summarize this URL: " + url_content), we must ensure that url_content isn’t ridiculously large (might summarize it in chunks instead), and ensure it’s properly sanitized in case it contains something that could confuse the LLM.

While prompt injection attacks are more of a client concern (the client’s LLM might be tricked by malicious content), as a server we should still be careful about blindly including user-provided text in system prompts or such.

Security considerations
Sampling means the server is sending data to the client’s LLM. Ensure that you trust the client or at least the client is the owner of the data being sent.

In many scenarios, the client is the end-user’s environment, so it’s their data or request anyway. But if your server holds sensitive data and the client asks to summarize it, by using sampling you are effectively sending that file’s content to the client’s LLM.

If that LLM is a third-party API, there could be privacy leaks. It may be fine if the user explicitly requested it, but you might want to provide warnings or require confirmation for such actions.

As a good practice, don’t enable sampling in tools that handle very sensitive operations unless necessary.

And consider implementing access controls. For instance, you can only allow sampling on certain data that the user is allowed to remove from the server’s premises.

Client-side handler errors
On the client side, inside sampling_handler, also use try/except around LLM calls (as we did in our example). If an exception occurs in the handler and isn’t caught, it will likely propagate back to the server as an error.

By catching it, we can return a controlled error message or even something else via fallbacks. This way, the server might still proceed, without terminating the execution.

By following these best practices, more resilient and user-friendly systems can be built. When done right, sampling in MCP can feel seamless: the user asks for something, the server maybe does a bit of work and then magically produces a result that was actually generated by the user’s own AI.

The better the errors are handled and the AI is guided with good prompts, the more it feels like a cohesive experience.

Conclusion
With this installment, we stepped beyond tools, resources, and prompts and dove into one of the most dynamic capabilities MCP offers called sampling.

Sampling empowers our systems to improvise, ask, generate, and reason in real time using the intelligence of the client's LLM.

This chapter was pivotal because it bridged two paradigms: deterministic protocol execution on the server, and probabilistic language-driven generation on the client.

To quickly recap:

We explored why sampling matters.
We examined how servers initiate sampling via ctx.sample(...), showing how this single call can unlock entire chains of inference while keeping server code lightweight.
On the client side, we implemented a full sampling handler, which allowed us to integrate our LLM backend and create a cleaner and more maintainable path.
This shift in approach, from manually wiring things in Part 3 to now utilizing FastMCP’s abstractions for client-side code is pivotal.

The earlier client was intentionally more explicit to illuminate the internals and lifecycle. But as we move forward, our focus is on streamlining and modularizing our implementations for robustness.

You might’ve noticed that while we covered essential code snippets, we didn’t walk through a full end-to-end project involving sampling.

That’s intentional since Sampling is an advanced concept and it deserves its own dedicated chapter focused on theoretical and foundational understanding.

In a future issue, we’ll demonstrate advanced workflows combining MCP capabilities with Agentic frameworks to build practically applicable Agentic systems.

Also, from here on, we’ve officially entered the advanced track of the MCP concepts. You now have the understanding of systems that don’t just execute; they think, generate, and adapt.

In upcoming chapters, we’ll deepen our capabilities with:

Sandboxing and security
Testing
Real-world use cases
By now, we have seen how MCP-enabled systems act, reason, reference, and generate, all within a clean, protocol-driven architecture. What started as a simple client-server interaction has grown into a full-fledged design for modular AI services.

As always, thanks for reading!

Any questions?

Feel free to post them in the comments.

Or

If you wish to connect privately, feel free to initiate a chat here:


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