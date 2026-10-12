# 后台管理（ProcessKill）

一款面向 Root 设备的后台管理与内存调节模块。按应用进程的 OOM 分值筛选后台进程，配合黑白名单、动态保护和低内存策略减少不必要的后台占用；同时提供可选的 ZRAM、Swapfile、VM 参数和 LMKD Hook 配置。

模块版本：**261004**
作者：**一只做梦的猫**

> 本仓库为捐赠版，捐赠版包含完整源码，可自行审计、编译和修改。捐赠仅出于自愿，捐赠后进 QQ 群：**945546357**。

---

## 效果

- 后台进程被清掉，可用内存上升
- mem 模式下只在内存低时才处理，配合白名单和前台保护，切回应用不容易重载
- 动态保护让反复被杀的应用进入保护期，不再来回杀，减少额外耗电和卡顿
- A0 LMKD Hook 拦住系统 lmkd 的杀进程调用，配合 ZRAM / Swapfile，后台更不容易被系统清掉
- 内存紧张时执行运存压制，回收后台内存、整理内存、压缩交换分区，缓解低内存卡顿；代价是可能带来短暂卡顿

## 开销

主服务是单个原生进程，用 epoll 事件驱动，空闲时阻塞等待，没有事件不醒。默认每 30 秒扫一次 `/proc`，开销随进程数变化，调大 `poll_interval` 可以降低频率。进程自身绑 CPU0；稳定的系统进程有 PID 缓存，不重复读取；热路径不调用重函数；日志每 10 轮裁剪一次，并保留固定行数上限。

日常待机时开销很小。明显占用资源的只有低内存触发运存压制时的一次性 CPU / IO 峰值，可能造成短暂卡顿。A0 脚本只在开机和改配置时各跑一次，LMKD Hook 是开机注入一次，之后几乎不占。

---

## 功能详解

### 1. 后台 OOM 压制

模块按应用进程的 OOM 分值筛出符合条件的后台进程，再按触发方式处理。

```
kill_trigger=poll
oom_threshold=800
poll_interval=30

# 仅 mem 模式生效
press_mem_percent=10
```

- `kill_trigger`：`poll` 到点就处理，`mem` 只在可用内存低于阈值时处理；默认 poll
- `oom_threshold`：OOM 阈值，数值越低压制范围越激进；默认 800
- `poll_interval`：检查间隔，单位秒，最低 10 秒；默认 30 秒
- `press_mem_percent`：可用内存低于该百分比才处理，范围 6～95，默认 10
- 支持处理应用主进程及其子进程，由 `deep_press` 控制，见下节
- 运行日志记录工作状态，便于排查是否命中规则

### 2. 浅度 / 深度压制

```
press_mode=shallow
deep_press=0
```

压制模式：

```
press_mode=shallow 浅度压制，过滤系统 / ROM 应用
press_mode=deep 深度压制，不过滤系统 / ROM 应用
```

主进程处理方式：

```
deep_press=0 只处理符合条件的子进程（推荐）
deep_press=1 同时处理符合条件的子进程和主进程
```

默认是浅度压制加只处理子进程，相对更稳妥。深度压制或主进程压制可能影响系统组件、推送服务或后台任务，请谨慎开启。

### 3. 黑白名单

名单文件：`/data/adb/modules/Process_Kill/黑白名单.txt`

- 正常填写包名或进程名：加入白名单
- 在包名或进程名前加 `!`：加入黑名单
- 黑名单应用在前台时仍受前台保护
- 名单以精确匹配为主，也支持 `*`、`?` 等通配写法

### 4. 前台应用保护与动态保护

```
protect_foreground=1
dyn_protect_enable=1
dyn_kill_threshold=3
dyn_protect_minutes=10
```

- 前台应用保护：默认开启，避免误处理当前正在使用的应用
- 动态保护：默认开启。某个后台应用在最近 5 轮内被处理达到设定次数后进入临时保护期
- `dyn_kill_threshold`：触发临时保护所需次数，范围 1～5，默认 3
- `dyn_protect_minutes`：临时保护时长，单位分钟，默认 10

动态保护针对被处理后立即被系统或自身服务拉起的应用，减少循环处理带来的额外耗电和卡顿。这类应用也可以自行判断是否加进白名单。

### 5. 低内存清爽后台

```
oldest_clean_enable=0
oldest_clean_mem_percent=10
```

开启后，可用内存百分比低于设定阈值时，优先停止驻留时间较久的普通后台应用，直到可用内存回到阈值以上。

- 默认关闭
- 触发阈值最低 6%，默认 10%
- 白名单、前台保护等规则仍应按实际使用场景配置

适合内存较小、后台堆积较多的设备；内存充足的设备通常无需开启。

### 6. 运存压制

> 功能来源：**海边会下雪么**、**魔威**。

```
heavy_reclaim_enable=1
heavy_reclaim_mem_percent=20
```

- 可用内存比例低于设定阈值时执行运存压制
- 默认开启，默认触发阈值 20%
- 阈值可设置范围 6～95
- 可在 Web 控制面板查看运存压制状态并重新加载配置

如果设备出现明显卡顿、后台频繁重启或应用切换体验变差，可先关闭此项对比测试。

### 7. A0 ZRAM 与 Swapfile 内存配置

> 功能来源：**魔威的 A0**。

A0 内存配置与后台压制相互独立，配置文件位于：

```
/data/adb/modules/Process_Kill/config/内存配置.conf
```

支持项目：

