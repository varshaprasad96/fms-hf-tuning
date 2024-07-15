# Global args, using sclorg image for depedency and final image build for now ################
ARG TAG=c9s
ARG IMAGE_VERSION=python-39-c9s
ARG WHEEL_VERSION=""
ARG PYTHON_VERSION="3.9"

ARG USER=tuning
ARG USER_UID=1000

# Base image for dependency installations #####################################################
FROM quay.io/sclorg/${IMAGE_VERSION}:${TAG} AS python-installations

ARG WHEEL_VERSION
ARG USER
ARG USER_UID

USER root

RUN useradd -u $USER_UID ${USER} -m -g 0 --system && \
    chmod g+rx /home/${USER}

# Install Git and clean up
RUN dnf install -y git && \
    rm -f /usr/share/doc/perl-Net-SSLeay/examples/server_key.pem && \
    dnf clean all

# Switch to the specified user and set up the working directory
USER ${USER}
WORKDIR /tmp

# Install Python build tool and dependencies for flash-attention.
RUN --mount=type=cache,target=/home/${USER}/.cache/pip,uid=${USER_UID} \
    python -m pip install --upgrade pip wheel build transformers accelerate matplotlib packaging "setuptools>=60"

# Clone and install Flash Attention with ROCm support
# Refer https://rocm.blogs.amd.com/artificial-intelligence/flash-attention/README.html#getting-started
# TODO:
# For setup.py from flash-attention to be installed successfully it needs to be able to recognize
# amd GPUs.
# A local run indicates from https://github.com/ROCm/flash-attention/blob/flash_attention_for_rocm/setup.py indicates
# cuda env vars need to be set. Verify if that's still the case, after AMD_GPU env is recognized.
# Refer: https://github.com/ROCm/flash-attention/blob/flash_attention_for_rocm/Dockerfile.rocm

#RUN git clone --recursive https://github.com/ROCm/flash-attention.git && \
#    cd flash-attention && \
#    ## Debug: The script errors with setup not available till run on a platform with AMD GPU.
#    python -m pip show setuptools && \
#    MAX_JOBS=$((`nproc` - 1)) python -m pip install -U --no-build-isolation -v .

COPY --chown=${USER}:root tuning tuning
COPY .git .git
COPY pyproject.toml pyproject.toml

# Build a wheel if PyPi wheel_version is empty else download the wheel from PyPi
RUN if [[ -z "${WHEEL_VERSION}" ]]; \
    then python -m build --wheel --outdir /tmp; \
    else pip download fms-hf-tuning==${WHEEL_VERSION} --dest /tmp --only-binary=:all: --no-deps; \
    fi && \
    ls /tmp/*.whl >/tmp/bdist_name

RUN --mount=type=cache,target=/home/${USER}/.cache/pip,uid=${USER_UID} \
    python -m pip install wheel && \
    python -m pip install "$(head bdist_name)" && \
    python -m pip uninstall wheel build -y && \
    # Cleanup the bdist whl file
    rm $(head bdist_name) /tmp/bdist_name

## Final image ################################################
FROM quay.io/sclorg/${IMAGE_VERSION}:${TAG} as release

ARG USER
ARG USER_UID
ARG PYTHON_VERSION

# Use root priviledges to add a new user, but restore permissions to
# system user in the end.
USER root

# Add a new system user
RUN useradd -u $USER_UID ${USER} -m -g 0 --system && \
    chmod g+rx /home/${USER} && \
    mkdir -p /licenses \

COPY LICENSE /licenses/

# Copy installed Python packages from the intermediate stage
COPY --from=python-installations /opt/app-root/lib/python${PYTHON_VERSION}/site-packages /opt/app-root/lib/python${PYTHON_VERSION}/site-packages

# Ensure the created user has access to the libraries
RUN chown -R ${USER}:0 /opt/app-root/lib/python${PYTHON_VERSION}/site-packages

# Set the PYTHONPATH environment variable
ENV PYTHONPATH="/opt/app-root/lib/python${PYTHON_VERSION}/site-packages"

RUN mkdir /app && \
    chown -R $USER:0 /app /tmp && \
    chmod -R g+rwX /app /tmp

# Set fms specific env vars.
ENV FSDP_DEFAULTS_FILE_PATH="/app/accelerate_fsdp_defaults.yaml"
ENV SET_NUM_PROCESSES_TO_NUM_GPUS="True"

## Copy scripts and default configs
COPY build/accelerate_launch.py fixtures/accelerate_fsdp_defaults.yaml /app/
COPY build/utils.py /app/build/
#RUN chmod +x /app/accelerate_launch.py

########## Install ROCm packages and verify installations. ##################
WORKDIR /opt/app-root/bin
ARG ROCM_VERSION=6.1
ARG AMDGPU_VERSION=6.1

# Base
RUN yum -y install git java-1.8.0-openjdk python; yum clean all

# Install ROCm AMD from:
# https://github.com/ROCm/ROCm-docker/blob/master/dev/Dockerfile-centos-7-complete
# Enable epel-release repositories
RUN yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm && \
    yum config-manager --set-enabled crb && \
    yum clean all && rm -rf /var/cache/yum

# Install required base build and packaging commands for ROCm
RUN yum -y install \
    bc \
    bridge-utils \
    glibc.i686 \
    numactl-libs \
    libssh \
    libunwind-devel \
    libunwind \
    pciutils \
    pciutils-libs \
    re2c \
    doxygen \
    elfutils-libelf-devel \
    expect \
    numactl-devel \
    pciutils-devel \
    qemu-kvm \
    subversion \
    dkms \
    dpkg \
    dpkg-dev \
    dpkg-perl && \
    yum clean all

# Enable the epel repository for fakeroot
# RUN yum --enablerepo=extras install -y fakeroot
# RUN yum clean all
RUN yum install -y fakeroot && \
    yum clean all

# Install the ROCm rpms
RUN echo "[ROCm]" > /etc/yum.repos.d/rocm.repo && \
    echo "name=ROCm" >> /etc/yum.repos.d/rocm.repo && \
    echo "baseurl=https://repo.radeon.com/rocm/el9/$ROCM_VERSION/main" >> /etc/yum.repos.d/rocm.repo && \
    echo "enabled=1" >> /etc/yum.repos.d/rocm.repo && \
    echo "gpgcheck=0" >> /etc/yum.repos.d/rocm.repo

RUN echo "[amdgpu]" > /etc/yum.repos.d/amdgpu.repo && \
    echo "name=amdgpu" >> /etc/yum.repos.d/amdgpu.repo && \
    echo "baseurl=https://repo.radeon.com/amdgpu/$AMDGPU_VERSION/el/9.2/main/x86_64" >> /etc/yum.repos.d/amdgpu.repo && \
    echo "enabled=1" >> /etc/yum.repos.d/amdgpu.repo && \
    echo "gpgcheck=0" >> /etc/yum.repos.d/amdgpu.repo

# Install rocm and amdgpu binaries
RUN yum install -y amdgpu-dkms rocm  && \
    yum clean all

RUN  tee --append /etc/ld.so.conf.d/rocm.conf <<EOF
/opt/rocm/lib
/opt/rocm/lib64
EOF

# TODO: For verification, takes in unccessary cycles! To remove!
RUN ldconfig

ENV PATH=$PATH:/opt/rocm-6.1.2/bin

# Need a better way to address this hack
RUN touch /.aim_profile && \
    chmod -R 777 /.aim_profile && \
    mkdir /.cache && \
    chmod -R 777 /.cache

WORKDIR /app

# Restore user
USER ${USER}

CMD [ "python", "/app/accelerate_launch.py" ]