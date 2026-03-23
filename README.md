# RouthSearch
[中文说明 / Chinese README](README.zh-CN.md)

## Description
RouthSearch, a tool to determine the valid three-dimensional PID parameter ranges of UAVs under different flight modes. This tool comprised of two modules: Boundary Identification Module and Misbehavior validation Module.

The boundary identification module first takes in PID parameters and their corresponding values range from the official documents of UAVs. It then efficiently searches the PID parameter space to find a classification boundary between valid and invalid configurations. The implementation of boundary identification module is available at `evaluation_script/`.

During the search, the boundary identification module continuously queries the misbehavior validation module to determine whether a given PID parameter configuration is valid. The implementation of misbehavior validation module is available at `routh_search/`.

## Purpose And Paper Relation
This repository is the artifact for the paper **RouthSearch: Inferring PID Parameter Specification for Flight Control Program by Coordinate Search**.

The goal of RouthSearch is not to find a single "best" PID tuple. Instead, it infers a **valid three-dimensional PID boundary** for a given flight mode, so dangerous PID configurations can be filtered before they cause unstable behavior such as oscillation, path deviation, loss of control, or crash.

At a high level, the paper combines three ideas:

1. Use the Routh-Hurwitz criterion to derive a theoretical PID boundary.
2. Use coordinate search to refine that boundary against the real autopilot and simulator behavior.
3. Use flight-task-specific oracles to decide whether a tested PID configuration is valid or invalid.

In this repository, the mapping from the paper to the code is:

* `evaluation_script/`: boundary identification and fuzzing scripts used to search the PID space.
* `routh_search/`: mission drivers and oracle implementations used to validate each tested PID configuration.
* `ardupilot_default_data/`: default parameter data used by the experiments.

For a detailed paper walkthrough, see `RouthSearch_完整中文翻译与细致解析.md`.
## Environment
1. Languages: 
   * Python(Python 3.10)
   * Bash

