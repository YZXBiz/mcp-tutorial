Daily Dose of Data Science
Sponsor
Newsletter
More
Search 🔎
Zhuoxin Yang
Jun 15, 2025
The Full MCP Blueprint: Building a Full-Fledged MCP Workflow using Tools, Resources, and Prompts
Model context protocol crash course—Part 4.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar

Recap
Introduction
Resources
Prompts
Key distinctions b/w tools, resources, and prompts
A practical demonstration
Conclusion and next steps
Recap
Before we dive into Part 4 of the MCP crash course, let’s briefly recap what we covered in the previous part of this course.

In Part 3, we built a custom MCP client from scratch, integrated an LLM into it (which acted as the brain), and understood the full lifecycle of the model context protocol in action.

We also explored the client-server interaction model through practical implementations, observed how tool discovery and execution are handled dynamically, and contrasted MCP’s design with traditional function calling and API-based approaches.

We concluded Part 3 with some "try out yourself" style exercises to reinforce practical learning, while our discussion emphasized how MCP streamlines integration through its decoupled and modular architecture.

The hands-on walkthrough in Part 3 not only demystified MCP as a protocol but also highlighted its core strengths, like scalability, extensibility, and seamless tool orchestration.

By learning how tools are registered, discovered, and executed without tight API coupling, we saw how MCP allows developers to build adaptable and maintainable AI systems with ease.

If you haven’t explored Part 3 yet, we highly recommend doing so first since it lays the essential groundwork for what follows in this part. You can read it below:

The Full MCP Blueprint: Building a Custom MCP Client from Scratch
Model context protocol crash course—Part 3.

Daily Dose of Data Science
Avi Chawla

Introduction
Until now, our focus has primarily been on tools.

However, tools, prompts, and resources form the three core capabilities of the MCP framework.

While we introduced resources and prompts briefly in Part 2, this part will deep-dive into their mechanics, distinctions, and implementation.

We now shift gears to explore resources and prompts in detail and bring clarity to the key ideas around resources and prompts, like how they differ from tools, how to implement them, and how they enable richer, more contextual interactions when used in coordination.

By the end of this part, you'll have a practical and intuitive understanding of:

What exactly are resources and prompts in MCP
Implementing resources and prompts server-side
How tools, resources, and prompts differ from each other
Using resources and prompts inside the Claude Desktop
A full-fledged real-world use case powered by coordination across tools, prompts, and resources
Every concept will be explained through clear examples and walkthroughs to develop a solid understanding.

Let’s begin!

Resources
Resources are one of the fundamental primitives of the model context protocol (MCP).

While tools are about doing (executing actions), resources are about knowledge.

They expose contextual data to language models, allowing them to reason without causing any side effects.

👉
In programming and systems, a "side effect" refers to any observable change in system state beyond simply returning a value. This includes actions like writing to a file, updating a database, making an API call, modifying external systems or resources, etc.
Intuition
Resources are read-only interfaces that expose data as structured, contextual information.

They act as intelligent knowledge bases or reference libraries, allowing models to access and reason about information.

Think of resources as a well-organized library: they provide access to books (data) that can be read and referenced, but the books themselves cannot be altered through the reading process.

👉
There's a common misconception that resources mean data. However, it’s important to remember that a resource is not the data itself, but rather an interface that provides access to data in a structured and controlled manner.
What can resources represent?
Resources can represent various types of information:

A file's contents
An API's cached response
A database snapshot
An extracted snippet from a document
System logs, configurations, or documentation
Since resources should not be used to execute or modify data, they offer a safe and predictable way to supply context into AI workflows, especially when dealing with static or semi-static knowledge.

👉
If the underlying data changes, the updated content won’t automatically propagate into the model’s context. The resource must be reloaded into context to reflect the new state.
URI-based resource identification
Resources are identified through unique URIs (uniform resource identifiers) following a structured format:

