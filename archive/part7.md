
Daily Dose of Data Science
Sponsor
Newsletter
More
Search 🔎
Zhuoxin Yang
Jul 6, 2025
The Full MCP Blueprint: Testing, Security, and Sandboxing in MCPs (Part B)
Model context protocol crash course—Part 7.

Avi Chawla
Akshay Pachaar
Avi Chawla, Akshay Pachaar

Recap of Part A
What's in Part B
Sandboxing MCP servers with Docker
Building a Docker-sandboxed server
MCP Inspector and Dockerized server
Conclusion
Recap of Part A
In the first part of this two-part chapter, we moved from functionality to safety, laying down the groundwork for testing and securing MCP systems.

We introduced the MCP Inspector, which acts as a live debugging interface for visualizing, replaying, and inspecting server behavior. It’s an essential tool for development, troubleshooting, and validating how your server behaves in real time.

We then dove into the security landscape of MCP, examining key risks and vulnerabilities:

Prompt injection via tool poisoning.
Server impersonation and tool name collisions.
Rug pull-style trust violations.
Overexposure of capabilities, especially via filesystem tools.
To mitigate these, we introduced MCP Roots, which let you define file access boundaries for your server and implemented them using FastMCP, alongside additional server initialization logic and client integration.

If you haven’t studied Part 6 yet, we strongly recommend going through it first. You can read it below:

The Full MCP Blueprint: Testing, Security and Sandboxing in MCPs (Part A)
Model context protocol crash course—Part 6.

Daily Dose of Data Science
Avi Chawla

With that security foundation in place, we’re now ready to tackle the final layer of protection: Sandboxing.

What's in Part B
In this second installment, we’ll focus on containerizing and sandboxing your MCP servers using Docker, ensuring that even if a server is compromised or receives unsafe input, the damage is confined.

We’ll cover:

How to containerize a FastMCP server from scratch.
Writing a secure Dockerfile, step by step.
Running containers with hardened settings, like read-only volumes, no network access, memory caps, and dropped Linux capabilities.
Integrating these sandboxed containers into your local tooling and IDEs:
Claude Desktop
Cursor IDE
Custom FastMCP clients
How to connect these sandboxed servers to MCP Inspector for real-time debugging inside a secure runtime.
By the end of this chapter, you’ll know how to:

Contain potentially dangerous execution flows.
Confidently share and run servers in untrusted environments.
Apply fine-grained resource controls to your AI toolchains.
Let’s dive in.

Sandboxing MCP servers with Docker
Running an MCP server (and its tools) directly on your machine means any code those tools execute (even potentially untrusted third-party code) runs with access to your local environment.

This poses security and stability concerns: the server could modify files, consume excessive resources, or conflict with your system’s environment.

Docker-based sandboxing addresses these issues by encapsulating the entire server in an isolated container environment.

Key benefits of containerizing the server include:

Security isolation: The server’s code and its tools run in a confined environment with limited access to the system. Docker containers have their own filesystem, process space, and networking by default, so even if an LLM-directed tool tries to do something dangerous, it’s constrained within the container.
Dependency and environment consistency: You can bake all required libraries and dependencies into the container image. This avoids the “works on my machine” problem and ensures the MCP server has the exact runtime it needs. The server won’t conflict with other Python packages on your system because it uses its own environment.
Reproducible deployment: A Docker image can be versioned and deployed anywhere (local machine, cloud VM, etc.) with identical behavior. This is particularly useful for MCP servers, which you might run locally during development and then later host remotely. With Docker, the transition is seamless.
Resource limiting control: Docker allows restricting CPU, memory, network, and even device access. You could run your container with limited CPU/RAM to prevent abuse, or with a read-only filesystem if the server doesn’t need to write files. These measures further sandbox the server’s capabilities.
In summary, sandboxing an MCP server via Docker means the entire MCP server (and all its tools) runs in an isolated container. This approach is often simple, robust, and also aligns with best practices for development.

