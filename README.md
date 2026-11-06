# JARVIS-An-Embodied-Co-Creator

**Project JARVIS｜桌面积木共创与局部编辑的具身协作者**

An Embodied Agent for Collaborative Tabletop Creation and Local Editing.

工程导论课程项目，计划利用 **Kinova 7 自由度机械臂**，让用户通过自然语言与机械臂共同修改桌面积木作品。例如：“把屋顶的三块积木换成蓝色，窗户保持不变。”系统需要记住已完成的步骤、保留人工修改，并根据实际执行结果接续任务。

当前仓库用于维护项目方案与协作文档，完整共创系统的功能和阶段仍为实施与验收目标；已发布的现场辅助软件见下方“辅助软件”部分。模型选型、真机接入和实际表现以代码与测试记录为准。

**当前路线（2026-11-06）：选择 π0.5，以 LeRobot 的 PyTorch＋LoRA 路线开展自采 Kinova 数据微调。此前的 VLA 代码和权重无法取得；团队报告已有控制程序或遥操作可完成取放，下一步检查其接口并接入录制。现场电脑承担采集、控制、数据处理与 Agent 开发；π0.5 的本地推理尚未验证，训练与备用推理资源需先小规模试跑。具体电脑清单在团队本地保存。**

**硬件接入更新（2026-11-02）：已通过 Kortex API 核实为 Kinova Gen3 Ultra Lightweight Robot（L53K，7 自由度），夹爪为 Robotiq 2F-85。通过网线连接机械臂，已使用 Python Kortex SDK 完成登录、状态读取及 J2/J3 角度参考登记。新电脑配置、现场手动摆位后标定及 Codex 提示词见 [Kinova_Kortex_Windows_从零配置与关节标定指南](docs/Kinova_Kortex_Windows_从零配置与关节标定指南.md)。腕部视觉模块已重新被设备列表识别，视频画面仍待验收。现场操作者报告此前已成功进行 VLA 推理；具体模型、动作适配和任务表现仍需项目验收记录。原先针对 FR3／DROID 的模型适配假设需重新验证。**

## 辅助软件：JARVIS Robot Console

我们已为本项目开发并独立发布 [JARVIS Robot Console](https://github.com/YellowSubmariner/JARVIS-Robot-Console)，作为 Windows 现场视觉辅助软件。当前提供**腕部相机**和 **Astra 全局相机**两个页面，支持两路独立取流、连接与重连、断线自动重试和截图，并为后续增加功能页面预留结构。

- **软件仓库与使用说明：** [YellowSubmariner/JARVIS-Robot-Console](https://github.com/YellowSubmariner/JARVIS-Robot-Console)
- **Windows 下载：** [v0.1.0 发布页](https://github.com/YellowSubmariner/JARVIS-Robot-Console/releases/tag/v0.1.0)，完整解压后运行 `JARVIS_Robot_Console.exe`。

当前版本用于视频监看和现场联调，机械臂运动控制、坐标标定与 VLA 执行仍需分别接入和验收。腕部视觉模块已重新被设备列表识别，视频画面仍待验收。

## 技术路线

- **LLM**：理解用户要求、关联上下文，参与任务层决策。
- **VLA**：根据当前子任务、最新图像和机器人状态生成具体动作。
- **Physical Agent Harness**：从第一阶段管理目标、记忆、进度、核验与有限异常恢复。
- **ScenePatch**：规划局部修改，在虚拟工作台中检查占用、材料和保护约束，并展示预览。

基本闭环：理解任务 → 观察与规划 → 预览确认 → VLA 执行 → 视觉核验 → 更新记忆并继续。只有实物核验通过，才记录任务完成。

## 项目文档

从 [文档导航](docs/README.md) 按当前任务阅读。承接、交接或恢复项目时，先读本 README、项目方案、实施总览和任务清单，再读对应专题；修改方案、接口或硬件信息时，同步更新相关文档。能力结论应有代码与测试记录，不能只凭聊天或旧版副本继续工作。

| 文档 | 内容 |
| --- | --- |
| [开题报告叙事与结构建议](docs/Project_JARVIS_开题报告叙事与结构建议.md) | 以“赛博造物”统一背景、项目目标与创新点，衔接十章报告结构；2026-11-07 |
| [π0.5：Kinova 数据采集与微调部署指南](docs/PI05_Kinova_数据采集与微调部署指南.md) | 关节角生成动作标签、双相机与 Kortex 采集、数据转换、权重下载、LoRA 训练及部署；当前操作入口 |
| [JARVIS 分阶段任务清单](docs/JARVIS_分阶段任务清单.md) | P0–P7 的可勾选任务、交付物、阶段通过条件和 GPU 总预算；2026-11-06 |
| [JARVIS 实施总览](docs/JARVIS_零基础实施训练与部署指南.md) | 项目模块、开发顺序、Agent 并行工作、约 48 小时 GPU 预算与课程验收；保留原文件路径 |
| [Background](docs/Background.md) | 项目背景、系统架构、五阶段计划与实验设计；v2.4.5，2026-11-06 |
| [Kinova Gen3 Windows/Kortex 配置与 J2/J3 标定指南](docs/Kinova_Kortex_Windows_从零配置与关节标定指南.md) | 新电脑网络、Python、SDK 校验、只读检查、手动摆位标定与 Codex 提示词 |
| [原始任务要求](docs/原始任务要求.md) | PuzzleMate 课程任务的截图文字整理与来源说明 |
| [第一版技术流程（Word）](docs/JARVIS第一版技术流程.docx) | 历史参考：保留早期流程图，当前模型、训练命令与实施安排以维护中的 Markdown 为准 |

## 推进计划

1. **单动作闭环**：完成一次 VLA 真机取放，支持核验、停止与失败暂停。
2. **多步编辑与记忆**：完成局部换色、区域保护、预览确认与暂停接续。
3. **人工协作与恢复**：记录人工修改，加入语音、上下文澄清及至少一种实际异常恢复；完成不少于 20 条独立测试指令和 10 次真机局部编辑。
4. **VLA 微调改进**：固定 Harness 与测试条件，比较微调前后的动作和任务表现。
5. **仿真与研究扩展**：围绕明确问题开展仿真与真机对照。

前三阶段构成课程主要交付；微调与仿真可按需要提前。基础场景限定为固定相机、单层网格、三色积木及拼图／备料／回收分区。

## 文档维护

文档职责与更新规则见 [文档导航](docs/README.md)。训练细节集中维护在 π0.5 专题，项目方案与任务清单引用它，减少多处命令相互矛盾；原始任务要求保留为课程需求依据。历史版本保留在 Git 提交记录中。

GitHub：[YellowSubmariner/JARVIS-An-Embodied-Co-Creator](https://github.com/YellowSubmariner/JARVIS-An-Embodied-Co-Creator)
