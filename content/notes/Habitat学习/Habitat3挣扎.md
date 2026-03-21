---
title: Habitat3 挣扎
date: 2026-03-20T22:00:00+08:00
draft: false
tags:
  - Habitat
  - Conda
summary: "安装Habitat3的心路历程"
---
```bash
source /home/zyg/anaconda3/etc/profile.d/conda.sh
conda activate habitat3
export LD_LIBRARY_PATH=/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu
export LD_PRELOAD=/lib/x86_64-linux-gnu/libEGL_nvidia.so.0:/lib/x86_64-linux-gnu/libGLX_nvidia.so.0
export __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __NV_PRIME_RENDER_OFFLOAD=1
python /home/zyg/habitat3_ws/habitat-lab/verify_display.py
```


```python
import sys
import os
# 替换下面的路径为你实际的 egg 文件路径
sys.path.append("/home/zyg/anaconda3/envs/habitat3/lib/python3.9/site-packages/habitat_sim-0.3.3-py3.9-linux-x86_64.egg")
import habitat_sim

# 1. 场景路径
test_scene = "data/scene_datasets/habitat-test-scenes/apartment_1.glb"
if not os.path.exists(test_scene):
	print("❌ 场景文件不存在，请检查路径")
	exit()
# 2. 配置仿真器
sim_cfg = habitat_sim.SimulatorConfiguration()
sim_cfg.scene_id = test_scene
sim_cfg.enable_physics = True
# 尝试指定 GPU 0 (NVIDIA)
#sim_cfg.gpu_device_id = 0
# 3. 配置传感器
agent_cfg = habitat_sim.agent.AgentConfiguration()
rgb_sensor = habitat_sim.CameraSensorSpec()
rgb_sensor.uuid = "color_sensor"
rgb_sensor.sensor_type = habitat_sim.SensorType.COLOR
rgb_sensor.sensor_subtype = habitat_sim.SensorSubType.PINHOLE
rgb_sensor.resolution = [512, 512]
rgb_sensor.position = [0.0, 1.5, 0.0]
agent_cfg.sensor_specifications = [rgb_sensor]
cfg = habitat_sim.Configuration(sim_cfg, [agent_cfg])
print("🚀 正在尝试初始化仿真器 (Python版)...")
try:
# 这里是关键：看 Python 能否成功创建上下文
sim = habitat_sim.Simulator(cfg)
print("✅ 仿真器初始化成功！")
# 尝试渲染一帧
agent = sim.initialize_agent(0)
obs = sim.step("move_forward")
if "color_sensor" in obs:
print(f"✅ 成功获取图像数据！尺寸: {obs['color_sensor'].shape}")
print("🎉 结论：你的环境可以正常运行 Python 代码！不用管 habitat-viewer 的报错了。")
else:
print("⚠️ 初始化成功但未获取到图像。")
sim.close()
except Exception as e:
print(f"❌ Python 初始化也失败了: {e}")
```


```bash
source /home/zyg/anaconda3/etc/profile.d/conda.sh
conda activate habitat3
cd /home/zyg/habitat3_ws/habitat-lab

python -m habitat_sim.utils.datasets_download --uids replica_cad_dataset --data-path data/
python -m habitat_sim.utils.datasets_download --uids hab_fetch --data-path data/
python -m habitat_sim.utils.datasets_download --uids ycb --data-path data/


export LD_LIBRARY_PATH=/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu
export LD_PRELOAD=/lib/x86_64-linux-gnu/libEGL_nvidia.so.0:/lib/x86_64-linux-gnu/libGLX_nvidia.so.0
export __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __NV_PRIME_RENDER_OFFLOAD=1

SDL_VIDEODRIVER=x11 python examples/interactive_play.py --cfg benchmark/rearrange/play/play.yaml


python examples/interactive_play.py --no-render --save-obs --save-obs-fname play.mp4


#确认数据集完整性
ls -l data/replica_cad/scene_dataset_config.json
ls -l data/replica_cad/navmeshes/v3_sc4_staging_00.navmesh
ls -l data/robots/hab_fetch/fetch_no_base.urdf
ls -l data/objects/ycb/ycb.scene_dataset_config.json
ls -l data/datasets/replica_cad/rearrange


```


