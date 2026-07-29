# home-nas —— Claude Code 家庭 NAS 搭建运维 Skill

把家庭 NAS（x86 小主机/旧 PC）搭成全自动媒体中心的实战经验封装成 Claude Code skill：
自动化媒体管线（Prowlarr→Sonarr/Radarr→qB→Bazarr→Jellyfin）、Immich 相册、Navidrome 音乐、
TaleBook 书库、AdGuard Home 去广告、Scrutiny 磁盘监控、备份自动化，
含中国大陆网络环境（镜像源/代理/TMDB 被墙）和弱 CPU 硬件（1080p 直放策略）的全部坑与解法。

## 用法

```bash
mkdir -p ~/.claude/skills/home-nas
cp SKILL.md ~/.claude/skills/home-nas/
```

之后在任何项目里对 Claude 说「帮我搭一个家庭 NAS 媒体中心」「我的 Jellyfin 不刮削了」等，
skill 会自动加载这套手册。

## 来源

提炼自真实运维记录：HP MicroServer Gen10（1核2线程/8GB/10TB+4TB）跑 20 个常驻容器，
照片/视频/音乐/书四条内容线全自动，并包含 Linux Docker 原生性能、容器内存保护、Immich ML 定时启停、`nas.local` 零 DNS 单点入口、中文字幕多源策略等经验。详细评估见 [../nas-review.md](../nas-review.md)，
脑暴清单见 [../nas-brainstorm.md](../nas-brainstorm.md)。
