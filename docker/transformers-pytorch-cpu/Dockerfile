FROM ubuntu:22.04
# CVE-2026-111943: upgrade the OS CA bundle from the distro security repo and fail if it is still vulnerable
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/* && \
    v="$(dpkg-query -W -f='${Version}' ca-certificates)" && \
    dpkg --compare-versions "$v" ge 20260816 || { echo "ca-certificates $v is below 20260816"; exit 1; }
LABEL maintainer="Hugging Face"
LABEL repository="transformers"

RUN apt update && \
    apt install -y bash \
                   build-essential \
                   git \
                   curl \
                   ca-certificates \
                   python3 \
                   python3-pip && \
    rm -rf /var/lib/apt/lists

RUN python3 -m pip install --no-cache-dir --upgrade pip && \
    python3 -m pip install --no-cache-dir \
    jupyter \
    torch

WORKDIR /workspace
COPY . transformers/
RUN cd transformers/ && \
    python3 -m pip install --no-cache-dir .

CMD ["/bin/bash"]