In fact, there's also a repository that has containerized versions of 450+ MCP servers in a single repo!


No manual setup—just pull the image.
Safe to run in isolated containers, unlike scripts.
Auto-updated daily.
This makes it the easiest and safest way to use MCP servers with Agents.

GitHub - metorial/mcp-containers: Containerized versions of hundreds of MCP servers 📡 🧠
Containerized versions of hundreds of MCP servers 📡 🧠 - metorial/mcp-containers

GitHub
metorial

Moving on, to download and set up Docker on your system based on your operating system, check out the official guide linked below:

Get Docker
Download and install Docker on the platform of your choice, including Mac, Linux, or Windows.

Docker Documentation

Building a Docker-sandboxed server
Let’s walk through setting up a simple example server and then creating a Docker image for it.

Server code
For demonstration, we’ll use a minimal toy server with a single tool. In practice, your server could be more complex, but this example keeps it simple while covering integration, and you can easily extend it to complex use cases as needed.

On a side note:

You can download the primary codebase for sandboxing with Docker for an SSE server using the link below. Please refer to the open-me.ipynb notebook for detailed setup instructions.


Download below:

mcp_docker_zip
mcp_docker_zip.zip3 KB
Below is the server.py code that defines a FastMCP server with a tool for adding two numbers. We register the tool using the @mcp.tool decorator, and then run the server:


This server is minimal but sufficient to illustrate sandboxing.

Note that we have used host="0.0.0.0" and not 127.0.0.1 here, because when you start a server and bind it to 127.0.0.1 it only listens to connections from inside the container itself, or from localhost on your machine, if not containerized.

Whereas, 0.0.0.0 listens to all available network interfaces, including:

traffic from other containers
traffic forwarded via docker run -p
Hence, make sure that you set host="0.0.0.0".

Now that we have our toy server, next, let’s containerize it.

Writing a Dockerfile
To Dockerize the server, we create a Dockerfile that defines how to build a Docker image containing our server and all its dependencies.

Here’s a step-by-step breakdown of a suitable Dockerfile:


Let’s unpack what this Dockerfile does:

Base image: We use python:3.11-slim as a base. This is a smaller Python image that still has everything we need to run FastMCP.
Create user: We create a nonrootuser and later switch to it. Running the server as a non-root user inside the container adds a layer of safety. Even if the container is compromised, it’s not running as root, and even within the container, it has limited permissions.
Working directory: /app will be our application directory in the container.
Requirements installation: We install the necessary Python packages. If needed, a requirements.txt file can also be used instead of specifying directly.
Copy code: We then copy in the actual server code.
Switch user: After copying the code, we switch to the non-root user we created earlier.
Expose port: We expose port 8000. This is because we plan to run the server using sse transport on port 8000.
CMD: This is the default command that the container will run. We run the server code using CMD.
When the image is built, pip will install the listed packages. This means the container has everything required to run our server, independent of the host machine's Python setup.

To build the Docker image, ensure that Docker is running on your system. Then, navigate to your project directory (where the Dockerfile, server.py, and related files are located) and run the following command:


This builds the image and tags it as mcp-contained (you can choose any other name if you prefer). The . specifies the current directory as the build context, which must include server.py as referenced in the Dockerfile.

Running the server container
Once built, you can run the container. Since our CMD starts the server in SSE mode on port 8000, you can run the container with:


This will start the container. The MCP server is listening on port 8000 inside the container, and -p 5000:8000 maps it to port 5000 on your localhost.

Now the server is accessible at http://localhost:5000/sse. We could use this for clients that support connecting to remote SSE servers.

Additional sandbox restrictions
Docker’s default isolation is good, but you can tighten it further for untrusted code.

For instance, to impose resource limits like memory and CPU, we can use something like:


Capabilities can be dropped with --cap-drop=ALL, since containers typically don’t need most Linux capabilities.


You could also run the container with a read-only filesystem (--read-only) if the server doesn’t need to write anything.

