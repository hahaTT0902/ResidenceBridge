# ResidenceBridge

> 跨服领地桥接插件 —— 让 [Residence](https://www.spigotmc.org/resources/residence.11480/) 领地数据在 Velocity / BungeeCord 多服网络中无缝共享

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/hahaTT0902/ResidenceBridge/releases)
[![License: CC0](https://img.shields.io/badge/License-CC0%201.0-lightgrey.svg)](LICENSE)
[![Java](https://img.shields.io/badge/Java-8%2B-orange.svg)](https://adoptium.net)
[![Minecraft](https://img.shields.io/badge/Minecraft-1.16%2B-brightgreen.svg)](https://papermc.io)

## 简介

ResidenceBridge 是一款 Bukkit/Paper 服务端插件，专为在 **Velocity / BungeeCord** 代理网络中运行多个子服的 Minecraft 服务器设计。

它通过共享的 **MySQL 数据库** 维护一张全局领地索引表，实现：

- **全服唯一领地名** —— 任意子服创建/重命名领地时自动校验全网是否重名
- **跨服领地传送** —— 在任意子服执行 `/res tp <名称>` 可自动切换至领地所在服务器并完成传送
- **实时数据同步** —— 各子服按配置间隔将本服领地数据同步至 MySQL

> 无需在代理端安装任何插件，ResidenceBridge 仅部署在各子服。

---

## 功能特性

| 特性 | 说明 |
|------|------|
| 🏠 全局唯一领地名 | 玩家创建或重命名领地前自动检查全服是否重名，防止冲突 |
| 🚀 跨服领地传送 | `/res tp` 支持传送至任意子服的领地，自动完成服务器切换 |
| 🔄 定时数据同步 | 按可配置的时间间隔将本服领地快照写入 MySQL，自动清理已删除的领地 |
| ✏️ 重命名冲突检测 | 领地重命名时检查新名称是否在全服已被占用 |
| 🗑️ 自动删除同步 | 领地被删除后自动从全局索引中移除对应记录 |
| ⚡ Velocity & BungeeCord 双支持 | 同时兼容 Velocity 插件消息频道与 BungeeCord 消息格式 |

---

## 环境要求

| 组件 | 版本要求 |
|------|---------|
| Minecraft 服务端 | Paper / Spigot **1.16+**（推荐 1.20.x） |
| [Residence](https://www.spigotmc.org/resources/residence.11480/) | 最新稳定版，安装在各子服 |
| 代理端 | Velocity 3.x 或 BungeeCord（需开启插件消息转发） |
| 数据库 | MySQL **5.7+** 或 MariaDB **10.4+** |
| Java | **8+** |

---

## 安装步骤

1. 下载最新版 `ResidenceBridge-x.x.x.jar`，放入每台**子服**的 `plugins/` 目录。
2. 确保每台子服已安装 **Residence** 插件。
3. 启动子服，插件会在 `plugins/ResidenceBridge/config.yml` 生成默认配置文件。
4. 编辑 `config.yml`，填写 MySQL 连接信息，并为每台子服设置**唯一**的 `server-id`。
5. 重启服务器，或执行 `/rb reload` 重载配置。

---

## 配置文件

> 路径：`plugins/ResidenceBridge/config.yml`

```yaml
# 当前子服的唯一标识（不同子服必须不同，建议使用小写字母、数字和连字符）
server-id: "survival-1"

mysql:
  host: "127.0.0.1"
  port: 3306
  database: "minecraft"
  username: "root"
  password: "password"
  # HikariCP 连接池最大连接数
  maximum-pool-size: 10

sync:
  # 服务器启动后延迟多少 tick 进行首次同步（20 tick = 1 秒）
  initial-delay-ticks: 40
  # 定时同步间隔（秒）
  interval-seconds: 60
  # 每次同步成功后是否在控制台打印日志
  log-success: false

teleport:
  # 跨服传送请求的有效期（秒），超时后即使玩家进入目标服也不会自动传送
  pending-expire-seconds: 30
  # 玩家加入目标服后延迟多少 tick 再执行传送（建议保留，等待领地数据加载完毕）
  join-delay-ticks: 40

velocity:
  # Velocity 插件消息频道名称（需与代理端配置一致）
  channel: "residencebridge:main"
  # 是否同时发送 BungeeCord 兼容消息（使用 BungeeCord 代理时需开启）
  fallback-bungee-channel: true

messages:
  duplicate: "&c全服已存在同名领地：&f%name%"
  not-found: "&c没有找到这个领地：&f%name%"
  switching: "&a正在传送到领地所在服务器：&f%server%"
  local-teleport-failed: "&c本服领地传送失败，请联系管理员。"
  connect-request-failed: "&c跨服传送请求失败，请稍后再试。"
```

---

## 指令与权限

| 指令 | 权限节点 | 默认 | 说明 |
|------|---------|------|------|
| `/rb reload` | `residencebridge.command.reload` | OP | 重载插件配置，无需重启服务器 |
| `/residencebridge reload` | `residencebridge.command.reload` | OP | 同上（别名） |

> 所有权限节点默认仅限 OP，可通过 LuckPerms 等权限插件为特定群组授权。

---

## 工作原理

### 创建领地

```
玩家执行 /res create <名称>
        │
        ▼
  BridgePlugin 拦截命令
        │
        ├── 向 MySQL 全局索引尝试占位
        │       │
        │       ├── 占位失败（已存在同名）──► 提示玩家，取消命令
        │       │
        │       └── 占位成功 ──► 放行原始命令
        │                             │
        │                    Residence 创建领地
        │                             │
        └────────────────── 确认成功后写入完整快照
```

### 跨服领地传送

```
玩家执行 /res tp <名称>
        │
        ▼
  查询 MySQL 全局索引
        │
        ├── 本服领地 ──► 直接传送
        │
        └── 外服领地 ──► 写入 pending_tp 记录
                              │
                    发送插件消息，切换玩家至目标服
                              │
                    玩家加入目标服时读取 pending_tp
                              │
                         延迟后本服执行领地传送
```

---

## 数据库表

| 表名 | 用途 |
|------|------|
| `residence_bridge_index` | 全局领地索引，各子服定期同步写入 |
| `residence_bridge_pending_tp` | 待执行的跨服传送任务（一次性消费） |

表结构由插件在首次启动时自动创建，无需手动建表。

---

## 构建

**发行版（供服务器使用，不含 TabooLib 本体）：**

```bash
./gradlew build
```

产物路径：`build/libs/ResidenceBridge-x.x.x.jar`

**开发 API 版（仅供开发者依赖，包含 TabooLib 但移除了逻辑代码）：**

```bash
./gradlew taboolibBuildApi -PDeleteCode
```

> `-PDeleteCode` 参数会移除所有逻辑代码以大幅减小体积。

---

## 许可证

本项目以 [CC0 1.0 通用公共领域贡献](LICENSE) 协议发布，即放弃全部著作权，允许任何人以任何目的自由使用、修改和分发。