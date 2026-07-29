---
name: home-nas
description: 家庭 NAS/旧服务器搭建与运维指南。当用户想在家用服务器（群晖以外的 x86 小主机、旧 PC、HP MicroServer 等）上搭建自动化媒体管线（*arr 全家桶 + Jellyfin）、相册(Immich)、音乐(Navidrome)、书库(TaleBook/calibre)、去广告(AdGuard Home)、磁盘监控(Scrutiny)、备份自动化时使用；尤其适用于中国大陆网络环境（镜像源、代理、TMDB 被墙）和弱 CPU 硬件的取舍判断。
---

# 家庭 NAS 搭建与运维手册

实战经验提炼自一台 HP MicroServer Gen10（1核2线程弱 CPU / 8GB / 10TB+4TB 双盘）上跑 20 个常驻容器的完整家庭媒体中心。

## 一、先问清三件事再动手

1. **硬件底细**：CPU 型号（决定能不能碰 4K/转码）、内存、盘位与容量。弱 CPU（<4 线程）铁律：**全走 1080p Direct Play，质量配置锁 HD-1080p，禁止 4K/REMUX-2160p**——4K 文件常带 TrueHD/Atmos 音轨，电视不认就触发转码，弱 CPU 转码=卡死。瓶颈永远在 NAS 的 CPU，不在电视。
2. **网络环境**：国内环境 docker.io 直连基本不通、TMDB 被墙、GitHub release 超时。镜像方案：`ghcr.nju.edu.cn`（ghcr 镜像）、`docker.1ms.run`（docker.io 镜像，拉完 retag）、apt/pip 用 USTC 源。
3. **权限模型**：用户有无 sudo？无 sudo 时，可利用用户的 Docker 权限临时启动一个特权容器，把宿主机根目录挂到 `/host`，再用 `chroot /host` 让命令在宿主机文件系统里执行：
   `docker run --rm --privileged --pid=host -v /:/host --entrypoint /bin/bash <任意可信镜像> -c "chroot /host <cmd>"`
   大白话：普通用户打不开宿主机“总电箱”，但 Docker 组能创建一个拿到总钥匙的临时维修间；`-v /:/host` 把整栋房子放进维修间，`chroot /host` 再把视角切回整栋房子。**这等价于 root 权限，只能用可信镜像，执行完必须 `--rm` 自动销毁，绝不用于日常命令。**

## 二、目标架构（按此顺序搭建）

```
Prowlarr(索引器) → Sonarr/Radarr(剧集/电影管理) → qBittorrent(下载)
                        ↓ 完成后通知                    ↓ 硬链接到媒体库
                   Bazarr(中文字幕)              Jellyfin(刮削+串流) → 手机/PC/电视
```

1. **qBittorrent** → 2. **Prowlarr**（索引器统一管理，自动同步给 *arr）→ 3. **Sonarr + Radarr**（根目录 `/tv` `/movies`，下载目录 `/downloads`，**remote path mapping 必须配**）→ 4. **Bazarr**（中文优先 zimuku；OpenSubtitles.com 必须填账户 username，不能填登录邮箱，Bazarr 自带应用 API key；免费额度低时补 `embeddedsubtitles`、`subf2m`、`yifysubtitles`、`bsplayer`。部分老中文源如 assrt/shooter 在当前版本被强制禁用）→ 5. **Jellyfin**（**重建容器必须带代理 env**：`HTTP_PROXY/HTTPS_PROXY=http://<宿主机IP>:7890`，否则 TMDB 刮削静默失败，不报明显错误）。
6. 质量配置：全部切 **HD-1080p（profile 4）**，已抓的 4K 队列项 `DELETE /api/v3/queue/<id>?removeFromClient=true&blocklist=true` 再重搜。

## 三、周边服务食谱（每个都有坑）

