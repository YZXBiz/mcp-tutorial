---
title: "The Full MCP Blueprint: Part 5 - Integrating Sampling into MCP Workflows"
subtitle: "Model context protocol crash course—Part 5"
authors:
  - name: Avi Chawla
  - name: Akshay Pachaar
date: "June 22, 2025"
source: "Daily Dose of Data Science"
---

# The Full MCP Blueprint: Part 5 - Integrating Sampling into MCP Workflows

_Model context protocol crash course—Part 5._

**Authors:** Avi Chawla, Akshay Pachaar  
**Date:** June 22, 2025  
**Source:** Daily Dose of Data Science

---

## Table of Contents

```{contents}
:depth: 3
```

---

## Recap

In Part 4 of this MCP crash course, we expanded our focus beyond tools and took a deep dive into the other two foundational pillars of MCP: resources and prompts.

We learned how resources allow servers to expose data, anything from static documents to dynamic query-backed endpoints.

Prompts, on the other hand, enable natural language-based behaviors that can dynamically shape responses based on input, roles, and instructions.

Through theory and hands-on walkthroughs, we explored how each capability differs in execution flow, when to use tools vs. resources vs. prompts, and how they can be combined to design sophisticated server logic.

We also discussed how Claude Desktop can leverage these primitives to orchestrate rich user interactions powered by model-controlled, application-controlled, and user-initiated patterns.

Overall, in Part 4, we learned everything that it takes to build full-fledged MCP servers that don't just execute code, but also expose data and template-based reasoning, allowing for cleaner separation of logic and language.

```{admonition} Prerequisites
:class: note
If you haven't explored Part 4 yet, we strongly recommend going through it first since it lays the conceptual scaffolding that'll help you better understand what we're about to dive into here.

You can read it here: [The Full MCP Blueprint: Part 4 - Building a Full-Fledged MCP Workflow using Tools, Resources, and Prompts](../part-4/mcp-blueprint-part-4.md)
```

---

## In This Part

In this chapter, we'll explore the fourth and most advanced MCP capability: **Sampling**.

Sampling is unique because it reverses the typical flow of control. Instead of the client requesting things from the server, sampling allows the **server to request completions from the client's language model**.

This creates a collaborative intelligence pattern where:

- The client provides the LLM "brain"
- The server provides the tools and logic
- Both work together to solve complex problems

We'll cover:

- **Concept and motivation** behind sampling
- **Architecture** of sampling workflows
- **Implementation** on both server and client sides
- **Practical use cases** and advanced patterns
- **Best practices** for error handling and reliability

---

## Introduction

Until now, we've seen a fairly straightforward interaction model in MCP:

1. Client connects to server
2. Client discovers server capabilities (tools, resources, prompts)
3. Client (via LLM) decides to use those capabilities
4. Server responds with results

But what if the server itself needs to perform reasoning? What if a tool needs to analyze text, generate content, or make intelligent decisions?

This is where **sampling** comes in.

```{admonition} Key Insight
:class: tip
Sampling allows MCP servers to "borrow" the client's LLM to perform reasoning tasks, creating a collaborative intelligence system where both client and server contribute their strengths.
```

---

## Concept and Motivation

### The Problem Sampling Solves

Consider these scenarios:

1. **A data analysis tool** that needs to interpret query results and generate insights
2. **A content management system** that needs to summarize documents or extract key points
3. **A code review tool** that needs to analyze code quality and suggest improvements
4. **A customer service bot** that needs to understand customer intent and generate appropriate responses

In each case, the server has access to data and domain logic, but needs language understanding and generation capabilities to be truly useful.

### Traditional Approaches and Their Limitations

**Option 1: Server runs its own LLM**

- ❌ Expensive (requires GPU resources)
- ❌ Redundant (client already has LLM access)
- ❌ Inconsistent (different models may give different results)

**Option 2: Server returns raw data, client does all reasoning**

