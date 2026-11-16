# 网站架构与部署验证说明

## 1. 项目架构

当前项目采用 Nginx + Go + MySQL 的基本架构：

```text
浏览器
  |
  | HTTP / HTTPS
  v
Nginx
  |------------------------------|
  |                              |
  v                              v
前端静态资源                    /api/ 请求
HTML / CSS / JavaScript          |
                                 v
                              Go 后端
                                 |
                                 v
                              MySQL
```

## 2. 各组件职责

### Nginx

- 作为 Web 请求入口。
- 根据站点配置提供前端静态文件。
- 将符合规则的 API 请求转发给 Go 服务。
- 可负责 HTTPS 终止和 HTTP 到 HTTPS 跳转（以实际配置为准）。

### Go 后端

- 接收并处理 API 请求。
- 执行业务逻辑。
- 通过 MySQL 驱动访问数据库。
- 应以受控方式运行，避免直接将数据库端口暴露到公网。

### MySQL

- 保存网站业务数据。
- 由 Go 后端按需访问。
- 应设置合理的账户权限，并建立备份与恢复方案。

### 前端

- 使用 HTML、CSS 和 JavaScript。
- 静态文件由 Nginx 提供。
- 通过 API 与 Go 后端交互。

## 3. 部署时需要记录的关键配置

| 配置项 | 需要记录的内容 |
| --- | --- |
| Nginx 静态目录 | 实际 root 路径 |
| Nginx 站点配置 | 实际配置文件路径 |
| API 路径 | 实际 location 规则 |
| Go 服务 | 可执行文件路径、运行用户、监听地址 |
| Systemd | 服务单元名称与关键配置 |
| MySQL | 版本、数据库名、连接方式（不记录密码） |
| 域名 | DNS 解析记录（公开前检查隐私） |
| HTTPS | 证书工具、证书路径、续期验证方式 |

## 4. 部署验证建议

### 检查系统与服务

```bash
# 查看 Ubuntu 版本
lsb_release -a

# 查看 Nginx 状态
sudo systemctl status nginx

# 查看 MySQL 状态
sudo systemctl status mysql

# 查看 Go 服务状态（将 myapp 替换为实际服务名）
sudo systemctl status myapp
```

### 检查 Nginx 配置

```bash
sudo nginx -t
```

只有配置检查通过后，再按实际情况重载 Nginx：

```bash
sudo systemctl reload nginx
```

### 检查端口监听

```bash
sudo ss -lntp
```

结合实际服务监听地址和端口判断服务是否启动。不要直接照搬示例端口，先核对配置文件。

### 检查日志

```bash
# Nginx 错误日志（常见默认路径，实际以配置为准）
sudo tail -n 100 /var/log/nginx/error.log

# Systemd 服务日志
sudo journalctl -u myapp -n 100 --no-pager
```

## 5. 安全与隐私检查

- 不向 GitHub 提交密码、私钥、Token、数据库凭据或真实环境变量。
- 不公开 SSH 私钥、云平台密钥和数据库备份。
- 不为方便调试而将 MySQL 端口无必要地暴露到公网。
- 防火墙只开放业务需要的端口。
- 公开备案或服务器截图前，遮挡个人身份信息、手机号、订单号、IP（如不希望公开）及控制台敏感字段。
- 在公开配置文件前检查是否包含真实域名、邮箱、路径或其他个人信息。

## 6. 复试展示建议

展示项目时，可以按以下顺序讲解：

1. 项目目标：为什么要搭建个人网站。
2. 技术选型：为什么使用 Nginx、Go 和 MySQL。
3. 架构关系：浏览器请求如何经过 Nginx 到达 Go，再访问 MySQL。
4. 部署过程：服务器准备、应用部署、域名与 HTTPS。
5. 问题解决：选择一到两个自己真正排查过的问题，说明现象、分析、验证。
6. 反思改进：第二次部署相比第一次有哪些改进，仍有哪些不足。

讲解时应以自己实际操作和能够解释清楚的配置为准，不要把尚未实现的功能或未验证的配置描述为项目成果。
