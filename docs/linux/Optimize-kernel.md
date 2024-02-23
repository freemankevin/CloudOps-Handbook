

### 1 常用配置

```shell
fs.file-max= 6815744
fs.aio-max-nr= 1048576
vm.drop_caches = 3
vm.max_map_count = 262144
kernel.sysrq = 0
kernel.dmesg_restrict = 1
kernel.shmall= 2097152
kernel.shmmax= 4294967295
kernel.shmmni= 4096
kernel.sem= 250 32000 100 128
net.ipv4.ip_forward = 1
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_local_port_range= 9000 65500
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.core.rmem_default= 262144
net.core.rmem_max= 4194304
net.core.wmem_default= 262144
net.core.wmem_max= 1048576
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1
```

这些内核配置项的含义如下：

- `fs.file-max`: 设置系统级别的文件描述符限制，表示系统支持的最大文件描述符数量。
- `fs.aio-max-nr`: 设置系统级别的异步 I/O（AIO）操作的最大数量。
- `vm.drop_caches`: 用于控制内核是否丢弃缓存的选项。该值为3表示丢弃页缓存和目录项缓存，但不丢弃索引节点缓存。
- `vm.max_map_count`: 设置进程可拥有的最大内存映射区域数量。
- `kernel.sysrq`: 用于控制系统的 SysRq 功能，0表示禁用。
- `kernel.dmesg_restrict`: 控制是否限制非特权用户对内核日志的访问权限，1表示限制。
- `kernel.shmall`: 指定系统范围内共享内存段的最大页框数。
- `kernel.shmmax`: 指定系统范围内单个共享内存段的最大大小。
- `kernel.shmmni`: 指定系统支持的最大共享内存段数量。
- `kernel.sem`: 设置系统中的信号量参数。
- `net.ipv4.ip_forward`: 启用 IPv4 数据包转发功能。
- `net.ipv4.conf.all.send_redirects` 和 `net.ipv4.conf.default.send_redirects`: 控制是否发送 ICMP 重定向报文。
- `net.ipv4.conf.all.accept_source_route` 和 `net.ipv4.conf.default.accept_source_route`: 控制是否接受源路由选项。
- `net.ipv4.conf.all.accept_redirects` 和 `net.ipv4.conf.default.accept_redirects`: 控制是否接受 ICMP 重定向报文。
- `net.ipv4.conf.all.secure_redirects` 和 `net.ipv4.conf.default.secure_redirects`: 控制是否只接受安全的 ICMP 重定向报文。
- `net.ipv4.icmp_echo_ignore_broadcasts`: 控制是否忽略对广播地址的 ICMP 回显请求。
- `net.ipv4.icmp_ignore_bogus_error_responses`: 控制是否忽略无效的 ICMP 错误响应。
- `net.ipv4.conf.all.rp_filter` 和 `net.ipv4.conf.default.rp_filter`: 控制是否启用反向路径过滤。
- `net.ipv4.tcp_syncookies`: 启用 TCP Syn Cookie 机制来抵御 SYN Flood 攻击。
- `net.ipv4.ip_local_port_range`: 设置本地端口的起始和结束范围。
- `net.ipv6.conf.all.accept_redirects` 和 `net.ipv6.conf.default.accept_redirects`: 控制是否接受 IPv6 的 ICMP 重定向报文。
- `net.core.rmem_default` 和 `net.core.rmem_max`: 控制接收缓冲区的默认大小和最大大小。
- `net.core.wmem_default` 和 `net.core.wmem_max`: 控制发送缓冲区的默认大小和最大大小。
- `net.bridge.bridge-nf-call-ip6tables` 和 `net.bridge.bridge-nf-call-arptables`: 控制是否在网络桥接时调

用 IPv6 和 ARP 的 iptables 规则。

这些配置项用于调整系统的网络、内存管理和安全策略等方面的行为。具体的配置值可以根据实际需求进行调整，但在修改这些配置项之前，建议先了解其含义和影响，并在进行修改时谨慎操作。
