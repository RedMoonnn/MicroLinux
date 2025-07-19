# MicroLinux 最小 Linux 系统构建项目

本项目旨在帮助初学者从零搭建一个基于 Linux 内核（bzImage）和 BusyBox 的最小 Linux 系统，并通过 QEMU 启动体验。你将学习到 Linux 内核、initramfs、BusyBox 的基础知识及实际操作流程。

---

## 📁 目录结构

```plaintext
MicroLinux/
├── busybox-1.36.1/         # busybox 源码及编译目录
├── initramfs/              # 用于打包 initramfs 的根文件系统目录
├── initramfs.img           # 打包生成的 initramfs 镜像
├── bzImage                 # Linux 内核镜像
├── Guide.md                # 详细操作手册
└── README.md               # 项目简介（本文件）
```

---

## ⚙️ 项目原理

MicroLinux 的核心思想是利用 Linux 内核（bzImage）和 BusyBox 工具箱，通过 initramfs（内存文件系统）实现一个极简可启动的 Linux 系统。

- **Linux 内核（bzImage）**：负责硬件管理、进程调度、内存管理等，是系统的核心。
- **initramfs**（必须）：一种内存中的临时根文件系统，系统启动时由内核加载，包含启动所需的最基本文件和脚本。没有 initramfs，内核无法挂载根文件系统，系统无法正常启动。
- **BusyBox**（推荐但非唯一选择）：集成了常用 Linux 命令的超小型工具箱，极大简化了最小系统的实现。你也可以用其他工具箱或自行编译的静态二进制替代 BusyBox，但必须保证根文件系统内有可用的 shell（如 /bin/sh）和必要的启动命令，否则系统启动后无法进入命令行环境。

**总结：**
- initramfs 是必须的，否则内核无法找到根文件系统。
- BusyBox 不是唯一选择，但你必须保证有可用的 shell 和基本命令，否则系统无法交互。

**启动流程简述：**
1. QEMU 启动时加载 bzImage（内核）和 initramfs.img（根文件系统）。
2. 内核解压并挂载 initramfs 为根目录，查找并执行根目录下的 `/init` 脚本。
3. `/init` 脚本挂载 proc、sysfs 等伪文件系统，最后执行 `/bin/sh`（由 busybox 或其他 shell 提供），进入命令行环境。
4. 用户即可在最小 Linux 环境下体验基本命令和操作。

---

## 🚀 快速开始

1. **准备 BusyBox**
   - 下载并静态编译 busybox，确保 `file busybox` 输出包含 `statically linked`。
2. **构建 initramfs 根目录结构**
   - 创建 `initramfs/` 及其子目录，将 busybox 拷贝到 `bin/` 并生成软链接。
3. **编写 /init 启动脚本**
   - 内容示例：
     ```sh
     #!/bin/sh
     mount -t proc none /proc
     mount -t sysfs none /sys
     echo "Welcome to my custom Linux!"
     exec /bin/sh
     ```
   - 赋予执行权限：`chmod +x initramfs/init`
4. **打包 initramfs 镜像**
   - 在 `initramfs/` 目录下执行：
     ```bash
     find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.img
     ```
5. **准备 Linux 内核（bzImage）**
   - 可用现成 bzImage 或自行编译。
6. **用 QEMU 启动系统**
   - 启动命令：
     ```bash
     qemu-system-x86_64 -kernel bzImage -initrd initramfs.img -nographic -append "console=ttyS0"
     ```

---

## 📝 主要文件说明
- `busybox-1.36.1/`：BusyBox 源码及编译产物
- `initramfs/`：根文件系统目录，包含启动脚本和必要目录
- `initramfs.img`：打包后的内存文件系统镜像
- `bzImage`：Linux 内核镜像
- `Guide.md`：详细操作手册，适合初学者
- `README.md`：项目简介

---

## 📚 进阶阅读
详细操作步骤、原理说明及更多排查技巧请参考 `Guide.md`。

---

如有疑问，欢迎提 issue 或留言交流！ 