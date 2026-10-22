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

RUN python3 -m pip install --no-cache-dir --upgrade pip
RUN python3 -m pip install --no-cache-dir --ignore-installed pycuda
RUN python3 -m pip install --no-cache-dir \
    pytorch-quantization --extra-index-url https://pypi.ngc.nvidia.com
RUN python3 -m pip install --no-cache-dir onnxruntime-gpu==1.19.2

WORKDIR /workspace
COPY . transformers/
RUN cd transformers/ && \
    python3 -m pip install --no-cache-dir .

RUN python3 -m pip install --no-cache-dir datasets \
    accelerate
