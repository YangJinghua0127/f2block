# Fail2Ban Block Manager

一个用于同步 Fail2Ban 封禁 IP 到 `iptables` 并记录到 SQLite 数据库的管理脚本，同时提供查询和统计功能。  
适合长期记录被封禁 IP 的信息，方便分析扫描攻击来源。

---

## 功能特点

- 自动同步 Fail2Ban 封禁 IP 到 `iptables`。
- 将 IP 封禁信息存储到 SQLite 数据库，避免日志文件过大。
- 支持查询最近封禁记录、统计信息、指定 IP 或国家的封禁记录。
- 数据库记录首次封禁时间和最近封禁时间。
- 简单易用的命令行参数操作。
- 数据库存储在 `~/.f2block/fail2ban_block.db`，无需担心日志混乱。

---

## 安装依赖

```bash
sudo apt update
sudo apt install -y sqlite3 curl iptables
```

将脚本保存为 `f2block` 并赋予执行权限：

```bash
chmod +x f2block
```

---

## 使用方法

### 同步 Fail2Ban 封禁列表

```bash
./f2block
```

脚本会自动：

1. 获取 `fail2ban-client status sshd` 当前封禁 IP 列表。
2. 检查 `iptables` 是否已封禁，未封禁则添加 DROP 规则。
3. 查询 IP 信息（国家、地区、城市、组织）。
4. 写入 SQLite 数据库（首次封禁记录 `first_seen`，最后封禁时间 `last_seen`）。

---

### 查询功能

| 参数       | 功能                                        |
| ---------- | ------------------------------------------- |
| `-l [N]`   | 查看最近 N 条封禁记录（默认 50 条）         |
| `-s`       | 显示统计信息（总封禁 IP 数量、Top 10 国家） |
| `-ip <IP>` | 查询指定 IP 的封禁信息                      |
| `-c <CC>`  | 查询指定国家（Country Code）的封禁记录      |
| `-h`       | 显示帮助信息                                |

#### 示例

```bash
./f2block -l         # 查看最近 50 条记录
./f2block -l 20      # 查看最近 20 条记录
./f2block -s         # 查看封禁统计
./f2block -ip 45.148.10.141  # 查询指定 IP
./f2block -c CN      # 查询中国封禁记录
./f2block -h         # 显示帮助
```

---

## 数据库说明

数据库文件位置：

```
~/.f2block/fail2ban_block.db
```

表结构：

| 列名         | 类型 | 说明            |
| ------------ | ---- | --------------- |
| `ip`         | TEXT | IP 地址（主键） |
| `country`    | TEXT | 国家代码        |
| `region`     | TEXT | 地区/省份       |
| `city`       | TEXT | 城市            |
| `org`        | TEXT | 组织/运营商     |
| `first_seen` | TEXT | 第一次封禁时间  |
| `last_seen`  | TEXT | 最近封禁时间    |

> 注意：同一个 IP 仅会插入一次，`last_seen` 会更新最新封禁时间。

---

## Cron 定时任务示例

可以将脚本加入 Cron，实现自动同步：

```bash
# 每 5 分钟同步一次 fail2ban 封禁 IP
*/5 * * * * f2block
```

---

## 注意事项

- 脚本默认只同步 `sshd` jail，如果你有其他 jail（如 nginx, apache），可以自行修改：

```bash
IPS=$(fail2ban-client status <jail_name> | sed -n '/Banned IP list:/s/.*Banned IP list:[[:space:]]*//p')
```

- `iptables` DROP 规则不会重复插入。
- 数据库不会无限增加重复记录，保证数据整洁。
