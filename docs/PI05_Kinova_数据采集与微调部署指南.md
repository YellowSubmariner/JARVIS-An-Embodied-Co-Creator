# JARVIS：π0.5 数据采集、动作标签、微调与部署指南

**版本：v1.0｜2026-11-06｜适用：Kinova Gen3 7 自由度 + Robotiq 2F-85 + 全局、腕部两路 RGB**

本文把“示范怎么录、关节角怎样变成标签、权重怎么下载、模型怎么微调、怎样接回机械臂”连成一条路线。当前选择 **LeRobot 的 PyTorch π0.5 + LoRA**，用自己的 Kinova 示范微调官方基础权重。

**验证边界：** 本文依据官方文档和固定版本源码核对接口，尚未在本项目上执行模型下载、GPU 训练或真机推理。示例中的数据契约是待实现和验收的约定；录制器、时间对齐器、Kinova 执行适配器仍需开发。用户已确认现有控制程序或遥操作能完成取放，但其代码、动作接口和实际频率尚未检查。文中的数据量、采样率、训练步数和资源余量是试验起点，不是成功保证。

项目总览见 [README](../README.md)，阶段推进见 [任务清单](JARVIS_分阶段任务清单.md)，网络与 SDK 环境见 [Kortex 配置记录](Kinova_Kortex_Windows_从零配置与关节标定指南.md)。

阅读顺序：第 1 节理解标签；第 2–6 节准备并采集；第 7 节转换数据；第 8–9 节下载与训练；第 10 节部署；第 11–12 节验收与安排。GitHub 右上角的文档大纲可跳转到各节。

## 1. 先回答：学长说“根据关节角生成标签”是什么意思

模型在每个时刻需要一组输入和一个正确答案：

```text
输入：当前两张图 + 当前 7 个关节角和夹爪状态 + 当前短任务
答案：接下来应执行的动作（训练标签）
```

这里的“标签”通常是若干个连续数值，不是人工给每一帧写“正在抓取”。例如机械臂用关节位置控制，动作就可以是 **7 个关节目标角 + 1 个夹爪目标 = 8 维向量**。任务文字如“把红色积木放到左侧标记框内”是另一类标签，一段示范通常填写一次。

### 1.1 三种标签来源要分清

| 方案 | 标签如何得到 | 前提与含义 |
| --- | --- | --- |
| A：记录控制目标 | 在控制程序真正发送动作的位置，记录通过限幅后的关节目标、夹爪目标和时间 | 优先检查是否具备；学习示范控制器的连续位置命令 |
| B：从实际轨迹生成目标 | 使用示范中下一采样时刻的实际关节角 `q(t+Δ)` | 学习沿实测轨迹前进；部署端要能按相同节奏追踪位置目标，需验证跟踪误差 |
| C：相对位移 | 例如 `q(t+Δ)-q(t)` | 也可以，但训练和执行必须都使用同一定义；首版不同时维护两套动作语义 |

学长说的很可能是 **B**。它是可行的轨迹重标注方法；究竟采用哪一种，要看现有程序实际能导出什么。**“只有实测关节角就永远无法训练”不准确；“同一帧关节角复制为动作就能训练”也不准确。**

例如，按 20 Hz 对齐，`Δ=0.05 秒`，这里只展开第一个关节，其余关节同理：

| 时间 | 当前实测角度，作为 state | 下一时刻实测角度，作为方案 B 的 action |
| --- | --- | --- |
| 0.00 s | 30° | 31° |
| 0.05 s | 31° | 32° |
| 0.10 s | 32° | 32.5° |

模型看到 0.00 s 的图像、状态和任务，学习预测接下来去 31°。如果动作是实际控制目标，控制程序此刻可能发送 31.4°，就应记录 31.4°；不能声称它与下一帧实际到达的 31°是同一个量。

**不要将 `state[t]` 原样写为 `action[t]` 来代表运动。** 这通常在教模型维持原位。稳定等待片段可以有相同数值，但取放过程应包含真实变化。

### 1.2 夹爪是最容易漏掉的一个标签

夹爪的“当前开合位置”和“控制目标”必须分别记录。夹爪收到完全关闭目标后，可能因为夹住积木而停在中间；中间位置不能自动替代“保持闭合”的目标。

首版建议：**关节轨迹可以用未来实测角重标注，夹爪尽量保留原程序发出的开合目标**。若只能获得夹爪反馈，需要根据操作者开/关事件和接触过程补充可靠标签，再检查回放；不能仅凭电流升高就判定抓取成功。

### 1.3 只有稀疏航点，也需要处理

若原程序只发一次“移动到某姿态”，随后机器人运动两秒，这是一条高层航点命令，不能未经处理就复制为 40 帧连续低层动作。应选择以下之一并固定：

1. 记录内部连续位置设定值，采用方案 A；
2. 记录实际轨迹，采用方案 B，并验证部署用的位置跟踪控制器；
3. 保持原本的高层动作接口，另外设计与其匹配的策略和时间语义。

反复调用高层 `ExecuteAction` 规划接口不等于连续关节伺服；不要把 Windows 上的 Python 录制循环直接当作机器人实时低层控制器。复用已验证的运动执行方式，先证明它能按定义跟踪这些目标。

若原程序发的是笛卡尔速度或关节速度，先按真实单位保存原始命令；不要把这些数值重命名成“关节目标角”。本指南以下的 **8 维位置动作示例仅在位置控制契约通过后适用**。

## 2. 现在已经有什么，还缺什么

