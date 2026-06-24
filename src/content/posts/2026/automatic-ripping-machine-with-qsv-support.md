---
pubDatetime: 2026-06-24T16:37:23.000+02:00
title: Automatic Ripping Machine with Intel QSV Support
draft: false
tags:
  - homelab
  - media collection
description: Details about how I (and Claude) set up an Automatic Ripping Machine with Intel Quick Sync Video (QSV) support for efficient media ripping and encoding.
---

When you want to automate and digitize your physical media library, sooner or later you will come across the **Automatic Ripping Machine (ARM)**.

Things might get tricky when you want to leverage modern hardware acceleration.
I wanted to use **Intel Quick Sync Video (QSV)** on my Intel Arc 770 GPU, but the official Docker image is not up to date and does not support `oneVPL` yet.
This is where a custom build comes in handy.

So here is the documentation of my setup including the most important troubleshooting notes.

---

## The Problem: Outdated Intel Stack

According to Gemini, older releases of HandBrake use the old *Intel Media SDK*, which is not compatible with modern Intel graphics architectures under Linux.
Therefore, to utilize Intel Quick Sync Video (QSV) effectively, it is necessary to build HandBrake with support for the newer **oneVPL (oneAPI Video Processing Library)** (`libvpl2`).

To run ARM (v2.23.3) with the stable HandBrake version (v1.11.2) and full QSV support, I built a custom Docker image.
To avoid bloating the final image with unnecessary compiler tools, we use a **Multi-Stage Dockerfile**.

### Multi-Stage Dockerfile

The `Dockerfile` uses a temporary builder in the first stage to compile HandBrake from source.
The Ubuntu version is determined dynamically using `lsb_release`, ensuring future compatibility.

```dockerfile
# === STAGE 1: Builder (build only in this stage) ===
FROM automaticrippingmachine/automatic-ripping-machine:2.23.3 AS builder

USER root

# Install dependencies & add Intel repo dynamically based on OS version
RUN apt-get update && apt-get install -y wget gpg lsb-release && \
    wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | \
    gpg --dearmor --output /usr/share/keyrings/intel-graphics.gpg && \
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu $(lsb_release -cs) client" > \
    /etc/apt/sources.list.d/intel-graphics.list && \
    apt-get update && apt-get install -y \
    build-essential \
    cmake \
    git \
    autoconf \
    libtool \
    pkg-config \
    libva-dev \
    libmfx-gen1.2 \
    libvpl2

# Clone HandBrake 1.11.2 and build with QSV
WORKDIR /tmp
RUN git clone --depth 1 --branch 1.11.2 https://github.com/HandBrake/HandBrake.git && \
    cd HandBrake && \
    ./configure --enable-qsv --disable-gtk --launch-jobs=$(nproc) --launch && \
    cd build && \
    make install


# === STAGE 2: Final image (slim for runtime) ===
FROM automaticrippingmachine/automatic-ripping-machine:2.23.3

USER root

# Install only the Intel graphics drivers required for runtime
RUN apt-get update && apt-get install -y wget gpg lsb-release && \
    wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | \
    gpg --dearmor --output /usr/share/keyrings/intel-graphics.gpg && \
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu $(lsb_release -cs) client" > \
    /etc/apt/sources.list.d/intel-graphics.list && \
    apt-get update && apt-get install -y \
    intel-media-va-driver-non-free \
    libmfx-gen1.2 \
    libvpl2 \
    && rm -rf /var/lib/apt/lists/*

# Copy the finished HandBrake 1.11.2 binary from the builder
COPY --from=builder /usr/local/bin/HandBrakeCLI /usr/bin/HandBrakeCLI

# Add the 'arm' user to the video and render groups
RUN usermod -a -G video arm 2>/dev/null || true && \
    usermod -a -G render arm 2>/dev/null || true

WORKDIR /opt/arm
```

Build the Docker image with the following command:

```bash
docker build --no-cache -t arm-qsv . 
```

## Troubleshooting

While running the ARM container I ran into several issues, which I want to document here for future reference.

### 1. Infinite Udev-Eject-Loop

**Problem:** After ripping a disc, the disc is ejected, but the ARM container immediately tries to eject it again, resulting in an infinite loop.

**Solution:** Modify your `arm.yaml` configuration file to include the following line:

```yaml
UNIDENTIFIED_EJECT: false
```

The documentation explicitly mentions my specfic problem and disc drive. Oops!

### 2. MakeMKV Key-Update fails (`URL_ERROR`)

**Problem:** For some reason, the MakeMKV key update failed with an `URL_ERROR` for me: `Error updating MakeMKV key, return code: 40 URL_ERROR`.

**Solution:** A quick solution was to specify DNS servers at the start of the container (or in the `docker-compose.yml`):

```bash
docker run docker run -d --name arm-qsv --dns 1.1.1.1 --dns 1.0.0.1 arm-qsv
```

Or in the `docker-compose.yml`:

```yaml
services:
  arm:
    image: arm-qsv
    container_name: arm-qsv
    dns:
      - 1.1.1.1 # Cloudflare DNS
      - 1.0.0.1
    devices:
      - /dev/sr0:/dev/sr0 # Optical drive
      - /dev/dri:/dev/dri # Intel Quick Sync GPU
```

---

## Final Notes

The content of this post was partially generated with the help of AI (Gemini).
The Dockerfile was generated by Gemini and Claude.

This Dockerfile uses third-party components and hard-coded URLs to download software from the internet. Always verify the integrity of downloaded software and ensure you are using trusted sources.

This setup works for me. Use at your own risk! :D