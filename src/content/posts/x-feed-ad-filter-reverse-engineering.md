---
title: "X Android 时间线广告逆向：从 promoted_metadata 到 DEX 补丁验证"
pubDatetime: 2026-10-17T21:40:00+08:00
modDatetime: 2026-10-17T23:16:00+08:00
description: "以 X 12.34.0 的 APKM 为样本，逐层追踪 GraphQL 条目、推广模型、模块过滤与 SSP 模式，复盘 Frida 原型、DEX 重写和静态验证的证据边界。"
tags: [Android, 逆向工程, DEX, 广告分析]
---

> 本文是对一个**指定版本、指定样本**的离线分析记录，不把文件校验说成真实设备去广告成功。文末现提供原始样本、非官方重签测试包及中间产物的公开链接：它们包含第三方应用内容，下载或使用前应自行确认授权与风险；测试包**不是官方发布，不建议安装在日常设备上**。应用版本变化后，混淆符号与调用关系都要重新核实。

最初的问题看似简单：时间线里出现推广帖，能不能找到对应的广告 View，把它隐藏？实际走完调用链后会发现，推广帖可能已经是一个和普通帖子共享渲染流程的模型；此外，还有独立的 RTB 图片广告和 SSP 投放路径。**如果只在 UI 层隐藏，数据层仍然保留了那个条目，位置、分页、模块子项乃至埋点都可能继续按它计算。**这次研究因此先追数据从哪里来，再决定在哪个边界过滤。

## 1. 固定样本和观察范围

输入是 [原始 x.apkm](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/x.apkm)，SHA-256 为 `b7bd98c97fcf6fdfcafc39e10f3c37e66d9f98c78de4470dea9c69688c955a10`。用 `aapt2` 检查主 APK 后，确认包名 `com.twitter.android`、版本 `12.34.0-prod.01`、versionCode `312340001`，最低系统版本 API 32。APKM 并非一个普通的单 APK：合并日志列出 **35 个分包**，包括语言、密度、arm64/armeabi-v7a 和 c2pa 分包；后者还带来了一个 DEX。合并步骤与结果见[脱敏构建日志片段](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/log-excerpts.txt)，未经删改的日志未公开。

这决定了两件事。第一，不能只看主包里某个 `classes.dex` 就推断整个安装包的类分布。第二，后续单 APK 测试构建涉及**分包合并与重签名**，即便补丁本身正确，资源或签名也可能引入独立的运行问题。

分析用的是主包 DEX 的字符串候选检索和本机 JADX 定点反编译。对完整主包的 REA/JADX 请求曾遇到超时，[package-evidence.json](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/package-evidence.json) 记录的是失败或部分结果，**不能充当完整反编译已成功的证据**。特别是关键转换函数 `J` 的结构化反编译失败，保留在[简化版 g.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/simple/coil3/network/g.java) 的是简化控制流与警告；下面会同时参考它的调用方，避免仅凭一段不可靠的伪 Java 下结论。

## 2. 从 GraphQL 字段找到模型，而不是从控件找广告

[GraphQL 字段解析 fw0.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/android/main/fragment/fw0.java) 的字段列表里同时出现 `tweet_results`、`social_context`、`promoted_metadata`、`preroll_metadata`。其中推广元数据被解析后，经上面的 [g.java 转换分支](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/simple/coil3/network/g.java)变成 `com.x.models.df`。这一步很关键：**字段名来自输入数据，`df` 则是进入时间线模型后的类型**；不能因为某个普通帖含有“推广”文字，就等同于它携带这种元数据。

在最终条目中可以找到三个不同形态：