The protocol and path structure are entirely specific to the MCP server implementation, allowing servers to create custom URI schemes that fit their specific use cases.

For example:

Resource types
Resources can contain two distinct types of content: text or binary data. This gives us text resources and binary resources.

Let's understand them!

Text resources
Text resources contain UTF-8 encoded text data, suitable for:

Source code files
Configuration files
Log files
JSON/XML data
Plain text documents
Binary resources
Binary resources contain raw binary data encoded in base64, suitable for:

Images and graphics
PDF documents
Audio and video files
Any non-text file formats
Resource discovery mechanisms
MCP provides two complementary approaches for resource discovery:

Direct resources
In this mechanism, an MCP server can expose concrete resources through the resources/list endpoint, which also provides metadata including URI, human-readable name, optional description, MIME type, and size information.

This approach works well for known, static resources that are always available.

👉
By static resources we refer to resources having fixed URIs.
Resource templates
For dynamic content, servers can expose URI templates like file://{path}, following RFC 6570 standards.

👉
RFC 6570 is a document published by the Internet Engineering Task Force (IETF) that defines the syntax and rules for URI Templates.
Resource templates are particularly powerful for scalable implementations.

Instead of registering thousands of individual files, a server can expose a single template that covers entire families of resources, dramatically reducing complexity while maintaining full functionality.

👉
It's okay if you don't understand this. Everything will become clear once we dive into the implementation.
Application-controlled access pattern
A crucial aspect of MCP resources is their application-controlled access pattern.

Unlike model-controlled primitives such as tools, resources require explicit client-side management.

To elaborate, when tools are invoked, the LLM returns the required tool call. The client, after permission from the user, invokes the tools that reside in the MCP server and receives the response. This depicts that the model is in control and the model decides what it is to be invoked:

But with resources, the client application must explicitly fetch and manage the data from the resources before providing it to the LLM, without the LLM initiating any action itself. This shows that the application is in the driver's seat for resources.

This design choice provides several important benefits.

Different MCP clients handle resource access differently based on their specific requirements and security models.

For example, Claude Desktop requires users to explicitly select resources before use, ensuring conscious decision-making about data access.

Other clients might implement automatic selection based on heuristics or context, while advanced implementations may eventually allow AI models to determine resource access independently.

This controlled access pattern ensures resources are used purposefully rather than arbitrarily, providing better security, performance, and predictability in AI interactions.

It also prevents accidental data exposure and helps maintain clear boundaries between different types of information.

Defining resources in the MCP server
We can register both static resources (fixed URIs) and dynamic ones (resources via parameterized templates; dynamic URIs) using the @resource decorator provided by fastmcp.

A static resource has a fixed URI and always returns the same content. Ideal for documents, help texts, or configuration, it’s registered like so:

In a similar manner, we can use templates like file://{path} to enable flexible resource construction, allowing clients to generate dynamic, valid URIs based on runtime parameters.

Resource metadata customization
When registering a resource, we can specify the resource's properties using arguments in the decorator:

Duplicate Resources
FastMCP allows us to control how duplicates are managed during resource registration:

The duplicate behavior options are:

"warn" (default): Logs a warning, and the new resource/template replaces the old one.
"error": Raises a ValueError, preventing a duplicate registration of resources under the same URI.
"replace": Silently replaces the existing resource/template with the new one.
"ignore": Keeps the original resource/template and ignores the new registration attempt.
This comprehensive resource system in FastMCP provides a robust mechanism for exposing contextual data to AI models.

Prompts
In the MCP framework, prompts are another key primitive that encapsulate reusable instructions for LLMs.

They encapsulate predefined roles, reasoning workflows, or multi-turn conversation scaffolds that clients can explicitly invoke to structure LLM interactions in a controlled manner.

While tools let models act and resources let them know about info, prompts guide how models approach a task (i.e., think), shaping their role, reasoning strategy, and output in a structured, reusable, and discoverable way.