```
addon_enable=true A0 内存调节总开关
hook_lmkd=true 开机注入 LMKD Hook
zram=true 由模块重建并接管 zram0
zram_size= ZRAM 容量（MB）
comp_algorithm= ZRAM 压缩算法
swap=false 是否启用 /data/swapfile
swap_size=2047 Swapfile 容量（MB）
swap_priority=-2 Swapfile 优先级
swappiness= ZRAM / Swapfile 使用积极性
min_free_kbytes= 可选 VM 参数，留空不修改
extra_free_kbytes= 可选 VM 参数，留空不修改
watermark_scale_factor= 可选 VM 参数，留空不修改
```

- `addon_enable=false` 时，模块不调整 ZRAM、Swapfile、`swappiness` 和 VM 水位参数
- `zram=true` 时，模块会尝试重建 `zram0`，并按设备支持情况从配置算法或兼容备用算法中选择
- 默认 ZRAM 容量根据设备总内存自动生成，约为总内存的 1.5 倍
- `swap=true` 时使用普通实体 Swapfile `/data/swapfile`；默认关闭，避免自动占用存储空间
- 三项 VM 参数留空即保持系统原值
- 内存配置修改后由监听器重新应用；LMKD Hook 属于开机注入，修改 `hook_lmkd` 后需重启设备

### 8. A0 LMKD Hook

> 功能来源：**魔威的 A0**。

LMKD Hook 默认开启，在本次开机的启动流程中尝试注入，启用后阻止 LMKD 通过 `kill` 或 `pidfd_send_signal` 结束进程。

- Hook 注入失败不会阻止其余后台管理和内存功能启动
- 关闭 Hook 后，如本次开机已完成注入，通常需要重启设备才能恢复系统原有 LMKD 状态
- 注入记录可在 `logs/lmkd_hook.log` 查看

不同 Android 版本、内核、SELinux 策略和厂商 ROM 的兼容性不同；如遇异常，请先关闭该项并重启验证。

### 9. 日志与通知

```
log_max_lines=100
log_level=INFO
通知=1
```

- `log_max_lines`：日志最多保留行数，默认 100
- `log_level`：日志等级，`DEBUG / INFO / WARN / ERROR`，默认 INFO
- `通知`：1 为开启，开启后开机时通过系统通知提示模块的检验与运行状态；默认开启

---

## Web 控制面板

在支持模块 WebUI 的管理器中点击模块的 **Web** 按钮打开控制面板，可用于：

- 查看模块状态、内存信息和应用进程列表
- 修改 OOM 阈值、触发方式、压制模式、轮询间隔与日志等级
- 开关前台保护、动态保护、低内存清爽后台和运存压制
- 查看、刷新与清空模块日志
- 设置 A0 内存配置：ZRAM、Swapfile、压缩算法、`swappiness` 和 VM 水位参数
- 查看 LMKD Hook 与运存压制状态
- 调整界面显示设置

未授权时进入的是授权页，授权成功后切换为完整配置界面。

---

## 常用文件

```text
主配置：/data/adb/modules/Process_Kill/配置文件.txt
黑白名单：/data/adb/modules/Process_Kill/黑白名单.txt
主日志：/data/adb/modules/Process_Kill/日志.log
A0 内存配置：/data/adb/modules/Process_Kill/config/内存配置.conf
A0 调节日志：/data/adb/modules/Process_Kill/logs/a0_zram.log
LMKD Hook 日志：/data/adb/modules/Process_Kill/logs/lmkd_hook.log
```

---

## 安装

一般直接覆盖安装即可，不需要先卸载旧版，除非另有说明。

---

## 授权与获取授权码

模块采用设备 ID 校验，默认绑定一台设备，最多两台；后续每增加一台需要额外付费并证明该设备属于你。

获取授权码：

1. 从群文件下载 `设备ID.sh`，执行后把自己的设备 ID 复制出来
2. 在群里发送（设备 ID 和斜杠命令之间要有空格）：

```
/绑定 设备ID
```

3. 会返回对应授权码，忘记可以用 `/查询` 找回

检验失败请联系叹雪。其他管理员的发言和叹雪无关。

---

## 常见问题

**Q：这个模块和 Aurora 的进程压制一样吗？**
A：不一样，功能更多。

**Q：和 Aurora 的进程压制冲突吗？**
A：只是功能重叠，建议只开一个。

**Q：需要元模块吗？**
A：0814 之后的版本都不需要。

**Q：和 A0 有区别吗？**
A：功能不同。这里用了 A0 的一部分源码和功能，但不完全等于 A0；本模块是主动压制，A0 是被动压制。

**Q：修改配置需要重启吗？**
A：除了修改 `hook_lmkd`，其他都不需要。

**Q：开销大吗？**
A：不大，可以用 scene 查看相关进程的 CPU 开销。

---

## 捐赠

捐赠版与免费版功能有差异，捐赠仅出于自愿。

- 微信赞赏码：见 `捐赠.jpeg`
- GitHub Star 也是一种支持

---

## ⚠️ 兼容性与使用建议

本模块会影响后台进程处理、内存回收、ZRAM、Swapfile、VM 参数以及可选的 LMKD 行为。不同设备的 Android 版本、厂商系统、内核实现和 Root 方案不同，实际效果会存在差异。

- 使用第三方内存调节、Swap、ZRAM 或 LMKD 模块时，请避免重复修改同一项参数
- 更改 LMKD Hook 开关后请重启设备；遇到异常时可先关闭 A0 内存调节与 Hook，再逐项排查
- 如遇卡顿、闪退、系统服务异常或无法开机，请在 Recovery / 管理器中停用模块并查看日志

使用本模块前请理解各项配置含义。