| 模型                                                                                                                             | 可见证据                                                                               | 分析含义                                                    |
| -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| [帖子模型 l1.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/models/timelines/items/l1.java) | `toString()` 明确写出 `promotedMetadata` 与 `prerollMetadata`；方法 `y()` 返回前者 `e` | 帖子可以带推广元数据，两个字段不能混用                      |
| [用户模型 d2.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/models/timelines/items/d2.java) | 字段 `h` 的类型为 `com.x.models.df`                                                    | 用户条目也可能携带推广元数据                                |
| [RTB 模型 r1.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/models/timelines/items/r1.java) | `toString()` 为 `UrtTimelineRtbImageAd`，字段 `j` 是推广元数据                         | 独立 RTB 图片广告需要按模型类型识别，不能依赖元数据一定非空 |

`r1` 的构造函数在当前反编译结果中还对其 `df` 参数做了非空检查；过滤实现仍额外按 `r1` 类型处理，是为了把这条**独立广告模型**与普通帖子模型明确区分，而不是把所有类型都靠同一个字段猜测。反过来，只有 `preroll_metadata`、视频卡片或商业内容披露的普通帖不应被顺手删除：本次目标不是视频前贴片，也不是泛化的内容审查。

这里还有一个容易误读的包名：`coil3.network.g` 与 `com.facebook.soloader.nativeloader.a` 是该版本 DEX 中的**实际混淆/优化后位置**，不是说 Coil 或 SoLoader 的原始库源码负责投放广告。以“包名看起来属于第三方库”为由跳过它们，会漏掉真实调用边。

## 3. 追到“可以安全返回 null 吗”

时间线不是只有一种新增指令。[时间线仓库 h1.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/repositories/urt/h1.java) 在同一处理段中对新增条目、替换、置顶与 `AddToModule` 分别处理。新增条目经 `coil3.network.g.K(su0, eh)` 转换后，调用方**只把非 null 结果加入条目列表**；替换与置顶也检查 `K` 的结果。模块增量路径经 `com.facebook.soloader.nativeloader.a.Q(qo0, eh, e4)` 包装子项；在[简化版 a.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/simple/com/facebook/soloader/nativeloader/a.java) 中，`Q` 调用 `g.J(...)`，若返回 null 就不创建包装项。

模块本身也经过 `K`：它先用 `J(...) != null` 筛掉不应保留的子条目，然后基于筛选后的列表继续生成模块子项；没有有效内容时还有返回 null 的路径。这比“隐藏 RecyclerView 的某个 View”更接近数据形成的地方：调用者本来就设计了**映射失败或跳过条目**的分支。

不过，静态上看到 `null` 容忍分支，**不等于**已经实测证明所有分页和去重逻辑安全。尤其是模块过滤前后可能重复调用映射器，所以一次真实屏幕展示不一定只对应一次 `J` 调用；Hook 的 `seen/dropped` 计数也不能被当作屏幕上广告的精确数量。

由此得到的最小过滤语义是：

```text
先运行原始 J，得到原本的时间线模型
  ├─ 原始结果为 null：仍返回 null
  ├─ 模型是独立 RTB 广告：返回 null
  ├─ 帖子/用户的推广元数据非 null：返回 null
  └─ 其他结果（普通帖、游标及无法确认的模型）：原样返回
```

选择**先运行原函数、再检查结果**，是为了尽量保留原有解析和错误语义；并不是直接让输入 JSON 字段消失。对当前已见过的路径这是合理切点，对未来新模型和缓存中已存在的条目则没有覆盖保证。

## 4. 为什么还要单独看 SSP

推广时间线条目与客户端 SSP 模式不是同一件事。在[SSP 模式选择 a1.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/googlessp/a1.java) 中，`c(timeline)` 会先读 `android_x_lite_ssp_ads_enabled`，再依据配置和时间线选择 `ClientDriven`、`ServerSlotClientFetch`、`ServerToServer` 或 `Disabled`；`d()` 与 `e()` 又分别涉及 DSP 上下文和调试 mock。另一个实现 [e.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/googlessp/e.java) 也有自己的 `b/c/d` 判断。两者都在 `classes4.dex`，不能只改其中一个然后声称 SSP 已关闭。

