# iOS 27.2 通道评估：为什么 AirTraffic 方案失效，以及还剩什么

> 本文是一次真机实测的记录，不是推测。
> 设备：iPhone19,7 · iOS **27.2 (24B119)**（作者验证环境是 27.0.1 / 24A446）
> 主机：macOS 27.0.1 (26A434) · Apple Silicon · `pymobiledevice3==11.32.0`
> 实测日期：2026-10-28

---

## 一、结论摘要

| 通道 | 27.0.1 | 27.2 (24B119) | 依据 |
|---|---|---|---|
| AirTraffic + AssetID `..` | ✅ 可用 | ❌ 静默忽略 | 见 §2.1 |
| AirTraffic + 目的地符号链接 | ✅ 可用 | ❌ 不跟随，在 Media 内建真实目录 | 见 §2.2 |
| 目的地相对 `..` / 绝对路径 / 前缀绕过 | — | ❌ 规范化后必须仍在 Media 内 | 见 §2.3 |
| AFC 符号链接 / 硬链接 | — | ❌ `OP_NOT_SUPPORTED` | 见 §3.1 |
| AFC `..` 穿越（Media、崩溃目录） | — | ❌ `INVALID_ARG` | 见 §3.2 |
| 解压器 zip-slip | — | ❌ 拒绝 | 见 §3.3 |
| 备份恢复（`mobilebackup2`） | 未测 | ⛔ 卡在 `BackupKeyBag` | 见 §4 |
| 越狱 / 降级 | — | 不在讨论范围 | — |

**一句话**：27.2 把"越界写入"统一收敛为「**目标真实路径必须仍在 `/var/mobile/Media` 内**」，同时 AFC 不再提供任何链接原语。不越狱、不降级的前提下，目前没有可用手段。

---

## 二、AirTraffic 逃逸在 27.2 的行为

原方案依赖两处缺陷：
1. `-[ATAirlock processCompletedAsset:]` 对 AssetID 不做路径校验，可用 `..` 逃出 `Media/Airlock/Book`
2. destination 只做字符串前缀检查 `hasPrefix:@"/var/mobile/Media/"`，不解析符号链接后的真实路径

### 2.1 源侧：AssetID 带 `..`

```
airlift.py read /var/mobile/Library/CallServices/Greetings/default/__probe__.caf
```

日志完整走完状态机（`SyncAllowed → ReadyForSync → AssetManifest → sync complete`），
但目标文件既没被搬走，也没有报错。**同步"成功"，动作没有发生。**

### 2.2 目的地：符号链接不再被跟随

用 `streaming_zip_conduit` 上传含相对符号链接的 zip，再让 ATAirlock 把 payload 搬到
`airlift-link-<tok>/<leaf>`（该链接指向 `../../../var/mobile/Library/...`）。

**对照实验**（关键证据）：

| 目标目录 | `listdir(link)` 结果 |
|---|---|
| `/var/mobile/Media/Books`（必定非空） | 只有我们自己写入的 `__p` |
| `/var/mobile/Library/Preferences` | 同样只有 `__p` |

结论：设备**没有跟随链接**，而是在 Media 里**新建了一个同名真实目录**，payload 落在里面。
读回的字节从未离开 Media——**任何"写入 Library 成功"的表象都是假阳性**。

### 2.3 目的地路径的解析顺序

传入 `/var/mobile/Media/../Library/Logs/CrashReporter/x.ips` 后，设备实际创建的是：

```
/var/mobile/Media/var/mobile/Library/Logs/CrashReporter/x.ips   ← 在 Media 内
```

说明解析顺序是：

```
去掉前导 "/" → 拼接 Media 根 → 词法规范化 → 校验仍在 Media 内 → 落盘
```

因此"合法前缀 + `..`"、"绝对路径"、"多级 `..`"三类变体全部无效。

---

## 三、其他写入通道排查

### 3.1 AFC 链接原语

`com.apple.afc` 的 `MAKE_LINK`（opcode 15）在 27.2 上**完全未实现**：

```
AfcException: [Errno AfcError.OP_NOT_SUPPORTED] Opcode: MAKE_LINK failed with status: 15
```

符号链接与硬链接都试过，Media 内、Media 外、绝对/相对目标全部同样结果。
（硬链接本是绕过符号链接检查的经典手法，此处不可用。）

### 3.2 AFC 路径穿越

- `com.apple.afc`（根 = `/var/mobile/Media`）：`../CallServices/...` → `INVALID_ARG`
- `com.apple.crashreportcopymobile`（根 = `/var/mobile/Library/Logs/CrashReporter`）：
  自己目录内**可写**，但 `../CallServices/...` 同样 `INVALID_ARG`

> 附带发现（与主结论无关但很重要）：`com.apple.afc` 的根是 `/var/mobile/Media`，
> **传入绝对路径会 `OBJECT_NOT_FOUND`**。仓库的 `tools/设备体检.py` 用
> `listdir("/var/mobile/Media")` 且把异常吞成 `[]`，因此它的"✅ 没有残留"
> 在 11.32.0 上是**假阴性**。替换流程本身不受影响（`airlift.py` 用的是相对路径）。

### 3.3 解压器 zip-slip

用条目名自带 `../`、`../../..`、绝对路径的 zip 测试：

- 相对条目名 → 未落盘
- 绝对符号链接目标 → `{'Error': 'ExtractionFailed'}`（解压层会拒绝逃逸目标）
- 相对符号链接在 `p0/p1/p2` 深度下可正常创建（`st_mode=40960`），但如 §2.2 所示不会被跟随

### 3.4 Media 内是否存在现成的越界链接

递归扫描（深度 4，跳过 DCIM/PhotoData 等）：**0 个符号链接**。没有现成梯子。

---

