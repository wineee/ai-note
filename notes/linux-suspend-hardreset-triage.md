# Linux 随机重启 / 唤醒失败：取证、定位与处置

> 实例环境：Lenovo YOGA Pro 14s ASP9（83HN，AMD Ryzen AI 9 365 / Radeon 860M，DCN 3.5）、Deepin 25、内核 7.3.0-amd64-desktop-rolling。

## 一、现象

- 机器毫无征兆重启，journal 突然截断，没有任何关机流程日志
- 合盖/待机后唤不醒，电源灯闪、屏幕黑、按键无响应，只能长按电源键
- 用户感知是"偶发"，实际是常态（本例一年 159 次）

## 二、第一步：判断"正常重启"还是"硬件硬复位"

```bash
# 1) 上次启动有没有关机流程（正常重启必有这三段，硬复位一条都没有）
sudo journalctl -b -1 | grep -cE "Shutting down\.|Journal stopped|systemd-shutdown"
# 2) 本次开机时内核解码出的【上次复位原因】
sudo dmesg | grep -i "reset reason"
# 3) 启动列表（硬复位通常只隔 20~40 秒就重新开机）
sudo journalctl --list-boots
```

AMD 平台复位原因对照（`Previous system reset reason [0x...]`）：

| 值 | 含义 | 性质 |
|---|---|---|
| `0x00080800` | 软件写 CF9 复位寄存器 | 正常重启 |
| `0x00200800` | ACPI 电源状态切换 | 正常开关机 |
| `0x00800800` | internal CPU shutdown event | 硬件致命（常见于 s2idle 唤醒失败） |
| `0x08000800` | uncorrected error → data fabric sync flood | 硬件致命（PCIe 错误升级而来） |

配套排除项：`journalctl -k | grep -c "Kernel panic"`、`ls /sys/fs/pstore/`（空说明没开 panic 转储，panic 无法 100% 排除）、OOM、GPU 复位、MCE/EDAC、SSD 的 `Unsafe Shutdowns`。

## 三、第二步：拿长期数据，别用最近几次启动推断规律

**最大的方法论教训。** journald 有容量上限，被高频日志刷爆后会 vacuum 掉历史。只看最近 4 次启动的 `PM: suspend entry` 计数为 0 就断定"这台机器没待机过，唤醒失败不适用"，结论完全错。

`/var/log/wtmp` 是二进制 utmp 记录（384 字节/条），**journald 不动它**，`last` 命令没装也能直接读：

```python
import struct
REC = 384
def parse(path):
    out = []
    with open(path, 'rb') as f:
        while True:
            b = f.read(REC)
            if len(b) < REC: break
            ut_type, = struct.unpack_from('h', b, 0)
            user = b[44:76].split(b'\0')[0].decode('utf-8', 'replace')
            host = b[76:332].split(b'\0')[0].decode('utf-8', 'replace')
            tv, = struct.unpack_from('i', b, 340)
            out.append((tv, ut_type, user, host))
    return out
# type=2 且 user='reboot'   → 开机
# type=1 且 user='shutdown' → 正常关机（pid=0 表示系统级 RUN_LVL）
# 每次开机前若没有 shutdown 记录 = 硬复位
```

统计出来后的价值远大于单次日志：本例 **410 次开机、159 次（38.8%）无任何关机流程**，且集中在 **13:00–13:15 共 40 次**（13:04 一分钟 12 次），对照同时段正常关机仅 23 次。这种分布不可能是固件随机故障，只能是人的作息——午休回来发现死机、长按电源重启，**直接锁定了触发场景**（空闲/待机）。

同法按月统计可判断"是最近才坏还是一直坏"（本例 2026-03 高达 68%，全年常态）；按"上次开机到本次硬复位的间隔"分桶可反推触发间隔（本例峰值 2–8 小时 = 上午开机到午休回来）。

## 四、第三步：分层定位故障在内核哪一层

`/sys/power/pm_test` 分五层，每层进入后 5 秒自动醒：

```bash
for lvl in freezer devices platform processors core; do
  echo "$lvl" | sudo tee /sys/power/pm_test >/dev/null
  echo mem  | sudo tee /sys/power/state >/dev/null
  sleep 8            # 必须等，见第八节坑 2
done
echo none | sudo tee /sys/power/pm_test   # 记得复位！
```

| 卡在 | 归属 | 可修性 |
|---|---|---|
| freezer | 进程/文件系统 | 能 |
| devices | 设备驱动 | 能（dmesg 会点名设备） |
| platform | ACPI/EC 固件 | 只能等 BIOS |
| processors / core | CPU/SoC S0i3 | 只能等 BIOS |

