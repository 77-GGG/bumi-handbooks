# Bumi 机器人 test_policy.py 笔记本外挂部署分步指南（含 FAQ）

> **说明**：这是部署手册成书之前的分步速查（9 步流程 + FAQ），保留下来供快速查阅。与本书正文不一致时，以 [bumi-deploy-handbook.md](bumi-deploy-handbook.md) 和 PDF 为准。

> 依据官方手册《Bumi 综合使用文档》（noetixrobotics.feishu.cn/docx/As2adPO27odPBfxQw1ucK6N4nNe）、《SDK 编译环境部署流程》（noetixrobotics.feishu.cn/docx/PP2BdXkSUoQ2VOxctJKcEdHanAg）以及本 SDK 源码（`examples/py/test_policy.py`、`config/dds.xml`、`config/bumi_ac.yaml`）整理。
>
> **适用场景**：在**自带显卡的笔记本电脑**上运行强化学习策略 `examples/py/test_policy.py`，通过**网线**直连 Bumi 机器人，属于手册中的"**外挂开发**"方式。

---

## 0. 部署架构与关键参数（先看懂再动手）

```
┌────────────────────┐        网线（Bumi 后背网口）        ┌──────────────────┐
│  笔记本（外挂主机）   │ <===============================> │ Bumi 运控板        │
│  Ubuntu 22.04 x86_64│                                    │ 192.168.55.102    │
│  静态 IP:           │      CycloneDDS 有线通信 500Hz      │ (不对用户开放)      │
│  192.168.55.101     │                                    └──────────────────┘
│  ONNX 策略推理       │
│  澳加狮手柄接收器 → USB (/dev/input/js0)                  │
└────────────────────┘
```

| 项目 | 值 | 说明 |
|---|---|---|
| 运控板 IP | `192.168.55.102` | `config/dds.xml` 中 Peer 指向它 |
| 笔记本必须配置的静态 IP | `192.168.55.101` | 运控板**只允许**与原版算力板的固定 IP 通信，外挂设备需伪装成原算力板（手册原文要求） |
| 原算力板账号 | `noetix / bumi` | 若你的 Bumi 自带算力板且在开机，会与笔记本 IP 冲突，需先关闭算力板 |
| 通信中间件 | CycloneDDS 0.11.0，禁用组播，Peer 静态指定 | 配置文件 `config/dds.xml` |
| 控制频率 | 500Hz（2ms 控制环），策略推理 50Hz（decimation=10） | `test_policy.py` 主循环 + `bumi_ac.yaml` |
| 策略文件 | `policy/policy.onnx`（21 自由度跑步策略） | 模型参数读取 `config/bumi_ac.yaml` 的 `run:` 段 |
| 手柄 | 澳加狮手柄，接收器插**笔记本** USB，设备节点 `/dev/input/js0` | 运控板接口插不了接收器，手柄数据必须从运行程序的机器直接读取 |

**重要前提**：SDK 只支持 Linux（`build.sh`、`.so` 库、`/dev/input/js0`）。手册要求 Ubuntu 22.04。Windows 笔记本请装双系统 Ubuntu 22.04（不推荐 WSL2：网口/手柄 USB 直通困难和 500Hz 实时性无法保证）。

**调试全程请使用机器人吊架**（手册原话：调试过程中，请全程使用机器人吊架，以免错误程序失控造成损失）。

---

## 第 1 步：笔记本安装 Ubuntu 22.04 与基础编译环境

**做什么**
1. 笔记本安装 Ubuntu 22.04 LTS（x86_64），接通电源、关闭休眠。
2. 安装编译依赖（手册《SDK 编译环境部署流程》+ SDK README）：

```bash
sudo apt update
sudo apt install -y cmake g++ build-essential libyaml-cpp-dev libeigen3-dev \
    libboost-all-dev libfmt-dev pybind11-dev python3-dev python3-pip
pip3 install pybind11
```

3. 若 `cmake --version` 低于 3.16（22.04 默认 3.22 一般够用），按手册装 cmake 3.26.3：解压后 `sudo mv cmake-3.26.3-Linux-x86_64 /opt/`，再 `sudo ln -s /opt/cmake-3.26.3-Linux-x86_64/bin/* /usr/bin/`。

