FROM mcr.microsoft.com/devcontainers/python:3.13

# Avoid prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Install system dependencies required by Home Assistant and MQTT
RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    libavcodec-dev \
    libavformat-dev \
    libavutil-dev \
    libjpeg-dev \
    libpcap-dev \
    libssl-dev \
    libturbojpeg0-dev \
    zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dev requirements
COPY .devcontainer/requirements-dev.txt /tmp/requirements-dev.txt
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r /tmp/requirements-dev.txt

# Set workspace directory
WORKDIR /workspaces/bmw-cardata-ha
