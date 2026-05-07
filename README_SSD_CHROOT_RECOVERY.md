# Ubuntu `prime-select intel` 后黑屏/进不去 TTY/GRUB 的离线修复指南（拆 SSD + USB）

> 适用场景：你在一台带 Intel + NVIDIA 的 Ubuntu 机器上执行了 `prime-select intel`（或类似切换），之后出现黑屏、图形冲突、进不去 TTY 或 GRUB，系统无法正常启动。

这份文档面向新手，给出一个**稳定、可重复**的修复方案：  
把故障机器的 SSD 拆下来，用 USB 连接到另一台正常 Ubuntu 电脑，在正常电脑里对故障系统做 `chroot` 修复。

---

## 1. 原理说明（为什么这个方法有效）

这个方案和 Live USB 救援本质一样：

- Live USB：用临时 Ubuntu 系统修故障盘
- 本文方法：用另一台已正常运行的 Ubuntu 系统修故障盘

关键点是：你不是“手改几个配置文件”，而是**进入故障系统环境执行官方命令**（`prime-select`、`update-initramfs`、`update-grub`），让驱动/引导链路完整更新。

---

## 2. 先识别哪块盘是故障 SSD

在正常 Ubuntu 电脑执行：

```bash
lsblk -f
```

根据示例场景：

- 当前正常系统：`/dev/nvme1n1p2`（`/`）、`/dev/nvme1n1p1`（`/boot/efi`）
- 外接故障 SSD：`/dev/sda2`（root）、`/dev/sda1`（EFI）

⚠️ **一定要以你自己机器的 `lsblk -f` 输出为准，绝不要照抄设备名。**

---

## 3. 挂载故障系统分区

如果外接盘被自动挂载在 `/media/...`，先卸载后再按标准路径挂载：

```bash
sudo umount /media/x/6f14b855-4c6b-451e-af4e-cfb74ec62395
```

挂载 root 到 `/mnt`：

```bash
sudo mount /dev/sda2 /mnt
ls /mnt
```

如果看到 `bin boot dev etc home usr var` 等目录，说明挂对了。

挂载故障系统 EFI：

```bash
sudo mkdir -p /mnt/boot/efi
sudo mount /dev/sda1 /mnt/boot/efi
```

---

## 4. 准备 chroot 环境

```bash
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /run /mnt/run
sudo cp /etc/resolv.conf /mnt/etc/resolv.conf
```

如果出现：

```text
cp: '/etc/resolv.conf' and '/mnt/etc/resolv.conf' are the same file
```

通常可忽略，这是符号链接/绑定挂载导致的常见提示。

---

## 5. 进入故障系统并切换显卡模式

```bash
sudo chroot /mnt
prime-select query
```

如果当前是 `intel`，先切到更稳的模式：

```bash
prime-select on-demand
```

在实际案例中，后续又切到了 `nvidia` 并成功恢复：

```bash
prime-select nvidia
update-initramfs -u
update-grub
```

> 建议：先用 `on-demand`，若你明确需要独显常驻再改 `nvidia`。

---

## 6. 常见报错处理

### 6.1 `prime-select: command not found`

在 `chroot` 内执行：

```bash
apt update
apt install --reinstall nvidia-prime ubuntu-drivers-common
prime-select on-demand
update-initramfs -u
update-grub
```

### 6.2 NVIDIA 驱动怀疑损坏

在 `chroot` 内执行：

```bash
ubuntu-drivers devices
ubuntu-drivers autoinstall
prime-select on-demand
update-initramfs -u
update-grub
```

### 6.3 `update-grub` 里 `os-prober` 警告

例如：

```text
Warning: os-prober will not be executed to detect other bootable partitions.
```

这通常不影响当前 Ubuntu 启动修复，可先忽略。

---

## 7. 正确退出与卸载

```bash
exit
sudo umount -R /mnt
sync
```

如果提示 busy：

```bash
cd ~
sudo umount -R /mnt
```

仍 busy 再排查占用：

```bash
sudo lsof +f -- /mnt
sudo fuser -vm /mnt
```

完成后安全移除 SSD，装回故障机器测试启动。

---

## 8. 一键命令模板（按顺序执行）

> 下面仅在你的故障盘确实是 `/dev/sda2` + `/dev/sda1` 时使用。

```bash
sudo umount /media/x/6f14b855-4c6b-451e-af4e-cfb74ec62395

sudo mount /dev/sda2 /mnt
ls /mnt

sudo mkdir -p /mnt/boot/efi
sudo mount /dev/sda1 /mnt/boot/efi

sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /run /mnt/run
sudo cp /etc/resolv.conf /mnt/etc/resolv.conf

sudo chroot /mnt
prime-select query
prime-select on-demand
prime-select nvidia
update-initramfs -u
update-grub
exit

sudo umount -R /mnt
sync
```

---

## 9. 这次问题的根因与经验总结

### 根因（本案例）

- 目标是“让 GPU 可休眠以降低风扇噪音”
- 执行 `prime-select intel` 后，Intel/NVIDIA 显示链路冲突
- 最终导致无法正常进入图形/TTY/GRUB 相关流程

### 经验

1. 不要在无法回滚的环境里直接切 `intel`  
2. 优先 `prime-select on-demand`，通常比纯 `intel` 更稳  
3. 操作前保留救援方案（Live USB 或本文的“拆 SSD 外接修复”）  
4. 修复时要确认自己在 `chroot /mnt` 里，避免误改救援机本体系统  

---

## 10. 最重要的安全提醒（务必看）

- **不要动当前救援机自己的系统盘分区**（例如本案例的 `nvme1n1p1/p2`）  
- 只对故障 SSD 的 root/EFI 分区操作（本案例是 `sda2/sda1`）  
- 所有设备名必须以你的 `lsblk -f` 输出为准  
- 如遇 LUKS 加密，需先 `cryptsetup open` 再挂载  
- 若修复后仍黑屏，检查 BIOS/UEFI 的 Secure Boot 设置与驱动签名问题  

---

如果你正好遇到和本文同类问题，这个方法通常能把系统从“完全进不去”状态拉回来。  
先按流程做一次，再考虑后续优化 GPU 策略（如 `on-demand` + 电源管理）来平衡噪音与稳定性。
