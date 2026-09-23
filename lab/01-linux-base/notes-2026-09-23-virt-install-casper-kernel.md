# 无图形环境下用 KVM 串口安装 Ubuntu Server（三次尝试与根因）

- **日期**：2026-09-23
- **耗时**：约 2 小时（其中大部分花在两次失败上）
- **结果**：✅ 成功进入安装界面并完成安装
- **宿主**：Ubuntu Server 22.04.5（待确认是否支持嵌套虚拟化）
- **环境归属**：`lab/01-linux-base/` —— KVM 实验虚拟机环境

---

## 一句话结论

**无图形环境下不要用 `virt-install --cdrom`**；应改用 `--location`，
并**显式指定** `kernel=casper/hwe-vmlinuz,initrd=casper/hwe-initrd`——
因为 Ubuntu **live-server** ISO 使用 **casper** 布局，而 `virt-install`
默认按 Debian Installer 风格的路径（`/install/vmlinuz`）去探测内核，
必然失败。

---

## 目的

通过 SSH 登录 Ubuntu Server 主机，在**无图形界面**的环境下用 KVM
创建虚拟机实例，作为后续实验的环境。

---

## 环境

| 项 | 值 |
|---|---|
| 宿主系统 | Ubuntu Server 22.04.5 |
| 访问方式 | SSH（无图形界面，这一点是问题的前提） |
| 组件 | `qemu-kvm`、`libvirt-daemon-system`、`virtinst` |
| 镜像 | `ubuntu-22.04.5-live-server-amd64.iso` |
| 镜像路径 | `/var/lib/libvirt/boot/` |
| 网络 | `kvm-wan`、`lab-db`（定义方式未记录，见待办） |

---

## 过程

### 尝试 1：`--cdrom`（失败）

```bash
sudo virt-install \
  --name lab-net-test \
  --memory 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/lab-net-test.qcow2,size=20,format=qcow2,bus=virtio \
  --cdrom /var/lib/libvirt/boot/ubuntu-22.04.5-live-server-amd64.iso \
  --network network=kvm-wan,model=virtio \
  --network network=lab-db,model=virtio \
  --os-variant ubuntu22.04 \
  --graphics none \
  --console pty,target_type=serial
```

- **预期**：进入安装界面
- **实际**：卡在以下输出，无法进入安装程序

```
Starting install... Creating domain... | 0 B 00:00:00
Running text console command: virsh --connect qemu:///system console lab-net-test
Connected to domain 'lab-net-test'
Escape character is ^] (Ctrl + ])
```

- **根因**：`--cdrom` 需要一个**图形界面**（VNC/SPICE）来呈现安装程序，
  在 `--graphics none` 下没有可用的显示输出，因此只停在串口控制台，
  安装程序不会启动。
- **结论**：改用 `--location`（从安装树启动，支持串口安装）

### 尝试 2：`--location` + `--extra-args`（失败）

```bash
sudo virt-install \
  --name lab-net-test \
  --memory 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/lab-net-test.qcow2,size=20,format=qcow2,bus=virtio \
  --location /var/lib/libvirt/boot/ubuntu-22.04.5-live-server-amd64.iso \
  --network network=kvm-wan,model=virtio \
  --network network=lab-db,model=virtio \
  --os-variant ubuntu22.04 \
  --graphics none \
  --console pty,target_type=serial \
  --extra-args 'console=ttyS0,115200n8'
```

- **预期**：进入安装界面
- **实际**：

```
Starting install...
ERROR    Couldn't find kernel for install tree.
Domain installation does not appear to have been successful.
If it was, you can restart your domain by running:
  virsh --connect qemu:///system start lab-net-test
otherwise, please restart your installation.
```

- **根因**：`virt-install` 在 `--location` 指向的 ISO 中，**没有找到它预期位置的内核和 initrd**。
  它按 `os-variant=ubuntu22.04` 的规则去找 **Debian Installer 风格**的路径
  （如 `/install/vmlinuz`、`/install/netboot/...`），
  但 Ubuntu 22.04 的 **live-server** ISO 用的是 **casper** 布局，
  内核位于 `/casper/` 目录下，因此自动探测失败。
- **结论**：显式指定 kernel 与 initrd 路径

### 尝试 3：显式指定 kernel / initrd（✅ 成功）

关键参数：

```
kernel=casper/hwe-vmlinuz,initrd=casper/hwe-initrd
```

完整命令：

```bash
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
```

- **结果**：✅ 通过串口进入安装界面，完成安装
- 进入控制台的方式：`virsh console lab-net-test`（退出：`Ctrl + ]`）

---

## 验证

> 🔴 **待补**：以下是需要补齐的验证项，补完后本记录才算闭合。

```bash
virsh list --all              # 状态应为 running
virsh domifaddr lab-net-test  # 应能看到两张网卡的 IP
ssh <user>@<ip>               # 应能登录
```

- [ ] 待补：`virsh list --all` 输出
- [ ] 待补：`virsh domifaddr` 输出（确认双网卡都拿到 IP）
- [ ] 待补：SSH 登录成功截图/输出

---

## 关键结论

| # | 结论 | 可迁移性 |
|---|---|---|
| 1 | 无图形环境不要用 `--cdrom`；用 `--location` + `--graphics none --console pty,target_type=serial` | 通用 |
| 2 | Ubuntu **live-server** ISO 是 casper 布局，需显式指定 `kernel=casper/hwe-vmlinuz,initrd=casper/hwe-initrd` | 通用（所有 Ubuntu live-server 版本） |
| 3 | `hwe-` 前缀是 Hardware Enablement 内核；若在特定平台上失败，可尝试非 hwe 版本 | 通用 |
| 4 | `--extra-args` 结尾的 `---` 用于跳过交互提示，避免安装被阻塞 | 通用 |
| 5 | **排障顺序：先确认"启动方式是否适配当前环境"（图形 vs 串口），再查"工具的默认假设是否匹配镜像布局"——不要先怀疑命令写错** | ★ 方法论 |

---

## 反思：我走了哪些弯路

- **尝试 1 失败后，第一反应是"参数写错了"**，于是去翻 `virt-install` 的参数列表，
  而不是先想"这个环境（无图形）是否支持这种启动方式"。
  → **应当先判断"方案与环境的适配性"，再抠参数细节。**
- **尝试 2 的报错信息其实已经把原因写清楚了**（`Couldn't find kernel for install tree`），
  但我一开始没意识到 `os-variant` 会决定**内核探测路径**。
  → **报错里出现的"找不到 X"，要立刻追问"它在哪个路径找 X、为什么是这个路径"。**

---

## 待办 / 下一步

- [ ] **补 `kvm-wan` / `lab-db` 两个 libvirt 网络的创建方式**（XML + `virsh net-define`）
      —— 当前是最大缺口，缺了它本记录**无法全新复现**
- [ ] 补上"验证"一节的真实输出
- [ ] 确认宿主是否支持嵌套虚拟化（决定此方案能否在云主机上复用）
- [ ] 用同一套参数再建 2 台 VM，验证 `kvm-wan` 出网 + `lab-db` 内网互通
- [ ] 网络配置理清后，整理成本目录 `README.md` 的标准步骤

---

## 参考

- `man virt-install` —— `--location`、`--cdrom`、`kernel=`/`initrd=` 参数说明
- `virsh` 子命令：`console`、`list`、`domifaddr`、`net-define`