配套看 `/sys/power/suspend_stats/`（`success` / `fail` / `failed_devices` / `last_failed_dev`）。

本例 `freezer`、`devices` 全过、`fail=0`、无设备失败 → **软件栈干净**，故障被逼到 `platform` 之下，即 ACPI 与 SoC 的 S0i3 进入路径。`platform` 及以下不建议再测：结论恒定（只能等 BIOS），而风险是真实挂死。

**先确认有没有 S3**：

```bash
cat /sys/power/mem_sleep      # 只有 [s2idle] 表示没有 S3 可退
sudo dmesg | grep -iE "ACPI: \(supports|Low-power S0"
```

只报 `S0 S4 S5` 时，`mem_sleep_default=deep` 等所有 S3 相关参数**全部无效**，别浪费时间。

## 五、SoC 与 GPU 侧证据

```bash
sudo cat /sys/kernel/debug/amd_pmc/smu_fw_info    # 需 amd_pmc 模块
```

`Last S0i3 Status: Unknown/Fail` + `Table Version: 0` = SMU 侧从未成功进入 S0i3。**但不能单独归因于固件**：若下游 amdgpu 卡住，SoC 同样完不成进入，统计一样是 Fail。

### amdgpu DCN3.5 的 IPS 回归（容易被漏掉）

同型号"唤不醒"在内核 bugzilla 有独立报告（如 219445），bisect 出的坏提交：

```
f6098641d3e1  drm/amd/display: fix s2idle entry for DCN3.5+
```

意图是"进 D3cold 前必须先进入 IPS"，但在 Ryzen AI 300（DCN 3.5）上导致 s2idle 进入/退出挂死。确认自己的 DCN 版本：

```bash
sudo dmesg | grep -i "Display Core"   # 例：Display Core v3.2.392 initialized on DCN 3.5
```

对应开关 `amdgpu.dcdebugmask`（位定义见内核 `drivers/gpu/drm/amd/include/amd_shared.h` 的 `DC_DEBUG_MASK`）：

| 值 | 名称 | 用途 |
|---|---|---|
| `0x10` | `DC_DISABLE_PSR` | PSR 导致花屏/随机卡死时的首选 |
| `0x600` | PSR + Panel Replay | 有报告修好锁屏/DPMS 卡死，但真 s2idle 仍失败 |
| `0x800` | `DC_DISABLE_IPS` | **直接关掉 IPS**，绕开上面那条回归路径 |
| `0x1000` | `DC_DISABLE_IPS_DYNAMIC` | 平时关 IPS、**进入 suspend 时仍开**（对本问题无用） |

`/sys/module/amdgpu/parameters/dcdebugmask` 是 `-r--r--r--`，**运行时改不了**，只能加内核参数重启。

## 六、加内核启动参数：Deepin/UOS 用 grub.d 片段

**不要改 `/etc/default/grub`，新增 `/etc/default/grub.d/99_xxx.cfg` 片段。**

```bash
# 0) 备份
sudo cp /boot/grub/grub.cfg /boot/grub/grub.cfg.bak-$(date +%Y%m%d)

# 1) 写片段（变量必须用 $ 引用原值，否则会把已有参数全丢掉）
sudo tee /etc/default/grub.d/99_myparams.cfg >/dev/null <<'EOF'
# 临时缓解（逐条可删）：
# 1) pcie_ports=native        —— 让内核接管 PCIe 端口服务与 AER，把 sync flood 降级为可恢复的 AER 报错
# 2) amdgpu.dcdebugmask=0x800 —— DC_DISABLE_IPS，绕开 DCN3.5 上 IPS 先于 D3cold 导致的 s2idle 挂死
GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT pcie_ports=native amdgpu.dcdebugmask=0x800"
EOF

# 2) 重新生成
sudo update-grub

# 3) 验证（普通用户 grep 会报"权限不够"，属正常）
sudo grep -c "amdgpu.dcdebugmask=0x800" /boot/grub/grub.cfg     # 应等于内核条目数
sudo grep -m1 "amdgpu.dcdebugmask" /boot/grub/grub.cfg | tr ' ' '\n' | tail -5

# 4) 重启后确认生效
cat /proc/cmdline | grep -o "amdgpu.dcdebugmask=0x800"
```

为什么用片段：
- `/usr/sbin/grub-mkconfig` 会 `source /etc/default/grub.d/*.cfg`，且**按文件名顺序在 `/etc/default/grub` 之后**加载 → `99_` 一定最后生效
- `/etc/default/grub` 由安装器生成；`11_dde.cfg` 还会被 dde-daemon 的 `org.deepin.dde.Grub2` 重写（覆盖 `GRUB_DEFAULT`/`GRUB_TIMEOUT`，但不碰 CMDLINE，所以 99 片段安全）
- 直接改 `/boot/grub/grub.cfg` 会在内核更新时被冲掉，片段是持久的

