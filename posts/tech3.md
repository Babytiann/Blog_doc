---
title: "从关机到进入视频会议，这期间到底经历了什么？"
published: 2025-11-26
tags: ["博客", "408", "技术"]
category: "技术"
lang: "zh-CN"
draft: false
comment: true
---

最近在刷牛客的时候也是遇到了一道特别考你基础掌握的咋样的题，就是从关机状态一直到开机并进入视频会议参加面试，这期间发生了什么。那么话不多说，讲讲我所知的吧 ---持续完善中，Linux Window可能穿插着。。

# 一、从开机引导到正式进入操作系统

> 计组、操作系统

## 1.从按下开机键到执行第一条指令

## 从按下电源键到加载 .efi：主板启动流程（修正版）

当机箱电源键短接 PWRBTN# 引脚后，信号被主板上的 **EC（Embedded Controller）或 PCH** 捕获，系统开始上电流程。随后 CPU 被释放，开始执行固件程序，也就是我们所说的 **BIOS/UEFI（固件）**。

### 1. CPU 复位入口（Reset Vector）

CPU 上电后会执行硬件 Reset Vector：

- 传统 BIOS：指定 CS:IP（0xF000:0xFFF0 → 0xFFFF0）
- 现代 UEFI：直接进入固件的 64-bit 初始化流程

BIOS/UEFI 可以看作一个“微型操作系统”，负责最初的硬件管理与引导。

### 2. UEFI 初始化硬件

UEFI 会初始化关键硬件，包括：

- CPU 与平台控制器（PCH/EC）
- 内存控制器与 DRAM
- PCIe 总线与显卡、NVMe、USB 等设备
- 加载 UEFI 驱动（磁盘、键盘、网络等）

> ⚠ 注意：UEFI **不会初始化 MMU 也不会建立页表**。内核启动后才启用分页。

完成硬件初始化后，UEFI 进入引导阶段。

### 3. 寻找可引导设备（ESP）

与 Legacy BIOS 不同：

- **BIOS** → 固定读取磁盘的 512B（MBR）
- **UEFI** → 读取 **EFI System Partition（FAT32）**
   在其中查找适合当前平台的 `.efi` 文件（如 `BOOTX64.EFI`）

### 4. 加载并执行 .efi 文件

UEFI 的磁盘驱动（NVMe、AHCI、USB）负责从磁盘读取 `.efi` 文件：

1. 调用设备驱动执行 I/O（NVMe 必定使用 DMA，其他设备可能是 PIO 或 DMA）

2. 将文件内容拷贝到主存（RAM）

3. 固件定位 PE/COFF 格式的 **入口点（EntryPoint）**--这其中还有相当多的具体细节，以后我会了再来补充吧

4. CPU 执行：

   ```
   jmp EntryPoint
   ```

至此，控制权从固件交到引导程序（如 GRUB 或 Linux Bootloader）手中

## 2.CPU 加载内核到切换完文件系统 (Linux)

当 UEFI 固件开始执行 `.efi` 引导程序后，引导程序会继续加载 Linux 内核（通常是 **vmlinuz**）以及与其配套的 **initramfs**。整个流程可以分为几个阶段：

### 1. 加载内核与 initramfs

UEFI 将 `vmlinuz` 和 `initramfs` 一并加载到内存中，并跳转到内核的入口点。
 **initramfs** 是内核启动时使用的一个 **临时文件系统**，通常以内存盘（ramdisk）的形式挂载。
 它可以非常小，小到只包含两个必需对象：

- `init`：第一个用户态程序
- `/dev/console`：为 `init` 提供标准输入/输出/错误（stdin/stdout/stderr）

### 2. initramfs 内部的用户态环境

尽管 initramfs 可以极简，但实际系统中通常会包含一个 **busybox**：

- 路径：`/bin/busybox`
- busybox 集成了大量基础命令（`vi`、`ping`、`tty` 等），方便系统在最小环境下完成初始化工作。

在这个阶段，Linux 还处于一个 **非常小、完全依赖内存** 的临时根文件系统中。
 任务包括：挂载磁盘、检查设备、加载真正的文件系统等。

### 3. 切换到真正的根文件系统

当 initramfs 完成必要的准备工作后，需要切换到磁盘中真正的 rootfs。
 这一步通常通过用户态工具 **switch_root** 完成，而它底层依赖的系统调用是：

