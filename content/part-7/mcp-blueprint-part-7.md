---
title: "The Full MCP Blueprint: Part 7 - Testing, Security, and Sandboxing in MCPs (Part B)"
subtitle: "Model context protocol crash course—Part 7"
authors:
  - name: Avi Chawla
  - name: Akshay Pachaar
date: "July 6, 2025"
source: "Daily Dose of Data Science"
---

# The Full MCP Blueprint: Part 7 - Testing, Security, and Sandboxing in MCPs (Part B)

_Model context protocol crash course—Part 7._

**Authors:** Avi Chawla, Akshay Pachaar  
**Date:** July 6, 2025  
**Source:** Daily Dose of Data Science

---

## Table of Contents

```{contents}
:depth: 3
```

---

## Recap of Part A

In the first part of this two-part chapter, we moved from functionality to safety, laying down the groundwork for testing and securing MCP systems.

We introduced the MCP Inspector, which acts as a live debugging interface for visualizing, replaying, and inspecting server behavior. It's an essential tool for development, troubleshooting, and validating how your server behaves in real time.

We then dove into the security landscape of MCP, examining key risks and vulnerabilities:

- Prompt injection via tool poisoning
- Server impersonation and tool name collisions
- Rug pull-style trust violations
- Overexposure of capabilities, especially via filesystem tools

To mitigate these, we introduced MCP Roots, which let you define file access boundaries for your server and implemented them using FastMCP, alongside additional server initialization logic and client integration.

```{admonition} Prerequisites
:class: note
If you haven't studied Part 6 yet, we strongly recommend going through it first. You can read it here: [The Full MCP Blueprint: Part 6 - Testing, Security and Sandboxing in MCPs (Part A)](../part-6/mcp-blueprint-part-6.md)
```

With that security foundation in place, we're now ready to tackle the final layer of protection: **Sandboxing**.

---

## What's in Part B

In this second installment, we'll focus on containerizing and sandboxing your MCP servers using Docker, ensuring that even if a server is compromised or receives unsafe input, the damage is confined.

We'll cover:

- How to containerize a FastMCP server from scratch
- Security-hardened Docker configurations
- Network isolation and resource limits
- Integration with Claude Desktop and Cursor
- Testing sandboxed servers with MCP Inspector

---

## Sandboxing MCP Servers with Docker

Docker containerization provides an excellent sandboxing solution for MCP servers, offering:

- **Process isolation** - Server runs in isolated environment
- **Resource limits** - CPU, memory, and disk usage controls
- **Network restrictions** - Limited network access
- **Filesystem boundaries** - Restricted file system access
- **Easy deployment** - Consistent environment across systems

### Basic Docker Setup

Let's start with a simple Dockerfile for an MCP server:

```dockerfile
# Dockerfile for MCP Server
FROM python:3.11-slim

# Create non-root user
RUN groupadd -r mcpuser && useradd -r -g mcpuser mcpuser

# Set working directory
WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy server code
COPY server.py .
COPY --chown=mcpuser:mcpuser . .

# Switch to non-root user
USER mcpuser

# Expose port (for SSE transport)
EXPOSE 8000

# Run server
CMD ["python", "server.py"]
```

### Security-Hardened Configuration

For production use, we need additional security measures:

```dockerfile
# Security-hardened Dockerfile
FROM python:3.11-slim

# Update system and install security updates
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Create restricted user
RUN groupadd -r mcpuser && \
    useradd -r -g mcpuser -d /app -s /bin/false mcpuser

# Set secure permissions
WORKDIR /app
RUN chown mcpuser:mcpuser /app

# Install Python dependencies as root
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    pip cache purge

# Copy application code
COPY --chown=mcpuser:mcpuser server.py .
COPY --chown=mcpuser:mcpuser config/ ./config/

# Create restricted directories
RUN mkdir -p /app/data /app/logs && \
    chown mcpuser:mcpuser /app/data /app/logs && \
    chmod 755 /app/data /app/logs

# Switch to non-root user
USER mcpuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

# Expose port
EXPOSE 8000

# Run with restricted resources
CMD ["python", "-u", "server.py"]
```

### Docker Compose for Complete Setup

