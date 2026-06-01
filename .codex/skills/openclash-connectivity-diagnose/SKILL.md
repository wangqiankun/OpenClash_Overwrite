---
name: openclash-connectivity-diagnose
description: 排查 OpenClash_Overwrite 仓库中的 OpenClash 连接、DNS、fake-ip、节点健康检查、策略组、覆写配置和路由器运行态问题。当节点无法解析、测速失败、策略组抖动、OpenClash DNS 代理异常，或需要进入 192.168.3.1 路由器现场检查时使用。
---

# OpenClash 连接异常排查

## 适用范围

遇到以下 OpenClash 运行态问题时使用这个 skill：

- 节点无法解析或无法测速。
- OpenClash DNS 代理 / fake-ip 表现不稳定。
- `url-test`、`fallback` 或 Smart 策略组把大量节点标记为失败。
- 本地配置和 `/etc/openclash/666OS-dns.yaml` 运行态配置不一致。
- OpenClash 覆写配置本地看起来已修改，但路由器运行态没有生效。

默认路由器目标是 `root@192.168.3.1`。在确认故障类型之前，不要先改配置。

## 排查流程

1. 记录本地配置意图：
   - `git status --short`
   - `rg -n "CORE_TYPE|AUTO_SMART_SWITCH|ENABLE_CUSTOM_DNS|ENABLE_RESPECT_RULES|APPEND_DEFAULT_DNS|proxy-server-nameserver|BaseFB|BaseUT|自建SG|流量节点" Overwrite/666OS-dns.conf Yaml/666OS-DNS-Conf.yaml`
2. 记录路由器运行态：
   - 当前 OpenClash / clash 进程和实际加载的配置文件。
   - 监听端口。
   - 最终生成的 YAML，而不是只看源 YAML。
   - OpenClash 日志。
3. 判断故障类型：
   - DNS 失败：出现 `no such host`、`DNS resolve failed`，或 fake-ip 无返回。
   - 节点入口失败：真实 DNS 能解析，但节点入口 TCP 连接失败。
   - 健康检查失败：API delay 返回 `504`，大量节点 `alive=false delay=0`。
   - 运行态生成失败：源配置和 `/etc/openclash/666OS-dns.yaml` 出现非预期差异。
4. 只在故障类型明确后做最小配置改动，并重新验证运行态。

## 路由器快速探针

在路由器上执行：

```sh
date
ps w | grep -E "[c]lash|[m]ihomo|openclash"
netstat -lntup 2>/dev/null | grep -E "(:7874|:7890|:7891|:9090|clash)" || true
uci show openclash | grep -Ei "core_type|auto_smart|custom_dns|respect_rules|redirect_dns|append_default|router_self_proxy|disable_udp"
sed -n "90,145p" /etc/openclash/666OS-dns.yaml
logread -e openclash | tail -n 120
```

重点：最终运行态以 `/etc/openclash/666OS-dns.yaml` 为准。`/etc/openclash/config/666OS-dns.yaml` 只是下载后的源配置。

## OpenClash 官方检查项

如果 DNS 看起来正常，但流量仍然失败，执行官方连接异常排查项：

```sh
iptables -t nat -nL --line-number
iptables -t mangle -nL --line-number
ip route
ip rule
ip tuntap list 2>/dev/null || true
```

检查 OpenClash 的 PREROUTING 规则是否存在，并确认前面没有其他劫持 / 重定向规则优先捕获同一类流量。如果使用 TUN 模式，确认 TUN 设备和路由规则存在。

参考：`https://github.com/vernesong/OpenClash/wiki/网络连接异常时排查原因`

## DNS 检查

在局域网客户端执行：

```sh
dig +time=2 +tries=1 @192.168.3.1 -p 7874 shlii-sg.19930405.xyz A
dig +time=2 +tries=1 @192.168.3.1 -p 7874 chatgpt.com A
```

fake-ip 模式下，如果 OpenClash DNS 返回 `198.18.x.x`，说明 fake-ip DNS 基本正常。

在路由器上测试真实上游 DNS，避免被 OpenClash fake-ip 劫持混淆：

