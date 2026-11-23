# fast-vpn

在 **AWS Lightsail + Ubuntu 24.04 LTS** 上用一条命令部署的 Hysteria2 订阅式 VPN。  
自带 Clash/Mihomo 订阅、大陆分流、HTTPS 证书自动化、流量统计和日常管理命令。

> **适合**：想在自己的 Lightsail 上快速搭一个稳定、可维护的个人/小团队 VPN。  
> **不适合**：需要 Web 面板、多用户计费、多节点编排，或非 Lightsail 平台的人。

当前基线：Hysteria **2.14.0**、UDP 443 + Salamander、16MB UDP 缓冲 + BBR、Hysteria ACL、订阅内证书指纹固定。详见 [config/software-baseline.json](config/software-baseline.json) 和 [docs/PARITY.md](docs/PARITY.md)。

## 目录

- [你能得到什么](#你能得到什么)
- [快速开始](#快速开始)
- [导入客户端](#导入客户端)
- [验收](#验收)
- [常用命令](#常用命令)
- [工作原理](#工作原理)
- [可选：Lightsail 流量统计](#可选lightsail-流量统计)
- [更新与回滚](#更新与回滚)
- [换 IP、改名、轮换](#换-ip改名轮换)
- [故障排查](#故障排查)
- [安全须知](#安全须知)
- [项目结构与测试](#项目结构与测试)
- [旧安装迁移](#旧安装迁移)
- [FAQ / 限制](#faq--限制)
- [参考](#参考)

## 你能得到什么

- **Hysteria2**：UDP 443 + `salamander` 混淆，不需要域名。
- **Clash/Mihomo 订阅**：CN 域名/IP 直连，其他走 `PROXY`；私有/链路本地地址直连；当前基线包含 t3 相关域名规则（不用可忽略）。
- **HTTPS 订阅**：nginx 在 TCP 443 提供订阅/规则文件，使用 Let's Encrypt 的 IP 证书（短期证书，自动续期）；TCP 80 用于 ACME 和 HTTP 订阅。
- **更稳的安全边界**：Hysteria 自签证书 + 订阅内 `fingerprint` 固定；Hysteria ACL 拒绝回环/私网/链路本地/云元数据地址；服务以非 root 的 `hysteria` 用户运行。
- **性能基线**：16MB UDP 缓冲 + BBR。
- **可观测性**：Lightsail 套餐流量（每 30 分钟）+ 按域名流量采集（SQLite + TUI）。
- **可维护**：`sub` 管理命令、systemd 服务/定时器、Geo 每天更新 4 次、62 项单元测试。
- **一条部署命令**：在 AWS 控制台准备好实例和端口后，执行一次 `deploy.sh`。

## 快速开始

### 0. 准备

| 项目 | 要求 |
|---|---|
| 实例 | AWS Lightsail，Linux/Unix → OS only → **Ubuntu 24.04 LTS**，amd64/arm64 |
| 网络 | 绑定**静态公网 IPv4**；记录准确 IP |
| 防火墙 | TCP **22 / 80 / 443**，UDP **443**（不要开放 25413） |
| 权限 | 能用 `sudo` 登录；实例可访问 Ubuntu 源、GitHub、snap |
| 参数 | 准确的 AWS 地区代码、Lightsail 实例名；节点名/订阅名可自定义 |
| 可选 | Lightsail 只读 IAM 身份（仅用于套餐流量显示） |
| 客户端 | Clash Mi、Clash Verge Rev、Clash Meta for Android（较新的 Mihomo 内核） |
| 域名 | **不需要**，证书直接签给公网 IP |

> `deploy.sh` 只适用于**全新专用实例**。已有 VPN 配置或已有服务的实例会被拒绝接管；旧安装请用 `update.sh` 或 `migrate.sh`。

### 1. 创建实例和静态 IP

1. 在 Lightsail 创建 Ubuntu 24.04 LTS 实例，记录**地区代码**（如 `ap-southeast-5`）和**准确实例名**。
2. 在同地区创建静态 IPv4 并 attach 到实例，记录公网 IP。
3. 确认能用 SSH 登录，并记下实例名；后面 `--region`、`--instance-name` 要用。

### 2. 开放防火墙

在 Lightsail → Networking → IPv4 firewall 添加入站规则；如果系统里启用了 UFW，也要同样放行。部署脚本**不会**改 AWS/UFW 防火墙。

| 协议 | 端口 | 用途 |
|---|---:|---|
| TCP | 22 | SSH；建议限制为自己的来源 IP，同时保留 Lightsail 控制台 SSH |
| TCP | 80 | HTTP 订阅 + Let's Encrypt 验证/续期；需长期可达 |
| TCP | 443 | HTTPS 订阅 + 规则文件 |
| UDP | 443 | Hysteria2 代理流量 |

不要开放 **25413**：它只在 `127.0.0.1` 提供域名统计 API。限制来源 IP 时，别忘了手机移动网络。

### 3. 部署

```bash
sudo apt-get update
sudo apt-get install -y git
git clone https://github.com/zheli2025/fast-vpn.git
cd fast-vpn
sudo bash deploy.sh --help
```

阅读 [Let's Encrypt 条款](https://letsencrypt.org/repository/) 后执行，替换所有尖括号字段：

```bash
sudo bash deploy.sh \
  --ip <静态公网IPv4> \
  --region <AWS地区代码> \
  --instance-name <Lightsail实例名> \
  --node '<节点显示名>' \
  --name <订阅名> \
  --accept-tos
```

示例值：`--region ap-southeast-5 --node 'Malaysia-01' --name my-vpn`。

参数说明：

- `--accept-tos`：接受 Let's Encrypt 服务条款。
- `--email`：可选 ACME 联系邮箱；不填则不登记邮箱。
- `--node`：最长 32 字符，不含双引号、反斜杠、控制字符，不要用 `PROXY`/`DIRECT` 等策略名。
- `--name`：订阅名，1–32 位小写字母、数字、连字符。
- `--instance-name`：必须和 Lightsail 控制台完全一致，否则流量统计会取错实例。

脚本会做这些事：

1. 安装依赖，统一为 `Etc/UTC`。
2. 下载并校验 Hysteria **2.14.0**（官方 `hashes.txt`）。
3. 创建 `hysteria`、`usage-tracker` 用户和目录，安装 16MB/BBR 参数。
4. 生成目标机独立凭据、Hysteria 配置（含 ACL）、本地统计 API。
5. 生成 HTTP 站点，申请 IP HTTPS 证书，启用自动续期，再生成双协议站点。
6. 下载 Geo 规则，启动 Hysteria、域名采集和定时器。
7. 尝试刷新 Lightsail 套餐流量（需要第 5 节的只读身份）。

如果网络/Geo 下载失败，修复后**用同一组参数重跑**即可，已有凭据和统计数据会保留。  
脚本只固定 Hysteria 版本，不固定 Ubuntu/Nginx/Python/snap 的所有版本。

## 导入客户端

```bash
sudo sub url
```

把输出的订阅 URL 导入 Clash Mi / Clash Verge Rev / Clash Meta for Android，更新订阅，选择 `PROXY` 节点，使用 **rule 模式**。

- URL 里含**私密 token**，不要截图、不要发群、不要贴日志。
- 订阅同时支持 HTTP 和 HTTPS；**日常用 HTTPS**，HTTP 会明文传输 token。
- 客户端支持时，会校验证书 `fingerprint`；`skip-cert-verify: true` 仍然保留，用于兼容旧客户端。
- 建议从手机移动网络再测一次，确认 UDP 443 在别的网络也可用。

## 验收

```bash
sudo sub status
sudo sub geo status
sudo sub traffic status
sudo sub usage status
sudo nginx -t
systemctl is-active hysteria-server nginx domain-usage
systemctl list-timers geo-update.timer traffic-update.timer domain-usage.timer snap.certbot.renew.timer
```

建议确认：

- 客户端能打开网页，出口 IP 是这台服务器；大陆站点按规则直连。
- 订阅无 token 返回 403，带正确 token 返回 200。
- `/geo/geosite-cn.mrs`、`/geo/geoip-cn.mrs` 能下载。
- 产生一些代理 TCP 流量后，`sudo sub usage` 能看到数据。
- 从手机/另一台机器测试，而不是只在 VPS 上自测。

## 常用命令

| 我想… | 命令 | 影响 |
|---|---|---|
| 查看状态 | `sudo sub status` | 只读 |
| 获取订阅 URL | `sudo sub url` | 输出含 token，勿分享 |
| 刷新订阅 YAML | `sudo sub update` | 不轮换密码；客户端刷新订阅 |
| 改节点名 | `sudo sub node '新节点名'` | 客户端刷新订阅 |
| 改订阅路径 | `sudo sub name new-name` | 客户端需要换新 URL |
| 轮换订阅 token | `sudo sub token` | 旧 URL 失效，客户端换新 URL |
| 轮换代理/混淆密码 | `sudo sub rotate` | Hysteria 短暂重启，客户端刷新订阅 |
| 换公网 IP | `sudo sub ip <新IP>` | 更新证书/订阅/流量；客户端换新 URL |
| 查套餐流量 | `sudo sub traffic status` | 只读 |
| 查域名用量 | `sudo sub usage` / `sudo sub usage tui` | 只读 |
| 更新 Geo | `sudo sub geo update` | 下载规则 |
| 看 Hysteria 日志 | `sudo journalctl -u hysteria-server -n 50 --no-pager` | 只读 |
| 检查代码更新 | `sudo bash update.sh --check` | 只读 |
| 应用代码更新 | `sudo bash update.sh --apply` | 更新程序文件，不重建订阅/密钥 |
| 回滚代码更新 | `sudo bash update.sh --rollback <备份ID>` | 只回退当次代码/单元 |

`sub help` 可以随时查看完整用法。

## 工作原理

```text
客户端（Clash / Mihomo）
   ├─ TCP 443 ──> nginx：HTTPS 订阅、规则文件、IP 证书
   ├─ TCP 80  ──> nginx：HTTP 订阅、ACME 验证
   └─ UDP 443 ──> Hysteria2：代理流量
                     ├─ ACL：拒绝回环/私网/链路本地/元数据地址
                     ├─ trafficStats：127.0.0.1:25413
                     └─ domain-usage：按域名统计（SQLite）
```

### 仓库 vs 运行时

这个仓库是**部署源码**，不是运行目录。systemd 的 `ExecStart` 全部指向系统路径，仓库里的文件只有 `deploy.sh`/`update.sh` 执行时才会被复制过去。

| 仓库 | 安装后 |
|---|---|
| `scripts/sub.sh` | `/usr/local/bin/sub` |
| `scripts/geo-update.sh` | `/usr/local/bin/geo-update` |
| `scripts/traffic-launcher.py` | `/usr/local/bin/traffic-update` |
| `scripts/traffic-update.py` | `/usr/local/lib/fast-vpn/traffic-update.py` |
| `scripts/domain-usage.py` | `/usr/local/bin/domain-usage` |
| `systemd/*` | `/etc/systemd/system/*` |
| `config/99-fast-vpn.conf` | `/etc/sysctl.d/99-fast-vpn.conf` |
| `config/tcp_bbr.conf` | `/etc/modules-load.d/tcp_bbr.conf` |
| —（下载） | `/usr/local/bin/hysteria` |
| —（生成） | `/etc/hysteria/config.yaml`、`/etc/hysteria/certs/` |
| —（生成） | `/var/www/html/<name>.yaml`、`/var/www/html/geo/` |
| —（生成） | `/etc/nginx/sites-available/vpn-subscription` |
| —（生成） | `/etc/letsencrypt/` |
| —（标记） | `/etc/fast-vpn/install.json` |

要点：

- 改仓库不会自动改线上；需要 `deploy.sh`、`update.sh --apply` 或手动安装。
- `/etc/hysteria/config.yaml`、证书私钥、订阅 YAML/token、SQLite、AWS 凭据都不在仓库里。
- `/etc/fast-vpn/install.json` 是防误接管的安装标记，不要手改。
- 当前基线/哈希/权限见 [docs/PARITY.md](docs/PARITY.md)。

## 可选：Lightsail 流量统计

VPN 本身**不需要** AWS 凭据；只有“套餐配额/已用流量”显示需要。

1. 在 IAM 创建只读身份，附加 [docs/aws-readonly-policy.json](docs/aws-readonly-policy.json)（只允许 `GetInstance`、`GetInstanceMetricData`）。
2. 在实例上配置 root 的默认 profile：

```bash
sudo install -d -m 0700 /root/.aws
sudoedit /root/.aws/credentials
sudo chmod 0600 /root/.aws/credentials
```

```ini
[default]
aws_access_key_id = <只读AccessKeyID>
aws_secret_access_key = <只读SecretAccessKey>
```

3. 刷新并查看：

```bash
sudo systemctl start traffic-update.service
sudo sub traffic status
```

说明：

- 不要复制其他服务器的密钥；不要提交 `/root/.aws/`。
- 地区/实例名/IP 在 `/etc/systemd/system/traffic-update.service.d/migrate.conf`，不含密钥。
- 流量是月初 UTC 起的 NetworkIn+NetworkOut，**不是 AWS 最终账单**；域名用量采样活跃 TCP 流，短连接/UDP 可能漏记。

## 更新与回滚

`update.sh` 用于**已有安装的代码更新**，默认只检查：

```bash
cd ~/fast-vpn
sudo bash update.sh --check
sudo bash update.sh --apply
```

可选：把 `systemd` 单元也同步成仓库版本（可能重启服务、短暂断线）：

```bash
sudo bash update.sh --check --update-units
sudo bash update.sh --apply --update-units
```

回滚：

```bash
sudo bash update.sh --rollback <备份ID>
```

说明：

- 每次实际更新会把程序/单元备份到 `/var/backups/fast-vpn-update/<ID>/`；回滚只恢复当次代码/单元，不重置配置或流量历史。
- `--check` 显示“待确认”表示某个程序是未收录旧版或有本地改动；`--apply` 会停止，不要强行覆盖，先人工核对。
- **`update.sh` 不升级 Hysteria 二进制、ACL 或 sysctl**，也不改订阅/密钥/证书。Hysteria 当前固定在 2.14.0；升级属于单独的、需要验证的变更。
- 新实例用 `deploy.sh`；旧安装用 `migrate.sh`；不要把 `deploy.sh` 跑在已有服务的实例上。

## 换 IP、改名、轮换

| 操作 | 命令 | 客户端影响 |
|---|---|---|
| 换公网 IP | `sudo sub ip <新IP>` | 需要换新 URL；认证/混淆密码/token 保留 |
| 改订阅路径 | `sudo sub name new-name` | 需要换新 URL |
| 改节点显示名 | `sudo sub node '新节点名'` | 刷新订阅即可 |
| 轮换 token | `sudo sub token` | 旧 URL 失效，换新 URL |
| 轮换代理/混淆密码 | `sudo sub rotate` | 刷新订阅；Hysteria 短暂重启 |
| 仅刷新订阅 | `sudo sub update` | 刷新订阅；不轮换密码 |

换 IP 前：

1. 在 AWS 绑定新静态 IPv4，并确认 TCP 80/443、UDP 443 已放行。
2. 通过新 IP SSH 登录同一实例。
3. 执行 `sudo sub ip <新IP>`，然后 `sudo sub status`、`sudo sub traffic update`。
4. 客户端换新 URL 并刷新订阅。

`sub ip` 会依次处理新 IP 的 HTTPS 证书、Hysteria 自签证书、订阅、流量统计和旧证书清理。它不是完整事务：如果中途失败，修复后运行 `sudo sub update`；不要对同一个 IP 反复执行 `sub ip`。

## 故障排查

### 客户端连不上

- Lightsail 防火墙和 UFW 是否放行 **UDP 443**？
- `systemctl status hysteria-server`、`sudo journalctl -u hysteria-server -n 50 --no-pager`。
- `sudo ss -ulnp | grep :443` 是否在监听。
- 客户端是否刷新了订阅、是否选中 `PROXY`、是否 rule 模式。
- Hysteria2 只走 UDP；如果当前网络封 UDP，换网络测试，没有 TCP 回退。

### 订阅返回 403

- URL 里的 token 是否完整；是否在别处执行过 `sudo sub token`。
- `sudo sub status` 看订阅/鉴权状态；`sudo sub url` 重新获取 URL（勿分享）。
- 如果怀疑泄露，执行 `sudo sub token` 轮换，然后客户端换新 URL。

### HTTPS 订阅/证书失败

- TCP 80 是否长期可达；IP 是否已 attach 到实例。
- `sudo /snap/bin/certbot renew --dry-run`
- `sudo journalctl -u snap.certbot.renew.service --since '24 hours ago'`
- `/var/log/letsencrypt/letsencrypt.log`
- 不要高频重复申请，避免触发 CA 限额。

### 没有套餐流量

- 是否配置了只读 AWS 身份；`sudo systemctl start traffic-update.service`。
- `sudo sub traffic status`；检查 `/etc/systemd/system/traffic-update.service.d/migrate.conf` 的地区/实例名。
- 没有 AWS 身份不影响 VPN 本身。

### Geo 规则缺失

```bash
sudo sub geo update
sudo sub geo status
sudo journalctl -u geo-update.service -n 50 --no-pager
```

### 域名用量为空

- 需要产生一些代理 **TCP** 流量后再看。
- `sudo sub usage status`；采集器每 5 秒看一次活跃流，短连接/UDP 可能漏记。
- 不代表完整账单。

### `update.sh` 显示“待确认”

- 某个已安装程序与仓库/基线不一致，可能有本地改动。
- 先 `sudo bash update.sh --check` 看差异，人工核对后再决定；不要直接强行覆盖。

## 安全须知

- **订阅 URL = 密码**：含 token，不要分享、不要贴日志/截图。泄露后执行 `sudo sub token`。
- **优先 HTTPS**：HTTP 订阅会明文传输 token；TCP 80 主要留给 ACME/兼容。
- **证书固定**：Hysteria 使用自签证书，订阅里用 `fingerprint` 固定；`skip-cert-verify: true` 只为兼容旧客户端。建议使用较新的 Mihomo。
- **ACL 不要随便删**：默认拒绝回环/私网/链路本地/云元数据地址，避免代理被用来访问服务器内网或 169.254.169.254。
- **SSH 收口**：限制 22 的来源 IP，保留 Lightsail 控制台 SSH 作为兜底。
- **AWS 凭据只读**：只附加 `GetInstance`、`GetInstanceMetricData`；放在 `/root/.aws/`，不要提交。
- **25413 不要对外开放**：它只监听 `127.0.0.1`。 
- **备份可能含密钥**：迁移备份在 root 私有目录；不要上传 Git/聊天。`update.sh` 的程序备份不含密钥。
- **仓库不存秘密**：`.gitignore` 排除了常见密钥/订阅/数据库/日志，但提交前仍要检查。

## 项目结构与测试

```text
.
├── deploy.sh           # 全新实例部署
├── update.sh           # 已有安装的代码更新/回滚
├── migrate.sh          # 基础旧安装迁移
├── scripts/            # sub、Geo、流量、域名统计等
├── systemd/            # 服务与定时器模板
├── config/             # 基线配置、sysctl
├── docs/               # 基线记录、迁移指南、AWS 只读策略
└── tests/              # 单元测试
```

本地检查（不读取生产密钥，不调用 AWS）：

```bash
bash -n deploy.sh update.sh migrate.sh scripts/*.sh
python3 -m unittest discover -s tests -v
```

当前 62 项测试应全部通过。测试会检查基线哈希和两处已记录的兼容修复，但不替代新实例的 ACME、公网 UDP、手机实际连接验收。

仓库只放代码/文档/模板；不要从运行目录复制私钥、订阅 YAML、SQLite、AWS 凭据或日志进仓库。

## 旧安装迁移

**已有服务/旧安装不要运行 `deploy.sh`。** 基础旧安装先读 [docs/MIGRATE.md](docs/MIGRATE.md)，然后：

```bash
sudo bash migrate.sh --check <参数...>
sudo bash migrate.sh --apply --replace-legacy-code <参数...>
```

需要顺带升级 Hysteria 时显式加 `--upgrade-hysteria`；旧公开订阅可加 `--archive-old-subscriptions`；旧静态站点可用 `--adopt-nginx-site`。完整参数和回滚说明见文档。

普通的已有 fast-vpn 安装只需要 `update.sh`，不要跑 `migrate.sh`。

## FAQ / 限制

**需要域名吗？**  
不需要。订阅 HTTPS 用 Let's Encrypt 的 IP 证书，短期证书自动续期；TCP 80 要长期可达。

**可以多用户吗？**  
当前是单密码/单订阅模型，多设备可共享同一订阅；`sub rotate` 会全局轮换。没有按用户计费/限额。

**支持 IPv6 客户端吗？**  
客户端配置是 `ipv6: false`，不做 IPv6 代理；服务器 OS 的 IPv6 保持启用但未用于客户端出口。

**支持 UDP 吗？**  
支持，Hysteria2 本身就是 UDP；ACL 同样生效。如果网络封 UDP 443，就连不上。

**有端口跳跃、mimic、Brutal 吗？**  
没有内置。可以在 Hysteria 配置/客户端上手动加；端口跳跃还需要改防火墙和权限，mimic 需要 root/eBPF，属于单独变更。

**能装在其他云/其他系统吗？**  
部署脚本面向 Lightsail + Ubuntu 24.04；其他平台需要自行适配防火墙、AWS 流量统计和依赖，不保证可用。

**流量统计是账单吗？**  
不是。套餐流量按月初 UTC 的 NetworkIn+NetworkOut 估算；域名用量是 5 秒采样的活跃 TCP 流，短连接/UDP 可能漏记。

**怎么卸载？**  
没有卸载脚本；最干净的方式是销毁 Lightsail 实例。不要只删仓库目录，systemd 服务和证书还在。

**会跟随 Hysteria 自动升级吗？**  
不会。仓库固定 Hysteria 2.14.0；升级需要单独改 `deploy.sh`/验证。`update.sh` 只更新代码组件。

**为什么有 TCP 80 和 TCP 443？**  
80 用于 ACME 验证和 HTTP 订阅，443 用于 HTTPS 订阅；UDP 443 是 Hysteria。不要开放 25413。

**成本大概是什么？**  
Lightsail 实例 + 静态 IP + 套餐流量；Let's Encrypt 免费。具体看 AWS 价格页和套餐额度。

## 参考

- [Hysteria 2 文档](https://v2.hysteria.network/)
- [Let's Encrypt 服务条款](https://letsencrypt.org/repository/)
- [Lightsail 防火墙说明](https://docs.aws.amazon.com/lightsail/latest/userguide/understanding-firewall-and-port-mappings-in-amazon-lightsail.html)
- [当前基线记录](docs/PARITY.md)
- [旧安装迁移指南](docs/MIGRATE.md)
- [AWS 只读策略](docs/aws-readonly-policy.json)
