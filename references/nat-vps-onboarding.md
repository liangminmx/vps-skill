# NAT VPS(NAT鸡) 接入流程

NAT 类 VPS(如 OVH NAT 鸡):**没有独立公网 IP**,所有访问经宿主机公网 IP + 映射端口转发。SSH 也用映射端口(如 192.99.100.74:20420),机器内网 IP 形如 `10.10.0.16`。

## 与普通 VPS 的差异
- `HostName` 填宿主机公网 IP,`Port` 填映射端口(不是 22,不是常见 20266)
- 机器本身无公网 IP,`info` 显示的是内网 IP(如 10.10.0.16)——这是正常的,不代表没连上
- 加固闭环(lock + fail2ban)与普通 VPS 完全相同,照常走

## 接入命令(与普通 VPS 一致,仅端口为映射端口)
```bash
bash <skill>/scripts/ssh-vps.sh add <alias> <host_ip> --user root --port <mapped_port> --password '<pw>'
# expect 缺失时用 sshpass 回退下发公钥(SKILL.md 有完整流程)
bash <skill>/scripts/ssh-vps.sh info <alias>   # 验证可登
bash <skill>/scripts/ssh-vps.sh lock <alias>   # 关密码登录
```

## 关键约定
- **别名不能含空格**:脚本校验别名仅允许字母/数字/`.`/`_`/`-`。用户给了「ovh nat鸡」这种带空格的中文名时,用连字符别名(如 `ovh-nat`),中文名写进 vps.config 的注释行(现有配置的 `# ovh nat鸡 (NAT VPS, 映射端口 20421-20449)` 就是这种写法)。
- **「可用端口范围」的含义**:服务商(如 OVH NAT)会把一段映射端口(如 20421-20449)交给这台机器。这串端口通常用于在**妙妙屋(mmwx)面板**里把该 NAT 服务器添加为「NAT 专线」,并基于它创建 mieru 节点。接到这类信息时要主动跟用户确认是否要进妙妙屋开节点。
- 加固完成后的验收到:仅 `~/.ssh/vps` 公钥可登、fail2ban sshd jail 生效。

## 典型场景(OVH NAT 鸡)
2026-10-15 实录:`192.99.100.74:20420`, Debian 12, 1核/973M/118G, 内网 10.10.0.16, 别名 `ovh-nat`, 映射端口 20421-20449。access 后 info 正常、lock 修复了 50-cloud-init + 99-nkx 两处覆盖、fail2ban 按 Debian 12 systemd backend 装好。
