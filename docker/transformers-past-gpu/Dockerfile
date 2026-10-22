ARG BASE_DOCKER_IMAGE
FROM $BASE_DOCKER_IMAGE
# CVE-2026-111943: upgrade the OS CA bundle from the distro security repo and fail if it is still vulnerable
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/* && \
    . /etc/os-release && v="$(dpkg-query -W -f='${Version}' ca-certificates)" && \
    case "$ID:$VERSION_ID" in \
      ubuntu:*) min=20260816 ;; debian:12*) min=20250419~deb12u2 ;; debian:13*) min=20250419+deb13u1 ;; \
      *) echo "unsupported base $ID $VERSION_ID"; exit 1 ;; \
    esac && \
    dpkg --compare-versions "$v" ge "$min" || { echo "ca-certificates $v is below the fixed version"; exit 1; }
# Pillow 13.0.0 (CVE-2026-96639 and others) requires Python >= 3.11: create a venv from a >= 3.11 interpreter
# (deadsnakes' 3.11 on Ubuntu, whose 22.04 ships 3.10; the distro python3 on Debian 12+) and put it first on PATH
RUN . /etc/os-release && apt-get update && \
    if [ "$ID" = ubuntu ]; then \
      DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends software-properties-common gpg-agent && \
      add-apt-repository -y ppa:deadsnakes/ppa && apt-get update && \
      DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends python3.11 python3.11-dev python3.11-venv && py=python3.11; \
    else \
      DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends python3 python3-dev python3-venv && py=python3; \
    fi && \
    rm -rf /var/lib/apt/lists/* && "$py" -m venv /opt/venv && \
    /opt/venv/bin/python -c 'import sys; assert sys.version_info >= (3, 11), sys.version' && \
    echo 'export PATH=/opt/venv/bin:$PATH' > /etc/profile.d/00-venv.sh
# The profile.d entry keeps the venv first for the login shell below (Debian's /etc/profile resets PATH)
ENV PATH=/opt/venv/bin:$PATH
LABEL maintainer="Hugging Face"

ARG DEBIAN_FRONTEND=noninteractive

# Use login shell to read variables from `~/.profile` (to pass dynamic created variables between RUN commands)
SHELL ["sh", "-lc"]

RUN apt update
RUN apt install -y ca-certificates git libsndfile1-dev tesseract-ocr espeak-ng ffmpeg git-lfs libaio-dev
RUN git lfs install
RUN python3 -m pip install --no-cache-dir --upgrade pip

ARG REF=main
RUN git clone https://github.com/huggingface/transformers && cd transformers && git checkout $REF
RUN python3 -m pip install --no-cache-dir -e ./transformers[dev,onnxruntime]

# When installing in editable mode, `transformers` is not recognized as a package.
# this line must be added in order for python to be aware of transformers.
RUN cd transformers && python3 setup.py develop

ARG FRAMEWORK
ARG VERSION

# Control `setuptools` version to avoid some issues
RUN [ "$VERSION" != "1.10" ] && python3 -m pip install -U setuptools || python3 -m pip install -U "setuptools<=59.5"

# Remove all frameworks
RUN python3 -m pip uninstall -y torch torchvision torchaudio tensorflow jax flax

# Get the libraries and their versions to install, and write installation command to `~/.profile`.
RUN python3 ./transformers/utils/past_ci_versions.py --framework $FRAMEWORK --version $VERSION

# Install the target framework
RUN echo "INSTALL_CMD = $INSTALL_CMD"
RUN $INSTALL_CMD

RUN [ "$FRAMEWORK" != "pytorch" ] && echo "`deepspeed-testing` installation is skipped" || python3 -m pip install --no-cache-dir ./transformers[deepspeed-testing]

# Remove `accelerate`: it requires `torch`, and this causes import issues for TF-only testing
# We will install `accelerate@main` in Past CI workflow file
RUN python3 -m pip uninstall -y accelerate

# Uninstall `torch-tensorrt` and `apex` shipped with the base image
RUN python3 -m pip uninstall -y torch-tensorrt apex

# Pre-build **nightly** release of DeepSpeed, so it would be ready for testing (otherwise, the 1st deepspeed test will timeout)
RUN python3 -m pip uninstall -y deepspeed
# This has to be run inside the GPU VMs running the tests. (So far, it fails here due to GPU checks during compilation.)
# Issue: https://github.com/microsoft/DeepSpeed/issues/2010
# RUN git clone https://github.com/microsoft/DeepSpeed && cd DeepSpeed && rm -rf build && \
#    DS_BUILD_CPU_ADAM=1 DS_BUILD_FUSED_ADAM=1 DS_BUILD_UTILS=1 python3 -m pip install . --global-option="build_ext" --global-option="-j8" --no-cache -v --disable-pip-version-check 2>&1

RUN python3 -m pip install -U "itsdangerous<2.1.0"

# When installing in editable mode, `transformers` is not recognized as a package.
# this line must be added in order for python to be aware of transformers.
RUN cd transformers && python3 setup.py develop