因此原型与补丁把这六个**广告专属**方法作为独立门控：返回模式的方法给出 `Disabled`，布尔开关给出 `false`。此处没有伪装 Premium、订阅或登录状态。这样做仍不代表阻止服务端继续下发广告字节，更不表示已覆盖视频前贴片、其他 SDK、动态模块或未知实验组。条目过滤和 SSP 关闭是两条互补的本地路径。

## 5. 先做非持久原型，再做可复核的 DEX 改动

第一阶段是 [x-feed-noads.frida.js](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/x-feed-noads.frida.js)：它先检查包名、版本名、versionCode，不匹配就不装 Hook；获取 `J` 的精确重载，先调用原实现，再按模型类型和 `df` 字段分类。若**分类器自身**抛异常，则保留原条目并计数；原始 `J` 抛出的异常照常传播，避免掩盖原应用故障。日志只记录有限的丢弃类型，不输出帖子正文或用户标识。

[test-hook.cjs](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/test-hook.cjs) 用模拟 Java 桥测试了普通帖、推广帖、推广用户、RTB、游标、非推广类型字段、分类失败放行、原函数异常和版本保护。它证明的是**脚本在 mock 契约下的分支行为**，不证明 Frida 在真实 Android 上能找到相同的类加载器，更不证明实际时间线无空白或无崩溃；这个原型没有真机运行记录。

第二阶段才是 DEX 文件级测试构建，核心记录在 [PatchFeed.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/PatchFeed.java) 与 [VerifyPatch.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/VerifyPatch.java)：

1. 使用 APKEditor 把 35 个分包合成测试用单 APK；主包原有 `classes.dex` 至 `classes13.dex`，c2pa 的 DEX 在合并时成为 `classes14.dex`。
2. `PatchFeed.java` 用 dexlib2 找到 `J` 的**精确参数与返回类型**。保留原实现为 `reversexOriginalJ`，新 `J` 调用原实现，再调用 `reversexFilter` 对结果分类；不是重新编译整份 JADX Java。
3. 主 DEX 方法引用已满，故将**整个目标类**移到新增 `classes15.dex`；主 DEX 其余类仍保留。代码检查只移动一类、两处 SSP 类各命中三个方法，预期改动计数为 `7`（一个 `J` 包装加六个 SSP 方法）。过滤辅助方法是额外新增的，不能把日志中的 `patches=7` 理解成 APK 总共只有七个新方法。
4. 输出后立即重新解析 DEX；再对齐并用独立测试证书签名。该测试签名不是官方签名，通常不能直接覆盖已有官方安装。

[脱敏的 patch.log 摘要](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/log-excerpts.txt) 记录识别到的三个 `df` 类型字段、六个 SSP 方法、输出的 `classes-patched.dex`（9,397 类）、`classes4-patched.dex`（9,804 类）以及单类的 `feed-filter.dex`。这些数字是构建过程检查点，不是运行时命中次数。

## 6. “验证通过”具体通过了什么

`VerifyPatch.java` 做了一个有用的反向对照：从改后的目标类里去掉新 `J` 和过滤辅助方法，把 `reversexOriginalJ` 恢复成原名字，再将类放回原类集合；对重建后的规范化 DEX 与原始 DEX 比对。对 `classes4.dex` 则先逐个确认六个方法确实返回 `Disabled` 或 `false`，再还原它们、比较规范化结果。[patch-verification.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/patch-verification.txt) 记录以上检查通过，说明在**这个规范化比较范围内**其他类、方法与元数据保持一致。由于 DEX 经过重写与类迁移，**不能**据此说原始整个 DEX 文件逐字节不变。

最终 APK 还逐项检查三个目标 DEX 与生成文件完全相同，并对其余 **5,135 个合并条目**读取 ZIP 成员的**解压后内容**进行字节比较。这个“逐字节”不是说压缩流、ZIP 排列或整体 APK 二进制原样保留。[signature-verification.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/signature-verification.txt) 显示 APK Signature Scheme v3 验证通过；它只证明测试包签名结构有效，并不证明 X 的登录、完整性校验或服务端会接受它。

