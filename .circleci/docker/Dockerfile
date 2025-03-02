# This requires cuda & cudnn packages pre-installed in the base image.
# Other available cuda images are listed at https://hub.docker.com/r/nvidia/cuda
ARG base_image="nvidia/cuda:11.7.0-cudnn8-devel-ubuntu18.04"
FROM "${base_image}"

ARG python_version="3.8"
ARG cuda="1"
ARG cuda_compute="5.2,7.5"
ARG cc="clang-8"
ARG cxx="clang++-8"
ARG cxx_abi="1"
ARG tpuvm=""

# Disable CUDA for PyTorch
ENV USE_CUDA "0"

# Enable CUDA for XLA
ENV XLA_CUDA "${cuda}"
ENV TF_CUDA_COMPUTE_CAPABILITIES "${cuda_compute}"
ENV TF_CUDA_PATHS "/usr/local/cuda,/usr/include,/usr"

# CUDA build guidance
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,utility
ENV PATH /usr/local/cuda:$PATH
ENV LD_LIBRARY_PATH /usr/local/cuda/lib64:$LD_LIBRARY_PATH

# AWS specific CUDA build guidance
ENV TORCH_CUDA_ARCH_LIST Maxwell
ENV TORCH_NVCC_FLAGS "-Xfatbin -compress-all"
ENV CUDA_PATH /usr/local/cuda

# Set C/C++ compilers
ENV CC "${cc}"
ENV CXX "${cxx}"

# Whether to build torch and torch_xla libraries with CXX ABI
ENV _GLIBCXX_USE_CXX11_ABI "${cxx_abi}"
ENV CFLAGS "${CFLAGS} -D_GLIBCXX_USE_CXX11_ABI=${cxx_abi}"
ENV CXXFLAGS "${CXXFLAGS} -D_GLIBCXX_USE_CXX11_ABI=${cxx_abi}"

# Whether to build for TPUVM mode
ENV TPUVM_MODE "${tpuvm}"

# Rotate nvidia repo public key (last updated: 04/27/2022)
# Unfortunately, nvidia/cuda image is shipped with invalid public key
RUN apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/3bf863cc.pub

# Install base system packages
RUN apt-get clean && apt-get update
RUN apt-get upgrade -y
RUN apt-get install --fix-missing -y python-pip python3-pip git curl libopenblas-dev vim jq \
  apt-transport-https ca-certificates procps openssl sudo wget libssl-dev libc6-dbg

# Install clang & llvm
ADD ./install_llvm_clang.sh install_llvm_clang.sh
RUN bash ./install_llvm_clang.sh

# Install valgrind
ADD ./install_valgrind.sh install_valgrind.sh
RUN bash ./install_valgrind.sh

# Sets up jenkins user.
RUN useradd jenkins && \
    mkdir /home/jenkins && \
    chown jenkins /home/jenkins
RUN echo 'jenkins ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

RUN mkdir -p /opt/conda /opt/cargo /opt/rustup /workspace /var/lib/jenkins && \
    chown jenkins /opt/conda /opt/cargo /opt/rustup /workspace /var/lib/jenkins
USER jenkins
WORKDIR /workspace

# Install openmpi for CUDA
run sudo apt-get install -y ssh
run sudo apt-get install -y --allow-downgrades --allow-change-held-packages openmpi-bin libopenmpi-dev

# Builds and configure sccache
ENV OPENSSL_INCLUDE_DIR /usr/include/openssl
ENV OPENSSL_LIB_DIR /usr/lib/x86_64-linux-gnu

ENV CARGO_HOME /opt/cargo
ENV RUSTUP_HOME /opt/rustup

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
RUN . $CARGO_HOME/env && \
    git clone --recursive https://github.com/pytorch/sccache.git && \
    cd sccache && \
    cargo install --path . && \
    cd .. && \
    rm -rf sccache

ENV PATH $CARGO_HOME/bin:$PATH

# Installs and configures Conda.
ADD ./install_conda.sh install_conda.sh
RUN sudo chown jenkins ./install_conda.sh
RUN bash ./install_conda.sh "${python_version}" /opt/conda

RUN echo "conda activate base" >> ~/.bashrc
RUN echo "export TF_CPP_LOG_THREAD_ID=1" >> ~/.bashrc
ENV PATH /opt/conda/bin:$PATH

RUN bash -c "source ~/.bashrc"
CMD ["bash"]