trouble
```txt
# 故障排除

这是一份常见问题列表以及相应的故障排除提示。

如果在Github上提交问题，请包含以下内容：
* 关于您环境的信息（操作系统、显卡等）。
* 你已经尝试过的故障排除步骤。
* 截图（如适用）。

- [常见问题](#常见问题)
  - [Habitat-Lab interactive_play.py](#habitat-lab-interactive_playpy)
  - [无法创建上下文](#无法创建上下文)
  - [NVIDIA A100上的黑色方块](#black-squares-on-nvidia-a100)
- [图形故障排除技巧](#图形故障排除技巧)
    - [通用提示](#通用提示)
    - [Linux](#linux)
    - [Windows](#windows)


## 常见问题

### Habitat-Lab interactive_play.py

在某些系统上，`examples/interactive_play.py`会因以下错误而崩溃：


X 失败请求的错误：BadAccess（尝试访问私有资源被拒绝）

这是一个与底层`pygame`库如何与Habitat交互相关的持续问题。`interactive_play`脚本的替代方案已在[路线图](habitat-hitl/README.md)中列出。

### 无法创建上下文

在某些系统上，无法找到有效的GPU：


Platform::WindowlessEglApplication::tryCreateContext()：在总共2个EGL设备中，无法找到CUDA设备0
无窗口上下文：无法创建无窗口上下文


这通常是由以下原因引起的：
* 图形驱动程序已过时、缺失或无效。
* 在 Linux 上，这可能是由于缺少或不完整的 `libglvnd` 安装。

请按照[以下](#graphics-troubleshooting-tips)的图形故障排除步骤进行操作。如果问题仍然存在，请随时在Github上提交问题。
### NVIDIA A100上的黑色方块
NVIDIA A100 GPU可能会导致Habitat传感器在某些环境中渲染出黑色矩形瑕疵。
如果您的设置出现此问题，请将CUDA驱动程序更新到最新版本（至少12.2）。
参见：https://github.com/facebookresearch/habitat-sim/issues/2310
## 图形故障排除技巧
这些步骤旨在缩小与图形相关问题的范围。
#### 一般提示
1. 提高日志记录的详细程度。
    使用以下环境变量启动应用程序：
    * `HABITAT_SIM_LOG=Debug`
      * 可选值，从最详细到最简略依次为：`VeryVerbose`、`Verbose`/`Debug`、`Warning`（默认值）、`Error`。
    * `MAGNUM_LOG=verbose`
      * 可选值，从最详细到最简洁依次为：`verbose`、`default`、`quiet`。
    * `MAGNUM_GPU_VALIDATION=ON`
2. 获取测试资源。
    * 在`habitat-lab`或`habitat-sim`存储库的根目录下，运行以下命令：`python -m habitat_sim.utils.datasets_download --uids habitat_test_scenes --data-path data/`
3. 通过运行基础查看器来验证问题是否出现在最小配置上。
    * 使用`habitat-sim` Conda包：
      * `MAGNUM_LOG=verbose MAGNUM_GPU_VALIDATION=ON habitat-viewer data/scene_datasets/habitat-test-scenes/skokloster-castle.glb`
    * 使用从源代码构建的`habitat-sim`：
      * `MAGNUM_LOG=verbose MAGNUM_GPU_VALIDATION=ON {habitat-sim}/build/viewer data/scene_datasets/habitat-test-scenes/skokloster-castle.glb`
4. 从头开始创建一个新的conda环境。一段时间后，该环境可能会偏离基线，从而导致冲突的出现。
#### Linux
1. *(NVIDIA)* 运行 `nvidia-smi` 命令。输出结果的首部应显示预期的驱动程序和 CUDA 版本。例如：
    ```NVIDIA-SMI 535.129.03 驱动程序版本：535.129.03 CUDA版本：12.2```
    如果命令返回错误或出现意外版本，请重新安装NVIDIA驱动程序。
2. 命令“eglinfo”应能正常运行，无错误。
    * *(NVIDIA, AMD)* 该命令应指示您的硬件GPU可用。
3. *(NVIDIA)* 请确保已安装并正确配置了`libglvnd`。此项配置经常被错误安装。
    1. 检查是否已安装`libglvnd`。
        * 基于Debian的系统：`apt list --installed | grep libglvnd`
        * 基于RPM：`dnf list installed | grep libglvnd`
    2. 确保存在针对NVIDIA的有效条目。
        在 `/usr/share/glvnd/egl_vendor.d/` 目录下（具体路径可能有所不同），请确保存在一个名为 `10_nvidia.json` 的条目。如果不存在，您可以尝试手动创建它。
        * 此文件应包含以下内容：
            ```
            {
                "file_format_version" : "1.0.0",，
                "ICD" : {
                    “library_path” : “libEGL_nvidia.so.0”
                }
            }
            ```
        * `library_path`字段应指向正确的NVIDIA EGL库。该库可通过命令`ldconfig -p | grep libEGL`找到。库`libEGL_nvidia`随NVIDIA驱动程序打包 - 如果缺失，请重新安装驱动程序。如果存在多个实例，则可能是驱动程序安装错误。
4. *(CPU渲染)* 如果你没有GPU，请确保你的CPU显卡驱动正常工作。
5. 如果在Wayland上运行，请尝试使用X11，反之亦然。虽然Habitat在两种显示协议下都应能正常工作，但这或许能揭示问题所在。

```

