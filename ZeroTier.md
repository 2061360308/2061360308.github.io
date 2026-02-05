---
title: 异地组网开黑之ZeroTier
data: 2026-02-15T22:21:37+08:00
lastmod:  2026-02-15T22:21:37+08:00
categories:
  - 折腾笔记
tags:
  - ZeroTier
  - 网络
---

相较于内网穿透这种方案 ZeroTier 更加适合于个人有限设备之间的互相通信，ZeroTier在条件允许的情况下会尽量使用点对点连接，因此延迟要更低一点。

个人实测这个服务很稳定，ZeroTier会注册系统服务，在学校的时候我给朋友电脑配置好了就没怎么管，过了两个月，我两都回家了某天测试开机后网络还是互通的Nice~

## 内网穿透关于ZeroTier的使用教程

前段时间注册使用了zerotier的免费账户，一个局域网可以加入10台设备，简单使用绰绰有余了。并且加入自己的mood（中继）节点后速度也还可以，实际上大部分网络都可以进行点对点直连，连接很稳定。由于客户端提供功能有限，ZeroTier实际使用过程中需要使用大量的命令行操作，这里记录一下，方便后续使用。

### 网络加入

思路就是下载ZeroTier官方提供的对应平台工具后续启动即可。由于Windows上有可视化界面，下载安装程序也较为简单，而后续配置命令大多是一致的，所以这里用Linux作为示例。

#### 安装

在 ubuntu 中安装 zerotier 使用 SSH 两种安装方法，安装过程会有进度显示：

● 如果您愿意依靠 SSL 来验证站点，则可以通过以下方式完成单行安装：

```bash
curl -s https://install.zerotier.com | sudo bash
```

否则，在基于 debian/ubuntu 的系统上，还可以使用下列命令（ centos 系统需要将 apt 替换为 yum ）：

```bash
sudo apt install zerotier-one
```

#### 加入网络

在要连接的Linux设备上输入如下命令加入网络，如果连接成功就会出现 200 join OK 的状态码提示：

对应网络名从官网账户获取即可。

> 加入成功后不要忘了去官网给对应设备授权，同时你可以为对应设备设置相应备注信息方便后续区分。

```bash
# 加入网络命令，操作成功则返回 “200 join OK” 
sudo zerotier-cli join  ###########
```

#### 其他设置

Linux下需要用命令启动服务开机自启，Windows需要在`计算机->管理->服务`中进行管理

```bash
sudo systemctl enable zerotier-one.service
```

断开网络命令

```bash
sudo zerotier-cli leave ###########
```

Linux 停止服务命令

```bash
sudo systemctl stop zerotier-one
```

Linux重启服务命令

```bash
sudo systemctl start zerotier-one
```

### 中继节点搭建（Linux）

> 除了下载安装等系统类型命令，zerotier-cli等工具提供的命令是多端通用的

需要先在成为中继节点的设备上执行上一步的安装ZeroTier步骤

#### 进入安装目录

```bash
cd /var/lib/zerotier-one/
```

#### 生成节点配置

```bash
zerotier-idtool initmoon identity.public > moon.json
```

#### 编辑配置

将上述生成的moon.json中的stableEndpoints选项内容改为`[ip/9993]`

#### 移动配置文件到对应位置

> 这里假设你生成的配置文件名为    000000be5c372689.moon
> 完整路径为  /var/lib/zerotier-one/moons.d/000000be5c372689.moon

```bash
mkdir moons.d
mv 000000be5c372689.moon moons.d/
```

#### 重启服务

```bash
systemctl restart zerotier-one
```

#### 在其他需要使用该中继节点的设备上进行配置

首先需要拷贝生成的`000000be5c372689.moon`配置文件到目标设备上

1. Windows上的路径为：C:\ProgramData\ZeroTier\One\moons.d\
2. Linux上的路径为：/var/lib/zerotier-one/moons.d/
3. 手机端有对应按钮选择文件即可自动加载配置文件

设置使用改中继节点

```bash
zerotier-cli orbit be5c372689 be5c372689
```

#### 再次重启服务

按照对应平台的方法重启服务即可

### 服务测试

```bash
zerotier-cli listpeers
```