```yaml
# docker-compose.yml
version: "3.8"

services:
  mcp-server:
    build: .
    container_name: mcp-server
    restart: unless-stopped

    # Security options
    security_opt:
      - no-new-privileges:true
    read_only: true

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.1"
          memory: 128M

    # Network restrictions
    networks:
      - mcp-network
    ports:
      - "127.0.0.1:8000:8000" # Bind only to localhost

    # Volume mounts (read-only where possible)
    volumes:
      - ./data:/app/data:rw,noexec,nosuid,nodev
      - ./logs:/app/logs:rw,noexec,nosuid,nodev
      - /tmp:/tmp:ro,noexec,nosuid,nodev

    # Environment variables
    environment:
      - PYTHONUNBUFFERED=1
      - MCP_LOG_LEVEL=INFO
      - MCP_MAX_CONNECTIONS=10

    # Process limits
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000

networks:
  mcp-network:
    driver: bridge
    internal: true # No external network access
```

---

## Building a Docker-Sandboxed Server

Let's create a complete example of a sandboxed MCP server:

### Server Code

```python
# server.py
import os
import logging
from pathlib import Path
from fastmcp import FastMCP

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/app/logs/server.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

# Create MCP server
mcp = FastMCP("Sandboxed Server")

# Define safe data directory
DATA_DIR = Path("/app/data")
DATA_DIR.mkdir(exist_ok=True)

@mcp.tool()
def safe_write_file(filename: str, content: str) -> str:
    """Write content to a file in the safe data directory."""
    try:
        # Validate filename
        if not filename.replace('.', '').replace('_', '').replace('-', '').isalnum():
            raise ValueError("Invalid filename format")

        if len(filename) > 100:
            raise ValueError("Filename too long")

        # Ensure file is in safe directory
        file_path = DATA_DIR / filename
        file_path = file_path.resolve()

        if not str(file_path).startswith(str(DATA_DIR.resolve())):
            raise ValueError("Path traversal attempt detected")

        # Write file with size limit
        if len(content) > 10000:  # 10KB limit
            raise ValueError("Content too large")

        with open(file_path, 'w') as f:
            f.write(content)

        logger.info(f"File written: {filename}")
        return f"Successfully wrote {len(content)} characters to {filename}"

    except Exception as e:
        logger.error(f"File write error: {str(e)}")
        raise

@mcp.tool()
def safe_read_file(filename: str) -> str:
    """Read content from a file in the safe data directory."""
    try:
        # Validate filename
        if not filename.replace('.', '').replace('_', '').replace('-', '').isalnum():
            raise ValueError("Invalid filename format")

        # Ensure file is in safe directory
        file_path = DATA_DIR / filename
        file_path = file_path.resolve()

        if not str(file_path).startswith(str(DATA_DIR.resolve())):
            raise ValueError("Path traversal attempt detected")

        if not file_path.exists():
            raise FileNotFoundError(f"File not found: {filename}")

        # Read with size limit
        if file_path.stat().st_size > 50000:  # 50KB limit
            raise ValueError("File too large to read")

        with open(file_path, 'r') as f:
            content = f.read()

        logger.info(f"File read: {filename}")
        return content

    except Exception as e:
        logger.error(f"File read error: {str(e)}")
        raise

@mcp.tool()
def list_files() -> str:
    """List files in the safe data directory."""
    try:
        files = []
        for file_path in DATA_DIR.iterdir():
            if file_path.is_file():
                stat = file_path.stat()
                files.append({
                    "name": file_path.name,
                    "size": stat.st_size,
                    "modified": stat.st_mtime
                })

        logger.info(f"Listed {len(files)} files")
        return str(files)

    except Exception as e:
        logger.error(f"List files error: {str(e)}")
        raise

@mcp.resource("health://status")
def health_check() -> str:
    """Health check endpoint."""
    return "Server is healthy"

if __name__ == "__main__":
    logger.info("Starting sandboxed MCP server")
    mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

### Requirements File

```text
# requirements.txt
fastmcp>=0.1.0
uvicorn>=0.20.0
fastapi>=0.100.0
pydantic>=2.0.0
```

### Build and Run

```bash
# Build the Docker image
docker-compose build

# Run the sandboxed server
docker-compose up -d

# Check logs
docker-compose logs -f mcp-server

