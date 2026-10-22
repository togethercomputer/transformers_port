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

ARG DEBIAN_FRONTEND=noninteractive

# Use login shell to read variables from `~/.profile` (to pass dynamic created variables between RUN commands)
SHELL ["sh", "-lc"]

# The following `ARG` are mainly used to specify the versions explicitly & directly in this docker file, and not meant
# to be used as arguments for docker build (so far).

ARG PYTORCH='2.0.1'
# (not always a valid torch version)
ARG INTEL_TORCH_EXT='1.11.0'
# Example: `cu102`, `cu113`, etc.
ARG CUDA='cu118'

RUN apt update
RUN apt install -y ca-certificates git libsndfile1-dev tesseract-ocr espeak-ng ffmpeg git-lfs
RUN git lfs install
RUN python3 -m pip install --no-cache-dir --upgrade pip

ARG REF=main
RUN git clone https://github.com/huggingface/transformers && cd transformers && git checkout $REF
RUN python3 -m pip install --no-cache-dir -e ./transformers[dev,onnxruntime]

# TODO: Handle these in a python utility script
RUN [ ${#PYTORCH} -gt 0 -a "$PYTORCH" != "pre" ] && VERSION='torch=='$PYTORCH'.*' ||  VERSION='torch'; echo "export VERSION='$VERSION'" >> ~/.profile
RUN echo torch=$VERSION
# `torchvision` and `torchaudio` should be installed along with `torch`, especially for nightly build.
# Currently, let's just use their latest releases (when `torch` is installed with a release version)
# TODO: We might need to specify proper versions that work with a specific torch version (especially for past CI).
RUN [ "$PYTORCH" != "pre" ] && python3 -m pip install --no-cache-dir -U $VERSION torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/$CUDA || python3 -m pip install --no-cache-dir -U --pre torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/nightly/$CUDA

RUN python3 -m pip install --no-cache-dir -U tensorflow==2.12 protobuf==3.20.3 tensorflow_text tensorflow_probability
RUN python3 -m pip uninstall -y flax jax

RUN python3 -m pip install --no-cache-dir intel_extension_for_pytorch==$INTEL_TORCH_EXT+cpu -f https://developer.intel.com/ipex-whl-stable-cpu

RUN python3 -m pip install --no-cache-dir git+https://github.com/facebookresearch/detectron2.git pytesseract
RUN python3 -m pip install -U "itsdangerous<2.1.0"

RUN python3 -m pip install --no-cache-dir git+https://github.com/huggingface/accelerate@main#egg=accelerate

# Add bitsandbytes for mixed int8 testing
RUN python3 -m pip install --no-cache-dir bitsandbytes

# For bettertransformer
RUN python3 -m pip install --no-cache-dir optimum

# For video model testing
RUN python3 -m pip install --no-cache-dir decord av==9.2.0

# For `dinat` model
RUN python3 -m pip install --no-cache-dir natten -f https://shi-labs.com/natten/wheels/$CUDA/

# When installing in editable mode, `transformers` is not recognized as a package.
# this line must be added in order for python to be aware of transformers.
RUN cd transformers && python3 setup.py develop