| 项目 | 2026-11-06 已知情况 | 对实施的影响 |
| --- | --- | --- |
| 机器人 | Kinova Gen3 L53K、7 自由度，Robotiq 2F-85 | 使用自己的关节、夹爪和相机定义；不套用 DROID/Franka 动作 |
| 现有示范方式 | 用户确认已有控制程序或遥操作能取放 | 不必先让未训练的 VLA 做示范；需要检查并接入记录点 |
| 旧 VLA | 旧代码与权重已无法取得 | 本轮从公开 π0.5 base 加自采数据开始 |
| 笔记本 | 现有 Windows 现场电脑，详细配置在团队本地保存 | 可做采集、Kortex、界面、Agent、数据处理；常规 π0.5 推理是否可用仍需实测 |
| 相机软件 | [Robot Console](https://github.com/YellowSubmariner/JARVIS-Robot-Console) 提供双路监看、重连和截图 | 可以复用取流代码；它目前不是已验收的训练数据录制器 |
| 需要补的软件 | 录制器、同步与标签转换、数据检查、推理与机器人之间的桥接 | 不存在可以直接复制的 `--robot.type=kinova_gen3` 一键全流程 |

设备端沿用已经验证的 Kortex Python 环境，具体依赖版本按本地环境记录保留。模型环境单独建立，避免为了安装训练依赖破坏机器人已有环境。

**不必等完整 Agent 框架搭好。** 数据线可以独立做“给定一个短任务 → 人示范 → 记录”。Agent 侧同时用静态照片、场景表和模拟执行结果开发语言理解、区域保护、记忆、预览及恢复逻辑。首次模型驱动真机前，执行侧必须已有停止、超时、限幅、日志与结果核验。

## 3. 具体需要采集什么

### 3.1 最小训练记录与辅助记录

| 数据 | 采集来源 | 是否作为首版模型输入/标签 |
| --- | --- | --- |
| 全局 RGB、腕部 RGB | 两个相机，各自取流 | 输入，必须固定相机身份、位置、裁剪和颜色通道 |
| 7 个实际关节角 | Kortex 反馈 | 输入；也可用于方案 B 的未来角度标签 |
| 夹爪实际开合位置 | Kortex/夹爪反馈，现场确认可读字段 | 输入 |
| 连续动作命令及发送时间 | 现有控制程序发送点 | 方案 A 的标签；同时帮助核验延迟、夹爪意图 |
| 短任务文字 | 人在每个 episode 开始时选择或填写 | 输入，不需每帧手工重复填写 |
| 真实时间戳、帧序号 | 采集程序、设备时间（如可用） | 对齐、质量检查和追溯，通常不直接输入模型 |
| 成功/失败/介入/中止 | 操作者 + 执行后图像 | 筛选、评估与后续恢复研究，不等于逐帧动作标签 |
| 速度、力矩、电流、末端位姿、故障 | Kortex 能读到的反馈 | 建议保留用于排查；首版不强行全部塞入 state |

深度图、逐帧目标框、逐帧“抓取阶段”标注不是这个 RGB + 关节位置方案的必需项。相机标定仍服务于网格映射、独立结果核验和执行约束；模型能读图不等于整个项目不需要标定。

### 3.2 Kortex 能直接读什么

以下依据官方协议定义；**本机 SDK、固件和夹爪连接方式仍要实际回读确认**。[K1][K2][K3]

| 字段或接口 | 含义与单位 |
| --- | --- |
| `BaseCyclic.RefreshFeedback()` | 只读反馈；不要与会发送命令的 `Refresh(Command)`、`RefreshCommand` 混淆 |
| `feedback.actuators[i].position` | 关节角，度；核对关节顺序与编号，不盲信设备列表顺序 |
| `.velocity` / `.torque` / `.current_motor` | 关节速度 deg/s、力矩 N·m、电流 A |
| `feedback.base.tool_pose_x/y/z` | 控制器估计的末端位置，米 |
| `tool_pose_theta_x/y/z` | 控制器末端姿态表示，度；不是外部测量的真值 |
| `feedback.base.commanded_tool_pose_*` | 部分末端指令目标；需要验证控制模式和更新语义，不等于所有模式下的完整 7 关节动作标签 |
| `feedback.interconnect.gripper_feedback.motor[...]` | 视连接而定；position 为百分比尺度，电流等字段按相应协议读取 |
| `Base.GetMeasuredJointAngles()` 等 | 高层只读接口，可用于检查；采集频率要实测 |

循环夹爪反馈的 `position` 通常是 **0–100**；高层位置命令使用 **0–1**。原始值连单位一起保存，然后统一映射。务必现场核对哪一端是开、哪一端是关，不把反馈百分比直接当毫米，也不把电流等同夹持力。

腕部 RGB 一般走独立视频流，地址模板为 `rtsp://<ROBOT_IP>/color`，其中 `<ROBOT_IP>` 在本地配置中填写；外部 Astra 是单独 USB 视频设备。**两路图像不包含在 `RefreshFeedback()` 返回包中**。先验收真实画面，再进行联合录制。

## 4. 先冻结一个“动作与数据契约”

契约就是所有程序共同遵守的一张表，写入每次采集的 metadata。下表是**建议模板**，不是已经验证的机器人配置。

| 项目 | 首版建议及必须确认的内容 |
| --- | --- |
| 任务范围 | 单块取放，先固定颜色、少数明确目标框，之后再扩展 |
| state 顺序 | `[q1,q2,q3,q4,q5,q6,q7,g_measured]` |
| action 顺序 | `[q1_target,...,q7_target,g_target]`；声明 A 或 B 标签来源 |
| 关节单位 | 原始 API 保留度；模型端可统一为弧度，状态与动作一致；发送前按接口转换回去 |
| 夹爪单位 | 模型端统一 0–1，明确开/关方向；实际反馈与目标分别存放 |
| 关节分支 | 处理 359°→1° 等角度回绕；训练与执行共享同一关节参考和映射 |
| 数据频率 | 先以 20 Hz 为候选；按图像延迟、控制能力实测后固定，不能仅改 fps 元数据 |
| 标签间隔 | 方案 B 示例 `Δ=1/fps`；改变 Δ 意味着改变目标前视时间，应生成新数据版本 |
| 相机键 | `observation.images.top`、`observation.images.wrist` |
| 图像 | RGB，固定分辨率和视野；保留原始图，不只留训练时的 224×224 图 |
| 控制频率 | 与模型决策频率分开；底层控制器可能需要更高频平滑跟踪 |
| 标定与版本 | 关节参考、夹爪范围、相机摆位、控制程序、转换器均保存版本 |

对连续转动关节，不能把 359°→1°直接相减成 −358°。需要按关节类型、限位和统一参考选择连续角度或等价角分支。不要每个 episode 随意重新设一个“零点”，否则同一物理姿态可能对应不同输入。若在数据中使用连续展开角，执行适配器必须明确怎样映射回控制器接受的目标。

文本目标也要有可学习的依据。`A3` 或 `red_03` 只是软件编号，基础模型不天然认识它。可在桌面使用固定可见标记、固定目标框，并在示范中一致对应；若采用图像叠加标记，训练和推理必须完全一致，不能只在部署时临时加框。

## 5. 怎样录制一次完整示范

### 5.1 先做录制程序，再批量采集

建议在现有程序旁增加以下能力：

1. **反馈读取：** 只读 Kortex 状态，记录请求/接收时间、序号及异常；开始时不要为了提速改机器人控制模式。
2. **两路取流：** 每路独立保存帧、设备时间（如可用）、主机接收时间和帧序号；检测缓存积压。
3. **动作记录：** 在实际发送点记录命令类型、值、目标生效时段、发送时间、控制模式及经过的限幅；保存原请求和最终发出值，训练使用与执行定义对应的一份。
4. **episode 控制：** 开始、结束、取消、成功/失败/介入标记；同一段内不能混入下一次任务或复位。
5. **回放与质量报告：** 同屏看两路视频、关节曲线、夹爪命令；显示缺帧、时延、停顿、异常和任务文字。

先录 **5–10 条短示范**验证程序。看见两路视频能同时播放，只证明能取流，不证明训练所需的时序正确。

### 5.2 操作者实际流程

```text
准备初始布局、检查设备和等待姿态
→ 选择短任务
→ 开始录制，保存一小段稳定初始观察
→ 用已有控制程序/遥操作完成取放
→ 松开积木，机械臂回到该任务约定的终态并稳定
→ 结束录制，标记成功/失败/是否介入，保存结果图
→ 停止记录后复位场景，开始下一条
```

“回到等待姿态”是否属于任务，要提前固定。如果属于任务就一致录进去；如果只是下一次示范的准备，就在结束记录后进行。失败和中止原始记录仍保留，但首轮成功模仿数据只纳入经过检查的有效段。

### 5.3 保存布局

推荐的原始日志布局是团队自定义格式，**不是 LeRobot 目录格式**：

```text
raw/episode_000001/
  metadata.json             # task、操作者、结果、动作契约、版本
  robot_feedback.jsonl      # 原始反馈 + 独立时间戳
  commands.jsonl            # 真正发出的命令 + 独立时间戳
  top.mp4 / top_frames.jsonl
  wrist.mp4 / wrist_frames.jsonl
  events.jsonl              # 开始、抓取、松开、介入、中止等
  outcome_top.png
```

有条件时保留能追溯到每一帧的源时间/呈现时间及主机时钟映射；不能只保存 MP4 而丢掉采集时间。JSON 中不写机械臂密码、HF token 等凭据。

## 6. 同步、清洗和自动生成动作标签

### 6.1 对齐的是实际时刻，不是保存文件的时刻

所有主机端采集记录统一使用单调时钟，另存一次与墙钟时间的对应关系。两台机器时钟需要同步并估计偏差。RTSP 可能缓冲数百毫秒，`read()` 返回时刻不一定是曝光时刻；保留可用的源时间，并用可观察的运动事件实测延迟。

首版统一以**本机决策时间网格 `t_k`**为准，采用部署时也能复现的读取规则：

1. 两路图像分别选在 `t_k` 前已经可获得的最新有效帧，保存曝光时间估计、接收时间和各路帧龄。
2. 关节和夹爪输入取 `t_k` 时已可获得的最新反馈，或在线同样能实现的因果状态估计；不要为了匹配旧 RTSP 图像而默默回退全部状态时间。
3. 方案 A 的动作对应 `t_k` 之后控制区间的指令；方案 B 用下一决策采样点的关节轨迹值生成目标。未来记录只用于监督标签，输入状态不能依赖部署时尚未到达的未来反馈插值。
4. 检查延迟是否稳定，拒收丢帧严重、画面冻结、时间倒退或延迟不可解释的片段。

离线不能使用“未来才到达的画面”而在线又拿不到相同信息。曝光时间用于估计延迟和拒收陈旧帧，不与动作起始的决策时间混用。若另选同步到共同历史曝光时刻的方案，必须重新定义动作相对当前决策时间的偏移并验证。不要用一个统一保存时间覆盖所有传感器时间戳。

20 Hz 的间隔是 50 ms；帧龄和时序误差的合格阈值须根据实测运动速度、相机帧率和控制容差制定，不能宣称所有设备都能达到某个固定毫秒值。减少运动速度或改变数据频率后，要更新契约并复验。

### 6.2 方案 B 的最小数学例子

已经完成时间整理、单位转换和角度分支处理后，设 `q[k]` 为按上述 `t_k` 和在线可复现规则读取的 7 维关节状态序列，`g_cmd[k]` 为 `t_k` 后该控制区间应维持的夹爪目标：

```python
import numpy as np

def relabel_next_joint_position(q, g_measured, g_cmd):
    # 前提：q 是同步后的 [N, 7] 数组，角度分支和单位已处理。
    # g_cmd 是与各控制区间对齐的目标，不是接触后的夹爪实测位置。
    q = np.asarray(q, dtype=np.float32)
    g_measured = np.asarray(g_measured, dtype=np.float32)
    g_cmd = np.asarray(g_cmd, dtype=np.float32)
    assert q.ndim == 2 and q.shape[1] == 7 and len(q) >= 2
    assert g_measured.shape == g_cmd.shape == (len(q),)
    state = np.column_stack([q[:-1], g_measured[:-1]])
    action = np.column_stack([q[1:], g_cmd[:-1]])
    assert np.isfinite(state).all() and np.isfinite(action).all()
    return state, action
```

对应图像和任务取前 `N-1` 帧。最后一个原始状态没有下一状态，不将下一 episode 的第一帧拿来补。这个函数只演示重标注，不解决相机同步、控制延迟、夹爪事件推断和真实机器人跟踪问题。

### 6.3 π0.5 的 action chunk 与 episode 尾部

π0.5 一次可以预测一段未来动作。若每行 action 定义为“下一步关节目标”，长度 H 的训练目标就是连续 H 行，覆盖未来 H 个控制间隔；**不要先自己平移一次，又在训练前重复平移一次**。

固定版本的数据读取器会在 episode 边界重复端点并给出 `action_is_pad`；但本指南核对的 **π0.5 loss 不自动使用这个时间填充 mask**。它裁掉的是补齐到 32 的多余动作维度，这与 episode 末端填充不同。[L4][L5]

首版采用明确的“任务结束后保持终态”约定：示范末端已经稳定、夹爪目标正确，末行代表持续保持该状态。这样重复末行才有合理含义。不得把急停瞬间、复位动作或未完成抓取当成这样的终态。若希望严格排除不完整动作窗口，需要额外修改并测试采样器或 loss；此版本没有可以直接照抄给 π0.5 的 `--policy.drop_n_last_frames` 参数。

### 6.4 采多少条、怎样划分

| 阶段 | 建议起点 | 要回答的问题 |
| --- | --- | --- |
| 录制验收 | 5–10 条 | 时间、画面、单位、标签、保存和回放是否正确？ |
| 窄任务试训 | 约 50–60 条有效成功示范 | 单一技能在少数起终位置能否学会？ |
| 扩展 | 根据失败补到约 150–300 条或更多 | 颜色、位置、布局和遮挡变化是否覆盖？ |

这些是安排工作的起点，**不是 π0.5 在 Kinova 上的样本复杂度结论**。同一位置重复 60 次和覆盖少数有意义变化的 60 次，不是同一种数据质量。控制任务范围，避免每条示范都是完全不同的新任务。

先按完整 episode、布局或采集批次划分 train/val/test，例如 70%/15%/15%，再分别转换。相邻帧不可随机分到不同集合，同一原始示范切出来的片段也不能跨集合。训练统计量只由 train 计算；val 用于选 checkpoint，test 最后一次验收使用。

## 7. 转换成 LeRobot 数据集

### 7.1 首版特征定义

```text
observation.images.top    RGB 图像
observation.images.wrist  RGB 图像
observation.state         float32[8]
action                    float32[8]
task                      当前短任务文字
```

这两个相机键和 8 维向量是我们自己的数据定义。训练命令采用 `policy.type + pretrained_path`，从数据集取得特征；不要直接套用 LIBERO 的 7 维动作或 SO-100 的关节顺序。模型内部保留 `max_state_dim=32`、`max_action_dim=32`，由模型补齐；**不要为适配 8 维数据把这两个内部维度改成 8**。[L1][L3][L4]

### 7.2 一个最小转换器

下面是**已同步、已筛选、已生成标签之后**的转换示例。它不直接连接相机和机器人，也不会替你纠正原始数据错误。先实现导出每个 episode 一个 `.npz`，字段如下：

| NPZ 字段 | 格式 |
| --- | --- |
| `state`, `action` | `[N,8]`，float32，前文契约 |
| `images_top`, `images_wrist` | `[N,H,W,3]`，uint8，RGB；每路尺寸在数据集内固定 |
| `task` | 单个字符串 |
| `fps` | 单个整数，例如 20 |

NPZ 方案便于小样本验收；大批量原始视频应流式解码转换，避免把整批视频一次加载进内存。原始精确时间戳留在原始日志和转换报告中，LeRobot 帧时间来自已验收的等间隔网格。

在第 8 节的 LeRobot 环境中，将下面代码保存为 `convert_aligned_npz.py`：

```python
import argparse
from pathlib import Path
import numpy as np
from lerobot.datasets.lerobot_dataset import LeRobotDataset

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--source", type=Path, required=True)
    parser.add_argument("--root", type=Path, required=True)
    parser.add_argument("--repo-id", required=True)
    args = parser.parse_args()
    files = sorted(args.source.glob("*.npz"))
    if not files:
        raise ValueError("没有找到已对齐的 episode NPZ")
    if args.root.exists():
        raise ValueError("输出目录必须不存在，避免覆盖旧数据版本")
    with np.load(files[0], allow_pickle=False) as first:
        fps = int(first["fps"].item())
        shapes = {name: tuple(first[f"images_{name}"].shape[1:])
                  for name in ("top", "wrist")}
    if fps <= 0:
        raise ValueError("fps 必须为正")
    joints = [f"q{i}" for i in range(1, 8)]
    features = {
        "observation.state": {"dtype": "float32", "shape": (8,),
                              "names": joints + ["gripper_measured"]},
        "action": {"dtype": "float32", "shape": (8,),
                   "names": [f"{q}_target" for q in joints] + ["gripper_target"]},
    }
    for name, shape in shapes.items():
        if len(shape) != 3 or shape[-1] != 3:
            raise ValueError("图像必须是 HWC RGB")
        features[f"observation.images.{name}"] = {
            "dtype": "video", "shape": shape,
            "names": ["height", "width", "channels"],
        }
    dataset = LeRobotDataset.create(
        repo_id=args.repo_id, root=args.root, fps=fps,
        robot_type="kinova_gen3", features=features,
        use_videos=True, image_writer_threads=4,
    )
    # robot_type 在这里仅是数据集说明，不会创建 Kinova 驱动。
    try:
        for path in files:
            with np.load(path, allow_pickle=False) as ep:
                states, actions = ep["state"], ep["action"]
                images = {name: ep[f"images_{name}"] for name in shapes}
                n = len(states)
                if n < 2 or int(ep["fps"].item()) != fps:
                    raise ValueError(f"帧数或 fps 不合格：{path}")
                for key, a in (("state", states), ("action", actions)):
                    if a.shape != (n, 8) or a.dtype != np.float32:
                        raise ValueError(f"{path}: {key} 格式错误")
                    if not np.isfinite(a).all():
                        raise ValueError(f"{path}: {key} 有 NaN/Inf")
                task = str(ep["task"].item()).strip()
                if not task:
                    raise ValueError(f"任务文字为空：{path}")
                for name, shape in shapes.items():
                    a = images[name]
                    if a.shape != (n, *shape) or a.dtype != np.uint8:
                        raise ValueError(f"{path}: {name} 图像格式错误")
                for i in range(n):
                    dataset.add_frame({
                        "observation.state": states[i],
                        "action": actions[i],
                        "observation.images.top": images["top"][i],
                        "observation.images.wrist": images["wrist"][i],
                        "task": task,
                    })
                dataset.save_episode()
    finally:
        dataset.finalize()
    print(f"已转换 {len(files)} 个 episode：{args.root}")

if __name__ == "__main__":
    main()
```

转换发生异常时，该输出视为未通过验收的数据版本；保留错误日志，修好源数据后用新目录重跑。`finalize()` 会完成写盘收尾，不能遗漏。[L5]

```bash
uv run --no-sync python convert_aligned_npz.py \
  --source /data/jarvis/aligned/train \
  --root /data/jarvis/datasets/train \
  --repo-id local/jarvis_train

uv run --no-sync python convert_aligned_npz.py \
  --source /data/jarvis/aligned/val \
  --root /data/jarvis/datasets/val \
  --repo-id local/jarvis_val

uv run --no-sync python convert_aligned_npz.py \
  --source /data/jarvis/aligned/test \
  --root /data/jarvis/datasets/test \
  --repo-id local/jarvis_test
```

`local/jarvis_train` 在这里是本地标识；完整提供 `root`，不需要先上传 Hugging Face，也不应把视频数据或模型权重提交到项目 GitHub。

训练前至少检查：完整 episode 数与原记录一致、两路图像可解码、RGB 正确、state/action 有限、单位与夹爪方向正确、动作变化合理、帧时间递增、无跨 episode 标签、末端稳定、train/val/test 原始 episode 清单不重叠。转换成功不代表这些检查自动通过。

## 8. 资源、环境与权重下载

### 8.1 资源怎样安排

| 用途 | 本轮安排 |
| --- | --- |
| 采集、录制、Agent、界面、预处理 | 现有游戏本可承担；优先把原始视频放在空间充足的磁盘 |
| 基础权重磁盘 | 所选 `lerobot/pi05_base` 的 `model.safetensors` 约 14.47 GB，约 13.47 GiB，另有小配置文件；这是该 F32 文件大小，不是推理显存 [M1] |
| LoRA 训练 | Linux 云 GPU；24 GB 卡只能视为小 batch 试跑候选，有条件选 40/48 GB 级别增加余量；必须实测该配置的峰值 |
| 全量微调 | 本轮不作为默认方案，资源需求明显更高 |
| 推理 | 在可用 GPU 上先测试；16–24 GB 是工程试验候选，不能据此保证速度。现场电脑常规部署未验证；若仅有约 8 GB 显存，尤其需要先测加载与峰值 |
| 磁盘计划 | 环境、base 和缓存先预留约 50–80 GB；云训练含数据和多份 checkpoint 可先留 150–200 GB，再按实测增加 |
| 主机内存 | 32 GB 从小 batch 和少量数据加载进程开始；64 GB 云主机更有余量；大视频应分批处理 |

openpi 官方给出的该框架一般估计是推理 `>8 GB`、LoRA `>22.5 GB`、全量微调 `>70 GB`。这些是 **openpi 的配置参考，不是本指南 LeRobot 配置的实测最低值**。[O1]

显存较小的游戏本仍能完成大量现场和软件工作。是否能通过特定精度、优化或量化运行 π0.5，必须另做兼容、精度和延迟验证，不将未经测试的量化方案作为课程交付前提。

### 8.2 固定版本

| 组件 | 本指南固定值 |
| --- | --- |
| LeRobot | `9e5ee9859480e425a97a6836bb3fee479539beb5` |
| π0.5 基础权重 | `lerobot/pi05_base` |
| 权重 revision | `22c30e5b34f9e7ab738b1b34fc2fc5ecfe2d28f0` |
| 模型环境 | Linux，Python 3.12，独立虚拟环境 |
| 微调方式 | LeRobot PyTorch + PEFT LoRA |

LeRobot 支持这条 π0.5 LoRA 路线；不能把 openpi 自己的 PyTorch LoRA 限制泛化成“所有 π0.5 都不支持 PyTorch LoRA”。两套框架分别管理依赖，本轮不混装。[L1][L2]

以下安装与训练命令使用 **Linux Bash**，不直接复制到 Windows PowerShell。云主机先确认 `git`、`uv`、NVIDIA 驱动和所需视频编解码依赖；`uv` 安装参见 [官方安装说明](https://docs.astral.sh/uv/getting-started/installation/)。

```bash
nvidia-smi
mkdir -p /data/jarvis/src /data/jarvis/models /data/jarvis/cache
cd /data/jarvis/src
git clone https://github.com/huggingface/lerobot.git
cd lerobot
git checkout 9e5ee9859480e425a97a6836bb3fee479539beb5
uv sync --locked --python 3.12 --extra training --extra pi --extra peft
uv run --no-sync python -c 'import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))'
```

输出应能识别实际 GPU。该版本安装说明的默认 Linux CUDA 13 wheel 要求 NVIDIA 驱动至少 580.65；云主机检查自己的驱动，并准备 FFmpeg 等视频依赖。安装失败先检查驱动、CUDA/PyTorch 组合及视频依赖，不随意升级整个锁定环境。训练机器的完整 `nvidia-smi`、代码 SHA 和最终依赖清单保存进实验记录。[L8]

### 8.3 下载权重与分词器资源

在 Hugging Face 登录并按页面要求取得 [PaliGemma](https://huggingface.co/google/paligemma-3b-pt-224) 资源访问权限。模型需要分词器；不需要为了分词器再下载一套完整 PaliGemma 大权重。凭据通过登录管理，不写进脚本或仓库。

```bash
export HF_HOME=/data/jarvis/cache/huggingface
uv run --no-sync hf auth login
uv run --no-sync hf download lerobot/pi05_base \
  --revision 22c30e5b34f9e7ab738b1b34fc2fc5ecfe2d28f0 \
  --local-dir /data/jarvis/models/pi05_base
```

第一次处理样本时会需要相应分词器文件；在联网环境完成一次真实预处理和离线推理检查，保留所需缓存，并记录实际解析出的资源 revision。复制部署时带走所需 tokenizer 文件/缓存，**不复制登录 token**。断网重启后能重新加载，才算离线资源准备通过。

不要只拿一个 `model.safetensors` 文件：保留整个基础权重目录中的配置与处理器资源。若在多处保存副本或转换精度，磁盘使用会增加。

## 9. 先做 200 步试跑，再决定正式训练

完整顺序是：数据回放通过 → 转换并读取 → 建立模型和处理器 → 小步训练 → 保存 → 重新加载 → 离线输出检查 → 决定长训。训练是在调整参数，200 步首先用于发现管线问题，不代表机器人已经学会取放。

### 9.1 归一化与配置

本指南默认使用 π0.5 的 `QUANTILES` 归一化，要求 train 统计里有 `q01/q99`。它把主要训练范围映射到约 `[-1,1]`，与状态离散分箱相配；它并不自动裁剪所有离群点，范围外的状态仍要检查。[L3][L6]

固定版本的数据写入器会生成分位数统计；跨 episode 聚合值是保守包络。建议转换完成后，只对 **train** 目录运行官方脚本，重算全体训练样本的直方图分位数估计，再开始训练：

```bash
uv run --no-sync python src/lerobot/scripts/augment_dataset_quantile_stats.py \
  --repo-id local/jarvis_train \
  --root /data/jarvis/datasets/train \
  --overwrite \
  --skip-images
```

这里 `--overwrite` 只用于刚生成的训练数据集统计；保留修改前的版本与转换记录。检查 `meta/stats.json` 中 state/action 的均值、标准差、`q01/q99`、维度和有限性。对 `q01==q99` 的维度，检查它是该任务中合理恒定的量，还是示范覆盖不足；数值防除零不能代替这项检查。该脚本是估计分位数，不是逐值排序的精确分位数。[L9]

官方也支持 `MEAN_STD`，可作为有记录的对照配置；但 z-score 不保证落在状态分箱范围内，大量越界会挤在边界桶。不要在部署时临时切换归一化方式，所有验证都加载所选 checkpoint 保存的处理器。

训练只读取 train 数据目录。虽然框架有数据集划分功能，但不能默认认为切出验证 episode 后归一化统计就自动只由训练子集重算。独立转换 train/val/test 更容易避免此问题。

### 9.2 200 步命令

确保 `/data/jarvis/datasets/train` 已通过第 7 节检查，运行目录仍是上面的 LeRobot 仓库：

```bash
uv run --no-sync lerobot-train \
  --policy.type=pi05 \
  --policy.pretrained_path=/data/jarvis/models/pi05_base \
  --policy.dtype=bfloat16 \
  --policy.device=cuda \
  --policy.gradient_checkpointing=true \
  --policy.push_to_hub=false \
  --policy.normalization_mapping='{"ACTION":"QUANTILES","STATE":"QUANTILES","VISUAL":"IDENTITY"}' \
  --peft.method_type=LORA \
  --dataset.repo_id=local/jarvis_train \
  --dataset.root=/data/jarvis/datasets/train \
  --batch_size=1 \
  --num_workers=2 \
  --steps=200 \
  --save_freq=100 \
  --env_eval_freq=0 \
  --trackio.enable=false \
  --output_dir=/data/jarvis/runs/pi05_smoke
```

`policy.type + pretrained_path` 表示选择策略类型并加载基础权重，从数据集确定本项目特征。不要再同时加 `policy.path`；它是另一种读取已有策略配置的入口。LoRA 默认目标层由这个版本的 π0.5 模型实现提供，首轮不随意改成未经核对的通用正则表达式。[L2][L4][L7]

这里没有配置机器人评估环境，也没有自动在独立 val 目录上选模型；`env_eval_freq=0` 仅禁用环境 rollout。需要另做下面的离线检查和受控真机验证。

试跑通过条件：输入是两路正确图像、state/action 是约定的 8 维，loss 有限、参与训练的参数确为 LoRA 预期部分，显存稳定，checkpoint 可重新加载，反归一化后的动作数值和单位合理。特别检查 state 是否因归一化和离散化被大量压到边界。

### 9.3 正式训练示例

试跑通过后，用新输出目录从同一基础权重开始正式实验。下例为 **10,000 步候选**，不是推荐所有任务都固定跑 10,000 步；batch 只能在实测余量允许时增加。

```bash
uv run --no-sync lerobot-train \
  --policy.type=pi05 \
  --policy.pretrained_path=/data/jarvis/models/pi05_base \
  --policy.dtype=bfloat16 \
  --policy.device=cuda \
  --policy.gradient_checkpointing=true \
  --policy.push_to_hub=false \
  --policy.normalization_mapping='{"ACTION":"QUANTILES","STATE":"QUANTILES","VISUAL":"IDENTITY"}' \
  --peft.method_type=LORA \
  --dataset.repo_id=local/jarvis_train \
  --dataset.root=/data/jarvis/datasets/train \
  --batch_size=1 \
  --num_workers=2 \
  --policy.scheduler_decay_steps=10000 \
  --steps=10000 \
  --save_freq=2000 \
  --env_eval_freq=0 \
  --trackio.enable=false \
  --output_dir=/data/jarvis/runs/pi05_run01
```

依据 warm-up 后的平均每步秒数估算时长：`步数 × 秒/步 ÷ 3600`，再加保存、加载和验证时间。先比较几个中间 checkpoint；loss 下降而留出任务表现变差时，要检查过拟合、动作语义和数据覆盖，不能仅继续加步数。

### 9.4 断点续训与备份

每个 checkpoint 应保留完整目录，包括 `pretrained_model/` 以及恢复训练用的 `training_state/`。LoRA 的模型部分通常是 adapter，不能因为它很小就认为自己已经得到一份独立完整的 π0.5。

续训要使用该 checkpoint 的 `train_config.json` 和 `resume` 入口，保持数据、特征、LoRA 和优化器定义一致。仅指定 adapter 作为预训练权重重新跑，属于另起训练，不是恢复优化器状态。输出目录、基础权重路径和数据目录换机器后都需核对。

例如正式训练在第 4,000 步存档后中断，可在原运行目录和数据路径仍有效的情况下恢复：

```bash
uv run --no-sync lerobot-train \
  --config_path=/data/jarvis/runs/pi05_run01/checkpoints/004000/pretrained_model/train_config.json \
  --resume=true
```

编号至少六位，例如 `000200`、`004000`、`010000`。已完成的 200 步试跑与正式新实验分开，正式训练不默认从试跑结束时的学习率和优化器状态接续。

训练结束或停机前保存：基础权重 revision、adapter、策略配置、前后处理器及训练统计、完整训练配置、优化器状态、数据划分清单、动作契约、运行日志、验证记录和代码 SHA。不要只拷一个 `last` 符号链接。

## 10. 怎样部署训练后的模型

### 10.1 必须带走的内容

| 内容 | 用途 |
| --- | --- |
| 原始 π0.5 base 与准确 revision | LoRA 需要的底座 |
| 选中的 `pretrained_model` 全目录 | adapter、策略配置、处理器与训练统计 |
| 分词器及所需缓存 | 任务文字和状态预处理 |
| 固定 LeRobot 环境与代码 SHA | 相同模型、处理器和 adapter 加载逻辑 |
| 动作契约、相机配置、Kinova 桥接程序 | 把模型数值对应到实际机器人 |

`adapter_config.json` 里的 `base_model_name_or_path` 可能保存了云端绝对路径。目标机器应提供可用的同版本 base 路径，必要时只修改部署副本中的路径并记录；不要误换成另一个模型。LeRobot 的策略工厂能识别 adapter 并加载底座，不能把 adapter 目录当完整 base 直接交给普通加载方式。[L7]

### 10.2 完整离线加载示例：不发送机器人动作

在训练环境中把下例保存为 `check_pi05_offline.py`，运行 `uv run --no-sync python check_pi05_offline.py`。默认读取 200 步试跑的存档，评估正式模型时修改 checkpoint 路径。该示例只验证模型加载与输出；先用 train 样本排查管线，选模型再读取独立 val 样本，处理器仍使用训练存档。

```python
from pathlib import Path
import json
import time

import torch

from lerobot.datasets.lerobot_dataset import LeRobotDataset
from lerobot.policies.pi05.configuration_pi05 import PI05Config
from lerobot.policies.factory import make_policy, make_pre_post_processors

# 在完成训练的 Linux GPU 环境中运行；所有路径按实际位置修改。
checkpoint = Path('/data/jarvis/runs/pi05_smoke/checkpoints/000200/pretrained_model')
train_root = Path('/data/jarvis/datasets/train')

if not torch.cuda.is_available():
    raise RuntimeError('此示例需要可用的 CUDA GPU。')
if not (checkpoint / 'adapter_config.json').is_file():
    raise FileNotFoundError('请填写 LoRA checkpoint 的 pretrained_model 目录。')
if not (train_root / 'meta' / 'info.json').is_file():
    raise FileNotFoundError('请先转换并验证本地 LeRobot 训练数据集。')

# base_model_name_or_path 必须指向原始训练所用的完整 base 权重。
# 换机器后若路径改变，先修正 adapter_config.json 的该字段，勿换成另一版本的 base。
with (checkpoint / 'adapter_config.json').open(encoding='utf-8') as f:
    adapter_cfg = json.load(f)
print('Base model:', adapter_cfg['base_model_name_or_path'])

# return_uint8=False 让数据集返回 CHW、float32、[0, 1] 的 RGB 图像。
dataset = LeRobotDataset(
    repo_id='local/jarvis_train',
    root=train_root,
    return_uint8=False,
)
cfg = PI05Config.from_pretrained(checkpoint)
cfg.device = 'cuda'
cfg.dtype = torch.bfloat16
if not cfg.use_peft:
    raise RuntimeError('此示例期望使用已训练的 LoRA 配置。')
if cfg.use_visual_memory or cfg.use_proprioceptive_memory:
    raise RuntimeError('此示例针对首版单帧配置；带历史记忆的模型需另外提供历史窗口。')

# factory 会读取 adapter_config.json，加载正确 base，再加载 LoRA adapter。
policy = make_policy(cfg=cfg, ds_meta=dataset.meta).eval()

# 加载训练时保存的归一化、tokenizer 和反归一化，不重算训练统计。
# tokenizer 首次使用可能需要联网及已接受 PaliGemma 许可；缓存齐全后才能断网。
preprocessor, postprocessor = make_pre_post_processors(
    policy_cfg=cfg,
    pretrained_path=checkpoint,
    preprocessor_overrides={'device_processor': {'device': 'cuda'}},
)

sample = dataset[0]
observation = {
    key: sample[key]
    for key in cfg.input_features
    if key.startswith('observation.')
}
observation['task'] = sample['task']
if observation['observation.state'].shape != (8,):
    raise ValueError('本示例要求 state = 7 个关节角 + 1 个夹爪状态。')

policy.reset()
batch = preprocessor(observation)
torch.cuda.synchronize()
start = time.perf_counter()
with torch.inference_mode():
    normalized_chunk = policy.predict_action_chunk(batch)
    # 完整 chunk 反归一化为训练数据约定的单位；此处不调用 Kortex、不下发动作。
    action_chunk = postprocessor(normalized_chunk).detach().cpu()
torch.cuda.synchronize()
elapsed = time.perf_counter() - start

expected = (1, cfg.chunk_size, 8)
if tuple(action_chunk.shape) != expected:
    raise ValueError(f'动作 shape 不符：{tuple(action_chunk.shape)}，期望 {expected}')
if not torch.isfinite(action_chunk).all():
    raise ValueError('动作包含 NaN 或 Inf。')

print('Task:', sample['task'])
print('Action chunk shape:', tuple(action_chunk.shape))
print('First action in dataset units:', action_chunk[0, 0].tolist())
print('Single cold inference seconds:', round(elapsed, 3))
print('此结果只证明能加载并生成数值；未证明动作正确、时序达标或适合真机执行。')
```

单次冷启动时间不代表稳定吞吐；在正式测速中预热、重复测量完整动作块预测，并记录 `torch.cuda.max_memory_allocated()` 等峰值及端到端延迟。模型数值正常也不能证明其动作可直接在机器人执行。

### 10.3 先离线，再接真实输入，最后控制机械臂

1. **离线检查：** 使用保存的真实样本，只生成动作、不连接运动接口。检查加载、形状、单位、反归一化、峰值显存和完整一次预测的延迟。
2. **实时输入检查：** 两路实时图像和状态输入模型，只显示建议动作，检查帧龄、关节跳变、夹爪方向和目标语义。
3. **受控短任务：** 在现场人员监督下，用已经验收的执行接口低速验证一个短技能；记录每次计划、实际发送、反馈和结果。
4. **重复验证：** 多种起始位置、少量留出布局，统计真实成功率、抓空、掉落、位置偏差、超时与人工介入。
5. **接入 Agent：** 只有已验证技能才开放给任务层调用；任务层以新观察核验完成，不以模型函数返回当作成功。

### 10.4 动作块如何执行

模型预测一段动作，不代表把所有目标瞬间发给机器人。按训练时的时间间隔安排目标，由已验证控制器处理跟踪、插值、速度/加速度和关节限制。默认 chunk 参数不是本台 Kinova 的安全执行时长。

动作块返回时检查从观测到返回的总年龄。超过该任务允许的时效则拒收，不能把已经过时的目标从第一步重新补播；允许的延迟和时序补偿必须与控制器共同验证。

例如 20 Hz 数据、50 步动作块对应 2.5 秒预测范围，不能不看新画面就默认全部执行。实际执行窗口取决于推理耗时和跟踪能力；若推理一次需要数百毫秒，却要求每 50 ms 完整重算，就需要调整调度或硬件，不能只调一个 fps 数字。使用缓存动作的 `select_action` 调用可能很快，测速应单独测完整动作块预测。

暂停、任务切换、场景变化、超时后，清空**模型内部动作缓存和执行端队列**。新任务使用新场景版本、时间戳和请求编号，拒绝旧请求的迟到结果；网络恢复后不能继续播放过期动作。时序复杂的异步或 RTC 方案放在同步方案正确之后验证。

### 10.5 游戏本放不下怎么办

游戏本继续负责相机、Kortex、记录、Agent 和界面；模型可部署在实验室大显存电脑，或在预算允许时放在云端。向模型端发送任务、图像和状态，返回动作块；机械臂控制连接留在现场电脑，由本地执行端验证和调度。

远端推理需要计入传输、排队和推理的完整延迟，并能拒绝过期结果、停止本地动作。租用 GPU 释放以后模型服务会停止，长期远端推理费用应单独安排；不能把 48 小时训练预算同时当作整个学期的在线算力。

## 11. 评估、失败排查与预算

### 11.1 怎样知道“训练成功了”

| 层次 | 要记录的证据 |
| --- | --- |
| 数据正确 | 同步回放、动作含义、夹爪事件、单位、划分和终态检查 |
| 软件正确 | 训练完成、checkpoint 可重载、离线动作有限且形状正确 |
| 技能有效 | 留出初始条件的实际取放成功率、落点误差、掉落、超时、介入次数 |
| 项目有效 | 局部编辑完成率、保护区误改次数、人工修改后接续能力 |

同一套动作数据可以训练模型，但不能再用同一批场景反复挑选后声称是独立测试。比较微调前后时固定相机、执行适配器、初始布局、Harness 和尝试预算。基础权重若离线检查就不适合本机动作，记录这一结果，不为凑基线强行上真机。

课程要求的“至少 20 条测试指令、10 次真机局部编辑”来自 [原始任务要求](原始任务要求.md)，不是训练示范数量，也不自动构成统计上充分的模型评估。

### 11.2 常见问题

| 现象 | 优先检查 |
| --- | --- |
| loss 降了但机器人不动 | action 是否误复制 state、目标时序是否错位、动作是否被适配器拒绝 |
| 动作突然大幅跳变 | 度/弧度、关节顺序、角度回绕、归一化、旧动作队列 |
| 抓住后又松开 | 夹爪反馈被误当目标、开合方向反了、命令事件对齐错误 |
| 看起来像“慢半拍” | RTSP 缓冲、图片与关节错时、推理延迟、错误执行频率 |
| 某个位置好，换位置失败 | 示范覆盖、目标可见依据、相机视野、语言与目标对应 |
| val 好，真机差 | 训练/部署预处理不同、状态统计泄露、控制器跟踪不同、评估场景太接近训练 |
| CUDA 显存不足 | 实际 batch、图片数、精度、是否启用 checkpointing、是否意外全量训练、其他 GPU 进程 |
| adapter 找不到基础模型 | `base_model_name_or_path` 指向旧机器路径，或 base revision 缺失 |
| 离线加载还请求网络 | tokenizer 缓存、processor 引用、基础模型路径未准备完整 |

### 11.3 约 48 小时 GPU 的分配

| 用途 | 上限建议 |
| --- | --- |
| 首次安装、加载、200 步与重载验证 | 4 小时 |
| 正式训练与中间 checkpoint 检查 | 24 小时 |
| 推理验证、必要补训 | 12 小时，共享额度 |
| 故障与重跑余量 | 8 小时 |

记录实际计费时间，包含安装、下载和空等。录制、清洗、补采、Agent 和界面开发时尽量释放训练 GPU。首次短试若发现动作标签、数据或环境错误，保存现场记录后停机修复，不自动耗完预算。

## 12. 接下来按什么顺序做

1. 检查现有取放程序：发关节位置、关节速度还是末端命令？是否能记录实际发出的命令？
2. 确认位置控制契约，优先记录所有原始反馈和命令；决定 A 或 B 标签方案，尤其确认夹爪目标。
3. 验收双相机、反馈时序和停止能力，录 5–10 条并回放。
4. 编写同步、重标注、转换和检查程序，冻结数据版本；此时即可并行开发 Agent。
5. 采窄任务数据，按完整 episode 分 train/val/test；先跑 200 步并重载。
6. 用实测速率安排正式训练，离线检查、受控真机验证后，再将技能接入 Agent。

**第一项真正需要的不是先搭完整 Agent，而是拿到现有控制程序，明确“它发了什么、我们录了什么、模型将来要输出什么”。**

## 版本与来源

本文的项目选择、数据契约、流程、预算和建议数量为本项目工程设计；官方 API、模型与框架能力以以下来源为据。代码片段为按接口编写的项目示例，不代表已完成安装、训练或真机测试。

- [K1：Kinova BaseCyclic 协议](https://github.com/Kinovarobotics/ros_kortex/blob/noetic-devel/kortex_driver/protos/BaseCyclic.proto)
- [K2：Kinova 夹爪循环反馈协议](https://github.com/Kinovarobotics/ros_kortex/blob/noetic-devel/kortex_driver/protos/GripperCyclicMessage.proto)
- [K3：Kinova Base 协议](https://github.com/Kinovarobotics/ros_kortex/blob/noetic-devel/kortex_driver/protos/Base.proto)
- [L1：固定版本 π0.5 文档](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/docs/source/pi05.mdx)
- [L2：固定版本 PEFT 文档](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/docs/source/peft_training.mdx)
- [L3：π0.5 配置与 LoRA 默认目标](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/policies/pi05/configuration_pi05.py)
- [L4：π0.5 模型、action chunk 与 loss](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/policies/pi05/modeling_pi05.py)
- [L5：LeRobotDataset 创建、写入与完成接口](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/datasets/lerobot_dataset.py)
- [L6：π0.5 前后处理器](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/policies/pi05/processor_pi05.py)
- [L7：策略工厂及 PEFT 加载](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/policies/factory.py)
- [M1：π0.5 基础权重文件与版本](https://huggingface.co/lerobot/pi05_base/tree/22c30e5b34f9e7ab738b1b34fc2fc5ecfe2d28f0)
- [O1：openpi 资源说明](https://github.com/Physical-Intelligence/openpi#requirements)
- [L8：固定版本安装说明](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/docs/source/installation.mdx)
- [L9：训练数据分位数统计脚本](https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/scripts/augment_dataset_quantile_stats.py)

[K1]: https://github.com/Kinovarobotics/ros_kortex/blob/noetic-devel/kortex_driver/protos/BaseCyclic.proto
[K2]: https://github.com/Kinovarobotics/ros_kortex/blob/noetic-devel/kortex_driver/protos/GripperCyclicMessage.proto
[K3]: https://github.com/Kinovarobotics/ros_kortex/blob/noetic-devel/kortex_driver/protos/Base.proto
[L1]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/docs/source/pi05.mdx
[L2]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/docs/source/peft_training.mdx
[L3]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/policies/pi05/configuration_pi05.py
[L4]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/policies/pi05/modeling_pi05.py
[L5]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/datasets/lerobot_dataset.py
[L6]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/policies/pi05/processor_pi05.py
[L7]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/policies/factory.py
[M1]: https://huggingface.co/lerobot/pi05_base/tree/22c30e5b34f9e7ab738b1b34fc2fc5ecfe2d28f0
[O1]: https://github.com/Physical-Intelligence/openpi#requirements
[L8]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/docs/source/installation.mdx
[L9]: https://github.com/huggingface/lerobot/blob/9e5ee9859480e425a97a6836bb3fee479539beb5/src/lerobot/scripts/augment_dataset_quantile_stats.py
