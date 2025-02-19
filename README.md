# 整体说明

测试平台主要由 Jenkins 主节点和 Jenkins 从节点组成：

- Jenkins 主节点，即 Jenkins 服务器，负责管理所有配置、插件、用户权限，以及任务的调度、控制等

- Jenkins 从节点，即工作代理，可以是虚拟机、物理机或容器。从节点通过 SSH 或 JNLP 等方式连接到主节点，并由主节点指挥执行实际的构建任务

使用 Docker 容器安装 Jenkins 主节点，可以减少运维的复杂性，具有快速部署、易于扩展、资源隔离、简化依赖管理等优点

本平台主要面向 RISC-V 设备，测试任务需要完成针对 RISC-V 的交叉编译，并将编译产物发送到 RISC-V 设备上执行。因此需要配置的从节点包括：

1. 编译节点，负责完成交叉编译工作。为了简化配置、环境隔离以及更好的版本管理，将 GCC、LLVM 等编译工具做成 Docker 镜像，将其配置为动态节点（Cloud Node）
2. RISC-V 设备节点，负责测试编译产物。这些设备长期运行、环境稳定，配置为静态节点

假设 Jenkins 主节点与编译节点的容器运行在同一台 x86 服务器上，整个平台结构如下：

​	![面向RISC-V设备的自动化测试与性能评估平台](image\面向RISC-V设备的自动化测试与性能评估平台.png)

下面介绍整个平台的安装配置。第一节从制作镜像开始，搭建出一个基础的 Jenkins；第二节将 RISC-V 设备作为从节点连接到 Jenkins 中；第三节将编译工具做成容器，并设置为 Jenkins 的动态节点

