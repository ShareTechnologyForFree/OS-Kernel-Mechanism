# qemu运行安卓内核

# **1. 下载 Android 内核源码（android-kernel）**

```bash
git clone https://android.googlesource.com/kernel/common android-kernel
# 拉到你需要的 GKI 分支/标签
```
关键是它自带 `tools/bazel` 构建系统，编译内核时会自动下载工具链，新机器只需有网络和 bazel 依赖（openjdk、libncurses 等）。

# **2. 制作根文件系统（不是"下载 initramfs"）**

initramfs 不是现成下载的，而是用 [prepare_shell_rootfs.sh](file:///home/zdy/android_code/cuttlefish/scripts/prepare_shell_rootfs.sh) 从 Alpine minirootfs 构建出来的：

```bash
./cuttlefish/scripts/prepare_shell_rootfs.sh
```
这个脚本自动完成：下载 `alpine-minirootfs-3.20.10-aarch64.tar.gz` → 解包到 `rootfs/` → 覆写 `/init`、`/etc/inittab` → 打包成 `initramfs.cpio.gz`（[L41-L122](file:///home/zdy/android_code/cuttlefish/scripts/prepare_shell_rootfs.sh#L41-L122)）。

更省事的做法：直接把整个 `cuttlefish/rootfs_shell/` 目录（含 `alpine-minirootfs.tar.gz` + `rootfs/`）从旧机器拷到新机器，就能跳过这步。依赖只要 `cpio`、`gzip`、`wget`。

# **3. 安装 QEMU 环境**

```bash
sudo apt install qemu-system-arm     # 提供 qemu-system-aarch64
```

# **4. 编译内核 → 打包 rootfs → 启动**

```bash
cd /home/zdy/android_code/cuttlefish/scripts
./launch_aarch64_qemu.sh --build kernel      # bazel run //common:kernel_aarch64_dist，产物 out/kernel_aarch64/dist/Image
./launch_aarch64_qemu.sh --start             # 自动重打包 rootfs 并启动 QEMU
```

所以你的 4 步应修正为：**① 下载 android-kernel 源码 → ② 构建根文件系统（跑 prepare_shell_rootfs.sh，或用现成 rootfs_shell 目录）→ ③ 装 QEMU → ④ bazel 编译内核 + 启动**。

另外注意两点：android-kernel 仓库很大（下载加编译要占用大量磁盘和时间）；`--build kernel` 走 bazel，首次会拉取工具链，需要网络。