# VPS 批量迁移到统一密钥 — 实战配方

## 场景
有多台 VPS，使用不同 SSH 密钥/密码，需要全部迁移到统一 `~/.ssh/vps` 密钥管理。

## 前置
```bash
ssh-keygen -t ed25519 -f ~/.ssh/vps -C "vps-key" -N ""
```

## 步骤

### 1. 密码型 VPS — 用 sshpass 下发公钥
```bash
sshpass -p '<password>' ssh -p <port> -o StrictHostKeyChecking=no root@<host> \
  "echo '<pubkey>' >> ~/.ssh/authorized_keys"
```

### 2. 密钥型 VPS — 用现有密钥下发新公钥
```bash
ssh -i /root/.ssh/<old_key> -p <port> -o StrictHostKeyChecking=no <user>@<host> \
  "echo '<pubkey>' >> ~/.ssh/authorized_keys"
```

### 3. 注册别名（不带 --password，密钥已在远端）
```bash
bash scripts/ssh-vps.sh add <alias> <host> --user <user> --port <port>
```

### 4. 验证
```bash
ssh <alias> "uptime"
```

## 已踩坑（2026-06-26）

| 坑 | 症状 | 修复 |
|----|------|------|
| UseKeychain yes | Linux SSH 全部报 bad config option，所有连接中断 | 已从脚本和 SKILL.md 移除 |
| expect 未安装 | `add --password` 报 command not found，别名创建但密钥未同步 | 用 sshpass 手动下发替代 |
| `IdentitiesOnly yes` + 别名 | 不影响，配合 `-i ~/.ssh/vps` 显式指定即可 |

## 8 机迁移实录（2026-06-26）
vmiss, dedirock, 61, zouter-jp, netcup, jgw, hk, tanzhen 全部成功。
vmiss 额外需要 `PubkeyAuthentication yes`（原本被关了）。