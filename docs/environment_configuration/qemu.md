# 在arm上开发


# QEMU

一个另类替代交叉编译的方法

QEMU 是一款免费开源的模拟器。我们将使用它来模拟 arm 架构，以便在 x86 电脑上编译出 arm 架构的程序

```bash
sudo apt install qemu-user-static
docker run --rm --privileged multiarch/qemu-user-static:register --reset
docker pull --platform arm64 osrf/ros:humble-desktop
docker run -it --platform arm64 \
    -v  ~/armHumble:/home \
    --network host \
    -v /usr/bin/qemu-aarch64-static:/usr/bin/qemu-aarch64-static \
    --name armHumble \
    arm64v8/ros:humble /bin/bash
```

# ARM 交叉编译与部署（x86 → Jetson Xavier NX）

目标机：Jetson Xavier NX + JetPack 5（宿主 Ubuntu 20.04；实际运行代码的容器镜像为
Ubuntu 22.04；CUDA 11.4、TensorRT 8.5.2、sm_72）。
开发机：x86_64 Ubuntu。本文以 `rmvision2026` 工作区为例。

与 QEMU 模拟编译不同，这里在 x86 上用 **aarch64 编译器 + 目标 sysroot** 直接产出 aarch64
二进制，速度接近本机编译；代价是首次需要准备交叉工具链和 sysroot。

> 注意：程序最终运行在 Xavier 的 Docker 容器里，容器基础镜像是 **Ubuntu 22.04**，
> 所以交叉 sysroot 取自该镜像（glibc 2.35）；宿主机的 Ubuntu 20.04 不直接加载这些二进制。
> 宿主机只提供 CUDA / TensorRT / 相机设备。

> 角色说明：**开发机 host** 负责交叉编译；**开发机上的 arm64 容器（QEMU）**只用于预构建
> `rm_interfaces`；**Xavier** 负责补编 TensorRT 和运行。

## 0. 命令执行位置和路径约定

本文所有仓库相关路径均相对于当前仓库根目录，不写死用户名或仓库绝对路径。

| 命令类型 | 执行位置 |
|---|---|
| 交叉编译、sysroot、rsync 准备 | x86 开发机仓库根目录 |
| Docker Compose 和 `docker exec` | Xavier 宿主机仓库根目录 |
| TensorRT 重编、ROS 启动和验证 | Xavier 容器内的工作空间根目录 |

在 x86 开发机仓库根目录设置：

```bash
cd rmvision2026

SYSROOT="$PWD/.cross/sysroot"
TOOLCHAIN="$PWD/.cross/toolchain"
DEPLOY="$PWD/deploy"
```

这些变量由当前工作目录生成，不包含硬编码的机器路径。

## 1. 一次性准备

### 1.1 开发机：安装交叉编译器

```bash
sudo apt install gcc-11-aarch64-linux-gnu g++-11-aarch64-linux-gnu
```

必须使用与 sysroot 匹配的 GCC 11：sysroot 来自 arm64 运行镜像（Ubuntu 22.04），
其 glibc 为 2.35、libstdc++ 为 6.0.30。GCC 14 的 libstdc++ 依赖 `GLIBC_2.36+`，
链接或运行会失败。

```bash
RM_CROSS_TOOLCHAIN_PREFIX="$TOOLCHAIN" \
./scripts/cross/install_toolchain_local.sh

export PATH="$TOOLCHAIN/usr/bin:$PATH"
export RM_CROSS_HOST_LIBS="$TOOLCHAIN/usr/lib/x86_64-linux-gnu"
```

### 1.2 开发机：导出目标 sysroot

sysroot 是 ARM64 运行镜像 rootfs 的一次性快照，包含 ROS Humble、OpenVINO、
OpenCV、Ceres 等。它不是从 Xavier 实时复制 `/usr` 和 `/opt`。

```bash
RM_SYSROOT="$SYSROOT" \
./scripts/cross/setup_sysroot.sh --force
```

导出内容位于：

```text
.cross/sysroot/usr
.cross/sysroot/lib
.cross/sysroot/opt
```

`setup_sysroot.sh` 会优先检查本地 `slirute/qidian:arm64v8` 镜像。只有镜像不存在时
才尝试从 Docker Hub 拉取。

脚本会额外做两件必要的事：

