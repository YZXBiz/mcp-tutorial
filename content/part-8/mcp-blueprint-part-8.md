---
title: "The Full MCP Blueprint: Part 8 - Practical MCP Integration with 4 Popular Agentic Frameworks"
subtitle: "Model context protocol crash course—Part 8"
authors:
  - name: Avi Chawla
  - name: Akshay Pachaar
date: "July 13, 2025"
source: "Daily Dose of Data Science"
---

# The Full MCP Blueprint: Part 8 - Practical MCP Integration with 4 Popular Agentic Frameworks

_Model context protocol crash course—Part 8._

**Authors:** Avi Chawla, Akshay Pachaar  
**Date:** July 13, 2025  
**Source:** Daily Dose of Data Science

---

## Table of Contents

```{contents}
:depth: 3
```

---

## Recap

Before we dive into Part 8 of the MCP crash course, let's briefly recap what we covered in the previous part of this course.

In Parts 6 and 7, we explored testing, security, and sandboxing in MCP-based systems. We began by examining MCP Inspector, a powerful tool that allows developers to visualize and debug interactions with MCP servers effectively.

Next, we discussed several key security vulnerabilities in MCP systems, including prompt injection, tool poisoning, and rug pull attacks, along with strategies for mitigating these risks.

We also explored MCP roots and how they can help limit unrestricted access, reducing potential security threats.

We then turned our focus to containerization and sandboxing techniques for MCP servers to enhance security and ensure reproducibility.

This included writing secure Dockerfiles for FastMCP servers, implementing hardened settings such as read-only filesystems, memory limits, and other container-level restrictions to reduce the attack surface.

Additionally, we covered how to integrate sandboxed servers with tools like Claude Desktop, Cursor IDE, and custom clients.

Finally, we demonstrated how to test these sandboxed servers using MCP Inspector, including connecting both STDIO and SSE-based sandboxed servers for debugging and inspection.

By the end of Part 7, we had a clear picture of how to examine and build secure environments for end-to-end cooperative intelligence systems developed with MCP.

```{admonition} Prerequisites
:class: note
If you haven't studied Part 7 yet, we strongly recommend going through it first. You can read it here: [The Full MCP Blueprint: Part 7 - Testing, Security, and Sandboxing in MCPs (Part B)](../part-7/mcp-blueprint-part-7.md)
```

---

## What's in This Part

In this chapter, we'll explore how MCP integrates with popular agentic frameworks, demonstrating its versatility and power in existing AI ecosystems.

We'll examine:

- **LangGraph** - State-based agent workflows
- **LlamaIndex** - RAG and knowledge management
- **CrewAI** - Multi-agent collaboration
- **PydanticAI** - Type-safe agent development

For each framework, we'll cover:

- Integration patterns and capabilities
- Implementation examples
- Benefits and limitations
- Best practices for MCP usage

---

## Framework Overview

MCP's standardized protocol makes it an excellent addition to existing agentic frameworks. Rather than replacing these frameworks, MCP enhances them by providing:

- **Standardized tool interfaces** across different agent systems
- **Reusable server components** that work with multiple frameworks
- **Consistent security models** for tool execution
- **Interoperability** between different AI systems

```{mermaid}
graph TB
    subgraph "Agentic Frameworks"
        A["LangGraph"]
        B["LlamaIndex"]
        C["CrewAI"]
        D["PydanticAI"]
    end

    subgraph "MCP Layer"
        E["MCP Protocol"]
        F["Tool Servers"]
        G["Resource Servers"]
        H["Prompt Libraries"]
    end

    subgraph "External Services"
        I["Databases"]
        J["APIs"]
        K["File Systems"]
        L["Cloud Services"]
    end

    A --> E
    B --> E
    C --> E
    D --> E

    E --> F
    E --> G
    E --> H

    F --> I
    F --> J
    G --> K
    H --> L

    style E fill:#ffff99
    style F fill:#99ccff
    style G fill:#99ccff
    style H fill:#99ccff
```

---

## LangGraph Integration with MCP

LangGraph excels at building complex, stateful agent workflows. MCP enhances LangGraph by providing standardized tool interfaces and dynamic capability discovery.

### Basic Integration

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolExecutor, ToolInvocation
from langchain_openai import ChatOpenAI
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
import asyncio

