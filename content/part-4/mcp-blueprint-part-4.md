---
title: "The Full MCP Blueprint: Part 4 - Building a Full-Fledged MCP Workflow using Tools, Resources, and Prompts"
subtitle: "Model context protocol crash course—Part 4"
authors:
  - name: Avi Chawla
  - name: Akshay Pachaar
date: "June 15, 2025"
source: "Daily Dose of Data Science"
---

# The Full MCP Blueprint: Part 4 - Building a Full-Fledged MCP Workflow using Tools, Resources, and Prompts

_Model context protocol crash course—Part 4._

**Authors:** Avi Chawla, Akshay Pachaar  
**Date:** June 15, 2025  
**Source:** Daily Dose of Data Science

---

## Table of Contents

```{contents}
:depth: 3
```

---

## Recap

Before we dive into Part 4 of the MCP crash course, let's briefly recap what we covered in the previous part of this course.

In Part 3, we built a custom MCP client from scratch, integrated an LLM into it (which acted as the brain), and understood the full lifecycle of the model context protocol in action.

We also explored the client-server interaction model through practical implementations, observed how tool discovery and execution are handled dynamically, and contrasted MCP's design with traditional function calling and API-based approaches.

We concluded Part 3 with some "try out yourself" style exercises to reinforce practical learning, while our discussion emphasized how MCP streamlines integration through its decoupled and modular architecture.

The hands-on walkthrough in Part 3 not only demystified MCP as a protocol but also highlighted its core strengths, like scalability, extensibility, and seamless tool orchestration.

By learning how tools are registered, discovered, and executed without tight API coupling, we saw how MCP allows developers to build adaptable and maintainable AI systems with ease.

```{admonition} Prerequisites
:class: note
If you haven't explored Part 3 yet, we highly recommend doing so first since it lays the essential groundwork for what follows in this part. You can read it here: [The Full MCP Blueprint: Part 3 - Building a Custom MCP Client from Scratch](../part-3/mcp-blueprint-part-3.md)
```

---

## Introduction

Until now, our focus has primarily been on tools.

However, tools, prompts, and resources form the three core capabilities of the MCP framework.

While we introduced resources and prompts briefly in Part 2, this part will deep-dive into their mechanics, distinctions, and implementation.

We now shift gears to explore resources and prompts in detail and bring clarity to the key ideas around resources and prompts, like how they differ from tools, how to implement them, and how they enable richer, more contextual interactions when used in coordination.

By the end of this part, you'll have a concrete understanding of:

- How to implement resources in MCP servers and what they enable
- How to create prompt capabilities for template-based reasoning
- When to use tools vs. resources vs. prompts
- How to combine all three capabilities to design sophisticated server logic
- Practical patterns for orchestrating multi-capability workflows

Let's get started!

---

## Resources

Resources in MCP provide read-only data that the AI can query for information. Unlike tools, which perform actions or execute code, resources are purely informational and have no side effects.

```{admonition} Key Insight
:class: tip
Think of resources as your server's knowledge base. They're like having a librarian that the AI can ask for specific documents or information when needed.
```

### What are Resources?

Resources are data sources that can be:

- Static files (documents, configurations, etc.)
- Dynamic data (database queries, API responses)
- Computed information (reports, analytics)
- Any read-only information the AI might need

### Key Characteristics of Resources

1. **Read-only**: Resources don't modify anything; they only provide information
2. **URI-based**: Each resource is identified by a URI pattern
3. **Application-controlled**: The host typically decides when to fetch resources
4. **Context-providing**: They give the AI background information to reason with

### Implementation Example

Let's create a resource server that provides access to documentation and configuration data:

```python
from fastmcp import FastMCP
import json
import os
from typing import Dict, Any

# Create MCP server
mcp = FastMCP("Resource Server")

@mcp.resource("docs://{document_name}")
def get_documentation(document_name: str) -> str:
    """Get documentation content by document name.

    Args:
        document_name: Name of the document to retrieve

    Returns:
        Content of the requested document
    """
    docs = {
        "api_reference": """
        # API Reference

        ## Authentication
        Use Bearer tokens in the Authorization header.

        ## Endpoints
        - GET /users - List all users
        - POST /users - Create a new user
        - GET /users/{id} - Get user by ID
        """,
        "setup_guide": """
        # Setup Guide

        1. Install dependencies: pip install -r requirements.txt
        2. Set environment variables: OPENAI_API_KEY
        3. Run the server: python server.py
        """,
        "troubleshooting": """
        # Troubleshooting

        ## Common Issues
        - Connection timeout: Check network settings
        - Authentication failed: Verify API keys
        - Rate limiting: Implement exponential backoff
        """
    }

    return docs.get(document_name, f"Document '{document_name}' not found")

@mcp.resource("config://{config_type}")
def get_configuration(config_type: str) -> str:
    """Get configuration data by type.

    Args:
        config_type: Type of configuration to retrieve

    Returns:
        Configuration data as JSON string
    """
    configs = {
        "database": {
            "host": "localhost",
            "port": 5432,
            "name": "myapp_db",
            "pool_size": 10
        },
        "cache": {
            "provider": "redis",
            "host": "localhost",
            "port": 6379,
            "ttl": 3600
        },
        "logging": {
            "level": "INFO",
            "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s",
            "handlers": ["console", "file"]
        }
    }

    config = configs.get(config_type, {"error": f"Configuration '{config_type}' not found"})
    return json.dumps(config, indent=2)

@mcp.resource("analytics://{metric_type}")
def get_analytics(metric_type: str) -> str:
    """Get analytics data by metric type.

    Args:
        metric_type: Type of analytics metric to retrieve

    Returns:
        Analytics data as formatted string
    """
    # In a real implementation, this would query an analytics database
    analytics = {
        "user_growth": """
        User Growth Analytics:
        - Total Users: 15,432
        - New Users (Last 30 days): 1,247
        - Growth Rate: +8.7% MoM
        - Churn Rate: 2.3%
        """,
        "performance": """
        Performance Metrics:
        - Average Response Time: 245ms
        - 99th Percentile: 890ms
        - Uptime: 99.97%
        - Error Rate: 0.08%
        """,
        "revenue": """
        Revenue Analytics:
        - Monthly Recurring Revenue: $45,890
        - Average Revenue Per User: $12.50
        - Customer Lifetime Value: $156
        - Revenue Growth: +15.2% MoM
        """
    }

    return analytics.get(metric_type, f"Analytics for '{metric_type}' not available")

if __name__ == "__main__":
    mcp.run()
```

### How Resources Work

1. **Resource Registration**: Resources are registered with URI patterns
2. **Discovery**: Clients can list available resources
3. **Fetching**: Clients request specific resource URIs
4. **Data Delivery**: Server returns the requested data

```{admonition} Resource vs Tool Decision
:class: tip
**Use Resources when you need to:**
- Provide read-only information
- Give context without side effects
- Share configuration or documentation
- Offer data for the AI to reason with

**Use Tools when you need to:**
- Perform actions or execute code
- Modify state or external systems
- Process data or perform calculations
- Interact with APIs that change things
```

---

## Prompts

Prompts in MCP are predefined conversation templates or workflows that can be injected to guide the AI's behavior. They provide structured ways to set up specific reasoning patterns or interaction flows.

### What are Prompts?

Prompts are:

- **Templates**: Predefined conversation starters or frameworks
- **Workflows**: Multi-step interaction patterns
- **Context setters**: Ways to establish specific roles or scenarios
- **Reusable patterns**: Common interaction templates that can be applied to different situations

### Implementation Example

Let's create a prompt server that provides various reasoning templates:

```python
from fastmcp import FastMCP
from typing import List, Dict, Any

# Create MCP server
mcp = FastMCP("Prompt Server")

@mcp.prompt("code_review")
def code_review_prompt() -> List[Dict[str, str]]:
    """Provides a comprehensive code review prompt template.

    Returns:
        List of message objects for code review workflow
    """
    return [
        {
            "role": "system",
            "content": """You are an expert code reviewer with deep knowledge of software engineering best practices.

Your review should cover:
1. **Code Quality**: Readability, maintainability, and organization
2. **Performance**: Efficiency and optimization opportunities
3. **Security**: Potential vulnerabilities and security issues
4. **Testing**: Test coverage and testability
5. **Documentation**: Code comments and documentation quality

Provide specific, actionable feedback with examples where possible."""
        },
        {
            "role": "user",
            "content": "Please review the following code:\n\n[CODE WILL BE INSERTED HERE]"
        }
    ]

@mcp.prompt("system_design")
def system_design_prompt() -> List[Dict[str, str]]:
    """Provides a system design analysis prompt template.

    Returns:
        List of message objects for system design workflow
    """
    return [
        {
            "role": "system",
            "content": """You are a senior software architect specializing in distributed systems design.

When analyzing system requirements, consider:
1. **Scalability**: How will the system handle growth?
2. **Reliability**: What are the failure modes and recovery strategies?
3. **Performance**: What are the latency and throughput requirements?
4. **Security**: How will the system protect against threats?
5. **Maintainability**: How easy will it be to modify and extend?
6. **Cost**: What are the infrastructure and operational costs?

Provide a structured analysis with diagrams where helpful."""
        },
        {
            "role": "user",
            "content": "Please design a system based on these requirements:\n\n[REQUIREMENTS WILL BE INSERTED HERE]"
        }
    ]

@mcp.prompt("debug_assistant")
def debug_assistant_prompt() -> List[Dict[str, str]]:
    """Provides a debugging assistant prompt template.

    Returns:
        List of message objects for debugging workflow
    """
    return [
        {
            "role": "system",
            "content": """You are an expert debugging assistant. Your goal is to help identify and resolve software issues systematically.

Follow this debugging methodology:
1. **Understand the Problem**: What is the expected vs. actual behavior?
2. **Gather Information**: What logs, error messages, or symptoms are available?
3. **Form Hypotheses**: What could be causing this issue?
4. **Test Systematically**: How can we verify each hypothesis?
5. **Isolate the Cause**: What specific component or code is responsible?
6. **Propose Solutions**: What are the possible fixes and their tradeoffs?

Ask clarifying questions if you need more information."""
        },
        {
            "role": "user",
            "content": "I'm experiencing this issue:\n\n[ISSUE DESCRIPTION WILL BE INSERTED HERE]"
        }
    ]

@mcp.prompt("technical_writer")
def technical_writer_prompt() -> List[Dict[str, str]]:
    """Provides a technical writing prompt template.

    Returns:
        List of message objects for technical writing workflow
    """
    return [
        {
            "role": "system",
            "content": """You are a professional technical writer who creates clear, comprehensive documentation.

Your writing should be:
1. **Clear**: Use simple language and avoid jargon when possible
2. **Structured**: Organize information logically with headings and sections
3. **Complete**: Cover all necessary information for the target audience
4. **Actionable**: Include specific steps, examples, and code samples
5. **Accessible**: Consider different skill levels and backgrounds

Include relevant diagrams, code examples, and troubleshooting sections."""
        },
        {
            "role": "user",
            "content": "Please create documentation for:\n\n[TOPIC WILL BE INSERTED HERE]"
        }
    ]

@mcp.prompt("data_analyst")
def data_analyst_prompt() -> List[Dict[str, str]]:
    """Provides a data analysis prompt template.

    Returns:
        List of message objects for data analysis workflow
    """
    return [
        {
            "role": "system",
            "content": """You are a senior data analyst with expertise in statistical analysis and data visualization.

Your analysis should include:
1. **Data Overview**: Summary statistics and data quality assessment
2. **Exploratory Analysis**: Key patterns, trends, and relationships
3. **Statistical Testing**: Appropriate tests for hypotheses
4. **Visualization**: Clear charts and graphs to illustrate findings
5. **Insights**: Actionable conclusions and recommendations
6. **Limitations**: Potential biases or limitations in the analysis

Use Python/pandas code examples where relevant."""
        },
        {
            "role": "user",
            "content": "Please analyze this dataset:\n\n[DATA DESCRIPTION WILL BE INSERTED HERE]"
        }
    ]

if __name__ == "__main__":
    mcp.run()
```

### How Prompts Work

