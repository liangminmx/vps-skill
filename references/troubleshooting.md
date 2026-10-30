# VPS SSH 常见问题速查

## 1. PubkeyAuthentication 被禁用

**症状**：密钥已写入 authorized_keys，但 SSH 只提示 `Permission denied (password)`，
不提 `(publickey)`。说明 sshd 根本没检查公钥。

**一步修复**：
```bash
ssh root@host "sed -i 's/PubkeyAuthentication no/PubkeyAuthentication yes/' /etc/ssh/sshd_config"
# 服务名可能是 ssh 或 sshd，先确认：
ssh root@host "systemctl list-units | grep -i ssh"
# 重载：
ssh root@host "systemctl reload ssh"   # 或 systemctl reload sshd
```

## 2. OOM Killer 导致服务全挂（小内存 Oracle 机器常见）

**症状**：ping 通但 SSH / HTTP 全不通。Cloudflare 返回 522。

**原因**：机器内存不足（如 956M），多个服务（komari、nginx、fail2ban 等）
同时运行触发 OOM Killer，内核杀掉关键进程（sshd、nginx），只剩内核存活。

**修复**：只能从 VPS 面板（Oracle Cloud Console）硬重启。

**预防**：
- 清掉不必要的服务（如空跑的 Docker，吃 ~66MB）
- 控制同时运行的服务数量
- 加 swap（已有则确认生效）

## 3. 批量迁移 VPS 到统一密钥

从多把不同密钥迁移到单一 `~/.ssh/vps` 的完整流程：

```
# 1. 生成统一密钥
ssh-vps.sh init-key

# 2. 密码型机器：add 一条龙
ssh-vps.sh add <alias> <ip> --password '<pw>'

# 3. 密钥型机器：手动下发
ssh -i old_key user@host "echo 'vps-pubkey-content' >> ~/.ssh/authorized_keys"
# 验证
ssh -i ~/.ssh/vps user@host "echo OK"
# 加别名
ssh-vps.sh add <alias> <ip> --user <user> --port <port>

# 4. 全量验证
for a in alias1 alias2 ...; do ssh $a "echo OK"; done
```

## 4. Linux 上 vps.config 包含 macOS 选项导致全部 SSH 断开

**症状**：`Bad configuration option: usekeychain`，所有连接不可用。

**修复**：
```bash
sed -i '/UseKeychain yes/d; /AddKeysToAgent yes/d' ~/.ssh/vps.config
```

## 5. Permission denied — 不是密钥不对，是用户不对

**症状**：统一密钥（~/.ssh/vps）和所有已知密码都试过了，`Permission denied (publickey)` 或 `Permission denied, please try again`，就是不让你进。试了 root 用户、试了多把密钥，全拒。

**根因**：**不是密钥不对，是用户名不对**。Oracle Cloud / AWS 等云厂商的初始默认用户不是 `root`：

| 厂商 | 默认用户 |
|------|----------|
| Oracle Cloud (Ubuntu) | `ubuntu` |
| Oracle Cloud (Oracle Linux) | `opc` |
| AWS Ubuntu | `ubuntu` |
| AWS Amazon Linux | `ec2-user` |
| Google Cloud | 自定义，root 通常禁用 |

**诊断**：SSH `-vvv` debug 输出中若只看到 `Authentications that can continue: publickey` 且所有密钥都被拒，99% 是用户名不对。

**修复**：
```bash
# 别假设 root，先试常见默认用户
ssh -i ~/.ssh/vps -p <port> ubuntu@<host>
ssh -i ~/.ssh/vps -p <port> opc@<host>
ssh -i ~/.ssh/vps -p <port> ec2-user@<host>
```

**反例**：本次 session 连 `161.33.32.236:20266`，用 root + 9 把密钥全试了一遍全拒，用户纠正为 `ubuntu` 后一把过。