class MCPLangGraphAgent:
    def __init__(self, mcp_server_script: str):
        self.mcp_server_script = mcp_server_script
        self.mcp_session = None
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
        self.tools = []

    async def initialize_mcp(self):
        """Initialize MCP connection and discover tools."""
        server_params = StdioServerParameters(
            command="python",
            args=[self.mcp_server_script]
        )

        # This is a simplified version - in practice you'd manage the connection lifecycle
        self.stdio_client = stdio_client(server_params)
        read, write = await self.stdio_client.__aenter__()
        self.mcp_session = ClientSession(read, write)
        await self.mcp_session.initialize()

        # Discover available tools
        tools_result = await self.mcp_session.list_tools()
        self.tools = tools_result.tools

    async def call_mcp_tool(self, tool_name: str, arguments: dict):
        """Call an MCP tool and return the result."""
        result = await self.mcp_session.call_tool(tool_name, arguments)
        return result.content[0].text if result.content else "No response"

    def create_graph(self):
        """Create a LangGraph workflow with MCP tools."""

        def agent_node(state):
            """Agent reasoning node."""
            messages = state["messages"]

            # Add available tools to the prompt
            tool_descriptions = "\n".join([
                f"- {tool.name}: {tool.description}"
                for tool in self.tools
            ])

            system_msg = f"""You are a helpful assistant with access to these tools:
            {tool_descriptions}

            When you need to use a tool, respond with: TOOL_CALL: tool_name: arguments_as_json
            """

            response = self.llm.invoke([
                {"role": "system", "content": system_msg},
                *messages
            ])

            return {"messages": messages + [response]}

        async def tool_node(state):
            """Tool execution node."""
            last_message = state["messages"][-1]

            if "TOOL_CALL:" in last_message.content:
                # Parse tool call
                tool_call = last_message.content.split("TOOL_CALL:")[1].strip()
                tool_name, args_json = tool_call.split(":", 1)

                try:
                    import json
                    arguments = json.loads(args_json.strip())
                    result = await self.call_mcp_tool(tool_name.strip(), arguments)

                    tool_response = f"Tool {tool_name} returned: {result}"
                    return {"messages": state["messages"] + [
                        {"role": "assistant", "content": tool_response}
                    ]}
                except Exception as e:
                    error_msg = f"Tool execution failed: {str(e)}"
                    return {"messages": state["messages"] + [
                        {"role": "assistant", "content": error_msg}
                    ]}

            return state

        def should_continue(state):
            """Decide whether to continue or end."""
            last_message = state["messages"][-1]
            if "TOOL_CALL:" in last_message.content:
                return "tools"
            return END

        # Build the graph
        workflow = StateGraph({"messages": list})

        workflow.add_node("agent", agent_node)
        workflow.add_node("tools", tool_node)

        workflow.set_entry_point("agent")
        workflow.add_conditional_edges(
            "agent",
            should_continue,
            {"tools": "tools", END: END}
        )
        workflow.add_edge("tools", "agent")

        return workflow.compile()

# Usage example
async def run_langgraph_mcp_example():
    agent = MCPLangGraphAgent("research_server.py")
    await agent.initialize_mcp()

    app = agent.create_graph()

    # Run the workflow
    result = await app.ainvoke({
        "messages": [{"role": "user", "content": "Search for recent AI papers and summarize the top 3"}]
    })

    print("Final result:", result["messages"][-1]["content"])

# Run the example
# asyncio.run(run_langgraph_mcp_example())
```

### Advanced LangGraph-MCP Patterns

```python
class AdvancedMCPLangGraph:
    def __init__(self):
        self.research_tools = []
        self.analysis_tools = []
        self.reporting_tools = []

    def create_research_workflow(self):
        """Create a multi-stage research workflow with MCP tools."""

        def research_planning_node(state):
            """Plan the research approach."""
            # Use MCP prompt templates for research planning
            return state

        def data_collection_node(state):
            """Collect data using MCP research tools."""
            # Use search tools, database queries, etc.
            return state

        def analysis_node(state):
            """Analyze collected data using MCP analysis tools."""
            # Use analysis and processing tools
            return state

        def synthesis_node(state):
            """Synthesize findings using MCP reporting tools."""
            # Use report generation and synthesis tools
            return state

        # Build complex workflow with multiple MCP tool categories
        workflow = StateGraph({"research_query": str, "findings": list, "report": str})

        workflow.add_node("plan", research_planning_node)
        workflow.add_node("collect", data_collection_node)
        workflow.add_node("analyze", analysis_node)
        workflow.add_node("synthesize", synthesis_node)

        # Define the workflow flow
        workflow.set_entry_point("plan")
        workflow.add_edge("plan", "collect")
        workflow.add_edge("collect", "analyze")
        workflow.add_edge("analyze", "synthesize")
        workflow.add_edge("synthesize", END)

        return workflow.compile()