## 四、备份恢复通道（唯一"正门"）

这条不依赖漏洞：`mobilebackup2` 恢复时按设计要把 `HomeDomain`（根 `/var/mobile`）写回设备，
`Library/CallServices/...` 正在其中。

### 4.1 协议流程（读代码 + 实测确认）

```
Host → Restore{Options}
Device → DLMessageDownloadFiles(["<UDID>/Status.plist", "<UDID>/Manifest.plist"])
Host   → 提供文件
Device → 解析 Manifest.db，按条目索要 <fileID[:2]>/<fileID>
Host   → 逐个提供
Device → 写入目标路径
```

**关键点：清单是权威的。** 设备会按我们写的 `Manifest.db` 来要文件，
所以理论上可以只放一个条目，写到任意 `HomeDomain` 路径。

### 4.2 逐层关卡与错误码

| 关卡 | 报错 | 处理 |
|---|---|---|
| 查找我的 iPhone 开启 | `MBErrorDomain/211` Find My iPhone must be disabled before restoring | 设备侧关闭（本机曾因开关灰化无法关闭） |
| `SystemDomainsVersion` 不匹配 | `MBErrorDomain/203` Unsupported system domains version: X | 取一个设备接受的值（实测 3.0 / 5.0 通过，37.0 被拒——表现为上限校验） |
| `Status.plist` 缺版本 | `MBErrorDomain/203` Unsupported backup status version: 0.0 | 补 `Version: "3.3"` |
| 锁屏 | `MBErrorDomain/208` Device locked | 保持解锁 + 关闭自动锁屏 |
| 缺钥匙包 | `MBErrorDomain/205` No keybag in manifest | 需要 `BackupKeyBag` |
| 钥匙包非法 | `MBErrorDomain/1` Error decoding keybag … `MKBKeyBagCreateWithData error (-1)` | **无法伪造** |

### 4.3 为什么 keybag 造不出来

`BackupKeyBag` 是 Apple 的 TLV 钥匙包（`VERS/TYPE/UUID/HMCK/WRAP/SALT/ITER…`），
其中 `HMCK` 是用**设备 UID 密钥**计算的 HMAC。主机没有 UID 密钥，因此无法生成可被
`MKBKeyBagCreateWithData` 接受的钥匙包（已实测：结构完整的 TLV 仍报 `error (-1)`）。

真钥匙包只在**备份结束时**随 `Manifest.plist` 发给主机。中途截获不可行：

- 实测备份开始后 `Info.plist`（13 MB）、`Status.plist` 先到，`Manifest.plist` 为 0 字节
- 掐断后 `Manifest.plist` 始终为空 → 它在最后

### 4.4 结论

要在 27.2 上走通这条路，目前唯一已知的具体线索是：

1. 做一次**完整本地备份**（本机约 280 GB，数小时）
2. 从其 `Manifest.plist` 取出 `BackupKeyBag`
3. 塞进最小骨架，重跑 canary 恢复
4. canary 落地后再写静音件

即使第 3 步通过，第 4 步仍不保证（设备可能有其他校验）。**成本高、周期长、结果不确定。**

---

## 五、给上游的问题：`airlift.py` 在 27.2 上会谎报成功

这是本轮最值得回填的一个 bug（与 Apple 的修补无关，是工具自身的健壮性问题）。

```
$ airlift.py write /var/mobile/Library/Caches/__airlift_probe__.txt /tmp/probe.txt --logging
{"ok": true, "wrote": "...", "size": 25}
```

但设备上什么都没有。原因：`_atc_sync()` 只确认"状态机跑完"，不校验搬运结果。
在漏洞仍有效的系统上这没问题（搬运动作确实发生），但在被修补的系统上，
设备会**静默忽略**越界资产，于是工具报告成功。

**建议**：`write` / `replace` 在返回前做一次读回校验（`read` 同一路径并比对哈希）；
或至少把"设备未执行搬运"与"写入成功"区分开。AirBeep 的 v2.5.2 已为此加了
"提示音状态校验"并承认 iOS 27.x 上"写入成功但未生效"。

---

## 六、可复现清单

本轮产生的脚本与产物：

| 路径 | 用途 |
|---|---|
| `ios27-backup-research/build_canary_backup.py` | 生成最小备份骨架（单条目 canary） |
| `ios27-backup-research/canary-backup/` | 骨架产物（含 Manifest.plist/db、Status、Info、payload） |
| 崩溃报告 AFC（`com.apple.crashreportcopymobile`） | 本轮用作"写入是否真的落地"的读回哨兵 |

关键命令：

```sh
# 只读探针：让设备解析我们的清单（非破坏性）
python -m pymobiledevice3 backup2 info  <backup-dir>

# 保守恢复（不删、不动系统文件、不重启）
python -m pymobiledevice3 backup2 restore <backup-dir> \
  --no-reboot --no-remove --no-system --skip-apps
```

> 注意：`restore` 期间设备必须保持**解锁 + 不自动锁屏**，否则 `MBErrorDomain/208`。

---

## 七、设备现状

本轮所有探测均在 Media 内完成并已清理；`Books.plist` 恢复为实验前状态（原本不存在）；
提示音文件**未被修改**（27.2 更新已将原件还原，提示音重新可闻）。

---

## 附：错误码速查

| 码 | 含义 | 触发场景 |
|---|---|---|
| 203 | Error reading backup properties | 清单字段不合法（版本、状态版本） |
| 205 | No keybag in manifest | `Manifest.plist` 缺 `BackupKeyBag` |
| 208 | Device locked | 恢复/备份过程中锁屏 |
| 211 | Find My iPhone must be disabled | 查找未关闭 |
| 1 | Error decoding keybag | `BackupKeyBag` 非法（MKB 解析失败） |