These and many more such options/flags can be tailored to your specific code. The main point is that with Docker, you have the toolkit to heavily sandbox the environment in which the server tools run.

With our server now running in Docker, let’s see how to connect it with various clients like Claude Desktop, Cursor IDE, and custom client implementations.

Integrating Docker-sandboxed servers with clients
We’ll explore integration in three contexts:

Custom FastMCP client
Claude Desktop
Cursor IDE
Each has slightly different integration methods, and we’ll show how to use our Docker-sandboxed server in each scenario.

Custom FastMCP client integration
FastMCP provides support for custom client implementation, and that makes it easy to call MCP servers from code.

Let’s say we want to write a small Python script that queries our Dockerized server. If the above toy server is running, we can connect to it with a Client.

For example:


Note that we create Client pointed at "http://localhost:5000/sse". The MCP server is listening on port 8000 inside the container, and -p 5000:8000 had mapped it to port 5000 on the localhost. Hence, the server is accessible at http://localhost:5000/sse.

Before running this script, ensure the Docker container is up and running. Once this is ensured, we can run the client script and we'll observe that our client code executes successfully.

In a case where we are using stdio transport (i.e. just mcp.run() in the server), we need to perform some changes.

Firstly, we need to update our Dockerfile as follows:


We can observe that we've basically just removed the EXPOSE 8000 line from our earlier Dockerfile, rest all remains the same.

Then we'll build the Docker image, and ensure that Docker is running on your system. Then, navigate to your project directory (where the Dockerfile, server.py, and related files are located) and as we did earlier, run the following command:


This builds the image and tags it as mcp-server-stdio.

Next, we'll modify the client implementation a bit, so that it looks something like:


The config variable follows the same pattern that we use to set configuration when using Claude/Cursor as clients. The -i in the "args" refers to interactive mode. It is necessary when using STDIO transport, in order to keep stdin/stdout open.

After this, we can simply execute our client script and observe the output.

Next, let's see how to use a Dockerized server with Claude Desktop as a client.

Claude Desktop Integration
Claude Desktop supports local STDIO transport for connecting to servers. This is the traditional method (available to all Claude Desktop users). Claude communicates to the server via standard input/output.

👉
Remote MCP server support is currently in beta and available for users on Claude Pro, Max, Team, and Enterprise plans (as of June 2025). Most users will still need to use local STDIO connections.
As before, we write the configuration and save it in the claude_desktop_config.json file, following the structure and conventions discussed in earlier chapters.


After this, we'll simply restart Claude Desktop, and we'll be able to see our server visible in the UI, as shown in the screenshot below:


Once we're here, we can simply interact with our toy server using relevant queries, as shown in the image below:


With this, we're done with Claude-based integration.

Now let's move to integrating our Dockerized toy server into Cursor.

Cursor IDE Integration
Cursor supports MCP both ways: STDIO (launch a process) or SSE (connect to a URL). We can integrate our sandboxed server in either mode.

Go to Cursor settings and then "Tools & Integrations" option, and then click on "Add custom MCP", as shown in the screenshot below:


Then, put the following configuration into the mcp.json file:

To launch via STDIO:

For SSE:

Once you specify the required configuration, you'll see something like the following:


Once all this is done successfully, we can simply interact with the server in Cursor's chat (in "Agent" mode).

With this, we also covered the integration of a sandboxed server with Cursor.

At this point, we have a complete picture of how sandboxing works and how it helps keep our machines safer, especially when dealing with server tools that run potentially untrusted code from third-party sources or if there are tools that execute code, such as that generated by an LLM.

Having covered the core of sandboxing thoroughly, let's briefly check out how we could access our sandboxed toy server within MCP Inspector.

MCP Inspector and Dockerized server
When developing or deploying an MCP server in a sandbox, it’s important to verify that it works correctly and securely. The MCP Inspector can help us do this.