**预期结果**
- `cmake --version` ≥ 3.16；`g++ --version` ≈ 11.4；`python3 --version` 为 3.10。
- 编译 SDK 不需要 ROS 环境（手册明确说明）。

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| apt 安装包 404 / 锁定 | 先 `sudo apt update`；有进程占用时等几秒重试 |
| cmake 版本过低，编译报 `CMake 3.16 required` | 按上面方法手动安装 cmake 3.26.3 |
| 只有 Windows 系统 | SDK 的库（`lib/x86_64/libnoetix_sdk.a`）与 `/dev/input/js0` 均为 Linux 专用，必须装 Ubuntu（双系统最佳），WSL2 直通网口/手柄并保证 500Hz 实时性非常困难，不建议 |

---

## 第 2 步：获取 SDK 并安装 Python 依赖

**做什么**
1. 在笔记本上获取 SDK（二选一）：

```bash
git clone https://github.com/Noetix-Robotics/noetix_sdk_bumi.git
# 或把本目录（含 build.sh、config、dds、examples、lib、policy、pybind）整体拷到笔记本，如 ~/noetix_sdk_bumi
```

2. 安装 Python 依赖（test_policy.py 需要）：

```bash
pip3 install onnxruntime numpy pyyaml
```

3. 确认关键文件存在：`policy/policy.onnx`、`config/bumi_ac.yaml`（须含 `run:` 段）、`config/dds.xml`。

**预期结果**
- `python3 -c "import onnxruntime, numpy, yaml; print('ok')"` 输出 `ok`。
- SDK 目录结构完整（build.sh / config / dds / examples / lib / policy / pybind）。

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| `pip3 install` 权限/externally-managed 报错 | 用 `pip3 install --user ...` 或虚拟环境 `python3 -m venv venv && source venv/bin/activate` |
| 换用自己的策略模型时缺 `policy/policy.onnx` | test_policy.py 固定加载 `<cwd>/policy/policy.onnx`，文件名/路径不能变；模型需内嵌 `joint_names / joint_stiffness / joint_damping / default_joint_pos / action_scale` 元数据，且与 `bumi_ac.yaml` 的 `run:` 段尺寸参数一致（actions_size=21、observations_size=71、stack_size=5） |
| 显卡（GPU）用不上？ | 正常现象：本例程 onnxruntime 显式使用 `CPUExecutionProvider`，模型很小 CPU 足够。想用 GPU 装的是 `onnxruntime-gpu`，并把 `load_model()` 中 providers 改为 `["CUDAExecutionProvider","CPUExecutionProvider"]`；注意 GPU 首帧初始化/调度抖动可能影响 500Hz 实时性，实机前先用吊架验证 |

---

## 第 3 步：编译 SDK（生成 Python 绑定 lowcontrol_py）

**做什么**
1. 进入 SDK 根目录执行：

```bash
cd ~/noetix_sdk_bumi
./build.sh
```

2. 编译 C++/pybind 需要 onnxruntime 开发包（CMakeLists 中 `find_package(onnxruntime REQUIRED)`）。若 cmake 报找不到，从 onnxruntime GitHub Releases 下载 `onnxruntime-linux-x64-*.tgz` 解压到 `/opt`，再手动编译：

```bash
mkdir -p build && cd build
cmake .. -DCMAKE_PREFIX_PATH=/opt/onnxruntime-linux-x64-1.17.0/lib/cmake
make -j8
```

**预期结果**
- 编译无报错；`build/` 目录生成 `lowcontrol_py*.so`（还有 `highcontrol_py*.so`、`mediacontrol_py*.so`）。
- 根目录生成 `example_low`、`example_high`、`example_policy_walk` 可执行文件。
- 验证：`cd ~/noetix_sdk_bumi && python3 -c "import sys; sys.path.append('./build'); import lowcontrol_py; print('ok')"` 输出 `ok`。

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| 运行时报 `error while loading shared libraries: libssl.so.1.1` | 手册常见问题①：Ubuntu 22.04 缺 libssl1.1。从手册下载对应架构 deb 并安装（x86_64）：`sudo dpkg -i libssl1.1_1.1.1f-1ubuntu2_amd64.deb` |
| `find_package(onnxruntime REQUIRED)` 失败 | 未装 onnxruntime C++ 开发包，按本步骤设置 `CMAKE_PREFIX_PATH` 指向其 cmake 目录 |
| 链接/编译报与 gcc 相关错误 | 手册要求 gcc 11.4（22.04 默认即 11.4），勿混用其他版本工具链 |
| 找不到 `libnoetix_sdk.a` / DDS 库 | 仓库不完整：`lib/x86_64/` 与 `dds/x86_64/` 必须存在；架构要匹配（笔记本用 x86_64，不要用 aarch64 的 docker 镜像 noetix_dds——该镜像仅限 arm64） |

