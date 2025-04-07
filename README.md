# Shadowsocks Rust 一键安装与优化脚本

本仓库提供一个用于安装、配置和优化 Shadowsocks Rust 的自动化脚本，适用于 Linux 系统，支持运行时手动输入端口、密码和加密方式。

---

## 功能

- **自动安装**：一键安装 Shadowsocks Rust。
- **动态配置**：运行脚本时手动输入端口、密码和加密方式。
- **系统优化**：自动启用 TCP BBR 拥塞控制，优化网络性能。
- **服务管理**：使用 `systemd` 管理 Shadowsocks 服务。
- **链接生成**：安装完成后自动生成 `ss://` 分享链接，方便客户端导入。

---

## 使用方法

### 一键运行脚本
```bash
bash <(curl -sSL https://raw.githubusercontent.com/lxiaolong068/shadowsocks-rust-install/main/shadowsocks-rust-install.sh)
```

### 1. 下载脚本
克隆仓库到本地：
```bash
git clone https://github.com/lxiaolong068/shadowsocks-rust-install.git
cd shadowsocks-rust-install
```

或直接下载脚本：
```bash
wget https://raw.githubusercontent.com/lxiaolong068/shadowsocks-rust-install/main/shadowsocks-rust-install.sh
```

### 2. 赋予执行权限
```bash
chmod +x shadowsocks-rust-install.sh
```

### 3. 运行脚本
```bash
sudo ./shadowsocks-rust-install.sh
```

### 4. 输入配置信息
按照提示输入以下内容：
- **监听端口**（默认 `8388`）：Shadowsocks 服务的端口。
- **密码**（默认 `yourpassword`）：用于连接的密码。
- **加密方式**（默认 `aes-128-gcm`）：脚本内部固定使用 `aes-128-gcm`，暂不支持运行时选择。

---

## 示例

运行脚本时的示例交互：
```
请输入 Shadowsocks-Rust 监听端口（默认: 8388）: 
请输入 Shadowsocks-Rust 密码（默认: yourpassword）: mysecretpassword
... [安装过程日志] ...
==> 获取公网 IP 地址...
===================================================================
Shadowsocks-Rust 安装完成，并已设置为 systemd 开机自启。
端口:          8388
密码:          mysecretpassword
加密方式:       aes-128-gcm
配置文件:       /etc/shadowsocks-rust/config.json

SS 链接:        ss://YWVzLTEyOC1nY206bXlzZWNyZXRwYXNzd29yZA==@YOUR_SERVER_IP:8388#shadowsocks_rust

BBR 已配置完成，查看是否生效:
  sysctl net.ipv4.tcp_congestion_control
  lsmod | grep bbr
===================================================================
如需再次修改 Shadowsocks-Rust 配置，请编辑: /etc/shadowsocks-rust/config.json
修改完成后，使用 systemctl restart shadowsocks-rust.service 重启生效。
脚本执行完毕，祝使用愉快！
```
*(注意：示例中的 `YOUR_SERVER_IP` 会被替换为您的服务器实际公网 IP 地址。)*

运行成功后，服务将自动启动，并使用 `systemd` 管理。脚本末尾会显示生成的 `ss://` 链接，您可以直接复制该链接到支持的 Shadowsocks 客户端中使用。

---

## 查看 SS 链接

如果您忘记了安装时生成的 `ss://` 链接，可以随时登录到服务器，运行以下命令重新生成并查看：

**前提条件:** 服务器需要安装 `jq` (用于解析 JSON) 和 `curl` (用于获取 IP)。如果未安装 `jq`，请先运行 `sudo apt update && sudo apt install jq -y`。

**查看链接命令:**
```bash
# 读取配置
CONFIG_FILE="/etc/shadowsocks-rust/config.json"
SS_PORT=$(jq -r '.server_port' $CONFIG_FILE)
SS_PASSWORD=$(jq -r '.password' $CONFIG_FILE)
SS_METHOD=$(jq -r '.method' $CONFIG_FILE)

# 获取公网 IP
PUBLIC_IP=$(curl -s https://api.ipify.org || curl -s ifconfig.me || echo "无法获取公网IP")

# 检查是否成功获取 IP
if [[ "$PUBLIC_IP" == "无法获取公网IP" ]]; then
  echo "错误：无法获取公网 IP 地址，无法生成链接。"
else
  # 生成 Base64 部分 (移除可能存在的换行符)
  BASE64_PART=$(echo -n "${SS_METHOD}:${SS_PASSWORD}" | base64)
  # 生成并输出链接 (使用脚本默认的 tag)
  SS_LINK="ss://${BASE64_PART}@${PUBLIC_IP}:${SS_PORT}#shadowsocks_rust"
  echo "您的 Shadowsocks 链接是:"
  echo $SS_LINK
fi
```
将以上整段命令复制粘贴到服务器终端运行即可。

---

## 系统优化

脚本会自动启用以下网络优化：
- 设置队列调度算法为 `fq`。
- 启用 TCP BBR 拥塞控制。

验证优化是否生效：
```bash
sysctl net.ipv4.tcp_available_congestion_control
lsmod | grep bbr
```

---

## 卸载说明

如果您需要卸载通过此脚本安装的 Shadowsocks-Rust 及相关配置，请按以下步骤操作：

1.  **停止并禁用服务**
    ```bash
    sudo systemctl stop shadowsocks-rust
    sudo systemctl disable shadowsocks-rust
    ```

2.  **删除 Systemd 服务文件**
    ```bash
    sudo rm /etc/systemd/system/shadowsocks-rust.service
    sudo systemctl daemon-reload
    ```

3.  **删除配置文件目录**
    ```bash
    sudo rm -rf /etc/shadowsocks-rust
    ```

4.  **卸载 Shadowsocks-Rust**
    *   删除符号链接（如果存在）：
        ```bash
        # 忽略可能出现的 "No such file or directory" 错误
        sudo rm /usr/local/bin/ssserver /usr/local/bin/sslocal /usr/local/bin/ssurl /usr/local/bin/ssmanager 2>/dev/null
        ```
    *   使用 Cargo 卸载（需要以安装时的用户身份运行）：
        ```bash
        ~/.cargo/bin/cargo uninstall shadowsocks-rust
        ```

5.  **（可选）恢复网络配置**
    *   如果您希望撤销 BBR 优化，请编辑 `/etc/sysctl.conf` 文件，删除或注释掉以下两行：
        ```
        net.core.default_qdisc = fq
        net.ipv4.tcp_congestion_control = bbr
        ```
    *   保存文件后，运行 `sudo sysctl -p` 应用更改。

6.  **（可选）卸载 Rust 工具链**
    *   如果不再需要 Rust，可以运行：
        ```bash
        rustup self uninstall
        ```

---

## 注意事项

- 请确保系统为 **Ubuntu 20.04** 或更高版本。
- 需要 `curl`、`build-essential`、`pkg-config`、`libssl-dev`、`jq` 等工具，脚本会自动尝试安装部分依赖，`jq` 可能需要手动安装。
- 脚本将修改系统的网络参数 (`/etc/sysctl.conf`) 以启用 BBR。
- 脚本会尝试通过 `https://api.ipify.org` 或 `ifconfig.me` 获取公网 IP 以生成 `ss://` 链接。如果服务器无法访问这些服务，链接将无法生成。

---

## 开源协议

本项目基于 [MIT License](LICENSE) 开源，欢迎自由使用和修改。
