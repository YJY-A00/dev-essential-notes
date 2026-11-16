# 第二次部署记录：开学后重新搭建网站

## 1. 本次实践背景

暑假首次使用的域名到期后，开学后我重新申请了云服务器，并重新安装、配置相关应用，再次搭建同一个 Go + MySQL + Nginx 个人网站。

这次实践沿用原有技术路线，但从一台全新的 Ubuntu 服务器开始重新准备环境。与第一次相比，本次记录更强调从系统初始化、基础工具安装到服务部署的完整过程。

## 2. 当前已确认的环境信息

| 项目 | 当前已知信息 |
| --- | --- |
| 云服务器 | 阿里云轻量应用服务器（具体规格待补充） |
| 操作系统 | Ubuntu（截图中未展示具体小版本；此前项目记录为 Ubuntu 24.04，建议登录服务器核实） |
| Web 服务 / 反向代理 | Nginx，通过 apt 安装 |
| 数据库 | MySQL Server，通过 apt 安装；目标版本为 MySQL 8.0 |
| 后端语言 | Go，截图采用 Go 1.23.4 Linux amd64 安装包作为示例 |
| Go 模块代理 | `https://goproxy.cn,direct` |
| 前端 | HTML / CSS / JavaScript（沿用原项目） |
| 项目架构 | Nginx + Go + MySQL |

> 版本说明：截图中的 Go 1.23.4 是文档命令所使用的版本示例。若服务器实际安装版本不同，应以 `go version` 输出为准。Ubuntu 的具体版本也请以 `lsb_release -a` 的结果为准。

## 3. 第二次部署过程

以下按截图中整理的实际操作顺序记录。命令中的用户名、数据库名和目录名属于示例时，应以服务器实际配置为准。

### 步骤 0：系统初始化与基础工具安装

拿到新的 Ubuntu 云服务器后，先更新软件源和已安装软件，再安装网络下载、版本管理、解压、编辑及防火墙工具。

```bash
# 更新软件源列表并升级已安装软件
sudo apt update && sudo apt upgrade -y

# 安装基础工具
sudo apt install -y curl wget git unzip vim ufw

# 放行 SSH、HTTP、HTTPS 端口
sudo ufw allow 22    # SSH
sudo ufw allow 80    # HTTP
sudo ufw allow 443   # HTTPS

# 启用 UFW 防火墙，并按提示确认
sudo ufw enable
```

启用防火墙前，应确认 SSH 的 22 端口已放行，避免把自己锁在服务器之外。云厂商安全组也需要允许相应入站端口。

### 步骤 1：安装并启动 Nginx

使用 Ubuntu 软件源安装 Nginx，并设置为开机启动、立即启动。

```bash
# 安装 Nginx
sudo apt install -y nginx

# 设置开机自启并立即启动
sudo systemctl enable --now nginx

# 检查 Nginx 运行状态
sudo systemctl status nginx
```

**验证方式：** 在浏览器中访问服务器公网 IP。如果出现 Nginx 默认欢迎页（通常显示 “Welcome to nginx!”），说明 Nginx 已能够对外提供默认页面。该验证只说明 Web 服务入口可访问，不代表网站前后端已经部署完成。

### 步骤 2：安装并初始化 MySQL

通过 apt 安装 MySQL Server，设置开机自启并检查服务状态。

```bash
# 安装 MySQL 服务端
sudo apt install -y mysql-server

# 设置开机自启并立即启动
sudo systemctl enable --now mysql

# 检查 MySQL 运行状态
sudo systemctl status mysql
```

随后进入 MySQL 命令行，创建项目数据库和仅供本机连接的应用账户，并授予该账户项目数据库的权限。

```sql
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'myapp'@'localhost' IDENTIFIED BY '请替换为强密码';

GRANT ALL PRIVILEGES ON myapp.* TO 'myapp'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

说明：

- `myapp` 是截图中的示例数据库名和应用用户名；如实际项目使用其他名称，应保持 Go 配置与 MySQL 配置一致。
- 密码必须替换为独立的强密码。**不要把真实密码、可用密码或包含真实凭据的命令提交到 GitHub。**
- 截图中特别提醒，数据库密码使用纯英文和数字组合，避免特殊字符导致连接字符串解析问题。更稳妥的做法是使用正确转义或参数化的 DSN 配置；不要为了绕开解析问题而降低密码强度。
- 应用账户使用 `localhost`，体现数据库仅供本机后端连接的配置思路；不要无必要地向公网开放 MySQL 端口。

### 步骤 3：安装 Go 后端语言环境

截图采用 Go 官方 Linux amd64 二进制压缩包安装方式，而不是直接使用 Ubuntu 软件源中的 Go 包。示例版本为 Go 1.23.4。

```bash
# 进入临时目录并下载 Go 安装包（示例版本：1.23.4）
cd /tmp
wget https://go.dev/dl/go1.23.4.linux-amd64.tar.gz

# 如果 /usr/local/go 已存在，先确认后再删除旧安装
sudo rm -rf /usr/local/go

# 解压到 /usr/local
sudo tar -C /usr/local -xzf go1.23.4.linux-amd64.tar.gz

