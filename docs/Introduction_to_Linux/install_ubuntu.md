# Ubuntu 双系统安装常见问题

Windows 与 Ubuntu 双系统安装过程中及装完后常见的问题与解决方法。

## Windows 与 Ubuntu 时间相差 8 小时

两个系统对硬件时钟（RTC）的理解不同：Windows 默认把它当作本地时间，Ubuntu 默认当作 UTC，双系统下就会差 8 小时。

在 Ubuntu 中让系统把 RTC 当作本地时间：

```bash
timedatectl set-local-rtc 1
```

验证：

```bash
timedatectl
```

## 安装时看不到硬盘（RST / VMD）

较新的 Intel 笔记本默认开启 Intel RST（VMD），Ubuntu 安装程序无法识别 NVMe 硬盘，分区界面是空的。

需要在 BIOS 中把 SATA/NVMe 模式从 `RAID`/`RST` 改成 `AHCI`。为避免 Windows 因此蓝屏，先在 Windows 里进入一次安全模式：

1. 在 Windows 运行 `msconfig`，在「引导」选项卡勾选「安全引导」，确定并重启
2. 进 BIOS 把磁盘模式改为 `AHCI`
3. 进入 Windows 安全模式后，再次运行 `msconfig` 取消「安全引导」，正常重启

## 进入 Ubuntu 黑屏或花屏（NVIDIA 显卡）

NVIDIA 显卡驱动未加载时，安装或首次启动可能黑屏、花屏或卡在 logo。

在 GRUB 菜单按 `e`，找到 `quiet splash` 那一行，在后面加上 `nomodeset`（或 `nouveau.modeset=0`），按 `Ctrl+X` 启动。进入系统后安装 NVIDIA 驱动即可。

## 开机直接进入 Windows，没有系统选择菜单

通常是 BIOS 启动顺序里 Windows 排在了 GRUB 前面。

进 BIOS 把 `ubuntu`/`GRUB` 设为第一启动项。若 GRUB 菜单被隐藏，可在 Ubuntu 中把超时时间改大：

```bash
sudo nano /etc/default/grub
```

改成：

```bash
GRUB_TIMEOUT=10
```

更新：

```bash
sudo update-grub
```

## GRUB 里没有 Windows 启动项

新版 GRUB 默认关闭了 os-prober，需要手动打开：

```bash
sudo nano /etc/default/grub
```

确保有这行（改成 false）：

```bash
GRUB_DISABLE_OS_PROBER=false
```

然后：

```bash
sudo update-grub
```

## 重装 Windows 后 GRUB 引导丢失

重装 Windows 会覆盖引导，导致开机直接进 Windows。用 Ubuntu 启动盘进入「试用」环境，安装 boot-repair 修复：

```bash
sudo add-apt-repository ppa:yannubuntu/boot-repair
sudo apt update
sudo apt install boot-repair
boot-repair
```

选择「推荐修复」即可。

## 无法挂载 Windows 的 NTFS 分区

Windows 的「快速启动」会让 NTFS 分区处于休眠状态，Ubuntu 挂载时提示 `Windows is hibernated`。

先按下方方法关闭 Windows 快速启动，临时修复可用：

```bash
sudo ntfsfix /dev/nvme0n1p1
```

## 第三方驱动无法加载（Secure Boot）

安全启动会阻止 NVIDIA 等未签名的内核模块加载，表现为驱动安装成功却无法使用。

进 BIOS 关闭 `Secure Boot`，或在安装驱动时用 MOK 对模块签名。

## 关闭 Windows 快速启动

控制面板 → 电源选项 → 选择电源按钮的功能 → 更改当前不可用的设置 → 取消勾选「启用快速启动」→ 保存。

关闭后 Ubuntu 才能正常读写 Windows 的 NTFS 分区。

## BitLocker 加密导致无法分区或挂载

BitLocker 会加密整个分区，Ubuntu 安装程序无法对它压缩/调整大小，装完后也无法读取。

安装前先在 Windows 中关闭 BitLocker、解密分区，再调整分区大小。

## UEFI 与 Legacy 启动模式不一致

两个系统必须使用相同的启动模式，否则 GRUB 无法引导 Windows。

在 Windows 运行 `msinfo32`，查看「BIOS 模式」是否为 `UEFI`。确保 Ubuntu 也用 UEFI 方式安装（用 UEFI 启动盘启动、磁盘用 GPT 分区），BIOS 中关闭 `CSM`/`Legacy` 只保留 UEFI。

## 压缩 C 盘时空间不足或无法压缩

Windows 磁盘管理压缩失败，通常是页面文件或休眠文件占据了分区末尾。

以管理员身份运行（Windows）：

```
powercfg /h off
```

并在「高级系统设置」中临时把虚拟内存（页面文件）设为「无分页文件」，压缩完成后再恢复。