- 把 sysroot 内的绝对符号链接改成相对链接（如 `libm.so -> /lib/...` 在 x86 上会指向
  开发机自己的 `/lib`，交叉链接时找不到目标库）；
- 把 ROS CMake 导出里写死的 `/usr/lib/aarch64-linux-gnu/libpython3.10.so` 重定位到
  sysroot 内，避免链接时报找不到。

### 1.3 Xavier：时钟、网络、容器

时钟必须正确，否则 make 会报 `Clock skew`，甚至漏编文件。先判断 Xavier 是否有外网：


#### Xavier 有外网（推荐）

JetPack 5 默认启用 `systemd-timesyncd`，网络可达就会自动通过 NTP 同步：

```bash
sudo timedatectl set-ntp true
sudo timedatectl set-timezone Asia/Shanghai
sudo systemctl restart systemd-timesyncd
sleep 5
timedatectl status
```

期望看到：

```text
Local time: ... CST
System clock synchronized: yes
NTP service: active
```

如果仍然是 `System clock synchronized: no`，说明 NTP（UDP 123）被网络阻断，检查：

```bash
journalctl -u systemd-timesyncd -n 30 --no-pager
timedatectl show-timesync --all
```

外网正常但 NTP 被封时，可以改用国内 NTP 服务器（编辑
`/etc/systemd/timesyncd.conf`）：

```ini
[Time]
NTP=ntp.aliyun.com ntp1.aliyun.com
FallbackNTP=pool.ntp.org
```

```bash
sudo systemctl restart systemd-timesyncd
timedatectl status
```

#### 情况 B：Xavier 无外网（只在局域网）

从开发机通过 SSH 把开发机的当前时间写给 Xavier：

```bash
ssh -t qidian@<XAVIER_IP> "sudo date -s '$(date '+%Y-%m-%d %H:%M:%S')'"
```

然后在 Xavier 上写回 RTC（板子有 RTC 电池时会持久化）：

```bash
sudo hwclock -w
date
```

没有 RTC 电池时，每次上电后都要重新对时；此时 `timedatectl` 显示
`System clock synchronized: no` 是正常的，只要 `date` 的本地时间正确即可。

#### 网络与容器检查

```bash
hostname -I                       # Xavier 的局域网 IP，开发机用它 SSH/rsync
docker info | grep -i runtime     # 应包含 nvidia
docker images | grep qidian       # arm64 镜像是否存在

docker image inspect \
  --format '{{.Architecture}}' \
  slirute/qidian:arm64v8
```

最后一条应输出 `arm64`。如果 Docker Hub 拉取超时但本地镜像存在，直接继续，
不要因为 `docker pull` 失败而中断部署。

验证 ARM64 执行时覆盖镜像 entrypoint，避免 `setupvars.sh` 消耗参数：

```bash
docker run --rm \
  --platform linux/arm64 \
  --entrypoint /bin/uname \
  slirute/qidian:arm64v8 -m
```

应输出 `aarch64`。如果本地没有镜像且网络拉取持续失败，可以在可联网机器执行
`docker save slirute/qidian:arm64v8 -o qidian-arm64v8.tar`，再把 tar 文件放到
仓库根目录，在此执行：

```bash
docker load -i qidian-arm64v8.tar
```


#### TensorRT / cuDNN 运行库

JetPack 5 上 TensorRT 的 dev 头文件通常由 `libnvinfer-dev` 安装到
`/usr/include/aarch64-linux-gnu/`，动态库在 `/usr/lib/aarch64-linux-gnu/`。
（`/usr/src/tensorrt` 不一定存在，不要假设。）

先用包管理器确认实际位置：

```bash
dpkg -S NvInfer.h
# 例如：libnvinfer-dev: /usr/include/aarch64-linux-gnu/NvInfer.h
```

容器不能整目录挂载宿主机的 `/usr/lib/aarch64-linux-gnu`（会混入宿主 glibc），
所以先在宿主机准备独立的 `include`/`lib` 目录：

```bash
sudo mkdir -p /usr/local/tensorrt/include /usr/local/tensorrt/lib /usr/local/cudnn/lib

# 头文件（NvInfer.h / NvOnnxParser.h / NvInferPlugin.h 等）
sudo cp -a /usr/include/aarch64-linux-gnu/Nv*.h /usr/local/tensorrt/include/

# TensorRT 动态库（含 builder resource / onnxparser / plugin）
sudo cp -a /usr/lib/aarch64-linux-gnu/libnv*.so* /usr/local/tensorrt/lib/

# cuDNN（TensorRT builder 默认只用 cuDNN tactic）
sudo cp -a /usr/lib/aarch64-linux-gnu/libcudnn*.so* /usr/local/cudnn/lib/
```

