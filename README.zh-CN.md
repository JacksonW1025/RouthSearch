# RouthSearch
[English README](README.md)

## 项目简介
RouthSearch 是一个用于确定无人机在不同飞行模式下三维 PID 参数有效范围的工具。该工具由两个模块组成：**边界识别模块**和**异常行为验证模块**。

边界识别模块首先读取无人机官方文档中的 PID 参数及其取值范围，然后在 PID 参数空间中高效搜索有效配置与无效配置之间的分类边界。该模块的实现位于 `evaluation_script/`。

在搜索过程中，边界识别模块会持续调用异常行为验证模块，以判断某一组 PID 参数配置是否有效。该模块的实现位于 `routh_search/`。

## 项目目的与论文关系
这个仓库是论文 **RouthSearch: Inferring PID Parameter Specification for Flight Control Program by Coordinate Search** 的实现工件。

RouthSearch 的目标不是找出单个“最优” PID 组合，而是为特定飞行模式推断一个**三维 PID 有效边界**，从而在飞行过程中拦截可能导致振荡、偏航、失控或坠毁的危险 PID 配置。

从论文方法上看，RouthSearch 结合了三部分：

1. 使用 **Routh-Hurwitz 稳定性判据** 推导理论 PID 边界。
2. 使用 **坐标搜索（coordinate search）** 在真实飞控与仿真环境中修正边界。
3. 使用面向具体飞行任务的 **oracle** 判断某一组 PID 配置是有效还是无效。

论文方法与代码目录的对应关系如下：

* `evaluation_script/`：边界识别与 fuzzing 脚本，用于搜索 PID 参数空间。
* `routh_search/`：飞行任务驱动脚本与 oracle 实现，用于验证每一组被测试的 PID 配置。
* `ardupilot_default_data/`：实验使用的默认参数数据。

如果你想看论文的中文解读，可参考 `RouthSearch_完整中文翻译与细致解析.md`。

## 环境要求
1. 语言：
   * Python（Python 3.10）
   * Bash