```sh
nslookup shlii-sg.19930405.xyz 119.29.29.29
nslookup fuckip-sg.19930405.xyz 119.29.29.29
```

启用 DNS 劫持时，不要把局域网客户端的 `dig @119.29.29.29` 当作真实上游行为证据；它仍可能被 OpenClash 截获。

## API 健康检查

从运行态 YAML 读取控制器密钥：

```sh
grep -nE "^(external-controller|secret):" /etc/openclash/666OS-dns.yaml
```

然后在本机查询：

```python
import json, urllib.request
base = "http://192.168.3.1:9090"
secret = "replace-with-secret"
headers = {"Authorization": "Bearer " + secret}
req = urllib.request.Request(base + "/proxies", headers=headers)
proxies = json.load(urllib.request.urlopen(req, timeout=5))["proxies"]
for name in ["默认代理", "香港自动", "狮城自动", "自建SG", "流量节点", "故障转移"]:
    item = proxies.get(name, {})
    print(name, item.get("type"), item.get("now"), item.get("alive"), len(item.get("all") or []))
```

对代表性节点强制发起一次 delay 检查：

```python
import urllib.parse, urllib.request
node = "replace-with-node-name"
url = "https://cp.cloudflare.com/generate_204"
path = "/proxies/" + urllib.parse.quote(node, safe="") + "/delay?timeout=5000&url=" + urllib.parse.quote(url, safe="")
req = urllib.request.Request("http://192.168.3.1:9090" + path, headers=headers)
print(urllib.request.urlopen(req, timeout=8).read().decode())
```

`504 Gateway Timeout` 表示该节点健康检查失败。如果大量节点同时失败，但 DNS fake-ip 正常返回，优先排查健康检查 / 网络路径抖动，而不是先假设 DNS 解析失败。

## 本项目已知坑位

- `AUTO_SMART_SWITCH = 0` 和 `CORE_TYPE = Meta` 必须在最终运行态里确认，不能只看本地覆写文件。
- `APPEND_DEFAULT_DNS = 1` 可能扩大 DNS 上游。如果生成后的 `proxy-server-nameserver` 出现非预期 DNS，比较源 YAML 和生成 YAML。
- `DOWNLOAD_FILE ... force=true` 会在重启 / 下载时覆盖路由器源配置。如果手动同步后的配置重启就消失，优先检查它。
- `ENABLE_CUSTOM_DNS = 1` 可能让 OpenClash 插件层 DNS 设置重写 YAML 的 `dns:` 段。
- `proxy-server-nameserver` 可能被生成逻辑从 `default-nameserver`、`nameserver` 或 `fallback` 扩展；如果要严格控制节点域名解析上游，需要让所有 DNS 上游列表保持一致。
- `url-test` 和 `fallback` 策略组会把公司网络抖动放大成“节点不可用”。`自建SG`、`流量节点` 这类小组，如果自动切换收益不高，可以改成 `select`。
- 父策略组 `alive=true` 但子节点全部 `alive=false`，说明存在缓存 / 组状态不一致。改策略前先通过 API 复查。

## 修改后的验证

执行：

```sh
git diff --check -- Overwrite/666OS-dns.conf Yaml/666OS-DNS-Conf.yaml
scp -O -q Overwrite/666OS-dns.conf root@192.168.3.1:/etc/openclash/overwrite/666OS-dns.conf
scp -O -q Yaml/666OS-DNS-Conf.yaml root@192.168.3.1:/etc/openclash/config/666OS-dns.yaml
ssh root@192.168.3.1 "/etc/init.d/openclash restart"
```

使用 `scp -O`，因为路由器可能没有 SFTP 服务。

重启后验证：

```sh
ssh root@192.168.3.1 "sed -n '90,145p' /etc/openclash/666OS-dns.yaml; grep -nE 'name: (自建SG|流量节点|香港自动|狮城自动)|type: (select|url-test|fallback)|interval:' /etc/openclash/666OS-dns.yaml | head -n 120"
```

只有当最终生成的运行态 YAML 和 API 状态都符合预期时，才能认为修改完成。
