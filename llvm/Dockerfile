# 使用 ubuntu:22.04 作为基础镜像，用于编译 LLVM
FROM ubuntu:22.04 AS builder

# 设置环境变量，构建镜像过程中所有交互操作都将自动忽略
ENV DEBIAN_FRONTEND=noninteractive 

# 安装构建工具和依赖项
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates gnupg ninja-build \
           build-essential cmake make python3 zlib1g wget subversion unzip git && \
    rm -rf /var/lib/apt/lists/*

# 定义构建参数 LLVM_TAG，用于克隆 LLVM 项目指定版本，默认 llvmorg-17.0.6
ARG LLVM_TAG=llvmorg-17.0.6

# 通过 Git 克隆 LLVM 项目代码
RUN git clone \
    -b ${LLVM_TAG} \
    --depth=1 \
    --recurse-submodules \
    --shallow-submodules \
    https://github.com/llvm/llvm-project

# 设置工作目录为 llvm-project，之后的命令将在该目录下执行
WORKDIR llvm-project

# 配置 LLVM 项目的构建过程，指定安装路径为 /opt/llvm
RUN cmake -G Ninja \
    -S llvm \
    -B build \
    -DCMAKE_INSTALL_PREFIX=/opt/llvm \
    -DLLVM_ENABLE_PROJECTS="clang" \
    -DCMAKE_BUILD_TYPE=Release \
    -DLLVM_TARGETS_TO_BUILD="X86;ARM;RISCV"
RUN cmake --build build --target install -- -j2


# 基于 ubuntu:22.04 镜像创建一个较轻的最终镜像，仅包含 LLVM 和其依赖项
FROM ubuntu:22.04

# 设置环境变量，以便可以直接运行 clang 等命令
ENV LLVM=/opt/llvm 
ENV PATH=${LLVM}/bin:$PATH

# 将 builder 阶段中编译好的 LLVM 文件复制到最终的镜像
COPY --from=builder /opt/llvm ${LLVM}

# 安装一些必要的依赖项，jdk 是容器作为代理连接 Jenkins 所必备的
RUN apt-get update && \
    apt-get install -y  --no-install-recommends libstdc++-10-dev binutils\
            cmake ninja-build make git openjdk-17-jdk-headless && \
    rm -rf /var/lib/apt/lists/*