# 建立全局命令软链接
sudo ln -sf /usr/local/go/bin/go /usr/local/bin/go
sudo ln -sf /usr/local/go/bin/gofmt /usr/local/bin/gofmt

# 验证 Go 是否安装成功
go version
```

> 注意：`sudo rm -rf /usr/local/go` 会删除该目录下已有的 Go 安装。只有确认该目录确实用于旧版 Go、且不需要保留时才执行。下载链接中的版本号和架构也应与实际目标环境匹配。

为解决国内服务器拉取 Go 模块依赖时连接默认代理超时的问题，配置 Go 模块代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct

# 查看配置结果
go env GOPROXY
```

截图中使用的是七牛云国内 Go 模块代理地址。配置后，Go 获取模块依赖时会优先使用该代理，并在需要时按配置回退到 direct。

### 步骤 4：上传 Go 项目代码到服务器

截图给出了两种方式：本地通过 `scp` 直接上传，或先将代码推送到 GitHub/Gitee，再在服务器上克隆。示例服务器 IP 为 `8.133.191.82`，项目目录为 `/opt/myapp`；这些是截图中的示例值，实际使用时应替换为当前服务器信息。

**方式一：使用 scp 从本地上传**

先在服务器上创建项目目录：

```bash
ssh root@8.133.191.82 "mkdir -p /opt/myapp"
```

再在本地 Windows PowerShell 或其他终端中执行：

```bash
scp -r ./你的本地Go项目目录/* root@8.133.191.82:/opt/myapp/
```

**方式二：通过 Git 克隆**

先在本地将代码推送到 GitHub 或 Gitee，再登录服务器执行：

```bash
cd /opt
git clone https://github.com/你的用户名/你的仓库名.git myapp
cd myapp
```

如果仓库是私有仓库，需要先配置服务器的 Git 身份验证方式。不要把访问令牌或私钥写入公开文档。

### 步骤 5：编译 Go 后端

进入项目目录，整理依赖并编译 Linux 可执行文件。截图中的输出文件名为 `myapp`。

```bash
cd /opt/myapp

# 整理并下载项目依赖
go mod tidy

# 编译为 Linux 可执行文件
go build -o myapp .

# 添加可执行权限
chmod +x myapp

# 按截图将项目目录所有者设置为 www-data
sudo chown -R www-data:www-data /opt/myapp
```

> 编译前应确认服务器 Go 环境与项目依赖兼容，且项目入口包位于当前目录（否则需调整 `go build` 的目标路径）。截图记录了将整个目录交给 `www-data` 的做法；生产环境还应遵循最小权限原则，避免让 Web 服务账户拥有不必要的源码或配置写权限。

### 步骤 6：使用 Systemd 管理 Go 后端

截图通过 Systemd 将 Go 程序配置为后台服务，使其能够在系统启动时运行，并在进程异常退出时自动重启。

创建服务文件：

```bash
sudo nano /etc/systemd/system/myapp.service
```

写入以下配置：

```ini
[Unit]
Description=Go Web App
After=network.target mysql.service

[Service]
User=www-data
Group=www-data
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/myapp
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

保存后执行：

```bash
# 重新加载 Systemd 配置
sudo systemctl daemon-reload

# 设置开机自启
sudo systemctl enable myapp

# 启动服务
sudo systemctl start myapp

