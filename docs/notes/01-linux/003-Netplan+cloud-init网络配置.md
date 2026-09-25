Netplan + cloud-init网络配置笔记

> Netplan 持久化、cloud-init 冲突、配置优先级、NoCloud 数据源、KVM Wi-Fi Bridge、云原生配置管理、常见错误排查。

## 一、核心速查表

| 问题                                | 结论                                                         |
| :---------------------------------- | :----------------------------------------------------------- |
| Netplan 配置重启会重置吗？          | 正常不会。配置持久化在 `/etc/netplan/*.yaml`。               |
| 只执行 `sudo netplan try` 呢？      | 只影响运行时，超时回滚运行时；不修改 YAML，重启后仍按 YAML 生效。 |
| 为什么重启后配置恢复默认？          | 多半是 `cloud-init` 重写了 `/etc/netplan/50-cloud-init.yaml`。 |
| 不要改哪个文件？                    | 不要直接改 `50-cloud-init.yaml`，它是 cloud-init 的受管文件。 |
| 不禁用 cloud-init，如何让配置生效？ | 创建 `60-custom.yaml`，文件名排序在 `50` 之后，后加载覆盖先加载。 |
| 更规范的做法？                      | 用 NoCloud seed ISO 提供 `network-config`，让 cloud-init 消费外部数据源。 |
| KVM 能桥接 Wi-Fi 吗？               | 通常不行，需 4addr/WDS 且 AP 支持。推荐 NAT、路由或 USB 直通。 |
| 云原生中重启算故障吗？              | 通常不算，服务不可用或违反 SLO 才算。                        |
| 出现两个 IPv4 地址？                | 多个配置源同时给同一接口分配地址，Netplan 合并导致。         |
| 重复 SSID 报错？                    | `50-cloud-init.yaml` 和自定义文件都定义了同一 Wi-Fi SSID，合并冲突。 |

------

## 二、Netplan 基础

### 1. 是什么

Netplan 是 Ubuntu 上的网络配置抽象层。
用 YAML 描述网络，Netplan 生成后端配置：

- `systemd-networkd`（Server 默认）
- `NetworkManager`（Desktop / Wi-Fi 常用）

### 2. 配置文件位置

```bash
/etc/netplan/*.yaml
```

权限要求：

```bash
sudo chmod 600 /etc/netplan/*.yaml
```

### 3. 基本结构

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 114.114.114.114]
```

### 4. 常用命令

```bash
sudo netplan generate      # 生成后端配置，不应用
sudo netplan apply         # 应用配置
sudo netplan try           # 临时应用，超时回滚
sudo netplan get           # 查看合并后的配置（较新版本）
sudo netplan status        # 查看状态（较新版本）
```

## 三、配置加载与优先级

### 1. 加载规则

Netplan 按文件名的**字典序**加载 `/etc/netplan/` 下所有 `*.yaml`。
**后加载的文件覆盖先加载文件中相同的配置项。**

### 2. 典型文件

```bash
50-cloud-init.yaml   # cloud-init 生成，每次启动可能重写
60-custom.yaml       # 你自定义，后加载，覆盖 50
```

### 3. 覆盖示例

`50-cloud-init.yaml`：

```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: true
```

`60-custom.yaml`：

```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: no
      addresses: [192.168.124.200/24]
      routes:
        - to: default
          via: 192.168.124.1
