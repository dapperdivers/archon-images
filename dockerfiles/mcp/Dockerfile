# MCP Service - Lightweight HTTP-based microservice  
FROM python:3.12-slim

WORKDIR /app

# Install system dependencies and clean up in single layer
RUN apt-get update && apt-get install -y \
    gcc \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Install dependencies
COPY pyproject.toml uv.lock ./

RUN pip install --upgrade pip && pip install --no-cache-dir -e . --group mcp --group server


# Create minimal directory structure
RUN mkdir -p src/mcp_server/modules src/server/services src/server/config

# Copy only MCP-specific files (lightweight protocol wrapper)
COPY src/mcp_server/ src/mcp_server/
COPY src/__init__.py src/

# Copy only the minimal server files MCP needs for HTTP communication
COPY src/server/__init__.py src/server/
COPY src/server/services/__init__.py src/server/services/
COPY src/server/services/mcp_service_client.py src/server/services/
COPY src/server/services/client_manager.py src/server/services/
COPY src/server/services/mcp_session_manager.py src/server/services/
COPY src/server/config/__init__.py src/server/config/
COPY src/server/config/service_discovery.py src/server/config/
COPY src/server/config/logfire_config.py src/server/config/

# Create non-root user for security
RUN addgroup --gid 1001 archon && \
    adduser --disabled-password --gecos '' --uid 1001 --gid 1001 archon && \
    chown -R archon:archon /app

USER 1001

# Set environment variables  
ENV PYTHONPATH="/app"
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV HOST=0.0.0.0
ENV PORT=8051
ENV ARCHON_MCP_PORT=8051
ENV LOG_LEVEL=INFO

# Expose MCP port
EXPOSE ${ARCHON_MCP_PORT}

# Note: MCP service may not have a /health endpoint by default
# Health check disabled - use TCP checks in Kubernetes instead

# Run the MCP server using module syntax (avoids naming conflicts)
CMD ["python", "-m", "src.mcp_server.mcp_server"]