Connecting to the sandboxed server
First, launch the MCP Inspector in a terminal and navigate to the address displayed. Then, to connect to our sandboxed servers:

For SSE, we do something like:

Since MCP server is listening on port 8000 inside the container, but '-p 5000:8000' maps it to port 5000 on the localhost.
For STDIO:

Once we are able to successfully connect to our servers, we can perform experimentation with the available server capabilities in the same manner as discussed in the last chapter.

More specifically, here's the procedure for a quick recap:

Once we have Node.js up and running, launch the MCP Inspector with:


The first time we run this command, it will download and launch the MCP Inspector. On subsequent runs, it will start it directly.

By default, it will launch the web UI on http://localhost:6274 or http://127.0.0.1:6274 (port 6274). You’ll see a message in the terminal once it’s running, leave this terminal open and the command running. You'll be seeing something like:


Now, open the MCP Inspector in your browser using the address displayed in the CLI, which includes a prefilled session token like http://localhost:6274/?MCP_PROXY_AUTH_TOKEN=<token> :


This will take you directly to the MCP Inspector interface.

Next, we need to connect to our server, which you would have already done above.

Once the connection is established, we have a view similar to what is shown in the screenshot below from the previous part:


Once we select to list the capabilities in their respective tabs, we see that the interface will list all tools, resources, and prompts the server has.

We can click on tools/list or similar options for prompts and resources in the history panel, to see the description (the docstring we provided), the expected input schema, or any other relevant information.


👉
In the case of our sandboxed server, we can launch the MCP Inspector from any location, regardless of the transport method or whether Python packages are available in the system or a local virtual environment. This is because Dockerized servers are self-contained. They include all necessary dependencies and do not depend on our machine's environment.
Now we can simply run our tool with the required arguments directly from the MCP Inspector:


With this, we’ve also briefly explored how MCP Inspector can be used with sandboxed servers.

More broadly, we’ve covered the fundamental idea behind sandboxing and why it plays such a critical role, especially in isolating MCP servers. Sandboxing not only enhances security but also moves us closer to building robust, production-ready systems.

We encourage you to revisit the concepts discussed, experiment with different implementations and use cases, and explore various other Docker flags/options to reinforce and deepen your understanding of the material we covered here.

Conclusion
With this chapter, we concluded our deep dive into testing, security, and sandboxing in MCP systems.

While Part 1 focused on detecting vulnerabilities and validating behavior, Part 2 was about enforcing runtime boundaries that prevent those vulnerabilities from turning into exploits.

We explored how Docker offers a lightweight but powerful solution for sandboxing servers:

A controlled environment for running untrusted logic.
A portable packaging format for distributing secure tools.
An enforcement layer for memory, filesystem, user permissions, and network access.
We:

Built and configured a minimal Dockerfile tailored to FastMCP.
Discussed each instruction for clarity, security, and reproducibility.
Ran the container with hardened flags—--read-only, --cap-drop, --memory, --user, --no-network, and more.
Then, we bridged these containers back to the real world. You learned how to:

Run the server locally while keeping it sandboxed.
Use it seamlessly with Claude Desktop and Cursor IDE.
Integrate it with your own FastMCP clients.
And finally, we reconnected all of this to MCP Inspector, showing that testing and sandboxing aren’t two isolated goals—but complementary parts of the same security loop.

Together, these two chapters form a holistic guide for elevating your MCP servers from experimental scripts to production-grade, secure, testable deployments.

You now have the tools, practices, and patterns to:

Inspect your servers
Harden them against misuse
Contain them inside secure sandboxes
Ship them with confidence
Whether you're building internal tooling, user-facing workflows, or AI assistants, these principles will help ensure your systems are not only powerful but also safe, auditable, and trustworthy.

If you want to explore further, try building your own multi-tool sandboxed server using the Docker blueprint we shared.

Up next, we’ll explore recent developments in MCP, address some advanced patterns, and walk through real-world applications that combine sampling, tools, agent frameworks, and security in more complex deployments.

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