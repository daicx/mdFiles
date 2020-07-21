# docker实战

## 安装

我们假设拿到一台新鲜的centos 8.

1. 更新系统软件包和内核为最新

   ```
   sudu su
   yum -y update
   ```

   

2. centos 7 直接使用以下命令

   ```
   yum install -y docker
   ```

3. centos 8 安装

   3.1 下载docker-ce的repo

   ```
   curl https://download.docker.com/linux/centos/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo
   ```

   3.2 安装依赖（这是相比centos7的关键步骤）

   ```
   yum install https://download.docker.com/linux/fedora/30/x86_64/stable/Packages/containerd.io-1.2.6-3.3.fc30.x86_64.rpm
   ```

   3.3 安装docker-ce

   ```
   yum install docker-ce
   ```

   3.4 启动docker

   ```
   systemctl start docker
   ```