---

## 第 4 步：配置网线直连（静态 IP 192.168.55.101）

**做什么**
1. 网线一端插笔记本有线网口，另一端插 **Bumi 后背的网口**（即原算力板接口；运控板通过内部交换机走有线）。
2. 给笔记本有线网卡配静态 IP（示例网卡名 `enp3s0`，用 `ip addr` 查实际名称）：

```bash
sudo ip addr add 192.168.55.101/24 dev enp3s0
sudo ip link set enp3s0 up
```

（永久生效可用 nmcli/netplan 配置，地址 192.168.55.101/24，不配网关。）
3. 机器人开机后（见第 6 步）测试连通：

```bash
ping 192.168.55.102
```

**预期结果**
- `ping 192.168.55.102` 有回复，丢包率低。
- `config/dds.xml` 无需修改：其 Peer 已指向 `192.168.55.102`。

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| ping 不通 | ① 笔记本 IP 不是 192.168.55.101（运控板只认这个 IP）；② 网线没插到 Bumi 后背正确网口；③ 机器人未开机/运控程序未启动完；④ 笔记本其他网卡（WiFi）路由抢占了 192.168.55.0 网段，可临时关 WiFi 或加明细路由 |
| IP 冲突提示 | 你的 Bumi 若自带算力板且在运行，它已占用 192.168.55.101。外挂方式下笔记本要"顶替"算力板：先让算力板下电/关闭，或直接改为在算力板上跑（`ssh noetix@192.168.55.101`，密码 bumi） |
| DHCP 自动获取了别的地址 | Bumi 侧没有 DHCP 服务，必须手工配静态 IP |
| NetworkManager 反复改写 IP | 用 nmcli 建一个 manual/静态连接配置，避免每次插拔变化 |

---

## 第 5 步：插入手柄接收器并验证 /dev/input/js0

**做什么**
1. 把**澳加狮手柄的 USB 接收器**插到笔记本 USB 口，打开手柄电源并配对。
2. 验证设备节点与数据：

```bash
ls -l /dev/input/js0
sudo apt install -y joystick        # 提供 jstest 工具
jstest /dev/input/js0               # 摇动摇杆/按键，观察数值变化
```

**预期结果**
- `/dev/input/js0` 存在；jstest 中 axes 在 [-1,1] 变化、按键位翻转。
- test_policy.py 初始化即调用 `aoliondriver.init("/dev/input/js0", 115200)`，读取的正是这个节点。

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| 没有 `/dev/input/js0` | 接收器没插到笔记本（不能指望机器人侧转发手柄数据——带算力板机型运控板插不了接收器，DDS 拿不到手柄数据）；换 USB 口，`lsusb` 确认设备枚举；手柄未开机/未配对 |
| 普通 Python 进程打不开设备（权限） | `sudo usermod -aG input $USER` 后重新登录，或临时用 `sudo` 运行程序 |
| jstest 有数据但程序里 `getremotedata()` 一直为 None | 设备不是标准 joystick 协议或节点不是 js0（多手柄/模拟器会变成 js1），确认 `/dev/input/` 下唯一手柄节点；代码写死 `js0`，必要时改 `init()` 第一个参数 |

---

## 第 6 步：机器人开机与安全放置（先于启动程序）

**做什么**（手册"启动设备及注意事项"）
1. 吊架安装/平整防滑地面，清出约 3m×3m 无杂物区域。
2. 电池插入后背电池仓，确认卡扣锁定。
3. 短按一次电池开关，确认电池指示灯亮起；随后**长按电池开关直至出现流水灯效果**，等待完成开机。
4. 开机后 Bumi 自动进入**全身使能模式**，所有关节自动回零位——过程中**不要触碰、阻挡关节**；也不要把关节卡在限位位置开机（有烧机风险）。
5. 让机器人保持趴卧（面朝上平躺）或吊挂姿态，等待程序控制。

