# local_service

本机自建服务的父仓库。每个服务是一个独立仓库，用 **git submodule** 挂在下面，父仓库只记录子仓库的提交指针。

## 子仓库

| 目录 | 仓库 | 端口 | 用途 | 自启方式 |
| --- | --- | --- | --- | --- |
| `usb_wireless_network/` | <https://github.com/Wuyanyan648/usb_wireless_network> | 8787 | RTL8156 2.5G USB 网卡管理（CLI）+ 家庭网关 Web 控制台 | `usbnicd.service`、`usbnic-boot.service`（system 级）+ `~/.config/autostart/usbnic-host.desktop`（登录后按模式打开界面） |
| `screen/` | <https://github.com/Wuyanyan648/screen> | 8765 / 8760 | 显示器输出管理与亮度调节（8765）+ 本机服务门户（8760） | `screen-manager.service`、`screen-portal.service`、`usbnic-session.service`（systemd **用户**单元）+ `~/.config/autostart/screen-session.desktop`、`screen-portal-open.desktop` |
| `theme_manager/` | <https://github.com/Wuyanyan648/theme_manager> | 8790 | 统一终端 / 桌面 GTK / gedit / Trae 的配色与字体：预设 + 微调 + 备份还原 | `theme-manager.service`（systemd **用户**单元，`WantedBy=default.target`，不需要 autostart 桌面项） |
| `usb_device_manager/` | <https://github.com/Wuyanyan648/usb_device_manager>（私有） | 8795 | USB 拓扑 / 端口供电与带宽 / 开机枚举失败历史，按设备开关通信链路（`authorized` 0/1） | `usbdevd.service`（system 级 root 单元）+ polkit 规则 `49-usbdevd.rules` |

门户（8760）的服务清单在 `screen/portal/services.json`，新增服务时在那里追加一条，保存后 5 秒内热加载。

## 为什么用 submodule（取舍）

采用 submodule 而不是 monorepo：

- 每个服务本来就已有独立远端、独立历史和独立 README，迁成 monorepo 会破坏它们各自的发布与回滚粒度。
- 父仓库只保存指针，子仓库可以单独 clone / 单独在别的机器上跑。
- 服务之间的依赖很弱（各自独立进程、各自端口），没有需要跨仓库原子提交的场景。

代价（必须知道）：

1. clone 必须带 `--recursive`，否则子目录是空的。
2. 子仓库有新提交后，父仓库不会自动跟随：要回到父仓库再提交一次指针，别人才能看到新版本。
3. 子仓库的提交必须**先推送**到它自己的远端，否则父仓库的指针指向远端不存在的提交，别人 `submodule update` 会失败。
4. 在子目录里 Git 可能处于 detached HEAD，直接改代码容易丢提交；动手前先 `git switch main`。

## Clone

```bash
git clone --recursive https://github.com/Wuyanyan648/local_service.git
```

已经 clone 但忘了 `--recursive`：

```bash
cd local_service
git submodule update --init --recursive
```

## 子仓库更新后如何提交指针

```bash
cd local_service/usb_wireless_network
git switch main && git pull --ff-only
# 改代码、提交、推送（推到它自己的远端）
cd ..
git add usb_wireless_network
git commit -m "chore: bump usb_wireless_network"
git push
```

拉取别人的指针更新：

```bash
git pull
git submodule update --init --recursive
```

## 新增一个服务

1. 单独建一个仓库，不要直接把代码放进父仓库。
2. 在父仓库里登记：

   ```bash
   cd ~/eric/wuyan_github/local_service
   git submodule add <仓库地址> <服务名>
   ```

3. 目录规范：
   - 仓库根目录即服务根目录。
   - 启动脚本放 `scripts/` 或仓库根的 `run-service.sh`。
   - 安装 / 卸载脚本自己负责写 systemd unit 与 autostart，路径用脚本自身位置推导（`readlink -f "$0"` 再取上级目录），不要写死绝对路径。
   - 每个服务的端口、启动方式、卸载方式写进它自己的 README。
4. 自启放哪：
   - 需要 root（改网络、改显示输出等）：system 级 unit，`/etc/systemd/system/<name>.service`。
   - 只需要登录后 GUI 环境：`~/.config/autostart/<name>.desktop`，`Exec` / `Path` 必须是迁移后的绝对路径。
