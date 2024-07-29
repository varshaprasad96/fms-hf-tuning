ARG BASE_IMAGE="registry.access.redhat.com/ubi9/ubi:latest"
ARG USER=tuning
ARG USER_UID=1000

################## Base image for roc-m ####################
FROM ${BASE_IMAGE} as rocm-base

ARG USER
ARG USER_UID

RUN useradd -u $USER_UID ${USER} -m -g 0 --system && \
    chmod g+rx /home/${USER}

RUN echo '[amdgpu]' > /etc/yum.repos.d/amdgpu.repo && \
    echo 'name=amdgpu' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'baseurl=https://repo.radeon.com/amdgpu/6.1.2/rhel/9.4/main/x86_64/' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'enabled=1' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'priority=50' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'gpgcheck=1' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'gpgkey=https://repo.radeon.com/rocm/rocm.gpg.key' >> /etc/yum.repos.d/amdgpu.repo && \
    echo '' >> /etc/yum.repos.d/amdgpu.repo && \
    echo '[ROCm-6.1.2]' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'name=ROCm6.1.2' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'baseurl=https://repo.radeon.com/rocm/rhel9/6.1.2/main' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'enabled=1' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'priority=50' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'gpgcheck=1' >> /etc/yum.repos.d/amdgpu.repo && \
    echo 'gpgkey=https://repo.radeon.com/rocm/rocm.gpg.key' >> /etc/yum.repos.d/amdgpu.repo


RUN yum -y install rocm hipcc && yum clean all

###################### Build pytorch and flash-attention ###################

FROM rocm-base as python-base

# Using the default gfx arch are specified here:
# https://docs.vllm.ai/en/latest/getting_started/amd-installation.html#option-1-build-from-source-with-docker-recommended
ARG FA_GFX_ARCHS="gfx90a;gfx942"

# Install python deps
RUN yum -y install python3 python3-pip python3-devel patch

# Install deps for pytorch
RUN yum -y install git bzip2 libX11-devel wget unzip

RUN python3 -m pip install --upgrade pip
RUN python3 -m pip install --no-cache-dir fastapi ninja tokenizers pandas wheel setuptools
RUN python3 -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.0

ENV PATH=$PATH:/opt/rocm/bin:/libtorch/bin:
# Adding this env vars in since rocm is specified instead of CUDA.
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/rocm/lib/:/libtorch/lib:
ENV CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/libtorch/include:/libtorch/include/torch/csrc/api/include/:/opt/rocm/include/:

ENV BUILD_TARGET='rocm'

RUN mkdir libs \
    && cd libs \
    && git clone https://github.com/ROCm/flash-attention.git \
    && cd flash-attention \
    && git submodule update --init \
    && export GPU_ARCHS=${FA_GFX_ARCHS} \
    && python3 setup.py install \
    && cd ..;
######################### Installations for fms ##################

FROM python-base as python-installations

ARG WHEEL_VERSION
ARG USER
ARG USER_UID

USER ${USER}
WORKDIR /tmp
RUN --mount=type=cache,target=/home/${USER}/.cache/pip,uid=${USER_UID} \
    python3 -m pip install --user build
COPY --chown=${USER}:root tuning tuning
COPY .git .git
COPY pyproject.toml pyproject.toml

# Build a wheel if PyPi wheel_version is empty else download the wheel from PyPi
RUN if [[ -z "${WHEEL_VERSION}" ]]; \
    then python3 -m build --wheel --outdir /tmp; \
    else pip download fms-hf-tuning==${WHEEL_VERSION} --dest /tmp --only-binary=:all: --no-deps; \
    fi && \
    ls /tmp/*.whl >/tmp/bdist_name

# Install from the wheel
RUN --mount=type=cache,target=/home/${USER}/.cache/pip,uid=${USER_UID} \
    python3 -m pip install --user wheel && \
    python3 -m pip install --user "$(head bdist_name)" && \
    # Cleanup the bdist whl file
    rm $(head bdist_name) /tmp/bdist_name

######################### Final image ##################

FROM python-installations as release

ARG USER

RUN mkdir -p /licenses
COPY LICENSE /licenses/

RUN mkdir /app && \
    chown -R $USER:0 /app /tmp && \
    chmod -R g+rwX /app /tmp

# Copy scripts and default configs
# TODO: Make sure you are running from the right context to be able to copy these.
COPY build/accelerate_launch.py fixtures/accelerate_fsdp_defaults.yaml /app/
COPY build/utils.py /app/build/
RUN chmod +x /app/accelerate_launch.py

ENV FSDP_DEFAULTS_FILE_PATH="/app/accelerate_fsdp_defaults.yaml"
ENV SET_NUM_PROCESSES_TO_NUM_GPUS="True"

# Need a better way to address this hack
RUN touch /.aim_profile && \
    chmod -R 777 /.aim_profile && \
    mkdir /.cache && \
    chmod -R 777 /.cache \

WORKDIR /app
USER ${USER}
COPY --from=python-installations /home/${USER}/.local /home/${USER}/.local

#CMD [ "python", "/app/accelerate_launch.py" ]