- ❌ Limited server intelligence
- ❌ Complex client-side logic
- ❌ Tight coupling between client and server logic

**Option 3: MCP Sampling**

- ✅ Server leverages client's existing LLM
- ✅ Collaborative intelligence
- ✅ Consistent reasoning context
- ✅ Server can focus on domain logic

### How Sampling Works

```{mermaid}
sequenceDiagram
    participant Client
    participant Server
    participant LLM

    Client->>Server: Call tool/resource
    Server->>Server: Process request
    Server->>Client: Request LLM sampling
    Client->>LLM: Forward sampling request
    LLM-->>Client: Return completion
    Client-->>Server: Return completion result
    Server->>Server: Use LLM result in logic
    Server-->>Client: Return final result
```

The server can request the client's LLM to:

- Analyze text and extract information
- Generate content based on templates
- Make decisions based on data
- Summarize complex information
- Transform data between formats

---

## Architecture

Sampling introduces a new layer to the MCP architecture:

```{mermaid}
graph TB
    subgraph "Client Side"
        A["Host Application"]
        B["MCP Client"]
        C["LLM"]
        A --> B
        B <--> C
    end

    subgraph "Server Side"
        D["MCP Server"]
        E["Business Logic"]
        F["Sampling Requests"]
        D --> E
        E --> F
    end

    B <--> D
    F -.-> B
    B -.-> C
    C -.-> B
    B -.-> F

    style F fill:#ff9999
    style C fill:#99ff99
```

### Context Object and Sampling Mechanism

Sampling requests include a **context object** that specifies:

- **Prompt**: The instruction for the LLM
- **Max tokens**: Limit on response length
- **Temperature**: Creativity/randomness setting
- **Stop sequences**: When to stop generation
- **Model preferences**: Specific model requirements

### Server-Side Implementation

Here's how to implement sampling in a FastMCP server:

```python
from fastmcp import FastMCP
import json
from typing import Dict, Any, List

# Create MCP server with sampling capabilities
mcp = FastMCP("Sampling Demo Server")

@mcp.tool()
async def analyze_customer_feedback(feedback: str) -> str:
    """Analyze customer feedback and extract key insights.

    Args:
        feedback: Raw customer feedback text

    Returns:
        Structured analysis of the feedback
    """
    # Use sampling to analyze the feedback
    analysis_prompt = f"""
    Please analyze the following customer feedback and provide a structured response:

    FEEDBACK:
    {feedback}

    Please provide:
    1. Overall sentiment (positive, negative, neutral)
    2. Key issues mentioned
    3. Specific complaints or praise
    4. Priority level (high, medium, low)
    5. Suggested actions

    Format your response as JSON with these keys: sentiment, issues, specifics, priority, actions
    """

    # Request LLM analysis via sampling
    result = await mcp.request_sampling(
        prompt=analysis_prompt,
        max_tokens=500,
        temperature=0.1  # Low temperature for consistent analysis
    )

    try:
        # Parse the LLM response
        analysis = json.loads(result)

        # Add our own processing/validation
        analysis["processed_at"] = "2025-01-01T12:00:00Z"
        analysis["confidence"] = "high" if len(feedback) > 100 else "medium"

        return json.dumps(analysis, indent=2)
    except json.JSONDecodeError:
        return f"Analysis: {result}\n\nNote: Response was not in JSON format"

@mcp.tool()
async def generate_email_response(customer_issue: str, tone: str = "professional") -> str:
    """Generate a customer service email response.

    Args:
        customer_issue: Description of the customer's issue
        tone: Tone for the response (professional, friendly, formal)

    Returns:
        Generated email response
    """
    tone_instructions = {
        "professional": "Use a professional, business-appropriate tone",
        "friendly": "Use a warm, friendly tone while remaining professional",
        "formal": "Use a formal, respectful tone suitable for serious issues"
    }

    email_prompt = f"""
    Generate a customer service email response for the following issue:

    CUSTOMER ISSUE:
    {customer_issue}

    TONE: {tone_instructions.get(tone, tone)}

    Requirements:
    - Acknowledge the customer's concern
    - Provide a clear solution or next steps
    - Include appropriate apology if needed
    - End with contact information offer
    - Keep it concise but complete

    Generate only the email body, no subject line.
    """

    # Request email generation via sampling
    email_response = await mcp.request_sampling(
        prompt=email_prompt,
        max_tokens=300,
        temperature=0.3  # Some creativity for natural language
    )

    # Add standard footer
    footer = "\n\nBest regards,\nCustomer Service Team\nEmail: support@company.com\nPhone: 1-800-SUPPORT"

    return email_response + footer

@mcp.tool()
async def summarize_document(document_text: str, summary_type: str = "brief") -> str:
    """Summarize a document using LLM sampling.

    Args:
        document_text: The full text of the document to summarize
        summary_type: Type of summary (brief, detailed, bullet_points)

    Returns:
        Generated summary
    """
    summary_instructions = {
        "brief": "Create a 2-3 sentence summary capturing the main points",
        "detailed": "Create a comprehensive paragraph summary with key details",
        "bullet_points": "Create a bulleted list of the main points and conclusions"
    }

    summary_prompt = f"""
    Please summarize the following document:

    DOCUMENT:
    {document_text}

    SUMMARY TYPE: {summary_instructions.get(summary_type, summary_type)}

    Focus on:
    - Main arguments or points
    - Key conclusions
    - Important data or statistics
    - Actionable insights
    """

    # Request summarization via sampling
    summary = await mcp.request_sampling(
        prompt=summary_prompt,
        max_tokens=400,
        temperature=0.2  # Consistent but slightly creative
    )

    # Add metadata
    word_count = len(document_text.split())
    metadata = f"\n\n---\nSummary generated from {word_count} word document\nSummary type: {summary_type}"

    return summary + metadata

@mcp.resource("sample_data://customer_feedback")
async def get_sample_feedback() -> str:
    """Get sample customer feedback for testing.

    Returns:
        Sample feedback data
    """
    sample_feedback = [
        {
            "id": 1,
            "feedback": "I love the new features in your app! The interface is so much cleaner and easier to use. However, I noticed that the search function is quite slow and sometimes doesn't return relevant results. Overall though, great job on the update!",
            "date": "2025-01-01"
        },
        {
            "id": 2,
            "feedback": "I'm really frustrated with the recent changes. The app keeps crashing when I try to upload photos, and customer service hasn't responded to my emails from last week. This is affecting my business and I'm considering switching to a competitor.",
            "date": "2025-01-02"
        },
        {
            "id": 3,
            "feedback": "Good app overall but the subscription pricing seems high compared to alternatives. Would love to see more flexible plans for small businesses like mine. The core functionality works well though.",
            "date": "2025-01-03"
        }
    ]

    return json.dumps(sample_feedback, indent=2)

if __name__ == "__main__":
    mcp.run()
```

### Client-Side Implementation

On the client side, you need to handle sampling requests. Here's how to extend our previous client to support sampling:

```python
import asyncio
import os
from dotenv import load_dotenv
import litellm
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

# Load environment variables
load_dotenv()

# Global session management
session = None

async def handle_sampling_request(request):
    """Handle LLM sampling requests from the server.

    Args:
        request: Sampling request with prompt and parameters

    Returns:
        LLM completion result
    """
    try:
        # Extract sampling parameters
        prompt = request.get("prompt", "")
        max_tokens = request.get("max_tokens", 150)
        temperature = request.get("temperature", 0.7)

        # Make LLM call
        response = await litellm.acompletion(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=max_tokens,
            temperature=temperature
        )

        return response.choices[0].message.content

    except Exception as e:
        return f"Sampling error: {str(e)}"

async def connect_to_server(server_script: str):
    """Connect to MCP server with sampling support."""
    global session

    server_params = StdioServerParameters(
        command="python", args=[server_script]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as client_session:
            session = client_session

            # Register sampling handler
            session.set_sampling_handler(handle_sampling_request)

            await session.initialize()

            # Run interactive demo
            await interactive_demo()

async def interactive_demo():
    """Interactive demo of sampling capabilities."""
    print("🤖 MCP Sampling Demo")
    print("Available commands:")
    print("1. analyze - Analyze customer feedback")
    print("2. email - Generate email response")
    print("3. summarize - Summarize document")
    print("4. quit - Exit demo")

    while True:
        command = input("\nEnter command: ").strip().lower()

        if command == "quit":
            break
        elif command == "analyze":
            await demo_analyze_feedback()
        elif command == "email":
            await demo_generate_email()
        elif command == "summarize":
            await demo_summarize_document()
        else:
            print("Unknown command. Try 'analyze', 'email', 'summarize', or 'quit'")

async def demo_analyze_feedback():
    """Demo customer feedback analysis."""
    print("\n📊 Customer Feedback Analysis Demo")

    # Get sample feedback
    feedback_resource = await session.read_resource("sample_data://customer_feedback")
    feedback_data = json.loads(feedback_resource.contents[0].text)

    print("Available feedback samples:")
    for item in feedback_data:
        print(f"{item['id']}: {item['feedback'][:100]}...")

    choice = input("Select feedback ID (1-3): ")
    try:
        selected = next(item for item in feedback_data if item["id"] == int(choice))

        print(f"\nAnalyzing: {selected['feedback']}")
        print("🔄 Processing with LLM...")

        # Call analysis tool (which will use sampling)
        result = await session.call_tool("analyze_customer_feedback", {
            "feedback": selected["feedback"]
        })

        print("✅ Analysis complete:")
        print(result.content[0].text)

    except (ValueError, StopIteration):
        print("Invalid selection")

async def demo_generate_email():
    """Demo email response generation."""
    print("\n📧 Email Response Generator Demo")

    issue = input("Describe customer issue: ")
    tone = input("Email tone (professional/friendly/formal): ") or "professional"

    print("🔄 Generating email response...")

    result = await session.call_tool("generate_email_response", {
        "customer_issue": issue,
        "tone": tone
    })

    print("✅ Email generated:")
    print(result.content[0].text)

async def demo_summarize_document():
    """Demo document summarization."""
    print("\n📄 Document Summarization Demo")

    # Sample document
    document = """
    The quarterly sales report shows strong performance across all regions.
    North America achieved 15% growth compared to Q3, with particularly strong
    performance in the technology and healthcare sectors. Revenue increased from
    $2.1M to $2.4M, exceeding our target by 8%.

    Europe showed steady growth of 7%, though this was below our 10% target.
    The main challenges were related to supply chain disruptions and increased
    competition in the automotive sector. However, the new product line
    introduced in October performed better than expected.

    Asia-Pacific region delivered exceptional results with 22% growth, driven
    by expansion into three new markets and successful partnerships with local
    distributors. This region is now our fastest-growing market.

    Looking ahead to Q1, we're optimistic about continued growth, with new
    product launches planned and expanded marketing efforts. Key priorities
    include addressing European supply chain issues and scaling our Asia-Pacific
    operations.
    """

    summary_type = input("Summary type (brief/detailed/bullet_points): ") or "brief"

    print("🔄 Summarizing document...")

    result = await session.call_tool("summarize_document", {
        "document_text": document,
        "summary_type": summary_type
    })

    print("✅ Summary complete:")
    print(result.content[0].text)

async def main():
    """Main function."""
    await connect_to_server("sampling_server.py")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Model Preferences

Sampling requests can specify model preferences to ensure the right LLM is used for each task:

```python
# High-precision analysis
result = await mcp.request_sampling(
    prompt=analysis_prompt,
    model_preferences={
        "hints": [
            {"name": "gpt-4", "priority": 1.0},
            {"name": "claude-3", "priority": 0.8}
        ]
    },
    max_tokens=500,
    temperature=0.1
)