Intuition
Prompts in MCP are predefined message templates exposed by the server and invoked explicitly by clients. They serve as:

Instructional blueprints (e.g., “explain code”, “summarize document”),
Domain-specific reasoning setups (e.g., “legal contract reviewer”),
Multi-turn conversational flows (e.g., “diagnostic interview”),
Task-specific agents (e.g., “git commit message generator”).
Standardized output formats (e.g., JSON, markdown, or bullet points).
👉
Prompts are selected by the user and set the context before the model begins generating.
Why have prompts?
The reason is simple.

Prompts enable consistent, server-defined strategies for guiding LLMs. The value of MCP prompts lies in centralized definition and standardized behavior:

The user does not have to write detailed prompts manually again and again.
Clients can discover available prompts via /prompts/list.
Updates to prompt logic can be deployed server-side.
Domain-specific prompts (e.g., legal, medical) can encapsulate expert workflows.
They can accept resources and parameterized inputs.
Prompts with FastMCP
In Python-based MCP servers using FastMCP, prompts are defined using the @mcp.prompt decorator. These decorated functions return message templates that get injected into the model's input.

Here's how one can define a very basic MCP prompt that returns a string:

We can also define prompts of a specific message type. Let's see how we can define such a prompt:

In the above MCP prompt, based on the user's input code generation query, a prompt template is generated and returned to the LLM.

We return an object of the class PromptMessage, which accepts two parameters:

role, which follows the traditional format for invoking LLMs with message history. Since a prompt encapsulates the user's original query, we specify the role as "user".
content, which is a TextContent object, which holds the content object created above using the language and task_description parameters.
Return values
FastMCP intelligently handles different return types from the prompt function we defined above.

For instance, if the return type is:

str: It is automatically converted to a single PromptMessage object.
PromptMessage: It is used directly as provided. (Note: A more user-friendly Message constructor is available that can accept raw strings instead of TextContent objects.)
list[PromptMessage | str]: This is used as a sequence of messages (a conversation) like we specify in regular LLM calls.
Any: If the return type is not one of the above, the return value is attempted to be converted to a string and used as a PromptMessage.
To elaborate more on the sequence of messages we discussed above, here's how to return multi-turn conversations with PromptMessage:

That said, it can be simplified by using the Message constructor provided:

Type Annotations
Suppose we define a content generation prompt that accepts a topic as a required string, a formatting parameter from a predefined set of options, and optional parameters for tone and word_count.

Then we can use regular Python type annotations in the following manner:

Next, we can define the prompt using these parameters, like we learned earlier:

Type annotations play a crucial role in FastMCP prompts, serving multiple purposes beyond simple documentation.

They inform FastMCP about expected parameter types, enable automatic validation of inputs received from clients, and are used to generate the prompt's schema for the MCP protocol.

This ensures that when clients invoke prompts, the parameters are properly validated before the function executes.

In fact, this is not just about MCP.

If you recall the AI Agents crash course, where we implemented the multi-agent pattern from scratch, we did something similar there as well:

Implementing Multi-agent Agentic Pattern From Scratch
AI Agents Crash Course—Part 12 (with implementation).

Daily Dose of Data Science
Avi Chawla

Prompt metadata
While FastMCP automatically infers prompt names and descriptions from function names and docstrings, we have full control over these metadata elements.

Similar to resources, the @mcp.prompt decorator accepts several parameters that allow us to customize how the prompt appears to the client.

We can override the prompt name, provide a custom description that's more client-friendly than the internal docstring, and add categorization tags.

Tags are particularly useful for organizing prompts in applications with many available options. They enable clients to filter, group, or search prompts based on functionality, domain, or use case.

Here is an example of how we can define our custom metadata:

Disabling prompts
We can control the visibility and availability of prompts by enabling or disabling them.

Disabled prompts will not appear in the list of available prompts, and attempting to call a disabled prompt will result in an Unknown prompt error.

