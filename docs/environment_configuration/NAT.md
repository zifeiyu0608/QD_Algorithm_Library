# 自动启用NAT

使用 systemd 让 NAT 规则开机自启。

这里默认 net0 为有线网卡，internet0 为无线网卡，都要替换成真实的，使用`ip a`查看网卡名称
### 1. 先写脚本 `/usr/local/bin/nat-route.sh`

```bash
#!/bin/bash

# 开启内核IP转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 清空旧nat/forward规则（可选，避免重复添加多条规则，不需要就删掉这两行）
iptables -t nat -F POSTROUTING
iptables -F FORWARD

# 你的三条规则
iptables -t nat -A POSTROUTING -o internet0 -j MASQUERADE
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i net0 -o internet0 -j ACCEPT
```

赋予执行权限：

```bash
chmod +x /usr/local/bin/nat-route.sh
```

### 2. 创建 systemd 单元文件 `/etc/systemd/system/nat-route.service`

```ini
[Unit]
Description=Net NAT forward iptables rule
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/nat-route.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

### 3. 启用并启动

```bash
systemctl daemon-reload
systemctl enable nat-route.service
systemctl start nat-route.service
```

查看状态：

```bash
systemctl status nat-route.service
```