**预期结果**
- 机器人开机完成，关节自动回零，处于使能趴卧状态，网络可达（第 4 步 ping 通）。

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| 开机时关节被地面/杂物卡住、异响 | 立即长按电池开关关机检查；开机姿势务必面朝上平躺、四肢自然摆放，脚底无异物 |
| 关节回零时机器人被拽动 | 全身使能回零过程请勿干预，否则初始化失败需重启 |
| 电池电量低/告警 | 低电量会导致运控异常甚至失控，先充满电再跑策略；程序里也可从 `bms_data.battery_soc` 读取电量 |

---

## 第 7 步：启动 test_policy.py

**做什么**
1. **必须从 SDK 根目录启动**（程序用相对路径找 `build/`、`config/dds.xml`、`policy/`）：

```bash
cd ~/noetix_sdk_bumi
python3 examples/py/test_policy.py
```

2. 保持终端在前台运行，不要让笔记本休眠/降频（建议插电源、设置 Performance 电源模式）。

**预期结果**（终端依次打印）
```
cur path is file:///home/<用户>/noetix_sdk_bumi/config/dds.xml
Load Onnx model from path: /home/<用户>/noetix_sdk_bumi/policy/policy.onnx
Load Onnx run model successfully !!!
```
- 程序常驻运行（500Hz 主循环），此时机器人**还没动作**（处于 DEFAULT 阻尼模式），等待手柄指令。
- 之后每次模式切换、以及进入策略模式后的第一帧观测，都会在终端打印日志（见第 8 步）。

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| `ModuleNotFoundError: lowcontrol_py` | ① 没编译（回第 3 步）；② 不在 SDK 根目录运行——程序把 `./build` 加进 sys.path，`cd` 到根目录再跑 |
| `FileNotFoundError: config/dds.xml` 或 CYCLONEDDS 连不上 | 同上，工作目录不对；确认 `CYCLONEDDS_URI` 打印为 `file://<SDK根目录>/config/dds.xml` |
| onnx 加载失败 / `policy_session is None` | `policy/policy.onnx` 缺失或损坏；`pip3 install onnxruntime` 未装 |
| yaml 报 KeyError `run` | `config/bumi_ac.yaml` 缺 `run:` 段或文件被改动 |
| 程序在跑、打印正常，但按手柄机器人无反应 | 两类断点：① DDS 没连上运控板（硬件状态回调没数据，回第 4 步查网络）；② 手柄没数据（`getremotedata()` 返回 None，回第 5 步 jstest 验证） |
| 控制卡顿/机器人抖动 | 笔记本节能降频或后台负载高：插电源、性能模式、关闭占 CPU 的程序；500Hz 实时性优先 |

---

## 第 8 步：手柄操作流程（状态机：阻尼 → 趴卧 → 站立 → 策略行走）

手柄按键与 button 索引对照（手册）：RB=1，**LB=2**，**X=5**，Y=6，B=7，A=8，**"+"=9**，**"−"=10**，无标名键=11，左摇杆按下=12。

程序内模式：`DEFAULT`（阻尼）→ `LIE`（趴卧回零）→ `STAND`（站立准备）→ `USERMODE`（策略行走），逐级切换：

| 顺序 | 按键 | 动作（做什么） | 预期结果 |
|---|---|---|---|
| ① | 按 **"+"** 一次 | 开始控制：从当前姿态缓慢插值到趴卧零位 | 终端打印 `start control`，机器人关节缓慢回零趴下 |
| ② | **LB + "−"** 同时按 | 趴卧 → 站立准备模式（四肢微曲站立，约 2 秒插值） | 终端打印 `STAND2LIE`（示例代码打印文案有误，实际是**进入 STAND 站立**），机器人自起立站稳 |
| ③ | **LB + X** 同时按 | 站立 → **策略控制行走模式**（ONNX 策略接管 21 关节） | 终端打印 `TO USERWALK MODE`，随后打印一屏观测数值（71×5 个数）；机器人姿态由策略维持 |
| ④ | 摇杆 | 控制运动：**axes[1]（左摇杆前后）= 前进/后退，axes[0] = 左右转向** | 前进最大约 0.5 m/s（scalex=0.5），摇杆推程小于 60% 时前进有死区（cmd<0.3 置零）；转向 |yaw|≤1.0、死区 0.3 |
| ⑤ | **无标名键(11)** | 行走 → 回站立 | 终端打印 `WALK2STAND`，策略退出、保持站立 |
| ⑥ | 再按 **"+"** | 停止控制，回到阻尼模式 | 终端打印 `stop control`，全身关节失力进入阻尼（手册：全身关节回 0 位进入阻尼模式） |

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| 按 "+" 无反应 | 手柄数据没到程序（jstest 验证、检查接收器）；或程序还没收到硬件状态（DDS 未连通，回第 4/7 步） |
| 组合键（LB+−、LB+X）不生效 | 必须**先按住 LB 再按第二个键**（代码检测 `button[2]==1 且另一键==1` 的边沿），逐个尝试并保持半秒 |
| 站立过程晃动/失败 | 地面不平/打滑、吊架吊带别太松；确保站立插值过程没有外物阻挡 |
| 终端突然打印 `Fall Protection Triggered (UserMode)!` 并全身变软 | 策略摔倒保护：机器人倾角超限（projectedGravity 判定），自动回 DEFAULT 阻尼。恢复流程：扶正机器人 → 按 "+"（打印 `stop control`）→ 再按 "+"（`start control`）→ LB+"−" 站立 → LB+X 重新进策略 |
| 行走中猛冲/失控 | 立刻按无标名键(11)回站立或按 "+" 进阻尼；首次测试务必用吊架并小幅推杆 |
| 把机器人拎起来后关节仍持续输出 | 手册禁止：行走步态下勿将机器人拎起双脚悬空（电机异常受力损坏风险）；先按键停止再搬运 |

