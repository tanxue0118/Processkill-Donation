# ProcessKill — 智能后台进程压制模块（捐赠版）

> 💝 **本版本为捐赠版**，感谢各位的支持与信任。捐赠版包含完整源码，可自行审计、编译和修改。
>
> 如您觉得本项目对您有帮助，欢迎通过以下方式支持开发者继续维护：
>
> - 🎁 微信赞赏码
> - ⭐ GitHub Star 也是一种支持
>
> 捐赠版与免费版功能有差异，捐赠仅出于自愿。

---

一个轻量级的 Android Magisk/KernelSU 后台进程管理守护进程，基于 OOM 分数自动压制后台进程，释放内存资源。

## ✨ 功能特性

| 功能 | 说明 |
|------|------|
| **OOM 阈值压制** | 当进程 `oom_score_adj` 达到阈值时自动 kill |
| **黑白名单** | 支持通配符匹配，白名单免杀、黑名单强杀 |
| **动态保护** | 若某进程在最近 5 轮中被杀次数过多，自动进入临时保护期，避免反复拉起增加功耗 |
| **低内存清爽后台** | 可用内存低于阈值时，自动 `force-stop` 最老的后台应用 |
| **前台保护** | 自动识别 `top-app`、`foreground`、`system-control` 组的进程并跳过 |
| **实时配置热加载** | 通过 `inotify` 监听配置文件修改，无需重启即生效 |
| **日志自动清理** | 超过最大行数自动截断，防止日志膨胀 |
| **累计统计** | 记录总压制次数，写入 `module.prop` 描述中展示 |

## 📁 文件结构

```
模块根目录/
├── processkill              # 主程序（编译产物）
├── service.sh               # Magisk 模块启动脚本
├── module.prop              # 模块属性（描述中会自动更新累计压制数）
├── 配置文件.txt              # 运行配置（自动生成）
├── 黑白名单.txt             # 进程黑白名单（自动生成）
├── 日志.log                 # 运行日志（自动生成）
└── 压制统计.txt             # 累计压制次数（自动生成）
```

## ⚙️ 配置说明

首次运行会自动生成 `配置文件.txt`，所有参数均可热修改：

```ini
# OOM 阈值（越高越激进，推荐 600~900）
oom_threshold=800

# 深度压制：1=杀所有符合条件的进程，0=只杀带冒号的子进程（推荐）
deep_press=0

# 轮询间隔（秒，最低 10）
poll_interval=30

# 日志最大行数
log_max_lines=100

# 是否记录每个被杀进程明细：1=开，0=关
verbose_kill_log=0

# 是否保护活跃进程组（top-app/foreground/system-control）
protect_foreground=1

# ---- 动态保护 ----
dyn_protect_enable=1
dyn_kill_threshold=3
dyn_protect_minutes=10

# ---- 低内存清爽后台 ----
oldest_clean_enable=0
oldest_clean_mem_percent=10
```

## 📝 黑白名单

编辑 `黑白名单.txt`：

```txt
# 白名单（免杀）
com.tencent.mm
com.tencent.mm:push
com.tencent.mobileqq
com.tencent.mobileqq:MSF

# 黑名单（以 ! 开头，无条件强杀）
#!com.example.badapp
```

- 支持 `fnmatch` 通配符，如 `com.example.*`
- 白名单进程永远不会被杀
- 黑名单进程无视 OOM 阈值直接 kill

## 🔧 编译

使用 Android NDK 交叉编译：

```bash
# 设置 NDK 工具链
export CC=aarch64-linux-android30-clang

# 编译
$CC -O2 -Wall -static -o processkill processkill.c
```

> **注意**：代码依赖 `sys/system_properties.h` 和 `sys/inotify.h` 等 Linux 特定头文件，仅适用于 Android/Linux 环境。

## 🧠 运行机制

```
┌─────────────────────────────────────────┐
│              主循环 (epoll)              │
│                                         │
│  ┌──────────┐    ┌───────────────────┐  │
│  │ timerfd  │    │     inotify       │  │
│  │ 定时触发  │    │  配置文件监听      │  │
│  └────┬─────┘    └────────┬──────────┘  │
│       │                   │             │
│       ▼                   ▼             │
│   do_press()        load_config()       │
│   遍历 /proc         load_list()        │
│   检查 OOM 分数      热更新配置          │
│   执行压制策略                           │
│       │                                  │
│       ▼                                  │
│   do_oldest_clean_if_needed()            │
│   低内存时清理最老后台应用               │
└─────────────────────────────────────────┘
```

### 压制流程

1. 扫描 `/proc` 下所有进程
2. 读取每个进程的 `cmdline`、`oom_score_adj`、`cpuset`、`cgroup`
3. 按以下优先级判断：
   - **黑名单** → 无条件 kill
   - **动态保护期** → 跳过
   - **前台/活跃组** → 跳过（若开启保护）
   - **OOM ≥ 阈值 或 在后台组** → 候选
   - **白名单** → 跳过
   - **深度模式关闭且为主进程** → 跳过
4. 对候选进程发送 `SIGKILL`
5. 更新动态保护状态
6. 若低内存功能开启，循环 `force-stop` 最老后台应用直到内存恢复

### 动态保护

使用 5 位滑动窗口记录每个进程最近 5 轮的被杀情况：

```
轮次:  [4] [3] [2] [1] [0]
        1   0   1   1   1  → 被杀 4 次
```

当被杀次数 ≥ `dyn_kill_threshold` 时，该进程进入保护期（默认 10 分钟），避免反复拉起造成的额外功耗。

## 📊 性能优化

- **CPU 亲和性**：绑定到 CPU0，减少大小核调度开销
- **进程名缓存**：系统包类型查询结果缓存（最多 512 条）
- **日志按需写入**：仅在每 10 轮检查一次日志行数
- **属性更新节流**：每 99 次压制才更新一次 `module.prop`
- **epoll 事件驱动**：无需 busy-wait，空闲时零 CPU 占用

## ⚠️ 注意事项

- 本模块需要 **root 权限** 运行
- 仅适用于基于 Linux 内核的 Android 系统
- `deep_press=1` 模式下会杀掉所有超过 OOM 阈值的进程（包括主进程），请谨慎使用
- 默认模式 (`deep_press=0`) 只杀带冒号的子进程（如 `com.app:service`），更加安全
- 压制统计达到 100000 后自动归零

## 📜 License

MIT License
