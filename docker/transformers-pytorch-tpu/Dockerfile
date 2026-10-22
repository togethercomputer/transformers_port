FROM google/cloud-sdk:slim
# CVE-2026-111943: upgrade the OS CA bundle from the distro security repo and fail if it is still vulnerable
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/* && \
    v="$(dpkg-query -W -f='${Version}' ca-certificates)" && \
    dpkg --compare-versions "$v" ge 20250419+deb13u1 || { echo "ca-certificates $v is below 20250419+deb13u1"; exit 1; }

# Build args.
ARG GITHUB_REF=refs/heads/main

# Python >= 3.11 is required by Pillow 13.0.0 (CVE-2026-96639 and others), which torchvision pulls in.
# torch_xla 2.7.0 publishes cp311 wheels.
ENV PYTHON_VERSION=3.11

RUN apt-get update && apt-get install -y --no-install-recommends \
         build-essential \
         cmake \
         git \
         curl \
         ca-certificates

# Install conda and python.
# Miniforge (conda-forge channel): Miniconda 4.7.12 cannot create a Python 3.11 env.
RUN curl -fsSL -o ~/miniforge.sh https://github.com/conda-forge/miniforge/releases/download/26.7.2-1/Miniforge3-26.7.2-1-Linux-x86_64.sh && \
    bash ~/miniforge.sh -b -p /root/miniconda3 && \
    rm ~/miniforge.sh

ENV PATH=/root/miniconda3/bin:$PATH

RUN conda create -y --name container python=$PYTHON_VERSION

# Run the rest of commands within the new conda env.
# Use absolute path to appease Codefactor.
SHELL ["/root/miniconda3/bin/conda", "run", "-n", "container", "/bin/bash", "-c"]
RUN conda install -y python=$PYTHON_VERSION mkl

# The gs://tpu-pytorch nightly wheels only exist for old Pythons (cp36m/cp37m); install the PyPI torch_xla release instead.
RUN pip install --no-cache-dir torch==2.7.0 torchvision==0.22.0 'torch_xla[tpu]==2.7.0' \
        -f https://storage.googleapis.com/libtpu-releases/index.html && \
    pip install --no-cache-dir -U "pillow>=13.0.0" && \
    apt-get update && apt-get install -y libomp5

ENV LD_LIBRARY_PATH=root/miniconda3/envs/container/lib


# Install huggingface/transformers at the current PR, plus dependencies.
RUN git clone https://github.com/huggingface/transformers.git && \
    cd transformers && \
    git fetch origin $GITHUB_REF:CI && \
    git checkout CI && \
    cd .. && \
    pip install ./transformers && \
    pip install -r ./transformers/examples/pytorch/_tests_requirements.txt && \
    pip install pytest

RUN python -c "import torch_xla; print(torch_xla.__version__)"
RUN python -c "import transformers as trf; print(trf.__version__)"
RUN conda init bash
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
CMD ["bash"]
