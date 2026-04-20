# Proxmox WHMCS对接模块

**安装、配置和使用指南**

开源 WHMCS 模块，用于自动化 VPS 部署

---

## 概览

- 将模块文件夹安装到 `modules/servers/`
- 使用 Proxmox root API 令牌（令牌 ID + 密钥）进行服务器身份验证
- 配置产品选项：CPU、RAM、磁盘大小、模板 VMID 和网络限速
- 在服务器的"分配的 IP 地址"字段中逐行定义 IP/MAC 池
- 自动化：创建、暂停和终止操作都是端到端处理
- 客户可以从 WHMCS 启动/停止/重启； VNC 控制台(可选的)通过反向代理启用

---


---

## 1. 安装

解压下载的压缩包

将 `proxmox_custom/` 文件夹上传到您的 WHMCS 目录：
   ```
   /path/to/whmcs/modules/servers/
   ```

---

## 2. 服务器凭据

在 WHMCS 中添加 Proxmox 服务器（**设置 → 服务器**）时，使用 Proxmox root 用户的 API 令牌进行身份验证（不进行权限分离）。

| WHMCS 字段 | 输入内容 |
|---|---|
| **主机名** | 您的 Proxmox 主机名（例如 `pve001.example.com`） |
| **用户名** | 令牌 ID（例如 `root@pam!whmcs`） |
| **密码** | 令牌密钥 |

### 在 Proxmox 上创建 API 令牌

```bash
pveum user token add root@pam whmcs --privsep=0
```

> [!WARNING]
> 使用专用的 API 令牌，并将密钥视为密码。`--privsep=0` 赋予令牌与 root 用户相同的权限。对于生产环境，请考虑创建具有最小权限的专用用户和权限分离的令牌。

---

## 3. 产品配置（模块设置）

在 WHMCS 产品设置中，配置以下选项：

| 选项 | 描述 |
|---|---|
| **CPU核心数** | CPU 核心数 |
| **RAM** | 内存大小（GB） |
| **磁盘大小** | 磁盘大小 GB。必须 ≥ 模板大小（建议最小 10 GB） |
| **模板ID** | Proxmox 上基础模板的 VMID |
| **网络速度** | 网络速率限制 MB/s（应用于网络适配器） |
| **节点** | Proxmox 节点名称（例如 `pve001`） |
| **主机名后缀** | VM 主机名的域名后缀（例如 `.vps.example.com`） |
| **开启控制台** | 在客户区显示 VNC 控制台按钮（`on` / `off`） |

---

## 4. IP、MAC 和网络配置

在 WHMCS 服务器设置中直接定义可用的 MAC 地址、公共 IP、网络桥接和 MTU 池。

在**分配的 IP 地址**字段中，每行使用以下格式输入一个配置：

```
[MAC 地址]=[公共 IP];[桥接],[MTU]
```

**示例：**

```
bc:24:11:23:0e:c2=181.13.218.180;vmbr0,1250
52:54:00:a6:6e:5b=200.89.174.82;vmbr0,1250
```

### 工作原理

模块使用带静态映射的 DHCP（配置为拒绝未知客户端）为 VM 分配 IP。IP 不是由模块在服务器操作系统内部强制配置的；相反，DHCP 服务器根据在 Proxmox 中配置的 MAC 地址分配正确的 IP。

---

## 5. 自动化模块操作

此模块自动化完整的 VPS 生命周期：

| 事件 | 模块执行的操作 |
|---|---|
| **创建时** | 创建与 WHMCS 用户 ID 匹配的 Proxmox 用户；克隆模板；配置 cloud-init；设置硬件参数（MAC、磁盘大小、RAM、网络速率）；并授予用户权限 |
| **暂停时** | 停止 VM 并撤销用户权限 |
| **终止时** | 永久删除 VM 和 Proxmox 用户 |

---

## 6. 客户区和访问

### 快捷操作

客户可以从 WHMCS 客户区安全地**启动**、**停止**、**重启**和（可选）**重装**他们的 VPS。客户区还显示：

- 实时 VM 状态（运行中/已停止）
- CPU 和 RAM 使用率
- 性能图表（CPU、内存、网络、磁盘 I/O）

如果不需要 VNC 控制台访问，提供 SSH 访问以及这些按钮通常就足够了。

### 邮件模板

创建自定义 WHMCS 欢迎邮件模板，在完成时向客户发送服务器访问凭据。

---

## 7. 网络架构

### 不使用 VNC 控制台（更简单的设置）

如果**不需要 VNC 控制台访问**，只需要 WHMCS 服务器能够访问 Proxmox。可以通过以下方式实现：

