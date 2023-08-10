```bash
# 查看内核tcp MTU探测是否开启
cat /proc/sys/net/ipv4/tcp_mtu_probing

# 开启内核tcp MTU探测
vim /etc/sysctl.d/99-sysctl.conf 文件最后添加
net.ipv4.tcp_mtu_probing=1
# 重新加载配置
sysctl --system
```