| 命题                                                | 本地证据能够支持的状态                                |
| --------------------------------------------------- | ----------------------------------------------------- |
| 找到了当前版本推广字段、模型和若干 `null` 消费路径  | 定点反编译与调用方代码支持；关键 `J` 本身有反编译警告 |
| 原型分类和版本保护逻辑成立                          | Node mock 测试通过，**未做真实 Frida 实测**           |
| 补丁 DEX 可解析、指定范围内差异可还原               | dexlib2 重读与规范化 DEX 对比通过                     |
| 合并包的非目标 ZIP 成员内容与基准一致、测试签名有效 | 解压后内容比对和 v3 验签通过                          |
| 安装、启动、登录、多页列表与广告效果                | **没有充分的真机验收证据，不能宣称通过**              |

## 7. 如果继续验证，该观察什么

至少要分别看推荐和关注时间线的刷新、多页加载、模块新增/替换、游标是否保留、缓存旧条目何时消失，以及 SSP 是否留下占位；同时记录崩溃和 ANR。模拟器或测试设备上还要独立检查**分包合并与重签名**是否造成启动问题，否则无法把故障归因于过滤逻辑。不要为验证而随意清除日常账号数据。本文止于现有文件能支持的结论：找到了有依据的切点，完成了静态可复核的测试构建，但没有证明“X 广告已经全部消失”。

## 8. 公开文件与中间产物说明

全部公开对象的**逐文件大小、SHA-256 和链接**收录在 [R2 文件清单 INDEX.md](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/INDEX.md)。以下按分析阶段说明用途。压缩包保留 `analysis/` 下的原始相对路径；可用 `zstd -dc 文件.tar.zst | tar -xf -` 解包。把目录打包是为了保留 8,569 个反编译/工具文件，而不是假装每个自动生成的 Java 文件都经过人工审读；本文**实际引用的关键 Java 文件另有单文件链接**。

### 样本与成品（第三方软件，非官方分发）

- [x.apkm](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/x.apkm)：35 分包的原始分析输入，SHA-256 `b7bd98c97fcf6fdfcafc39e10f3c37e66d9f98c78de4470dea9c69688c955a10`。
- [analysis/base.apk](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/base.apk)：从输入提取的主 APK，供 Manifest、主 DEX 和版本核对。
- [dist/X-12.34.0-noads-universal.apk](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/dist/X-12.34.0-noads-universal.apk)：合并分包、修改并重签名的**未完成实机验收的本地测试包**，不是官方 APK；SHA-256 `70c4ce5d994469047be80bead31053f263d2725bec566602f0605d3ca0ed9a5f`。通常不能覆盖官方签名版本，不建议在日常设备安装。
- [dist/README.md](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/dist/README.md)：测试包修改范围、静态校验与安装风险说明；[dist/SHA256SUMS](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/dist/SHA256SUMS)：成品校验值；[APK .idsig](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/dist/X-12.34.0-noads-universal.apk.idsig)：生成的签名附属文件，不是私钥。

### 研究笔记、检索输入与索引

- [广告插入与去除方案.md](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/%E5%B9%BF%E5%91%8A%E6%8F%92%E5%85%A5%E4%B8%8E%E5%8E%BB%E9%99%A4%E6%96%B9%E6%A1%88.md)：早期调用链、过滤点和待验证项；与本文一起阅读时以文件所述时间阶段为准。
- [package-evidence.json](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/package-evidence.json)：REA 请求超时/部分结果，**不是成功分析报告**；[targets.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/targets.txt)、[converter-targets.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/converter-targets.txt)、[converter-targets-v2.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/converter-targets-v2.txt)：JADX 定点定位的类列表。
- [dex_string_refs.py](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/dex_string_refs.py)：DEX 字符串候选检索脚本；[ad-string-refs.jsonl](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/ad-string-refs.jsonl)、[ad-string-refs-short.jsonl](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/ad-string-refs-short.jsonl)、[ad-config-refs.jsonl](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/ad-config-refs.jsonl)：候选引用数据。字符串命中**不能独立证明行为**。
- [class-candidates.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/class-candidates.txt)、[class-dex-map.json](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/class-dex-map.json)：候选类与其所在 DEX 的定位索引；[FocusedDecompile.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/FocusedDecompile.java)：定点反编译辅助代码。