```

---

## LlamaIndex Integration with MCP

LlamaIndex focuses on RAG (Retrieval-Augmented Generation) and knowledge management. MCP enhances LlamaIndex by providing standardized access to diverse data sources and processing tools.

### MCP Tools as LlamaIndex Functions

```python
from llama_index.core.tools import FunctionTool
from llama_index.core.agent import ReActAgent
from llama_index.llms.openai import OpenAI
import asyncio

class MCPLlamaIndexIntegration:
    def __init__(self, mcp_server_script: str):
        self.mcp_server_script = mcp_server_script
        self.mcp_session = None
        self.llm = OpenAI(model="gpt-4o")

    async def initialize_mcp(self):
        """Initialize MCP connection."""
        # Similar to LangGraph example
        server_params = StdioServerParameters(
            command="python",
            args=[self.mcp_server_script]
        )

        self.stdio_client = stdio_client(server_params)
        read, write = await self.stdio_client.__aenter__()
        self.mcp_session = ClientSession(read, write)
        await self.mcp_session.initialize()

    async def create_function_tools(self):
        """Convert MCP tools to LlamaIndex FunctionTools."""
        tools_result = await self.mcp_session.list_tools()
        function_tools = []

        for tool in tools_result.tools:
            # Create a closure to capture the tool name
            def make_tool_function(tool_name):
                async def tool_function(**kwargs):
                    result = await self.mcp_session.call_tool(tool_name, kwargs)
                    return result.content[0].text if result.content else "No response"
                return tool_function

            # Create FunctionTool from MCP tool
            function_tool = FunctionTool.from_defaults(
                fn=make_tool_function(tool.name),
                name=tool.name,
                description=tool.description,
                # Convert MCP schema to LlamaIndex format
                fn_schema=self._convert_mcp_schema(tool.inputSchema)
            )
            function_tools.append(function_tool)

        return function_tools

    def _convert_mcp_schema(self, mcp_schema):
        """Convert MCP input schema to LlamaIndex format."""
        # Implementation would convert between schema formats
        return mcp_schema  # Simplified

    async def create_agent(self):
        """Create a LlamaIndex agent with MCP tools."""
        tools = await self.create_function_tools()

        agent = ReActAgent.from_tools(
            tools,
            llm=self.llm,
            verbose=True,
            system_prompt="""You are a research assistant with access to various tools.
            Use the available tools to gather information and provide comprehensive responses.
            """
        )

        return agent

# Usage example
async def run_llamaindex_mcp_example():
    integration = MCPLlamaIndexIntegration("knowledge_server.py")
    await integration.initialize_mcp()

    agent = await integration.create_agent()

    response = await agent.achat("Find information about climate change impacts and create a summary")
    print("Agent response:", response)

# asyncio.run(run_llamaindex_mcp_example())
```

### Knowledge Graph Integration

```python
from llama_index.core import KnowledgeGraphIndex
from llama_index.core.storage import StorageContext
from llama_index.graph_stores.neo4j import Neo4jGraphStore

class MCPKnowledgeGraph:
    def __init__(self):
        self.graph_store = Neo4jGraphStore(
            username="neo4j",
            password="password",
            url="bolt://localhost:7687"
        )

    async def build_knowledge_graph_with_mcp(self, mcp_data_sources):
        """Build knowledge graph using MCP data sources."""

        # Use MCP resource servers to gather documents
        documents = []
        for source in mcp_data_sources:
            # Query MCP resource endpoints
            resource_data = await self.query_mcp_resource(source)
            documents.extend(self.process_resource_data(resource_data))

        # Build knowledge graph
        storage_context = StorageContext.from_defaults(graph_store=self.graph_store)
        kg_index = KnowledgeGraphIndex.from_documents(
            documents,
            storage_context=storage_context,
            max_triplets_per_chunk=10,
            include_embeddings=True
        )

        return kg_index

    async def query_mcp_resource(self, resource_uri):
        """Query an MCP resource endpoint."""
        # Implementation to query MCP resources
        pass

    def process_resource_data(self, data):
        """Process MCP resource data into LlamaIndex documents."""
        # Convert MCP data to Document objects
        pass
```

---

## CrewAI Integration with MCP

CrewAI focuses on multi-agent collaboration. MCP provides standardized tools that multiple agents can share and coordinate around.

### Multi-Agent MCP Setup

```python
from crewai import Agent, Task, Crew
from crewai_tools import BaseTool
import asyncio

class MCPCrewTool(BaseTool):
    """CrewAI tool that wraps MCP functionality."""

    def __init__(self, mcp_session, tool_name, tool_description):
        self.mcp_session = mcp_session
        self.tool_name = tool_name
        super().__init__(
            name=tool_name,
            description=tool_description
        )

    async def _run(self, **kwargs):
        """Execute the MCP tool."""
        result = await self.mcp_session.call_tool(self.tool_name, kwargs)
        return result.content[0].text if result.content else "No response"

