# files/ —— 烤进固件的 overlay(消灭"每次刷机重贴")

OpenWrt/ImmortalWrt buildroot **自动把仓库根的 `files/` 目录复制进 rootfs**(squashfs 层)。
把本目录整体放到 fork(`zxa24/immortalwrt`)**仓库根**下叫 `files/`,再跑 `build-xr1710g.yml`,
这些东西就随固件烤进去,`sysupgrade` / 干净重刷都不再丢。~~**workflow 无需改动**(make 自动并入)。~~

> 🔴 **2026-08-23 订正:「workflow 无需改动」正是问题本身,现已改掉。**
>
> 「make 自动并入」= **无条件**并入。而本目录是给 **xr1** 写的:`wg-failover-watchdog` 写死
> `ivpn_mad ivpn_lis ivpn_tor`、`rc.local` 写死 `192.168.1.0/24`(096 **迁移前**的 LAN)。
> 于是用同一个 workflow 给**别的设备**编「干净底包」时,这些东西**静默**跟着烤进去 ——
> xr1-cn 刷完后 `/etc/rc.local` 带着一个本 fleet 里根本不存在的网段、
> `/usr/sbin/wg-failover-watchdog` 在遍历本机没有的接口,就是这么来的。
>
> ⇒ `build-xr1710g.yml` 已新增 **`overlay` 输入**与一个裁剪步骤:
>
> | 取值 | 行为 |
> |---|---|
> | `xr1`（默认） | 全量烤入 —— **与历史行为完全一致**,不影响既有构建 |
> | `common` | 只留设备中立件(`README` / `lib/firmware` / `etc/hotplug.d/net` / `www`),剔除 xr1 专属件 |
> | `none` | 整个 `files/` 不烤 |
>
> `common` 带**反向校验**:剔除后若仍有不在设备中立清单里的文件,**显式失败** ——
> 防的是日后往本目录加了新东西、而 workflow 的分类清单没跟上就被静默烤进去。
> 三个取值都会**全量打印**最终写进 rootfs 的文件清单(不静默裁剪,也不静默全量)。
>
> **往本目录新增文件时,请同时更新 workflow 那一步的 `XR1_ONLY` / `COMMON_RE`**,
> 否则 `overlay=common` 的构建会直接失败(这是刻意的)。

## 装了什么(都是过去"刷完 overlay 就丢"要手动重贴的)

| 路径 | 作用 | 丢失场景 |
|---|---|---|
| `lib/firmware/regulatory.db` | 6GHz 补丁 regdb(去 NO-IR),md5 与设备现用一致 | 任何 sysupgrade(非 /etc) |
| `usr/sbin/wg-failover-watchdog` | WG 假死切换看门狗(recovery-log §23)。接口列表 = `ivpn_lis ivpn_mad ivpn_cph ivpn_sto`。**2026-07-31 已刷新到 v7 版**(自带锁 / tailscale 旁路自愈 + 事后复核 / ntfy 告警 / 主用判死触发 ivpn-rank / HEARTBEAT) | 任何 sysupgrade |
| `etc/hotplug.d/iface/99-tailscale-bypass` | **ifup/ifupdate 后自动补回 tailscale 旁路 pref-1000 rule**(recovery-log §28 D1:接口销毁重建会冲掉它们,而 `/etc/resolv.conf` 由 tailscale 独占指向 MagicDNS ⇒ 路由器本机域名解析静默全废;rc.local 只在开机跑,兜不住) | 任何 sysupgrade |
| `www/speedtest/**` | OpenSpeedTest 静态站(index 已改 Upload→CGI;含 cgi-bin/ost-upload) | 任何 sysupgrade |
| `etc/rc.local` | ① tailscale 旁路 kill-switch 打标(deploy-log §1;**开机那一次**,reload 场景归 hotplug)② 开机补生成 30MB 下载测试文件 | 干净重刷(config-keeping 会保留旧的) |
| `etc/uci-defaults/99-xr1710g-overlay` | 首刷一次:建 uhttpd :3000 实例 + tailscale 防火墙区 + wg cron + chmod 兜底 | 干净重刷 |

## ⚠️ 尚未烤进来、但**应该**烤的(2026-07-31 审计 D1)

`apply-vpn-xr1710g.sh` v7 还装了三个 `/usr/sbin` 脚本,它们**只在 overlay**:

| 路径 | 作用 | 现状 |
|---|---|---|
| `usr/sbin/ntfy-notify` | 告警发送器(绑健康隧道 + 查 HTTP 状态 + 边沿/冷却) | ❌ 不在本目录 |
| `usr/sbin/ivpn-rank` | 按实测重排优先级阶梯(cron */15) | ❌ 不在本目录 |
| `usr/sbin/ts-bypass-restore` | tailscale 旁路恢复的单一实现 | ❌ 不在本目录 |

四个脚本(含 watchdog)**都已加进 `/etc/sysupgrade.conf`** ⇒ **keep-settings** sysupgrade 能带过去。
但**干净重刷 / `sysupgrade -n` 仍会丢**,而丢掉之后的表现极具迷惑性:`/etc/crontabs/root` 保留
(cron 条目还在,每分钟去执行一个不存在的文件)、`/etc/ntfy.conf` 保留(凭据还在,"是不是配好了?"
照样过)⇒ **看起来一切正常,实际上告警、排序、旁路自愈全没了**。

⇒ **要真正免疫,需人工把这三个脚本同步进本目录 `usr/sbin/` 再重编固件。** 部署脚本**不做**这一步
(本目录在固件 fork 仓库里,且要跑一次 Actions)。同步时注意:它们由 `printf` 注入头部变量,
烤进固件的静态副本必须把变量值写死(参照 `usr/sbin/wg-failover-watchdog` 头两行的写法)。

## 刻意不烤的东西

- **`www/speedtest/downloading`(30MB)**:不进 git/镜像(会把固件撑大 30MB)。
  改由 `rc.local` 开机用 `/dev/zero` 现生成(LAN HTTP 无压缩,填零不影响测速)。
- **VPN/mwan3/kill-switch 主体**:仍由 `vpn-migration/apply-vpn-xr1710g.sh` 部署
  (含真值密钥注入,不该进固件)。本 overlay 只补它之外"刷完就丢的边角料"。
- **DHCP 保留 / LAN IP / 无线**:属站点配置,config-keeping sysupgrade 保留;干净重刷靠
  `apply-vpn` + 手动,不写死进固件。

## 用法

1. 把整个 `files/` 目录提交到 fork 仓库**根**(与 `Makefile`、`target/` 同级)。
   - git:`cp -r files/ <fork-checkout>/ && git add files && git commit && git push`
   - 或 GitHub 网页 "Add file → Upload files"(636K、38 个文件,可拖整个目录)。
   - ⚠️ 保留可执行位:`usr/sbin/wg-failover-watchdog`、`www/speedtest/cgi-bin/ost-upload`、
     `etc/hotplug.d/iface/99-tailscale-bypass` 要 `chmod +x`(git 会记录该位;uci-defaults 首刷也会兜底 chmod)。
2. Actions 跑 `build-xr1710g.yml` → 出的固件已内含以上。
3. 刷完开机自动就绪:`http://10.0.1.1:3000` 测速可用、6GHz 正常、tailscale 旁路生效、wg 看门狗在跑。

## 校验(刷完后)

```sh
md5sum /lib/firmware/regulatory.db          # 应 = 6bf90b37888231ac185f25ee26425050
ls -la /www/speedtest/downloading           # rc.local 已补生成 ~30MB
curl -s -o /dev/null -w '%{http_code}\n' http://10.0.1.1:3000/   # 200
ip rule show | grep '^1000:'                # tailscale 旁路两条(v4)
ip -6 rule show | grep 'pref 1000'          # v6 一条
uci get firewall.ts.name                    # tailscale
crontab -l | grep wg-failover               # 每分钟
ls -l /etc/hotplug.d/iface/99-tailscale-bypass   # 应存在且可执行
# hotplug 真触发自检(§28b 测A):删规则 -> ifup 垫底隧道 -> 应自动补回并打日志
ip rule del to 100.64.0.0/10 lookup 52 pref 1000; ifup ivpn_sto; sleep 8
ip rule show | grep -c 'lookup 52'          # 应回到 3
logread | grep ts-bypass | tail -1          # 应有 restored ... action=ifup
```

> 变更来源:2026-07-12,**2026-07-30 更新**(§28 轮换:watchdog 接口列表 + 新增 iface hotplug)。
> regdb 见 `../6ghz-regdb/`;speedtest/tailscale/wg 细节见
> `../recovery-log.md §23/§24/§25/§28/§28b`、`../tailscale/deploy-log.md`。