1. **Prompt Registration**: Prompts are registered with unique names
2. **Discovery**: Clients can list available prompts
3. **Retrieval**: Clients request specific prompts by name
4. **Injection**: The prompt messages are injected into the conversation
5. **Customization**: Template placeholders can be filled with specific content

```{admonition} Prompt Benefits
:class: tip
**Prompts enable:**
- Consistent reasoning patterns across interactions
- Reusable expertise captured in templates
- Structured workflows for complex tasks
- Role-based AI behavior (code reviewer, analyst, etc.)
- Quality control through proven conversation patterns
```

---

## Key Distinctions Between Tools, Resources, and Prompts

Understanding when to use each capability is crucial for designing effective MCP systems.

```{mermaid}
graph TD
    A["MCP Capabilities"] --> B["Tools"]
    A --> C["Resources"]
    A --> D["Prompts"]

    B --> B1["Execute Actions"]
    B --> B2["Modify State"]
    B --> B3["Side Effects"]
    B --> B4["Model-Controlled"]

    C --> C1["Provide Data"]
    C --> C2["Read-Only"]
    C --> C3["No Side Effects"]
    C --> C4["App-Controlled"]

    D --> D1["Set Context"]
    D --> D2["Guide Behavior"]
    D --> D3["Templates"]
    D --> D4["User-Controlled"]

    style B fill:#ff9999
    style C fill:#99ccff
    style D fill:#99ff99
```

### Control Patterns

| Capability    | Who Controls     | When Used                          | Purpose                            |
| ------------- | ---------------- | ---------------------------------- | ---------------------------------- |
| **Tools**     | Model/AI         | AI decides it needs to take action | Execute functions, make changes    |
| **Resources** | Application/Host | App provides context as needed     | Give AI information to reason with |
| **Prompts**   | User/Developer   | User selects interaction mode      | Set up specific reasoning patterns |

### Decision Framework

**Use Tools when:**

- The AI needs to take action
- External systems need to be modified
- Computations need to be performed
- Side effects are acceptable/desired

**Use Resources when:**

- The AI needs background information
- Context is required for reasoning
- Data needs to be retrieved
- No modifications should occur

**Use Prompts when:**

- You want specific interaction patterns
- Role-based behavior is needed
- Reusable workflows are beneficial
- Consistent reasoning is important

---

## A Practical Demonstration

Let's build a comprehensive example that combines all three capabilities in a content management system:

```python
from fastmcp import FastMCP
import json
import datetime
from typing import List, Dict, Any

# Create MCP server that combines tools, resources, and prompts
mcp = FastMCP("Content Management Server")

# Sample data store (in production, this would be a database)
content_store = {
    "articles": [
        {
            "id": 1,
            "title": "Introduction to MCP",
            "content": "MCP is a protocol for AI-tool integration...",
            "author": "Alice",
            "created_at": "2025-01-01T10:00:00Z",
            "status": "published"
        },
        {
            "id": 2,
            "title": "Advanced MCP Patterns",
            "content": "This article explores advanced patterns...",
            "author": "Bob",
            "created_at": "2025-01-02T14:30:00Z",
            "status": "draft"
        }
    ],
    "templates": {
        "blog_post": "# {title}\n\nBy {author}\n\n{content}",
        "tutorial": "# {title}\n\n## Overview\n{overview}\n\n## Steps\n{steps}"
    }
}

# TOOLS - For actions and modifications
@mcp.tool()
def create_article(title: str, content: str, author: str) -> str:
    """Create a new article.

    Args:
        title: Article title
        content: Article content
        author: Author name

    Returns:
        Success message with article ID
    """
    new_id = max([a["id"] for a in content_store["articles"]]) + 1
    new_article = {
        "id": new_id,
        "title": title,
        "content": content,
        "author": author,
        "created_at": datetime.datetime.now().isoformat() + "Z",
        "status": "draft"
    }
    content_store["articles"].append(new_article)
    return f"Article created successfully with ID: {new_id}"

@mcp.tool()
def update_article_status(article_id: int, status: str) -> str:
    """Update the status of an article.

    Args:
        article_id: ID of the article to update
        status: New status (draft, published, archived)

    Returns:
        Success or error message
    """
    for article in content_store["articles"]:
        if article["id"] == article_id:
            article["status"] = status
            return f"Article {article_id} status updated to: {status}"
    return f"Article {article_id} not found"

@mcp.tool()
def delete_article(article_id: int) -> str:
    """Delete an article.

    Args:
        article_id: ID of the article to delete

    Returns:
        Success or error message
    """
    for i, article in enumerate(content_store["articles"]):
        if article["id"] == article_id:
            deleted = content_store["articles"].pop(i)
            return f"Article '{deleted['title']}' deleted successfully"
    return f"Article {article_id} not found"

# RESOURCES - For read-only data access
@mcp.resource("articles://{status}")
def get_articles_by_status(status: str) -> str:
    """Get articles filtered by status.

    Args:
        status: Status to filter by (all, draft, published, archived)

    Returns:
        JSON string of filtered articles
    """
    if status == "all":
        filtered_articles = content_store["articles"]
    else:
        filtered_articles = [a for a in content_store["articles"] if a["status"] == status]

    return json.dumps(filtered_articles, indent=2)

@mcp.resource("article://{article_id}")
def get_article_by_id(article_id: str) -> str:
    """Get a specific article by ID.

    Args:
        article_id: ID of the article to retrieve

    Returns:
        JSON string of the article or error message
    """
    try:
        aid = int(article_id)
        for article in content_store["articles"]:
            if article["id"] == aid:
                return json.dumps(article, indent=2)
        return f"Article {article_id} not found"
    except ValueError:
        return f"Invalid article ID: {article_id}"

@mcp.resource("templates://{template_type}")
def get_content_template(template_type: str) -> str:
    """Get content templates for different article types.

    Args:
        template_type: Type of template (blog_post, tutorial, etc.)

    Returns:
        Template string or error message
    """
    template = content_store["templates"].get(template_type)
    if template:
        return template
    else:
        available = ", ".join(content_store["templates"].keys())
        return f"Template '{template_type}' not found. Available: {available}"

@mcp.resource("analytics://content")
def get_content_analytics() -> str:
    """Get content analytics and statistics.

    Returns:
        Analytics data as formatted string
    """
    articles = content_store["articles"]
    total = len(articles)
    by_status = {}
    by_author = {}

    for article in articles:
        status = article["status"]
        author = article["author"]

        by_status[status] = by_status.get(status, 0) + 1
        by_author[author] = by_author.get(author, 0) + 1

    analytics = {
        "total_articles": total,
        "by_status": by_status,
        "by_author": by_author,
        "most_productive_author": max(by_author.items(), key=lambda x: x[1])[0] if by_author else "None"
    }

    return json.dumps(analytics, indent=2)

# PROMPTS - For guided interactions
@mcp.prompt("content_creator")
def content_creator_prompt() -> List[Dict[str, str]]:
    """Provides a content creation prompt template.

    Returns:
        List of message objects for content creation workflow
    """
    return [
        {
            "role": "system",
            "content": """You are a professional content creator and editor. Your role is to help create high-quality, engaging content.

When creating content, consider:
1. **Audience**: Who will be reading this content?
2. **Purpose**: What should readers learn or do after reading?
3. **Structure**: How should the content be organized for clarity?
4. **Tone**: What voice and style are appropriate?
5. **SEO**: How can we optimize for searchability?
6. **Engagement**: How can we make this content compelling?

Always suggest improvements for readability, accuracy, and impact."""
        },
        {
            "role": "user",
            "content": "I need help creating content. Here are my requirements:\n\n[CONTENT REQUIREMENTS WILL BE INSERTED HERE]"
        }
    ]

@mcp.prompt("content_editor")
def content_editor_prompt() -> List[Dict[str, str]]:
    """Provides a content editing prompt template.

    Returns:
        List of message objects for content editing workflow
    """
    return [
        {
            "role": "system",
            "content": """You are an experienced content editor focused on improving clarity, flow, and impact.

Your editing process should address:
1. **Clarity**: Is the message clear and easy to understand?
2. **Structure**: Does the content flow logically?
3. **Conciseness**: Can we say the same thing more efficiently?
4. **Accuracy**: Are all facts and claims correct?
5. **Consistency**: Is terminology and style consistent throughout?
6. **Grammar**: Are there any grammatical or spelling errors?

Provide specific suggestions with explanations for your changes."""
        },
        {
            "role": "user",
            "content": "Please edit this content:\n\n[CONTENT TO EDIT WILL BE INSERTED HERE]"
        }
    ]

@mcp.prompt("content_strategist")
def content_strategist_prompt() -> List[Dict[str, str]]:
    """Provides a content strategy prompt template.

    Returns:
        List of message objects for content strategy workflow
    """
    return [
        {
            "role": "system",
            "content": """You are a content strategist who helps plan and optimize content for maximum impact.

Your strategic analysis should cover:
1. **Goals**: What are the business/educational objectives?
2. **Audience**: Who is the target audience and what do they need?
3. **Competition**: How does this content differentiate from existing content?
4. **Distribution**: What channels will be used to reach the audience?
5. **Metrics**: How will success be measured?
6. **Timeline**: What's the publication and promotion schedule?

Provide actionable recommendations for content planning and optimization."""
        },
        {
            "role": "user",
            "content": "I need a content strategy for:\n\n[CONTENT STRATEGY REQUIREMENTS WILL BE INSERTED HERE]"
        }
    ]

if __name__ == "__main__":
    mcp.run()
```