如果头文件不在上述目录，用 `find / -name NvInfer.h 2>/dev/null` 定位后替换复制路径。

**必须用 `cp` 而不是 `ln -s`**：指向 `/usr/lib/aarch64-linux-gnu/...` 的软链接在容器内
是断链，链接或运行时都会失败。

容器内验证：

```bash
ls /usr/local/tensorrt/include/NvInfer.h
ls /usr/local/tensorrt/lib/libnvinfer.so
ls /usr/local/cudnn/lib/libcudnn.so*
```

## 2. 开发机：交叉编译

```bash
# apt 安装的工具链
RM_SYSROOT="$SYSROOT" \
RM_CROSS_JOBS=2 \
RM_CROSS_MAKE_JOBS=1 \
./scripts/cross/build_arm64.sh

# 本地解压的工具链
PATH="$TOOLCHAIN/usr/bin:$PATH" \
RM_CROSS_HOST_LIBS="$TOOLCHAIN/usr/lib/x86_64-linux-gnu" \
RM_SYSROOT="$SYSROOT" \
RM_CROSS_JOBS=2 \
RM_CROSS_MAKE_JOBS=1 \
./scripts/cross/build_arm64.sh
```

脚本执行流程：

> 内存较小的开发机（例如 16 GB）建议降低并行度，否则 `cc1plus` 可能被 OOM 杀掉：
> ```bash
> RM_SYSROOT="$PWD/.cross/sysroot" \
> RM_CROSS_JOBS=2 RM_CROSS_MAKE_JOBS=1 ./scripts/cross/build_arm64.sh
> ```
> 默认是 4 个包并发、单包 `make -j2`；脚本通过 `MAKEFLAGS` 覆盖 colcon 默认的
> `-j$(nproc)`，避免“包并发数 × 单包并行数”把内存打满。

1. 用 QEMU arm64 容器预构建 `rm_interfaces`。ROS 2 的 `rosidl` 代码生成与宿主机 Python
   耦合较深，消息包用 QEMU 原生构建一次最稳（约 1.5 分钟）。修改 `.msg/.srv` 后使用
   `--rebuild-interfaces`。
2. 其余 13 个包用 `cmake/aarch64-toolchain.cmake` 真交叉编译。
3. 生成部署副本，并把其中的构建机 sysroot 路径改写成设备路径。

产物：

- `.cross/sysroot/ros_ws/install`：**staging 树**，保留开发机路径，供下次增量构建
  使用（不要部署它，否则 colcon 会去 source 不存在的 `/ros_ws/install`）；
- `.cross/sysroot/ros_ws/install-deploy`：**部署副本**，内嵌路径已改为设备上的
  `/ros_ws/install`。

关键机制：

- `CMAKE_SYSROOT` + `CMAKE_FIND_ROOT_PATH_MODE_*=ONLY`：只在 sysroot 里找头文件和库，
  不会误用 x86 的 OpenCV/ROS；
- `CMAKE_INSTALL_PREFIX=/ros_ws/install` + `CMAKE_STAGING_PREFIX=<sysroot>/ros_ws/install`：
  文件落在开发机，config/RPATH 写设备路径；
- `-DRM_ENABLE_TENSORRT=OFF`：交叉构建没有 aarch64 `nvcc`，TRT 部分在设备上补编；
- 链接参数把 sysroot 的 GCC 11 库目录放在最前，避免链接到 cross 包自带的新版 libstdc++；
- `PYTHONPATH` 指向 sysroot 的 `python3.10/site-packages`，供 `ament_cmake` 配置阶段使用。

编译完成后校验：

```bash
S="$SYSROOT/ros_ws/install-deploy"

file "$S/lib/librm_utils.so"
# ELF 64-bit LSB shared object, ARM aarch64

aarch64-linux-gnu-readelf -d "$S/lib/librm_inference.so" | grep NEEDED
# 交叉产物只有 libopenvino，不应有 libnvinfer
```

生成独立部署目录：