# Creative content generation
result = await mcp.request_sampling(
    prompt=creative_prompt,
    model_preferences={
        "hints": [
            {"name": "gpt-4", "priority": 1.0},
            {"name": "claude-3", "priority": 0.9}
        ]
    },
    max_tokens=800,
    temperature=0.8
)
```

---

## Sampling Use Cases

### 1. Intelligent Data Analysis

```python
@mcp.tool()
async def analyze_sales_data(data: str) -> str:
    """Analyze sales data and provide insights."""
    prompt = f"""
    Analyze this sales data and provide key insights:

    {data}

    Focus on:
    - Trends and patterns
    - Anomalies or outliers
    - Recommendations for improvement
    - Key metrics summary
    """

    return await mcp.request_sampling(prompt=prompt, temperature=0.2)
```

### 2. Natural Language to SQL

```python
@mcp.tool()
async def natural_language_to_sql(query: str, schema: str) -> str:
    """Convert natural language to SQL query."""
    prompt = f"""
    Convert this natural language query to SQL:

    Query: {query}

    Database schema:
    {schema}

    Return only the SQL query, no explanation.
    """

    return await mcp.request_sampling(prompt=prompt, temperature=0.0)
```

### 3. Content Transformation

```python
@mcp.tool()
async def transform_content(content: str, target_format: str) -> str:
    """Transform content between different formats."""
    prompt = f"""
    Transform this content to {target_format} format:

    {content}

    Maintain the key information while adapting to the target format.
    """

    return await mcp.request_sampling(prompt=prompt, temperature=0.3)
```

---

## Advanced Use Cases and Patterns

### Multi-Step Reasoning

```python
@mcp.tool()
async def complex_analysis(data: str) -> str:
    """Perform multi-step analysis with LLM reasoning."""

    # Step 1: Initial analysis
    initial_prompt = f"Analyze this data and identify key patterns: {data}"
    initial_analysis = await mcp.request_sampling(prompt=initial_prompt)

    # Step 2: Deep dive based on initial findings
    deep_dive_prompt = f"""
    Based on this initial analysis: {initial_analysis}

    Now perform a deeper analysis focusing on the most significant patterns.
    Provide specific recommendations.
    """
    deep_analysis = await mcp.request_sampling(prompt=deep_dive_prompt)

    # Step 3: Generate action plan
    action_prompt = f"""
    Based on these analyses:
    Initial: {initial_analysis}
    Deep: {deep_analysis}

    Create a prioritized action plan with specific steps.
    """
    action_plan = await mcp.request_sampling(prompt=action_prompt)

    return f"Initial Analysis:\n{initial_analysis}\n\nDeep Analysis:\n{deep_analysis}\n\nAction Plan:\n{action_plan}"
```

### Collaborative Workflows

```python
@mcp.tool()
async def collaborative_document_review(document: str, review_type: str) -> str:
    """Collaborative document review with multiple LLM perspectives."""

    perspectives = {
        "technical": "Review from a technical accuracy perspective",
        "editorial": "Review for clarity, flow, and readability",
        "business": "Review for business value and strategic alignment"
    }

    reviews = []
    for perspective, instruction in perspectives.items():
        if review_type == "all" or review_type == perspective:
            prompt = f"{instruction}\n\nDocument:\n{document}"
            review = await mcp.request_sampling(prompt=prompt)
            reviews.append(f"{perspective.title()} Review:\n{review}")

    # Synthesize reviews
    synthesis_prompt = f"""
    Synthesize these reviews into a comprehensive assessment:

    {chr(10).join(reviews)}

    Provide overall recommendations and priority issues.
    """

    synthesis = await mcp.request_sampling(prompt=synthesis_prompt)

    return f"{chr(10).join(reviews)}\n\nSynthesis:\n{synthesis}"
