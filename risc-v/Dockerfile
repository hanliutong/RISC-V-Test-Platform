# 使用 ubuntu:22.04 作为基础镜像，用于编译 riscv-gnu-toolchain
FROM ubuntu:22.04 as builder

# 设置环境变量，构建镜像过程中所有交互操作都将自动忽略
ENV DEBIAN_FRONTEND=noninteractive

# 安装构建工具和依赖项
RUN apt-get update && apt-get install -y \
    autoconf automake autotools-dev curl python3 python3-pip libmpc-dev libmpfr-dev libgmp-dev gawk build-essential \
    flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev git ninja-build cmake libglib2.0-dev libslirp-dev && \
    rm -rf /var/lib/apt/lists/*

# 定义构建参数 COLLAB_TAG，默认值为 2024.04.12。这个参数将在后续的 git clone 命令中使用，用于指定 RISC-V GNU 工具链的版本
ARG COLLAB_TAG=2024.04.12

# 克隆 RISC-V GNU 工具链仓库
RUN git clone \
    --depth=1 \
    -b ${COLLAB_TAG} \
    https://github.com/riscv-collab/riscv-gnu-toolchain.git

# 设置工作目录为 lriscv-gnu-toolchain，之后的命令将在该目录下执行
WORKDIR riscv-gnu-toolchain

# 初始化并更新子模块
# 使用 GCC_TAG 参数来指定 GCC 的特定版本（默认为 releases/gcc-14.1.0）
RUN git submodule update --init gcc
ARG GCC_TAG=releases/gcc-14.1.0
RUN git -C gcc remote update && git -C gcc fetch origin ${GCC_TAG} && git -C gcc checkout ${GCC_TAG}

# 配置并构建 RISC-V 工具链
RUN ./configure --prefix=/opt/riscv 
RUN make linux

# 基于 ubuntu:22.04 镜像创建一个较轻的最终镜像，仅包含 riscv-gnu-toolchain 和其依赖项
FROM ubuntu:22.04

# # 设置环境变量，以便可以直接运行 riscv64-unknown-linux-gnu-gcc 等命令
ENV RISCV=/opt/riscv 
ENV PATH=$RISCV/bin:$PATH

# 将 builder 阶段中编译好的文件复制到最终的镜像
COPY --from=builder /opt/riscv ${RISCV}

# 安装依赖项，jdk 是容器作为代理连接 Jenkins 所必备的
RUN apt-get update && \
    apt-get install -y cmake ninja-build make git openjdk-17-jdk-headless && \
    rm -rf /var/lib/apt/lists/*

# 定义数据卷，工具链所在的目录 /opt/riscv，可以挂载至主机中
VOLUME ${RISCV}