```bash
mkdir -p "$DEPLOY"
rm -rf "$DEPLOY/install"
cp -a "$SYSROOT/ros_ws/install-deploy" "$DEPLOY/install"
```

## 3. 开发机：同步到 Xavier

```bash
TARGET=<user>@<jetson-ip>
ssh "$TARGET"
```

同步源码（`--delete` 会删除设备上开发机已不存在的旧文件，例如重构前的
`tensorrt_inference.cpp`）：

```bash
rsync -a --delete --no-times \
  --exclude .git --exclude build --exclude install --exclude log \
  --exclude qd2026-log --exclude MvSdkLog --exclude .pixi \
  --exclude 'record*/' --exclude '*.bag' --exclude '*.mcap' \
  ./ "$TARGET:rmvision2026/"
```

同步交叉部署副本（用交叉产物替换设备旧 `install/`，避免 merge/非 merge 布局混用）：

```bash
rsync -a --delete \
  "$DEPLOY/install/" \
  "$TARGET:rmvision2026/install/"
```

## 4. Xavier：启动容器

```bash
cd rmvision2026
rm -rf build log

docker compose \
  -f docker-compose.yaml \
  -f docker-compose.arm64.yaml \
  up -d --force-recreate --remove-orphans

docker exec -it rv_devel_ bash
```

首次或镜像不存在时先构建：

```bash
docker compose -f docker-compose.yaml -f docker-compose.arm64.yaml build
```

`docker-compose.arm64.yaml` 中的动态库路径必须同时包含 OpenVINO、TBB、
TensorRT、cuDNN、CUDA 和 Jetson tegra 目录：

```yaml
    environment:
      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: all
      LD_LIBRARY_PATH: /usr/local/runtime/lib/aarch64:/usr/local/runtime/3rdparty/tbb/lib:/usr/local/tensorrt/lib:/usr/local/cudnn/lib:/usr/local/cuda-11.4/lib64:/usr/lib/aarch64-linux-gnu/tegra
```

容器系统库路径必须使用绝对路径；仓库和部署路径仍使用相对路径。

## 5. Xavier 容器内：补编 TensorRT 部分

```bash
source /usr/local/setupvars.sh
source ./install/setup.bash

rm -rf build/rm_interfaces build/rm_utils
colcon build \
  --merge-install \
  --symlink-install \
  --parallel-workers 2 \
  --packages-select rm_interfaces rm_utils \
  --cmake-args \
    -DRM_ENABLE_TENSORRT=ON \
    -DRM_CUDA_ARCHITECTURES=72 \
    -DBUILD_TESTING=OFF \
  --event-handlers console_direct+

source ./install/setup.bash
```

只编这两个包的原因：

- 其余包继续使用交叉产物；
- 交叉构建固定关闭 TensorRT，只有设备上有 `nvcc` + TensorRT；
- `--merge-install` 与交叉产物的安装布局保持一致。

编译日志中应出现 `tensorrt_backend.cpp`、`openvino_backend.cpp`，而不是旧的
`tensorrt_inference.cpp`。

## 6. 启动与验证

```bash
source /usr/local/setupvars.sh
source ./install/setup.bash
ros2 launch rm_bringup bringup_SingleProcess.launch.py
```

检查：

```bash
# 日志应出现：推理后端: 请求 AUTO -> tensorrt:sm72:fp16

# 首次构建引擎，之后直接加载
ls -l qd2026-log/tensorrt/
# 0526_<hash>_trt8.5.2_sm72_fp16_cudnn.engine
# buff_repvgg_<hash>_trt8.5.2_sm72_fp16_cudnn.engine

# 相机图像应为 bgr8
ros2 topic hz /camera_info

# 动态库解析
ldd ./install/lib/librm_inference.so | grep -E "openvino|nvinfer|not found"
```

首次启动时，`armor_detector` 会在初始化阶段同步构建 TensorRT 引擎。期间出现：

```text
service /armor_detector/set_mode not available, waiting again...
service /armor_solver/set_mode not available, waiting again...
```

是正常的；此时后续组件尚未加载。

构建过程中目录可能暂时不存在，只有
`builder->buildSerializedNetwork(...)` 成功返回后才会创建目录并写入文件。

在 Xavier 宿主机仓库根目录监控：

```bash
docker exec rv_devel_ \
  ps -eo pid,pcpu,pmem,etime,cmd --sort=-pcpu | head

timeout 3 tegrastats --interval 1000
```