- **pivot_root**：真正负责把根文件系统切换到磁盘上的版本。

完成切换后，系统的根目录就从 initramfs 迁移到了磁盘中的真实 Linux 文件系统，之后的启动流程（如 systemd）才真正开始。

/init 会通过执行execve 系统调用变成 systemd 这个进程，pid为 1，随后以 **服务单元（unit）** 去依序拉起一堆守护进程，通过 fork，execve 构建整颗操作系统下的进程树。

systemd 会在此时启动图形服务（GDM，lightDM）

# 二、进入桌面并打开 app 进入链接

> 操作系统、计网

双击桌面图标就是运行硬盘里的可执行文件（Windows 是 .exe，Linux 是 ELF）。 可执行文件加载时，操作系统先把程序通过 mmap 映射到内存里。 静态链接库已经在可执行文件中，无需加载（当然现在几乎所有的应用程序都是动态链接）。 动态库在加载阶段由 Loader（Windows Loader 或 Linux ld.so）按需加载。 在 Linux 中，如果是动态链接的 ELF，内核会先加载动态链接器 ld-linux.so.2，再由它加载所有依赖的 .so，最后跳到程序入口。

## 1. 进程层面

程序启动后会进入 Main Thread（UI 线程）

这个主线程负责初始化 app 的 UI 界面，初始化窗口系统。随后会启动渲染引擎绘制 UI，最后启动网络线程。期间会处理很多用户的I/O 操作

当 `_start` → `main` 被执行，程序进入用户空间正式运行。

以下直到网络层面是 AI 的。。因为我目前不是特别懂

### 1. Main Thread（主线程 / UI 线程）

在桌面 GUI 程序中，**主线程通常负责：**

- 初始化应用运行环境（AppContext、Application 对象）
- 初始化 UI 框架（Qt/GTK/Win32/GDI+/DirectUI 等）
- 创建程序主窗口（Window / Activity / Stage）
- 设置事件循环（Event Loop）
- 响应用户输入（键盘、鼠标事件）
- 驱动整个应用生命周期

也就是说，你看到的界面初始化全在 **主线程** 完成。

------

### 2. 渲染线程（Render Thread）

启动 UI 框架后，会创建一个专门的渲染线程：

- 合成 UI 树（View Tree / Widget Tree）
- 栅格化元素（Rasterization）
- GPU Command 提交
- 每帧重绘（VSync 驱动）

在 Windows 上可能用 **DirectX / Skia / GDI+**
 在 Linux 桌面通常用 **OpenGL、Wayland、X11 + GL、Skia**
 Mac 上是 **Metal + CoreAnimation**

这样主线程就不会因为绘图阻塞。

------

### 3. 网络线程（IO Thread）

应用启动后通常会创建专门用于网络/IO 的线程：

- 处理 HTTP/TCP/WebSocket 请求
- 异步 IO（epoll/kqueue/IOCP）
- 数据收发、协议解析
- 不阻塞 UI

浏览器、视频会议、IM、客户端软件几乎都有专门的 IO Thread。

## 2. 网络层面

### 1.解析 URL --直接输入会议代码的还有额外逻辑

1. 解析协议 ---https
2. 解析域名 host
3. 解析路径、token、room id 等参数
4. 查询本地缓存（浏览器缓存 → OS 缓存 → hosts）

------

### **2. 如果无缓存：进行 DNS 查询**

浏览器将查询发送到**本地 DNS（LDNS）**，LDNS 负责整个递归查询。

1. **LDNS → 根服务器：**
    根服务器返回“去找对应的顶级域名服务器（如 .com）”。
2. **LDNS → TLD 顶级域名服务器：**
    TLD 返回“example.com 的权威 DNS 地址”。
3. **LDNS → 权威 DNS 服务器：**
    权威 DNS 返回最终 IP（A / AAAA 记录）。
4. **LDNS 缓存结果并返回给浏览器。**

### 3. 建立 TCP 连接

TCP 连接会经历三次握手，总共会传递三次报文，连接完成后会进行 TLS/SSL 握手，实现我们常见的 HTTPS

#### 1. TLS 握手开始（协商加密参数）

##### 1. Client Hello（客户端问候）

客户端向服务器发送以下信息：

- **Client Random**（客户端随机数）
- **支持的 TLS 版本**
- **支持的加密套件列表**（如 TLS_RSA_WITH_AES_128_CBC_SHA）
- **会话恢复 Session ID / Session Ticket** 等扩展

