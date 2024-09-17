# 使用 Jenkins 官方基础镜像
FROM jenkins/jenkins:2.414.3-jdk17

# 使用 root 权限来安装插件
USER root

# 安装指定的插件
COPY --chown=jenkins:jenkins plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli -f /usr/share/jenkins/ref/plugins.txt


# 在首次启动时跳过默认的安装引导页面
ENV JAVA_OPTS=-Djenkins.install.runSetupWizard=false

# 切换回 Jenkins 用户
USER jenkins