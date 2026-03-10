# RoboCasa 仓库导读（中文）

这份文档面向第一次接触 RoboCasa 的同学，用“从 0 到跑起来 + 从代码到场景”的方式说明三件事：
1. 虚拟环境（依赖环境）是怎么搭起来的。
2. 仓库里各个核心目录分别负责什么。
3. **训练用仿真场景（以厨房场景为例）是如何一步步生成出来的。**

---

## 1) 虚拟环境是怎么搭起来的

RoboCasa 官方推荐使用 **Conda** 建立独立环境，然后在这个环境里用 `pip` 安装两个关键 Python 包：
- `robosuite`：底层仿真框架（机器人、控制器、与 MuJoCo 对接）
- `robocasa`：任务、场景、资产与工具层

### Step A. 用 conda 建隔离环境

```bash
conda create -c conda-forge -n robocasa python=3.11
conda activate robocasa
```

这样做的目的：
- 把 RoboCasa 的依赖和系统 Python 隔离，避免版本冲突。
- 固定 Python 版本（文档给的是 3.11），减少兼容性问题。

### Step B. 先装 robosuite（而且是 master 分支）

```bash
git clone https://github.com/ARISE-Initiative/robosuite
cd robosuite
pip install -e .
```

这里 `-e` 是 editable 模式：
- 源码改动会立即生效，开发和调试更方便。
- RoboCasa 依赖 robosuite 提供环境底座（控制、仿真接口等）。

### Step C. 再装 robocasa 本体

```bash
cd ..
git clone https://github.com/robocasa/robocasa
cd robocasa
pip install -e .
```

这一步会读取 `setup.py` 里的 `install_requires`，自动安装 RoboCasa 需要的 Python 依赖（如 `numpy`、`mujoco`、`gymnasium`、`h5py` 等）。

### Step D. 初始化运行时配置 + 下载资产

```bash
python -m robocasa.scripts.setup_macros
python -m robocasa.scripts.download_kitchen_assets
```

这两步是 RoboCasa “真正可运行”的关键：
- `setup_macros` 会把 `robocasa/macros.py` 复制成 `robocasa/macros_private.py`，用于保存你本机私有配置（不进 Git）。
- `download_kitchen_assets` 会下载厨房资产（体量很大，约 10GB），没有资产很多任务无法渲染或初始化。

### Step E. 虚拟环境最终长什么样

可以把完整结构理解成：
- **Conda 层**：`python=3.11` + 一些底层二进制依赖。
- **Pip 包层**：`robosuite` + `robocasa`（editable 安装）+ `setup.py` 声明的依赖。
- **运行时配置层**：`macros_private.py` 里的本机配置（例如 SpaceMouse 产品 ID、数据集路径）。
- **外部数据层**：厨房资产与（可选）数据集文件。

---

## 2) 各文件夹作用（从“最常用”角度）

下面先看仓库顶层，再看 `robocasa/` 包内部。

## 顶层目录

- `README.md`：安装、快速开始、演示入口说明。
- `setup.py`：Python 打包入口，声明依赖和包信息。
- `requirements.txt`：开发环境简化依赖（当前主要是 `-e .`）。
- `docs/`：Sphinx 文档源码（介绍、任务、数据集、基准、API 说明等）。
- `tests/`：测试脚本（任务有效性、布局、数据集播放、确定性与速度等）。
- `robocasa/`：核心 Python 包（环境、模型、脚本、工具、wrapper）。

## `robocasa/` 核心包

### 1. `robocasa/environments/`
环境定义层，核心在 `kitchen/`：
- `kitchen.py`：厨房任务基类。
- `atomic/`：原子任务（单步/短链条能力）。
- `composite/`：复合任务（多阶段任务链）。

可以理解为：这里定义“机器人要做什么任务、成功条件是什么、场景如何重置”。

### 2. `robocasa/models/`
资产与场景建模层：
- `objects/`：物体定义与注册。
- `fixtures/`：固定装置（柜子、水槽、炉灶等）定义。
- `scenes/`：厨房场景构建逻辑（布局 + 风格 + 组合规则）。
- `assets/`：原始资产文件（对象、fixture、scene 蓝图等）。

