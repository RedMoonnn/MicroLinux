# 最小 Linux 系统构建手册（超详细版）

本手册将一步步教你如何从零开始，构建一个只依赖 Linux 内核（bzImage）和 BusyBox 文件系统（initramfs）的最小 Linux 系统，并用 QEMU 启动。**即使你是初学者，也能看懂！**

---

## 🗂️ 推荐目录结构

```plaintext
MicroLinux/
├── busybox-1.36.1/         # busybox 源码及编译目录
│   ├── busybox             # 编译生成的 busybox 可执行文件
│   └── ...                 # 其它 busybox 源码和文件
├── initramfs/              # 用于打包 initramfs 的根文件系统目录
│   ├── bin/                # 存放 busybox 及其软链接
│   ├── sbin/
│   ├── etc/
│   ├── proc/
│   ├── sys/
│   ├── usr/
│   │   ├── bin/
│   │   └── sbin/
│   ├── dev/
│   └── init                # 启动脚本（必须有执行权限）
├── initramfs.img           # 打包生成的 initramfs 镜像
├── bzImage                 # Linux 内核镜像（可用现成的或自己编译）
├── Guide.md                # 你的操作手册
└── ...                     # 其它文档或脚本
```

---

## 🛠️ 步骤总览
1. 准备 BusyBox
2. 构建 initramfs 根目录结构
3. 编写 /init 启动脚本
4. 打包 initramfs
5. 准备 Linux 内核（bzImage）
6. 用 QEMU 启动整个系统

---

## 0️⃣ 背景知识

- **Linux 内核（bzImage）**：操作系统的核心，负责管理硬件和系统资源。
- **BusyBox**：一个集成了许多常用 Linux 命令的超小型工具箱，非常适合嵌入式和最小系统。
- **initramfs**：内存中的临时根文件系统，系统启动时加载，里面包含启动所需的最基本文件。
- **QEMU**：一个开源虚拟机，可以模拟各种硬件环境，非常适合测试和学习。

---

## 1️⃣ 下载并编译 BusyBox

### 1.1 安装编译依赖

```bash
sudo apt update
sudo apt install build-essential libncurses-dev wget -y
```

### 1.2 下载 BusyBox 源码

```bash
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xjf busybox-1.36.1.tar.bz2
cd busybox-1.36.1
```

### 1.3 配置和编译（务必静态编译，防止 QEMU 启动 panic）

> **强烈建议 busybox 必须静态编译，否则 QEMU 启动会 panic！**

- 推荐直接编辑 `.config` 文件，添加：
  ```
  CONFIG_STATIC=y
  ```
- 如果你能用 `make menuconfig`，也可以在 `BusyBox Settings` → `Build Options` 勾选 `Build BusyBox as a static binary (no shared libs)`。
- 然后编译：
  ```bash
  make clean
  make defconfig
  make -j$(nproc)
  ```
- 检查 busybox 是否为静态编译：
  ```bash
  file busybox
  # 输出应包含 statically linked
  ```

---

## 2️⃣ 构建 initramfs 根目录结构

```bash
mkdir -p initramfs/{bin,sbin,etc,proc,sys,usr/bin,usr/sbin,dev}
```

- 拷贝 busybox 到 bin 目录：
  ```bash
  cp busybox-1.36.1/busybox initramfs/bin/
  ```
- 生成所有命令软链接：
  ```bash
  cd initramfs/bin
  for cmd in $(./busybox --list); do
      ln -sf busybox $cmd
  done
  cd ../..
  ```
- 检查 /bin/sh 是否存在：
  ```bash
  ls -l initramfs/bin/sh
  ```

---

## 3️⃣ 编写 /init 脚本

```bash
vim initramfs/init
```
内容如下：
```sh
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo "Welcome to my custom Linux!"
exec /bin/sh
```

- 赋予执行权限：
  ```bash
  chmod +x initramfs/init
  ```
- 检查编码和权限：
  ```bash
  file initramfs/init
  cat -A initramfs/init  # 检查是否有 ^M
  ```

---

## 4️⃣ 打包成 initramfs 镜像

```bash
cd initramfs
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.img
cd ..
```
- 检查镜像内容：
  ```bash
  zcat initramfs.img | cpio -t | grep '^init$'
  zcat initramfs.img | cpio -t | grep '^bin/sh$'
  ```

---

## 5️⃣ 准备 Linux 内核（bzImage）

- 如果你已经有 bzImage 文件，直接用即可。
- 如果自己编译：
  ```bash
  wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.1.tar.xz
  tar -xf linux-6.1.1.tar.xz
  cd linux-6.1.1
  make defconfig
  make -j$(nproc)
  # 编译后，bzImage 通常在 arch/x86/boot/bzImage
  cp arch/x86/boot/bzImage ../
  cd ..
  ```

---

## 6️⃣ 使用 QEMU 启动系统

```bash
qemu-system-x86_64 \
  -kernel bzImage \
  -initrd initramfs.img \
  -nographic \
  -append "console=ttyS0"
```

---

## 🛠️ 常见问题与排查流程

### 1. QEMU 启动 panic: No working init found
- 检查 initramfs.img 是否包含根目录下的 init 文件。
- 检查 /init 是否有执行权限、无 BOM、无 Windows 换行。
- 检查 /bin/sh 是否存在且为 busybox 软链接。
- 检查 busybox 是否为静态编译（file busybox 应含 statically linked）。
- 每次修改后都要重新打包 initramfs.img。

### 2. busybox 不是静态编译导致 panic
- 重新配置 busybox 静态编译，见上文。

### 3. menuconfig 无法使用
- 直接编辑 .config 文件，手动添加 `CONFIG_STATIC=y`。

### 4. initramfs.img 打包不当
- 必须在 initramfs 目录下执行打包命令，且用 find . 开头。
- 检查镜像内容，确保 /init、/bin/sh 都在。

### 5. /init 脚本内容问题
- 必须以 `#!/bin/sh` 开头，内容无 BOM、无 ^M。
- 最后一行必须 `exec /bin/sh`。

---

## 📦 完整目录结构复查

```plaintext
MicroLinux/
├── busybox-1.36.1/
│   └── busybox
├── initramfs/
│   ├── bin/
│   │   ├── busybox
│   │   └── [所有命令软链接]
│   ├── sbin/ etc/ proc/ sys/ usr/bin/ usr/sbin/ dev/
│   └── init
├── initramfs.img
├── bzImage
├── Guide.md
└── ...
```

---

如有任何疑问，欢迎随时留言！ 