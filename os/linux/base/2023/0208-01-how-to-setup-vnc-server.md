# 搭建vnc服务
### 一. 基本信息
- 操作系统及版本: 7.9.2009
- 软件信息版本：tigervnc-server-1.8.0-23.el7_9.x86_64
### 二. 安装步骤

#### 1. 在centos上安装
##### 安装步骤
``` bash
# 更新系统 可跳过
yum update -y
# 安装桌面
yum groups install "GNOME Desktop" -y
# 安装 tiger-vnc
yum install tigervnc-server -y

# 查看vpc 版本
rpm -qa|grep vnc 

# 旧版本使用 vi /etc/sysconfig/vncservers 进行配置
# 但是新版本已经提示使用/lib/systemd/system/vncserver@.service作为模板进行配置

# 复制文件作为模板，@后的数字表示服务序号，可开启多个
cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:1.service

# 使用vi 按照提示编辑配置文件 将模板中的<USER>改为指定用户（本文使用root）
vi /etc/systemd/system/vncserver@:1.service

# 设置密码 根据提示输入
vncpasswd 

# 启动服务并设置为立刻启动
systemctl enable vncserver@:1.service --now

# 查看服务状态
systemctl status vncserver@:1.service

```
##### 附录
1. /etc/systemd/system/vncserver@:1.service

```
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=simple

# Clean any existing files in /tmp/.X11-unix environment
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
# ExecStart=/usr/bin/vncserver_wrapper <USER> %i
ExecStart=/usr/bin/vncserver_wrapper root %i
ExecStop=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'

[Install]
WantedBy=multi-user.target
```

### 三、连接方式
1. window 下载tigervnc-client
2. linux 
    - centos: yum install tigervnc-client
3. 连接时，输入ip:1 连接启动的vnc
