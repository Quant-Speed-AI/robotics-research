# XLeRobot 官方文档：0.4 双轮差速 / 12V 适配调研

核查日期：2026-10-07。用户已确认持有 XLeRobot 0.4、双轮差速、12V 套件。本次阅读官方文档与源码，未安装依赖、运行策略、连接机器人或验证实际动作。实际电机型号、供电输出、厂家校准与计算机配置仍待实物核对。

## 结论

官方资料可以作为主参考，但必须按章节选择对应版本。`latest` 首页仍标注 0.3.0，双轮装配正文已介绍 0.4.0，英文安装页已改用插件，遥操作与 ACT 部分仍保留旧流程。它不是一套版本完全同步、可以从头逐条复制的操作手册。[官方首页](https://xlerobot.readthedocs.io/en/latest/index.html)、[双轮装配](https://xlerobot.readthedocs.io/en/latest/hardware/getting_started/assemble_2wheel.html)、[英文安装](https://xlerobot.readthedocs.io/en/latest/software/getting_started/install.html)

适合这套设备的起点是双轮插件 `lerobot_robot_xlerobot_2wheels`，机器人类型 `xlerobot_2wheels`；先保留厂家配置、固定底盘打通单臂，再扩展双臂与差速底盘，之后做演示采集和 ACT 基线。SmolVLA、π0.5、LLM Agent、整机 RL 是后续选择，需要各自配套代码和验证，不能用模型名称替代设备兼容证据。[插件源码说明](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/plugins/README.md)

## 1. 官方章节阅读导航

| 章节 | 对当前设备的用途 | 核验判断 |
| --- | --- | --- |
| [双轮装配](https://xlerobot.readthedocs.io/en/latest/hardware/getting_started/assemble_2wheel.html) | 0.4 底盘和附加打印件 | 优先阅读；舵机与无刷路线需区分，页面仍在完善 |
| [BOM](https://xlerobot.readthedocs.io/en/latest/hardware/getting_started/material.html) / [传统装配](https://xlerobot.readthedocs.io/en/latest/hardware/getting_started/assemble.html) | 供电、控制板、接线与通用结构 | 有参考价值；17 电机及三轮 ID 属旧结构 |
| [安装](https://xlerobot.readthedocs.io/en/latest/software/getting_started/install.html) | LeRobot 加 XLeRobot 插件 | 采用对应底盘包；不要与旧复制文件流程混合 |
| [单臂示例](https://xlerobot.readthedocs.io/en/latest/software/getting_started/SO101.html) | 关节、末端、双臂与视觉跟随 | 路径以软件目录实际文件为准；入口链接需看当前站点导航 |
| [整机遥操作](https://xlerobot.readthedocs.io/en/latest/software/getting_started/XLeRobot_teleop.html) | 键盘、Xbox、Joy-Con、VR | 网页多处还是原三轮脚本，双轮应选专门示例 |
| [ACT](https://xlerobot.readthedocs.io/en/latest/software/getting_started/VLA_ACT.html) | 单臂/VR 采集、相机一致性 | 部署示例缺模型路径，不能直接作为策略执行命令 |
| [SmolVLA / ACT](https://xlerobot.readthedocs.io/en/latest/software/getting_started/VLA_smol.html) | 双臂示教、三相机、训练与评估 | 社区 fork 流程，不等于当前主线整机双轮教程 |
| [π0.5](https://xlerobot.readthedocs.io/en/latest/software/getting_started/VLA_pi05.html) | 双臂外部分支与数据集入口 | 是路线说明，尚非完整主线操作手册 |
| [LLM Agent](https://xlerobot.readthedocs.io/en/latest/software/getting_started/LLM_agent.html) | 视觉、语音、移动工具和策略调用 | RoboCrew 集成；示例包含横移工具，双轮需适配 |
| [RL](https://xlerobot.readthedocs.io/en/latest/software/getting_started/RL.html) | 单臂 HIL-SERL / sim2real 入口 | 页面明确整机官方 RL 代码仍待提供 |
| [树莓派](https://xlerobot.readthedocs.io/en/latest/software/getting_started/raspberry_pi_setup.html) | 无线整机主机、VNC、SSH | 可选扩展；旧复制配置与 LeKiwi 部分不能默认套用 |
| [仿真](https://xlerobot.readthedocs.io/en/latest/simulation/index.html) | 网页体验、MuJoCo、ManiSkill | 三条路线的底盘和系统要求不同 |

硬件与仿真的细节另见 [官方硬件与仿真核验](XLeRobot官方硬件与仿真核验_2026-10-07.md)。

## 2. 对 0.4 双轮 / 12V 最关键的配置

当前舵机双轮源码预期两条总线：port1 左臂 ID 1–6、头部 7–8；port2 右臂 ID 1–6、左右轮 9–10。推导本体共 16 台电机，不含示教 leader 臂。两条独立总线可各有 1–6；不能把总线合并后仍保留重复 ID。以上是代码定义，不是实物扫描结果。[双轮电机定义](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/src/robots/xlerobot_2wheels/xlerobot_2wheels.py#L78)

双轮配置默认串口 `/dev/ttyACM0`、`/dev/ttyACM1`，轮半径 0.05m、轮距 0.25m，摄像头字典为空。真实相机、端口及尺寸需填写；装配页的名义 5 英寸轮与代码半径默认值并不自动对应。`use_degrees=False` 的关节表示也需与示教数据和模型一致。底盘只有 `x.vel` 与 `theta.vel`；源码把转速参数按度/秒换算，不能当作弧度/秒输入。[双轮配置](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/src/robots/xlerobot_2wheels/config_xlerobot_2wheels.py)、[运动学实现](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/src/robots/xlerobot_2wheels/xlerobot_2wheels.py#L369)

12V 已由用户确认；仍要对照实际舵机、电源和控制板。旧 BOM 的供电搭配只是原配置参考。官方首页 600–1000g 单臂负载与此前 OneRobot 教程的 400g 建议口径不同，均缺乏足以迁移到这套设备的全姿态测试依据；应保留供应商对具体套件的限制并做任务实测，不能取较大的数字作为验收值。[官方 BOM](https://xlerobot.readthedocs.io/en/latest/hardware/getting_started/material.html)、[官方限制](https://xlerobot.readthedocs.io/en/latest/index.html#limitations)、[此前教程记录](XLeRobot双从教程_调研与避坑_2026-10-07.md)

## 3. 软件安装与版本不能混用

英文安装页与源码确认了插件路线：共享运动学包加双轮机器人包，VR 时再加入 VR teleoperator。下面只记录官方包入口，本次没有执行安装，也没有证明它们与任意 LeRobot 版本都兼容。

```bash
# 在已选定提交的 XLeRobot 仓库根目录
pip install -e software/plugins/xlerobot_model
pip install -e software/plugins/lerobot_robot_xlerobot_2wheels
# 仅选用 VR 时加入
pip install -e software/plugins/lerobot_teleoperator_xlerobot_vr
```

[插件 README](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/plugins/README.md)说明这些包由 LeRobot 自动发现。双轮插件版本是 **0.1.0**，这与硬件 **0.4** 是不同编号；其依赖声明 `lerobot[feetech]` 未固定版本，安装时可能随上游变化。[插件元数据](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/plugins/lerobot_robot_xlerobot_2wheels/pyproject.toml)

当前 LeRobot 安装页要求 Python ≥3.12、PyTorch ≥2.10，录制、训练与硬件依赖按功能分组。XLeRobot SmolVLA 页却创建 Python 3.10 并克隆 `kahowang/lerobot`：这是社区 fork 配套环境，不能直接拼接当前主线安装命令。[LeRobot 安装](https://huggingface.co/docs/lerobot/installation)、[SmolVLA 配套流程](https://xlerobot.readthedocs.io/en/latest/software/getting_started/VLA_smol.html)

进一步源码对照：该教程使用 `bi_so101_follower` / `bi_so101_leader`；当日 LeRobot 主线注册的是 `bi_so_follower` / `bi_so_leader`。这是接口版本差异，不应只替换一个字符串就宣称所有相机、子配置和动作映射都已适配。[主线 follower 配置](https://github.com/huggingface/lerobot/blob/ca69a2068462a37f7cdcb74180927a2f863d2bf7/src/lerobot/robots/bi_so_follower/config_bi_so_follower.py)、[主线 leader 配置](https://github.com/huggingface/lerobot/blob/ca69a2068462a37f7cdcb74180927a2f863d2bf7/src/lerobot/teleoperators/bi_so_leader/config_bi_so_leader.py)

## 4. 遥操作、校准与停止的实际边界

双轮对应示例是 `software/examples/4_xlerobot_2wheels_teleop_keyboard.py` 和 `7_xlerobot_2wheels_teleop_joycon.py`；不要默认用网页列出的无 `2wheels` 示例。Joy-Con 流程还涉及额外库和 Linux 蓝牙步骤，需按控制电脑的平台选择。[双轮键盘源码](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/examples/4_xlerobot_2wheels_teleop_keyboard.py)、[双轮 Joy-Con 源码](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/examples/7_xlerobot_2wheels_teleop_joycon.py)

`connect()` 并非纯只读：已有校准文件的默认恢复分支会向电机写校准；没有文件时可进入人工校准。厂家预配置与所用校准文件必须对应，连接前保留文件与设备 ID。默认 `max_relative_target=None`，未启用单次目标变化限制。[连接流程](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/src/robots/xlerobot_2wheels/xlerobot_2wheels.py#L172)、[配置默认值](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/src/robots/xlerobot_2wheels/config_xlerobot_2wheels.py)

源码静态疑点：限幅分支以带 `.pos` 的动作键查 `Present_Position` 读回的电机名键，存在键格式不一致，需在接实机前核验；不能仅设置参数就认定限幅有效。主机 watchdog 默认 500ms 后调用 `stop_base()`，代码目标是停止底盘，不能解释为所有手臂都有硬件急停。本次未执行这两条路径。[限幅实现](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/src/robots/xlerobot_2wheels/xlerobot_2wheels.py#L572)、[主机 watchdog](https://github.com/Vector-Wangel/XLeRobot/blob/b017b5e6354bd9f61f4247a920c72622ca0aade0/software/src/robots/xlerobot_2wheels/xlerobot_2wheels_host.py#L96)

## 5. 从采集到自主策略的判断

官方 ACT 页保留旧复制 `record.py` / VR 文件流程，整机类型仍为 `xlerobot`；其“Deploy a Model”命令依然包含 VR teleop，却未指定模型路径。这解释了飞书中相同问题的官方来源：问题不能只归因于供应商改写。[ACT 章节](https://xlerobot.readthedocs.io/en/latest/software/getting_started/VLA_ACT.html#deploy-a-model)

SmolVLA/ACT 页给出的评估示例包含 `--policy.path`，但它录制的是双臂 SO-101 工作流。仅双臂的 12 维动作与双轮整机的头部、底盘动作不同，模型与数据必须使用相同定义。页面约 20 段演示学任务是社区演示描述，不是对你的任务成功率的承诺。[SmolVLA/ACT 章节](https://xlerobot.readthedocs.io/en/latest/software/getting_started/VLA_smol.html)

当前 LeRobot 主线提供 `lerobot-rollout`，要求真实 checkpoint/Hub 模型路径；可选录制评估和 RTC 等策略。但这只能确认主线部署接口存在，不能替代 XLeRobot 双轮插件的端到端兼容测试。[当前部署说明](https://huggingface.co/docs/lerobot/inference)

π0.5 页面链接 `xuweiwu/XLeRobot`、`xuweiwu/openpi` 和双臂数据集，并说明部分工作仍在外部分支。适合后续专门复现，暂不作为第一条通电链路。[π0.5 页面](https://xlerobot.readthedocs.io/en/latest/software/getting_started/VLA_pi05.html)

LLM Agent 页面使用 RoboCrew，通过摄像头、语音和已训练的 VLA 工具完成动作；全文示例包含侧移工具，双轮不能照搬。需要先完成可重复执行的本地动作/策略，再把它们提供给上层 Agent。[Agent 章节](https://xlerobot.readthedocs.io/en/latest/software/getting_started/LLM_agent.html)

## 6. Mac、仿真和绘画任务

Mac 可用于阅读、开发和网页仿真体验；当前 LeRobot 支持 `mps` 设备，但是否满足 ACT 的速度与内存仍需具体机型和数据验证。Linux 的串口、udev、apt 和 Joy-Con 服务命令不能原样复制到 Mac。[LeRobot 安装](https://huggingface.co/docs/lerobot/installation)、[部署设备选项](https://huggingface.co/docs/lerobot/inference)

官方网页 MuJoCo-GS-Web 的 XLeRobot 底盘声明为 2 DOF，更接近双轮控制；旧本地 MuJoCo 示例仍含横移，ManiSkill 使用虚拟平面关节。三者不能直接视作已核验的用户套件数字孪生。本次未运行网页或本地仿真。[官方仿真入口](https://xlerobot.readthedocs.io/en/latest/simulation/index.html)、[网页模型说明](https://github.com/Vector-Wangel/MuJoCo-GS-Web#-supported-robots--keyboard-control)、[详细硬件与仿真报告](XLeRobot官方硬件与仿真核验_2026-10-07.md)

对此前绘画目标，官方文档提供运动学和末端控制入口，但还需笔夹、纸面坐标标定、抬落笔、接触控制和轨迹质量验收。这是对工程缺口的判断；没有找到可以直接在该套件上运行的完整官方绘画操作章。

## 7. 建议下一步及通过标准

| 顺序 | 实施内容 | 完成依据 |
| --- | --- | --- |
| 1 | 核对12V电源、电机、两板总线、原ID与厂家校准 | 清单与实际设备对应，原配置有备份 |
| 2 | 固定 LeRobot 与 XLeRobot 提交，先验证插件导入与注册 | 无硬件动作的导入/配置检查通过，记录依赖版本 |
| 3 | 固定底盘，先单臂、再双臂低幅遥操作 | 方向、范围、两臂独立性、退出与停止有观察记录 |
| 4 | 离地核查轮方向，填实际轮半径/轮距，再地面差速移动 | 前后、转向与停止正确，无横移命令误用 |
| 5 | 先少量采集检查相机和动作同步，再训练 ACT | 独立评估记录成功率、耗时、失败与人工介入 |
| 6 | 按任务添加绘画轨迹、SmolVLA、π0.5或 Agent | 每次扩展保持原基线可复现，并有任务结果证据 |

上述均为后续实施建议。本轮没有达到装配、通电、策略执行或实机验收阶段。

## 8. 可复核的源码快照

- XLeRobot `main`：`b017b5e6354bd9f61f4247a920c72622ca0aade0`。
- LeRobot `main`：`ca69a2068462a37f7cdcb74180927a2f863d2bf7`。
- 静态代码核验不等于这些提交已经组合运行通过；动态 `latest` 文档仍可能继续变化。