可以理解为：这里决定“世界长什么样”。

### 3. `robocasa/demos/`
交互演示脚本：看任务样例、看厨房布局和风格、看物体库、键盘/设备遥操作。

### 4. `robocasa/scripts/`
工具脚本集合：资产下载、数据集下载、示范采集、数据回放/转换、资产处理。

### 5. `robocasa/utils/`
底层公共工具：环境构建、采样、数据集注册、格式桥接。

### 6. `robocasa/wrappers/`
环境包装层：对接 Gym/Gymnasium，方便 RL / 评测统一接入。

---

## 3) 重点：一个“训练用厨房仿真场景”是怎样生成的？

下面讲的是**从你调用 `gym.make(...)` 到拿到一个可训练场景**的核心流水线。

## 总览（先记住这 8 步）

1. 选择任务类（例如 `PickPlaceCounterToCabinet`）
2. 决定布局 / 风格候选集合（layout / style）
3. 在 reset / load 时采样一个 `(layout_id, style_id)`
4. 根据 layout YAML + style YAML 构建 `KitchenArena`（fixture 级别）
5. 进行 fixture 摆放、相对对齐、组旋转、辅助件绑定
6. 根据任务定义采样 object（目标物 + 干扰物）并放到 fixture 指定区域
7. 设定机器人初始位姿并做短暂物理“稳定步进”
8. 进入训练循环：`obs -> action -> step -> success/reward`

如果把它类比成拍电影：
- layout/style 决定“片场装修风格”
- fixture 决定“厨房固定家具如何摆”
- object cfg 决定“道具放哪、放几个、哪些是目标”
- 任务类决定“演员（机器人）该完成什么剧情”

## 第 0 层：入口参数如何影响后续场景

在 `create_env(...)` 中，`split` 会被翻译成对象实例划分与 layout/style 的候选范围：
- `pretrain`：layout/style 走 train 组
- `target`：固定到 1~10 的 target 组合
- `all`：全量组合

也就是说，**训练集 / 目标集的“场景分布”在入口就已经控制住了**。

## 第 1 层：任务类决定“需要哪些场景元素”

以 `PickPlaceCounterToCabinet` 为例：
- 在 `_setup_kitchen_references()` 里，它声明自己需要一个 cabinet 和一个靠近该 cabinet 的 counter。
- 在 `_get_obj_cfgs()` 里，它定义了目标物体 `obj` 和两个干扰物体的采样与放置区域。
- 在 `_check_success()` 里，它定义成功判据（物体进柜子 + 夹爪远离物体）。

这一步可以理解为：任务类写下“场景需求规格说明书”。

## 第 2 层：`Kitchen._setup_model()` 选择 layout/style 并搭骨架

在 `Kitchen._setup_model()` 里做几件关键事情：
- 从 episode meta 或候选池中确定 `layout_id` 和 `style_id`。
- 创建 `KitchenArena(layout_id, style_id, ...)`。
- 从 arena 里拿到 fixture 配置，构造 `ManipulationTask`（机器人 + 场景 + fixture）。

这一步结束后，场景有了“固定装置骨架”，但还没把任务物体放好。

## 第 3 层：`KitchenArena` 读取 YAML 蓝图并创建 fixture

`KitchenArena` 会：
1. 读取 layout yaml（空间拓扑与 fixture 分组）
2. 读取 style yaml（材质、样式、fixture 默认配置选择）
3. 调用 `create_fixtures(...)` 生成 fixture 对象集合

此外，还会处理：
- `enable_fixtures`：把默认关闭的 fixture 打开
- `update_fxtr_cfg_dict`：覆盖某些 fixture 配置
- `clutter_mode`：是否启用 clutter fixture

## 第 4 层：`create_fixtures` 做几何装配（最关键）

`create_fixtures(...)` 的核心逻辑是“把 YAML 变成可放入 MuJoCo 的真实对象”：

- 语法检查（`check_syntax`）
  - 例如相对对齐时必须同时给 `align_to` 和 `side`。
