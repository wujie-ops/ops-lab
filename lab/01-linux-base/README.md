# KVM 实验虚拟机环境

- **宿主**：待补（本机 / 云主机？规格？）
- **系统**：Ubuntu Server 22.04.5
- **搭建日期**：2026-09-23
- **可复现性**：🟡 部分步骤依赖手动操作（网络定义文件未记录，见下方待补项）
- **预计耗时**：约 30 分钟（首次；踩坑过程见 [notes-2026-09-23](notes-2026-09-23-virt-install-casper-kernel.md)）

---

## 目标

用 KVM 在本地跑多台虚拟机，作为后续实验的底座——不需要为每个实验单独买云主机。

待补：明确要支撑哪些实验场景（例如：MySQL 主从复制、Nginx 负载均衡、多机故障演练）。

---

## 环境与前提

| 项 | 值 |
|---|---|
| 宿主系统 | Ubuntu Server 22.04.5 |
| CPU / 内存 / 磁盘 | 待补 / 待补 / 待补 |
| 虚拟化支持 | 待验证：`egrep -c '(vmx\|svm)' /proc/cpuinfo`（返回 > 0 为支持） |
| 组件 | `qemu-kvm`、`libvirt-daemon-system`、`virtinst` |
| 镜像 | `ubuntu-22.04.5-live-server-amd64.iso` |
| 镜像存放路径 | `/var/lib/libvirt/boot/` |
| 磁盘镜像目录 | `/var/lib/libvirt/images/` |

> ⚠️ **如果宿主是云主机**：必须确认支持**嵌套虚拟化**，否则会报
> `KVM acceleration can not be used`。用 `kvm-ok` 或
> `egrep -c '(vmx|svm)' /proc/cpuinfo` 验证。
> （本环境宿主类型待确认，这是决定能否复用本方案的关键前提。）

---

## 网络规划

> 🔴 **待补**：本节是当前记录最大的缺口——命令里引用了
> `network=kvm-wan` 和 `network=lab-db`，但**没有记录这两个网络是怎么创建的**，
> 因此本环境目前**无法全新复现**。

| libvirt 网络 | 类型 | 网段 | 作用 | 状态 |
|---|---|---|---|---|
| `kvm-wan` | 待补（推测 NAT） | 待补 | 待补（推测：VM 出网、装软件） | 已存在 |
| `lab-db` | 待补（推测 isolated） | 待补 | 待补（推测：数据库内网通信） | 已存在 |

**待补内容**（补完请删掉上面表格里的"待补"）：

- [ ] 两个网络的 XML 定义文件（放本目录 `networks/` 下）
- [ ] 创建命令，例如：
      `virsh net-define kvm-wan.xml && virsh net-start kvm-wan && virsh net-autostart kvm-wan`
- [ ] 确认开机自启是否已配置：`virsh net-list --all`
- [ ] 网段划分理由（为什么用这个网段、为什么需要两张网卡）
- [ ] VM 之间的连通性验证结果

---

## 步骤

```bash
# 1. 安装组件
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system virtinst bridge-utils

# 2. 确认 libvirtd 运行
systemctl status libvirtd

# 3. 准备镜像目录与 ISO
sudo mkdir -p /var/lib/libvirt/boot
# 将 ubuntu-22.04.5-live-server-amd64.iso 放到该目录

# 4. 创建虚拟机（本方案唯一验证过的形式，参数说明见下节）
sudo virt-install \
  --name lab-net-test \
  --memory 2048 --vcpus 2 \
  --disk path=/var/lib/libvirt/images/lab-net-test.qcow2,size=20,format=qcow2,bus=virtio \
  --location /var/lib/libvirt/boot/ubuntu-22.04.5-live-server-amd64.iso,kernel=casper/hwe-vmlinuz,initrd=casper/hwe-initrd \
  --network network=kvm-wan,model=virtio \
  --network network=lab-db,model=virtio \
  --os-variant ubuntu22.04 \
  --graphics none \
  --console pty,target_type=serial \
  --extra-args 'console=ttyS0,115200n8 ---'

# 5. 连接串口控制台，在控制台内完成安装
virsh console lab-net-test
# 退出控制台：Ctrl + ]
```

---

## 关键参数说明

| 参数 | 作用 | 不做会怎样 |
|---|---|---|
| `--location ...` 代替 `--cdrom` | 从安装树启动，支持串口输出 | 卡在 `Escape character is ^]`，进不去安装程序 |
| `kernel=casper/hwe-vmlinuz,initrd=casper/hwe-initrd` | 显式指定内核与 initrd 路径 | 报 `ERROR Couldn't find kernel for install tree` |
| `--graphics none --console pty,target_type=serial` | 无图形环境走串口控制台 | 无界面可用，装不了 |
| `--extra-args 'console=ttyS0,115200n8 ---'` | 内核输出转串口；结尾 `---` 防止交互提示阻塞 | 控制台黑屏 / 安装被提示卡住 |
| `bus=virtio`、`model=virtio` | 使用 virtio 半虚拟化驱动 | 性能差（但兼容性更好） |

> 完整踩坑过程与根因分析见
> [notes-2026-09-23-virt-install-casper-kernel.md](notes-2026-09-23-virt-install-casper-kernel.md)

---

## 验证

> 🔴 **待补**：本节需要真实命令输出，不能只写"已完成"。

```bash
# 虚拟机状态
virsh list --all

# 网卡与 IP
virsh domifaddr lab-net-test

# 从宿主机登录（用安装时创建的用户）
ssh <user>@<ip>

# 在 VM 内验证出网
ping -c 2 8.8.8.8
ping -c 2 www.aliyun.com
```

> 待补：以上命令的实际输出截图或文本。

---

## 资源占用

| 项 | 值 |
|---|---|
| 单台 VM 规划 | 2 vCPU / 2 GB / 20 GB qcow2 |
| 当前运行台数 | 待补 |
| 实际磁盘占用 | 待补（`du -sh /var/lib/libvirt/images/`） |
| 宿主可再承载 | 待补 |

---

## 清理

```bash
# 停止并删除虚拟机（含磁盘）
virsh destroy lab-net-test
virsh undefine lab-net-test --remove-all-storage

# 仅停止，保留磁盘
virsh shutdown lab-net-test
```

---

## 目录内容

| 文件 | 说明 |
|---|---|
| `README.md` | 本文件：环境的标准搭建步骤（SOP） |
| `notes-2026-09-23-virt-install-casper-kernel.md` | 2026-09-23 的过程记录：三次尝试与根因 |
| `TEMPLATE-notes.md` | 过程记录模板（以后写新记录时复制它） |
| `networks/` | 待补：libvirt 网络的 XML 定义 |

---

## 待办

- [ ] 补 `kvm-wan` / `lab-db` 网络的 XML 与创建命令（**阻塞复现，最高优先**）
- [ ] 补"验证"一节的真实输出
- [ ] 补宿主规格与虚拟化支持确认
- [ ] 确认目标实验场景，把"目标"一节写实
- [ ] 用同一套参数再建 2 台 VM，验证双网卡互通
