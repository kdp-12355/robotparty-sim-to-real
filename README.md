# robotparty-sim-to-real、

# RPO-Flat 平地盲行走 Sim-to-Real 调试成功总结

从 IsaacLab 仿真训练到 Orange Pi 真机部署，你已经完成了 RPO-Flat 平地盲行走策略的 Sim-to-Real 部署与初步验证。下面按实际操作顺序整理，方便后续复现、写论文和部署其他 RPO 策略时参考。

## 一、整体流程

### 1. IsaacLab 仿真训练

RPO-Flat · 平地速度条件行走

训练得到 `model_9000.pt`

### 2. 导出 ONNX

`exported/policy.onnx`

输入 780 维，输出 23 维

### 3. Orange Pi 部署

配置模型路径、观测布局、动作缩放和关节映射

### 4. 真机控制链路验证

手柄 `/joy` → `inference_node` → ONNX → 关节控制

### 5. 真机初步验证成功

新模型成功加载，推理启动，右摇杆可以触发机器人行走

## 二、训练服务器：确认模型与导出文件

### 1. 使用已复现的 RPO-Flat 模型

训练服务器上的模型路径：

```
/workspace/IsaacLab/logs/rsl_rl/rpo_flat/2026-09-16_13-36-32/
```

关键文件：

```
model_9000.pt
exported/policy.onnx
```

已有训练结果：

| 指标                  | 结果      |
| ------------------- | ------- |
| Mean reward         | +41.86  |
| Mean episode length | 996.04  |
| 训练耗时                | 约 12 小时 |

这些数据是仿真训练结果，不代表真机性能指标。

### 2. 确认 ONNX 文件

训练服务器上的 ONNX：

```
/workspace/IsaacLab/logs/rsl_rl/rpo_flat/2026-09-16_13-36-32/exported/policy.onnx
```

验证 SHA256：

Bash

```
sha256sum /workspace/IsaacLab/logs/rsl_rl/rpo_flat/2026-09-16_13-36-32/exported/policy.onnx
```

记录的哈希值：

```
113cc9a203438cd2bee9bf013aa10bf9538e09089afabec56f22db1b403f7eca
```

这是后续核对文件传输是否一致的依据。

## 三、Orange Pi：部署 ONNX 模型

### 3. 将模型传输到真机

部署目录：

```
~/roboparty_deploy
```

模型存放位置：

```
src/inference/robots/rpo/models/
```

新模型文件名：

```
policy_rpo_flat_20260916.onnx
```

验证文件：

Bash

```
cd ~/roboparty_deploy

ls -lh src/inference/robots/rpo/models/policy_rpo_flat_20260916.onnx

sha256sum src/inference/robots/rpo/models/policy_rpo_flat_20260916.onnx
```

真机记录的 SHA256 与训练服务器一致：

```
113cc9a203438cd2bee9bf013aa10bf9538e09089afabec56f22db1b403f7eca
```

这一步确认了真机使用的是目标 RPO-Flat ONNX，而不是原先的 `policy.onnx`。

### 4. 修改模型配置

配置文件：

```
src/inference/robots/rpo/configs/default.yaml
```

将模型名切换为：

YAML

```
model_names: ["policy_rpo_flat_20260916.onnx"]
```

此前备份的原配置：

```
src/inference/robots/rpo/configs/default.yaml.bak
```

可以用于恢复原来的模型配置。

## 四、核对仿真与真机的策略接口

这是 Sim-to-Real 部署中非常关键的一步。

### 5. 观测布局保持一致

RPO-Flat 的单帧观测包含 78 维：

| 观测项           | 维度 |
| ------------- | -- |
| `ang_vel`     | 3  |
| `gravity_b`   | 3  |
| `cmd_vel`     | 3  |
| `dof_pos`     | 23 |
| `dof_vel`     | 23 |
| `last_action` | 23 |
| 合计            | 78 |

使用 10 帧历史堆叠：

78×10=78078\times10=78078×10=780

因此 ONNX 输入为 `[1,780]`，输出为 `[1,23]`。

部署配置中的关键参数：

YAML

```
obs_layouts:
  - "ang_vel:3, gravity_b:3, cmd_vel:3, dof_pos:23, dof_vel:23, last_action:23"

frame_stacks: [10]
obs_stack_orders: ["frame_major"]
action_scale: [0.25]
decimation: 5
dt: 0.004
joint_num: 23
```

历史帧按旧到新的顺序排列；初始时用当前帧填充历史缓冲区。

### 6. 确认动作缩放与关节映射

部署端将策略输出转换成关节目标，关键逻辑为：

C++

```
act_[joint_idx] =
    output_buffer[i] * action_scale_[joint_idx]
    + joint_default_angle_[joint_idx];
```

其中：

* `action_scale = 0.25`

* `joint_default_angle` 是默认关节角度

* `usd2urdf` 负责仿真关节顺序到真机关节顺序的映射

因此，不能只复制 ONNX 而忽略观测顺序、动作缩放、默认关节角度和关节映射。这些配置必须与训练策略相匹配。

