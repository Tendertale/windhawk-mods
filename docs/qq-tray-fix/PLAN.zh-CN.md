# QQ 托盘图标消失：调查与 mod 开发计划

日期：2026-11-14。状态：已建立 fork、固定源码基线并完成初步定位；尚未执行登录/最小化对照实验，尚无修复代码或已验证的根因。

## 1. 仓库与开发入口

| 项目 | 地址 / 值 |
| --- | --- |
| 官方发布源码 | https://github.com/ramensoftware/windhawk-mods |
| 本次 fork | https://github.com/Tendertale/windhawk-mods |
| 计划分支 | `plan/qq-tray-visibility` |
| 官方源码基线 | `57b6069a465ecca327e085d3d0b2057fad533a43` |
| 基线 mod 版本 | Windows 11 Taskbar Styler 1.11.1 |
| 本地目录 | `C:\Users\Ardeon\Documents\Codex\2026-11-14\wo-x20\outputs\windhawk-mods` |
| 核心文件 | `mods/windows-11-taskbar-styler.wh.cpp` |
| 作者开发仓库 | https://github.com/m417z/my-windhawk-mods |
| 主题源指南 | https://github.com/ramensoftware/windows-11-taskbar-styling-guide/tree/main/Themes/WindowGlass |
| 官方开发指南 | https://github.com/ramensoftware/windhawk/wiki/Creating-a-new-mod |

本地使用浅克隆和稀疏检出，当前包含目标源码、根 README 和提交校验 workflow。Git 历史与其他 mod 未全部下载；后续确有需要时再扩展。`origin` 指向个人 fork，`upstream` 指向官方发布仓库。

进入现有仓库：

```powershell
Set-Location 'C:\Users\Ardeon\Documents\Codex\2026-11-14\wo-x20\outputs\windhawk-mods'
git switch plan/qq-tray-visibility
```

开始实现时，从固定基线另建实现分支，以便将可发布的 mod 变更与计划文档分开：

```powershell
git switch -c feat/qq-tray-fix 57b6069a465ecca327e085d3d0b2057fad533a43
```

## 2. 目标与实际环境

用户报告：QQ 登录后最小化，图标同时从任务栏和“显示隐藏的图标”区域消失。已确认有问题的主题为用户所称 Windows Glass（Split）；源码中的准确名称为 `WindowGlass (Split)`，配置 ID 为 `WindowGlass_variant_Split`。

本轮通过只读查询获得的本机状态：

| 项目 | 已观察值 |
| --- | --- |
| Windows | 24H2，build `26100.9711` |
| Windhawk | `1.7.3` |
| 已安装 Styler | `1.11.1`，启用 |
| 当前主题 | `WindowGlass_variant_Split` |
| 用户自定义样式与常量 | 当前配置为空项 |
| QQ 可执行文件版本 | `9.9.35.52892`；ProductVersion `9.9.35.52892-aff854e8` |
| 相关并存 mod | `taskbar-tray-system-icon-tweaks` `1.3.1`，启用 |

托盘调整 mod 当前隐藏音量、电池和麦克风系统图标；读到的设置中没有 QQ 隐藏项。这只能帮助缩小调查范围，不能排除两个 mod 的交互。

任务栏窗口按钮与通知区图标是独立对象。验收要求是：依照 QQ 与 Windows 当前的最小化/托盘偏好，QQ 的真实托盘图标在主托盘或隐藏图标面板中可见且可操作；若 QQ 选择保留任务栏按钮，该按钮也应正常。不能把 QQ 自身选择最小化到托盘、不保留任务栏按钮的行为误判为 Styler 故障。

截图用于确认外观与用户描述，尚未建立每张截图与登录阶段、主题状态的对应关系。

## 3. 源码定位与已知事实

下面的行号均相对于固定基线，改动后可能移动。