关键点：
- **必须 `$GRUB_CMDLINE_LINUX_DEFAULT` 自引用**，写成等号覆盖会把 `splash quiet locales=...` 等系统参数一起抹掉。
- 命中数 = **内核条目数**。8 个 menuentry 里 7 个是内核（另一个是 UEFI 固件设置），所以是 7 不是 8。
- **`GRUB_TIMEOUT=1`** 是 Deepin 默认值，出问题只有 1 秒停在菜单。怕起不来就临时在同一片段加 `GRUB_TIMEOUT=5`。
- 撤销：删掉片段里的参数（或整个文件）后 `sudo update-grub`，重启即恢复。**必须删片段**，改 `grub.cfg` 不持久。
- 验证生效前先看 `/proc/cmdline`，别拿"改完还没重启的 grub.cfg"当已生效。

### 参数候选与取舍

| 参数 | 针对 | 备注 |
|---|---|---|
| `amdgpu.dcdebugmask=0x800` | s2idle 挂死（DCN3.5 IPS 回归） | **先试这个**，免费、可逆、直接打靶 |
| `pcie_ports=native` | sync flood（`0x08000800`） | 固件经 `_OSC` 拒绝把 AER 交给 OS 时，PCIe 错误会升级成 data fabric sync flood。`dmesg \| grep _OSC` 可见 `platform does not support [... AER]`。加此参数能把致命复位降级为普通 AER 报错（同时段会出现 `AER: Corrected error received: 0000:xx:00.0`，那个设备地址就是报修证据） |
| `processor.max_cstate=1` | 深度节能 | 同类案例两轮测试判为**无效**；且需先确认 `cpuidle` 真有深 C-state，否则只是涨功耗 |
| `cpufreq.default_governor=performance` | 频率波动 | 若已全是 performance，加了是空操作（`cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor \| sort -u`） |
| `nvme_core.default_ps_max_latency_us=0` | NVMe APST | 低风险但不对症，可后叠 |

## 七、刷 BIOS：Linux 单系统怎么做

前提事实：联想消费本（IdeaPad / Yoga / 小新）**基本不进 LVFS**，`fwupd` 查不到；UEFI 设置里**没有**"从 U 盘选择固件"；官网只给 Windows `.exe`。
→ **"查"完全可以在 Linux 下做，"刷"需要 WinPE。**

### 1. 先确认机型（别只信 product name）

```bash
sudo dmidecode -s system-product-name   # 例 83HN
sudo dmidecode -s system-version        # 例 YOGA Pro 14s ASP9（中文名）
sudo dmidecode -s system-sku-number     # LENOVO_MT_83HN_BU_idea_FM_...
sudo dmidecode -t bios | grep -E "Version|Release Date|Revision"
```

国内/国外名字不同（`YOGA Pro 14s ASP9` = `Yoga Pro 7 14ASP9`），但**机型号（83HN）和 BIOS 前缀（PSCN）不变**，用机型号搜官方页面才准。

### 2. 拿版本历史：README 比页面信息量大

直链规律（消费本）：

```
https://download.lenovo.com/consumer/mobiles/<版本小写>.exe
https://download.lenovo.com/consumer/mobiles/<版本小写>.txt   ← README，含全量逐版本 changelog
```

**页面只显示"当前版本"的更新说明，README 里才有全量历史**（本例一次拿到 14→17→21→22→23→24 共 6 个版本）。核对完整性：`sha256sum xxx.exe` 与支持页公布的 SHA256 对比。

### 3. 解包看构件（判断能否在 Linux 下刷）

```bash
7z x -opscn24 pscn24ww.exe && ls pscn24/
```

| 文件 | 作用 |
|---|---|
| `Win<PSCNxxWW>.fd` | 固件镜像本体，`strings` 可见 `$PSCN24WW` |
| `H2OFFT-Wx64.exe` / `H2OFFT64.sys` / `FlsHook.exe` | Windows 刷写器 |
| `InterToolx64.efi` | UEFI Shell 刷写器（Insyde H2OFFT Shell） |
| `platform.ini` | 刷写策略配置 |

```ini
[AC_Adapter]
Flag=3            ; 3 = 同时校验电源与电量
BatteryBound=30   ; 电量必须 ≥30%
[Platform_Check]
Flag=1            ; 校验项目号，不对会拒绝（"This BIOS is not for your Notebook PC."）
```

### 4. 三条路径

