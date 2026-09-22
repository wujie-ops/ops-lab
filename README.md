# ops-lab

我的运维实验环境、脚本库与知识沉淀。

## 这是什么

一个持续更新的运维工作台：把学到的东西动手做一遍、把踩过的坑记下来、
把重复劳动写成脚本。所有实验都可在自己的环境复现。

## 目录

| 目录                                             | 内容                                                         |
| ------------------------------------------------ | ------------------------------------------------------------ |
| [`lab/`](lab/)                                   | 实验环境：Linux、Nginx、MySQL、Redis、Docker、监控、Ansible、云 |
| [`scripts/`](scripts/)                           | 脚本库：备份、巡检、部署（见 [脚本清单](scripts/README.md)） |
| [`docs/notes/`](docs/notes/)                     | 学习笔记（按主题编号）                                       |
| [`docs/troubleshooting/`](docs/troubleshooting/) | 故障记录与复盘                                               |
| [`docs/diary/`](docs/diary/)                     | 实验日志（按月）                                             |

## 实验环境 （规划）

| 角色        | 规格                       | 用途                |
| ----------- | -------------------------- | ------------------- |
| 控制节点    | ubuntu server 22.04 / 2C4G | Ansible、脚本开发   |
| 应用节点 ×2 | ubuntu server 22.04 / 2C4G | Nginx、Tomcat、应用 |
| 数据库      | ubuntu server 22.04 / 2C4G | MySQL 主从、Redis   |
| 监控        | ubuntu server 22.04 / 2C2G | Prometheus、Grafana |
| 云环境      | 待定                       | VPC、CLB、RDS 实验  |

## 内容索引 （规划）

完整索引见 [`INDEX.md`](INDEX.md)。

## 相关项目 （规划）

完整项目已独立成仓：
- [fault-drills](../fault-drills) — 6 类故障演练与复盘
- [prometheus-stack](../prometheus-stack) — 监控告警体系

## 联系

GitHub: [@wujie-ops](https://github.com/wujie-ops) · Email: wujie.ops@gmail.com