---

## 第 9 步：停止程序与关机

**做什么**
1. 先按 **"+"** 让机器人 `stop control`（进阻尼、关节失力），**再** `Ctrl+C` 退出程序（避免在策略输出中直接断开）。
2. 关机（手册）：APP 执行一键收纳 → 扶稳机器人坐于地面 → 长按电池开关直至完全关机 → 按卡扣取出电池充电。

**预期结果**
- 程序干净退出，无报错堆栈；机器人关节阻尼松弛，可安全搬运。

**可能遇到的问题**
| 问题 | 原因/解决办法 |
|---|---|
| 直接 Ctrl+C 后机器人保持最后一个力矩指令短暂僵硬 | SDK 发送线程随进程退出即停，运控侧有安全处理；但规范做法仍是先 "+" 停控再退出 |
| 一键收纳失败 | 周围 3m×3m 有杂物、地面打滑；先清理再重试（手册禁止杂物环境使用收纳/起身） |

---

## 附 A：模式/按键速查卡

```
开机(趴卧使能) → [+]=start control(LIE) → [LB & −]=STAND → [LB & X]=USERMODE(策略行走)
                     ↑                        ↑                    |
                     |                     [键11]=WALK2STAND ←────┘
                     └── 再按[+]=stop control(DEFAULT 阻尼)
摇杆：axes[1] 前进/后退（≤0.5m/s，<60%行程死区） | axes[0] 转向（|yaw|≤1.0，死区0.3）
摔倒保护：自动回 DEFAULT 阻尼，终端打印 Fall Protection Triggered
```

## 附 B：安全红线（摘自手册）

1. 调试**全程使用吊架**，防止程序错误导致失控损坏。
2. 开机时关节勿处限位、勿阻挡回零；面朝上平躺、地面平坦防滑、脚底无异物。
3. 起身/收纳动作波及范围 3m×3m 内不得有杂物；禁止运动中人为干扰（策略为盲走）。
4. 禁止快速反复切换模式；禁止行走时拎起机器人双脚悬空。
5. 电池未正确安装禁止开机；低电量勿跑策略。
6. 同一时间只允许一个控制源（APP/手柄直连/本 SDK 程序不要同时控制）。

## 附 C：故障排查总表

| 现象 | 首查 | 次查 |
|---|---|---|
| 编译失败 | cmake/gcc 版本、onnxruntime 包 | `./build.sh clean` 后重编 |
| import 失败 | 是否在 SDK 根目录、build/*.so 是否存在 | 第 3 步 |
| 程序正常但机器人无反应 | `ping 192.168.55.102`、笔记本 IP 是否 192.168.55.101 | jstest 手柄、DDS 配置 |
| libssl.so.1.1 报错 | 装手册提供的 x86_64 deb | 第 3 步 FAQ |
| 机器人软掉/Fall Protection | 扶正后按恢复流程 | 第 8 步 FAQ |
| 运动卡顿 | 电源模式/后台负载/网卡丢包（`ping -f` 简单压测） | 换更短网线、固定网卡 |
