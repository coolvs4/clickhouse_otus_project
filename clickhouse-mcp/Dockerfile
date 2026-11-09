
FROM python:3.14-slim

ENV PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

RUN python -m venv /opt/mcpo-env \
    && python -m venv /opt/clickhouse-env

RUN /opt/mcpo-env/bin/pip install --upgrade pip==26.2.2 \
    && /opt/mcpo-env/bin/pip install \
        mcpo==0.0.20 \
        mcp==1.30.0 \
        fastapi==0.129.2 \
        starlette==0.52.1 \
        pydantic==2.14.2 \
        pydantic-core==2.50.2 \
        uvicorn==0.54.0 \
        anyio==4.16.0 \
        httpx==0.28.1

RUN /opt/clickhouse-env/bin/pip install --upgrade pip==26.2.2 \
    && /opt/clickhouse-env/bin/pip install \
        mcp-clickhouse==0.7.0 \
        fastmcp==4.0.11 \
        fastmcp-slim==4.0.11 \
        mcp==2.3.0 \
        pydantic==2.14.2 \
        pydantic-core==2.50.2 \
        pydantic-settings==2.16.2 \
        starlette==1.10.0 \
        uvicorn==0.54.0 \
        anyio==4.16.0

ENV MCP_PORT=8000

EXPOSE 8000

CMD ["/opt/mcpo-env/bin/mcpo", "--host", "0.0.0.0", "--port", "8000", "--", "/opt/clickhouse-env/bin/mcp-clickhouse"]
