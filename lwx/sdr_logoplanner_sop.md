# 具身智能机器狗开发部署与联调标准作业程序 (SOP)

**文档版本:** V1.0
**日期:** 2026.05.02

---

## 一、 目的与概述
本 SOP 旨在规范越疆机器狗底层 SDK 与上层 AI 视觉导航算法的本地部署流程。主要涵盖Conda 环境配置、SDK 安装测试、算法本地部署。

**⚠️ 核心安全守则：**
在进行任何代码测试前，**必须确保机器狗处于悬空架设状态**，四腿离开桌面或地面，防止代码暴走造成人员伤害或设备损坏。

---

## 二、 硬件与网络环境准备（通信链路）

### 1. 电脑系统配置
* 使用双系统或独立主机运行ubuntu，切勿使用虚拟机和wsl
* **操作系统:** Ubuntu 22.04 LTS
* **显卡 (GPU):** 必须配备 NVIDIA 独立显卡,如果没有独显，机器狗会卡成幻灯片。
配有独立显卡的电脑运行ubuntu时需要上nvidia官网安装对应的驱动，否则可能导致系统无法正常启动。
安装成功后，重启电脑，设置成独显单独运行的模式，即可顺畅打开ubuntu

打开全新的终端 (`Ctrl + Alt + T`)，安装基础工具：
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install build-essential cmake git curl net-tools python3-pip -y
```

### 2. 物理连接
* 使用网线将 Ubuntu 电脑与机器狗网口直连。
* 确保机器狗已开机并完成自检。


### 3. 电脑端 IP 配置
* 进入 Ubuntu 网络设置 -> 有线连接 -> IPv4 选项。
* 模式切换为：**手动 (Manual)**。
* IP 地址配置：
  * **Address:** `192.168.5.10` （最后一位必须非 2）
  * **Netmask:** `255.255.255.0`
* 保存并**重启有线网络连接**（关闭后再次开启）。

### 4. 连通性测试
打开终端，执行 Ping 测试：
```bash
ping 192.168.5.2
```
合格标准: 持续输出 64 bytes from 192.168.5.2...，无丢包。按 Ctrl+C 停止。

---

## 三、 软件环境配置 (Conda 虚拟环境)

为防止系统 Python 环境污染，需使用 Conda 隔离机器人控制环境与 AI 算法环境。

### 1. 首次配置 (如遇 ToS 报错)
若创建环境时提示 `Terms of Service have not been accepted`，执行以下命令同意条款：
```bash
conda tos accept --override-channels --channel [https://repo.anaconda.com/pkgs/main](https://repo.anaconda.com/pkgs/main)
conda tos accept --override-channels --channel [https://repo.anaconda.com/pkgs/r](https://repo.anaconda.com/pkgs/r)
```

### 2. 创建并激活专属环境
```bash
conda create -n dobot python=3.10 -y
conda activate dobot
```

---

## 四、 底层驱动 SDK 部署与测试 (运动控制)

此阶段旨在通过 Python 代码实现对机器狗的高层动作调用。

### 1. 安装高层控制包 (开发模式)
在 `(dobot)` 环境下执行：
```bash
cd ~/dobot_quad_sdk/high_level/python
pip install -e .
```
*(注：`-e .` 表示以可编辑模式安装，后续修改 SDK 源码可实时生效)*

### 2. 动作连接测试
```bash
python examples/e1_get_available_motions.py
```
* **预期结果:** 终端打印出机器狗支持的所有动作列表。

### 3. 基础运动测试 (确认已架空)
```bash
python examples/e3_auto_state_switch.py
```
* **预期结果:** 机器狗四腿按预设逻辑进行状态切换与动作执行。

### 🚨 紧急停止预案
若测试中发生失控，立即在终端执行：
```bash
python examples/kill_robot.py 192.168.5.2:50051
```
*(执行流程：切换到 PASSIVE 状态 → 等待 5 秒 → 终止控制器进程。)*

---

## 五、 上层 AI 算法部署 (LoGoPlanner)

此阶段负责给机器狗安装用于导航决策的“大脑”。

### 1. 获取项目代码
```bash
cd ~
git clone [https://github.com/InternRobotics/NavDP.git](https://github.com/InternRobotics/NavDP.git)
```

### 2. 安装算法依赖
确保已进入 `NavDP` **项目根目录**（包含 `requirements.txt` 的目录）：
```bash
cd ~/NavDP
conda activate dobot
pip install -r requirements.txt
```
*(⚠️ 注意：此过程涉及 PyTorch 及 CUDA 依赖，若环境无独立 NVIDIA 显卡或 CUDA 版本不匹配，需前往 PyTorch 官网重新获取对应 GPU Driver 版本的安装命令。)*

### 3. 修复依赖冲突 (抢救通信桥梁)
安装 LoGoPlanner 依赖时，会强制降级 protobuf 和 grpcio，导致机器狗控制包失效。**安装完 requirements 后，必须强制把以下包升回官方版本：**
```bash
pip install grpcio==1.78.0 grpcio-tools==1.78.0 protobuf==6.33.6
```

### 4. 下载 AI 模型权重 (Checkpoints)
* 查阅 NavDP/logoplanner 的 README 文档，找到作者提供的模型权重下载链接。
* 将下载好的 `.pth` 文件放置在项目指定的目录下（通常为 `baselines/logoplanner` 或新建的 `weights` 文件夹）。
* 建议先根据 README 运行一次脱机 Demo，测试 AI 算法本身能否在电脑上正常输出指令。

---

## 六、 视觉感知与系统桥接 (缝合流程)

实现从“眼睛看”到“大脑想”再到“四肢动”的闭环控制框架。

### 1. 视觉通路测试 (以 USB 摄像头为例)
创建测试脚本 `test_camera.py`：
```python
import cv2
cap = cv2.VideoCapture(0)
while True:
    ret, frame = cap.read()
    if ret:
        cv2.imshow("Vision Test", frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break
cap.release()
```

### 2. 桥接脚本开发框架 (ai_driver.py)
在主目录新建 `ai_driver.py` 文件，核心代码骨架如下：
```python
import cv2
import time
from dobot_quad import RobotClient
# 导入 AI 模型 (需根据实际路径修改)
# from logoplanner.agent import LoGoPlannerModel 

def main():
    print("--- 系统初始化 ---")
    dog = RobotClient("192.168.5.2")
    cap = cv2.VideoCapture(0)
    # ai_brain = LoGoPlannerModel(weights_path="logoplanner.pth")
    
    print("--- 开始 AI 导航循环 (按 q 退出) ---")
    try:
        while True:
            ret, frame = cap.read()
            if not ret: break
            cv2.imshow("Vision", frame)

            # --- AI 决策层 ---
            # action = ai_brain.predict(frame)
            # vx, vyaw = action['v_x'], action['v_yaw']
            vx, vyaw = 0.05, 0.0  # 测试用死值：0.05m/s前进
            
            # --- 硬件控制层 ---
            # dog.set_velocity(vx=vx, vy=0.0, vyaw=vyaw)
            
            time.sleep(0.1) # 10Hz 频率
            if cv2.waitKey(1) & 0xFF == ord('q'): break
    finally:
        print("安全关闭系统...")
        cap.release()
        cv2.destroyAllWindows()
        # dog.set_velocity(vx=0.0, vy=0.0, vyaw=0.0) 

if __name__ == "__main__":
    main()
```

---

## 七、 日常快速启动清单

完成上述所有配置后，日常开机联调只需执行以下三步：

1. **查网络:** 确认网线连接，终端执行 `ping 192.168.5.2` 检查通信。
2. **进环境:** 打开终端输入 `conda activate dobot`
3. **跑主控:** 测试连接与动作 (确认通信正常)`python examples/e1_get_available_motions.py`
	(如果屏幕打印出一长串动作名称，说明连接成功)
	执行基础测试 (看狗能不能动)`python examples/e3_auto_state_switch.py`

---
**编写者:** [赖维轩]
**最后修订:** 2026年5月2日
