# 如何强制玩家切换到公网IP进入 RadminVPN 虚拟局域网下的 Minecraft 服务器
v1.0 | 作者 AXFJ | 文档遵循 CC BY 4.0 协议。

[English](README.md) | **中文**

## 1. 前言

> 在 RadminVPN 等虚拟局域网下开服，经常会遇到恶意破坏的玩家。由于虚拟网络的特殊性，用户可以使用特殊手段切换IP，因此ban-ip并不能有效地阻挡破坏者。

- 本教程讲述了如何强制所有玩家自动切换到公网IP加入服务器。
- 在按照教程配置后，可以达到以下效果：
  1. 玩家通过虚拟IP进入你的服务器
  2. 玩家被无缝重定向到公网IP服务器（可以是FRP），因此自己也会使用公网IP进入服务器
  3. 因此玩家暴露自己的虚拟IP和公网IP，若被ban-ip，即使切换虚拟IP也会用相同的公网IP进入服务器，大幅提高服务器的安全性

## 2. 原理

（涉及游戏通信协议技术，不想看可以跳到下一节）

原版游戏有 `/transfer` 指令，该指令可以将指定玩家转移到另一个服务器。因此，通过使用一个特殊服务端，在玩家一旦加入服务器后立即发送 `ServerTransferS2CPacket` 到我们的另一个处在公网IP的服务器，从而以公网IP进入服务器。

需要注意的是，这个过程对于客户端是**不透明的**。因此，别有用心的玩家可以修改客户端使其接收到转移包后，发现是目标服务器是公网IP后立刻断开连接。不过这样也无大碍，毕竟他无法进入真正的服务器。对于这类可能存在的拒绝进一步连接的客户端即是可疑IP。

## 3. 准备工作

准备好这些：
1. [Transfer Server Router](github.com/AXFJ/Transfer-Server-Router/releases)：转移服核心，或者从[蓝奏云](https://wwati.lanzouu.com/i6jPf44cj7zc) 下载
2. Minecraft 服务器核心：建议使用 Paper 或 Purpur。文档中使用 1.21.11 版本（对于非 Paper 及其衍生核心的配置略复杂，后面会提到）
3. 内网穿透（FRP）或 公网IP服务器：推荐使用内网穿透[ChmlFrp](https://chmlfrp.net)，稳定免费
4. 局域网广播器（可选）：用于将你的服务器广播到局域网世界列表。可以使用[Sculk Toolkit](https://github.com/AXFJ/Sculk-Toolkit)（或从[蓝奏云](https://wwati.lanzouu.com/i6Ofv44cjsib)下载）
然后就可以开始配置了。

---

## 4. 开始配置

### 1. 启动转移服务

1. 打开 Transfer Server Router。
   应该会显示如下日志：
   ```text
   Transfer Server Router v1.0 by AXFJ
   See https://github.com/AXFJ/Transfer-Server-Router.
   [2026-08-24 18:12:50] [INFO] [-] Configuration file tsr_server.properties does not exist, using default configuration and creating default file.
   [2026-08-24 18:12:50] [INFO] [-] Listen on：0.0.0.0:25565 -> example.com:25565
   [2026-08-24 18:12:50] [INFO] [-] Protocol Version：774
   [2026-08-24 18:12:50] [INFO] [-] Limitations：Total Concurrent=5, Per-IP Concurrent=2, Rate=1.0 req/s, Timeout=15s
   ```
   这时就可以关闭程序了。

   此时程序文件夹下会多出一个 `tsr_server.properties`，是转移服的配置文件，打开它，找到如下几项：
   ```properties
   ip=0.0.0.0
   port=25565
   ```
   这是转移服要运行在的位置，建议使用默认25565端口，其余按照自己需求修改，`ip` 一般无需修改。
   ```properties
   target-ip=example.com
   target-port=25565
   ```
   这是转移服在玩家连接后将要转移的位置，稍后我们会修改。其它项一般无需修改。
2. 启动转移服。
   
### 2. 准备内网穿透

这个部分将会使用[ChmlFrp](https://chmlfrp.net)进行演示，使用别的也行，操作大致相同。

1. 打开[ChmlFrp](https://chmlfrp.net)，注册账号。
2. 打开仪表盘，找到“隧道管理”。
3. 选择你喜欢的节点创建一个隧道。

   <img src="./images/1.png" alt="替代文本" width="300" height="200">

   内网端口是你的服务器运行的端口，外网端口可以任选。
   “额外参数”一定要填写如下：
   ```ini
   proxy_protocol_version = v2
   ```
4. 此时，你可用把 `tsr_server.properties` 中的 `target-ip` 和 `target-port`改成你的隧道信息了。推荐使用25566端口。

### 3. 运行内网穿透

1. 从你的内网穿透官网下载客户端，登录。
2. 启用你的隧道。

至此内网穿透部分配置完成

### 4. 启动 Minecraft 服务器

1. 从 [Paper](https://fill-ui.papermc.io/projects/paper/version/1.21.11) 官网下载 1.21.11 服务器核心。更高或更低的版本可能兼容，建议先使用 1.21.11 版本
2. 保存到任意文件夹，打开终端运行：
   ```bash
   java -jar paper-1.21.11-132.jar
   ```
3. 出现“Done”时关闭服务器，进行进一步配置。
4. 打开 `server.properties`，修改如下几项：
   ```properties
   accept-transfers=true
   ...
   ip=127.0.0.1
   port=<你内网穿透中设定的隧道本地IP>
   ...
   ```
   这里必须把`ip`设置成`127.0.0.1`。
5. 对于 Paper 系服务器：

   1. 打开`config\paper-global.yml`
   2. 找到这一段：
      ```yml
      proxies:
        ...
        proxy-protocol: false
        ...
      ```
      把 `proxy-protocol` 改成 `true`。
    
    对于非 Paper 系服务器（上游 Spigot 也算），需要安装插件，这里不多赘述。以下是参考：

    - Spigot / Bukkit：HAProxy Detector
    - Fabric / Quilt: Proxy Protocol Support
    - Forge / NeoForge: 同上
    - 原版：无法配置
  
  6. 启动服务器。

## 5. 完成

至此，所有程序已经配置完成。现在打开游戏客户端，输入你的虚拟IP（转移服的端口），看看是否连接到了公网服务器，以及服务器后台是否显示了你的公网IP。

## 6. FAQ

出现任何问题，先把教程从头到尾再过一遍，如果还是有问题，可以联系我。

QQ：3647980885

邮箱：its-axfj@outlook.com

哦对了，如果你要使用广播器请使用仅LAN模式。
