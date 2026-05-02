# LogoPlanner/NavDP 复现
（LogoPlanner是NavDP的其中一个baseline）
## 1. 环境背景

**计算环境**: Ubuntu 22.04 + CUDA 12.1 + PyTorch 2.3.0 

注意：
**配置环境不要直接使用命令行```pip install -r requirements.txt```，一定先弄好pytorch版本2.3+，默认下载的2.3以下版本会少模块**
**不要用虚拟机/wsl复现这个项目，可视化环节有底层文件对应不上**

---
## 2. 模型检查点/场景资源/预训练文件 下载
这个项目有很多资源需要去其他地方下载，很多需要用到huggingface，注册huggingface时可能会拒绝注册，可以**换一下梯子的节点**，我是换成美国的就注册好了

想复现每种baseline都需要下载它对应的预训练文件、资源，在zread.ai里输入这个仓库链接，出来的快速入门文档里“下载模型检查点与场景资源”里有总结每个模型对应的场景资源，这个链接里也有预训练资源，下载后放在对应baseline的对应方法文件夹里，记住路径，后面打开这个服务器要用到这个路径`${SAVE_PTH_PATH}`。

LogoPlanner相关资源的链接在这https://huggingface.co/InternRobotics/LoGoPlanner
## 3. 依赖库 (Pi3) 部署流程
LogoPlanner的github代码中有一个子模块Pi3，子模块在git克隆拉取的时候不会下载到本地，只会新建一个空文件夹，需要单独再拉取。如果你直接用NavDP这个github的目录里的路径，无法拉取到，想尝试本地下载也点不开。
但其实Pi3是交大实验室的另一个开源项目，地址为 https://github.com/yyfz/Pi3.git ，该问题处理方式如下：

1.  **准备目录结构**:
    ```bash
    cd ~/NavDP/baselines/logoplanner
    mkdir -p Pi3
    cd Pi3
    ```
2.  **克隆公开仓库 (镜像加速)**:
    ```bash
    git clone https://github.com/yyfz/Pi3.git .
    ```
3.  **验证结构**: 确保存在路径 `~/NavDP/baselines/logoplanner/Pi3/pi3/models/pi3.py`。

4.  **设置 PYTHONPATH**:
每次启动前需声明环境变量，确保 Python 能正确索引到 Pi3 模块。
    ```bash
    cd ~/NavDP/baselines/logoplanner
    export PYTHONPATH=$PYTHONPATH:$(pwd):$(pwd)/Pi3
    ```

## 4. 代码兼容性修正 (核心)
**如出现报错找不到模块torch.nn.attention，非常有可能是pytorch版本低，AI可能会说2.2以上就行，觉得你是多个torch混乱，但其实要升级版本到2.3+**

可用下面指令卸载当前并安装2.3+pytorch:
    ```bash
    pip uninstall torch torchvision torchaudio -y
    pip install torch==2.3.0+cu121 torchvision==0.18.0+cu121 torchaudio==2.3.0+cu121 \
      --index-url https://download.pytorch.org/whl/cu121
    ```


# 弄清LogoPlanner和机器狗SDK（相机、机器人动作）的输入输出

> 最终目标：根据LogoPlanner和机器狗SDK（相机、机器人动作）的输入输出写文件 将LoGoPlanner接入ROVER X1机器狗(相机)

三方分工：
- **LoGoPlanner Server**：不改。接收 RGB+深度+目标坐标，返回速度命令序列。
- **ROVER X1 SDK**：不改。提供相机数据订阅（DDS）和整机速度控制（gRPC）。
- **host.py**：本次唯一新写的代码。负责数据格式转换、协议适配、控制循环。

---


## LoGoPlanner的输入输出

在 NavDP 仓库下定位以下两个文件

| 文件 | 作用 |
|---|---|
| `baselines/logoplanner/logoplanner_realworld_server.py` | 真机版 Flask server，定义 HTTP 接口 |
| `baselines/logoplanner/lekiwi_logoplanner_host.py` | LeKiwi 小车的 host 模板（**重点**） |

从 server 文件提取以下关键事实：

- **`POST /navigator_reset`**
  - 请求体：`{"intrinsic": 3×3 list, "stop_threshold": float, "batch_size": int}`
  - 任务开始前调用一次。

- **`POST /pointgoal_step`**
  - multipart 上传：`image`（RGB JPEG）+ `depth`（16-bit PNG，单位毫米）
  - form 字段：`goal_data = {"goal_x": [..], "goal_y": [..]}`（list 形式，长度 = batch_size）
  - 返回：`{"cmd_list": [[vx, vy, wz], ...]}`
  - **注意 `wz` 是度/秒**（server 末尾 `* 180.0 / np.pi`），不是 rad/s。
  - 一次返回最多约 20 条命令，是 server 端 MPC 已经展开好的速度序列。

- **关键性质**：`pointgoal_step` 不需要客户端提供机器人位姿。LoGoPlanner 通过视觉 memory 隐式估计自身位置，goal 为相对启动点的世界坐标。

##  机器狗 相机、动作的输入输出

按以下顺序检查 `dobot_quad_sdk` 目录：

1. **`high_level/python/`**（gRPC）
   - `RobotClient.balance_stand()` / `stand_down()` / `set_target_state("walk")`：状态切换。
   - `RobotClient.velocity_sequence([(vx, vy, vyaw, dur), ...])`：整机速度控制。
   - 这是控制下行的**唯一可用通道**（见下条）。

2. **`low_level/python/`**（DDS）
   - `e1_rgb_image_sub.py`：订阅 `rt/camera/camera2/image_compressed`，JPEG 格式 → 与 LoGoPlanner 期待的 RGB 直接兼容。
   - `e2_depth_image_sub.py`：订阅 `rt/camera/camera2/image_depth`，16UC1 编码、单位毫米 → 与 LoGoPlanner 期待的深度格式直接兼容，**不需要单位换算**。
   - `e9_motor_cmd_pub.py`：12 关节级 PD 控制（`q/dq/tau/kp/kd`），**层级太低**，不可用——它需要你自己实现整套步态控制器。

3. **结论**
   - 控制：走 high-level gRPC `velocity_sequence`。这条路有阻塞和节奏抖动的风险，但是唯一选择。
   - 感知：走 low-level DDS 订阅，相机数据直接可用。

### 1.3 列出格式映射表

写代码前先把字段对齐写清楚：

| LoGoPlanner 需要 | ROVER X1 提供 | 转换                                                        |
|---|---|-----------------------------------------------------------|
| RGB（PIL，RGB 顺序） | DDS CompressedImage（JPEG，BGR 解码） | `cv2.imdecode` → `cv2.cvtColor(BGR2RGB)` → `Image.fromarray` |
| Depth（PIL `I;16`，单位 mm） | DDS Image 16UC1（uint16，单位 mm） | `Image.fromarray(arr, mode='I;16')`                       |
| 相机内参 K（3×3） | **SDK 未提供** | 需在出厂文件查看                                                  |
| `(vx, vy, wz)` rad/s | `velocity_sequence` 的 `(vx, vy, vyaw, dur)` | wz 度→弧度，附加 duration                                       |

