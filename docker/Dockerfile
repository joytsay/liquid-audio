# syntax=docker/dockerfile:1
ARG BASE_IMAGE=dustynv/vllm:r36.4-cu129-24.04
FROM ghcr.io/astral-sh/uv:0.8.13 AS uv
FROM ${BASE_IMAGE} AS runtime

ENV DEBIAN_FRONTEND=noninteractive \
    UV_PYTHON_DOWNLOADS=never \
    PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates ffmpeg libsndfile1 libgomp1 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=uv /uv /usr/local/bin/uv

# Read metadata without importing CUDA modules or requiring a GPU at build time.
# Pin the image's exact versions, including any vendor/local version suffixes.
RUN python3 - <<'PY'
import sys
from importlib.metadata import version
from pathlib import Path

if sys.version_info < (3, 12):
    raise SystemExit("The base image must provide Python 3.12 or newer")
pins = [f"{name}=={version(name)}" for name in ("torch", "torchaudio")]
Path("/opt/constraints.txt").write_text("\n".join(pins) + "\n")
PY

# Reuse the image's active Python environment, including any existing venv.
# Creating /opt/venv here could erase the image's precompiled packages.

WORKDIR /opt/liquid-audio
COPY pyproject.toml README.md LICENSE ./
COPY docker/demo-constraints.txt /opt/demo-constraints.txt
# Resolve third-party dependencies before copying frequently changed source.
RUN --mount=type=cache,target=/root/.cache/uv \
    uv pip install --python python3 --index-url https://pypi.org/simple \
    --constraint /opt/constraints.txt --constraint /opt/demo-constraints.txt \
    --requirements pyproject.toml --extra demo
COPY src ./src
RUN --mount=type=cache,target=/root/.cache/uv \
    uv pip install --python python3 --index-url https://pypi.org/simple --no-deps .

WORKDIR /workspace
EXPOSE 7860
ENTRYPOINT []
CMD ["python3", "-m", "liquid_audio.demo.chat"]