5. 门户登记：在 `screen/portal/services.json` 的 `services` 数组里追加一条（`id` / `name` / `desc` / `url` / `health` / `manage` / `unit`），保存后门户 5 秒内热加载。
6. 回到本 README 的子仓库表格补一行。

## 绝对路径清单（迁移后必查）

改目录后，下面这些地方不会自动跟随，必须手动更新，否则下次登录 / 重启就起不来：

- `/etc/systemd/system/usbnicd.service`（`WorkingDirectory`、`ExecStart`）
- `/etc/systemd/system/usbnic-boot.service`（`ExecStart`）
- `~/.config/autostart/usbnic-host.desktop`（`Exec`）
- `~/.config/systemd/user/{screen-manager,screen-portal,usbnic-session,theme-manager}.service`（`WorkingDirectory`、`ExecStart`、`Documentation`）
- `~/.config/autostart/{screen-session,screen-portal-open}.desktop`（`Exec`、`Path`）
- `/etc/systemd/system/usbnicd.service.d/10-screen-session.conf`（`ExecStop`，含 usb 仓库绝对路径）
- `/etc/polkit-1/rules.d/49-screen-usbnicd.rules`

重装方式：`usb_wireless_network/scripts/install-host-boot.sh`（system 级，需 root）、`screen/install-autostart.sh`（用户级，无需 sudo）、`theme_manager/install-autostart.sh`（用户级，无需 sudo），三者都按脚本所在位置推导路径，迁移后重跑一次即可。

## 迁移记录与回滚

- 迁移日期：2026-11-09
- 迁移内容：
  - `~/eric/wuyan_github/usb_wireless_network` → `~/eric/wuyan_github/local_service/usb_wireless_network`
  - `~/eric/wuyan_github/screen` → `~/eric/wuyan_github/local_service/screen`
- 迁移前备份（已从 `/tmp` 转入持久目录，共 7,061,207 字节）：
  - `~/.local/share/local_service-migration-backup/local_service_migration_backup_20261109_221230/`（usb_wireless_network，310 个文件，6,411,815 字节）
    - `repos/usb_wireless_network/`：完整仓库快照（含 `.git`）
    - `user_autostart/`：`usbnic-host.desktop`、`screen-manager.desktop`、`hdmi-watchdog.desktop`
    - `system_units/`：`usbnicd.service`、`usbnic-boot.service`
    - `git_state/`：迁移前后的 HEAD / 分支 / 远端 / 端口 / xrandr
  - `~/.local/share/local_service-migration-backup/local_service_migration_backup_20261109_223138/`（screen，147 个文件，649,392 字节）
    - `repos/screen/`：完整仓库快照（含 `.git`）
    - `user_systemd/`：`screen-manager.service`、`screen-portal.service`、`usbnic-session.service`
    - `user_autostart/`：`screen-session.desktop`、`screen-portal-open.desktop`
    - `git_state/`：迁移前后的 HEAD、`before.txt`、`xrandr_before.txt`、`ports_before.txt`、`health_before.txt`

回滚步骤（`$BK` 指向持久备份目录，直接复制可跑）：

```bash
BK=~/.local/share/local_service-migration-backup

# 1. 把目录移回原位
mv ~/eric/wuyan_github/local_service/usb_wireless_network ~/eric/wuyan_github/usb_wireless_network
mv ~/eric/wuyan_github/local_service/screen ~/eric/wuyan_github/screen

# 2. 恢复自启配置
cp $BK/local_service_migration_backup_20261109_221230/user_autostart/usbnic-host.desktop ~/.config/autostart/
cp $BK/local_service_migration_backup_20261109_223138/user_autostart/*.desktop ~/.config/autostart/
cp $BK/local_service_migration_backup_20261109_223138/user_systemd/*.service ~/.config/systemd/user/
sudo cp $BK/local_service_migration_backup_20261109_221230/system_units/*.service /etc/systemd/system/

# 3. 重载并重启
systemctl --user daemon-reload
systemctl --user restart screen-manager.service screen-portal.service
sudo systemctl daemon-reload
sudo systemctl restart usbnicd.service

# 4. 父仓库里把子模块记录撤掉（只影响父仓库，不动已移回的目录）
cd ~/eric/wuyan_github/local_service
git rm --cached screen usb_wireless_network
```

备份在仓库目录之外，不会被父仓库跟踪；需要腾空间时直接删 `$BK` 下对应时间戳目录即可。
