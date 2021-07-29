# 在 WSL 中使用 Cuda（docker 方式）
WSL 指 Windows Subsystem for Linux。

本文基本参照 [Ubuntu 官方说明](https://ubuntu.com/blog/getting-started-with-cuda-on-ubuntu-on-wsl-2) 操作，对于额外遇到的问题进行补全。
## 零. 问题汇总
### 0.2. Docker 报错
1. docker: Cannot connect to the Docker daemon  
docker 服务未启动，如下启动
```bash
service docker start
```
2. docker: Got permission denied while trying to connect to the Docker daemon  
需要 `sudo`，或者按下文2.1中设置用户组操作。
3. 运行需要 cuda 的任务时遇到 docker: Error response from daemon: OCI runtime create failed
## 一. 准备工作
### 1.1. 启用 Windows 预览体验计划
在 win10 左下快捷搜索或设置中搜索 “insider” 即可进入设置界面，一路开通即可。
### 1.2. 安装 WSL
按 [Windows 官方指南](https://docs.microsoft.com/zh-cn/windows/wsl/install-win10) 走到“快速入门-创建用户账户和密码”即可。

**确认版本**：在 Ubuntu 中输入
```bash
uname -r
```
**确认输出的信息中有 “WSL2”。**

#### 1.2.1 将 WSL1 升级到 WSL2
如果已经在 WSL1 中安装并使用过 Ubuntu 时，先执行 [官方指南](https://docs.microsoft.com/zh-cn/windows/wsl/install-win10) 中“快速入门-安装 WSL 1 与更新到 WSL 2”中步骤3和4。  
然后关掉或退出所有在 Ubuntu 中的命令行界面，然后在 Win10 命令行中 
```bash
wsl --set-version Ubuntu 2
```
其中 `Ubuntu` 可以替换为具体在用的子系统名称。

### 1.3. Nvidia for CUDA on WSL 驱动安装
这个是安装在 win10 上的专门驱动，从[这里](https://developer.nvidia.com/cuda/wsl/download)获取。

## 二. 安装 docker 与 nvidia-docker
### 2.1. 安装 docker
遇到错误：  
按照 [Ubuntu 官方说明](https://ubuntu.com/blog/getting-started-with-cuda-on-ubuntu-on-wsl-2) 中的 `sudo apt -y install docker.io` 安装后输入下面这个命令启动 docker
```bash
sudo service docker start
```
报错 **docker: unrecognized service**。因此我卸载 docker 后重新按[其他教程](https://www.cnblogs.com/walker-lin/p/11214127.html)安装。

卸载命令
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

然后按照[教程](https://www.cnblogs.com/walker-lin/p/11214127.html)依次进行
```bash
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

sudo apt-key fingerprint 0EBFCD88

sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io
```

到这里 docker 应该已经安装好了，下面设置免 sudo 运行 docker。将其中的 `用户名` 替换为真实用户名。
```bash
sudo gpasswd -a 用户名 docker

sudo service docker restart

newgrp - docker

```

可以验证 docker 正常工作
```bash
docker run hello-world
```
    