运行sc0-3
```bash
cd /home/zyg/habitat3_ws/habitat-lab
conda activate habitat3

# 如果你之前需要这个才能跑通渲染
export __NV_PRIME_RENDER_OFFLOAD=1

# sc0–sc2
python examples/interactive_play.py --cfg benchmark/rearrange/play/play.yaml habitat.dataset.split=train

# sc3
python examples/interactive_play.py --cfg benchmark/rearrange/play/play.yaml habitat.dataset.split=val


```
### habitat-hitl解决方案
readme
```txt
# 人机交互（HITL）框架
HITL框架将真实的人类用户带入Habitat虚拟环境。利用该框架构建交互式桌面和虚拟现实（VR）应用程序，使用户能够与模拟机器人和其他虚拟代理进行交互。将这些应用程序部署给用户，以收集交互数据，用于代理评估和训练。
HITL框架由`habitat-hitl` Python库、示例[桌面应用程序](../examples/hitl/)，以及我们基于Unity的[VR客户端](../examples/hitl/pick_throw_vr/README.md#vr)组成。
该框架基于`habitat-lab`和`habitat-baselines`构建。它提供了封装器，用于实时加载、模拟和渲染Habitat环境，包括虚拟代理和策略推理。为了使用户能够与这些代理进行交互，该框架提供了图形用户界面（GUI），包括一个带有3D视口的窗口、相机控制和化身控制助手，以及虚拟现实（VR）集成。

  <img src="../res/img/hitl_tool.gif" height=400>

- [人机交互（HITL）框架](#人机交互-hitl-框架)
  - [系统要求](#系统要求)
  - [安装](#installation)
  - [HITL应用示例](#example-hitl-applications)
  - [虚拟现实（VR）HITL应用](#vr-hitl-applications)
  - [配置](#configuration)
  - [最小化HITL应用](#最小化HITL应用)
## 系统要求
* **操作系统**：macOS或Linux。我们已测试过Fedora，但其他Linux发行版应该也能正常运行。
* **CPU/GPU**：Apple M1 Pro/Max、Intel Core i7 + 独立显卡或同等配置。
* **显示设备**：配备外接显示器的笔记本电脑或台式机。我们尚未测试远程桌面或其他无头选项。
* **存储空间**：约20 GB，包括Habitat依赖项和运行时数据（如HSSD）。这不包括常见的第三方库和操作系统库。一台Ubuntu机器需要大约60 GB的存储空间，用于操作系统以及运行HITL应用程序所需的其他所有内容。
* **虚拟现实（VR）：** HITL VR采用客户端/服务器模型，该模型既需要一台桌面系统（如上所述），也需要一个VR头戴设备。有关详细信息，请参阅[`pick_throw_vr/README.md`](../examples/hitl/pick_throw_vr/README.md)。
示例HITL（人机交互测试）应用程序配置为每秒运行30步（SPS）。如果您的系统不符合上述规格，则SPS会降低，用户体验也会下降。
## 安装
1. 克隆Habitat-lab [主分支](https://github.com/facebookresearch/habitat-lab)。
2. 按照[安装说明](https://github.com/facebookresearch/habitat-lab#installation)安装Habitat-lab。
    * HITL框架依赖于`habitat-lab`和`habitat-baselines`这两个包。虽然这些包与HITL框架位于同一仓库中，但HITL框架并不一定需要导入这些位于[人机交互（HITL）框架](#human-in-the-loop-hitl-framework)中的包
3. 安装Habitat-sim主分支（[https://github.com/facebookresearch/habitat-sim）](https://github.com/facebookresearch/habitat-sim%EF%BC%89)。
    * [从源代码构建](https://github.com/facebookresearch/habitat-sim/blob/main/BUILD_FROM_SOURCE.md)，或安装[conda包](https://github.com/facebookresearch/habitat-sim#recommended-conda-packages)。
        * 请务必包含Bullet物理引擎，例如`python setup.py install --bullet`。
4. 安装`habitat-hitl`包。
    * 在`habitat-lab`根目录下，运行`pip install -e habitat-hitl`。
5. 为我们的示例HITL（人机交互训练）应用程序下载所需资源（请注意，数据集下载器应从habitat-lab/目录下运行）：
    ```bash
    python -m habitat_sim.utils.datasets_download \
    --uids hab3-episodes habitat_humanoids hab_spot_arm ycb hssd-hab \
    --data-path data/
    ```
## 数据目录和运行位置
HITL应用程序（以及一般的Habitat库）期望在运行位置（即当前工作目录）中有一个`data/`目录。请注意我们上述安装步骤中的`--data-path`参数。以下有两种选项可供考虑：
1. 按照上述安装步骤，将数据下载到`habitat-lab/data`目录。这是许多Habitat教程和实用程序的默认位置。从此位置运行您的HITL应用程序，例如`habitat-lab/$ python /path/to/my_hitl_app/my_hitl_app.py`。
2. 将数据下载（或使用符号链接）到您的HITL应用程序的根目录，例如`/path/to/my_hitl_app/data`。从此位置运行您的HITL应用程序，例如`/path/to/my_hitl_app/$ python my_hitl_app.py`


## 示例HITL应用
请查看我们的示例HITL应用[点击此处](../examples/hitl/)。
将这些作为参考来创建您自己的HITL（硬件在环）应用程序。我们建议您首先将一个示例应用程序文件夹（如`hitl/pick_throw_vr/`）复制粘贴到您自己的Git仓库中，例如`my_repo/my_pick_throw_vr/`。
## VR HITL应用
HITL框架可用于构建桌面应用程序（通过键盘/鼠标控制）以及**虚拟现实（VR）**应用程序。请参阅我们的[Pick_throw_vr](../examples/hitl/pick_throw_vr/README.md)示例应用程序。
## 配置
HITL（人机交互测试）应用程序使用Hydra进行配置，例如，控制桌面窗口的宽度和高度。请参阅[`hitl_defaults.yaml`](./config/hitl_defaults.yaml)以及每个示例应用程序的单独配置文件，例如[`pick_throw_vr.yaml`](../examples/hitl/pick_throw_vr/config/pick_throw_vr.yaml)。另请参阅[`habitat-lab/habitat/config/README.md`](../habitat-lab/habitat/config/README.md)。
## 最小化HITL应用
通过实现派生的AppState、加载合适的Hydra配置并调用`hitl_main`，构建一个最小的桌面HITL应用。让我们来仔细看看：
# minimal_cfg.yaml
# 有关Hydra的更多信息，请参阅habitat-lab/habitat/config/README.md。
# @package _global_
默认值：
  # 我们加载了以Spot机器人为特色的`pop_play` Habitat基线模型
  # 以及HSSD场景中的人形实体。请参阅habitat-baselines/README.md。
  - social_rearrange: pop_play
  # 为HITL框架加载默认参数。请参阅
  # habitat-hitl/habitat_hitl/config/hitl_defaults.yaml。
  - hitl_defaults
  - _self_

# minimal.py
import hydra
import magnum
from habitat_hitl.app_states.app_state_abc import AppState
from habitat_hitl.core.gui_input import GuiInput
from habitat_hitl.core.hitl_main import hitl_main
from habitat_hitl.core.hydra_utils import register_hydra_plugins
class AppStateMinimal(AppState):
"""
A minimal HITL app that loads and steps a Habitat environment, with
a fixed overhead camera.
"""
def __init__(self, app_service):
self._app_service = app_service

def sim_update(self, dt, post_sim_update_dict):
"""
The HITL framework calls sim_update continuously (for each

"frame"), before rendering the app's GUI window.
"""
# run the episode until it ends
if not self._app_service.env.episode_over:
self._app_service.compute_action_and_step_env()

# set the camera for the main 3D viewport
post_sim_update_dict["cam_transform"] = magnum.Matrix4.look_at(
eye=magnum.Vector3(-20, 20, -20),
target=magnum.Vector3(0, 0, 0),
up=magnum.Vector3(0, 1, 0),
)

  

# exit when the ESC key is pressed
if self._app_service.gui_input.get_key_down(KeyCode.ESC):
post_sim_update_dict["application_exit"] = True

@hydra.main(version_base=None, config_path="./", config_name="minimal_cfg")
def main(config):
hitl_main(config, lambda app_service: AppStateMinimal(app_service))

if __name__ == "__main__":
register_hydra_plugins()
main()


See the latest version of the minimal app [here](../examples/hitl/minimal/).

```