class MCPCrewIntegration:
    def __init__(self, mcp_server_script: str):
        self.mcp_server_script = mcp_server_script
        self.mcp_session = None
        self.tools = []

    async def initialize_mcp(self):
        """Initialize MCP and create tools."""
        # Initialize MCP connection (similar to previous examples)
        server_params = StdioServerParameters(
            command="python",
            args=[self.mcp_server_script]
        )

        self.stdio_client = stdio_client(server_params)
        read, write = await self.stdio_client.__aenter__()
        self.mcp_session = ClientSession(read, write)
        await self.mcp_session.initialize()

        # Create CrewAI tools from MCP tools
        tools_result = await self.mcp_session.list_tools()
        for tool in tools_result.tools:
            crew_tool = MCPCrewTool(
                self.mcp_session,
                tool.name,
                tool.description
            )
            self.tools.append(crew_tool)

    def create_research_crew(self):
        """Create a research crew with MCP tools."""

        # Research Agent
        researcher = Agent(
            role='Senior Research Analyst',
            goal='Conduct thorough research on given topics',
            backstory="""You are an experienced research analyst with access to
            various data sources and research tools. You excel at finding relevant
            information and identifying key insights.""",
            tools=self.tools,
            verbose=True
        )

        # Analysis Agent
        analyst = Agent(
            role='Data Analyst',
            goal='Analyze research findings and extract insights',
            backstory="""You are a skilled data analyst who can process complex
            information and identify patterns, trends, and actionable insights.""",
            tools=self.tools,
            verbose=True
        )

        # Writer Agent
        writer = Agent(
            role='Technical Writer',
            goal='Create comprehensive reports from research and analysis',
            backstory="""You are a technical writer who excels at synthesizing
            complex information into clear, actionable reports.""",
            tools=self.tools,
            verbose=True
        )

        return [researcher, analyst, writer]

    def create_research_tasks(self, topic: str):
        """Create research tasks for the crew."""

        research_task = Task(
            description=f"""Conduct comprehensive research on: {topic}

            Use available research tools to gather information from multiple sources.
            Focus on recent developments, key statistics, and expert opinions.
            Compile your findings into a structured research report.""",
            agent=self.researcher,
            expected_output="Comprehensive research report with sources and key findings"
        )

        analysis_task = Task(
            description=f"""Analyze the research findings for: {topic}

            Review the research report and identify:
            - Key trends and patterns
            - Important statistics and metrics
            - Potential implications and impacts
            - Areas for further investigation

            Use analytical tools as needed for data processing.""",
            agent=self.analyst,
            expected_output="Detailed analysis with insights and recommendations"
        )

        writing_task = Task(
            description=f"""Create a final report on: {topic}

            Synthesize the research and analysis into a comprehensive report including:
            - Executive summary
            - Key findings and insights
            - Data visualizations (if applicable)
            - Recommendations and next steps

            Use reporting tools to format and enhance the final output.""",
            agent=self.writer,
            expected_output="Publication-ready comprehensive report"
        )

        return [research_task, analysis_task, writing_task]

# Usage example
async def run_crewai_mcp_example():
    integration = MCPCrewIntegration("multi_tool_server.py")
    await integration.initialize_mcp()

    agents = integration.create_research_crew()
    tasks = integration.create_research_tasks("Artificial Intelligence in Healthcare")

    crew = Crew(
        agents=agents,
        tasks=tasks,
        verbose=True
    )

    result = crew.kickoff()
    print("Crew result:", result)