2. 飞控程序：
   * [Ardupilot](https://ardupilot.org/)
   * [PX4](https://px4.io/)

3. 模拟器：
   * [Ardupilot SITL Simulator](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html)
   * [jMAVSim](https://docs.px4.io/main/en/sim_jmavsim/)

### 安装说明
#### Ardupilot
按官方文档安装 Ardupilot：[Ardupilot Setup](https://ardupilot.org/dev/docs/building-setup-linux.html)

#### Ardupilot SITL
按官方文档安装和使用 Ardupilot SITL：[Ardupilot SITL setup](https://ardupilot.org/dev/docs/copter-sitl-mavproxy-tutorial.html)

#### PX4
按官方文档安装 PX4：[PX4 Setup](https://docs.px4.io/main/en/dev_setup/building_px4.html)

#### jMAVSim
按官方文档配置 jMAVSim：[jMAVSim Setup](https://docs.px4.io/main/en/sim_jmavsim/)

## 项目结构
```text
RouthSearch/
|-- ardupilot_default_data/              # 默认参数
|-- evaluation_script/                   # 评估/搜索脚本
|-- routh_search/                        # 飞行任务与 oracle 代码
|-- README.md                            # 英文说明
|-- README.zh-CN.md                      # 中文说明
```

各目录说明如下：

* **`ardupilot_default_data/`**：保存 ArduPilot 的默认参数数据。
* **`evaluation_script/`**：保存论文实验使用的 Bash 搜索脚本。
* **`routh_search/`**：保存具体飞行模式任务和判断飞行结果是否异常的 oracle。

## 如何使用这个仓库
这个仓库有两种主要使用方式。

### 1. 直接运行原生任务脚本和 oracle
这是最适合理解仓库结构、验证环境是否正常、或者手工测试单组 PID 参数的方式。

典型流程如下：

1. 启动对应的模拟器与飞控程序。
2. 运行 `routh_search/` 下的某个任务脚本。
3. 对生成的飞行日志运行匹配的 oracle 脚本。

示例：

```bash
python3 -m compileall routh_search

# ArduPilot 示例
python3 routh_search/ardupilot/start_ardupilot.py
python3 routh_search/ardupilot/rtl_mode.py <config.json>
python3 routh_search/ardupilot/rtl_mode_oracle.py <bin_log_path>

# PX4 示例
# 需要先单独启动 PX4 SITL，例如在 PX4 仓库中运行 jMAVSim。
python3 routh_search/px4/px4_hold_mode.py
python3 routh_search/px4/px4_hold_mode_oracle.py <ulg_log_path>
```

仓库中已有的任务与 oracle 包括：

* ArduPilot 任务：`brake_mode.py`、`circle_mode.py`、`rtl_mode.py`、`zigzag_mode.py`
* ArduPilot oracle：`routh_search/ardupilot/` 下对应的 `*_mode_oracle.py`
* PX4 任务：`px4_hold_mode.py`、`px4_land_mode.py`、`px4_orbit_mode.py`、`px4_rtl_mode.py`
* PX4 oracle：`routh_search/px4/` 下对应的 `*_oracle.py`

运行时的关键前提：

* 脚本通过 `udp:127.0.0.1:14550` 连接 MAVLink。
* ArduPilot 脚本依赖环境变量 `ARDUPILOT_HOME` 与 `ARDUPILOT_FUZZ_HOME`。
* PX4 脚本默认你已经启动了 PX4 SITL，并且可以访问生成的 `.ulg` 日志。

### 2. 运行完整的边界搜索脚本
这是论文实验使用的入口，对应 README 中的边界识别模块。

`evaluation_script/` 下每个飞行模式通常包含两个 Bash 入口：

* `*_identify_bounder.sh`：识别有效/无效 PID 边界。
* `*_fuzz.sh`：基于边界继续搜索更多无效 PID 配置。

示例：

```bash
bash evaluation_script/rtl_routh/rtl_routh_identify_bounder.sh
bash evaluation_script/rtl_routh/rtl_routh_fuzz.sh
bash evaluation_script/hold_roll_roth/px4_hold_roll_routh_identify_bounder.sh
```

这些脚本的基本流程是：

1. 生成一组待测试的 PID 参数。
2. 启动任务执行器。
3. 对生成日志执行对应的 oracle。
4. 根据有效/无效结果更新边界。

## 当前仓库状态说明
这个仓库更适合作为**研究工件**理解，而不是直接开箱即用的产品。

`routh_search/` 下的 Python 任务与 oracle 脚本，在飞控与仿真环境配置完成后，可以直接用于原生验证；但 `evaluation_script/` 这一层仍然明显依赖作者原始实验环境。

具体来说，`evaluation_script/` 中当前仍引用了：

* 硬编码路径，例如 `/home/li/UAVFuzzing`、`/home/li/pgfuzz/...`
* Docker 镜像，例如 `rtl_mode:latest`、`rtl_oracle:latest`、`px4_hold_mode` 及对应 oracle 镜像
* 辅助文件，例如 `generate_abnormal_config.py` 和 `default.json`

其中有一部分并没有随仓库一起提供。因此：

* 如果你的目标是理解系统、验证环境、跑通单个任务，建议优先使用 `routh_search/`。
* 如果你的目标是完整复现实验表格 RQ1-RQ7，则需要先根据本地环境修改 `evaluation_script/`，并补齐缺失的 Docker 镜像和辅助文件。

## 实验结果
### RQ1

| 模式 | Total | Identified | Accurate | MR | HR |
| ---------- | -----: | ---------: | -------: | ----: | ----: |
| AP:Zigzag  | 28,162 |     21,107 |   19,367 | 31.2% | 91.8% |
| AP:Brake   | 36,584 |     33,357 |   32,902 | 10.1% | 98.6% |
| AP:RTL     | 27,267 |     26,624 |   26,545 |  2.6% | 99.7% |
| AP:Circle  | 20,477 |     23,016 |   17,022 | 16.9% | 74.0% |
| PX4:Orbit  |  8,979 |      8,103 |    6,630 | 26.2% | 81.8% |
| PX4:Return |  9,592 |      8,604 |    8,429 | 12.1% | 98.0% |
| PX4:Land   |  9,610 |      9,123 |    8,885 |  7.5% | 97.4% |
| PX4:Hold   |  4,930 |      4,394 |    4,165 | 15.5% | 94.8% |
| Average    | 18,200 |     16,791 |   15,493 | 15.3% | 92.0% |

### RQ2

| 模式 | RouthSearch | PGFuzz₁ | PGFuzz₂ | PGFuzz₃ |
| ---------- | ----------: | ------: | ------: | ------: |
| AP:Zigzag  |       3,390 |     557 |     617 |     518 |
| AP:Brake   |      11,820 |     406 |     289 |     262 |
| AP:RTL     |       4,969 |   1,319 |   1,325 |   1,286 |
| AP:Circle  |       3,533 |      93 |      78 |      76 |
| PX4:Orbit  |       1,909 |     290 |     236 |     207 |
| PX4:Return |       1,092 |     299 |     278 |     256 |
| PX4:Land   |       2,037 |     635 |     495 |     576 |
| PX4:Hold   |       2,070 |     305 |     193 |     171 |
| Average    |       3,853 |     488 |     439 |     419 |

### RQ3

| 模式 | Total | RouthSearch-DSOff |  | RouthSearch |  |
| ---------- | -----: | ----------------: | ----: | ----------: | ----: |
|            |        | Number | MR | Number | MR |
| AP:Zigzag  | 28,162 | 531 | 98.1% | 19,367 | 31.2% |
| AP:Brake   | 36,584 | 32,515 | 11.1% | 32,902 | 10.1% |
| AP:RTL     | 27,267 | 26,531 | 2.7% | 26,545 | 2.6% |
| AP:Circle  | 20,477 | 15,678 | 23.4% | 17,022 | 16.9% |
| PX4:Orbit  | 8,979 | 3,779 | 57.9% | 6,630 | 26.2% |
| PX4:Return | 9,592 | 5,340 | 44.3% | 8,429 | 12.1% |
| PX4:Land   | 9,610 | 6,737 | 29.9% | 8,885 | 7.5% |
| PX4:Hold   | 4,930 | 3,175 | 35.6% | 4,165 | 15.5% |
| Average    | 18,200 | 11,786 | 37.9% | 15,493 | 15.3% |

### RQ4

| Step Size | Total | Identified | Accurate | MR | HR |
| --------- | ------: | ---------: | -------: | ----: | ----: |
| 1x        | 374,578 | 330,727 | 325,300 | 13.2% | 98.4% |
| 10x       | 36,584 | 33,357 | 32,902 | 10.1% | 98.6% |
| 100x      | 4,012 | 3,812 | 3,673 | 8.4% | 96.4% |

### RQ5

| 模式 | Experimental Setting | MR | HR | CPU hours |
| ----- | ----: | ----: | ----: | ----: |
| AP: RTL | Exhaustive | 6.7% | 98.9% | 9,067 |
|  | 10x Sampling | 2.6% | 99.7% | 907 |

### RQ6

| 模式 | Online | Offline |
| ----- | ----: | ----: |
| AP:Brake | 96.5% | 99.4% |
| AP:Circle | 72.7% | 96.5% |
| AP:RTL | 99.7% | 99.7% |
| AP:Zigzag | 97.1% | 97.1% |
| PX4:Hold | 100% | 100% |
| PX4:Land | 100% | 100% |
| PX4:Orbit | 28.5% | 97.3% |
| PX4:Return | 100% | 100% |

### RQ7

| 模式 | RouthSearch-CS |  |  | RouthSearch-HC |  |  | RouthSearch-GA |  |  |
| ----- | ----: | ----: | ----: | ----: | ----: | ----: | ----: | ----: | ----: |
|  | MR | HR | number | MR | HR | number | MR | HR | number |
| AP:Zigzag | 31.2% | 91.8% | 3,390 | 98.3% | 25.1% | 481 | 93.0% | 20.2% | 1,928 |
| AP:Brake | 10.1% | 98.6% | 11,820 | 93.5% | 49.3% | 2,373 | 89.9% | 35.5% | 3,688 |
| AP:RTL | 2.6% | 99.7% | 4,969 | 95.0% | 37.9% | 1,379 | 88.4% | 30.3% | 3,204 |
| AP:Circle | 16.9% | 74.0% | 3,533 | 95.4% | 45.6% | 934 | 90.2% | 19.1% | 1,874 |
| PX4:Orbit | 26.2% | 81.8% | 1,909 | 98.0% | 22.8% | 173 | 96.4% | 42.2% | 319 |
| PX4:Return | 12.1% | 98.0% | 1,092 | 98.3% | 37.6% | 162 | 96.4% | 40.3% | 345 |
| PX4:Land | 7.5% | 97.4% | 2,037 | 96.3% | 40.8% | 358 | 95.8% | 39.3% | 401 |
| PX4:Hold | 15.5% | 94.8% | 2,070 | 96.8% | 21.9% | 158 | 94.5% | 29.8% | 270 |
| Average | 13.0% | 92.0% | 3853 | 96.5% | 35.1% | 752 | 93.1% | 32.1% | 1504 |