```bash
cd /home/zyg/habitat3_ws/habitat-lab
conda activate habitat3
export DISPLAY=:0
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json
export LD_PRELOAD=/lib/x86_64-linux-gnu/libGLX_nvidia.so.0:/lib/x86_64-linux-gnu/libEGL_nvidia.so.0
SDL_VIDEODRIVER=x11 python examples/interactive_play.py --cfg benchmark/rearrange/play/play.yaml habitat.dataset.split=train


python examples/interactive_play.py --cfg benchmark/rearrange/play/play.yaml habitat.dataset.split=train --no-render --save-obs --save-obs-fname sc0_sc2.mp4


```

修改了源文件
![](images/Pasted%20image%2020260115170613.png)![](images/Pasted%20image%2020260115171619.png)



```bash
# 新的修改
export DISPLAY=:0
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json
export LD_PRELOAD=/lib/x86_64-linux-gnu/libGLX_nvidia.so.0:/lib/x86_64-linux-gnu/libEGL_nvidia.so.0
export SDL_VIDEODRIVER=x11

#如果仍报 BadAccess，再执行一次（每次登录只需一次）：
xhost +SI:localuser:$USER

python examples/interactive_play.py --no-render --save-obs --save-obs-fname play.mp4



# 一直渲染111
cd /home/zyg/habitat3_ws/habitat-lab
conda activate habitat3
export DISPLAY=:0
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json
export LD_PRELOAD=/lib/x86_64-linux-gnu/libGLX_nvidia.so.0:/lib/x86_64-linux-gnu/libEGL_nvidia.so.0
export SDL_VIDEODRIVER=x11
python examples/interactive_play.py --never-end



# pygame heiping 
# nb!!!!!!!!!!!!
# 这个是好的，可以现在pygame完全实现了！！！
export SDL_VIDEODRIVER=x11
export SDL_VIDEO_X11_FORCE_EGL=1
export SDL_RENDER_DRIVER=software
python examples/interactive_play.py --never-end

```
![](images/Pasted%20image%2020260115174428.png)
![](images/Pasted%20image%2020260115174620.png)
```txt
使用I/J/K/L键可控制机器人底座向前/向左/向后/向右移动，使用W/A/S/D键可控制手臂末端执行器向前/向左/向后/向右移动，使用E/Q键可控制手臂上下移动。通过末端执行器控制手臂可能较难操作。更多详情请参阅文档。尝试移动底座和手臂，使其触碰桌上的红色碗。祝您愉快！

注意：当前在Ubuntu 20.04上，交互式测试会失败，并报错：X错误：请求失败：BadAccess（尝试访问私有资源被拒绝）。我们正在努力修复此问题，一旦修复完成，将立即更新相关说明。该脚本在MacOS上运行正常，无报错。
```