By default, all prompts are enabled.

We can disable a prompt upon creation using the enabled parameter in the decorator like @mcp.prompt(enabled=False).

Duplicate prompts
In a manner similar to resources, we can configure how the FastMCP server handles attempts to register multiple prompts with the same name.

We utilize the on_duplicate_prompts setting during FastMCP initialization.

Same as on_duplicate_resources, the behavior options for on_duplicate_prompts are:

"warn" (default): Logs a warning, and the new prompt replaces the old one.
"error": Raises a ValueError, preventing a duplicate registration.
"replace": Silently replaces the existing prompt with the new one.
"ignore": Keeps the original prompt and ignores the new registration attempt.
👉
Analogous to on_duplicate_resources and on_duplicate_prompts, we have on_duplicate_tools setting as well.
Prompts, resources, and tools together form a powerful and composable foundation for building intelligent, modular AI workflows with the Model Context Protocol.

Now that we are well-familiar with all three, let's take a moment to study some of the key distinctions among them.

Key distinctions b/w tools, resources, and prompts
The fundamental distinction lies in who controls what and when things happen:

Tools: Model-controlled actions (LLM decides to execute; requires user permission)
Resources: Application-controlled context (system provides information; may require users to select)
Prompts: User-controlled interactions (human initiates workflows)

Tools represent active, functional operations that enable AI models to perform actions. They embody the "do something now" pattern.
Key characteristics include:
Dynamic operations: Perform real-time computations, data fetching, or processing.
Active execution: Tools trigger immediate actions with potential side effects.
AI-initiated: The language model decides when and how to invoke tools.
Safety and control: Requires explicit user permission.
Real-time results: Return current, computed, or fetched data.
To elaborate more on "when to use tools", while there are no strict rules, it's helpful that we have certain development practices aligned with MCP. Use tools when:
Data is dynamic and requires real-time fetching
Computation is needed based on user input
External APIs must be called with specific parameters
Actions need to be performed
Filtering or processing is required before returning results
Next, resources provide passive, contextual information that serves as reference material for AI reasoning. They represent the "here's some information" pattern.
Key characteristics include:
Contextual: Provides background information for AI reasoning
Read-only: Typically consumed as-is without computation
Application-controlled: The system determines what resources are available
Reference material: Used to inform AI responses rather than trigger actions like tools
To elaborate more on "When to use resources", just like tools, we can follow a rule of thumb for when to use resources. This helps us cover a wide range of use cases and maintain a clear separation from tools and prompts while staying true to MCP principles. Hence, use resources when:
Data is static or changes in a known manner or infrequently
Documentation or guidelines need to be referenced
Context is needed for AI to understand domain-specific information
Background knowledge for reasoning is needed
Pre-existing datasets should be made available
Finally, prompts provide templated workflows and user interactions. They guide user interactions with AI models. They represent the "here's how to interact" pattern.
Key characteristics include:
User-initiated: Humans choose when to invoke prompt templates
Templated: Pre-defined structures with parameters
Workflow-oriented: Guide specific types of interactions or tasks
Reusable: Standardized patterns that can be applied repeatedly
Interactive: Often includes placeholders for user customization
To elaborate more on "When to use prompts", similar to tools and resources, we can have a few heuristics for using prompts. Use prompts when:
Standardized workflows benefit from a consistent structure
Complex interactions need guidance
Domain expertise can be captured in reusable patterns
User onboarding requires guided experiences
Efficient patterns should be embedded in interaction patterns
To better understand, consider the following use cases:

In a software development assistant:

Tools can be used for code generation, execution, and testing.
Resources can be used for language documentation and company standards.
Prompt templates can be used for bug analysis.
In a customer service platform

Tools can be used for real-time order lookup and return/refund investigations.
Resources can be used for platform policies regarding customer issues and satisfaction.
Prompt templates can be used for handling complex customer issues that might require personalized communication.
Overall, the distinctions between tools, resources, and prompts in MCP reflect fundamental differences in purpose, control, and data characteristics.