| 位置 | 事实与调查意义 |
| --- | --- |
| `mods/windows-11-taskbar-styler.wh.cpp:2`、`:15` | mod ID 为 `windows-11-taskbar-styler`，作者 m417z，GPLv3；仅注入 `explorer.exe`。派生版本需保留许可和原作者归属。 |
| 同文件 `:644` | 内置主题由主题指南中的 YAML 自动生成。若证实是主题规则错误，需要保存可回送指南的对应 YAML 修改，避免只改生成结果。 |
| 同文件 `:3628` | `g_themeWindowGlass_variant_Split` 为 Split 完整规则入口。 |
| 同文件 `:3634`、`:3656`、`:3796` | 任务栏宽度、托盘边距、根网格列定义参与分段布局；需观察新增图标后的实际测量宽度与裁剪。 |
| 同文件 `:3667` | `SystemTray.NotifyIconView#NotifyItemIcon` 覆盖 `Padding`、`CornerRadius`、`Margin`；没有 QQ 专属选择器。需确认实际命中哪些主托盘/overflow 元素。 |
| 同文件 `:3781` | overflow 背景规则覆盖背景、边框、圆角，未直接隐藏 QQ 图标。 |
| 同文件 `:11479` | `VisualTreeWatcher::OnVisualTreeChange`：新增 XAML 元素进入 `ApplyCustomizations`，移除时进入 `CleanupCustomizations`。 |
| 同文件 `:19680` | `HandleVirtualizingRepeater` 处理 `ElementClearing` / `ElementPrepared`；回收复用是动态新增图标的候选调查路径。 |
| 同文件 `:19799`、`:20075` | `ApplyCustomizations` / `CleanupCustomizations` 为样式应用与恢复入口。 |

本轮搜索没有发现该源码直接调用 `Shell_NotifyIcon` 或使用 `NOTIFYICONDATA`；其职责是 Explorer XAML 样式处理。现有证据不能证明 QQ 登录后消失一定由布局、回收复用或 Shell 注册中的某一项导致。

