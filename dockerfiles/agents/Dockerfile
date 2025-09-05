# Fast single-stage agents build with security hardening
FROM python:3.12-slim

# Set proper environment defaults for Kubernetes
ENV HOST=0.0.0.0
ENV PORT=8052
ENV ARCHON_AGENTS_PORT=8052
ENV LOG_LEVEL=INFO
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONPATH=/app:/app/src

WORKDIR /app

# Install system dependencies and clean up in single layer
RUN apt-get update && apt-get install -y \
    gcc \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Copy and install requirements from local source
COPY pyproject.toml uv.lock ./

RUN pip install --upgrade pip && pip install --no-cache-dir -e . --group agents --group server

# Copy application code
COPY . .

# Create non-root user for security (works standalone + Kubernetes can override)
RUN addgroup --gid 1001 archon && \
    adduser --disabled-password --gecos '' --uid 1001 --gid 1001 archon && \
    chown -R archon:archon /app

USER 1001

EXPOSE ${ARCHON_AGENTS_PORT}

# Health check using Python
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD python -c "import urllib.request, os; urllib.request.urlopen(f'http://localhost:{os.environ[\"ARCHON_AGENTS_PORT\"]}/health').read()" || exit 1

# Run the agents service (matching original Archon structure)
CMD sh -c "python -m uvicorn src.agents.server:app --host 0.0.0.0 --port ${ARCHON_AGENTS_PORT}"