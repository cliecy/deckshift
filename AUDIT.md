# DeckShift 审计记录

- 上游：https://github.com/28allday/deckshift ，审计基于 commit `c7fb8b0`（与本机安装版本一致）
- 本机：Omarchy / Hyprland，AMD RX 9070 XT，LG ULTRAFINE 4K，xpadneo 蓝牙手柄
- 日期：2026-10-30

审计对象既包括仓库里的 `deckshift.sh` / `uninstall.sh` / 插件 QML，也包括它实际装到系统里的文件
（`/usr/local/bin/*`、`/usr/lib/os-session-select`、`/etc/sudoers.d/*`、polkit / udev 规则、sddm 配置）。

## 严重

### S1. sysctl 免密规则可直接提权到 root
`deckshift.sh:980-997` 写入 `/etc/sudoers.d/gaming-mode-sysctl`：

```
%video ALL=(ALL) NOPASSWD: /usr/bin/sysctl -w vm.swappiness=*
...
```

sudoers 里的 `*` 能匹配空格，所以这条规则并不只放行“设置 swappiness”，而是放行
`sysctl -w vm.swappiness=<任意> <任意更多参数>`。例如附带一个 `kernel.core_pattern=|<程序>`
就能让内核以 root 身份执行任意程序。结果是：**任何以本用户身份运行的进程（浏览器插件、游戏、
npm 包……）都能不输密码静默拿到 root**。`sudo -n -l` 已确认这些规则在本机生效。

`nvidia-smi -pm *` / `nvidia-smi -pl *` / `powerprofilesctl set *` 也是同样的写法（本机无 NVIDIA，
`powerprofilesctl` 危害较小，但写法同样不应该存在）。

**状态：已在 `audit/initial` 修复（仓库内）。**
- sysctl 规则全部删除（没有任何 DeckShift 脚本用过它们）
- 其余规则改成不带通配符的精确命令：`powerprofilesctl set performance|balanced|power-saver`、
  `nvidia-smi -pm 0|1`
- NVIDIA 功耗墙改由新的 root 辅助脚本 `/usr/local/bin/deckshift-nvidia-power-limit max|default` 处理，
  瓦数由脚本自己从驱动读取，不再接受调用者传入的参数
- sudoers 文件先用 `visudo -c` 校验再安装；重新安装时只要发现旧文件里有 `*`，就先删掉
  （即使用户在提示里选择跳过设置）

## 高

### H1. 从 Steam “退出到桌面”不会恢复系统休眠（今天黑屏的根因）
- `switch-to-gaming:40` 进入时 `systemctl mask --runtime sleep.target suspend.target …`
- 只有 `switch-to-desktop`（Super+Shift+R 路径）会 unmask
- Steam 菜单的“切换到桌面”走的是 `/usr/lib/os-session-select`（`deckshift.sh:2483` 起），
  它不 unmask、不重开蓝牙、也不还原电源状态（只能依赖 wrapper 的 trap）
- wrapper 的 `cleanup()` 也不 unmask

后果：
1. 游戏模式内 Steam 的“闲置 1 小时睡眠”变成假睡：系统休眠被拒绝（日志 `Unit suspend.target is masked`），
   Steam 只把屏幕关掉。2026-10-30 12:33 进入假睡，14:10 唤醒后界面没有重绘，黑屏，右键只出 CEF 菜单，
   直到 14:15:54 手柄连上才恢复。
2. 回到桌面后休眠一直被禁用，直到重启（当前 `systemctl is-enabled suspend.target` = `masked-runtime`）。

**状态：已在 `audit/initial` 修复（仓库内）。**
- 新增 `/usr/local/bin/deckshift-restore-desktop-state`，负责还原电源状态、解除休眠屏蔽、重开蓝牙，
  可以重复调用。`saved-state` 改为按固定键解析，不再 `source`（顺带解决 L1）
- 三条退出路径都会调用它：`switch-to-desktop`、`os-session-select`（Steam 菜单），以及 wrapper 的 `cleanup()`
- wrapper 启动 30 秒后解除休眠屏蔽，游戏模式里 Steam 的闲置睡眠会真正挂起系统，不再是假睡。
  如果这块显卡从挂起恢复有问题，可以在 Steam 里把闲置睡眠关掉

