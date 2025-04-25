# 实验0.1：虚拟机环境搭建

## 目标
创建并管理使用虚拟机监控程序的虚拟机，理解虚拟化的基本原理。

## 准备工作
- 一台至少8GB内存、50GB可用磁盘空间的电脑
- 管理员权限

## 步骤

### 1. 安装虚拟机监控程序

根据您的系统选择合适的虚拟机监控程序：

#### Windows用户
- **选项1: VMware Workstation**
  - 访问 https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html 下载安装包
  - 运行安装程序，按照向导完成安装
  - 安装过程中可能需要输入许可证密钥（评估版提供30天试用期）
  - 安装完成后重启计算机以完成设置

- **选项2: VirtualBox** 
  - 访问 https://www.virtualbox.org/wiki/Downloads 下载安装包
  - 运行安装程序，按照向导完成安装

- **选项3: Hyper-V** (Windows 10 专业版或企业版)
  - 打开"控制面板" → "程序" → "程序和功能" → "启用或关闭Windows功能"
  - 勾选"Hyper-V"并点击确定

#### macOS用户
- **VirtualBox**
  - 访问 https://www.virtualbox.org/wiki/Downloads 下载安装包
  - 打开下载的磁盘映像，双击安装程序

#### Linux用户
- **VirtualBox**
  ```bash
  # Ubuntu/Debian
  sudo apt update
  sudo apt install virtualbox
  
  # Fedora/CentOS
  sudo dnf install VirtualBox
  ```

### 2. 创建新虚拟机

我们将创建一个运行Ubuntu Server的虚拟机：

#### 下载Ubuntu Server ISO
- 访问 https://ubuntu.com/download/server 下载最新的Ubuntu Server ISO

#### 使用VMware Workstation创建虚拟机
1. 启动VMware Workstation并点击"创建新虚拟机"或"文件" → "新建虚拟机"
2. 选择"典型(推荐)"安装类型，点击"下一步"
3. 选择"安装程序光盘映像文件(iso)"，浏览并选择下载好的Ubuntu Server ISO，点击"下一步"
4. 提供虚拟机信息：
   - 输入虚拟机名称（例如"UbuntuVM"）
   - 选择虚拟机存储位置
5. 指定磁盘容量：
   - 设置最大磁盘大小为20GB
   - 选择"将虚拟磁盘拆分成多个文件"（推荐用于便携性）
6. 点击"自定义硬件"进行高级设置：
   - 内存：设置为2048MB（2GB）
   - 处理器：设置为2个CPU核心
   - 网络适配器：选择"NAT"（默认）
7. 点击"完成"创建虚拟机
8. 虚拟机将自动启动并进入Ubuntu安装过程



### 3. 安装操作系统

1. 启动虚拟机
2. 按照Ubuntu Server安装向导操作：
   - 选择语言：English
   - 选择键盘布局
   - 选择"Install Ubuntu Server"
   - 网络配置：使用默认设置
   - 存储配置：使用整个磁盘
   - 设置用户名和密码
   - 安装SSH服务器（推荐）

3. 安装完成后，登录系统并验证功能：
   ```bash
   # 查看内存使用情况
   free -h
   
   # 查看磁盘使用情况
   df -h
   
   # 查看CPU和进程
   top
   ```

### 4. VMware特有功能使用

如果您使用VMware Workstation，可以利用以下特有功能：

1. **安装VMware Tools**：
   - 在虚拟机运行状态下，点击"虚拟机" → "安装VMware Tools"
   - 在Ubuntu中挂载并安装工具：
   ```bash
   sudo mount /dev/cdrom /mnt
   cd /mnt
   sudo cp VMwareTools-*.tar.gz /tmp/
   cd /tmp
   sudo tar xzvf VMwareTools-*.tar.gz
   cd vmware-tools-distrib
   sudo ./vmware-install.pl
   ```

2. **创建快照**：
   - 点击"虚拟机" → "快照" → "拍摄快照"
   - 输入快照名称和描述

3. **克隆虚拟机**：
   - 关闭虚拟机
   - 右键点击虚拟机 → "管理" → "克隆"
   - 选择"创建完整克隆"或"创建链接克隆"

### 5. 性能分析

完成以下实验并记录观察结果：

1. 在虚拟机内创建一个简单的CPU压力测试：
   ```bash
   # 安装压力测试工具
   sudo apt update
   sudo apt install stress
   
   # 运行CPU压力测试
   stress --cpu 2 --timeout 60s
   ```

2. 在宿主机上观察资源使用情况
   - 在VMware中，可以通过"虚拟机" → "管理" → "虚拟机性能"查看资源使用图表
   
3. 记录以下指标：
   - 虚拟机启动时间
   - 在压力测试期间的CPU使用率
   - 虚拟机内存占用

## 思考问题

1. 虚拟机与裸机相比，性能有什么差异？造成这些差异的原因是什么？
2. 虚拟化技术的主要优势和局限性是什么？
3. 为什么容器技术成为虚拟机的轻量级替代方案？
4. VMware Workstation与其他虚拟化解决方案（如VirtualBox、Hyper-V）在性能和功能上有何区别？

## 扩展练习

- 使用VMware Workstation的快照功能创建虚拟机快照，然后修改系统，最后恢复到快照状态
- 尝试配置不同的网络模式（如桥接网络、仅主机网络），并测试虚拟机与宿主机及外部网络的连接
- 尝试创建虚拟机模板，并基于该模板快速部署多个相同配置的虚拟机 