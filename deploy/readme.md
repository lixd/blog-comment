# Deploy
如何在新服务器上部署？

**1）在新服务器上安装 caddy**

> [下载](https://github.com/caddyserver/caddy/releases) 解压并移动到 /usr/local/bin 目录下

```bash
cd ~
wget https://github.com/caddyserver/caddy/releases/download/v2.5.2/caddy_2.5.2_linux_amd64.tar.gz
mkdir caddy
tar -zxvf caddy_2.5.2_linux_amd64.tar.gz -C caddy
mv caddy/caddy /usr/local/bin/
```



测试能否正常使用
```bash
[root@iZ2ze7t0i43yaaiwlly7l1Z ~]# caddy version
v2.5.2 h1:eCJdLyEyAGzuQTa5Mh3gETnYWDClo1LjtQm2q9RNZrs=
```



**2）写入 Caddyfile**

将当前目录下的 Caddyfile 复制到服务器的 /etc/caddy 目录

> 如果域名有更改需要更新 Caddyfile

同时在该目录下创建 caddy.env 文件

```bash
touch  /etc/caddy/caddy.env
```



**3）创建对应的静态文件目录**

```bash
mkdir -p /var/www/html/public
mkdir -p /var/www/logs/
```



**4）写入 systemd 配置文件并启动服务**
将当前目录下的 caddyd.service 复制到服务器的 `/usr/lib/systemd/system` 目录下,并启动对应服务

```bash
 systemctl daemon-reload
 systemctl enable caddyd --now
```

检测运行情况

```bash
systemctl status caddyd
```





**5）检测服务器是否安装 rsync**

> centos 系列自带 rsync，如果是其他发行版可能需要手动安装

```bash
[root@iz2ze0ephck4d0aztho5r5z blog]# rsync --version
rsync  version 3.1.2  protocol version 31
```



**6）更新 github repo 中的 secret**

和服务器相关的都需要更新：

*  DEPLOY_HOST
* DEPLOY_PORT
* DEPLOY_USER
* DEPLOY_KEY



**7）commit 触发 github action**

检测是否正常运行。