- WHMCS 和 Proxmox 之间的专用网络 / VPN
- 直接连接端口 8006
- 内部反向代理

```
┌──────────┐         ┌──────────────┐
│  WHMCS   │────────▶│  Proxmox     │
│  服务器  │  :8006  │  (仅 API)    │
└──────────┘         └──────────────┘
    客户不需要直接访问 Proxmox。
```

### 使用 VNC 控制台（需要反向代理）

如果**想为客户提供 VNC 控制台访问**，能够通过浏览器访问 Proxmox Web 界面。由于 Proxmox 在端口 8006 上运行并使用自签名证书，所以**反向代理**是必需的：

```
┌──────────┐         ┌───────────────┐         ┌──────────────┐
│  客户    │────────▶│ 反向代理      │────────▶│  Proxmox     │
│  浏览器  │  :443   │ (Nginx/Caddy)│  :8006  │  节点        │
└──────────┘         └───────────────┘         └──────────────┘
                      vps.example.com
```

> [!IMPORTANT]
> 反向代理 主机名（例如 `vps.example.com`）必须与 WHMCS 中配置的**服务器主机名**匹配。这是用于 API 调用和控制台 URL 的主机名。

---

## 8. VNC 控制台设置

### 要求

1. **客户可以通过**反向代理访问 Proxmox（如上所述）
2. **`console-login.html`** 在与 Proxmox 反向代理**相同的域名**上提供服务
3. 模块设置中 **EnableConsole** 设置为 `on`

### 为什么需要 `console-login.html`

当客户点击"控制台"时，模块：

1. 在**服务器端**（PHP）与 Proxmox 进行身份验证并获取认证票据
2. 将客户重定向到 Proxmox 域名上的 `console-login.html`
3. 该页面通过 JavaScript 设置 `PVEAuthCookie`（同域名，浏览器允许）
4. 重定向到 Proxmox 内置的 noVNC 控制台

这是必要的，因为浏览器阻止为不同域名设置 cookie。WHMCS 服务器（例如 `example.com`）不能为 Proxmox 域名（例如 `vps.example.com`）设置 cookie。辅助页面通过在 Proxmox 域名本身上运行来弥合这一差距。

### 部署 `console-login.html`

该文件必须可以通过与 Proxmox 反向代理**相同的域名**上的 URL 访问。默认路径为：

```
https://vps.example.com/consolevnc/console-login.html
```

> [!TIP]
> 您可以在 `proxmox_custom.php` 中搜索 `consolevnc` 并更新 URL 来更改此路径。

将文件放在您的**反向代理**服务器上，而不是 Proxmox 服务器本身。

---

### Nginx 示例

```nginx
server {
    listen 443 ssl;
    server_name vps.example.com;

    # 提供控制台登录辅助文件
    location /consolevnc/ {
        alias /var/www/consolevnc/;
    }

    # 将其他所有内容代理到 Proxmox
    location / {
        proxy_pass https://vps.example.com:8006;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

然后放置文件：

```bash
mkdir -p /var/www/consolevnc/
cp console-login.html /var/www/consolevnc/
```

---

### Caddy 示例


```
vps.example.com {
    handle_path /consolevnc/* {
        root * /var/www/consolevnc/
        file_server
    }
    reverse_proxy https://vps.example.com:8006 {

        transport http {
            tls_insecure_skip_verify
        }
    }
}
```
然后放置文件：

```bash
mkdir -p /var/www/consolevnc/
cp console-login.html /var/www/consolevnc/
```

---

## 9. 故障排除

### 控制台显示"401 未授权"

`PVEAuthCookie` 未被设置。验证：

- `console-login.html` 可以在 `https://<proxmox主机名>/consolevnc/console-login.html` 访问
- URL 域名与 WHMCS 中配置的**服务器主机名**匹配
- `EnableConsole` 设置为 `on`

### VM provisioning 失败

检查 WHMCS**模块日志**（实用工具 → 日志 → 模块日志）以获取详细的错误消息。常见问题：

- API 令牌权限不足
- 模板 VMID 不存在
- 节点名称不匹配
- IP/MAC 池未在服务器设置中配置

### VM 被意外删除

如果从此模块的先前版本升级，确保没有遗留的异步 provisioning 钩子：

- 从模块目录中删除 `hooks.php`（如果存在）
- 从模块目录中删除 `hooks/` 文件夹（如果存在）
- 从 WHMCS `includes/hooks/` 中删除任何 `proxmox_async_provisioning.php` 副本
- 如果存在，清理 `mod_proxmox_tasks` 表：`DELETE FROM mod_proxmox_tasks;`

### 自定义

此模块是 100% 开源的。您可以自由修改代码以适应您的工作流程。

---