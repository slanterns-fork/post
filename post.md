# MAINLINE
#### BUILD NGINX
```bash
# 安装必要工具
apt install build-essential libpcre3 libpcre3-dev zlib1g-dev unzip git autoconf libtool automake

# 创建编译目录
mkdir build_ngx && cd build_ngx

# brotli
git clone https://github.com/eustas/ngx_brotli.git #google/ngx_brotli 年久失修
cd ngx_brotli
git submodule update --init
cd ..

# patches
git clone https://github.com/kn007/patch.git #cloudflare/sslconfig 年久失修

# openssl
wget -O openssl.tar.gz -c https://www.openssl.org/source/openssl-1.1.1-pre6.tar.gz
tar zxf openssl.tar.gz
mv openssl-1.1.1-pre6/ openssl

# nginx
wget -c https://nginx.org/download/nginx-1.14.0.tar.gz
tar zxf nginx-1.14.0.tar.gz

# patch
cd nginx-1.14.0
patch -p1 < ../patch/nginx_auto_using_PRIORITIZE_CHACHA.patch
patch -p1 < ../patch/nginx.patch

# make
./configure --add-module=../ngx_brotli --with-openssl=../openssl --with-http_v2_module --with-http_ssl_module --with-http_gzip_static_module --with-http_v2_hpack_enc --with-file-aio
make
make install
```

&ensp;

#### CONFIGURE NGINX
systemd 相关：

提供一种比 service start xxx 更优雅的方式。

将 `nginx.service` 丢到 `/etc/systemd/system`，内容（基于官方源里的`nginx-common_1.14.0-0ubuntu1_all.deb`解包得到并乱改）：
```
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/local/nginx/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/usr/local/nginx/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /usr/local/nginx/logs/nginx.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
```
然后  `systemctl enable nginx`

之后可用 `systemctl start|reload|stop ngnix` . Done.
![](.\pic\nginx_it_works!.png)


&ensp;

---
# MISC
#### BBR
开启BBR以优化拥塞控制算法。在学校电信出口环境下连接 vultr 东京机房 SFTP 大约 1M/s.

p.s.Kernel 4.9 以后并不需要手动安装内核。

编辑 /etc/sysctl.conf
```bash
# max open files
fs.file-max = 51200
# max read buffer
net.core.rmem_max = 67108864
# max write buffer
net.core.wmem_max = 67108864
# default read buffer
net.core.rmem_default = 65536
# default write buffer
net.core.wmem_default = 65536
# max processor input queue
net.core.netdev_max_backlog = 4096
# max backlog
net.core.somaxconn = 4096

# resist SYN flood attacks
net.ipv4.tcp_syncookies = 1
# reuse timewait sockets when safe
net.ipv4.tcp_tw_reuse = 1
# turn off fast timewait sockets recycling
net.ipv4.tcp_tw_recycle = 0
# short FIN timeout
net.ipv4.tcp_fin_timeout = 30
# short keepalive time
net.ipv4.tcp_keepalive_time = 1200
# outbound port range
net.ipv4.ip_local_port_range = 10000 65000
# max SYN backlog
net.ipv4.tcp_max_syn_backlog = 4096
# max timewait sockets held by system simultaneously
net.ipv4.tcp_max_tw_buckets = 5000
# turn on TCP Fast Open on both client and server side
net.ipv4.tcp_fastopen = 3
# TCP receive buffer
net.ipv4.tcp_rmem = 4096 87380 67108864
# TCP write buffer
net.ipv4.tcp_wmem = 4096 65536 67108864
# turn on path MTU discovery
net.ipv4.tcp_mtu_probing = 1

# for high-latency network
net.ipv4.tcp_congestion_control = bbr

# for low-latency network, use cubic instead
# net.ipv4.tcp_congestion_control = cubic

net.core.default_qdisc = fq
```
执行
```
sysctl -p
sysctl net.ipv4.tcp_available_congestion_control
```
如果结果是这样
```
"root@debian-512mb-sgp1-01:~# sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = bbr cubic reno    p.s.顺序大概不重要？
```
执行 `lsmod | grep bbr`，以检测 BBR 是否开启。
```
tcp_bbr                20480  7
```

refs：
https://github.com/shadowsocks/shadowsocks/wiki/Optimizing-Shadowsocks

https://zhuanlan.zhihu.com/p/24418274

https://www.mf8.biz/linux-kernel-with-tcp-bbr/

&ensp;

#### 公钥登陆
提高安全性，简化登陆。

`ssh-keygen -t ed25519(|rsa|dsa|ecdsa)` , 一路回车。

修改 `/etc/ssh/sshd_config`：
```
StrictModes yes
AuthorizedKeysFile	.ssh/id_ed25519.pub
```
`systemctl restart sshd`
把生成的 `/root/.ssh/id_ed25519` 拖回本地。修改 Winscp 登录选项：
![](.\pic\winscp_key_login.png)
Winscp 会帮助你将其转换为 putty 格式的密钥。填入用户名，密码留空。

尝试登陆成功后（~~如果你不想失去你的服务器的话~~），修改
```
PasswordAuthentication no
```
重启 sshd 服务。尝试用密码连接：
![](.\pic\passwd_disabled.png)
Done.
