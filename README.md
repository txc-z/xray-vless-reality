# xray-vless-reality（防偷跑版）

Xray **VLESS + REALITY** 极简一键安装脚本，基于 [crazypeace/xray-vless-reality](https://github.com/crazypeace/xray-vless-reality) 修改。

与原版唯一的实质区别：生成的服务端配置合并了 XTLS 官方的 **REALITY 防偷跑（防被借道）** 写法，未授权回落流量只能到达指定的伪装目标站，无法被他人拿你的服务器 IP 当跳板白嫖目标站的流量。

- 参考资料（官方示例）：<https://github.com/XTLS/Xray-examples/blob/main/VLESS-TCP-REALITY%20(without%20being%20stolen)/config_server.jsonc>
- 脚本安装的 Xray 版本：v25.10.15（脚本内写死）

## 防偷跑原理

普通 REALITY 配置里，`realitySettings.dest` 直接填写目标站（如 `speed.cloudflare.com:443`）。任何未授权连接都会被服务端原样转发到目标站——也就是说，任何人都可以把你的服务器 IP 当作通往该目标站的免费跳板（“偷跑流量”），目标站是测速类站点时尤其容易被白嫖带宽。

本版把回落链路拆成两段：

```
未授权连接 ──► REALITY 入站 (监听端口, 默认 443)
                │  dest 不再直连目标站, 改指本机内部端口
                ▼
             dokodemo-door 入站 (127.0.0.1:8964, tag: dokodemo-in)
                │  sniffing 嗅出 TLS SNI, routeOnly 只用于路由匹配, 不改写目的地
                ▼
             routing 按 inboundTag=dokodemo-in 过滤:
                ├─ SNI == 伪装域名 → direct 出站 → 伪装域名:443 (真实目标站)
                └─ 其它任何 SNI    → block 出站 (blackhole, 直接断掉)
```

要点：

- **内部回落端口：8964**（写死，只监听 `127.0.0.1`，外部连不进来；安装端口不要设为 8964，脚本会提醒冲突）。
- dokodemo 的 `settings.address`（真实目标）、REALITY 的 `serverNames`、routing 里 dokodemo-in 的 `domain` 三处域名必须一致。脚本里三处都用同一个 `${domain}` 变量，天然同步；日后手改 `config.json` 时务必三处同改。
- dokodemo 的 `settings` 不能省：省掉它，dokodemo 会拿连接自身的原始目的地（即它自己的监听地址）当转发目标，形成自拨号死循环，一次探针就能堆满连接、打满文件句柄。
- 授权客户端完全不受影响：客户端配置、分享链接与原版一致，已部署的客户端不用换。

## 默认值

| 参数 | 默认值 | 说明 |
|---|---|---|
| 伪装域名 (SNI / dest) | **speed.cloudflare.com** | 与官方防偷跑示例一致；支持 TLS 1.3 + H2 的境外站点 |
| 安装端口 | 443 | 不要与内部回落端口 8964 冲突 |
| UUID | 由本机 IP + 主机名 + 时区派生 | 同一台机器重装不变，节点信息不失效 |
| 私钥 / 公钥 / ShortId | 由 UUID 派生 | 同上，重装可复现 |

## 使用

系统要求：Debian 10+。

```bash
bash <(curl -L https://raw.githubusercontent.com/txc-z/xray-vless-reality/main/install.sh)
```

带参数执行（跳过交互）：

```bash
# 参数依次为: 网络栈(4/6) 端口 域名 UUID, 缺省项用默认值
bash install.sh 4 443 speed.cloudflare.com
```

或用环境变量（定义任意一个即进入环境变量模式，忽略命令行参数）：

```bash
_MYIP_=1.2.3.4 _MYPORT_=443 _MYDOMAIN_=speed.cloudflare.com _MYUUID_=<uuid> bash install.sh
```

安装完成后脚本会打印节点信息与二维码，并把分享链接存到 `~/_vless_reality_url_`。

节点参数：VLESS + TCP + REALITY，flow 为 `xtls-rprx-vision`，指纹 `random`。

## 与原版的差异一览

只改了脚本生成 `config.json` 的部分，其余（安装流程、BBR、WARP、出站与分流规则、链接输出）与原版一致：

1. 新增 dokodemo-door 入站（`127.0.0.1:8964`，tls 嗅探 + `routeOnly`）；
2. `realitySettings.dest` 由 `${domain}:443` 改为 `127.0.0.1:8964`；
3. routing 规则最前面新增两条 dokodemo-in 规则（域名命中 → direct，其余 → block，均带 `"type": "field"`，因此生成配置在 Xray 1.8.0 上也能加载）；
4. 默认域名由 `learn.microsoft.com` 改为 `speed.cloudflare.com`；
5. 安装端口为 8964 时给出冲突提醒。

## 验证记录

本仓库的 install.sh 生成的配置已实测（Xray 26.10.31）：

- 授权客户端经隧道访问目标站：HTTP 200；
- 未授权探针带正确 SNI 回落：拿到目标站真实证书（校验通过）；
- 未授权探针带其它 SNI：被 routing 拦截（服务端日志 `taking detour [block]`），无证书返回；
- 生成配置在 Xray 1.8.0 / 24.9.30 / 26.10.31 上均通过 `xray run -test`。