# Stop the server
docker-compose down
```

---

## MCP Inspector and Dockerized Server

You can use MCP Inspector to test your Dockerized server:

### Testing with Inspector

```bash
# Test the containerized server
npx @modelcontextprotocol/inspector http://localhost:8000/sse
```

### Advanced Inspector Configuration

Create an inspector configuration file:

```json
{
  "servers": {
    "sandboxed-server": {
      "url": "http://localhost:8000/sse",
      "name": "Sandboxed MCP Server",
      "description": "Docker containerized MCP server for safe execution"
    }
  },
  "testing": {
    "timeout": 30000,
    "retries": 3
  }
}
```

### Integration Testing Script

```python
# test_sandboxed_server.py
import asyncio
import json
from mcp.client.sse import sse_client
from mcp import ClientSession

async def test_sandboxed_server():
    """Test the sandboxed MCP server."""

    print("🧪 Testing Sandboxed MCP Server")

    try:
        async with sse_client("http://localhost:8000/sse") as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()

                # Test tool listing
                tools = await session.list_tools()
                print(f"✅ Found {len(tools.tools)} tools")

                # Test safe file operations
                print("\n📁 Testing file operations...")

                # Write a test file
                result = await session.call_tool("safe_write_file", {
                    "filename": "test.txt",
                    "content": "Hello from sandboxed server!"
                })
                print(f"✅ Write result: {result.content[0].text}")

                # Read the test file
                result = await session.call_tool("safe_read_file", {
                    "filename": "test.txt"
                })
                print(f"✅ Read result: {result.content[0].text}")

                # List files
                result = await session.call_tool("list_files", {})
                print(f"✅ Files: {result.content[0].text}")

                # Test security boundaries
                print("\n🔒 Testing security boundaries...")

                try:
                    await session.call_tool("safe_read_file", {
                        "filename": "../etc/passwd"
                    })
                    print("❌ Security test failed - path traversal allowed")
                except Exception as e:
                    print(f"✅ Security test passed - path traversal blocked: {str(e)}")

                print("\n🎉 All tests completed!")

    except Exception as e:
        print(f"❌ Test failed: {str(e)}")

if __name__ == "__main__":
    asyncio.run(test_sandboxed_server())
```

---

## Production Deployment Considerations

### Container Security Best Practices

1. **Use minimal base images**
2. **Run as non-root user**
3. **Enable read-only filesystem**
4. **Set resource limits**
5. **Disable unnecessary capabilities**
6. **Use security scanning tools**

### Monitoring and Logging

```yaml
# docker-compose.production.yml
version: "3.8"

services:
  mcp-server:
    build: .
    # ... previous configuration ...

    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

    # Health monitoring
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  monitoring:
    image: prom/prometheus
    container_name: mcp-monitoring
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - mcp-network
```

### Automated Security Scanning

```bash
# security-scan.sh
#!/bin/bash

echo "🔍 Running security scans..."

# Scan Docker image for vulnerabilities
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image mcp-server:latest

# Check for secrets in code
docker run --rm -v "$(pwd):/app" trufflesecurity/truffletog:latest filesystem /app

# Analyze Dockerfile
docker run --rm -i hadolint/hadolint < Dockerfile

echo "✅ Security scan completed"
```

---

## Conclusion

In this part, we've completed our comprehensive security coverage by implementing:

- **Docker containerization** for process isolation
- **Security-hardened configurations** with minimal attack surface
- **Resource limits and restrictions** to prevent abuse
- **Testing strategies** for sandboxed environments
- **Production deployment** best practices

```{admonition} Security Defense in Depth
:class: tip
Effective MCP security requires multiple layers:
1. **Input validation** and sanitization
2. **Access controls** and permission boundaries
3. **Process isolation** through containerization
4. **Monitoring and logging** for threat detection
5. **Regular updates** and security patches
```

```{admonition} Next Chapter Preview
:class: note
In Part 8, we'll explore integrating MCP with popular agentic frameworks like LangGraph, CrewAI, LlamaIndex, and PydanticAI, showing how MCP enhances existing AI workflows.

Read Part 8 here: [The Full MCP Blueprint: Part 8 - Practical MCP Integration with 4 Popular Agentic Frameworks](../part-8/mcp-blueprint-part-8.md)
```

By combining the security measures from Parts 6 and 7, you now have a complete toolkit for building secure, production-ready MCP systems that users can trust.

Thanks for reading!

---

## Discussion

Any questions? Feel free to post them in the comments or connect via chat for private discussions.

```{admonition} Tags
:class: note
Topics: Agents, MCP, Docker, Sandboxing, Security, Production Deployment
```