```

最终生效：`dhcp4: no` + 静态地址。
**建议在 60 中写完整接口配置，避免和 50 的 DHCP 冲突。**

### 4. 验证最终配置

```bash
sudo netplan get
sudo netplan generate
cat /run/systemd/network/*.network
ip a
ip route
```

## 四、cloud-init 与 Netplan

### 1. cloud-init 是什么

云环境/虚拟机启动时的初始化框架。
负责：主机名、用户、SSH 密钥、网络、磁盘、软件包、执行脚本。

### 2. 为什么覆盖 Netplan

在 Ubuntu 上，cloud-init 默认使用 Netplan 作为网络渲染器。
它每次启动可能重新生成 `/etc/netplan/50-cloud-init.yaml`，并调用 `netplan apply`。

### 3. 数据源（Datasource）

cloud-init 从外部数据源读取配置：

| 数据源       | 提供方               | 场景            |
| :----------- | :------------------- | :-------------- |
| 云平台元数据 | AWS/阿里云/OpenStack | 公有云          |
| NoCloud      | 你自己               | KVM、本地虚拟机 |
| ConfigDrive  | OpenStack 等         | 私有云          |

如果没有数据源，cloud-init 回退默认逻辑：第一个网卡 DHCP，生成 `50-cloud-init.yaml`。

### 4. 查看 cloud-init 状态

```bash
cloud-init status --long
sudo cat /var/log/cloud-init.log
sudo cat /run/cloud-init/instance-data.json
```

## 五、让自定义配置生效的三种方法

### 方法一：`60-custom.yaml` 覆盖（简单，推荐本地测试）

bash

```bash
sudo vim /etc/netplan/60-custom.yaml
sudo chmod 600 /etc/netplan/60-custom.yaml
sudo netplan apply
```

- 文件名排序大于 `50`，后加载覆盖 `50-cloud-init.yaml`。
- 重启后依然生效，因为文件还在。
- 缺点：配置仍在实例内部，重建实例会丢失。

### 方法二：cloud-init `write_files`（次规范）

```bash
sudo tee /etc/cloud/cloud.cfg.d/90-custom-netplan.cfg <<'EOF'
#cloud-config
write_files:
  - path: /etc/netplan/60-custom.yaml
    permissions: '0600'
    content: |
      network:
        version: 2
        ethernets:
          enp1s0:
            dhcp4: false
            addresses: [192.168.124.200/24]
            routes:
              - to: default
                via: 192.168.124.1
            nameservers:
              addresses: [8.8.8.8, 114.114.114.114]
EOF

sudo cloud-init single --name write-files
sudo netplan apply
```

### 方法三：NoCloud `network-config` 数据源（最规范）

- 在 KVM 宿主机准备 seed ISO。
- ISO 卷标必须是 `cidata`。
- 包含 `meta-data`、`user-data`、`network-config`。
- `network-config` 就是 Netplan YAML。
- 虚拟机启动时挂载，cloud-init 直接消费，不再生成默认 `50-cloud-init.yaml`。

生成示例：

```bash
mkdir -p ~/seed && cd ~/seed
# 编写 meta-data, user-data, network-config
genisoimage -output seed.img -volid cidata -rational-rock -joliet \
  user-data meta-data network-config
```

启动虚拟机时挂载：

```bash
virt-install ... --disk path=seed.img,device=cdrom ...
```

## 六、配置外置与“消费外部数据源”

### 1. 配置外置

配置的权威来源在实例外部，实例启动时自动拉取并收敛。

| 配置源       | 谁提供 | 传输方式      |
| :----------- | :----- | :------------ |
| 云平台元数据 | 云平台 | HTTP/虚拟设备 |
| NoCloud      | 你自己 | seed ISO      |
| Git          | 团队   | 工具拉取      |
| 配置中心     | 运维   | API           |

### 2. “消费”是什么意思

英文 `consume`，在 IT 里意思是 **读取并使用**。
“cloud-init 消费外部数据源” = cloud-init 从外部读取配置并应用到系统。

### 3. 规范链路

```bash
外部数据源声明配置 → cloud-init 读取 → 生成内部配置 → 系统收敛
```

## 七、云原生实践视角

### 1. 重启算服务故障吗？

- 通常不算。重启是正常生命周期事件。
- 只有服务不可用、违反 SLO、需要人工救火时才算故障。

### 2. 配置管理原则

- 声明式，而非命令式。
- 不可变基础设施。
- 配置外置。
- 幂等收敛。
- 可重复。

### 3. 手工改 `50-cloud-init.yaml` 为什么不符合云原生实践

- 手工 SSH 进去改文件。
- 改的是 cloud-init 的受管文件。
- 配置只存在实例磁盘，重建即丢。
- 重启后 cloud-init 按自己的声明重新生成，覆盖修改。

规范流程：

- 用 NoCloud `network-config` 声明网络。
- 或用 `60-custom.yaml` 覆盖。
- 或用 cloud-init `write_files` 声明。
- 生产环境用 Terraform/Ansible + Git 管理。

------

## 八、常见错误与解决

### 1. 重复 SSID 报错

**错误：**

```bash
/etc/netplan/60-custom.yaml:11:17: Error in network definition:
wlp2s0: Duplicate access point SSID 'wifi-name'
```

**原因：**
`50-cloud-init.yaml` 和 `60-custom.yaml` 都定义了 `wlp2s0` 的同一个 SSID，Netplan 合并后重复。

**解决：**
这是临时办法，为了最快恢复网络，直接手动修复重复 SSID。

### 2. 出现两个 IPv4 地址

**现象：**

```bash
IPv4 address for wlp2s0: 192.168.124.200
IPv4 address for wlp2s0: 192.168.124.22
```

**原因：**
多个配置源同时给 `wlp2s0` 分配地址，Netplan 合并后两个都生效。

**排查：**

```bash
ip -4 addr show wlp2s0
ls -l /etc/netplan/
cat /etc/netplan/*.yaml
sudo netplan get
nmcli device status
```

**解决：**

- 【手工修复重复配置，通常是配置了多个静态IP，或者开启的 dhcp 同时设置了静态IP】
- 禁用 cloud-init 网络管理，删除 `50-cloud-init.yaml`。
- 确保 `60-custom.yaml` 里 `addresses` 只有一份，`dhcp4: no`。

### 3. Wi-Fi 配置建议

```yaml
network:
    version: 2
    wifis:
        wlp2s0:
            access-points:
                wifi-name:
                    password: password
            dhcp4: no
            addresses:
              - 192.168.124.100/24
            routes:
              - to: default
                via: 192.168.124.1
            nameservers:
              addresses:
                - 202.96.128.86
                - 202.96.128.166
```

要点：

- 加 `renderer: NetworkManager`。
- 用 `match` 匹配 MAC，避免接口名变化。
- 权限 `600`。
- 确认静态 IP 不在 DHCP 池内。

## 九、排查命令清单

### Netplan 配置不生效或重启丢失

```bash
# 1. 查看配置文件
ls -l /etc/netplan/
cat /etc/netplan/*.yaml

# 2. 查看 cloud-init 是否在管理网络
cloud-init status --long
ls -l /etc/cloud/cloud.cfg.d/
cat /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# 3. 查看合并后的配置
sudo netplan get
sudo netplan generate

# 4. 查看实际网络
ip a
ip route
resolvectl status

# 5. 查看日志
sudo journalctl -u systemd-networkd
sudo cat /var/log/cloud-init.log
```

## 十、一句话总结

> **Netplan 配置写在 `/etc/netplan/\*.yaml`，正常重启不会丢。**
>
> **重启后恢复默认，通常是因为 cloud-init 重写了 `50-cloud-init.yaml`。**
>
> **不要直接改 `50-cloud-init.yaml`。**
>
> **不禁用 cloud-init 时，用 `60-custom.yaml` 覆盖，或用 NoCloud `network-config` 让 cloud-init 消费外部数据源。**
>
> **KVM 里 Wi-Fi Bridge 通常不可行，优先 NAT、路由或 USB 直通。**
>
> **云原生实践：配置外置、声明式、幂等，重启是常态，服务不可用才是故障**