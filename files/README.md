# files/ —— 烤进固件的 overlay(消灭"每次刷机重贴")

OpenWrt/ImmortalWrt buildroot **自动把仓库根的 `files/` 目录复制进 rootfs**(squashfs 层)。
把本目录整体放到 fork(`zxa24/immortalwrt`)**仓库根**下叫 `files/`,再跑 `build-xr1710g.yml`,
这些东西就随固件烤进去,`sysupgrade` / 干净重刷都不再丢。**workflow 无需改动**(make 自动并入)。

## 装了什么(都是过去"刷完 overlay 就丢"要手动重贴的)

| 路径 | 作用 | 丢失场景 |
|---|---|---|
| `lib/firmware/regulatory.db` | 6GHz 补丁 regdb(去 NO-IR),md5 与设备现用一致 | 任何 sysupgrade(非 /etc) |
| `usr/sbin/wg-failover-watchdog` | WG 假死切换看门狗(recovery-log §23) | 任何 sysupgrade |
| `www/speedtest/**` | OpenSpeedTest 静态站(index 已改 Upload→CGI;含 cgi-bin/ost-upload) | 任何 sysupgrade |
| `etc/rc.local` | ① tailscale 旁路 kill-switch 打标(deploy-log §1)② 开机补生成 30MB 下载测试文件 | 干净重刷(config-keeping 会保留旧的) |
| `etc/uci-defaults/99-xr1710g-overlay` | 首刷一次:建 uhttpd :3000 实例 + tailscale 防火墙区 + wg cron + chmod 兜底 | 干净重刷 |

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
   - ⚠️ 保留可执行位:`usr/sbin/wg-failover-watchdog`、`www/speedtest/cgi-bin/ost-upload`
     要 `chmod +x`(git 会记录该位;uci-defaults 首刷也会兜底 chmod)。
2. Actions 跑 `build-xr1710g.yml` → 出的固件已内含以上。
3. 刷完开机自动就绪:`http://10.0.1.1:3000` 测速可用、6GHz 正常、tailscale 旁路生效、wg 看门狗在跑。

## 校验(刷完后)

```sh
md5sum /lib/firmware/regulatory.db          # 应 = 6bf90b37888231ac185f25ee26425050
ls -la /www/speedtest/downloading           # rc.local 已补生成 ~30MB
curl -s -o /dev/null -w '%{http_code}\n' http://10.0.1.1:3000/   # 200
ip rule show | grep '^1000:'                # tailscale 旁路两条
uci get firewall.ts.name                    # tailscale
crontab -l | grep wg-failover               # 每分钟
```

> 变更来源:2026-07-12。regdb 见 `../6ghz-regdb/`;speedtest/tailscale/wg 细节见
> `../recovery-log.md §23/§24/§25`、`../tailscale/deploy-log.md`。
