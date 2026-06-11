# f2block

Fail2Ban 威胁追踪系统 — 实时监控 Fail2Ban 封禁事件，自动查询 IP 地理信息，记录到 SQLite 数据库并执行 iptables 全端口封禁。

## 功能

- **实时监控** — 守护进程跟踪 `fail2ban.log`，捕获 Ban 事件并自动处理
- **IP 地理信息** — 通过 ipinfo.io 查询 IP 归属（国家/地区/城市/组织），已有记录自动跳过 API 调用
- **iptables 封禁** — 自动对被封 IP 执行全端口 DROP 规则
- **白名单** — 白名单中的 IP 永不封禁，添加时自动解封
- **启动回扫** — 守护进程启动时自动回扫日志中已有事件，重启不丢数据
- **日志轮转感知** — 检测 logrotate 导致的日志轮转，自动回扫新文件
- **数据导出** — 支持 CSV / JSON 格式导出

## 依赖

- `bash`
- `sqlite3`
- `iptables`
- `curl`
- `fail2ban`（已运行）
- `jq`（可选，用于更可靠的 JSON 解析）

## 安装

```bash
# 克隆到 /root/.f2block
git clone https://github.com/YOUR_USERNAME/f2block.git /root/.f2block

# 添加执行权限
chmod +x /root/.f2block/f2block /root/.f2block/f2blockd

# 创建软链接（可选，方便全局调用）
ln -s /root/.f2block/f2block /usr/local/bin/f2block

# 安装 systemd 服务
cp /root/.f2block/f2block.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now f2block.service
```

## 文件说明

| 文件 | 说明 |
|------|------|
| `f2blockd` | 守护进程，监控日志并处理封禁事件 |
| `f2block` | 命令行查询工具 |
| `f2block.service` | systemd 服务单元文件 |
| `f2block.db` | SQLite 数据库（自动创建） |
| `allowlist.txt` | 白名单（自动创建） |

## 用法

### 查询

```bash
f2block -l              # 查看最近50条封禁记录
f2block -l 20           # 查看最近20条
f2block -s              # 统计信息（IP数、事件数、国家Top10）
f2block -ip 1.2.3.4     # 查询指定IP
f2block -c CN           # 按国家查询
f2block -j sshd         # 按 jail 查询
f2block -top            # 封禁次数最多的IP
f2block -events 50      # 最近封禁事件
```

### 管理

```bash
f2block -unban 1.2.3.4  # 解封IP（移除iptables + 删除数据库记录）
f2block -allow 10.0.0.1 # 添加到白名单（自动解封）
f2block -allowlist      # 查看白名单
f2block -allowdel 10.0.0.1  # 从白名单移除
```

### 导出

```bash
f2block -export csv     # 导出为 CSV
f2block -export json    # 导出为 JSON
```

### 服务管理

```bash
systemctl status f2block     # 查看状态
systemctl restart f2block    # 重启
systemctl stop f2block       # 停止
journalctl -u f2block -f     # 查看实时日志
```

## 数据库结构

### blocked_ips

| 字段 | 类型 | 说明 |
|------|------|------|
| ip | TEXT | IP 地址（主键） |
| country | TEXT | 国家代码 |
| region | TEXT | 地区 |
| city | TEXT | 城市 |
| org | TEXT | 组织/ISP |
| first_seen | TEXT | 首次发现时间 |
| last_ban_time | TEXT | 最后封禁时间 |
| ban_count | INTEGER | 封禁次数 |

### ban_events

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER | 自增主键 |
| ip | TEXT | IP 地址 |
| jail | TEXT | Fail2Ban jail 名称 |
| ban_time | TEXT | 封禁时间 |

## 配置

守护进程中的可调参数（`f2blockd` 顶部）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `LOGFILE` | `/var/log/fail2ban.log` | Fail2Ban 日志路径 |
| `API_DELAY` | `1` | ipinfo.io 请求间隔（秒） |

## License

MIT