### DEX、反编译代码与工具

- [classes.dex](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/classes.dex)、[classes4.dex](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/classes4.dex)、[classes10.dex](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/classes10.dex)、[classes11.dex](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/classes11.dex)：从主 APK 提取用于候选类归属和修改的原始 DEX；不是包内所有 DEX 的独立清单。
- [classes-patched.dex](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/classes-patched.dex)、[classes4-patched.dex](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/classes4-patched.dex)、[feed-filter.dex](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/feed-filter.dex)：重写后的主 DEX、SSP DEX 和迁移到新增 DEX 的目标类。
- [decompiled.tar.zst](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/decompiled.tar.zst)：早期广覆盖 JADX 产物；[focused.tar.zst](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused.tar.zst)：定点反编译树；[simple.tar.zst](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/simple.tar.zst)：关键复杂方法的简化控制流；[tools.tar.zst](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/tools.tar.zst)：本地保留的 APKEditor/dexlib2 工具 JAR。归档是原样中间产物，自动生成代码包含反编译误差。
- 单文件交叉核查入口：[GraphQL 字段 fw0.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/android/main/fragment/fw0.java)、[推广结构 dw0.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/android/main/fragment/dw0.java)、[时间线仓库 h1.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/repositories/urt/h1.java)、[帖子 l1.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/models/timelines/items/l1.java)、[用户 d2.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/models/timelines/items/d2.java)、[RTB r1.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/models/timelines/items/r1.java)。
- 其他重点入口：[SSP a1.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/googlessp/a1.java)、[SSP e.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/focused/com/x/googlessp/e.java)、[简化转换 g.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/simple/coil3/network/g.java)、[模块包装 a.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/simple/com/facebook/soloader/nativeloader/a.java)。

### 原型、构建过程和静态验证

- [x-feed-noads.frida.js](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/x-feed-noads.frida.js)、[test-hook.cjs](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/test-hook.cjs)：非持久原型与 Node mock 契约测试；**不是**真实设备测试日志。
- [PatchFeed.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/PatchFeed.java)、[VerifyPatch.java](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/VerifyPatch.java)：DEX 改写与反向规范化差异验证；[final-badging.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/final-badging.txt)：最终测试包的包名/版本等静态信息。
- [x-merged-unsigned.apk](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/x-merged-unsigned.apk)：合并分包后的未签名基准；[x-noads-unsigned.apk](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/x-noads-unsigned.apk)：替换 DEX 后未签名包；[x-noads-aligned.apk](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/x-noads-aligned.apk)：对齐后的签名前中间包。**三者不是可直接当正式安装包使用的发布物**。
- [patch-verification.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/patch-verification.txt)、[signature-verification.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/build/signature-verification.txt)：非设备级静态验证结果；[log-excerpts.txt](https://assets.diaodaquan.dpdns.org/reversex/x-12.34.0/analysis/log-excerpts.txt)：对合并、补丁与测试输出整理的**脱敏片段和验证摘要**，不是原始日志。

**明确未公开**：`analysis/signing/` 的测试私钥与口令、`analysis/device/` 的原始设备日志和截图、原始 `*.log`。这些文件不属于上述公开清单；尤其不能把第三方应用的日志原样当作去广告实机证据。原始与测试 APK 虽已按要求公开，仍可能受第三方软件许可、平台规则与签名兼容性限制。
