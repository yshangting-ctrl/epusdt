# Epusdt v2.0.0：Ubuntu + 宝塔面板部署教程

这份包提供两种部署方式：

- **方案 A：Docker（推荐）**——架构自动匹配、升级和回滚更简单，数据集中保存在 `data/`。
- **方案 B：宝塔 Supervisor + 官方二进制**——不使用 Docker，包内同时提供 `amd64` 和 `arm64` 发布包。

建议生产环境采用方案 A。两种方案只能选一种，不要同时启动，否则会争用 8000 端口。

## 0. 部署前准备

1. 一台 Ubuntu 22.04/24.04 服务器，建议至少 1 核、1 GB 内存、10 GB 可用磁盘。
2. 已安装宝塔面板，软件商店已安装 Nginx。方案 A 还需安装 Docker/容器管理器。
3. 准备独立支付域名，例如 `pay.example.com`，将 DNS 的 A 记录指向服务器公网 IP；有 IPv6 时再正确配置 AAAA。
4. 云厂商安全组和系统防火墙只需放行 `80`、`443`；SSH 与宝塔端口建议只允许管理员 IP。**不要对公网放行 8000**。
5. 确认当地关于加密资产、支付、税务、反洗钱和制裁合规的要求。此项目是非托管软件，不替你承担合规责任。

## 1. 在宝塔中建立站点和 HTTPS

1. 打开宝塔 → **网站** → **添加站点**。
2. 域名填 `pay.example.com`，PHP 选择“纯静态”，数据库不用创建。
3. 进入站点 → **SSL**，申请 Let's Encrypt 证书并开启“强制 HTTPS”。
4. 进入站点 → **反向代理** → 添加反向代理：
   - 代理名称：`epusdt`
   - 目标 URL：`http://127.0.0.1:8000`
   - 发送域名：`$host`
   - 内容替换：留空
5. 如需手动调整 Nginx，可参考 `nginx/epusdt.conf.example`。其中已关闭代理缓冲，以兼容后台实时统计。

先建站和证书，再初始化 Epusdt。这样安装向导中的公开地址可直接填写最终 HTTPS 域名。

## 2. 方案 A：Docker 部署（推荐）

### 2.1 上传并解压

在宝塔 **文件** 中建立目录：

```text
/www/wwwroot/epusdt-docker
```

将本部署包上传并解压到该目录。解压后应能看到：

```text
docker-compose.yml
scripts/
nginx/
release/
```

### 2.2 启动

在宝塔终端执行：

```bash
cd /www/wwwroot/epusdt-docker
bash scripts/start.sh
```

配置已固定使用官方镜像 `gmwallet/epusdt:v2.0.0`，并把宿主机端口绑定在 `127.0.0.1:8000`，因此只能通过本机宝塔 Nginx 访问。

如果提示没有 `docker compose`，请先在宝塔软件商店安装 Docker/容器管理器；也可按 Docker 官方文档安装 Compose 插件。

### 2.3 完成首次安装向导

浏览器打开：

```text
https://pay.example.com
```

建议填写：

| 字段 | 值 |
|---|---|
| 应用名称 | 自定义，例如 `My Pay` |
| 对外地址 / App URI | `https://pay.example.com`，末尾不加 `/` |
| HTTP 监听地址 | **`0.0.0.0`**（Docker 模式必须这样填） |
| HTTP 端口 | `8000` |
| Runtime 目录 | `./runtime` |
| 日志目录 | `./logs` |
| 订单过期时间 | `10` 分钟，或按业务需要调整 |
| 回调最大重试 | 建议先用默认值 |

安装成功页会显示随机生成的管理员初始密码。管理员用户名固定为 `admin`。请立即保存密码、登录后台并修改为独立强密码；不要把密码或商户密钥发给任何人。

如果误把容器监听地址填成 `127.0.0.1`，在服务器编辑：

```text
/www/wwwroot/epusdt-docker/data/.env
```

把：

```ini
http_listen=127.0.0.1:8000
```

改为：

```ini
http_listen=0.0.0.0:8000
```

然后执行：

```bash
cd /www/wwwroot/epusdt-docker
docker compose restart epusdt
```

### 2.4 检查运行状态

```bash
cd /www/wwwroot/epusdt-docker
bash scripts/status.sh
```

也可以分别执行：

```bash
docker compose ps
docker compose logs --tail=100 epusdt
curl -I http://127.0.0.1:8000/
```

看到容器为 `Up`，且通过支付域名能打开页面，即表示基础部署完成。

## 3. 方案 B：宝塔 Supervisor + 官方二进制

### 3.1 判断服务器架构

在宝塔终端执行：

```bash
dpkg --print-architecture
```

- 输出 `amd64`：使用 `release/epusdt-2.0.0-linux-amd64.tar.gz`
- 输出 `arm64`：使用 `release/epusdt-2.0.0-linux-arm64.tar.gz`

### 3.2 校验并解压

以下以 amd64 为例：

```bash
cd /www/wwwroot
mkdir -p epusdt
cd epusdt
cp /你的部署包路径/release/epusdt-2.0.0-linux-amd64.tar.gz .
cp /你的部署包路径/release/SHA256SUMS-v2.0.0.txt .
sha256sum -c SHA256SUMS-v2.0.0.txt --ignore-missing
tar -xzf epusdt-2.0.0-linux-amd64.tar.gz
chmod +x epusdt
chown -R www:www /www/wwwroot/epusdt
```

校验应显示 `OK`。ARM64 服务器把文件名中的 `amd64` 改为 `arm64`。

首次启动不需要手动建立 `.env`；程序会打开安装向导并在提交后创建它。

### 3.3 安装并配置 Supervisor

