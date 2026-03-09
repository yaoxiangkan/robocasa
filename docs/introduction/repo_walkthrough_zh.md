# RoboCasa 仓库导读（中文）

这份文档面向第一次接触 RoboCasa 的同学，用“从 0 到跑起来”的方式说明两件事：
1. 虚拟环境（依赖环境）是怎么搭起来的。
2. 仓库里各个核心目录分别负责什么。

## 1) 虚拟环境是怎么搭起来的

RoboCasa 官方推荐使用 **Conda** 建立独立环境，然后在这个环境里用 `pip` 安装两个关键 Python 包：`robosuite`（底层仿真框架）和 `robocasa`（任务、场景、资产与工具）。

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

这一步会读取 `setup.py` 里的 `install_requires`，自动装 RoboCasa 需要的 Python 依赖（如 `numpy`、`mujoco`、`gymnasium`、`h5py` 等）。

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

交互演示脚本：
- 看任务样例
- 看厨房布局和风格
- 看物体库
- 键盘/设备遥操作

适合做“验证安装是否成功”和“快速直观了解能力边界”。

### 4. `robocasa/scripts/`

工具脚本集合：
- 资产下载（`download_kitchen_assets.py`）
- 数据集下载（`download_datasets.py`）
- 采集示范（`collect_demos.py`）
- 数据集回放/转换（`dataset_scripts/`）
- 资产处理工具（`asset_scripts/`）

适合数据生产、资产预处理、批处理流程。

### 5. `robocasa/utils/`

底层公共工具：
- 数据集注册和访问
- 环境辅助函数
- 与其他训练/数据格式工具链的桥接（如 robomimic、USD、GR00T 相关工具）

可以理解为“胶水层”：给上层任务和脚本提供复用能力。

### 6. `robocasa/wrappers/`

环境包装层：
- 对接 Gym/Gymnasium 的接口
- 额外渲染或边界处理 wrapper

作用是把 RoboCasa 环境适配到更通用的 RL / 评测工作流中。

---

## 3) 一句话理解 RoboCasa 的运行链路

“先用 Conda 建 Python 隔离环境 → 装 robosuite 底座 → editable 安装 robocasa → 生成私有宏配置 + 下载资产 → 通过 demos / gym wrapper / scripts 去跑任务、采数据、做评测。”

如果你后续愿意，我还可以继续给你做一版“开发者视角导图”：
- 从 `gym.make(...)` 到任务 reset/step 的调用栈。
- 一个 composite task 从场景采样到成功判定的数据流。
- 你要新增一个任务时，最小改动路径是什么。