```

---

## Error Handling and Best Practices

### Robust Error Handling

```python
@mcp.tool()
async def robust_analysis(data: str) -> str:
    """Analysis with comprehensive error handling."""

    try:
        # Primary analysis attempt
        result = await mcp.request_sampling(
            prompt=f"Analyze: {data}",
            max_tokens=500,
            temperature=0.2
        )

        # Validate result
        if len(result.strip()) < 50:
            raise ValueError("Response too short")

        return result

    except Exception as e:
        # Fallback to simpler analysis
        try:
            fallback_result = await mcp.request_sampling(
                prompt=f"Provide a brief summary of: {data}",
                max_tokens=200,
                temperature=0.1
            )
            return f"Fallback analysis: {fallback_result}"

        except Exception as fallback_error:
            # Final fallback to rule-based analysis
            return f"Basic analysis: Data contains {len(data)} characters. Analysis failed due to: {str(e)}"
```

### Best Practices

1. **Set appropriate timeouts**:

```python
result = await mcp.request_sampling(
    prompt=prompt,
    timeout=30  # 30 second timeout
)
```

2. **Use specific prompts**:

```python
# Good: Specific and structured
prompt = """
Analyze this customer review and return JSON with:
- sentiment: positive/negative/neutral
- key_issues: list of specific issues
- priority: high/medium/low

Review: {review}
"""

# Bad: Vague and unstructured
prompt = f"What do you think about this review: {review}"
```

3. **Implement fallback chains**:

```python
models_to_try = ["gpt-4", "gpt-3.5-turbo", "claude-3"]
for model in models_to_try:
    try:
        result = await mcp.request_sampling(
            prompt=prompt,
            model_preferences={"hints": [{"name": model, "priority": 1.0}]}
        )
        break
    except Exception:
        continue
```

4. **Cache expensive operations**:

```python
import hashlib

# Cache LLM results for expensive operations
cache = {}

def get_cache_key(prompt: str) -> str:
    return hashlib.md5(prompt.encode()).hexdigest()

@mcp.tool()
async def cached_analysis(data: str) -> str:
    cache_key = get_cache_key(data)

    if cache_key in cache:
        return cache[cache_key]

    result = await mcp.request_sampling(prompt=f"Analyze: {data}")
    cache[cache_key] = result
    return result
```

---

## Conclusion

Sampling represents the pinnacle of MCP capabilities, enabling true collaborative intelligence between clients and servers. By allowing servers to leverage the client's LLM, we create systems that are:

**More Intelligent** - Servers can perform reasoning and language understanding
**More Efficient** - No need for duplicate LLM infrastructure
**More Consistent** - Single LLM context across all operations
**More Powerful** - Combination of server logic and LLM capabilities

Key takeaways from this part:

1. **Sampling reverses control flow** - Servers can request LLM completions from clients
2. **Collaborative intelligence** - Best of both worlds: server logic + LLM reasoning
3. **Advanced patterns** - Multi-step reasoning, collaborative workflows, intelligent analysis
4. **Robust implementation** - Error handling, fallbacks, and best practices are crucial

```{admonition} Next Chapter Preview
:class: note
In the next part, we'll shift focus from functionality to reliability and security, exploring testing strategies, security considerations, and sandboxing techniques for production MCP deployments.

Read Part 6 here: [The Full MCP Blueprint: Part 6 - Testing, Security and Sandboxing in MCPs (Part A)](../part-6/mcp-blueprint-part-6.md)
```

With sampling, you now have the complete MCP toolkit to build sophisticated AI systems that combine the best of structured programming with flexible language understanding. The possibilities are truly limitless!

Thanks for reading!

---

## Discussion

Any questions? Feel free to post them in the comments or connect via chat for private discussions.

```{admonition} Tags
:class: note
Topics: Agents, MCP, Sampling, LLM, Collaborative Intelligence, Advanced Workflows
```