若 `component_container_mt` 持续保持较高 CPU，且日志没有 TensorRT 错误，应继续等待。
Xavier NX 首次 cuDNN tactic 构建可能需要 10～30 分钟。

一次实测参考：

```text
TensorRT 推理平均：11.58 ms
完整检测平均：12.61 ms
图像到解算输出平均：34.10 ms
图像到解算输出 P95：51.10 ms
检测循环：约 58～60 FPS
```

## 7. 日常迭代

| 改动内容 | 开发机 | Xavier |
|---|---|---|
| 只改检测器/解算器等 C++ | `build_arm64.sh` → rsync `install-deploy/` | 重启 bringup |
| 改 `rm_utils` | 同上 | 容器内重编 `rm_utils` |
| 改 `rm_interfaces` 的 msg/srv | `build_arm64.sh --rebuild-interfaces` → rsync `install-deploy/` | 容器内重编 `rm_interfaces rm_utils` |
| 改 launch/参数 YAML | rsync 源码 | 重启 bringup |

设备侧重编命令固定为：

```bash
source /usr/local/setupvars.sh
source ./install/setup.bash
colcon build \
  --merge-install \
  --symlink-install \
  --parallel-workers 2 \
  --packages-select rm_interfaces rm_utils
source ./install/setup.bash
```

## 8. 常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| `cc1plus: 已杀死` / `Killed` | 并行度过高，OOM | `RM_CROSS_JOBS=2 RM_CROSS_MAKE_JOBS=1 ./scripts/cross/build_arm64.sh` |
| colcon 报 `The build time path "/ros_ws/install" doesn't exist` | staging 安装树被改成了设备路径 | `rm -rf .cross/sysroot/ros_ws/{install,build,log,install-deploy}` 后重新构建 |
| 设备上 `source` 报 `not found: <构建机sysroot>/...` | 把 staging `install/` 同步上去了 | 只同步 `install-deploy/`，不要同步 `install/` |
| 编译了 `tensorrt_inference.cpp` | Xavier 源码是旧的 | 用 `rsync --delete` 同步源码 |
| `Clock skew detected` | Xavier 时钟错误 | 手动 `sudo date -s`，或从开发机 SSH 对时 |
| `libpython3.10.so` 找不到 | sysroot 内 ROS 导出路径未重定位 | 重新执行 `setup_sysroot.sh` |
| `libopcodes-*.so` 找不到 | 本地解压工具链缺少 host 库路径 | 设置 `RM_CROSS_HOST_LIBS=$PWD/.cross/toolchain/usr/lib/x86_64-linux-gnu` |
| Docker Hub pull 超时 | 网络不可达，但本地可能已有镜像 | 先执行 `docker image inspect slirute/qidian:arm64v8`；本地存在就继续 |
| `libopenvino.so.2530` 找不到 | 未加载 OpenVINO/TBB 路径 | `source /usr/local/setupvars.sh`，并在 `LD_LIBRARY_PATH` 加入 `/usr/local/runtime/lib/aarch64:/usr/local/runtime/3rdparty/tbb/lib` |
| `libnvdla_compiler.so` / `libcuda.so.1` 找不到 | 未加载 Jetson tegra 运行时目录 | 在 `LD_LIBRARY_PATH` 加入 `/usr/lib/aarch64-linux-gnu/tegra` |
| colcon 报 `layout 'merged'` | 交叉产物是 merged install | Xavier 重编必须加 `--merge-install` |
| 镜像入口只打印 OpenVINO 初始化 | `setupvars.sh` 消耗了 entrypoint 位置参数 | 显式使用 `bash -lc`，或修改 `/rm_entrypoint.sh` 先保存 `$@` |
| `qd2026-log/tensorrt/` 不存在 | 引擎仍在构建 | 它不是启动时创建；等待 `.engine` 生成 |
| 首次启动很慢 | 正在构建 TRT 引擎 | 正常；删除 `qd2026-log/tensorrt/*.engine` 可强制重建 |

> 与 QEMU 方案的关系：QEMU 模拟编译不需要准备工具链和 sysroot，但整工作区编译约 10 分钟；
> 交叉编译首次准备稍多，之后增量构建只需几秒到一分钟。两者产出的都是 aarch64 二进制，
> 但 TensorRT 引擎、相机和 OpenVINO 推理功能仍必须在 Xavier 上验证。