这是 TLS 握手的起点，用于表明客户端的能力和偏好。

------

##### 2. Server Hello（服务器问候）

服务器从客户端的能力列表中做选择，并返回：

- **Server Random**（服务器随机数）
- **最终选择的 TLS 版本**
- **最终选择的加密套件**
- **会话恢复相关扩展**

至此，双方已经明确后续使用哪种加密与认证方式。

------

#### 2. 服务器身份认证

##### 3. Server Certificate（服务器证书）

服务器向客户端发送数字证书，其中包含服务器的公钥。

客户端收到证书后会执行以下验证步骤：

1. **使用本地内置的 CA 根证书验证证书签名**，确保证书未被篡改
2. **检查证书有效期**，确保证书未过期
3. **校验证书中的域名是否与目标域名匹配**

验证通过，则客户端可以信任服务器的公钥，用来进行后续的密钥交换。

##### 4. Server Hello Done

服务器表示自己的 TLS 参数已经发送完毕，等待客户端继续下一步。

#### 3. 生成对称密钥（RSA 密钥交换）

##### Client Key Exchange

客户端执行以下操作：

1. **生成 pre-master secret**（预主密钥）
2. 使用服务器证书中的 **服务器公钥** 加密 pre-master
3. 将加密后的 pre-master 发送给服务器

服务器收到后：

- 使用自己的 **私钥** 解密，得到相同的 pre-master

现在双方都有三个数据

- Client Random
- Server Random
- pre-master secret

TLS/SSL到此时正式建立成功，随后客户端和服务端都以此来发送数据

### 4. 建立 WebSocket 连接：从 HTTP 到 WebSocket 的协议升级

在完成 TCP 三次握手以及 TLS 握手之后，虽然客户端与服务器之间已经建立了一个加密的通信通道，但为了实现实时的音视频数据传输，我们需要更适合长连接和全双工通信的协议 —— WebSocket。

#### 1. 客户端发起 WebSocket 协议升级请求

WebSocket 的建立并不是重新创建一条新的连接，而是在已经建立好的 TCP（或 TLS）通道上，使用一次 HTTP 请求向服务器发起协议升级（Protocol Upgrade）：

客户端发送类似如下的 HTTP 报文：

```
GET /xxx HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: <随机字符串>
Sec-WebSocket-Version: 13
```

关键字段说明：

- `Upgrade: websocket` 表示客户端希望将当前连接从 HTTP 升级为 WebSocket
- `Connection: Upgrade` 表示本次请求用于协议切换
- `Sec-WebSocket-Key` 是客户端随机生成的校验 key
- `Sec-WebSocket-Version` 一般为 13（RFC 6455）

这一步可以理解为：**客户端向服务器提出协议升级申请**。

------

#### 2. 服务器返回 101，确认协议升级

如果服务器支持 WebSocket，它会返回以下响应：

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: <服务器生成的校验值>
```

其中最关键的是：

- **状态码 101**：表示服务器同意切换协议
- `Sec-WebSocket-Accept`：对客户端 key 进行特定算法处理后的返回值，用于校验请求合法性

当客户端收到 101 响应，即表示服务器已经接受升级请求，此时 HTTP 协议正式完成向 WebSocket 的切换。

------

#### 3. 切换成功后进入 WebSocket 全双工通信阶段

协议升级完成后：

- 不再有 HTTP 报文格式
- 双方开始使用 WebSocket 帧（Frame）进行通信
- 连接默认保持长连接状态
- 客户端和服务器均可主动发送数据（全双工）
- 非常适合实时音视频、白板协作、聊天、直播等场景

从这一刻开始，WebSocket 连接正式建立，客户端即可通过这条通道持续收发实时数据。

实际视频会议通常不是只用 WebSocket，还会使用 WebRTC（STUN/TURN/ICE），不过基础版本用 WebSocket 也能讲清楚。



# 结语

当然整个流程还有无数细节可以继续深挖，比如 UEFI 的内存映射策略、内核启动参数、TLS 的密钥派生细节、WebSocket 帧格式、视频会议的编解码链路等等，篇幅有限这里就不展开了。

本文主要尝试把“从按下电源键到进入视频会议”的全链路串成一个整体脉络，让大家看到一个完整的系统视角。

 如果文中有不准确或可以补充的地方，也非常欢迎交流指正，我会第一时间更新完善。