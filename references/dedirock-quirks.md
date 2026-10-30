# Dedirock 环境注意事项

## 自定义 nginx 路径

Dedirock 的 nginx **不是**系统路径的标准安装：

| 组件 | 实际路径 | 系统路径（空/无效） |
|------|---------|-------------------|
| 二进制 | `/usr/local/nginx/sbin/nginx` | `/usr/sbin/nginx` ❌ |
| 配置文件 | `/usr/local/nginx/nginx.conf` | `/etc/nginx/nginx.conf` ⚠️ |
| 站点配置 | `/usr/local/nginx/servers/*.conf` | `/etc/nginx/sites-enabled/` ❌ |

systemd 服务名: `nginx.service`（正常可用 `systemctl restart nginx`）

### 添加站点配置

1. 在 `/usr/local/nginx/servers/` 下创建 `.conf` 文件
2. 确保 `/usr/local/nginx/nginx.conf` 的 `http {}` 块内有 `include servers/*.conf;`
3. **注意**: `sed '/^}$/i\...'` 会匹配**第一个** `}`（events 块），不是 http 块。应手动确认插入位置
4. 测试: `/usr/local/nginx/sbin/nginx -t`
5. 重载: `systemctl reload nginx`（或 restart）

### certbot

- certbot nginx 插件**不可用**（插件查找系统 nginx 路径，与 `/usr/local/nginx` 不兼容）
- 签 SSL 需用 `certbot certonly --standalone` 模式，**先 `systemctl stop nginx`**，签完再启动

## 端口冲突

- mmw-agent 默认监听 443（IPv4+IPv6），如需 nginx 绑 443 必须先停 mmw-agent：
  ```bash
  systemctl stop mmw-agent
  ```

## 已部署服务（2026-08-06 更新）

| 服务 | 端口 | 管理 |
|------|------|------|
| 妙妙屋X (mmwx) | 12889 | `systemctl start/stop mmwx` |
| nginx | 80, 443 | `systemctl start/stop nginx` |
| x-ui | 2096 | `systemctl start/stop x-ui` |
| xray | 内部 | x-ui 管理 |
| frps | 8055, 8056 | systemd |
| komari-agent | — | systemd |
| fail2ban | — | systemd |
| mmw-agent | 已停 ❌ | `systemctl stop mmw-agent` |