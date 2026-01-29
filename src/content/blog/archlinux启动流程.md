---
title: archlinux启动流程
description: archlinux启动流程
pubDate: 01 29 2026
image: /image/image1.jpg
categories:
  - Archlinux
tags:
  - Archlinux
---


本文大部分引自archlinux的中文wiki

大致流程可以总结为
固件  -> 引导加载程序  -> 内核 -> initramfs ->早期用户空间 -> 晚期用户空间

# 固件
下面引用自维基百科

[固件](https://zh.wikipedia.org/wiki/%E5%9B%BA%E4%BB%B6 "zhwp:固件")是开机时最先执行的程序。
**固件**（英语：firmware），是一种嵌入在[硬件](https://zh.wikipedia.org/wiki/%E7%A1%AC%E9%AB%94 "硬件")设备中的[软件](https://zh.wikipedia.org/wiki/%E8%BB%9F%E9%AB%94 "软件")。通常它是位于[特殊应用集成电路](https://zh.wikipedia.org/wiki/%E7%89%B9%E6%AE%8A%E6%87%89%E7%94%A8%E7%A9%8D%E9%AB%94%E9%9B%BB%E8%B7%AF "特殊应用集成电路")（ASIC）或[可编程逻辑器件](https://zh.wikipedia.org/wiki/%E5%8F%AF%E7%A8%8B%E5%BC%8F%E9%82%8F%E8%BC%AF%E8%A3%9D%E7%BD%AE "可编程逻辑器件")（PLD）之中的[闪存](https://zh.wikipedia.org/wiki/%E5%BF%AB%E9%96%83%E8%A8%98%E6%86%B6%E9%AB%94 "闪存")或[EEPROM](https://zh.wikipedia.org/wiki/EEPROM "EEPROM")或[PROM](https://zh.wikipedia.org/wiki/PROM "PROM")里，有的可以让用户更新。可以应用在非常广泛的电子产品中，从[遥控器](https://zh.wikipedia.org/wiki/%E9%81%A5%E6%8E%A7%E5%99%A8 "遥控器")、[计算器](https://zh.wikipedia.org/wiki/%E8%AE%A1%E7%AE%97%E5%99%A8 "计算器")到[电脑](https://zh.wikipedia.org/wiki/%E7%94%B5%E8%84%91 "电脑")中的[键盘](https://zh.wikipedia.org/wiki/%E9%94%AE%E7%9B%98 "键盘")、[硬盘](https://zh.wikipedia.org/wiki/%E7%A1%AC%E7%9B%98 "硬盘")，甚至[工业机器人](https://zh.wikipedia.org/wiki/%E5%B7%A5%E4%B8%9A%E6%9C%BA%E5%99%A8%E4%BA%BA "工业机器人")中都可见到它的身影。

顾名思义，固件是介于软件和硬件之间的。像软件一样，它是由电脑所执行的程序。然而它是对于硬件内部而言更加贴近以及更加重要的部分，而对于外在世界而言较无重要的意义。

我们这里主要介绍两种：UEFI和BIOS

## UEFI
统一可扩展固件接口（Unified Extensible Firmware Interface，简称 UEFI）**是操作系统和固件之间的接口。UEFI 提供了启动操作系统或运行预启动程序的标准环境。

## BIOS
基本IO系统，（Basic Input-Output System）大多数情况下储存在主板自身的一块闪存内，独立于其它系统存储。

## UEFI和BIOS
UEFI是BIOS的现代替代品，BIOS则较为传统
### 区别

|特性|BIOS|UEFI|
|---|---|---|
|**程序模式**|16位实模式|32/64位保护模式|
|**用户界面**|文本菜单，键盘操作|图形界面，支持鼠标和触摸|
|**硬盘分区**|**MBR**（最大2TB，4主分区）|**GPT**（容量极大，分区数多）|
|**启动流程**|从MBR的固定扇区读取代码|从ESP分区中的**可执行文件**启动|
|**启动速度**|相对较慢|通常更快（支持快速启动）|
|**安全功能**|无或很弱|**安全启动**，防止恶意软件|
|**扩展性**|差|好，模块化设计|
|**最大硬盘**|2TB|理论18EB（当前受OS限制）|
### 共同点
核心作用：
- **开机自检**： 检查CPU、内存、硬盘、显卡等关键硬件是否正常。
    
- **初始化硬件**： 加载硬件的基本驱动程序。
    
- **引导操作系统**： 按照预设顺序（如硬盘、U盘、光盘）寻找可启动设备，并加载该设备上**主引导记录**（MBR）中的引导程序，从而启动操作系统（如Windows、Linux）。


## 工作流程
以bios为例
在电脑开启之后，bios会直接存在于内存之中，然后检测接入电脑的各种输入输出设备，比如U盘，硬盘，显示器，键盘，显卡等，然后会进行加电自检，检查计算机设备硬件是否存在问题，进而保证计算机的正常运行。

接下来就要分两种情况了
首先要补充一点知识
#### EFI系统分区
EFI系统分区（也称为 ESP）是一个与操作系统无关的分区，其中存储了由 UEFI 固件启动的 UEFI 引导加载器、应用程序和驱动，是 UEFI 启动所必须的。

![alt text](<../../../public/image/archlinux1/Pasted image 20260129103045.png>)

可见我的磁盘标签是GPT并且具有EFI分区，我的电脑是使用的UEFI,绝大多数现代电脑也都用的UEFI

### 使用UEFI的情况
1. 加电自检后，UEFI 初始化引导所需的硬件（硬盘、键盘控制器等等）。
2. 固件读取 NVRAM 中的引导项，以决定要启动哪一个 EFI 应用程序，以及从哪启动（比如从哪一个硬盘和分区）。
	- 一个引导项可能对应的只是一块硬盘。在这种情况下，固件会寻找硬盘上的 EFI 系统分区，并尝试在后备引导路径 `\EFI\BOOT\BOOTx64.EFI` 处（在 IA32（32 位）UEFI 的系统上为 `BOOTIA32.EFI`）查找 EFI 应用程序。这就是UEFI 可引导可移除介质的工作原理。
3. 固件启动 EFI 应用程序。
	- 这可以是一个引导加载程序，或者是使用 EFISTUB 的 Arch 内核本体。
	- 还可以是一些其他的 EFI 应用程序，比如 UEFI shell 或引导管理器（例如 systemd-boot) 或 rEFInd）。

如果启用了安全启动，启动过程将会通过签名验证 EFI 二进制文件的真实性。

## 使用BIOS的情况
1. 上电自检后，BIOS 初始化引导所需的硬件（硬盘、键盘控制器等等）。
2. BIOS 启动在“BIOS 硬盘顺序”中第一块硬盘上的前 440 字节代码(即主引导记录引导代码区域)
3. 引导加载程序在 MBR 引导代码的第一阶段，之后会从下列任意一处启动第二阶段代码（如果有的话）：
    - MBR 之后的下一个磁盘扇区，即所谓 MBR 后间隙（post-MBR gap，仅在 MBR 分区表上有）。
    - 分区或者无分区磁盘的卷引导记录（Volume Boot Record，VBR）。
    - GRUB 特定 BIOS 引导分区（仅限 GPT 分区硬盘上的 GRUB，用于 GPT 上没有 MBR 后间隙的情况）
- 真正的引导加载程序启动。
- 随后，引导加载程序通过链式加载或直接加载操作系统内核的方式加载操作系统。


# 引导加载程序
这是前文BIOS和UEFI启动的程序

引导加载程序(boot loader)，负责用指定的内核参数加载内核和其他initramfs映像

引导管理器(boot managerc),让用户使用启动选项菜单或其他方式控制启动过程

一些程序例如GRUB兼具上面两者的功能

在 UEFI 的情况下，内核本身可以由 UEFI 使用 EFI boot stub接启动。要在引导前编辑内核参数，可以使用引导管理器或是单独的引导加载程序。

### 注意
引导加载程序必须能够访问通常位于 `/boot` 目录下的内核和 initramfs 映像才能成功引导 Arch 系统。也就是说，引导加载程序必须解决从块设备、堆叠块设备（LVM、RAID、dm-crypt、LUKS 等）开始，到内核和 initramfs 映像所在文件系统为止的访问。
![alt text](<../../../public/image/archlinux1/Pasted image 20260129104748.png>)

因为几乎没有引导加载程序支持堆叠块设备，并且文件系统引入的一些新特性可能尚未有任何引导加载程序支持，所以用广泛支持的文件系统（例如 FAT32）单独创建 /boot 分区通常更可行。

# 内核
然后就到了下一步内核
boot loader会启动包含内核的vmlinux映像

内核是操作系统的核心。它运行于一个叫_内核空间_的底层上，负责机器硬件和应用程序之间的交流。在继续进入用户空间前，内核会首先执行硬件枚举和初始化。

> 在linux系统中，**vmlinux**（**vmlinuz**）是一个包含linux kernel的静态链接的可执行文件，文件类型可能是linux接受的可执行文件格式之一（[ELF](https://zh.wikipedia.org/wiki/%E5%8F%AF%E5%9F%B7%E8%A1%8C%E8%88%87%E5%8F%AF%E9%8F%88%E6%8E%A5%E6%A0%BC%E5%BC%8F "可执行与可链接格式")、[COFF](https://zh.wikipedia.org/wiki/COFF "COFF")或[a.out](https://zh.wikipedia.org/wiki/A.out "A.out")），vmlinux若要用于调试时则必须要在开机前增加symbol table。
> 随着 linux Kernel 的成长，核心的内容日益增加超越了原本的限制大小。bzImage (big zImage) 格式则为了克服此缺点开始发展，利用将核心切割成不连续的存储器区块来克服大小限制。
> bzImage 格式仍然是以 zlib 算法来做压缩，虽然有一些广泛的误解就是因为以 bz- 为开头，而让人误以为是使用 bzip2 压缩方式（bzip2 包所带的工具程序通常是以 bz- 为开头的，例如 bzless, bzcat ...）。
> bzImage 文件是一个特殊的格式，包含了 bootsect.o + setup.o + misc.o + piggy.o 串接。piggy.o 包含了一个 gzip 格式的 vmlinux 文件（可以参看 arch/i386/boot／下的 compressed/Makefile piggy.o）

# initramfs
_initramfs_（初始内存文件系统，init ial RAM file system）映像是一个 cpio存档文件，为早期用户空间（见下文）启动晚期用户空间提供了必要的文件。这包括了所有用于定位，访问和挂载根文件系统的内核模块、用户空间工具、相关库文件、类似 udev 规则的支持文件等。得益于 initramfs 的概念，它可以处理更加复杂的配置场景，例如从外置硬盘启动，堆叠设备（例如逻辑卷，软 RAID，压缩和加密），或是在早期用户空间中运行一个微型 SSH 服务器，以供远程解锁或为根文件系统执行维护任务。

绝大部分内核模块都将在初始化流程的后期阶段，由udev在根切换到根文件系统后加载。

具体流程如下：

1. `/`下的根文件系统原本是一个空的 rootfs，它是一个特殊的 tmpfs 或 ramfs 实例。这里就是 initramfs 会解压到的临时根文件系统。
2. 内核会将其内置 initramfs 解压到临时根文件系统下。Arch Linux 官方支持的内核使用空白存档作为内置 initramfs，即构建内核时的默认行为。
3. 然后，内核会按照引导加载器传递的命令行参数指定的顺序解压外置 initramfs 映像，覆盖掉之前内置 initramfs 或其它解压出来的文件。注意，可以将多个 initramfs 映像合并为一个文件，内核会按照文件内的顺序加载映像。
    - 如果首个 initramfs 映像未经压缩，那么内核会在解包该映像后在 `/kernel/x86/microcode/` 目录查找 CPU 微码更新，在 `/kernel/firmware/acpi/` 目录查找 ACPI 表更新。
    - 在适用的情况下，在处理完 CPU 微码和 ACPI 表更新后，内核会继续解压剩余的 initramfs 映像。

initramfs 映像是 Arch Linux 推荐的早期用户空间配置方法，并可通过 mkinitcpio，dracut 或 booster来生成。

- 内置 initramfs 是内核镜像里的 “附属品”，体积只有几十 KB，仅包含最基础的内核启动代码，没有任何硬件驱动；
- 外置 initramfs 是独立文件，体积几十 MB，包含了你在 `mkinitcpio.conf` 中配置的所有模块、钩子、脚本，是系统启动真正依赖的早期用户空间。

内核保留内置迷你 initramfs，核心是为了**兼容性兜底**：

- 如果系统没有外置 initramfs（比如极简内核、嵌入式系统），内置 initramfs 能保证内核至少能启动到 “紧急 shell”；
- 对于桌面 / 服务器系统（如 Arch），外置 initramfs 可灵活定制（加驱动、加加密脚本），无需重新编译内核 —— 这也是 mkinitcpio 的核心价值：用户不用改内核，只需定制外置 initramfs 就能适配不同硬件。


> udev 是一个用户空间的设备管理器，用于为事件设置处理程序。作为守护进程， udev 接收的事件主要由 linux 内核生成，这些事件是外部设备产生的物理事件。总之， udev 探测外设和热插拔，将设备控制权传递给内核，例如加载内核模块或设备固件。
> 
> udev是一个用户空间系统，可以让操作系统管理员为事件注册用户空间处理器。为了实现外设侦测和热插拔，_udev_ 守护进程接收 Linux 内核发出的外设相关事件; 加载内核模块、设备固件; 调整设备权限，让普通用户和用户组能够访问设备。
> 
> Ramfs 是一种极简的文件系统，它将 Linux 的磁盘缓存机制（页缓存与目录项缓存）封装为可动态调整大小的**基于内存的文件系统**。



### mkinitcpio

外置的initramfs是由mkinitcpio.conf生成的

Arch Linux 中 `/etc/mkinitcpio.conf` 文件里 `MODULES` 配置项的含义，以及当前配置的 `nvme` 和一系列 `nvidia` 相关模块的作用 —— 这是定制 initramfs（早期用户空间）的核心配置，决定了哪些内核模块会被**强制打包进 initramfs**，并在系统启动最早期（所有启动钩子运行前）加载，是解决硬件驱动早期加载、避免启动故障的关键。

#### MODELES
这个数组是手动指定的、需要**强制打包进 initramfs** 的内核模块，每一个模块都对应核心硬件功能，且都是 “内核自动探测 / 钩子加载可能不及时” 的关键模块：

这是定制 initramfs（早期用户空间）的核心配置，决定了哪些内核模块会被**强制打包进 initramfs**，并在系统启动最早期（所有启动钩子运行前）加载，是解决硬件驱动早期加载、避免启动故障的关键。

我这里主要加载了NVMe 固态硬盘的核心驱动模块以及nvidia驱动的一些模块

给和我一样的新手提醒：NVIDIA 闭源驱动是**第三方模块**，内核默认不识别，无法通过 udev 自动探测加载；若不提前打包进 initramfs，而是靠 `modules-load.d` 加载，会导致：

- 早期用户空间阶段显卡无输出（开机黑屏）；
- 切换到真正根文件系统后才加载 NVIDIA 驱动，出现显示闪烁、分辨率异常；
- 启用 KMS 早启动（Arch 推荐配置）时，必须在 initramfs 阶段加载 `nvidia_drm`，否则显卡驱动无法正常初始化。

因为靠 `modules-load.d`加载时在早期用户空间阶段，而在那时，你还没加载驱动
![alt text](<../../../public/image/archlinux1/Pasted image 20260129211504.png>)

#### BINARIES
`BINARIES` 用于将你需要的**额外二进制可执行文件**（比如命令行工具、自定义程序）加入到 CPIO 格式的 initramfs 镜像中（initramfs 本质是 cpio 压缩包，`mkinitcpio` 就是 “make init cpio” 的缩写）；
比如你想在 initramfs 阶段（启动时的紧急 shell）执行 `lsblk` 查看磁盘分区（默认 initramfs 没有 `lsblk`），就需要添加：
``` shell
BINARIES=(/usr/bin/lsblk)
```

如果你想让 initramfs 自动读取密钥文件解密根分区（无需开机手动输密码），可把密钥文件添加到 `FILES`：

``` shell
# 示例：添加LUKS解密密钥文件到initramfs
FILES=(/etc/cryptkey.bin)
```

#### FILES
`FILES` 和 `BINARIES` 作用类似（都是向 initramfs 中添加文件），但处理方式完全不同；
对于files来说会`as-is`（原样添加）—— mkinitcpio 会把你指定的文件**原封不动**复制到 initramfs 中而Binaries不是

#### HOOKS
HOOKS 是整个配置文件中最重要的项
既控制 “哪些模块 / 脚本被打包进 initramfs”，也控制 “启动时按什么顺序执行什么操作”；
**顺序极其重要**（后一个钩子依赖前一个的执行结果），一般不要乱改顺序；
**必需钩子**：
- `base`：必选（除非你完全清楚自己在做什么），包含 initramfs 运行的最基础脚本 / 模块（比如 shell、基础工具）；
- `udev`/`systemd`：二选一必选（自动加载模块的核心），我用的是 `systemd` 替代了传统的 `udev`；
- `filesystems`：必选（除非你在 MODULES 里手动指定了所有文件系统模块），负责加载 ext4/xfs/btrfs 等文件系统驱动；
![alt text](<../../../public/image/archlinux1/Pasted image 20260129213441.png>)

#### COMPRESSION_OPTIONS
`COMPRESSION`：设置 initramfs 镜像的压缩算法
- **核心作用**：指定 mkinitcpio 生成 initramfs 镜像时使用的**压缩算法**（initramfs 本质是 cpio 包 + 压缩层，最终生成 `initramfs-linux.img` 是 “cpio + 压缩” 的组合）；
- **默认行为**：mkinitcpio 会自动适配内核版本 ——Linux 内核 ≥5.9 用 `zstd`（Arch 主流内核都满足），<5.9 用 `gzip`；
- **特殊值**：若设为 `COMPRESSION="cat"`，则生成**未压缩**的 initramfs 镜像（体积最大，但解压最快）。


#### MODULES_DECOMPRESS

- **核心作用**：开关（`yes`/`no`），控制 mkinitcpio 生成 initramfs 时，是否先**解压内核模块（.ko）和固件文件**，再打包进镜像；
- **默认行为**：`no` → 内核模块 / 固件保持**原始压缩状态**（Linux 内核模块默认是 xz/gzip 压缩的），直接打包进 initramfs；
- **开启（yes）的目的**：配合「高压缩参数」（如 xz -9e、zstd -22）进一步减小 initramfs 体积 —— 因为模块先解压再用指定算法压缩，比 “模块自带压缩 + initramfs 压缩” 的「双重压缩」效率更高；
- **开启（yes）的代价**：早期启动阶段（initramfs 解压后），模块会以**未压缩状态**加载到内存，占用更多 RAM；
- **关键注意**：开启后，解压后的模块会放在 initramfs 的 “未压缩早期 CPIO” 中，避免双重压缩（否则会抵消高压缩的收益）。

# 早期用户空间

Linux 内核启动后，**本身无法直接识别所有硬件和根文件系统**（比如加密分区、LVM 逻辑卷、NVMe 固态硬盘驱动、RAID 控制器等），如果直接尝试挂载根文件系统，大概率会 “找不到设备” 或 “无法解析文件系统”。

因此，内核会先加载一个**精简的、内存中的临时文件系统（initramfs/initrd）** —— 这就是 “早期用户空间”：它是一个迷你版的用户空间环境，包含了启动真正根文件系统所需的最小化工具、驱动和脚本，核心使命是 “帮内核扫清障碍，让内核能成功挂载并切换到真正的根文件系统”。

简单类比：早期用户空间就像 “系统启动的前置助手”，先帮内核搞定硬件识别、加密解密、存储栈组装这些 “前置工作”，再把控制权交还给真正的根文件系统。


早期用户空间阶段（亦称“initramfs 阶段”）在由 initramfs映像提供文件的 rootfs 中进行，始于内核以 PID 1 执行 `/init`。

`/init`程序我的是systemd

1. 加载内核模块（systemd-modules-load (8)）

- **作用**：加载挂载真正根文件系统必需的内核模块（驱动）。
    
    内核本身只内置了最基础的驱动，像 NVMe 硬盘、SATA 控制器、USB 存储、加密分区（dm-crypt）、LVM（dm-mod）等驱动，都以 “模块” 形式存在，需要手动加载。
- **实现**：基于 systemd 的 initramfs 会通过 `systemd-modules-load.service`，读取 `/etc/modules-load.d/`、`/usr/lib/modules-load.d/` 等配置文件，自动加载指定模块；如果是 BusyBox 版 initramfs，则通过 `modprobe`/`insmod` 命令手动加载。
- **举例（Arch 场景）**：如果你的根分区在 NVMe 硬盘上，必须加载 `nvme` 模块；如果用了 LUKS 加密，必须加载 `dm-crypt` 模块 —— 少了这些，内核找不到根硬盘，直接卡在 “Waiting for root device”。

 2. 构建存储栈 + 解密根文件系统

这是早期用户空间最复杂也最核心的工作（尤其对加密 / RAID/LVM 系统）：

- **“存储栈” 是什么**：把底层硬件→逻辑卷 / RAID→加密层→文件系统 这一系列组件组装起来，形成能访问根分区的完整链路。
    
    比如：`NVMe 硬盘（/dev/nvme0n1p3）` → `LUKS 加密层（dm-crypt）` → `LVM 逻辑卷（vg0/root）` → `ext4 文件系统`，这个 “栈” 必须在早期用户空间组装完成。
- **核心工具**：
    
    - `dm-crypt`：解密 LUKS 加密的根分区（Arch 中加密根分区时，initramfs 会弹出密码输入界面，输入后解密）；
    - `dm-verity`：验证根文件系统的完整性（防止篡改）；
    - `mdadm`：组装软 RAID 阵列；
    - `LVM（lvm2）`：激活逻辑卷组 / 逻辑卷；
    - `systemd-repart`：动态调整分区大小（比如启动时自动扩容根分区）。
    
- **关键注意点**：如果根分区加密，解密操作**只能在早期用户空间做**—— 因为解密前根文件系统完全不可访问，没有任何工具能运行。

3. udev 解析块设备持久化名称

- **问题背景**：Linux 设备名（如 `/dev/sda3`）是动态的（比如插了 U 盘后，硬盘可能从 sda 变成 sdb），直接用动态名找根分区会出错。
- **udev 的作用**：在早期用户空间中，udev 会扫描硬件，将动态设备名映射为**持久化名称**（比如 `/dev/disk/by-uuid/xxxx`、`/dev/mapper/cryptroot`），确保内核能精准找到根分区，不会因设备名变动导致启动失败。

4. 加载 DRM 模块（早启动 KMS）

- **DRM（Direct Rendering Manager）**：Linux 显卡驱动的核心框架；KMS（Kernel Mode Setting）：内核模式设置，负责显卡分辨率、显示输出。
- **为什么早加载**：
    
    1. 支持启动时的图形化启动画面（比如 Arch 的 plymouth 美化启动界面）；
    2. 内核能更早输出显卡相关日志，方便排查启动故障；
    3. 避免后续切换根文件系统时出现显示异常（比如黑屏、分辨率错误）。
    

 5. 其他关键任务（挂载根前必做）

 `fsck` 和 “从休眠中恢复” 是早期用户空间的额外核心任务，且**只能在挂载真正根文件系统前执行**：

- `fsck`：文件系统检查。如果根文件系统有损坏，`fsck` 必须在 “未挂载” 状态下执行（挂载后执行会破坏文件系统），因此只能放在早期用户空间；
- 休眠恢复：从交换分区 / 休眠镜像恢复系统时，需要先挂载休眠镜像所在的分区，且此时不能挂载根文件系统（否则会冲突），因此也必须在早期用户空间完成。

## systemd
systemd 是**Linux 系统的系统和服务管理器**，也是现代绝大多数 Linux 发行版（包括你使用的 Arch Linux）默认的**PID 1 进程**（系统启动后用户空间运行的第一个进程），核心替代了传统的 SysVinit、Upstart 等初始化系统，负责**接管系统启动、管理服务生命周期、统筹系统各类资源**，是 Linux 系统运行的核心管家。

简单说：系统开机后，内核完成初始化后，第一个启动的用户空间程序就是 systemd，之后所有的系统服务、应用进程，几乎都是由 systemd 启动 / 管理的，它也会全程监控这些进程，同时处理系统关机、休眠、设备挂载等核心操作。

## modules-load.d
`modules-load.d` 是 **systemd 系 Linux 系统的标准化配置目录**，专门用于定义 “系统启动时需要**自动加载的内核模块**”。

我这里是开机自动加载tun内核模块，里面一般写的是虚拟驱动模块，因为一般的硬件驱动模块（NVMe、SATA、USB 存储）被内核通过 udev 自动探测加载，无需配 conf，而功能类模块（dm-crypt、LVM/dm-mod）：mkinitcpio 钩子自动打包加载，无需配 conf
![alt text](<../../../public/image/archlinux1/Pasted image 20260129210706.png>)
# 晚期用户空间

晚期用户空间从 init进程开始。Arch 官方支持的 systemd基于单元和服务的概念，但这里描述的功能在很大程度上与其它 init 系统重叠。

### getty

init会为每个虚拟终端（通常有六个）调用一次 getty，它会初始化终端并保护其免受未授权访问。在提供用户名和密码后，_getty_ 会对照 `/etc/passwd` 和 `/etc/shadow` 检查是否正确。如果正确，就接着调用 login(1)。

##### /etc/passwd
`/etc/passwd` 是 **Linux 系统的核心用户账户配置文件**，存储了系统中 ** 所有用户（包括 root、普通用户、系统服务用户）** 的基础账户信息，是系统识别、验证用户身份的核心依据，**所有用户都拥有只读权限**，仅 root 可修改，是 Linux 多用户管理的基础文件。

早期该文件还存储用户的加密密码，后因安全问题（全局可读易被破解），密码被迁移到 `/etc/shadow`（仅 root 可读），如今 `/etc/passwd` 仅保留**非敏感的基础用户信息**，这是 Linux 安全设计的重要调整。

`/etc/passwd` 中**每行对应一个用户**，行内用**冒号 `:`** 分隔为**7 个固定字段**，字段顺序不可乱，空字段也需保留冒号（格式错误会导致用户登录失败）。这里不再展开

#### 为什么需要 /etc/shadow？

早期 Linux 把用户加密密码直接存在 `/etc/passwd` 的第二个字段，但 `/etc/passwd` 为了让系统程序识别用户，必须设置**全局可读权限**（644），这意味着任何用户都能读取加密密码串，再通过暴力破解工具尝试解密，存在极大安全风险。

为了解决这个问题，Linux 引入了**影子密码机制**：将**加密密码、密码有效期、账户锁定**等敏感信息从 `/etc/passwd` 迁移到 `/etc/shadow`，并将其权限严格限制为**仅 root 可读写（600）**，而 `/etc/passwd` 仅保留非敏感的基础用户信息，实现了**敏感信息与基础信息的分离**，大幅提升了系统账户安全。

迁移后，`/etc/passwd` 的第二个字段固定为占位符 `x`，表示**密码信息已迁移至 /etc/shadow**。

#### login

_login_ 会根据 `/etc/passwd` 设置环境变量并启动用户 shell，从而为用户配置一个会话。在成功登录后，启动登录 shell 前，_login_ 程序会显示 /etc/motd（message of the day）的内容，你可以用它来显示服务条款以提醒用户你的本地策略，也可以显示其它提示信息。

#### /etc/motd
`/etc/motd` 是 Linux 系统的**每日提示信息文件**，全称 **Message of the Day**，核心作用是**用户通过本地终端 / SSH 远程登录系统后，自动显示的欢迎 / 提示信息**，是系统管理员发布系统通知、维护提醒、安全警告的常用方式，普通用户也可自定义个性化登录欢迎语，**仅对终端登录生效，图形界面登录不会显示该文件内容**。

#### shell

用户的 shell启动后，在显示命令行提示符前，通常会执行一个运行时配置文件（例如 bashrc。如果用户账户配置为在登录时自动启动 X，那么运行时配置文件会调用 startx 或 xinit，具体内容请参考[#图形会话（Xorg）](https://wiki.archlinuxcn.org/wiki/Arch_%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B#图形会话（Xorg）)。

### 显示管理器

这里还没提到wayland

#### 图形会话（Xorg）

xinit 会调用用户的 xinitrc 运行时配置文件，后者一般会启动一个窗口管理器或。如果用户退出了窗口管理器，_xinit_、_startx_、shell、login 就会依次中断，返回到 _getty_ 或显示管理器。