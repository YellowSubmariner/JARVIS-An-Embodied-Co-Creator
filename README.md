# JARVIS-An-Embodied-Co-Creator

**Project JARVIS｜桌面积木共创与局部编辑的具身协作者**

An Embodied Agent for Collaborative Tabletop Creation and Local Editing.

工程导论课程项目，计划利用 **Kinova 7 自由度机械臂**，让用户通过自然语言与机械臂共同修改桌面积木作品。例如：“把屋顶的三块积木换成蓝色，窗户保持不变。”系统需要记住已完成的步骤、保留人工修改，并根据实际执行结果接续任务。

当前仓库用于维护项目方案与协作文档，完整共创系统的功能和阶段仍为实施与验收目标；已发布的现场辅助软件见下方“辅助软件”部分。模型选型、真机接入和实际表现以代码与测试记录为准。

**硬件接入更新（2026-11-02）：已通过 Kortex API 核实为 Kinova Gen3 Ultra Lightweight Robot（L53K，7 自由度），夹爪为 Robotiq 2F-85。本机通过网线访问 `192.168.1.10`，已使用 Python Kortex SDK `2.8.0.post5` 完成登录、状态读取及 J2/J3 角度参考登记。新电脑配置、现场手动摆位后标定及 Codex 提示词见 [Kinova_Kortex_Windows_从零配置与关节标定指南](docs/Kinova_Kortex_Windows_从零配置与关节标定指南.md)。腕部视觉模块已重新被设备列表识别，视频画面仍待验收。现场操作者报告此前已成功进行 VLA 推理；具体模型、动作适配和任务表现仍需项目验收记录。原先针对 FR3／DROID 的模型适配假设需重新验证。**

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

**每次承接、交接或恢复本项目，必须先阅读本 README 和以下全部维护 Markdown 的当前版本；方案、分工、接口或硬件信息变化时，必须同步更新相关文档。不能只凭聊天记录或旧版副本继续工作。**

| 文档 | 内容 |
| --- | --- |
| [JARVIS 零基础实施、训练与部署指南](docs/JARVIS_零基础实施训练与部署指南.md) | 新人入门、数据采集、代码与权重下载、约 48 小时 GPU 安排、Windows 游戏本部署和验收；2026-11-05 |
| [Background](docs/Background.md) | 项目背景、系统架构、五阶段计划与实验设计；v2.4.4，2026-11-05；保留已核实的 Gen3 与 Kortex 接入记录，按模块说明协作接口 |
| [Kinova Gen3 Windows/Kortex 配置与 J2/J3 标定指南](docs/Kinova_Kortex_Windows_从零配置与关节标定指南.md) | 新电脑网络、Python、SDK 校验、只读检查、手动摆位标定与 Codex 提示词 |
| [原始任务要求](docs/原始任务要求.md) | PuzzleMate 课程任务的截图文字整理与来源说明 |
| [第一版技术流程（Word）](docs/JARVIS第一版技术流程.docx) | 组内交流用的流程图及各阶段说明，π0.5 为待验证的候选 VLA |

## 推进计划

1. **单动作闭环**：完成一次 VLA 真机取放，支持核验、停止与失败暂停。
2. **多步编辑与记忆**：完成局部换色、区域保护、预览确认与暂停接续。
3. **人工协作与恢复**：记录人工修改，加入语音、上下文澄清及至少一种实际异常恢复；完成不少于 20 条独立测试指令和 10 次真机局部编辑。
4. **VLA 微调改进**：固定 Harness 与测试条件，比较微调前后的动作和任务表现。
5. **仿真与研究扩展**：围绕明确问题开展仿真与真机对照。

前三阶段构成课程主要交付；微调与仿真可按需要提前。基础场景限定为固定相机、单层网格、三色积木及拼图／备料／回收分区。

## 文档维护

后续以仓库 `docs/` 中的文档为维护入口。调整技术路线、接口或实施安排时，同步更新 Background、实施指南及本 README，并注明日期与版本；原始任务要求保留为课程需求依据。实施进展和实验结论应附对应代码或测试记录。

GitHub：[YellowSubmariner/JARVIS-An-Embodied-Co-Creator](https://github.com/YellowSubmariner/JARVIS-An-Embodied-Co-Creator)
