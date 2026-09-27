# VS Code Remote SSH 代理转发与服务器环境配置指南

通过 VS Code 的 SSH 反向端口转发（Remote Port Forwarding），将本地 Windows 宿主机上的代理网络安全地共享给远程 Linux 服务器，实现服务器终端一键科学上网。

## 核心原理

```
[Windows 宿主机]                       [远程 Linux 服务器]
Clash (127.0.0.1:7890) <=== SSH 隧道 === 127.0.0.1:7890 (RemoteForward)
                                               ^
                                               |
                                     export http_proxy=...
                                               |
                                       终端进程 (curl, git, pip...)

```

* **高安全性**：无需开启 Clash 的“局域网连接（Allow LAN）”，无需暴露公网端口或调整路由器防火墙。

* **隔离性好**：流量仅通过 SSH 加密隧道穿透，断开 VS Code 连接即自动切断。

## 操作步骤

### Step 1. 确认 Windows 本地代理状态

1. 启动 Windows 上的 Clash / Clash Verge 客户端。

2. 确认代理内核处于工作状态，默认混合端口（Mixed Port）通常为 **`7890`**。

### Step 2. 配置 VS Code 本地 SSH 转发

1. 在 VS Code 中按下快捷键 `Ctrl + Shift + P`（macOS 为 `Cmd + Shift + P`）。

2. 输入并选择 **`Remote-SSH: Open SSH Configuration File...`**。

3. 选择对应的 SSH 配置文件（通常位于 `C:\Users\<你的用户名>\.ssh\config`）。

4. 在对应服务器的配置块后添加 `RemoteForward 7890 127.0.0.1:7890`：

```
Host your-server-name
    HostName 10.x.x.x
    User your_username
    Port 22
    RemoteForward 7890 127.0.0.1:7890

```

> **注意**：修改保存后，需在 VS Code 中**重新连接远程窗口**（或按 `Ctrl + Shift + P` 执行 `Developer: Reload Window`）以建立反向转发隧道。

### Step 3. 在服务器配置一键代理开关

连接到远程服务器后，在终端运行以下命令，将快捷代理函数一键写入 `~/.bashrc`：

```
cat << 'EOF' >> ~/.bashrc

# ==========================================
# Proxy Settings for VS Code Remote Forward
# ==========================================
proxy_on() {
    export http_proxy="http://127.0.0.1:7890"
    export https_proxy="http://127.0.0.1:7890"
    export all_proxy="socks5://127.0.0.1:7890"
    export HTTP_PROXY="http://127.0.0.1:7890"
    export HTTPS_PROXY="http://127.0.0.1:7890"
    export ALL_PROXY="socks5://127.0.0.1:7890"
    echo -e "\033[32m[+] Proxy enabled (127.0.0.1:7890)\033[0m"
}

proxy_off() {
    unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
    echo -e "\033[31m[-] Proxy disabled\033[0m"
}
EOF

```

> **说明**：如果使用的是 `zsh`，请将上述命令中的 `~/.bashrc` 替换为 `~/.zshrc`。

### Step 4. 生效配置

在服务器终端执行刷新命令：

```
source ~/.bashrc

```

### Step 5. 验证与日常使用

#### 1. 开启代理

```
proxy_on

```

终端将输出提示：`[+] Proxy enabled (127.0.0.1:7890)`。

#### 2. 测试连通性

```
curl -I https://www.google.com

```

若返回 `HTTP/2 200` 或 `HTTP/1.1 200 OK`，说明代理配置成功。

#### 3. 关闭代理

不需要使用时，运行以下命令即可恢复直连：

```
proxy_off

```

## 常见排错清单 (Troubleshooting)

| **报错现象** | **可能原因** | **解决办法** | 
| `bash: proxy_on: command not found` | 当前服务器的 `~/.bashrc` 未写入或未刷新 | 重新执行 **Step 3** 与 **Step 4** | 
| `curl: (7) Failed to connect ... Connection refused` | 1\. Windows 端 Clash 未启动  2\. SSH 隧道未建立或端口未映射 | 1\. 检查 Windows Clash 运行状态  2\. 检查 `~/.ssh/config` 是否包含 `RemoteForward`  3\. VS Code 重新载入窗口 | 
| VS Code 下方“端口 (Ports)”显示异常 | 转发方向配反或误点了手动转发 | 删除 Ports 面板里的冲突项，依赖 `ssh config` 自动转发 | 
| `Git clone` 依然很慢 | Git 未读取环境变量代理 | 运行：`git config --global http.proxy http://127.0.0.1:7890` | 
