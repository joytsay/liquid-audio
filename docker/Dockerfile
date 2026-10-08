# syntax=docker/dockerfile:1
ARG BASE_IMAGE=nvcr.io/nvidia/l4t-jetpack:r36.4.0
FROM ghcr.io/astral-sh/uv:0.8.13 AS uv
FROM ${BASE_IMAGE} AS base

ENV DEBIAN_FRONTEND=noninteractive \
    UV_PYTHON_INSTALL_DIR=/opt/python \
    PATH=/opt/venv/bin:$PATH \
    PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates ffmpeg libsndfile1 libgomp1 libopenblas0-pthread \
    && rm -rf /var/lib/apt/lists/*
COPY --from=uv /uv /usr/local/bin/uv

# The host's Python 3.10 does not satisfy this project's Python >=3.12 requirement.
RUN uv python install 3.12 && uv venv --python 3.12 /opt/venv
COPY docker/constraints.txt /opt/constraints.txt

# The Jetson index does not provide the required 2.8.0 cp312 wheels.
# Build both packages against JetPack's CUDA/cuDNN, targeting Orin (SM 8.7).
FROM base AS torch-builder
ARG MAX_JOBS=2
ENV CUDA_HOME=/usr/local/cuda \
    PATH=/opt/venv/bin:/usr/local/cuda/bin:$PATH \
    TORCH_CUDA_ARCH_LIST=8.7 \
    MAX_JOBS=${MAX_JOBS} \
    CMAKE_BUILD_PARALLEL_LEVEL=${MAX_JOBS} \
    USE_CUDA=1 \
    USE_CUDNN=1 \
    USE_CUSPARSELT=0 \
    USE_DISTRIBUTED=1 \
    USE_NCCL=0 \
    USE_MPI=0 \
    USE_TENSORPIPE=0 \
    USE_KINETO=0 \
    USE_MKLDNN=0 \
    BUILD_TEST=0
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential git libopenblas-dev \
    && rm -rf /var/lib/apt/lists/*
RUN uv pip install --python /opt/venv/bin/python \
    pip 'setuptools>=62.3,<80' wheel 'cmake>=3.27,<4' ninja \
    astunparse numpy pyyaml 'typing-extensions>=4.10' requests packaging
RUN git clone --branch v2.8.0 --depth 1 --recurse-submodules --shallow-submodules \
    https://github.com/pytorch/pytorch.git /opt/pytorch
WORKDIR /opt/pytorch
RUN PYTORCH_BUILD_VERSION=2.8.0+jetson PYTORCH_BUILD_NUMBER=1 \
    python -m pip wheel --no-build-isolation --no-deps --wheel-dir /opt/wheels .
RUN uv pip install --python /opt/venv/bin/python \
    --constraint /opt/constraints.txt /opt/wheels/torch-*.whl

RUN git clone --branch v2.8.0 --depth 1 --recurse-submodules --shallow-submodules \
    https://github.com/pytorch/audio.git /opt/torchaudio
WORKDIR /opt/torchaudio
# Audio decoding uses soundfile/librosa; omit unused optional codec extensions.
RUN BUILD_VERSION=2.8.0+jetson PYTORCH_VERSION=2.8.0+jetson \
    BUILD_SOX=0 USE_FFMPEG=0 BUILD_CUDA_CTC_DECODER=0 \
    python -m pip wheel --no-build-isolation --no-deps --wheel-dir /opt/wheels .

FROM base AS runtime
COPY --from=torch-builder /opt/wheels /opt/wheels
RUN uv pip install --python /opt/venv/bin/python \
    --constraint /opt/constraints.txt /opt/wheels/*.whl \
    && rm -rf /opt/wheels

WORKDIR /opt/liquid-audio
COPY pyproject.toml README.md LICENSE ./
COPY src ./src
RUN uv pip install --python /opt/venv/bin/python \
    --constraint /opt/constraints.txt '.[demo]'

WORKDIR /workspace
EXPOSE 7860
CMD ["python3", "-m", "liquid_audio.demo.chat"]