Understanding these distinctions is important since it enables developers to build more effective, secure, and user-friendly AI integrations while adhering to the model context protocol.

The key is matching the primitive to the specific use case based on data dynamism, required actions, and control patterns.

Now that we have a holistic understanding of tools, resources, and prompts, let's build a server that brings it all together, demonstrating their effective use in practice.

A practical demonstration
To reinforce the ideas we discussed, let's implement a real-world MCP server example using Claude Desktop as the client/host, centered around job search and analysis.

More specifically, we shall implement an MCP server that demonstrates coordinated use of tools, resources, and prompts to power an intelligent job search assistant.

Tools:
A tool that fetches job listings using an external API based on parameters like role and location. It stores the raw results and returns key details for the model to generate a response.
Another tool that allows the user to save specific jobs of interest in a clean, structured format with essential information from the raw results.
Resources:
A resource exposing the user’s saved jobs.
A resource exposing the user’s resume.
Prompts:
A prompt that analyzes a given number of jobs for a given role and location to uncover trends.
A personalized recommendation prompt that uses the resume to recommend tailored roles, skills, and companies. The resume should be explicitly loaded along with it, as a resource.
Another prompt to summarize how well the resume matches with the saved job data. Both the resume and saved jobs will be explicitly loaded as resources along with the prompt.
This diagram depicts the workflow below:

The user inputs the query through the MCP host (Claude Desktop).
This goes to the MCP server, which, based on the query, decides to invoke the relevant tools/resources/prompts in the correct order to generate a response.
This is returned to the MCP host, where the LLM uses that data to generate a response for the user.
Setup
We'll be using the JSearch API to fetch job postings in real time. The free tier offers 200 API calls per month, which is sufficient for development and basic experimentation.

You can obtain your own API key by visiting: https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch.

💡
Open the above link → click on the blue button in the top right corner → subscribe to free Basic plan → Find the "X-RapidAPI-Key" after subscribing.
Next, you need to download and set up Claude Desktop on your system:

https://claude.ai/download
👉
Note, we aren't using Cursor in this demonstration since at the time of publishing this part of the course, it lacks support for prompts and resources.
The next step is project initialization:

Then move inside the project directory with cd job_server_project and initialize a virtual environment with uv venv. Once this is done, activate the virtual environment with:

Install the following packages:

Done!

Server program
Now, create a file named server.py and follow along!

You can download the code for this whole article, along with a full readme notebook with step-by-step instructions below.

It contains a few Python files and a notebook file:

Download below:

Prompts-and-Resources
Prompts-and-Resources.zip4 KB
Firstly, add the following imports at the top of server.py code file:

Then set up storage locations and API settings:

We can see we have specified a path for the resume. Hence, we need to store a resume at ./resume as resume.pdf, for it to be fetched as a resource.

For the purpose of this walk-through, you can use this dummy resume:

resume
resume.pdf96 KB
Next, we initialize the MCP server:

Now we'll define our first tool search_jobs that accepts the role, location, and the number of results to return.

Tools
We start by declaring the function, decorating it, and defining the docstring:

Next, we invoke the job search endpoint to get the relevant jobs and store them in the job_list object:

If no results were found, we simply return here:

Otherwise, we first store the data locally:

Finally, we return the relevant information from this tool:

And that's our search_jobs tool, which takes a role, location, and an optional max_results parameter.

It builds the API query string dynamically based on user input, invokes the JSearch API to get relevant job listings by sending the request and extracts the first max_results jobs.

It also saves the full raw response to disk for later use.

Finally, the tool returns a clean, structured summary of each job, ready for use with an AI model.

👉
Job descriptions are often lengthy. So we sliced them: keeping the first 1000 characters and (if long enough) the last 1000 characters, a quick heuristic to preserve both the introduction and fine print, while keeping in mind the LLM context window.
Each returned job dictionary includes: job_id, title, company, location, description (sliced) and apply_link.

