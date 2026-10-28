# 静音 iOS 通话录音提示音

> **不用装 App、不用越狱、不用签名、不用 VPN、不用配对文件 —— 只需要一根数据线。**
>
> 把 iPhone 用数据线连到 Mac，把系统通话录音的开始 / 结束提示音替换成等长静音，
> 并保留完整备份随时可还原。

**简体中文** · [English](README.en.md)

---

## ⚠️ 在把手机插上去之前，请先读完这一节

我理解你在担心什么——**把手机连到电脑上跑一份陌生人写的代码，谁知道会发生什么。**
所以这一节不讲好话，只讲事实，**并且告诉你怎样自己验证**。

### 它做了什么

- 把你 iPhone 上**两个音频文件**（通话录音的开始/结束提示音）替换成**时长完全一致的静音**
  - `/var/mobile/Library/CallServices/Greetings/default/StopDisclosure.caf`
  - `/var/mobile/Library/CallServices/Greetings/default/StartDisclosureWithTone.m4a`
- 替换前先把原件**读回你电脑**存起来，随时可还原

### 它没有做什么

- ❌ **没有联网**。我 grep 过整个 `airlift.py`：**没有任何 HTTP / urllib / requests / socket 调用**。
  全部通信都走 USB，只发生在你的手机和你的电脑之间
- ❌ **没有越狱**，没有提权，没有动 kernel
- ❌ **没有自动扫描**。改哪个路径由命令里**显式写死**，就是上面那两条
- ❌ **总共只碰媒体/用户数据这一层**，见下方「风险与边界」

### 它为什么够不到"变砖"

这是机制层面的论证，不是保证：

> 变砖需要**修改引导链（SecureROM / iBoot）、系统分区（APFS 快照）或 SEP**。
> 这些在**未越狱**的设备上受 **Secure Boot 与代码签名**保护，**根本写不进去**。
>
> 而本方案走的是 **AirTraffic**——Apple 自己的媒体同步协议。这个通道的**能力上限就是媒体分区**。
> 换句话说：**它不是"小心地不去碰系统分区"，而是这条通道本身没有那个权限。**

### 我不会向你保证的事

- 不保证 **Apple 未来不会在系统更新里还原这些文件**（实际上这类文件曾被静默更新过）
- 不保证 **在除了 iOS 27.0.1 (24A446) 之外的版本上一定能用**
- 不保证 `pymobiledevice3` 换版本后仍然兼容——**这个库迭代极快，我们已经因此踩过坑**
  （所以我们锁定了实测版本，见 `requirements.txt`）

### 你可以自己验证的三件事（推荐按顺序做）