| 路线 | 做法 | 评价 |
|---|---|---|
| **A. WinPE** | Hiren's BootCD PE 写 U 盘 → 进 PE → 跑 `<ver>.exe` | **推荐**，官方刷写器，逻辑与 Windows 一致 |
| B. UEFI Shell | FAT32 放 `.fd` + `platform.ini` + `InterToolx64.efi`，Shell 执行 | 产线工具，无官方支持，写坏返厂，不建议 |
| C. 不刷 | 先用内核参数规避 | 见第六节 |

硬条件：**接原装电源 + 电量 ≥30%**，全程不断电。

### 5. 决策前必须核的两件事

1. **单向门**：出现 `Do not downgrade BIOS version to V<n> or earlier` 就是有去无回（本例 24WW 明文禁止回退到 23WW 及更早）。
2. **changelog 是否真的覆盖你的问题**：本例 6 个版本**没有一条**提到 sleep/suspend/resume/S0i3/稳定性，只有安全 CVE 与 PI 版本号；论坛同机型用户在 V21 上依然"固件不支持任何待机状态"。"刷最新 BIOS 就能修好"需要证据，不能默认。

### 6. 成本更低的第三方证据（刷之前先查）

- 内核 bugzilla / `lore.kernel.org` / `ladr` 搜"机型 + suspend"——可能有现成 bisect 结论（本例的 `f6098641d3e1` 就是这样挖出来的）
- 官方论坛同机型主题——"某版本是否修好"的实测反馈（本例靠这个否掉了刷 BIOS 的预期）
- ArchWiki 该机型页面——常直接写明 fwupd 是否支持、刷写变通做法

## 八、踩过的坑

1. **`pm_test` 忘了复位**：测试中断后文件停在 `[devices]`，此后所有待机都变成"只冻结不睡觉"。出事第一件事 `echo none | sudo tee /sys/power/pm_test`。
2. **`echo mem` 之后的 5 秒里敲 sudo 会报"账户验证失败，您的账户是不是上锁了？"**。这是假故障：整个用户空间被冻结，恢复后终端会话/PAM 状态对不上。查证：`sudo faillock --user $USER`（应为空）、`sudo passwd -S $USER`（应为 `P`）。循环里 `sleep 8` 即可避免。
3. **`sudo ls /boot/... *.bak*` 报"没有那个文件"**：`/boot` 是 `drwx------ root`，普通用户 shell 无法展开 glob。必须 `sudo ls -la /boot/grub/` 直接列目录。
4. **普通用户 `grep /boot/grub/grub.cfg` 报"权限不够"**：`grub.cfg` 是 `-rw------- root`，属正常，不是失败。
5. **用最近几次启动推断长期规律**：见第三节，这是本次最大的判断错误来源。

## 九、处置顺序（性价比从高到低）

1. **改电源策略止血**（立刻生效、不用重启）——合盖就关机，直接消灭待机唤醒这一类：
   ```bash
   gsettings set com.deepin.dde.power line-power-lid-closed-action 'shutdown'
   gsettings set com.deepin.dde.power battery-lid-closed-action 'shutdown'
   gsettings set com.deepin.dde.power line-power-sleep-delay 0     # 0 = 永不休眠
   gsettings set com.deepin.dde.power battery-sleep-delay 0
   ```
   代价是合盖要重新开机，但原本也是"回来强制重启"，等于无损。
2. **加 `amdgpu.dcdebugmask=0x800`** 实测一次 `systemctl suspend`（免费、可逆）。醒了看 `Last S0i3 Status` 是否从 `Fail` 变 `Success`——变了说明是 GPU IPS 拖死，BIOS 不用刷。
3. **加 `pcie_ports=native`** 治 sync flood 那条支线（与上一条可同时加，互不干扰）。
4. **刷 BIOS**（第七节，需 WinPE，注意单向门）。
5. **都不行**：合盖关机 + 用休眠（S4）代替待机，然后报修。带上 `dmesg | grep "reset reason"` 的输出，连续出现 sync flood / internal CPU shutdown 属保修范围。

## 十、可能省下来的无用功

- ACPI 无 S3（只报 `S0 S4 S5`）时，所有 S3/`mem_sleep_default=deep` 相关参数无效。
- `processor.max_cstate=1` 在同类案例已被两轮测试判为无效。
- governor 已是 performance 时再加 `cpufreq.default_governor=performance` 是空操作。
- 若 `low-power-action='hibernate'` 但无 `resume=` 参数、且 swap < 内存 → 低电量休眠必然失败后直接掉电，应改成 `'suspend'` 或先配好 swap。
- 休眠可用条件快速自查：`cat /sys/power/state` 含 `disk`、`CONFIG_HIBERNATION=y`、`SwapTotal ≥ image_size`、`/sys/power/resume` 非 0。休眠走 S4 不碰 S0i3，是 s2idle 坏掉时的替代路径。