## 环境对比
以下是 **(habitat3)** 与 **(base)** 环境中参数不同的地方对比：
### 1. 核心变量修改 (值发生改变)

|变量名|(habitat3) 的值|(base) 的值|说明|
|---|---|---|---|
|**CONDA_DEFAULT_ENV**|`habitat3`|`base`|环境名称|
|**CONDA_PREFIX**|`.../envs/habitat3`|`.../anaconda3`|环境路径|
|**CONDA_SHLVL**|`2`|`1`|嵌套层级|
|**CONDA_PROMPT_MODIFIER**|`(habitat3)`|`(base)`|终端提示符|
|**PATH**|`/home/zyg/anaconda3/envs/habitat3/bin:...` (habitat优先)|`/home/zyg/anaconda3/bin:...` (base优先)|可执行程序搜索顺序|
|**LD_LIBRARY_PATH**|`/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu`|`/opt/ros/humble/...:/opt/ros/humble/lib`|**关键差异**：habitat3 指向系统库，base 指向 ROS 库|
|**GSETTINGS_SCHEMA_DIR**|`.../envs/habitat3/share/glib-2.0/schemas`|`.../anaconda3/share/glib-2.0/schemas`|GNOME 配置路径|
|**SYSTEMD_EXEC_PID**|`2243`|`11268`|进程 PID (运行时的随机差异)|
|**GNOME_TERMINAL_SCREEN**|`.../screen/61ab9170...`|`.../screen/45b5736f...`|终端窗口 ID (运行时的随机差异)|