测试用例一节，以 [OpenCV](https://github.com/opencv/opencv) 面向 RISC-V 平台的交叉编译为例，测试平台的可用性。整个测试从拉取不同分支代码、到对分支分别进行编译、最终将编译产物发送到 RISC-V 设备上执行，完成了面向 RISC-V 设备的自动化测试工作。在这一节详细介绍了测试所用的 Jenkinsfile，可作为同类型测试的参考

------



[TOC]

------



# Jenkins 主节点容器

Jenkins 主节点主要用于管理、调度和配置，是 Jenkins 系统的核心。下面先介绍 Jenkins 容器的镜像制作，然后介绍如何启动容器，最终可以访问 Jenkins web 页面。此外还给出了备用镜像，以进行快速搭建



## 制作镜像

Docker 的安装与配置可以参考[Docker — 从入门到实践](https://yeasy.gitbook.io/docker_practice/install)。如制作镜像时由于网络问题而无法拉取基础镜像，可以参考[这里](https://www.bilibili.com/read/cv35387254/)解决。假设服务器已经可以使用 Docker

[Jenkins 目录](jenkins) 中包含一个用于制作镜像的 Dockerfile 和制作镜像过程中使用到的插件列表 plugins.txt。使用下列命令构建镜像：

```bash
docker build -t buddy-jenkins .
```

得到一个名为 buddy-jenkins 的镜像，可以使用 `docker image ls` 命令查看：

![image-20240916131605821](image\image-20240916131605821.png) 

制作出的镜像主要在 Jenkins 官方镜像的基础上添加了一些插件，详细信息可以参考官方镜像的 [README.md](https://github.com/jenkinsci/docker/blob/master/README.md)



## 备用镜像

在 [docker](docke) 目录下有所有使用到的镜像，包括 Jenkins 镜像以及 llvm 等编译工具镜像，可以直接下载到本地。下载后使用下列命令解压：

```bash
docker load -i <path_to_save>/<image_name>.tar
```

若需保存自己制作的容器，可以使用下列命令压缩：

```bash
docker save -o <path_to_save>/<image_name>.tar <image_name>:<tag>
```



## 启动容器

使用下列命令启动 Jenkins 容器：

```bash
docker run -d -p 8080:8080 -p 5000:5000 -v jenkins_home:/var/jenkins_home --restart=on-failure --name jenkins buddy-jenkins
```

- `-d` ：代表 detached mode，Docker 会将容器放在后台运行，并返回容器 ID

- `-p 8080:8080 -p 5000:5000`：将容器的 8080 端口和 5000 端口映射到宿主机的 8080 端口和 5000 端口

  - Jenkins 的 8080 端口是 Jenkins 的 Web 管理界面端口，映射后通过宿主机的 `http://localhost:8080`，可以访问运行在容器中的 Jenkins 的 Web 界面
  - Jenkins 的 5000 端口是从节点代理端口（Jenkins Slave Agent），用于主节点与从节点之间的通信

- `-v jenkins_home:/var/jenkins_home`：`-v` 选项用于挂载数据卷，这里将容器内部的 `/var/jenkins_home` 目录挂载到数据卷 `jenkins_home` 中

  - `/var/jenkins_home` 是 Jenkins 的默认数据存储路径，保存了 Jenkins 的配置文件、插件、工作区、构建历史等所有数据。通过挂载这个卷，即使容器被删除或重启，Jenkins 的数据仍然存储在宿主机中

  - 在主机中使用下列命令可以查看数据卷信息：

    ```bash
    docker volume inspect jenkins_home
    ```

    可以看到数据卷位于宿主机的 `/var/lib/docker/volumes/jenkins_home/_data` 目录下，进入这个目录查看 Jenkins 数据，效果等同于进入 Jenkins 容器中的 `/var/jenkins_home` 目录

    ![image-20240916150713478](image\image-20240916150713478.png)

- `--restart`：定义容器的重启策略。`on-failure` 表示容器只有在由于某种错误而崩溃（非正常退出）时，才会自动重启

- `--name` ：为容器指定一个名字，这里命名为 `jenkins`

  - 可以通过容器名称来进行后续操作，而不需要使用容器 ID。例如使用 `docker exec` 进入容器：

    ```bash
    docker exec -it jenkins /bin/bash
    ```

容器启动后，访问宿主机 `http://localhost:8080` 查看 Jenkins web 页面：

![image-20240916103248312](image\image-20240916103248312.png)

这是一个空白的 Jenkins 服务器，接下来将添加从节点，完善平台功能



------



# RISC-V 设备节点

RISC-V 设备作为远程工作代理，执行测试任务。配置 Jenkins 从节点需要 RISC-V 设备安装 java 环境以及 ssh，假设所有设备已安装完成。下面以香蕉派开发板 BPI-F3 为例子，说明手动配置流程



## 手动配置

BPI-F3 基本信息如下：

| 名称   | 架构 / 操作系统              | ip             | 登录账号 | 密码 |
| ------ | ---------------------------- | -------------- | -------- | ---- |
| BPI-F3 | RISC-V RVV / Bianbu (debian) | 192.168.15.164 | k1       | 1    |

### 1. 创建凭证

在 Jenkins 面板中依次点击 `Dashboard > Manage Jenkins > Credentials > system > Global credentials (unrestricted) > add Credential` 出现创建凭证页面，如下所示：

![Untitled](image\Untitled.png)

凭证用于SSH连接，可以是用户名和密码、SSH密钥等，这里使用用户名加密码的方式。在 `Kind` 下选中 `Username with password”`，填入可以登录 BPI-F3 的账号和密码

### 2. 创建节点

在 Jenkins 面板中依次点击 `Dashboard > Manage Jenkins > Nodes > New node` 创建节点，如下所示：

![image-20240917111016457](image\image-20240917111016457.png)

填入节点名称，并勾选 `Permanent Agent`，进入节点配置页面：

![image-20240917111757976](image\image-20240917111757976.png)

- `Number of execution`：表示节点上可以同时运行的并行作业数量，常见设置为2
- `Remote root directory`：指定 Jenkins 的工作目录，即 Jenkins在此节点上存储构建工作区和其他文件的目录。**设置时注意该目录的权限，确保 Jenkins 对其可写可执行**
- `Lables`：用于标识节点，可以在作业配置中指定使用某个特定标签的节点来执行作业
- `Usage`：确定 Jenkins 如何使用此节点，按需可选：
  - `Utilize this node as much as possible`：尽可能多地在此节点上运行作业
  - `Only build jobs with label expressions matching this node`：仅在指定了匹配此节点标签的作业时使用此节点
- `Launch method`：选择如何启动节点，这里选择 **Launch agent via SSH**
  - `Host`：填入SSH主机名
  - `Credentials`：选择上一步配置成功的凭据

![Untitled (3)](image\Untitled (3).png)

- `Host Key Verification Strategy`：配置SSH连接的主机密钥验证策略，这里选择 `Non-verifying Verification Strategy`（不进行验证）
- `Availability`：控制节点的可用性，按需可选
  - `Keep this agent online as much as possible`：节点尽可能保持在线状态
  - `Take this agent online when in demand and take offline when idle`：当有需要时节点上线，当空闲时节点下线
  - `Take this agent offline when computer is idle`：当计算机空闲时节点下线

### 3. 验证

配置完成后，可以在节点的 log 页面查看是否连接成功

![无标题4](image\无标题4.png)



------



# 编译工具节点

将不同版本的 GCC、LLVM 做成容器镜像后，项目可以指定不同的容器来使用不同的编译工具版本，易于版本管理。下面先介绍编译工具的镜像制作，然后介绍在 Jenkins 中手动配置Docker 动态节点的流程



## 制作镜像

下面介绍 llvm 和 riscv-gnu-toolchain 的镜像构建，也可以下载已经构建完成的镜像，具体参见 [备用镜像](##备用镜像) 一节

### llvm

制作 llvm 镜像的 Dockerfile 保存在 [llvm 目录](llvm) 下，使用下列命令构建镜像：

```bash
docker build -t llvm:18 --build-arg LLVM_TAG=llvmorg-18.1.7 .
```

- `-t llvm:18`：指定了构建的镜像标签为 `llvm:18`。镜像标签（tag）有助于对不同版本的镜像进行管理
- `--build-arg` ：在构建时传递构建参数
  - `LLVM_TAG=llvmorg-18.1.7`：设置 `LLVM_TAG` 构建参数， Dockerfile 在执行 `git clone` 时，克隆 LLVM 项目的 `llvmorg-18.1.7` 版本

使用下面的命令即可构建 llvm 17 版本：

```bash
docker build -t llvm:17 LLVM_TAG=llvmorg-17.0.6 .
```

参数 `LLVM_TAG` 的值是 [llvm 仓库 ](https://github.com/llvm/llvm-project)代码的 tag 值，可以自行构建其他版本

### riscv-gnu-toolchain

制作 riscv-gnu-toolchain 的镜像的 Dockerfile 保存在 [riscv 目录](riscv) 下，使用下列命令构建镜像：

```bash
docker build -t riscv-gnu-toolchain:13.3.0 --build-arg GCC_TAG=releases/gcc-13.3.0 .
```

- `-t riscv-gnu-toolchain:13.3.0`：指定构建镜像的标签
- `--build-arg`：构建时传递构建参数
  - `GCC_TAG=releases/gcc-13.3.0`：设置 `GCC_TAG` 构建参数， Dockerfile 在执行 `git clone` 时，构建的 gcc 版本为 `gcc-13.3.0`

使用下面的命令即可构建 gcc 14 版本：

```
docker build -t riscv-gnu-toolchain:14.1.0 --build-arg GCC_TAG=releases/gcc-14.1.0 .
```

参数 `GCC_TAG` 的值是 [riscv-gnu-toolchain 仓库 ](https://github.com/riscv-collab/riscv-gnu-toolchain)代码的 tag 值，可以自行构建其他版本



## 添加节点

1. 在 Jenkins 面板中依次点击 `Dashboard > Manage Jenkins > Clouds > New cloud` 出现创建动态节点页面，如下所示。填写名称，并选择节点类型为 Docker：

![image-20240916203211943](image\image-20240916203211943.png)

2. 点击 `Docker Cloud details` 填写节点地址，由于 Jenkins 容器与编译工具容器都运行在同一台 x86 服务器上，这里填写的 `Docker Host URI` 就是这台宿主机的地址，点击右下方 `Test Connection` 可以测试是否连通：

![image-20240917075818694](image\image-20240917075818694.png)

注意，为了使 Jenkins 容器可以通过网络与宿主机上的 Docker 守护进程进行通信，需要让 Docker 守护进程（docker.service）监听一个 TCP 端口。修改 `/usr/lib/systemd/system/docker.service` 配置文件，在文件中的 `ExecStart` 行后添加 `-H tcp://0.0.0.0:2375`，然后重载 systemd 配置并重启 Docker 服务

```bash
systemctl daemon-reload
systemctl restart docker.service
```

修改后在上述配置页面中 `Docker Host URL` 填入 `tcp://172.17.0.1:2375` 即可

点击保存后可以在 `Dashboard > Manage Jenkins > Docker` 中查看已经启动的容器和安装的镜像

![image-20240917150520693](image\image-20240917150520693.png)

3. 进入配置完成的 Docker 服务器（x86），依次点击 `Configure > Docker Agent templates > Add Node Property` 配置具体容器节点

- `Lables`、`Name`：用于标记节点，在后续测试中可以使用标签或名称指定任务运行的节点
- `Docker Image`：填写镜像名称或 id，Jenkins 会动态拉取这个镜像并创建容器
- `Enable`： 使容器可用

![无标题2](image\无标题2.png)

- `Instance Capacity` ：指该节点可以同时处理的最大并发构建数量。这里设为2，表示如果有两个并发的构建任务，Jenkins 将会启动两个容器来分别处理
- `Remote File System Root`：容器的工作空间所在的目录路径
- `Usage`：决定 Jenkins 如何使用该节点来执行构建任务

![image-20240917075931070](image\image-20240917075931070.png)

- `Pull strategy`：指在启动 Docker 容器时镜像的拉取策略。编译容器都已保存在宿主机（x86）上，因此选择 `Never pull`

![image-20240917075948860](image\image-20240917075948860.png)

- 点击 `Container settings`，进行进一步配置

![无标题](image\无标题.png)

- `Mounts`：将宿主机的文件或目录挂载到 Docker 容器内部，使得容器可以访问宿主机上的数据或资源。

![无标题5](image\无标题5.png)

- `/var/lib/docker/volumes/jenkins_home/_data` 是 Jenkins 容器在宿主机上的数据存储路径，详情参见 [Jenkins-启动容器一节](##启动容器)。这里将其映射到 llvm 容器的 `/home/jenkins` 下。这个目录应与 `Remote File System Root` 中设置的目录一致，容器就能直接使用宿主机中存储的数据，同时直接将构建产物保存到宿主机中

- 使用 llvm 交叉编译时需要用到 risc-v 工具链，可以借用 riscv-gnu-toolchain 容器中的已经编译完成的工具链。使用下列命令在宿主机上先启动一次 riscv-gnu-toolchain 容器，将容器的工具链保存到数据卷 riscv_tool 中

  ```bash
  docker run -it --name riscv-14 -v --rm riscv_tool:/opt/riscv riscv-gnu-toolchain:14.1.0
  ```

  `/var/lib/docker/volumes/riscv_tool/_data` 是这个数据卷在宿主机上的数据存储路径（可以使用 `docker volume inspect riscv_tool` 查看），llvm 将其映射到了自己的 `/opt/riscv` 目录下，在容器中进行交叉编译时可以指定这个目录为工具链的存储位置

riscv-gnu-toolchain 容器的配置类似，在 `Mounts` 处只需映射 Jenkins 容器的数据卷即可

编译容器节点的基本配置如上，实际使用中如需更多配置可以参考 [Docker | Jenkins plugin](https://plugins.jenkins.io/docker-plugin/)



------



# 测试用例

以 [OpenCV](https://github.com/opencv/opencv) 面向 RISC-V 平台的交叉编译为例，对比使用同一工具链构建下的两个不同分支代码在 RISC-V 开发板上的执行表现。支持用户选择仓库的不同分支、构建工具、测试文件以及负责执行的 RISC-V 设备。下面先介绍 OpenCV 的手动测试流程，随后给出自动测试任务在 Jenkins 中的配置以及测试效果，最后详细解释了自动测试任务中使用到的 jenkinsfile，这个文件位于 [opencv_test](opencv_test) 目录下

## OpenCV 手动测试

OpenCV是一个被广泛应用的开源计算机视觉算法库，支持 x86, ARM, RISC-V 平台的本机编译和交叉编译，有较为成熟的性能评估方案（基于Google Test）和大量的测试用例，能够生成格式化的性能数据。为了更好理解测试工作，下面先介绍 OpenCV 面向 RISC-V 平台的手动交叉编译过程，[参考链接]([OpenCV RISC V · opencv/opencv Wiki · GitHub](https://github.com/opencv/opencv/wiki/OpenCV-RISC-V))

1. 交叉编译

   利用 clang 编译器交叉编译时，通过 `-DRISCV_CLANG_BUILD_ROOT=` 指定编译器,  `-DRISCV_GCC_INSTALL_ROOT` 指定交叉编译工具链。下列命令行指令供参考，假设clang在`/opt/llvm`, 工具链在`/opt/riscv`

   ```
               cmake -G Ninja ../ \\
                   -DRISCV_CLANG_BUILD_ROOT=/opt/llvm \\
                   -DRISCV_GCC_INSTALL_ROOT=/opt/riscv \\
                   -DCMAKE_BUILD_TYPE=Release \\
                   -DBUILD_SHARED_LIBS=OFF \\
                   -DBUILD_EXAMPLES=OFF \\
                   -DWITH_PNG=OFF \\
                   -DOPENCV_ENABLE_NONFREE=ON \\
                   -DWITH_OPENCL=OFF \\
                   -DCMAKE_TOOLCHAIN_FILE=../platforms/linux/riscv64-clang.toolchain.cmake \\
                   -DCPU_BASELINE=RVV \\
                   -DCPU_BASELINE_REQUIRE=RVV \\
                   -DRISCV_RVV_SCALABLE=ON
   ```
   
2. 测试

   性能测试用例会被构建于 `<OpenCV构建目录>/bin` ，以 `opencv_perf_<module>` 命名，如核心模块的性能测试用例为 `opencv_perf_core`。

   指定 `--gtest_output=xml:./result.xml` 参数可以获得 xml 格式的性能数据。可以通过  `python <opencv_dir>/modules/ts/misc/summary.py result1.xml result2.xml` 来对比两个（或多个）性能数据。[参考链接](https://github.com/opencv/opencv/wiki/HowToUsePerfTests)


   ```bash
   ./opencv_perf_core --gtest_filter="BinaryOpTest.transposeND*" --gtest_output=xml:./result2.xml
   ```



## OpenCV 测试项目设置

点击 Jenkins 主页面 New Item，新建一个测试任务，填写任务名字，并选择 Pipeline 以创建一个流水线项目

![创建任务1](image\创建任务1.png)

将代码粘于下方 Script 处，勾选 Use Groovy SandBox

![创建任务2](image\创建任务2.png)

保存后点击左侧 Build Now 开始构建。带参数的项目配置完成后需要先构建一次， 第二次构建才会出现参数选择界面

在下方的参数构建页面中，

- BRANCH_1 与 BRANCH_2：选择 opencv 被构建的两个不同分支
- BUILD_NODE：选择构建工具，当前支持 llvm-18、llvm-17、riscv-13、riscv-14，构建工具的相关设置详见 [编译工具节点](#编译工具节点)
- ARCHIVE_FILES：选择被测试的构建产物，支持多选
- EXECUTION_NODES：选择执行测试文件的节点，支持多选。执行节点的相关设置详见 [RISC-V 设备节点](#RISC-V设备节点)

![image-20240914155037905](image\image-20240914155037905.png)

左侧上方可以对项目进行构建、配置、删除、重命名等操作，下方是构建历史，点进历史可以看到每次构建的详细信息

一次成功的构建会存档构建产物和 xml 格式的报告。例如，`a1_opencv_perf_core_BPI-F3.xml` 表示分支一构建出的测试文件 `opencv_perf_core` 在设备 BPI-F3 上的执行报告；目录 `build_4.x_riscv-14` 中会存放使用 riscv-14 编译的 4.x 分支的产物

![image-20240914160031734](image\image-20240914160031734.png)

点击 Console Output 可以看到完整的执行日志

![image-20240914160920732](image\image-20240914160920732.png)



## OpenCV 测试项目代码详解

Jenkinsfile 是用来定义 Jenkins 自动化流水线的脚本，描述了项目的构建、测试和部署过程，基于 Groovy 语法编写。下面详细介绍本次测试所使用的 Jenkinsfile



### Jenkinsfile 基本结构

Jenkinsfile 主要由 **Pipeline**、**Agent**、**Stages** 和 **Steps** 几个关键部分组成。下面给出了一个流水线的简单示例，进阶流水线语法知识可以参考 [官方流水线语法教程]([Pipeline Syntax (jenkins.io)](https://www.jenkins.io/doc/book/pipeline/syntax/))

```groovy
pipeline {
    agent none
    
    environment {
        REPO_URL = 'https://github.com/opencv/opencv.git' // 仓库地址
    }
	
	parameters {
        gitParameter branchFilter: 'origin/(.*)', defaultValue: '4.x', name: 'BRANCH', type: 'PT_BRANCH'
    }
	
    stages {
        stage('Checkout Branch') { // 阶段一：拉取代码
            steps {
				// 从Git仓库拉取
            	checkout([
                        $class: 'GitSCM',
                        branches: [[name: "${BARNCH}"]], // 拉取参数BRANCH所选定的分支
                        userRemoteConfigs: [[url: "${REPO_URL}"]] // 
                        extensions: [ cloneOption(shallow: true) ] // 浅克隆
                    ])
            }
        }
        
        stage('Build') { // 阶段二：构建项目
            steps {
            	sh 'echo "Building the project..."' // 使用 shell 命令构建项目
            }
        }
        
        stage('Archive') { // 阶段三：存档构建产物
            steps {
            	sh 'echo "Archiving ..."' 
            }
        }
        
        stage('Test') {// 阶段四：测试
            steps {
            	sh 'echo "Running tests..."' // 使用 shell 命令运行测试
            }
        }
    }
}
```

- **`pipeline {}`**：表示整个流水线，包含流水线所有的阶段和步骤

- **`agent any`**：`agent` 指定在哪个节点中运行流水线，可以是特定机器、Docker 容器等。`none` 则表示不分配任何节点，每个阶段的执行节点可能不同，可以在后面不同阶段中重新定义 `agent`

- **`environment`** ：用于定义环境变量。环境变量可以在流水线的各个阶段中使用，通常用来存储一些全局信息。这里定义了变量 `REPO_URL` ，用来存储仓库地址

- **`parameters`**：定义在执行流水线时，用户可以提供的输入值（参数），参数可以用来动态配置流水线的运行行为。这里的`gitParameter branchFilter` 定义了 `BRANCH` 参数，用于选择拉取的仓库分支

- **`stages {}`**：表示流水线的各个阶段，每个阶段对应一个步骤

- **`steps {}`**：每个阶段的具体操作，步骤可能是运行脚本、调用工具、执行命令等

- **`stage('Checkout Branch')`**：定义一个名为 `Check out` 的阶段，该阶段从 Git 仓库拉取代码

整个流水线分为拉取代码（Checkout Branch）、构建（Build）、存档（Archive）以及测试（Test）四个阶段，整个测试文件将在这个基础上补充完整



### 1. 拉取代码 Checkout

Jenkins 容器的数据保存在数据卷 jenkins_home 中，工具链容器将工作目录挂载到了相同位置，因此在 Jenkins 容器中拉取的代码，可以在工具链容器中访问到。因此拉取代码阶段可以直接使用 Jenkins 主节点作为工作节点，添加 `agnet` 块，通过标签 `jenkins-docker` 指定

```groovy
        stage('Checkout') { // 克隆所选分支
            agent {
                node {
                    label 'jenkins-docker' // Jenkins 控制器的内置节点
                }
            }
            
            steps {
                checkout([
                        $class: 'GitSCM',
                        branches: [[name: "${BRANCH}"]],  // BRANCH 参数供用户选择分支
                        userRemoteConfigs: [[url: "${REPO_URL}"]] // 
                        //可选：浅克隆
                        //extensions: [ cloneOption(shallow: true) ]
                    ])
            }
        }
```

使用 `Git Parameter` 插件定义 `BRANCH` 参数选择拉取的分支，同时定义了环境变量 `REPO_URL` 保存仓库地址。由于网络问题，可能构建前无法获取到分支列表，或者构建时拉取代码失败。可以预先将仓库克隆到服务器上，将 `REPO_URL` 替换为本地路径，规避网络问题并且节约构建时间

注意，当 URI 更换为本地路径时，`Git Parameter` 插件只能获取本地已保存的分支。可以在克隆仓库时添加 `--mirror` 选项，它会克隆仓库的完整镜像，包括所有分支、标签和配置

```bash
git clone --mirror <repository-url>
```

除了克隆所有分支，也可以仅将所需的分支提前保存到本地。以保存 OpenCV 5.x 分支为例，使用以下命令：

```bash
git checkout -b 5.x origin/5.x
git fetch origin
```

使用本地克隆存在一定的安全问题，因此需要在 Jenkins 中进行配置。在 Scritpt Console 中输入下列命令：

```groovy
hudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT = true
```

![image-20240915150901694](image\image-20240915150901694.png)



### 2. 构建 Build

拉取代码后使用编译工具进行构建，因此构建阶段选择编译容器为工作节点。目前支持的编译工具有 llvm 以及交叉编译工具链 riscv-gnu，因此在参数 `parameters` 一节新添参数 `BUILD_NODE`，选项即为容器的标签。在 Build 阶段中，`agent` 指定的运行节点即参数 `BUILD_NODE` 所选的节点

```groovy
parameters {
        // 构建节点参数，用于选择不同构建工具
        choice choices: ['llvm-18', 'llvm-17', 'riscv-13', 'riscv-14'], name: 'BUILD_NODE'
    }
```

构建阶段首先新建构建目录 `build_${BRANCH_1}_${BUILD_NODE}` ，例如 `build_4.x_llvm-18`，后续存档的构建产物和报告都放在该目录下

```groovy
stage('Build') { // 构建阶段
            agent {
                node { 
                    label "${BUILD_NODE}" // 参数 BUILD_NODE 指定编译工具容器
                }
            }

            steps {
                script { 
                    def buildDir = "build_${BRANCH_1}_${BUILD_NODE}" //构建目录，方便后续存档
                    sh "rm -rf ${buildDir}" 
                    sh "mkdir ${buildDir}" 
                    dir("${buildDir}"){ //更改工作目录，后续命令都工作在 buildDir 下
                        if (LLVM_NODES.contains(BUILD_NODE)) { // 如果编译工具是llvm，则执行 llvm 的编译命令
                            sh """
                                ${LLVM_COMMAND}
                                ${NINJA_COMMAND}
                            """
                        } else if (RISCV_NODES.contains(BUILD_NODE)) {// 如果编译工具是riscv-gnu，则执行 riscv的编译命令
                            sh """
                                ${RISCV_COMMAND}
                                ${NINJA_COMMAND}
                            """
                        }
                    }
                }
            }
        }
```

两类编译工具命令不同，因此需要分别处理，将编译命令放入环境变量中方便修改。下列命令将构建过程输出到 `.xml` 文件中：

```groovy
environment {
        
        LLVM_NODES = "llvm-18, llvm-17"
        RISCV_NODES = "riscv-14 riscv-13"

        //构建命令，将构建过程输出至 .xml 中，方便存档
        LLVM_COMMAND = '''
            cmake -G Ninja ../ \\
                -DCMAKE_BUILD_TYPE=Release \\
                -DBUILD_SHARED_LIBS=OFF \\
                -DBUILD_EXAMPLES=OFF \\
                -DWITH_PNG=OFF \\
                -DOPENCV_ENABLE_NONFREE=ON \\
                -DWITH_OPENCL=OFF \\
                -DCMAKE_TOOLCHAIN_FILE=../platforms/linux/riscv64-clang.toolchain.cmake \\
                -DRISCV_CLANG_BUILD_ROOT=/opt/llvm \\
                -DRISCV_GCC_INSTALL_ROOT=/opt/riscv \\
                -DCPU_BASELINE=RVV \\
                -DCPU_BASELINE_REQUIRE=RVV \\
                -DRISCV_RVV_SCALABLE=ON >> cmake_report.xml 2>&1
                '''
        RISCV_COMMAND = '''
            cmake -GNinja ../ \\
                    -DTOOLCHAIN_COMPILER_LOCATION_HINT=/opt/riscv/bin \\
                    -DCMAKE_BUILD_TYPE=Release \\
                    -DBUILD_SHARED_LIBS=OFF \\
                    -DWITH_OPENCL=OFF \\
                    -DCMAKE_TOOLCHAIN_FILE=../platforms/linux/riscv64-gcc.toolchain.cmake \\
                    -DRISCV_RVV_SCALABLE=ON \\
                    -DCPU_BASELINE=RVV \\
                    -DCPU_BASELINE_REQUIRE=RVV >> cmake_report.xml 2>&1
            '''
        NINJA_COMMAND = ''' ninja >> ninja_report.xml 2>&1'''
    }
```



### 3. 存档 Archive

OpenCV 构建完成的测试文件放在 `构建目录/bin/ ` 下。测试文件众多，因此新增文件选择参数 `ARCHIVE_FILES`，支持用户选择其中的几个测试文件进行测试：

```groovy
parameters {
       // 构建文件参数，用于选择测试的文件，多选
        extendedChoice(
            name: 'ARCHIVE_FILES',
            type: 'PT_CHECKBOX',
            description: 'OpenCV Benchmarks',
            multiSelectDelimiter: ',',
            value: 'opencv_perf_core,opencv_perf_imgproc,opencv_perf_dnn,opencv_test_core,opencv_test_imgproc,opencv_test_dnn', // 选项值
            defaultValue: 'opencv_perf_core,opencv_test_core' // 默认值
        )
    }
```

存档阶段将用户选择的测试文件以及报告存档（archiveArtifacts），便于后续查看。同时暂存（stash）测试文件，留至下一阶段测试。代码如下：

```groovy
stage('Archive'){ // 存档
            agent {
                node {
                    label "${BUILD_NODE}"
                }
            }
            steps {
                script {
                    def buildDir = "build_${BRANCH_1}_${BUILD_NODE}" 
                    archiveArtifacts artifacts: "${buildDir}/*.xml"  // 存档构建报告
                    def files = ARCHIVE_FILES.split(',')
                    files.each { file ->
                        archiveArtifacts artifacts: "${buildDir}/bin/${file}" // 存档测试文件，便于以后查看
                    }
                    dir("${buildDir}/bin"){
                        stash name: 'archivedFiles_1', includes: "${ARCHIVE_FILES}" //暂存测试文件，下一阶段使用
                    }
                    sh "rm -rf ${buildDir}" // 删除构建目录
                }
            }
        }
```

`archiveArtifacts` 用于将文件永久保存到 Jenkins 服务器上，以便后续查看和下载。文件存档后可以通过 Jenkins 的构建记录访问，即使流水线结束后依然可用。在服务器/容器中的存档路径为 `${JENKINS_HOME}/jobs/${JOB_NAME}/builds/${BUILD_NUMBER}/archive` 

如图，每一次归档文件都放在一个单独的文件夹（以build号命名）里：

![image-20240915180424124](image\image-20240915180326406.png)

`stash` 用于在同一个流水线的不同节点或阶段之间临时传递文件，特别是当这些阶段在不同的 Jenkins 节点上执行时。与存档不同，文件只是暂时存储在 Jenkins 中，只在流水线运行时有效



### 4. 执行 Test

执行阶段负责将之前暂存的文件发送到 RISC-V 设备上执行，并存档执行结果。新增参数 `EXECUTION_NODES`，支持用户选择多个RISC-V 设备

````groovy
    parameters {
        // 运行节点参数，用于选择测试文件的运行节点，多选
        extendedChoice(
            name: 'EXECUTION_NODES',
            type: 'PT_CHECKBOX',
            multiSelectDelimiter: ',',
            value: 'BPI-F3,k235,k236', // 选项值
            defaultValue: 'k235' //默认值
        )
    }
````

本阶段在多个节点上执行测试，因此不使用 `agent` 块定义执行节点，而是使用 `node()` 指定；使用 `unstash` 取回上个阶段暂存的文件，这些文件会被提取到当前工作目录中。

遍历这些工作节点，在每个节点上执行相关操作。这里使用 Shell 命令来运行，并将输出结果重定向到 `a1_${file}_${nodeLabel}.xml` 文件中。如当前处理的是 `opencv_perf_core` 文件，并且运行节点是 `k235`，则输出结果会被保存到 `a1_opencv_perf_core_k235.xml` 中。`|| true` 用来确保即使执行命令失败（如测试未通过），流水线也不会中断。代码如下：

```groovy
stage('Test') {
            steps {
                script {
                	// 获取用户选择的执行节点（通过参数传递），将其按逗号分割为数组
                    def nodes = params.EXECUTION_NODES.split(',')
                    nodes.each { nodeLabel -> // 遍历每个执行节点
                        node(nodeLabel) {
                            unstash 'archivedFiles_1' // 从之前暂存的文件中取回文件，准备在该节点上使用
                            def files = params.ARCHIVE_FILES.split(',')  // 获取用户选择要运行的文件
                            files.each { file -> 
                                // 执行文件，并将输出结果重定向到对应的 XML 文件中
                                sh "./${file} > a1_${file}_${nodeLabel}.xml || true"
                            }
                            archiveArtifacts artifacts: '*.xml' // 存档所有生成的 XML 文件（测试结果）
                            sh 'rm -rf *' // 删除节点上的所有文件，清理工作目录
                        }
                    }
                }
            }
        }
```



------

至此，测试文件完成了

  1.  从指定分支拉取代码
  2.  选择编译工具链进行构建 
  3.  在指定的节点上运行测试
  4.  存档构建和测试的结果

第二个分支操作相同，完整代码详见 [Jenkinsfile](opencv_test) 
