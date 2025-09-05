# Server Service - Web crawling and document processing microservice with security hardening

# Build stage
FROM python:3.12 AS builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY pyproject.toml uv.lock ./

RUN pip install --upgrade pip && pip install --user --no-cache-dir -e . --group server

# Runtime stage
FROM python:3.12-slim

# Re-declare ARG for runtime stage
ARG BUILD_VERSION="2025-09-01T13:00:00Z-remove-manual-playwright"


WORKDIR /app

# Set proper environment defaults for Kubernetes  
ENV HOST=0.0.0.0
ENV PORT=8181
ENV ARCHON_SERVER_PORT=8181
ENV LOG_LEVEL=INFO
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# Install runtime dependencies for Playwright (minimal set)
RUN apt-get update && apt-get install -y \
    wget \
    ca-certificates \
    fonts-liberation \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libatspi2.0-0 \
    libcups2 \
    libdbus-1-3 \
    libdrm2 \
    libgbm1 \
    libgtk-3-0 \
    libnspr4 \
    libnss3 \
    libwayland-client0 \
    libxcomposite1 \
    libxdamage1 \
    libxfixes3 \
    libxkbcommon0 \
    libxrandr2 \
    xdg-utils \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Copy Python packages from builder
COPY --from=builder /root/.local /root/.local

# Ensure playwright browsers are available  
ENV PATH=/root/.local/bin:$PATH
ENV PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers
ENV PYTHONPATH="/root/.local/lib/python3.12/site-packages"
# Install playwright browsers (playwright itself comes from crawl4ai dependency)
RUN echo "=== BUILD VERSION: ${BUILD_VERSION} ===" && \
    mkdir -p /opt/playwright-browsers && \
    python -m playwright install chromium && \
    chmod -R 755 /opt/playwright-browsers

# Copy server code and tests (selective copying like upstream)
COPY src/server/ src/server/
COPY src/__init__.py src/
COPY tests/ tests/

# Create non-root user for runtime security
RUN addgroup --gid 1001 archon && \
    adduser --disabled-password --gecos '' --uid 1001 --gid 1001 archon && \
    # Create home directory structure
    mkdir -p /home/archon/.cache && \
    chown -R archon:archon /app /home/archon && \
    # Allow non-root user access to Python packages in /root/.local
    chmod -R 755 /root/.local && \
    chmod 755 /root

# Switch to non-root user for runtime
USER 1001
ENV HOME=/home/archon
ENV PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers
ENV PYTHONPATH="/app:/root/.local/lib/python3.12/site-packages:$PYTHONPATH"

EXPOSE ${ARCHON_SERVER_PORT}

# Health check using Python (faster than curl for our use case)
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD python -c "import urllib.request, os; urllib.request.urlopen(f'http://localhost:{os.environ[\"ARCHON_SERVER_PORT\"]}/health').read()" || exit 1

# Run the server with configurable port (using JSON array format for better signal handling)
CMD ["sh", "-c", "python -m uvicorn src.server.main:socket_app --host 0.0.0.0 --port ${ARCHON_SERVER_PORT} --workers 1"]