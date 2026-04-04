FROM mcr.microsoft.com/devcontainers/python:3.11

# Python setup
RUN pip install --upgrade pip setuptools wheel

# Python deps
COPY requirements-dev.txt /tmp/requirements-dev.txt

RUN if [ -f /tmp/requirements-dev.txt ]; then \
    pip install --no-cache-dir -r /tmp/requirements-dev.txt; \
    fi