---

### 2. (habitat3) 独有的变量 (在 base 中不存在)

**这些全部是为 Habitat 仿真器配置的图形驱动和渲染参数：**
- `CONDA_PREFIX_1="/home/zyg/anaconda3"` (记录上一级环境路径)
- **显卡驱动强制加载:**
    - `LD_PRELOAD="/lib/x86_64-linux-gnu/libGLX_nvidia.so.0:/lib/x86_64-linux-gnu/libEGL_nvidia.so.0"`
- **SDL (仿真器窗口) 配置:**
    - `SDL_RENDER_DRIVER="software"`
    - `SDL_VIDEODRIVER="x11"`
    - `SDL_VIDEO_X11_FORCE_EGL="1"`
- **NVIDIA / EGL 厂商配置:**
    - `__EGL_VENDOR_LIBRARY_DIRS="/home/zyg/anaconda3/envs/habitat3/share/glvnd/egl_vendor.d"`
    - `__EGL_VENDOR_LIBRARY_FILENAMES="/usr/share/glvnd/egl_vendor.d/10_nvidia.json"`
    - `__GLX_VENDOR_LIBRARY_NAME="nvidia"`
    - `__NV_PRIME_RENDER_OFFLOAD="1"`
---
### 3. (base) 独有的变量

_(无。base 环境中的变量在 habitat3 中基本都存在，只是值不同)_