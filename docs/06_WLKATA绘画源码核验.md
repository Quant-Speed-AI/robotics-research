# WLKATA AI Drawing Extension：绘画源码核验及 SO-101 移植参考

核验时间：2026-10-02，Asia/Shanghai。方法：读取 WLKATA 官方仓库 README、`drawing.py`、依赖文件及官方 SDK 文档。本轮属于静态源码审阅，未连接或驱动机械臂，未验证实画质量。读取的是当时 `main` 页面，未固定 commit。

## 结论

**这个项目适合参考“线稿变路径、标定纸面、机械臂落笔”的工程分层。它没有在已审阅的绘画主流程中实现 GPT 自动生成线稿的 API 集成，也没有实现可直接驱动 SO-101 的适配。**

README 的自拍流程要求操作者在 GPT/图生图工具中生成简洁线稿，下载 PNG，再交给绘画模块。GPT 的作用在图片准备阶段；仓库主流程随后处理本地图片。不能把这个仓库当成“GPT 已接管机械臂绘画”的完成品。README 列出的实测硬件是 MT4 或 Mirobot。[官方 README](https://github.com/wlkata/WLKATA-AI-Drawing-Extension#example-2--selfie-to-sketch-drawing)

## 源码实际做了什么

已读文件：[drawing.py](https://github.com/wlkata/WLKATA-AI-Drawing-Extension/blob/main/src/wlkatapython_extensions/drawing/drawing.py)。

| 环节 | 源码证据 |
|---|---|
| 图像转路径 | `edges_from_image()`：灰度图、可选高斯模糊、反向二值化、Canny、`findContours`、`approxPolyDP` |
| 点与版面 | `extract_points_groups()` 提取点；`invert_point_groups()` 翻转/旋转；`resize_point_groups()` 缩放进设定矩形 |
| 默认范围 | X 为 200–300、Y 为 -50–50；矩形为 100×100 坐标单位，不能从 README 的 A4 描述推导为默认整张 A4 |
| 纸面标定 | `cali_z()` 回零后，让人输入下降步长确定一个 Z；未见三点平面拟合或压力闭环 |
| 开始绘画 | `pre_drawing()` 显示 Matplotlib 路径预览、用户确认、巡游边框后再确认 |
| 下发执行 | `draw_points()` 将路径点转成 WLKATA `M20 G90 G00 X…Y…Z…`，用 `sendMsg()` 发送；轮廓间抬笔，轮询 Idle |

这段代码自己没有实现机械臂逆运动学和关节时间参数化，依赖厂商控制器执行笛卡尔命令。其图片处理是边缘轮廓追踪，没有看到线条中心骨架提取、笔画语义规划、最短空行程排序或笔尖受力反馈。

## 会影响绘画结果的源码细节

静态审阅发现以下事项，均来自上述 `drawing.py`，尚未做实机复现：

- 点数过滤会删除较短轮廓：边段少于 20 被略过，提取点少于 50 又被略过；复杂轮廓还有截断和抽样。这会影响简单图形、小字和面部细节，不能直接沿用为通用绘图预处理。
- Canny 默认保留内外轮廓，黑色粗线可能变成双边线；这是算法选择带来的工程风险，需要以预览核对原图。
- `KeyboardInterrupt` 分支的急停处理仍为 TODO；不能把终端 Ctrl+C 等同于已确认停止机械运动。

据此，第一次复用时应先通过方形、圆、短线、字母和头像的离线路径对比验证，再做抬笔空跑。预处理的目标应是保住原始线稿意图，并限制总路径长度与抬笔次数，而不是尽可能提取所有像素边缘。

## SDK、依赖和许可证

扩展依赖 `wlkatapython`、OpenCV、Matplotlib、pyserial、tqdm。`pyproject.toml` 要求 Python ≥3.12，README 仍写 Python 3.8+；部署环境应以已验证的依赖组合为准。[pyproject.toml](https://github.com/wlkata/WLKATA-AI-Drawing-Extension/blob/main/pyproject.toml)、[requirements.txt](https://github.com/wlkata/WLKATA-AI-Drawing-Extension/blob/main/requirements.txt)

厂商当前 `wlkatapython` SDK 使用 UART/RS485 串口与 G-code，支持多种 WLKATA 本体，标 MIT；官方说明部分功能需要多功能控制器。[当前官方 SDK](https://github.com/wlkata/WLKATA-Python-SDK-wlkatapython)

旧 `mirobot-py` 仓库明确停止维护并指向该新 SDK，因此新工程不应只根据旧教程选择包。[旧 SDK 状态](https://github.com/wlkata/mirobot-py)

**绘画扩展与底层 SDK 的许可证不同。** GitHub 为绘画扩展标识 GPL-3.0，SDK 标 MIT。本次扩展 `LICENSE` 正文抓取失败，因此已确认仓库标识，未完成许可条文核查；不能把 SDK 的 MIT 许可套用到扩展源码。[扩展仓库及 LICENSE 入口](https://github.com/wlkata/WLKATA-AI-Drawing-Extension)

## 相对 SO-101 的移植难点

SO-101 官方关节表是肩部旋转、肩抬升、肘、腕俯仰、腕滚转和夹爪；夹爪开合不提供额外末端姿态自由度。其原生控制链是 Feetech 总线舵机及标定，不是 WLKATA 的笛卡尔 G-code 接口。[SO-101 官方装配与标定](https://huggingface.co/docs/lerobot/so101)

以下是面向新工程的建议设计，并非 WLKATA 已实现的能力：

1. **笔尖坐标**：建立纸面坐标系和笔尖工具偏置，使用纸面三点拟合或可靠水平工装。换笔、换笔架后必须重新测量。
2. **运动适配**：把二维笔画与抬/落笔状态转换成三维目标，经过 SO-101 逆运动学、关节限位检查、连续解选择，再以受控速度下发关节轨迹。禁止把 WLKATA 的 XYZ 字符串直接发给 SO-101 舵机。
3. **几何可达性**：先在固定台面验证一个小画幅，记录笔尖角度与可达区域。五个运动关节通常不能让位置与三个姿态角全部独立自由设定，应只约束绘画所需的姿态。
4. **纸面接触**：加可滑动/弹性的笔架补偿纸张起伏与机械误差，先验证接触深度，再考虑压力传感。仅把笔夹进普通夹爪，无法保证整张纸线宽一致。
5. **轨迹质量**：把图片转为中心线/矢量笔画，做去噪、几何简化、弧长采样和笔画排序。不要照搬当前以点数删除和截断轮廓的逻辑。
6. **执行与恢复**：明确连接、标定、预览、抬笔空跑、绘画、暂停、撤笔、完成状态；记录已完成笔画，失联时由确定性控制层处理。

建议初始画幅约 60–80 mm 方形、单色低压力笔，以线条简单的图案验证；该范围是工程起步目标，未对 SO-101 实测。验收先看闭合误差、重复绘制偏移、抬笔是否留痕、跨画幅接触是否稳定，再讨论肖像速度和收费。

## “GPT＋SO-101 画画”的合理分工

```text
用户描述/照片
  → 已确认可用的生成模型或人工制作线稿
  → SVG/中心线笔画 + 用户预览
  → 确定性的版面/路径/抬笔规划
  → 纸面坐标变换 + SO-101 逆运动学
  → 限位/速度检查 + 舵机执行
  → 实画结果检查
```

生成模型负责画什么、风格和内容；机器人程序负责路径是否可达、笔尖在哪里、何时抬笔、如何停机。若用户说“GPT6”，本核验不推定任何特定公开 API 型号或图像能力，实际集成时由主方案核对可调用模型。画画这类确定路径任务的首版不必先做模仿学习或 VLA 训练。

WLKATA 还提供独立的官方机械臂 MCP，列有串口连接、运动、停止及原始命令工具，但这是另一仓库，不是绘画扩展已经集成了图像生成，也不适配 SO-101。[WLKATA MCP](https://github.com/wlkata/wlkata_arm_MCP)

## 尚未验证

尚未验证：GPT/图像模型端到端 API、SO-101 实机画幅与精度、笔架结构、连续工作温升、一次画完的成功率、模型费用与整图耗时、商业展示付费意愿。本文件提供的是可实施结构和源码缺口，不是已完成工程或商业运行证明。