- 样式注入（`load_style_config`）
  - 根据 fixture 类型去 `fixture_registry/*.yaml` 找默认配置并合并。
- 初始化 fixture 实例（`initialize_fixture`）
  - 把 `type` 映射到具体 fixture 类并创建对象。
- 位置解算
  - 支持绝对位置、相对对齐（`get_relative_position`）、堆叠（`stack_on`）。
- 组级变换
  - 使用 `group_origin` / `group_pos` / `group_z_rot` 对一组 fixture 统一旋转平移。
- 个体旋转
  - 支持每个 fixture 的 `z_rot` 增量。
- 主-辅 fixture 绑定
  - 如主设备和辅助件之间建立引用关系。

这一步完成后，你可以把它看成“厨房硬装已经搭完”。

## 第 5 层：`Kitchen._load_model()` 里放 fixture、再放 object

在 `_load_model()` 中，RoboCasa 会：
- 先完成 fixture 的采样摆放（包括 base-auxiliary 特殊对）。
- 调用 `_setup_kitchen_references()` 建立任务引用（如 `self.cab`、`self.counter`）。
- 调用 `_create_objects()` 根据 `_get_obj_cfgs()` 创建任务物体。
- 构建 object placement initializer，并采样 object 位置。
- 若某一步采样失败（`PlacementError`），会销毁并重试 `_load_model()`（最多 50 次）。

这套“失败重试”机制保证了最终环境可用，不会把明显不合法的摆放交给训练。

## 第 6 层：object 是怎么被采样并放进去的

object 配置由任务给出（`_get_obj_cfgs()`），每条通常包含：
- 从哪个对象组抽样（`obj_groups`）
- 是否必须可抓、可洗、可微波等属性约束
- 放到哪个 fixture（`placement.fixture`）
- 在 fixture 的哪个区域（`size`、`pos`、`offset`、`sample_region_kwargs`）

`sample_object(...)` 最终调用 `sample_kitchen_object(...)`，结合 split、registry、尺寸约束等完成抽样。

## 第 7 层：reset 阶段把“静态场景”变成“可交互场景”

`_reset_internal()` 会：
- 调用任务的 `_setup_scene()`（例如开柜门、开灶台等任务前置状态）。
- 将 object placement 写入仿真关节位姿。
- 设置机器人底座位置（可从 meta 恢复，或按 anchor 采样）。
- 执行若干空动作步进，让物体稳定（settle）。

到这里，训练所需的一个 episode 初始场景才算真正完成。

## 第 8 层：进入训练循环

当你开始 `env.step(action)`：
- wrapper 会把动作字典映射成底层控制向量。
- 场景按 MuJoCo 物理推进。
- 任务类的 `_check_success()` 计算成功与奖励（在 gym wrapper 中是稀疏奖励 0/1）。

因此，训练过程中的“场景多样性”主要来自：
- layout/style 采样
- object 组与实例采样
- 同一 fixture 内的随机放置
- 机器人初始位姿扰动

---

## 4) 用一个具体例子串起来：`PickPlaceCounterToCabinet`

你调用：

```python
env = gym.make("robocasa/PickPlaceCounterToCabinet", split="pretrain", seed=0)
obs, info = env.reset()
```

背后大致发生的是：

1. `split="pretrain"` 把 layout/style 和 object split 约束到 pretrain 范围。
2. `Kitchen` 从候选池随机一个 `(layout_id, style_id)`。
3. `KitchenArena` 读取该 layout + style 的 YAML，创建 fixture。
4. `PickPlaceCounterToCabinet` 注册 `cab` 和 `counter` 引用。
5. 该任务的 `_get_obj_cfgs()` 定义目标物与干扰物放置规则。
6. placement sampler 给出每个 object 的具体位姿。
7. reset 时开柜门，把 object/robot 放到位，做稳定步进。
8. 你看到的就是一个“可训练 episode 起点场景”。

---

## 5) 一句话总结

RoboCasa 场景不是“一次性加载一个大地图”，而是：
**任务逻辑（要做什么） + layout/style 蓝图（厨房长什么样） + 采样器（每次具体怎么摆）** 在 reset 阶段动态合成出的可训练仿真场景。