### H2. 会话切换靠 `pkill -9` 和 `systemctl restart sddm` 硬切
- `switch-to-gaming` 先 `pkill -9 gamescope`，再 `systemctl restart sddm`，直接把 Hyprland 整个会话打掉。
  每次切换 Hyprland 都在退出时段错误（`libaquamarine CDRMBackend::flushAsyncCommitEvents`，
  今天 4 次 coredump）。
- `os-session-select` 用 `nohup … systemctl restart sddm & disown`，而 `switch-to-desktop` 的注释
  明确说后台重启在 `Relogin=true` 下可能触发第二次重启、把刚起来的 Hyprland 杀掉，两条路径的做法互相矛盾。
- 免密规则只放行 `restart sddm`，等于本用户随时可以不输密码重启显示管理器。

**修复**：切换改成“写好 sddm 下次会话 → 让当前会话正常退出（`hyprctl dispatch exit` /
`steam -shutdown` 后 gamescope 退出）→ 由 sddm `Relogin=true` 自动登入目标会话”，去掉 restart sddm。

## 中

### M1. udev 规则把 CPU 调速器和 GPU 性能档位设成全局可写（0666）
`deckshift.sh:960-967`：`chmod 666 …/scaling_governor` 和
`…/power_dpm_force_performance_level`。本机任何用户、任何沙箱外进程都能改 CPU 调速器和 GPU 性能档位。
**修复**：改为 `chgrp video` + `0664`，或者交给 power-profiles-daemon 处理。

### M2. polkit 规则对 wheel 组无条件放行网络和磁盘操作
- `50-gamescope-networkmanager.rules`：`settings.modify.system`、`settings.modify.hostname` 等，
  wheel 用户无需认证就能改系统级网络配置和主机名
- `50-udisks-gaming.rules`：`filesystem-mount-system`、`encrypted-unlock`、`power-off-drive` 无需认证

这些规则不限会话（ssh 会话也适用），而游戏模式实际只需要 Wi-Fi 扫描/连接和挂载外接盘。
**修复**：收窄到 `subject.active && subject.local`，并去掉 hostname / system 级修改。

### M3. root 脚本使用 /tmp 下固定路径的标记文件
`gamescope-nm-start/stop` 以 root 身份使用 `/tmp/.gamescope-started-nm`。本机开了
`fs.protected_symlinks=1`，符号链接攻击已被缓解，但其他用户可以抢先创建这个文件，
让 root 去停掉 NetworkManager、重启 iwd。**修复**：改用 `/run/deckshift/`（root 拥有，0700）。

### M4. NetworkManager 相关规则只放行 wheel，其余规则放行 video
同一套功能，sudoers 有的放行 `%video`、有的放行 `%wheel`。不是 wheel 的用户进游戏模式时
`gamescope-nm-start` 会静默失败。属于一致性问题，顺手统一成一个专用组。

## 低 / 稳定性

- L1. `restore_balanced_mode` 直接 `source ~/.cache/deckshift/saved-state`（用户态，无提权风险，
  但应该用解析代替 `source`）。
- L2. `switch-to-gaming` 里 `sudo -n … 2>/dev/null` 把所有失败都吞掉了，出问题时日志里什么也没有。
- L3. Steam 启动时会按自己保存的状态先关掉蓝牙适配器再打开（`SetEnabled enable:0` → `1`），
  导致手柄晚 20 秒左右才连上，可以在 Steam 设置里固定蓝牙开启。
- L4. 大量针对 NVIDIA / Intel / 多显示器 / ChimeraOS 旧包的兼容分支，本机用不到，fork 后可以删掉以缩小审计面。

## 建议的修改顺序

1. S1：删除 sysctl 免密规则（立刻做，不影响功能）
2. H1：统一还原逻辑，修好休眠
3. M1–M3：收紧 udev / polkit / 标记文件
4. H2：重写会话切换流程
5. L4：删掉用不到的分支

## 注意

修仓库里的 `deckshift.sh` 不会改动已经装到系统里的文件，S1 / H1 要重新运行安装脚本才会生效。
重新安装之前，S1 涉及的 `/etc/sudoers.d/gaming-mode-sysctl` 应该先手动删除。

验证情况：生成的脚本都通过了 `bash -n`；新的 sudoers 内容通过了 `visudo -c`；还原脚本在桩环境
（替换掉 sudo / systemctl）里跑过一遍，调用顺序符合预期。还没有在真机上完整走过一次进入和退出游戏模式。