| 服务 | 关键经验 |
|---|---|
| **Immich**（相册） | 备份 = rsync 照片库 + `docker exec immich_postgres pg_dump` 在线 dump（留 7 天），**这是全家唯一不可再生数据，必须每日自动备份到第二块盘**。8GB 小内存可给 server/ML/Postgres/Redis 分别设 1.5G/2G/1G/256M 上限；ML 平时停止，用 systemd timer 每日启动 90 分钟消化智能搜索/人脸任务，能省约 1.1G 常驻 RAM |
| **Navidrome**（音乐） | 0.63 起扫描调度写 `/data/navidrome.toml` 的 `[Scanner] Schedule="@every 1h"`，旧的 ND_SCANSCHEDULE 环境变量已失效 |
| **B 站收藏夹同步** | yt-dlp 必须带 `--concurrent-fragments 8`，否则被限速到 12KiB/s；cookie 文件 chmod 600 |
| **TaleBook**（书库） | 本质是 calibre。批量管理用容器内 `calibredb`（库路径 /data/books/library）；用户自助传书：scp 到 imports/ → 网页扫描；OPDS 给手机阅读器用，**URL 末尾必须带斜杠** `/opds/` |
| **Heimdall**（导航页） | 改配置直接改 `config/www/app.sqlite`（先备份）；`window_target` 必须设 current，iframe 会被各服务 X-Frame-Options 挡成白页 |
| **Uptime Kuma**（监控） | 批量加监控可直接 INSERT kuma.db 后 restart；告警渠道推荐 Server酱（微信） |
| **AdGuard Home**（去广告 DNS） | 可预写 AdGuardHome.yaml 免向导（密码要 bcrypt 哈希）；**容器不走 env 代理**，github.io 上的过滤器要用代理下载后 `docker cp` 进 `/opt/adguardhome/work/data/filters/<id>.txt` 再 restart；上游用阿里/腾讯 DoH |
| **Scrutiny**（磁盘 SMART） | 装完立刻看首次扫描——重映射扇区/待映射扇区(Current_Pending_Sector)非零且增长的盘要立刻计划更换。**待映射 3000+ = 随时会挂** |

## 四、备份策略（systemd timer，每日 03:00）

- 备份什么：**照片库+照片 DB（不可再生）> 所有服务配置（重建痛苦）> 媒体库（可重下，不备）**
- `nas-backup.timer` → `nas-backup.sh`：rsync --delete 各配置目录 + pg_dump，日志记每步退出码
- 装任何新服务后第一件事：把它的配置目录加进备份脚本

## 五、Docker 性能、资源与统一入口

- **Linux Docker 是原生容器，不需要 Linux 虚拟机**：容器直接共享宿主机内核，`overlay2 + cgroup v2` 下开销接近普通进程。macOS 的 Docker Desktop/OrbStack 必须先运行 Linux VM；OrbStack 优化的是 VM 和文件共享层，Linux 上没有这一层需要替代。
- 8GB NAS 应为重点容器设上限，防止失控：Jellyfin/Immich Server 约 1.5G、Immich ML 2G、Postgres 1G、qB 768M、Sonarr/Radarr/Bazarr 512M、Prowlarr 384M。留出约 2G 给 Linux 页缓存。
- **入口不必依赖家庭 DNS**：运行 Avahi/mDNS，把主机名设成 `nas` 并限制只在物理网卡发布（否则可能发布 Docker 网桥 IP），再用 Caddy 占用 80 端口反代 Heimdall。局域网访问 `http://nas.local`；NAS 关机时只是该地址打不开，不会影响全家上网。
- Caddy 只做 LAN HTTP 入口；不要因此把 80/443 映射到公网。外部访问仍走 Tailscale。

## 六、安全基线（家用环境）

- **绝不公网端口映射**，外访只走 Tailscale（免费、免配置 NAT 穿透）
- 密钥纪律：API key 用的时候内联读配置文件（`grep -oP`），**绝不打印到输出**；临时凭据（如 Jellyfin 临时 ApiKey 插 sqlite）用完立即删
- 生成的密码不落明文：写 chmod 600 文件让用户自己看，只把哈希写进配置

## 七、排障速查

| 症状 | 病因 → 解法 |
|---|---|
| Jellyfin 不刮削元数据且日志无报错 | TMDB 被墙 → 容器加代理 env 重建 |
| Sonarr 搜索到但不下载 | 索引器没同步/质量配置不匹配 → 查 Prowlarr 同步和 profile |
| 下载完不进媒体库 | remote path mapping 没配，或硬链接跨盘失败 |
| 小米电视播 4K 卡死 | 触发了转码 → 换 1080p 版本（见第一节铁律） |
| Jellyfin 没有中文字幕 | 先分清“完全无字幕”与“下载包已有但未识别”。Bazarr OpenSubtitles.com 的 username 字段不能填邮箱；免费额度低时同时启用 zimuku、embeddedsubtitles、subf2m、yifysubtitles、bsplayer。部分中文 provider 在特定 Bazarr 版本会被 `PROVIDERS_FORCED_OFF`，不要硬开；无结果再考虑手动季包或 Mac 跑语音识别/翻译 |
| docker pull 超时 | 换 ghcr.nju.edu.cn / docker.1ms.run |
| sqlite 报 no such column | 版本 schema 差异，先 `\,.schema` 看真实列名再查 |

## 八、玩法路线图（搭完主线后按性价比补）

Scrutiny(磁盘命根子) → qB 手机遥控 → Vaultwarden(密码库) → Homepage(导航页) → Home Assistant(智能家居) → Alist(网盘聚合) → Memos(笔记)
