# coding=utf-8
# Copyright 2021 NVIDIA Corporation. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
FROM nvcr.io/nvidia/pytorch:23.07-py3
# CVE-2026-111943: upgrade the OS CA bundle from the distro security repo and fail if it is still vulnerable
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/* && \
    v="$(dpkg-query -W -f='${Version}' ca-certificates)" && \
    dpkg --compare-versions "$v" ge 20260816 || { echo "ca-certificates $v is below 20260816"; exit 1; }
LABEL maintainer="Hugging Face"
LABEL repository="transformers"

RUN apt-get update
RUN apt-get install -y ca-certificates sudo

# Pillow 13.0.0 (CVE-2026-96639 and others) requires Python >= 3.11, but every NGC PyTorch image that still ships
# TensorRT 8.x (used by evaluate-hf-trt-qa.py) runs Python 3.10. Keep this base for CUDA 12.1 / TensorRT 8.6
# (trtexec) and run Python from a deadsnakes 3.11 venv that comes first on PATH. The base's own Python 3.10
# Pillow is removed so the image does not ship a vulnerable copy.
RUN python3 -m pip uninstall -y pillow && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends software-properties-common gpg-agent && \
    add-apt-repository -y ppa:deadsnakes/ppa && apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends python3.11 python3.11-dev python3.11-venv && \
    rm -rf /var/lib/apt/lists/* && python3.11 -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH

RUN python3 -m pip install --no-cache-dir --upgrade pip
# Same torch (2.1.0, CUDA 12.1) and TensorRT (8.6.1) versions as the 23.07 base, built for Python 3.11
RUN python3 -m pip install --no-cache-dir torch==2.1.0 "numpy<2" --extra-index-url https://download.pytorch.org/whl/cu121
RUN python3 -m pip install --no-cache-dir tensorrt-bindings==8.6.1 tensorrt-libs==8.6.1 --extra-index-url https://pypi.nvidia.com
RUN python3 -m pip install --no-cache-dir pycuda
# 2.1.3 is the newest pytorch-quantization wheel that installs on Python 3.11 (2.2.x wheels are cp38/cp310 only)
RUN python3 -m pip install --no-cache-dir \
    pytorch-quantization==2.1.3 --extra-index-url https://pypi.nvidia.com
RUN python3 -m pip install --no-cache-dir onnxruntime-gpu==1.19.2

WORKDIR /workspace
COPY . transformers/
RUN cd transformers/ && \
    python3 -m pip install --no-cache-dir .

RUN python3 -m pip install --no-cache-dir datasets \
    accelerate