Then we define our second tool, save_job. It will be invoked after the search job method to further process and save the job data retrieved from the search method.

We start by declaring the function, decorating it, and defining the docstring:

Next, we open the temporary JSON file created in the search_jobs tool:

Moving on, we select the specific job_id from the above JSON:

Recall that in search_jobs tool, we returned several key details like job_id, title, company, location, description, and apply_link:

Thus, when the LLM on the client side decides to invoke the save_job method after search_job, it will have visibility over all that data, which it will use to prepare the function call of the save_job method.

Thus, the capabilities of this LLM can be used to automatically determine the salary from the available details (from job description, for instance) and pass that to the save_job method.

Thus, if the LLM provides salary data, we store that as final_salary:

If the salary data were not available in the function call prepared by the LLM, we can try parsing structured fields from JSON, and if still nothing is found, we specify final_salary as "not specified"

Once done, we prepare the final structured data and store it:

To recap, in this tool, we first validate if the job_id we are trying to save is a valid one or not. If invalid, we exit, returning a not found message.

If valid, we check if the LLM supplied the salary parameter by intelligently extracting it from the description of the job. Note that we do not implement any parsing logic for this.

If the LLM provided a value, we store it as the salary for that corresponding job. If it didn't, we try to find it from the stored raw JSON entry of that job provided by the API.

If we find something, we store it as salary, else we default to "Not Specified".

Then we compose the job data in a clean, structured manner and save it as a json file. We finally return the success message.

Now that we have our tools, let's take a look at the required resources.

Resources
We need to set up two resources, one for the resume and another for exposing the saved data.

First, we define our resume resource:

Our code is using pypdf2 for reading our resume PDF file. PdfReader extracts content from our resume file and returns it as markdown.

Next, we add our second resource. This resource exposes the saved job data:

This resource fetches all json files, each representing one job. These are the jobs saved by the candidate with the help of LLM. If there is no data available, it returns "# No saved jobs found." else loads the data in the content variable as markdown and returns content.

Finally, we need to define our prompts, and then we are almost ready.

Prompts
We'll be defining three prompts: one for job market analysis, another for personalized job recommendations, and then finally for creating a match report based on resume and jobs.

Firstly, we define the analyze_job_market prompt:

This prompt accepts the role, location and num_jobs parameter. If the user uses this prompt, it will automatically invoke search_jobs tool (we have specified that in step 1 of the prompt) and prepare an analysis report based on the collected data.

Next up, we define the personalized_job_recommender prompt:

When we are using this prompt in Claude Desktop, we need to ensure that we provide the resume resource along with.

Now, we finally define our last prompt and last of all capabilities for this server script.

This prompt creates a report of how well the resume matches to the preferred (saved) jobs. We define it as create_match_report:

Similar to the above prompt, we need to make sure that we provide the resume and saved jobs data as resources to Claude Desktop when using this prompt.

Now, after we have defined all the server capabilities, we end the server code with:

Note, we need to ensure that we use stdio as the transport mechanism, since as of now (at the time of publishing of this part), Claude Desktop does not support the sse transport mechanism.

Connecting to Claude Desktop
With our server's code in place, let's go ahead and connect our server to Claude Desktop.

Inside Claude Desktop, go to settings and then the "Developer" tab.

Click on "Edit Config" and then inside claude_desktop_config.json add the following stdio configuration:

If you are on a Mac device, do this instead:

Once done, close Claude Desktop and re-open it. One should see the available capabilities as shown in the screenshots below:

The above screenshot shows available tools, and the one below shows the available prompts and resources.

👉
If you're on Windows and restarting the Claude Desktop application does not work for you, just run its installer file (.exe) once and you should be good to go.
Once this is done, we can start interacting with our capabilities via Claude Desktop.

To use resources and prompts in Claude Desktop:

Click the “+” icon in the interface.
Select “Add from mcp_job” from the dropdown menu.
Click on the desired resource or prompt to add it.
Suppose if we want to use analyze_job_market prompt, once we follow the above steps, a popup as the one shown below appears:

You can provide the required parameter values and then simply press enter.

The above screenshot shows the analyze_job_market prompt as per the values entered by us.

Below, we have attached several screenshots showcasing our outputs, where we utilize a combination of capabilities in different types of usage intent and scenarios:

The above screenshot involves the use of the job market analysis prompt to invoke the job search tool for 10 AI engineer roles in Lucknow and create an analysis report.

In the screenshot above, we demonstrate the job-saving tool by randomly saving the first five jobs from the ten we got earlier.

The screenshot above shows how we can use the resume and saved jobs resources along with the job match prompt to get a match report on the suitability of the various saved jobs, as per the resume. The screenshot below is a continuation of the same response.

Similarly, various other possibilities and combinations can be tried out by the use of a mix of prompts, resources and tools.

👉
Note that these screenshots are attached to provide an idea of things, and do not show the complete responses. In your usage, the patterns and responses will differ based on your data and the LLM's understanding of the same. However, the behavior will be grounded and predictable.
We encourage you to try and experiment with more queries and combinations, and observe how the capabilities work differently yet complement each other.

Conclusion and next steps
With this part, we’ve officially expanded our MCP knowledge from tools to the powerful abstractions of prompts and resources, which are considered as two cornerstones that elevate the MCP framework from being just a task execution layer to becoming a context-aware and intelligent interface protocol.

While previous parts established how tools serve as side-effectful operations that "do" things, this installment clarified how resources and prompts serve as the essential “knowledge” and “reasoning” layers of any agentic system.

By introducing resources, we’ve brought a structured, read-only context into scope, allowing LLMs to see the world as it is, without altering its state. These are not just pieces of data but context interfaces that expose knowledge in ways models can use intelligently and consistently.

To recap:

Resources give your AI agents something to refer to, something like a firm ground/knowledge to reason on the back of.
On the other hand, prompts extend the expressiveness of your system. They’re not merely templates, but reconfigurable mental models you provide to LLMs.
We saw how prompts act as reusable, parameterized instructions, letting us define interactions that are composable, interpretable, and shareable.

With prompts, we’re now enabling LLMs to not only act based on tools and data but to adapt their tone, purpose, and objective in a consistent, structured way.

Together, prompts and resources unlock a critical trifecta in the MCP universe:

Tools: for doing
Resources: for knowing
Prompts: for thinking
From here on out, we begin to see MCP not as a technical curiosity but as a language for designing intelligent systems.

This chapter also illustrates how intuitive and powerful FastMCP makes this journey. Whether it was defining resources like a local resume file or prompts like job market analysis, the protocol’s flexibility remained intact.

But this simplicity masks an enormous amount of design value. Your system is now on track to reason across contexts, adapt behavior dynamically, and scale across changing workflows.

Even from this foundational prompt-resource setup, the groundwork for deeply agentic behavior is now in place.

To recap our learning path:

Part 1 grounded us in what MCP is and why it exists.
Part 2 unpacked the lifecycle, protocol design, and server-side design.
Part 3 gave us a complete walkthrough of a working client that connects, discovers, and invokes tools.
This chapter (Part 4) integrated prompts and resources to make AI systems that are not just functional, but deeply contextual and intelligent.
In the upcoming chapters, we’ll go even deeper. Topics will include:

Sampling workflows
Sandboxing
Testing and debugging
Real-world deployments and integrations
Ultimately, the goal is not just to build a working demo, but to showcase how MCP can support large-scale, maintainable, composable AI systems in production, ones that can evolve with needs, data, and models.

This has been a pivotal chapter, where we provided the necessary build-up to understand that we use tools to act, resources to observe, and prompts to shape cognition.

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