源码永久链接：[Split 主题](https://github.com/ramensoftware/windhawk-mods/blob/57b6069a465ecca327e085d3d0b2057fad533a43/mods/windows-11-taskbar-styler.wh.cpp#L3628)、[动态元素入口](https://github.com/ramensoftware/windhawk-mods/blob/57b6069a465ecca327e085d3d0b2057fad533a43/mods/windows-11-taskbar-styler.wh.cpp#L11479)、[元素回收处理](https://github.com/ramensoftware/windhawk-mods/blob/57b6069a465ecca327e085d3d0b2057fad533a43/mods/windows-11-taskbar-styler.wh.cpp#L19680)。

## 4. 原因假设与证伪条件

以下按当前调查优先级排序，均未被证实。

1. **Split 布局或通知图标样式影响动态新增项。** 预测：同一 QQ 登录流程在无样式/普通 WindowGlass 下正常，Split 下异常；QQ 对应图标元素存在，但尺寸、父级裁剪、可见性或命中区域异常。分组撤掉 Split 的布局或 NotifyIcon 样式后应恢复。
2. **与托盘系统图标调整 mod 交互。** 预测：Split 单独启用正常，两者共同启用才异常。单独关闭托盘调整 mod 应改变结果；若毫无影响则降低此假设优先级。
3. **新增/回收元素样式状态未正确刷新或恢复。** 预测：异常依赖 QQ 在 Styler 加载之后登录，或依赖托盘项移入/移出 overflow；日志显示元素创建后被回收/复用，保留了不适用的属性覆盖。仅布局静态规则不足以解释。
4. **QQ 或 Explorer 的通知图标生命周期问题。** 预测：关闭相关 mod 后仍复现，或 QQ 本应创建图标时没有对应的 Shell/Explorer 证据。没有 XAML Add 事件本身不能证明 QQ 未注册图标，还需核对事件监听是否覆盖正确线程、已有元素是否已创建及 Shell 状态。

## 5. 执行步骤与改动边界

### A. 建立最小复现记录

先在本机私有工作目录保存原始设置和必要版本信息。QQ 账号、聊天内容、桌面截图、完整注册表导出和运行日志均不提交到公开 fork。

使用同一 QQ 版本、账号状态、托盘偏好和显示设置，每次只改变一个变量：

| 对照 | Styler | 托盘系统图标调整 mod | 目的 |
| --- | --- | --- | --- |
| A0 | 关闭 | 关闭 | 验证原生环境是否正常 |
| A1 | 无主题 / 无自定义样式 | 关闭 | 区分 Styler 引擎与主题 |
| A2 | 普通 `WindowGlass` | 关闭 | 区分普通主题与 Split |
| A3 | `WindowGlass_variant_Split` | 关闭 | 验证 Split 单独复现 |
| A4 | `WindowGlass_variant_Split` | 开启 | 对照当前组合 |

每组记录：启动未登录、登录完成、最小化、展开隐藏图标面板、由真实托盘图标恢复窗口。分别记录任务栏按钮、主托盘、overflow 三处结果；记录是否重新启动 QQ/Explorer，防止把重启产生的图标重新注册误当作样式修复。

先完成上述诊断对照，之后再开展与实际改动相关的验收；不额外执行泛化一致性检查。

### B. 定位丢失发生在哪一层

优先使用上游已有的元素新增、匹配、应用及回收日志。日志不足时，才在上述关键入口添加临时 `[QQTRAY-DIAG]` 插桩；仅针对相关托盘元素，记录元素 ID、实际类型/名称、父链、匹配规则、Visibility、Opacity、布局尺寸及回收/重建事件。QQ 元素身份需要结合 tooltip/可访问名称、窗口或图标标识确认，不能靠“第几个图标”。

必要时使用 XAML 调试工具观察主托盘与 overflow 的真实树。只有证据指向注册层时，再使用带图标拥有者/GUID 与返回值的通知图标诊断；不以全系统 API hook 作为默认实现。

### C. 实现可安装的派生 mod

计划 mod ID：`windows-11-taskbar-styler-qq-fix`；计划文件：`mods/windows-11-taskbar-styler-qq-fix.wh.cpp`。这两个名称目前为预留，尚未创建修复源码。

采用 **基于原 Styler 源码的派生 mod**，复用其样式引擎并保留 Split 外观，在证实的失效位置修改。原 Styler 与派生版本在测试时交替启用，避免两个样式引擎同时覆盖同一 XAML 属性，从而影响归因与卸载恢复。Windhawk mod 没有在这里已确认的“向另一个 mod 注入补丁”的稳定扩展接口，因此不预设第二个独立样式引擎叠加运行。

具体补丁按 A/B 的证据选择：

- **主题规则问题：** 修改 Split 对应的布局/通知图标选择器或属性，限制覆盖范围，保留 Windows 自身的可见性与托盘迁移行为；同时保存对应主题 YAML 差异供上游指南使用。
- **元素生命周期问题：** 阅读将修改的完整函数及相关调用，修复共有的应用/恢复或回收路径；复用已有状态和事件机制。只有确实需要时才新增设置开关。
- **与另一个 mod 的交互：** 根据双方写入同一属性的证据，明确修复归属；若根因在另一个 mod，不把无效补丁塞进 Styler。
- **注册层问题：** 将证据与结论单独记录，先判断 Styler 是否实际触发了故障；缺少托盘注册时，仅强制 XAML `Visible` 无法创建真实图标。

派生文件保留 m417z 和贡献者归属、GPLv3 声明、基线版本；为新增改动标注派生维护者和独立版本。官方仓库要求新增 mod 的文件名与 `@id` 一致，`@github` 与提交者一致。

### D. 编译、验收和交付

按照官方开发指南，在 Windhawk 主界面选择 **Create a new mod**，将派生 `.wh.cpp` 放入其编辑环境，通过 **Compile Mod** 编译测试。使用 Windhawk 自带工具链；不把普通 C++ 编译通过当作注入和兼容性验证。

先编译未改逻辑的基线派生版本，确认现有 Windhawk 1.7.3 能正常加载，再应用补丁。同一对照中只启用原版或派生版之一。若修改了重要状态逻辑，在可触达真实失效路径的位置留下最小回归检查；仅匹配源码字符串的检查不能证明图标恢复。

验收条件：

1. 在原本会失败的最小复现流程中，连续三次登录并最小化后，真实 QQ 图标按托盘偏好出现在主托盘或 overflow，能通过原本支持的点击方式恢复窗口。
2. QQ 已登录时加载 mod、mod 已加载后登录 QQ，两种顺序均正常；图标在主托盘与 overflow 之间迁移后仍正常。
3. QQ 退出重启、重登，以及 Explorer 重建托盘后，图标可正常重新出现；不出现重复/残留图标。
4. 复现环境中的显示缩放、隐藏图标面板、其他托盘图标和 Split 两段外观无新增问题；存在多屏时再覆盖该实际环境。
5. 禁用派生版并恢复原版/原设置可回退；最终代码移除临时诊断插桩。

最终交付：可安装 `.wh.cpp`、最小修复差异、根因说明、实际复现与验收记录、安装和回退步骤。安装测试完成前不标记“已修复”。若后续向官方投稿，新增 mod 的 PR 仅携带该 `.wh.cpp`；作者原 mod 的改进建议走其开发仓库，主题规则改动走主题指南仓库。

## 6. 当前阶段完成范围

- 已完成：个人 fork、本地可操作仓库、固定基线、当前主题与版本定位、源码关键路径、原因假设及执行计划。
- 后续工作：A/B 复现、确认根因、实现派生 mod、编译及实际桌面验收。
- 本轮仅查询了本机状态，未改动 Windhawk/QQ 配置，未切换主题或重启 Explorer。