2. Flight Control Programs:
   * [Ardupilot](https://ardupilot.org/)
   * [PX4](https://px4.io/)

3. UAV Simulator:
   * [Ardupilot SITL Simulator](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html)
   * [jMAVSim](https://docs.px4.io/main/en/sim_jmavsim/)

### Setup & Usage Instruction
#### Ardupilot
Install Ardupilot by following instructions from this link: [Ardupilot Setup](https://ardupilot.org/dev/docs/building-setup-linux.html)

#### Ardupilot SITL Simulator
Setup and using Ardupilot SITL by following instructions from this link: [Ardupilot SITL setup](https://ardupilot.org/dev/docs/copter-sitl-mavproxy-tutorial.html)

#### PX4
Install Ardupilot by following instructions from this link: [PX4 Setup](https://docs.px4.io/main/en/dev_setup/building_px4.html)

### jMAVSim
Setup and Running jMAVSim by following instructions from this link: [jMAVSim Setup](https://docs.px4.io/main/en/sim_jmavsim/)

## Project Structure
```
RouthSearch/
|-- ardupilot_default_data/              # Default parameters
|-- evaluation_script/                   # evaluation shell script
|-- routh_search/                        # code of flight task and oracle
|-- README.md                            # Project documentation
```

This project, `RouthSearch`, explores automated test generation for UAV (Unmanned Aerial Vehicle) flight missions.  It leverages parameter fuzzing techniques, specifically focusing on the open-source autopilot system ArduPilot.

The project is structured as follows:

* **`ardupilot_default_data/`**: This directory contains the default parameter settings for the ArduPilot system.

* **`evaluation_script/`**: Contains shell scripts used for evaluation.

* **`routh_search/`**:  This directory contains the code executing specific UAV flight task and the "oracle" used to determine the success or failure of a given flight task.  

## How To Use This Repository
There are two practical ways to use this repository.

### 1. Run The Native Mission And Oracle Scripts
This is the best entry point if you want to understand the repository, smoke-test the environment, or manually validate one PID configuration.

Typical workflow:

1. Start the corresponding simulator and autopilot.
2. Run one mission script under `routh_search/`.
3. Run the matching oracle on the generated flight log.

Examples:

```bash
python3 -m compileall routh_search

# ArduPilot example
python3 routh_search/ardupilot/start_ardupilot.py
python3 routh_search/ardupilot/rtl_mode.py <config.json>
python3 routh_search/ardupilot/rtl_mode_oracle.py <path-to-bin-log>

# PX4 example
# Start PX4 SITL separately, for example with jMAVSim in your PX4 checkout.
python3 routh_search/px4/px4_hold_mode.py
python3 routh_search/px4/px4_hold_mode_oracle.py <path-to-ulg-log>
```

The mission/oracle pairs in this repository are:

* ArduPilot: `brake_mode.py`, `circle_mode.py`, `rtl_mode.py`, `zigzag_mode.py`
* ArduPilot oracles: `*_mode_oracle.py` under `routh_search/ardupilot/`
* PX4: `px4_hold_mode.py`, `px4_land_mode.py`, `px4_orbit_mode.py`, `px4_rtl_mode.py`
* PX4 oracles: `*_oracle.py` under `routh_search/px4/`

Important runtime assumptions:

* The scripts communicate with the autopilot through MAVLink on `udp:127.0.0.1:14550`.
* ArduPilot scripts expect `ARDUPILOT_HOME` and `ARDUPILOT_FUZZ_HOME` to be set.
* PX4 scripts assume a running PX4 SITL instance and access to the generated `.ulg` logs.

### 2. Run The Full Boundary Search Scripts
This is the experiment layer used in the paper.

Each flight mode has two Bash entrypoints under `evaluation_script/`:

* `*_identify_bounder.sh`: identifies the valid/invalid PID boundary.
* `*_fuzz.sh`: continues exploring the space to find more invalid PID configurations.

Examples:

```bash
bash evaluation_script/rtl_routh/rtl_routh_identify_bounder.sh
bash evaluation_script/rtl_routh/rtl_routh_fuzz.sh
bash evaluation_script/hold_roll_roth/px4_hold_roll_routh_identify_bounder.sh
```

These scripts repeatedly:

1. Generate a PID configuration to test.
2. Launch a mission runner.
3. Execute the corresponding oracle on the produced log.
4. Update the inferred boundary from the observed valid/invalid result.

## Current Repository State
This repository is best understood as a research artifact rather than a polished product. The native Python mission/oracle scripts are directly usable once the simulator environments are prepared, but the full evaluation layer still assumes the original experiment setup.

In particular, the scripts under `evaluation_script/` currently reference:

* hard-coded paths such as `/home/li/UAVFuzzing` and `/home/li/pgfuzz/...`
* Docker images such as `rtl_mode:latest`, `rtl_oracle:latest`, `px4_hold_mode`, and related oracle images
* helper files such as `generate_abnormal_config.py` and `default.json`

Some of these referenced artifacts are not versioned in this repository. As a result, the `routh_search/` scripts are the most reliable starting point for understanding and validating the system, while reproducing the full RQ1-RQ7 evaluation requires adapting the experiment scripts to your local environment and supplying the missing Docker/helper artifacts.

## Evaluation Result
### RQ1

| Mode       |  Total | Identified | Accurate |    MR |    HR |
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

| Mode       | RouthSearch | PGFuzz₁ | PGFuzz₂ | PGFuzz₃ |
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

| Mode       |  Total | RouthSearch-DSOff |       | RouthSearch |       |
| ---------- | -----: | ----------------: | ----: | ----------: | ----: |
|            |        |            Number |    MR |      Number |    MR |
| AP:Zigzag  | 28,162 |               531 | 98.1% |      19,367 | 31.2% |
| AP:Brake   | 36,584 |            32,515 | 11.1% |      32,902 | 10.1% |
| AP:RTL     | 27,267 |            26,531 |  2.7% |      26,545 |  2.6% |
| AP:Circle  | 20,477 |            15,678 | 23.4% |      17,022 | 16.9% |
| PX4:Orbit  |  8,979 |             3,779 | 57.9% |       6,630 | 26.2% |
| PX4:Return |  9,592 |             5,340 | 44.3% |       8,429 | 12.1% |
| PX4:Land   |  9,610 |             6,737 | 29.9% |       8,885 |  7.5% |
| PX4:Hold   |  4,930 |             3,175 | 35.6% |       4,165 | 15.5% |
| Average    | 18,200 |            11,786 | 37.9% |      15,493 | 15.3% |

### RQ4

| Step Size |   Total | Identified | Accurate |    MR |    HR |
| --------- | ------: | ---------: | -------: | ----: | ----: |
| 1x        | 374,578 |    330,727 |  325,300 | 13.2% | 98.4% |
| 10x       |  36,584 |     33,357 |   32,902 | 10.1% | 98.6% |
| 100x      |   4,012 |      3,812 |    3,673 |  8.4% | 96.4% |

### RQ5

| Mode | Experimental Setting | MR | HR | CPU hours |
| ----- | ----: | ----: | ----: | ----: |
| AP: RTL | Exhaustive | 6.7% | 98.9%  | 9,067 |
|  | 10x Sampling |      2.6%  | 99.7%  | 907 |

### RQ6

| Mode | Online  | Offline |
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

| Mode | RouthSearch-CS |  |  | RouthSearch-HC |  |  | RouthSearch-GA |  |  |
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