**① 先审代码。** `macos/airlift.py` 一共 **764 行**，用编辑器过一遍。
里面 90% 不是我们写的——它来自公开的上游项目
[airlift](https://github.com/0xjohnnydev/airlift) / [AirliftSilence](https://github.com/YiHoooong/AirliftSilence)，
**我们只改动了"从哪里加载系统库"那约 70 行**，全部差异都在 `macos/enablement.patch` 里可以逐行对照。

**② 先只读试水，零风险。** 第 1 步的 `设备体检.py scan` **只读**设备媒体目录，
不触发任何写入，跑多少次都不会改变设备。**先跑它，看能不能正常连上、有没有报错。**

**③ 再看 `read` 那一小步。** 它是往返操作——**成功时文件会回到原位**。
如果它跑通了，说明整条链路是通的；如果它失败了，你会立刻知道，而且**此时你还没写任何东西**。

> **如果你不打算读代码，也不打算跑只读步骤——那就别用。** 这不是客套，是认真的建议。

### 🚫 什么情况下不要用

- **iOS 27.2 beta 3 或更高** —— Apple 已修补，**跑了也没用，别白折腾**
- **这是你唯一的主力机，且"手机出一点问题就完蛋"** —— 那就不值得。诚实地讲，这不是给所有人用的工具
- **你不熟悉终端命令** —— 请先找个懂的人一起看一遍

---

## 这是什么

iOS 18.1 起，Apple 加入了通话录音功能，但会在录音开始和结束时播放一段**语音提示**——
中文设备上说的是「此通话将被录音」，英文设备上是 "This call will be recorded"。
这段提示音**通话双方都能听到**，目的是告知对方。

本项目把这两个提示音文件替换成**时长完全一致的静音文件**，因此录音功能本身不受影响，
录音的开始 / 结束时机也不会变化。

### 与之前那些方案的区别

| | 依赖 iOS App 的方案（如 AirBeep） | **本项目** |
|---|---|---|
| iPhone 上装 App | 必须 | **不需要** |
| IPA 签名 / 免费账号 7 天限制 | 必须 | **不需要** |
| 本地回环 VPN（LocalDevVPN 等） | 必须 | **不需要** |
| 配对文件 | 必须 | **不需要** |
| 需要的东西 | 一堆前置条件 | **一台Mac+一根数据线** |

原因是我们找到了一个**更直接的入口**，见下方「原理」。

---

## 🙏 致谢：这个方案是用 DSH + DeepSeek 做出来的

**特别感谢 [大肥鱼 DeepSeek](https://www.deepseek.com/) 和 [DSH（DeepSeek Harness）](https://github.com/deepseek-ai/deepseek-harness)。**

这套方案不是"先有思路再写代码"，而是**人机协作一路摸索出来的**。下面这些关键突破，
每一个都是 AI 在会话中主动排查出来的，而不是我原本就知道的：

- 最初的目标方案是另一个项目（需要给 iPhone 装 App + IPA 签名 + 本地回环 VPN）。
  是 AI 逐条否掉了 **7 条更麻烦的路线**，才找到现在这条最短的
- **「macOS 上本来就有 `AirTrafficHost.framework`」** 这个决定性发现，来自核对上游代码
  —— 上游之所以只做 Windows 版，仅仅因为它按目录去找 `.dll`；而 macOS 上的框架
  **符号完全一致**（实测 9/9 + 8/8 可用）。换了库加载方式之后，**一台 Windows 电脑都不需要了**
- 那个把所有人引向错误方向的 `product_type` bug，是逐行读源码抓出来的。
  它的恶劣之处在于：异常被 `except: continue` 静默吞掉，最后报出 `no paired iPhone found`
  ——**你会以为是线的问题、信任没点，而真正的原因在代码里**
- 连「静音件必须按设备实际时长生成」这件事，也是在真机上 dump 出原件、比对哈希后才暴露的：
  **差了 400 毫秒**。只看文档、照搬作者现成的静音件，绝对发现不了

**所以我特别鼓励你：直接用 DSH 来做这件事。** 你完全可以复现这条路径：

```sh
# 1. 装好 DSH，然后在你的工作目录里启动它
dsh
# 2. 直接对它说：
> 帮我克隆 https://github.com/jiutinglifeng/ios-calltone-silence
> 然后读 README，按里面的步骤帮我把这台 iPhone 的通话录音提示音去掉
```

把 iPhone 用数据线连上 Mac，剩下的交给它。README 写得足够具体——
包括**出错时该看哪个文件、哪句报错对应什么动作**（见 `docs/失败签名对照表.md`）。

> 如果你遇到新问题，让它把这轮的发现补进文档，再开一个 PR 回来。
> **这个方案的下一次改进，可能就来自你。**
>
> 说句实话：本项目自己的增量其实只有三点——把 `airlift.py` 移植到 macOS、
> 修掉 3 个会误导人的 bug、以及验证「静音件必须按设备实际时长生成」。
> **剩下的全是上游研究者多年的工作**，见文末致谢表。

---

## 支持的设备

**已在下列环境完整验证（读、写、还原、重启后复测全部通过）：**

| 项 | 值 |
|---|---|
| 设备 | iPhone 19,7 |
| 系统 | **iOS 27.0.1 (24A446)** |
| 电脑 | Mac（Apple Silicon） |

验证内容包括：替换成功、读回哈希一致、**实际通话确认提示音消失**、
以及**重启手机后依然有效**。

> 一个已知残留：对方仍能听到极轻的三声引导音（"嘟-嘟-滴"），
> iPhone 上对应提示是「即将播放此通话正在录音的提示」。
> 这说明它**和提示音是两个不同的音频主体**，未被本次替换覆盖。

### ⚠️ 关于 iOS 版本

| iOS 版本 | 状态 |
|---|---|
| iOS 18.0 – 27.0.1 | 上游研究认为可用（我们只验证了 27.0.1） |
| **iOS 27.2 beta 3 及以上** | ❌ **已被 Apple 修补，方案失效** |

> **如果你在 27.0.1 或更低版本：建议先关闭「设置 › 通用 › 软件更新 › 自动更新」。**
> 漏洞一旦被修补就不会再回来。

> **27.2 实测（2026-10-28，iPhone19,7 / 24B119）**：`ATAirlock` 不再跟随符号链接，
> AssetID 里的 `..` 被静默忽略，AFC 也不再提供链接原语。工具在此环境下会
> **报 `ok: true` 但设备上没有任何变化**——`write` / `replace` 的返回值不能当作成功依据。
> 完整实验、错误码与备份恢复通道的排查见
> [docs/iOS-27.2-通道评估.md](docs/iOS-27.2-通道评估.md)。

### 其他平台

核心工具本身是跨平台的——`macos/airlift.py` 里同时保留了 Windows 的
`AirTrafficHost.dll` 加载分支（按 `sys.platform` 自动选择），`设备体检.py`、
`演练-救援流程.py`、`requirements.txt` 也没有任何 macOS 专有依赖。

但**我们只在 Mac 上跑通过**，下面这些是已知情况：

| 平台 | 状态 |
|---|---|
| **macOS** | ✅ 完整验证（本机 Apple Silicon） |
| **Windows** | 🟡 代码齐备，**未在真机上验证**。唯一缺的是 `静音件生成.sh` 的等价实现（它是 bash + macOS 的 `afconvert`） |
| **Linux** | ❌ 大概率不可行 —— 这个 exploit 依赖 Apple 自己的 `AirTrafficHost` 框架，Linux 上没有 |

> **想帮忙？** Windows 的静音件生成是目前唯一明确的缺口。
> 关键悬念：**PCM 格式的 CAF 配正确时长是否也能用**——如果能，Windows 用户就能用
> `ffmpeg` 直接生成，问题当场解决。细节见 **[issue #1](https://github.com/jiutinglifeng/ios-calltone-silence/issues/1)**，也欢迎直接提 PR。

---

## 原理

这套方法建立在上游 [airlift](https://github.com/0xjohnnydev/airlift) 项目的
**AirTraffic / ATAirlock 沙盒逃逸**研究之上。

AirTraffic 是 Apple 用来从 Mac 同步媒体（书籍、有声书）到 iOS 的官方协议。
它存在一处路径校验缺陷：

1. 通过 `com.apple.streaming_zip_conduit` 上传一个特制压缩包，内含**相对符号链接**
   `p0/p1/p2/link → ../../../<目标路径>`
2. 用 AFC 覆盖写 `Media/Books/Sync/Books.plist`，其 AssetID 中带 `..` 路径
3. 走完 AirTraffic 状态机（`HostInfo → RequestingSync → ReadyForSync →
   AssetManifest → FileComplete`）触发搬运
4. **缺陷点**在 `-[ATAirlock processCompletedAsset:]`：
   - AssetID（identifier）**未做路径校验**，可用 `..` 逃出 `/var/mobile/Media/Airlock/Book`
   - destination **只做字符串前缀检查** `hasPrefix:@"/var/mobile/Media/"`，
     不解析符号链接后的真实路径

于是文件可以被写到沙盒之外，也就是那两个提示音所在的目录。

### 本项目的关键发现：macOS 不需要另找工具

上游 `AirliftSilence` 只提供 **Windows** 版本，理由是"漏洞需要 `AirTrafficHost.dll`"。

**但这个理由只对那个 Windows 移植版成立。** macOS 上同样存在
`/System/Library/PrivateFrameworks/AirTrafficHost.framework`，而且
**符号完全相同**。实测结果：

| 库 | 需要的符号 | macOS 可用性 |
|---|---|---|
| `AirTrafficHost.framework` | 9 个（`ATHostConnection*` / `ATCFMessage*`） | ✅ **9 / 9** |
| `CoreFoundation.framework` | 8 个（`CFString*` / `CFData*` / `CFPropertyList*` / `CFRelease`） | ✅ **8 / 8** |

因为 `airlift.py` 里的 ctypes 签名绑定本来就是平台通用的，**只需换掉"从哪里加载这两个库"**，
其余逻辑一行不用改。这就是 `macos/enablement.patch` 做的事（约 70 行）。

---

## 🔴 风险与边界

### 它实际会碰设备上的什么

| 操作 | 位置 | 说明 |
|---|---|---|
| 临时改写同步索引 `Books/Sync/Books.plist`，**用完恢复** | 媒体分区 | 触发逃逸所必需；脚本会先保存再还原 |
| 在 `Media/` 下放临时压缩包与符号链接，**用完删除** | 媒体分区 | 逃逸的中间产物 |
| `read` / `replace` 时把**目标音频文件**移到 `Media/` 再写回 | 媒体分区 | 就是那两条你显式指定的路径 |

**全部在「媒体 / 用户数据」这一层。**

### 最坏情况是什么

诚实地说：**我们实测到的最坏情况是——某次写入失败，导致目标音频文件缺失或损坏。**

表现是：录音时的提示音消失、或者播放异常。**不影响开机、不影响打电话、不影响任何其他功能。**

而且这个状态**有明确的恢复路径**（见下方「还原与退路」），因为：

1. 失败时原件**会留在设备的 `Media/` 目录**，没有消失；`tools/设备体检.py` 就是用来把它捞出来的
2. 你在第 2 步 dump 出来的备份是**独立于设备的另一份拷贝**
3. 最后一个兜底：**重装或恢复系统**一定能让 Apple 重新下发这些文件

### 它做不到什么

- ❌ **够不到引导链 / 系统分区 / SEP** —— 这三样受 Secure Boot 与代码签名保护，
  未越狱设备写不进去；而 AirTraffic 这条通道**本身没有那个权限**
- ❌ 不会影响你的照片、聊天记录、账号、其他 App
- ❌ 不会让设备失去保修资格（没有越狱、没有改系统分区）

### 它不保证什么

- **不保证长期有效**：Apple 可能在任何一次系统更新里还原这些文件。这类文件**曾被静默更新过**
- **不保证跨版本兼容**：我们只在 **iOS 27.0.1 (24A446)** 上实测过
- **不保证第三方依赖不变**：`pymobiledevice3` 迭代极快，我们**已经因此踩过一次坑**，
  所以锁定了实测版本（见 `requirements.txt`）
- **不保证穷尽了所有失败模式** —— 任何声称"绝对不会有别的意外"的项目都在吹牛。
  我们能说的是：**实测过的失败都是可恢复的，且恢复手段不依赖设备本身**

### 供应链上唯一不能替你担保的一环

```
pymobiledevice3
```

这是第三方库，不是我们写的。我们在 `requirements.txt` 里**锁定了实测通过的版本**，
但你要清楚：**这是本方案里唯一一个我们无法完全替你审完的部分**。
建议装之前看一眼它的仓库（它是个知名的开源 iOS 工具库）。

> **我们做过的审计**：`macos/airlift.py` 764 行，逐行读过；无网络调用；
> 改动相对上游只有约 70 行（`macos/enablement.patch` 可对照）。

---

## 使用方法

### 环境准备

```sh
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

> `requirements.txt` **锁定了我们实测通过的 `pymobiledevice3` 版本**。
> 为什么锁定：这个库迭代极快（PyPI 上 700+ 版本），内部 API 会随版本变动——
> **我们在开发过程中就因此踩过一次坑**，工具报出一句完全误导人的错误。
> 建议就用锁定版本；如果想升级，先跑只读的 `设备体检.py scan` 验证兼容性。

> **如果遇到 `PermissionError: ~/.pymobiledevice3`**：那只是配对记录缓存目录不可写，
> 不是设备问题。加一个环境变量即可，下文命令里已经带了。

### 第 1 步：连接与体检（零风险）

用数据线连上 iPhone、解锁。然后：

```sh
cd macos
AIRLIFT_PAIRING_CACHE=/tmp/pd3-pairing ../.venv/bin/python ../tools/设备体检.py scan
```

这一步**只读**设备的媒体目录，不触发逃逸，跑多少次都安全。
它会检查之前有没有操作失败留下的残留文件。

### 第 2 步：dump 原件（必须先做）

```sh
AIRLIFT_PAIRING_CACHE=/tmp/pd3-pairing .venv/bin/python airlift.py read \
  /var/mobile/Library/CallServices/Greetings/default/StopDisclosure.caf \
  backup-StopDisclosure.caf --logging

AIRLIFT_PAIRING_CACHE=/tmp/pd3-pairing .venv/bin/python airlift.py read \
  /var/mobile/Library/CallServices/Greetings/default/StartDisclosureWithTone.m4a \
  backup-StartDisclosureWithTone.m4a --logging
```

`read` 是**往返操作**：把文件移到临时区、读出、再写回原位。成功时文件回到原样。

> **为什么必须先 dump**：见下方「🔴 第一个坑」。**这一步不能跳过。**

### 第 3 步：按你的设备实际时长生成静音件（关键）

```sh
cd ..
./tools/静音件生成.sh macos/backup-StopDisclosure.caf macos/silent-Stop.caf
./tools/静音件生成.sh macos/backup-StartDisclosureWithTone.m4a macos/silent-Start.m4a
```

脚本会读原件的**编码 / 采样率 / 声道 / 时长**，生成完全匹配的静音文件并自动校验。

> ⚠️ **不要直接使用任何现成的静音件**，包括本仓库 `macos/` 里那份——
> **它的时长是按另一台设备生成的**。原因见下方第一个坑。
>
> 补充：`ffmpeg` **无法把 Opus 封装进 CAF**，而 `StopDisclosure.caf` 正是 Opus/CAF。
> 所以生成脚本对 CAF 走的是 macOS 自带的 `afconvert`，对 m4a 走 `ffmpeg`。

### 第 4 步：替换

```sh
cd macos
AIRLIFT_PAIRING_CACHE=/tmp/pd3-pairing .venv/bin/python airlift.py replace \
  /var/mobile/Library/CallServices/Greetings/default/StopDisclosure.caf \
  silent-Stop.caf --backup device-backup-Stop.caf --logging

AIRLIFT_PAIRING_CACHE=/tmp/pd3-pairing .venv/bin/python airlift.py replace \
  /var/mobile/Library/CallServices/Greetings/default/StartDisclosureWithTone.m4a \
  silent-Start.m4a --backup device-backup-Start.m4a --logging
```

**每次都要检查输出里的 `previousSize` 不是 `null`：**

```json
{"ok": true, "size": 4327, "previousSize": 41944, "backup": "device-backup-Stop.caf"}
```

- `previousSize` 是数字 → ✅ 原件已安全读回备份，继续
- `previousSize: null` → 🔴 **原件没读回来**，它留在设备的 `Media/airlift-replaced-*`，
  **立刻停下**，用 `tools/设备体检.py scan` 找出来抢救

### 第 5 步：验证

```sh
AIRLIFT_PAIRING_CACHE=/tmp/pd3-pairing .venv/bin/python airlift.py read \
  /var/mobile/Library/CallServices/Greetings/default/StopDisclosure.caf verify.caf
shasum -a 256 verify.caf silent-Stop.caf   # 两个哈希应一致
```

然后**打一通电话实际录音**，确认提示音消失；再**重启手机测一次**。

---

## 🔴 常见坑（两个）

### 一、静音件的时长必须按你的设备生成

提示音是**会说话的通知音**，内容随系统语言变化，所以**不同设备的文件时长不同**。

我们在一台 24A446 设备上实测到的差异：

| 文件 | 本设备 | 上游作者 dump | 上游配套静音件 |
|---|---|---|---|
| `StopDisclosure.caf` | Opus 48k **1.520000 s** | Opus 48k 1.920000 s | 1.940000 s |
| `StartDisclosureWithTone.m4a` | AAC 44.1k 1.618639 s | AAC 44.1k 1.618639 s | 1.671837 s |

**Stop 提示音相差 400 毫秒。** 如果直接使用上游配套的静音件，会比原件长 420 毫秒——
播放时机被明显拉长。

> 一个旁证：AirBeep 的作者把「**更改系统语言**」当作强制恢复原始提示音的手段。
> 如果提示音与语言无关，这个手段不会有效。

**所以：先 dump，再按实际时长生成。** 这是必需步骤，不是可选优化。

### 二、上游代码有个 bug 会让你误判「手机没连上」

`airlift.py` 里有一行：

```python
pt = ld.product_type or ""
```

但 `pymobiledevice3` 11.x 的 **lockdown 客户端没有 `product_type` 属性**
（它只属于 irecv / DFU 子系统）。这行抛 `AttributeError`，紧接着的
`except Exception: continue` 把它**静默吞掉**——每台设备都被跳过，
最后报出一句与实际原因无关的 **`no paired iPhone found`**。

**你会以为是线的问题、信任没点、驱动没装，其实代码里出错了。**

本仓库的 `macos/airlift.py` 已修复（并让错误信息带上真实原因）。
同类问题我们一共修了 3 处，见 `docs/HANDOFF.md`。

---

## 还原与退路

用你自己 dump 的原件写回：

```sh
cd macos
AIRLIFT_PAIRING_CACHE=/tmp/pd3-pairing .venv/bin/python airlift.py write \
  /var/mobile/Library/CallServices/Greetings/default/StopDisclosure.caf \
  <你的原件备份路径> --logging
```

> `write` 适用于目标路径为空的情况；若目标已存在，用 `replace`。

**注意**：本仓库 `device-backup-24A446/` 里是**另一台设备**的原件，
**不要用来还原你自己的设备**（时长不同）。请用你自己第 2 步 dump 出来的备份。

**另一个兜底**：作者提到更改系统语言可强制系统重新生成默认提示音。
此说法未经我们独立验证。

---

## ❓ 常见问题

**Q：我完全不懂技术，能用吗？**
不建议单独操作。请先找懂的人一起看一遍代码和步骤，或者直接用 DSH 让它带你走（见上方致谢一节）。

**Q：会不会把我的手机弄坏？**
见「风险与边界」。实测到的最坏情况是提示音文件缺失/损坏，**不影响开机和通话功能**，
且有明确的恢复路径。但**没有任何人（包括我们）能给你 100% 的保证**——
所以请先做第 2 步的备份。

**Q：它会读取我的隐私数据吗？**
不会。源码里**没有任何网络调用**，全部走 USB 在你手机和电脑之间。
它只按命令里显式写死的路径操作那两个音频文件（外加 AirTraffic 逃逸所需的临时文件）。

**Q：为什么不用那些现成的 App？**
因为那些 App 需要：给 iPhone 装 App、IPA 签名（免费账号 7 天过期）、
LocalDevVPN 本地回环隧道、美区 Apple ID。我们找到一个更直接的入口之后，
**这些全都不需要了**。

**Q：升级系统后会失效吗？**
**iOS 27.2 beta 3 起已被 Apple 修补**，升级后跑也没用。
如果你在 27.0.1 或更低版本，**建议现在就关掉自动更新**。

**Q：更新系统或重装系统后，提示音会回来吗？**
可能会。这类音频文件**曾被 Apple 静默更新过**。如果提示音自己回来了，
按同样步骤再跑一次即可。

**Q：我的是别的 iPhone 型号，可以用吗？**
原理上可以，但**我们只在一台 iPhone 19,7 / iOS 27.0.1 (24A446) 上实测过**。
关键点：**静音件必须按你设备原件的实际时长生成**（见第 3 步），
不能照搬本仓库里那份——它是另一台设备的。

**Q：为什么工具文件名是中文的？**
因为项目起始于中文用户。如果你希望英文文件名，欢迎提 PR。

---

## 目录结构

```
requirements.txt           锁定的依赖版本（实测通过）
README.md                  本文（中文）
README.en.md               English version

macos/                     核心方案
  airlift.py                 AirTraffic 逃逸实现（含 3 个修复 + macOS 支持）
  airlift_gui.py             图形界面（可选）
  enablement.patch           相对上游的全部改动
  silent-Stop.caf            本设备实际使用的静音件（示例，请按你的设备重新生成）
  silent-Start.m4a           同上

tools/                     辅助工具
  静音件生成.sh              按设备实际格式生成静音件（关键）
  设备体检.py                只读扫描 / 抢救残留原件
  演练-救援流程.py           端到端演练救援链路（20 项断言，不需要真机）

docs/
  HANDOFF.md                完整技术记录：执行过程、踩过的坑、全部实测数据
  失败签名对照表.md          12 个真实错误串 → 对应处理动作
  iOS-27.2-通道评估.md       27.2 实测：为什么失效、各类绕法的结果、备份恢复卡在哪

device-backup-24A446/      本项目的真机原件（供格式参照，不要用于还原其他设备）
```

---

## ⚖️ 法律

- 这段提示音的目的是**让对方知道通话正在被录音**。移除它意味着对方不再被系统告知。
- 不同司法辖区对通话录音的同意要求不同。**部分国家和地区要求通话双方均同意**，
  在这些地方使用本方法可能违法。
- **请自行确认你所在地区的法律，并在必要时取得对方同意。** 这是使用者自己的责任，
  与本项目作者无关。

---

## 致谢

本项目的技术基础全部来自以下上游工作：

| 项目 | 作者 | 贡献 |
|---|---|---|
| [airlift](https://github.com/0xjohnnydev/airlift) | Johnny Franks | 最初的 AirTraffic / ATAirlock 沙盒逃逸研究（macOS PoC） |
| [airlift-windows](https://github.com/DahanGuy/airlift-windows) | Guy Dahan | 用 ctypes 调用 `AirTrafficHost.dll` 的 Windows 移植 |
| [AirliftSilence](https://github.com/YiHoooong/AirliftSilence) | YiHoooong | 面向「静音通话提示音」的封装、Grappa 会话修复、备份与校验流程 |
| [AirCard-iOS](https://github.com/Mak5er/AirCard-iOS) | Mak5er | Grappa 客户端令牌与修复思路 |
| [bl_sbx](https://github.com/h2zi/bl_sbx) | h2zi | 替换这两个文件的先行者 |

**本项目的增量**只有三项：把 `airlift.py` 移植到 macOS（换库加载）、
修掉 3 个会导致误判的 bug、以及验证「静音件必须按设备实际时长生成」这件事。

## 许可

上游项目均为 **MIT License**（Copyright © 2026 Johnny Franks），详见 `LICENSE`。
本项目在其基础上修改，同样以 MIT 发布。