# 查看服务状态
sudo systemctl status myapp
```

**验证方式：** 查看服务状态，确认出现 `active (running)`。如果启动失败，可使用以下命令查看服务日志：

```bash
sudo journalctl -u myapp -n 100 --no-pager
```

> `After=mysql.service` 只表示启动顺序，不等于数据库连接成功，也不代表应用具备数据库就绪重试能力。实际运行时仍需检查 Go 程序的数据库连接配置、启动日志和接口响应。

### 步骤 7：配置 Nginx 静态资源与 API 反向代理

截图采用的架构是：Nginx 直接提供前端静态文件，将以 `/api/` 开头的请求转发给监听在本机 `127.0.0.1:8080` 的 Go 后端。

创建站点配置文件：

```bash
sudo nano /etc/nginx/sites-available/myapp
```

截图中的配置示例如下：

```nginx
server {
    listen 80;
    server_name 你的新域名 www.你的新域名;

    # 前端静态资源目录
    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 请求反向代理到 Go 后端
    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

启用站点并检查配置：

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/myapp

# 移除默认站点（仅在确认不再需要时执行）
sudo rm -f /etc/nginx/sites-enabled/default

# 检查 Nginx 配置语法
sudo nginx -t

# 配置检查通过后平滑重载
sudo systemctl reload nginx
```

注意事项：

- `root /var/www/myapp` 是前端静态文件目录，需要先将前端文件部署到该目录；它与后端项目目录 `/opt/myapp` 不同。
- `proxy_pass` 的端口必须与 Go 程序实际监听端口一致。截图使用的是 `8080`。
- Go 后端应监听本机回环地址（例如 `127.0.0.1:8080`），避免在不需要时直接暴露后端端口。
- `server_name` 应替换为当前真实域名，不要继续沿用已经过期的旧域名。
- 如果前端不是单页应用，需根据实际路由调整 `try_files` 规则。

### 步骤 8：绑定新域名并配置 HTTPS

截图给出的前提是：域名已完成 ICP 备案，并且已在 DNS 服务商处添加 A 记录，解析到当前服务器公网 IP。域名和服务器信息应以本次实际使用的为准。

**第一步：修改 Nginx 域名配置**

将站点配置中的 `server_name` 改为新域名，例如：

```nginx
server_name yourdomain.com www.yourdomain.com;
```

修改后检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**第二步：安装 Certbot 和 Nginx 插件**

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

**第三步：申请证书并让 Certbot 配置 Nginx**

将命令中的示例域名替换为自己的真实域名：

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

按提示填写邮箱、同意服务条款，并根据实际需求选择是否将 HTTP 自动重定向到 HTTPS。申请前应确认域名 DNS 已正确解析到该服务器，且公网 80/443 端口可访问。

**第四步：测试证书自动续期**

```bash
sudo certbot renew --dry-run
```

截图说明 Let's Encrypt 证书有效期为 90 天，Certbot 可通过定时任务自动尝试续期。执行 dry-run 是对续期流程的测试，不等于已经验证未来每次续期都必然成功。

## 4. 关键配置关系

本次部署截图体现的请求链路如下：

```text
用户浏览器
    |
    | HTTP / HTTPS
    v
Nginx（80 / 443）
    |
    |-- 静态页面请求 --> /var/www/myapp
    |
    |-- /api/ 请求 --> 127.0.0.1:8080
                           |
                           v
                       Go 后端
                           |
                           v
                       MySQL
```

其中，截图中的后端服务目录为 `/opt/myapp`，前端静态目录为 `/var/www/myapp`，Go 服务端口为 `8080`。这些值属于本次操作说明中的配置示例，最终应以服务器实际配置和验证结果为准。

## 5. 上线验收清单

- [ ] SSH 可以正常连接，防火墙规则不会阻断管理连接。
- [ ] Nginx 服务正常运行，公网 IP 可以访问默认页或已部署站点。
- [ ] MySQL 服务正常运行，项目应用账户可以连接目标数据库。
- [ ] `go version` 显示预期的 Go 版本。
- [ ] `go env GOPROXY` 显示预期的模块代理配置。
- [ ] Go 项目代码已上传或克隆，依赖整理和编译成功。
- [ ] Systemd 服务状态为 `active (running)`，并能查看运行日志。
- [ ] 前端文件已部署到 Nginx 配置的静态目录。
- [ ] Nginx 配置检查通过，API 反向代理能够访问 Go 后端。
- [ ] 新域名 DNS 解析指向当前服务器。
- [ ] HTTPS 证书申请成功，浏览器访问无证书错误。
- [ ] `certbot renew --dry-run` 测试通过。
- [ ] 服务器重启后，相关服务能够按预期恢复。

## 6. 与第一次部署的对比

| 对比维度 | 第一次部署 | 第二次部署 |
| --- | --- | --- |
| 服务器 | 暑假使用的旧服务器环境 | 重新申请的云服务器 |
| 系统准备 | 以第一次部署记录为准 | 从 Ubuntu 基础环境重新初始化 |
| Nginx | 完成网站 Web 服务配置 | 使用 apt 安装并设置开机自启 |
| MySQL | 完成项目数据库配置 | 使用 apt 安装 MySQL Server 并创建项目数据库账户 |
| Go | 完成后端运行环境配置 | 使用 Go 官方 Linux amd64 压缩包安装，并配置模块代理 |
| 项目上传 | 以第一次部署记录为准 | 记录 scp 上传和 Git 克隆两种方式 |
| 后端进程 | 以第一次部署记录为准 | 使用 Systemd 管理 Go 程序 |
| 前后端联调 | 以第一次部署记录为准 | Nginx 静态资源托管并将 /api/ 反向代理到 Go 服务 |
| 域名 | 暑假申请的域名，后来到期 | 重新申请域名，具体信息待补充 |
| 备案与 HTTPS | 以第一次部署记录为准 | 记录 DNS、Certbot 证书申请与续期测试流程 |

## 7. 本次复盘

第二次部署从一台新的 Ubuntu 服务器开始，重新完成系统更新、基础工具安装、防火墙规则配置，以及 Nginx、MySQL 和 Go 运行环境的准备；随后将 Go 项目部署到服务器，编译为 Linux 可执行文件，并通过 Systemd 管理后端进程；最后使用 Nginx 提供静态资源和 API 反向代理，并通过 Certbot 申请 HTTPS 证书。

这份记录不仅保留了部署命令，也梳理了各组件之间的关系：Nginx 作为统一 Web 入口，Go 负责后端 API 逻辑，MySQL 保存业务数据，Systemd 负责后端进程管理，Certbot 协助管理 HTTPS 证书。

需要说明的是，本文根据操作截图整理，命令和配置中的 IP、域名、路径、用户名、端口均应以实际服务器为准。正式用于项目展示前，还应补充真实的服务状态、接口访问、HTTPS 访问和重启验证结果，避免将操作指南误写成未经验证的上线结论。