宝塔 → **软件商店** → 安装 **Supervisor 管理器**，然后添加守护进程：

| 设置 | 值 |
|---|---|
| 名称 | `epusdt` |
| 运行目录 | `/www/wwwroot/epusdt` |
| 启动用户 | `www` |
| 启动命令 | `./epusdt http start` |
| 自动启动 | 开启 |

启动后打开 `https://pay.example.com` 完成安装。二进制/Supervisor 模式的监听地址建议填 `127.0.0.1`，端口填 `8000`。其他字段与 Docker 方案相同。

注意：启动命令必须是 `./epusdt http start`，并确保运行目录正确，否则程序可能在错误位置查找 `.env` 和数据文件。

## 4. 后台必须完成的配置

首次登录后，至少完成以下项目：

1. **修改管理员密码**。
2. **建立商户 API Key**，保存 `pid` 与 `secret_key`，按需设置调用方 IP 白名单。
3. **配置链、代币和 RPC 节点**，停用不使用的网络。
4. **添加收款钱包地址**。只添加公开收款地址，**绝不要上传助记词或私钥**。
5. **配置汇率**：选择可靠的汇率 API，或设置固定汇率；确认币种和小数位符合业务逻辑。
6. **设置通知渠道**（可选），例如 Telegram。
7. 用极小金额做一次完整测试：下单 → 转账 → 链上确认 → 异步回调 → 商户订单入账。

推荐 GMPay 下单接口：

```text
POST https://pay.example.com/payments/gmpay/v1/order/create-transaction
```

从 v2.0.0 开始，GMPay 请求和回调签名已由 MD5 改为 **HMAC-SHA256**。旧版 GMPay 客户端会收到 `401 Unauthorized`，必须先升级签名逻辑；EPay 兼容接口仍使用 MD5。

不要再使用已经移除的旧路由：

```text
/payments/epusdt/v1/order/create-transaction
```

## 5. 备份、升级和恢复

### Docker 备份

```bash
cd /www/wwwroot/epusdt-docker
bash scripts/backup.sh
```

脚本会短暂停止服务以确保 SQLite 备份一致，并在 `backups/` 生成压缩包，然后自动重新启动服务。请再把备份同步到另一台机器或对象存储；只放在原服务器上不算可靠备份。

核心数据位于：

```text
data/.env
data/epusdt.db
data/runtime/
```

### Docker 升级

1. 阅读新版本发布说明与迁移说明。
2. 修改 `docker-compose.yml` 中的镜像版本号。
3. 执行 `bash scripts/upgrade.sh`；脚本会先备份，再拉取并重建容器。
4. 用小额订单重新验证支付和回调。

不要长期使用不可追踪的 `latest` 做生产升级；固定版本更容易回滚。

### Supervisor 备份

先在宝塔停止 `epusdt` 守护进程，再备份整个 `/www/wwwroot/epusdt` 目录，完成后重新启动。升级时保留现有 `.env`、数据库和 `runtime/`，只替换官方新版本的 `epusdt` 二进制和 `.env.example`。

### 恢复原则

停止服务后，把备份恢复到原路径，确认属主/权限正确，再启动服务。恢复前务必再保存一份当前目录，避免覆盖后无法反悔。

## 6. 常见故障

### 域名返回 502

```bash
curl -I http://127.0.0.1:8000/
```

- 无法连接：查看 Docker/Supervisor 状态和日志。
- Docker 初始化后才出现：检查 `data/.env` 的 `http_listen` 是否为 `0.0.0.0:8000`。
- 本机正常但域名 502：检查宝塔反向代理目标是否正好是 `http://127.0.0.1:8000`。

### HTTPS 正常，但回调失败

- 回调地址必须是公网可访问的 HTTPS 地址，且不能是 `localhost`、内网 IP。
- 检查商户系统是否及时返回成功响应。
- 检查域名证书链、DNS、CDN/WAF 和日志。
- 若用了 Cloudflare，不要启用会缓存 API 响应的规则。

### 支付不到账或长时间不确认

- 核对网络、代币合约、收款地址和小数位。
- 检查 RPC 节点健康状态和服务器出站网络。
- 确保用户实际转入的网络与订单指定网络一致。
- 先在区块浏览器确认交易，再查看 Epusdt 日志；不要在未核验链上交易时手工标记已支付。

### 忘记初始密码

初始密码在安装完成时返回，修改密码后不会继续以明文提供。优先从部署时保存的密码管理器查找；不要随意删除数据库重新安装，因为这会丢失订单和配置。

## 7. 安全检查清单

- [ ] 8000 端口只监听 `127.0.0.1`，公网安全组未放行 8000。
- [ ] 全站 HTTPS，证书自动续期正常。
- [ ] 宝塔面板、SSH 仅允许管理 IP，系统和面板定期更新。
- [ ] 管理员密码已修改，商户密钥未写入前端或公开仓库。
- [ ] 只配置收款地址，不在服务器存储钱包私钥/助记词。
- [ ] API Key 启用了合理的 IP 白名单，停用不使用的链和接口。
- [ ] 已做小额真实支付、回调、重复通知和超时订单测试。
- [ ] 已建立异机/异地备份，并实际演练过恢复。
- [ ] 已确认业务所在地的法律、税务、制裁与反洗钱要求。

## 8. 官方链接

- 项目仓库：https://github.com/GMwalletApp/epusdt
- v2.0.0 发布页：https://github.com/GMWalletApp/epusdt/releases/tag/v2.0.0
- Docker 部署：https://epusdt.com/zh/guide/installation/docker
- aaPanel / 宝塔部署：https://epusdt.com/zh/guide/installation/aapanel
- 官方文档：https://epusdt.com