### Usage Scenarios

This comprehensive server enables rich workflows:

**Scenario 1: Content Creation Workflow**

1. **Prompt**: Use "content_creator" prompt to get AI guidance on content strategy
2. **Resource**: Check "templates://blog_post" to get the template format
3. **Tool**: Use "create_article" to create the new content
4. **Resource**: Check "analytics://content" to see impact on content metrics

**Scenario 2: Content Management Workflow**

1. **Resource**: Query "articles://draft" to see all draft articles
2. **Resource**: Get specific article with "article://1"
3. **Prompt**: Use "content_editor" prompt to get editing suggestions
4. **Tool**: Use "update_article_status" to publish when ready

**Scenario 3: Content Analysis Workflow**

1. **Resource**: Get "analytics://content" for overview
2. **Resource**: Query "articles://published" for published content
3. **Prompt**: Use "content_strategist" prompt for planning next content
4. **Tool**: Create new articles based on strategy

```{admonition} Orchestration Benefits
:class: tip
By combining all three capabilities, you create a rich, flexible system where:
- **Tools** handle all the actions (CRUD operations)
- **Resources** provide context and data for decision-making
- **Prompts** guide the AI toward specific expert behaviors

This creates a much more powerful and nuanced interaction than using any single capability alone.
```

---

## Conclusion and Next Steps

In this part, we've expanded our MCP toolkit beyond tools to include resources and prompts, creating a complete capability framework for building sophisticated AI systems.

**Key Takeaways:**

**Resources provide context** - They give the AI access to information without side effects, enabling more informed decision-making.

**Prompts provide guidance** - They set up specific reasoning patterns and expert behaviors that improve AI performance.

**Combined capabilities are powerful** - Using tools, resources, and prompts together creates rich, flexible workflows that can handle complex real-world scenarios.

**Control patterns matter** - Understanding who controls what (model, application, or user) helps you design better interaction patterns.

With this foundation, you now have all the building blocks needed to create comprehensive MCP servers that can:

- Execute actions through tools
- Provide context through resources
- Guide behavior through prompts
- Orchestrate complex multi-step workflows

```{admonition} Next Chapter Preview
:class: note
In the next part, we'll explore sampling - the fourth MCP capability that allows servers to request completions from the client's language model, enabling truly collaborative AI workflows.

Read Part 5 here: [The Full MCP Blueprint: Part 5 - Integrating Sampling into MCP Workflows](../part-5/mcp-blueprint-part-5.md)
```

The combination of tools, resources, and prompts gives you everything needed to build production-ready MCP systems that provide real value to users. These capabilities work together to create AI systems that are not just functional, but intelligent, context-aware, and expertly guided.

Thanks for reading!

---

## Discussion

Any questions? Feel free to post them in the comments or connect via chat for private discussions.

```{admonition} Tags
:class: note
Topics: Agents, MCP, Resources, Prompts, Tools, Workflows
```
