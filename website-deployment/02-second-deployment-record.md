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
ufw allow 22    # SSH
ufw allow 80    # HTTP
ufw allow 443   # HTTPS

# 启用 UFW 防火墙，并按提示确认
ufw enable
```

如果当前登录用户不是 root，执行 UFW 命令时需要使用 `sudo ufw ...`。启用防火墙前，应确认 SSH 的 22 端口已放行，避免把自己锁在服务器之外。

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

## 4. 后续部署步骤（待补充实际操作）

当前截图覆盖了系统初始化、Nginx、MySQL 和 Go 环境安装。以下步骤尚未由本次截图证明，后续应根据真实操作记录补充，不将其视为已完成：

### 4.1 Go 项目代码与依赖

- 项目代码如何上传或克隆到服务器：待补充。
- `go mod tidy` 或实际依赖安装命令：待补充。
- Go 编译命令、二进制文件路径：待补充。
- 后端实际监听地址和端口：待补充。

### 4.2 数据库连接与业务数据

- 实际数据库名、表结构和初始化方式：待补充。
- Go 后端读取数据库连接配置的方式：待补充。
- 数据备份和恢复方式：待补充。

### 4.3 Systemd 服务管理

- 实际 service 单元名称和配置：待补充。
- 服务启动、开机自启和日志检查结果：待补充。

### 4.4 Nginx 站点配置与前后端联调

- 前端静态文件部署目录：待补充。
- Nginx 站点配置文件路径：待补充。
- 静态资源与 API 反向代理规则：待补充。
- `nginx -t` 检查结果及重载记录：待补充。

### 4.5 域名、备案与 HTTPS

- 新域名注册商、DNS 解析记录：待补充。
- ICP 备案办理过程和状态：待补充。
- HTTPS 证书申请、自动续期和验证：待补充。

## 5. 上线验收清单

- [ ] SSH 可以正常连接，防火墙规则不会阻断管理连接。
- [ ] Nginx 服务正常运行，公网 IP 可以访问默认页或已部署站点。
- [ ] MySQL 服务正常运行，项目应用账户可以连接目标数据库。
- [ ] `go version` 显示预期的 Go 版本。
- [ ] `go env GOPROXY` 显示预期的模块代理配置。
- [ ] Go 后端成功编译并由服务管理器正常运行。
- [ ] Nginx 静态资源和 API 反向代理配置检查通过。
- [ ] 域名解析指向当前服务器。
- [ ] HTTPS 证书有效（如已启用）。
- [ ] 服务器重启后，相关服务能够按预期恢复。

## 6. 与第一次部署的对比

| 对比维度 | 第一次部署 | 第二次部署 |
| --- | --- | --- |
| 服务器 | 暑假使用的旧服务器环境 | 重新申请的云服务器 |
| 系统准备 | 以第一次部署记录为准 | 从 Ubuntu 基础环境重新初始化 |
| Nginx | 完成网站 Web 服务配置 | 使用 apt 安装并设置开机自启 |
| MySQL | 完成项目数据库配置 | 使用 apt 安装 MySQL Server 并创建项目数据库账户 |
| Go | 完成后端运行环境配置 | 使用 Go 官方 Linux amd64 压缩包安装，并配置模块代理 |
| 防火墙 | 以第一次部署记录为准 | 配置 UFW，放行 SSH、HTTP、HTTPS |
| 域名 | 暑假申请的域名，后来到期 | 重新申请域名，具体信息待补充 |
| 备案与 HTTPS | 以第一次部署记录为准 | 以第二次实际办理记录为准 |

## 7. 本次复盘

第二次部署从一台新的 Ubuntu 服务器开始，重新完成系统更新、基础工具安装、防火墙规则配置，以及 Nginx、MySQL 和 Go 运行环境的准备。与仅记录最终架构相比，这份过程记录能够展示我如何从操作系统层面逐步搭建 Web 服务运行环境。

后续还需要补充 Go 项目部署、数据库连接、Systemd 服务、Nginx 站点配置、域名解析、备案和 HTTPS 等环节的实际命令与验证结果。完成这些内容后，再总结两次部署在操作熟练度、环境配置和故障排查方面的具体变化。