## 五、启动真机并验证推理

### 7. 启动部署程序

在 Orange Pi 上执行：

Bash

```
cd ~/roboparty_deploy
./tools/start_robot.sh
```

程序会启动推理与手柄相关组件。启动日志中已经观察到：

```
[inference_node] Motors initialized
[inference_node] Inference started
```

这说明电机初始化和推理启动已完成。

此前也遇到过电机离线和 CAN 通信异常；因此，不能把每次启动失败都归因于 ONNX 模型。后来原模型也出现过红灯、重启后恢复的情况，表明硬件通信稳定性应与模型问题分开排查。

### 8. 确认 ROS 2 节点和话题

执行：

Bash

```
ros2 node list
```

实际看到：

```
/inference_node
/joy_node
```

检查推理节点：

Bash

```
ros2 node info /inference_node
```

确认它订阅：

```
/joy
/cmd_vel
/joint_ref_states
```

同时发布：

```
/action
/imu
/joint_states
```

检查手柄节点：

Bash

```
ros2 node info /joy_node
```

确认它发布：

```
/joy
```

并且通过：

Bash

```
ros2 topic echo /joy
```

观察到消息时间戳持续变化，证明手柄数据流正常。

当时 `/cmd_vel` 的检查结果为：

```
Publisher count: 0
Subscription count: 2
```

这说明虽然推理节点订阅了 `/cmd_vel`，但当时没有 ROS 2 节点发布该话题。手柄控制实际走的是 `/joy` 到 `inference_node` 的直接路径。

## 六、验证真机行走行为

### 9. 用手柄控制 RPO-Flat

目前确认的控制链路是：

### 手柄右摇杆

发布 `/joy` 消息

### `subs_joy_callback()`

解析摇杆 axes，更新 `cmd_vel_`

### RPO-Flat ONNX

根据本体状态和速度指令生成 23 维动作

### 真机关节控制

转换关节顺序，叠加默认关节角度

关键代码位于：

```
src/inference/src/ros_interface.cpp
```

其中：

C++

```
void InferenceNode::subs_joy_callback(
    const std::shared_ptr<sensor_msgs::msg::Joy> msg)
```

负责处理手柄输入，例如：

C++

```
cmd_vel_[0] = std::clamp(
    msg->axes[4] * clip_cmd_[1],
    clip_cmd_[0],
    clip_cmd_[1]);
```

这属于实时摇杆速度控制：摇杆位置映射为速度指令，策略再生成对应的行走步态。

### 10. 当前验证结果与边界

| 验证项目                  | 状态     |
| --------------------- | ------ |
| ONNX 文件与训练服务器一致       | 已确认    |
| 新模型配置已切换              | 已确认    |
| `inference_node` 启动推理 | 已确认    |
| `/joy` 持续发布           | 已确认    |
| 右摇杆可以触发行走             | 已确认    |
| `/cmd_vel` 有独立发布者     | 当时没有   |
| 长时间持续行走的稳定性           | 尚未充分验证 |
| 真机复杂地形适应能力            | 尚未验证   |

你观察到的现象是：

> 右摇杆前推，机器人走一下，然后很快停止。

结合源码，当前实现是实时摇杆控制，摇杆回中后速度指令可能回到零。因此，这个现象并不能说明 RPO-Flat 不能持续行走，也不意味着模型有问题。

如果希望通过命令让它持续前进，可以切换到 `/cmd_vel` 控制模式后持续发布非零速度。不过，首次测试应在安全支撑、急停可用的条件下从低速开始；停止发布命令后不能假定机器人一定自动停止，必须主动发送零速度指令并确认响应。

## 七、最终结论

本次工作完成了以下关键环节：

* IsaacLab 中训练得到 RPO-Flat 策略并导出 ONNX。

* 将 ONNX 部署到 Orange Pi，并通过 SHA256 确认文件一致。

* 配置 780 维观测、10 帧历史堆叠、动作缩放和关节映射。

* 启动真机推理，确认 `/joy` 和 `inference_node` 的 ROS 2 通信关系。

* 使用右摇杆成功触发机器人行走。

因此，现阶段可以总结为：

RPO-Flat 平地盲行走策略已完成从仿真到真机的初步部署与功能验证，实时手柄控制链路已打通。

需要注意，严格意义上的 Sim-to-Real 性能验证还应补充多次重复测试、持续行走时长、速度跟踪误差、停止响应和真机稳定性数据。

论文中可以将本阶段描述为：

> We deployed the RPO-Flat policy exported from IsaacLab to an Orange Pi-based humanoid robot and verified real-world inference and joystick-conditioned locomotion. The policy receives proprioceptive observations and commanded velocities without exteroceptive terrain sensing.

中文对应：

> 我们将 IsaacLab 训练并导出的 RPO-Flat 策略部署至基于 Orange Pi 的仿人机器人，验证了真机推理和基于摇杆速度指令的行走功能。该策略仅使用本体感知观测和速度指令，不依赖外部地形感知传感器。
