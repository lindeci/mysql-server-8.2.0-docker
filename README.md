- [概述](#概述)
- [使用说明](#使用说明)
- [深入开发工作](#深入开发工作)
  - [GDB debug 版本的 MySQL 服务](#gdb-debug-版本的-mysql-服务)
  - [更新 MySQL 源码的脚本](#更新-mysql-源码的脚本)
  - [后续重新编译 MySQL 源码的脚本](#后续重新编译-mysql-源码的脚本)
  - [本镜像制作的 Dockerfile 脚本](#本镜像制作的-dockerfile-脚本)

# 概述
本程序基于 Ubuntu 镜像构建，相当一个 Ubuntu 虚拟机。里面已经安装好了 MySQL 源码开发的环境。启动容器镜像后可以直接进行开发、调试、测试工作。

# 使用说明
```bash
# 1、先确保本地已经安装和启动 docker，请确保 docker 的根目录有 8G 的空间

# 2、下载镜像压缩包
git clone https://github.com/lindeci/mysql-server-8.2.0-docker.git

# 3、解压、加载容器镜像
cd mysql-server-8.2.0-docker
tar xvf mysql-debug-docker.tar.gz
docker load < mysql-debug-docker.tar

# 3、启动容器镜像
docker run --name mysql-docker -it -d  -w /data -p 36000:22 mysql-debug-docker:8.2.0

# 4、进入容器执行 init.sh 脚本，它会编译、初始化 MySQL 数据目录
docker exec -it -w /data mysql-docker bash
#    日志输出的倒数第二行会包含 MySQL 初始化的密码，请记住
sh init.sh   # 因为编译 MySQL 可能耗费20分钟，需要耐心等待。 请一定要记住日志输出的倒数第二行，里面包含 MySQL 初始化的密码

# 5、在容器中启动 MySQL 服务
/data/mysql-server-8.2.0/build/runtime_output_directory/mysqld-debug --user=mysql --datadir=/data/mysql-server-8.2.0/data --socket=/data/mysql-server-8.2.0/data/mysql.sock.lock &

# 6、在容器中使用 MySQL 客户端进行操作
/data/mysql-server-8.2.0/build/runtime_output_directory/mysql -uroot -p'步骤4中日志倒数第二行的密码' --socket=/data/mysql-server-8.2.0/data/mysql.sock.lock
#    修改 MySQL 密码，以后直接使用 root 这个简单密码登录数据库
ALTER USER root@localhost IDENTIFIED BY 'root';

# 7、格式美化，容器的终端显示中文时可能会出现格式不美观，需要调整终端编码
export LANG="en_US.UTF-8"
```

# 深入开发工作
## GDB debug 版本的 MySQL 服务
上面在容器中编译出来的 `/data/mysql-server-8.2.0/build/runtime_output_directory/mysqld-debug` 已经是 debug 版本，可以直接调试

## 更新 MySQL 源码的脚本
```sh
cd /data/mysql-server-8.2.0
git fetch
```

## 后续重新编译 MySQL 源码的脚本
```sh
cmake --build /data/mysql-server-8.2.0/build --config Debug --target mysqld -j 32
```
上面的 `-j 32` 是线程数，大家根据自己的硬件资源进行调整

## 本镜像制作的 Dockerfile 脚本
```sh
# 使用本地的 ubuntu 镜像作为基础镜像
FROM ubuntu

# 更新 apt 包列表
RUN apt-get update

# 安装必要的软件包
RUN apt-get install -y apt-utils cmake git bison libssl-dev libncurses5-dev g++ pkg-config bzip2 vim openssh-server

# 添加 mysql 用户和用户组
RUN addgroup mysql
RUN adduser --disabled-password --gecos "" --ingroup mysql mysql

# 为了在容器中开启22端口，创建 SSHD 目录。后续可以使用 VScode 连接容器进行 MySQL 源码开发
RUN mkdir /var/run/sshd
# 设置 SSH
RUN echo 'root:root' | chpasswd
RUN sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
# SSH 登陆修复
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
RUN echo "export VISIBLE=now" >> /etc/profile

# 创建 /data 目录并设置为工作目录
WORKDIR /data

# 将当前目录的文件复制到镜像的 /data 目录
COPY boost_1_77_0 /data/boost_1_77_0
# 克隆 GitHub 仓库
RUN git clone https://github.com/lindeci/mysql-server-8.2.0-debug.git mysql-server-8.2.0
COPY init.sh /data

CMD ["/usr/sbin/sshd", "-D"]
```