# asyncio.run(run_crewai_mcp_example())
```

---

## PydanticAI Integration with MCP

PydanticAI emphasizes type safety and structured outputs. MCP's schema-driven approach aligns well with PydanticAI's philosophy.

### Type-Safe MCP Integration

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent, RunContext
from typing import List, Optional
import asyncio

class ResearchQuery(BaseModel):
    topic: str = Field(description="Research topic")
    sources: List[str] = Field(description="Preferred data sources")
    depth: str = Field(description="Research depth: 'shallow', 'medium', 'deep'")

class ResearchResult(BaseModel):
    findings: List[str] = Field(description="Key research findings")
    sources_used: List[str] = Field(description="Sources consulted")
    confidence_score: float = Field(description="Confidence in findings (0-1)")
    recommendations: List[str] = Field(description="Recommended next steps")

class MCPPydanticAgent:
    def __init__(self, mcp_server_script: str):
        self.mcp_server_script = mcp_server_script
        self.mcp_session = None

    async def initialize_mcp(self):
        """Initialize MCP connection."""
        server_params = StdioServerParameters(
            command="python",
            args=[self.mcp_server_script]
        )

        self.stdio_client = stdio_client(server_params)
        read, write = await self.stdio_client.__aenter__()
        self.mcp_session = ClientSession(read, write)
        await self.mcp_session.initialize()

    async def mcp_search_tool(self, ctx: RunContext[None], query: str) -> str:
        """MCP-powered search tool for PydanticAI."""
        result = await self.mcp_session.call_tool("search_papers", {"query": query})
        return result.content[0].text if result.content else "No results"

    async def mcp_analyze_tool(self, ctx: RunContext[None], data: str) -> str:
        """MCP-powered analysis tool for PydanticAI."""
        result = await self.mcp_session.call_tool("analyze_data", {"data": data})
        return result.content[0].text if result.content else "No analysis"

    def create_research_agent(self) -> Agent[None, ResearchResult]:
        """Create a type-safe research agent with MCP tools."""

        agent = Agent(
            'openai:gpt-4o',
            result_type=ResearchResult,
            system_prompt="""You are a research assistant with access to specialized tools.
            Use the available tools to conduct thorough research and provide structured results.
            Always include confidence scores and source citations.""",
            tools=[self.mcp_search_tool, self.mcp_analyze_tool]
        )

        return agent

# Usage example with type safety
async def run_pydantic_mcp_example():
    integration = MCPPydanticAgent("research_server.py")
    await integration.initialize_mcp()

    agent = integration.create_research_agent()

    # Type-safe input
    query = ResearchQuery(
        topic="Machine Learning in Drug Discovery",
        sources=["pubmed", "arxiv", "clinical_trials"],
        depth="deep"
    )

    # Get structured output
    result = await agent.run(
        f"Research {query.topic} using {', '.join(query.sources)} with {query.depth} analysis"
    )

    # Type-safe result
    research_result: ResearchResult = result.data
    print(f"Found {len(research_result.findings)} findings")
    print(f"Confidence: {research_result.confidence_score}")
    print(f"Sources: {research_result.sources_used}")

# asyncio.run(run_pydantic_mcp_example())
```

---

## Recommendations

Based on our exploration of MCP with different frameworks, here are key recommendations:

### Framework Selection Guide

**Choose LangGraph + MCP when:**

- Building complex, stateful workflows
- Need sophisticated agent reasoning patterns
- Require fine-grained control over execution flow
- Working with multi-step processes

**Choose LlamaIndex + MCP when:**

- Focus is on RAG and knowledge management
- Need powerful document processing capabilities
- Building knowledge graphs or semantic search
- Working with large document collections

**Choose CrewAI + MCP when:**

- Building multi-agent systems
- Need agent specialization and collaboration
- Working on complex projects requiring diverse skills
- Want role-based agent interactions

**Choose PydanticAI + MCP when:**

- Type safety is critical
- Need structured, validated outputs
- Building production systems with strict contracts
- Working in environments requiring schema validation

### Universal Best Practices

1. **Tool Reusability**: Design MCP servers to be framework-agnostic
2. **Error Handling**: Implement robust error handling across framework boundaries
3. **Schema Consistency**: Maintain consistent schemas across different integrations
4. **Performance Monitoring**: Monitor MCP tool performance across all frameworks
5. **Security**: Apply consistent security measures regardless of framework choice

---

## Conclusion

MCP's integration with popular agentic frameworks demonstrates its versatility and value as a universal protocol for AI-tool interaction. By providing standardized interfaces, MCP enables:

- **Code reuse** across different frameworks
- **Consistent tooling** regardless of agent implementation
- **Simplified maintenance** of tool ecosystems
- **Enhanced interoperability** between AI systems

```{admonition} Next Chapter Preview
:class: note
In Part 9, we'll bring everything together by building a comprehensive research assistant using LangGraph and MCP, demonstrating advanced patterns and real-world implementation techniques.

Read Part 9 here: [The Full MCP Blueprint: Part 9 - Building a Full-Fledged Research Assistant with MCP and LangGraph](../part-9/mcp-blueprint-part-9.md)
```

The future of AI development lies in standardized, interoperable systems, and MCP provides the foundation for building such systems today.

Thanks for reading!

---

## Discussion

Any questions? Feel free to post them in the comments or connect via chat for private discussions.

```{admonition} Tags
:class: note
Topics: Agents, MCP, LangGraph, LlamaIndex, CrewAI, PydanticAI, Integration, Frameworks
```
