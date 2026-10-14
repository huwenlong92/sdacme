# SDACME

轻量、自托管的 SSL 证书管理工具。提供独立命令行客户端和带 Web 界面的管理服务，支持证书签发、续期、下载、部署和多机器证书状态上报。

[下载 v0.2.1](https://github.com/huwenlong92/sdacme/releases/tag/v0.2.1) · [所有版本](https://github.com/huwenlong92/sdacme/releases)

## 两个程序怎么选

| 程序 | 用途 | 运行位置 |
| --- | --- | --- |
| `sdacmectl` | 独立签发、续期、导出、部署证书；可向管理端上报 | 需要管理证书的机器 |
| `sdacme` | 管理服务与内嵌 Web；管理员登录、节点管理、中心本机证书任务 | 管理服务器 |

只需要命令行管理证书时，安装 `sdacmectl` 即可。需要 Web 界面时安装 `sdacme`；若管理端也要签发证书，再配置其本机 controller。远程客户端独立续期，中心当前只接收公共证书元数据，不上传私钥、不下发远程任务。

## 功能

- **证书申请**：HTTP-01、手动 DNS、自动 DNS、CNAME 委托；支持根域、多 SAN 和泛域名。
- **DNS 账号**：Cloudflare、阿里云 DNS、腾讯云 DNSPod；CNAME 固定委托目标，续期沿用最新凭据。
- **证书管理**：ECC P-256、RSA 2048/3072/4096；查看实际 SAN、有效期、签发机构与证书组织信息。
- **下载与部署**：PEM tar.gz、密码保护的 PFX、明确文件路径安装；阿里云 CDN/DCDN、七牛、腾讯云 CDN/EdgeOne。
- **续期与记录**：ctl cron、管理端本机定时续期、申请记录；自动 DNS 的 TXT 精确值清理和失败重试。所有续期由 sdacme 自己调度。
- **Web 界面**：总览、证书、自动部署、操作记录、帮助文档、亮暗主题；远程节点状态和证书列表。

当前内置申请入口使用 Let’s Encrypt DV。多 CA、付费 OV、ZIP/JKS 导出与远程任务下发尚未实现。云厂商适配已具备，真实账号上传和边缘传播仍需按目标环境验收。

## 下载与安装

当前发行包为 **Linux x86_64 / amd64**，需要 glibc ≥ 2.34 与 libgcc；已在 Debian 12/13 验证。证书操作需要系统 curl、OpenSSL，运行环境由 sdacme 自动准备；管理服务安装器需要 root 和正在运行的 systemd。目标机不需要 Rust 或 Node。

| 安装包 | 校验和 |
| --- | --- |
| [sdacmectl-linux-amd64.tar.gz](https://github.com/huwenlong92/sdacme/releases/download/v0.2.1/sdacmectl-linux-amd64.tar.gz) | [checksums-linux-amd64.txt](https://github.com/huwenlong92/sdacme/releases/download/v0.2.1/checksums-linux-amd64.txt) |
| [sdacme-server-linux-amd64.tar.gz](https://github.com/huwenlong92/sdacme/releases/download/v0.2.1/sdacme-server-linux-amd64.tar.gz) | [checksums-server-linux-amd64.txt](https://github.com/huwenlong92/sdacme/releases/download/v0.2.1/checksums-server-linux-amd64.txt) |

下面是全新安装示例，以 root 执行。已有安装请使用升级模式；不要用示例配置覆盖既有 ACME 账户或证书目录。

### 安装客户端 sdacmectl

```sh
mkdir -p /root/sdacme-download/v0.2.1
cd /root/sdacme-download/v0.2.1
SDACME_RELEASE_URL=https://github.com/huwenlong92/sdacme/releases/download/v0.2.1
curl -fLO "$SDACME_RELEASE_URL/sdacmectl-linux-amd64.tar.gz"
curl -fLO "$SDACME_RELEASE_URL/checksums-linux-amd64.txt"
sha256sum -c checksums-linux-amd64.txt
tar -xzf sdacmectl-linux-amd64.tar.gz
cd sdacmectl-linux-amd64
sh scripts/install.sh install "$PWD"
sdacmectl --version
sdacmectl config check
```

安装器只安装客户端和准备配置，不创建续期 cron。全新环境再运行 `sdacmectl init`，自动准备专属证书运行环境。账户、证书、日志、锁和备份统一放在 `~/.sdacme/`；已有数据先按迁移流程处理，不重复初始化。

客户端默认配置为 `~/.sdacme/config.toml`；可用 `--config` 或 `SDACME_CONFIG` 指定。root 没有个人配置时，可沿用已有 `/etc/sdacme/config.toml`。

### 安装管理端 sdacme

```sh
mkdir -p /root/sdacme-download/v0.2.1
cd /root/sdacme-download/v0.2.1
SDACME_RELEASE_URL=https://github.com/huwenlong92/sdacme/releases/download/v0.2.1
curl -fLO "$SDACME_RELEASE_URL/sdacme-server-linux-amd64.tar.gz"
curl -fLO "$SDACME_RELEASE_URL/checksums-server-linux-amd64.txt"
sha256sum -c checksums-server-linux-amd64.txt
tar -xzf sdacme-server-linux-amd64.tar.gz
cd sdacme-server-linux-amd64
sh scripts/install-server.sh install "$PWD"
systemctl status sdacme.service
curl --fail http://localhost:9020/api/health
```

服务以 `sdacme` 专用用户运行。配置为 `/etc/sdacme-server/server.toml`，数据位于 `/var/lib/sdacme`；初始管理员 `admin`，随机初始密码位于 `/var/lib/sdacme/bootstrap-password`，安装器不打印密码。

服务默认只监听 `127.0.0.1:9020`。公网访问需配置 HTTPS 反代，并将 `allowed_origins` 设为实际 HTTPS 域名、`cookie_secure` 设为 `true`。Nginx upstream 应与监听地址匹配，避免 localhost 解析到 IPv6 后连接失败。

## sdacmectl 常用命令

```sh
sdacmectl --help
sdacmectl dns --help
sdacmectl export --help
sdacmectl --config /absolute/path/config.toml list
```

`--key-length` 为全局选项，可选 `ec-256`、`2048`、`3072`、`4096`。续期、导出和安装时，采用与原管理证书相同的算法。

| 命令 | 作用 |
| --- | --- |
| `init [--archive 文件]` | 初始化专属工作目录和运行环境；不覆盖已有数据 |
| `config init / example / check / lock-file` | 创建配置、查看示例、检查配置、显示锁路径 |
| `issue 域名… --webroot 目录` | 通过 HTTP-01 签发；不要求 Nginx |
| `dns issue --account 名称 域名…` | 使用已配置 DNS 账号自动签发 |
| `dns prepare 域名…` / `dns complete 主域名` | 生成手动 TXT、解析生效后完成签发 |
| `dns delegation-records --delegation 名称 域名…` | 生成固定 CNAME；不申请证书 |
| `dns delegate --delegation 名称 域名…` | 配好 CNAME 后委托签发 |
| `list` / `inventory` / `inspect 文件` | 管理列表、公共元数据 JSON、查看 PEM 证书 |
| `renew 主域名` / `cron` | 单证书续期检查、定时批量续期检查 |
| `export 主域名 --output 目录` | 导出 PEM 或 PFX 副本 |
| `install 主域名 --cert-file 文件 --key-file 文件` | 安装并登记续期部署路径 |
| `discover` / `adopt 域名` | 读取 Nginx 真实证书路径、预览接管；`--apply` 才写入 |
| `deploy providers / credential-example / list` | 查看云提供方、凭据示例、部署目标 |
| `deploy plan 名称` / `deploy run 名称` | 预览或执行云证书部署 |
| `report` / `report --watch` | 单次或持续上报公共证书元数据 |
| `add 域名 端口 [--dry-run]` / `cert 域名` / `status [域名]` | 本机 Nginx 站点助手 |
| `doctor` / `setup [--dry-run]` / `schedule` | 检查环境、准备 HTTP 验证片段、打印 cron |

### HTTP 与泛域名签发

HTTP 验证要求域名公开验证路径能访问指定 webroot；泛域名使用 DNS。

```sh
sdacmectl issue app.example.com --webroot /var/www/acme
sdacmectl --key-length 2048 issue rsa.example.com --webroot /var/www/acme

# 手动 DNS：按输出添加 TXT，生效后完成
sdacmectl dns prepare example.com '*.example.com'
sdacmectl dns complete example.com

# 自动 DNS：先配置 cloudflare-main 账号
sdacmectl dns issue --account cloudflare-main example.com '*.example.com'

# CNAME 委托：先配置 main-validation，再添加输出的 CNAME
sdacmectl dns delegation-records --delegation main-validation example.com '*.example.com'
sdacmectl dns delegate --delegation main-validation example.com '*.example.com'
```

第一个域名是管理主域名，后续续期、导出和安装使用它。CLI 中根域和泛域名需分别传入，通配符用引号包裹；Web 填写 `*.example.com` 会自动补充根域。DNS 账号获取步骤、权限和委托配置见管理端“帮助文档”。

### DNS 账号配置示例

在客户端 config.toml 中添加账号和可选委托配置。其他提供方使用 `provider = "aliyun"` 或 `"tencent"`，凭据字段见管理端帮助与项目 DNS 文档。

```toml
[[dns_accounts]]
name = "cloudflare-main"
provider = "cloudflare"
credentials_file = "/root/.sdacme/cloudflare.toml"

[[dns_delegations]]
name = "main-validation"
zone = "validation.example.org"
dns_account = "cloudflare-main"
```

cloudflare.toml 的内容为 `CF_Token = "REPLACE_WITH_YOUR_API_TOKEN"`；Token 需有对应区域读取、DNS 记录编辑权限。凭据明文存于私有文件，目录 700、文件 600，不写入源码；委托账号应能管理验证域名的 TXT。保存后运行 `sdacmectl config check`，再按上面的自动 DNS 或 CNAME 命令申请。

### 导出、安装与续期

```sh
# PEM：cert.pem、chain.pem、fullchain.pem、privkey.pem
sdacmectl export example.com --output /root/cert-delivery/pem

# PFX：密码放在仅所有者可读的文件，不放命令参数
sdacmectl export example.com --format pfx --output /root/cert-delivery/pfx \
  --password-file /root/private/pfx-password

# 登记文件路径，后续续期沿用；该目录须预先准备
sdacmectl install example.com --cert-file /etc/nginx/ssl/example/fullchain.pem \
  --key-file /etc/nginx/ssl/example/privkey.pem --reload-nginx

sdacmectl renew example.com
sdacmectl schedule
```

`export` 产生副本，不改变续期部署路径；`install` 才登记部署位置。`schedule` 只打印 cron，审核后由管理员安装，不要重复配置两套续期任务。正常续期不需要 `--force`，手动 DNS 续期仍需人工完成验证。

### 云部署与节点上报

```sh
# 先配置 deployments 目标和私有凭据文件
sdacmectl deploy providers
sdacmectl deploy plan assets
sdacmectl deploy run assets

# 先配置 reporting 的管理端 HTTPS 地址、node_id、token_file
sdacmectl report
sdacmectl report --watch
```

上报配置追加到客户端 config.toml：

```toml
[reporting]
node_id = "node-01"
server_url = "https://acme.example.com"
token_file = "/root/.sdacme/node-token"
interval_seconds = 120
allow_http = false
```

连接 token 写入仅所有者可读的文件。节点在 Web 登记后获取独立连接 token；已有机器应沿用其 ACME 目录和配置。上报与续期独立，上报服务不会代替 cron。云目标支持 `ali-cdn`、`ali-dcdn`、`qiniu`、`tencent-cdn`、`tencent-edgeone`，EdgeOne 需 ZoneId。

## sdacme 服务命令

管理端只有三个业务命令；证书操作在 Web 中进行，或由 `sdacmectl` 执行。

| 命令 | 作用 |
| --- | --- |
| `sdacme --config 文件 init` | 全新环境生成配置和随机初始密码文件，不覆盖已有文件 |
| `sdacme --config 文件 check` | 检查管理服务配置 |
| `sdacme --config 文件 serve` | 启动管理服务与内嵌 Web |

```sh
# 直接运行：仅全新环境需要 init，已有安装不要重新初始化
sdacme --config /absolute/path/server.toml init
sdacme --config /absolute/path/server.toml check
sdacme --config /absolute/path/server.toml serve

# 使用安装包配置的 systemd 服务
systemctl status sdacme.service
systemctl restart sdacme.service
journalctl -u sdacme.service --since '-5 minutes' --no-pager
```

管理端安装器自动准备 `/var/lib/sdacme/.sdacme/`，并关联其本机 controller。运行环境随安装包初始化，不需单独安装或管理；专用服务用户不读取 root 的账户目录。

## 升级与文档

下载新版本并验证校验和，在新解包目录使用 `scripts/install.sh upgrade "$PWD"` 升级客户端，使用 `scripts/install-server.sh upgrade "$PWD"` 升级服务。升级保留配置；服务安装器停机备份 binary、unit 和私有数据。回滚、卸载、自定义监听地址的完整步骤见各包内 README，服务后台的“帮助文档”提供 DNS 账号与申请流程说明。续期统一使用 `sdacmectl cron` 或管理端调度，不重复配置两套任务。

本仓库用于公开 **Release 制品和使用说明**。源码独立维护于 [Rust 仓库](https://gitee.com/sd0/sdacme) 和 [Web 仓库](https://gitee.com/sd0/sdacme-web)；不包含生产配置、访问密钥、节点 token、私钥或数据库。
