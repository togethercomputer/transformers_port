FROM nvidia/cuda:11.8.0-cudnn8-devel-ubuntu22.04
# CVE-2026-111943: upgrade the OS CA bundle from the distro security repo and fail if it is still vulnerable
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/* && \
    v="$(dpkg-query -W -f='${Version}' ca-certificates)" && \
    dpkg --compare-versions "$v" ge 20260816 || { echo "ca-certificates $v is below 20260816"; exit 1; }
# Pillow 13.0.0 (CVE-2026-96639 and others) requires Python >= 3.11, but Ubuntu 22.04 ships 3.10:
# install CPython 3.11 from the deadsnakes PPA into a venv that comes first on PATH (python3/pip resolve to it)
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends software-properties-common gpg-agent && \
    add-apt-repository -y ppa:deadsnakes/ppa && apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends python3.11 python3.11-dev python3.11-venv && \
    rm -rf /var/lib/apt/lists/* && python3.11 -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
LABEL maintainer="Hugging Face"
LABEL repository="transformers"

RUN apt update && \
    apt install -y bash \
                   build-essential \
                   git \
                   curl \
                   ca-certificates && \
    rm -rf /var/lib/apt/lists

RUN python3 -m pip install --no-cache-dir --upgrade pip && \
    python3 -m pip install --no-cache-dir \
    jupyter \
    tensorflow \
    torch

RUN git clone https://github.com/NVIDIA/apex
RUN cd apex && \
    python3 setup.py install && \
    pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" ./

WORKDIR /workspace
COPY . transformers/
RUN cd transformers/ && \
    python3 -m pip install --no-cache-dir .

CMD ["/bin/bash"]
