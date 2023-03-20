# Nextcloud-Deployment-Playbook-Non-docker-way

特别感谢  [Carsten Rieger IT Services](https://www.c-rieger.de/cloud-und-konferenzsysteme/) 。本文是基于 [Carsten Rieger](https://www.c-rieger.de/author/carsten/) 所分享得 [Nextcloud Installationsanleitung](https://www.c-rieger.de/nextcloud-installationsanleitung/)

这份 Nextcloud 25 安装指南描述了如何在 Ubuntu 20.04 LTS focal、22.04 LTS jammy（x86-64）服务器上安装、配置、硬化、监控和扩展 Nextcloud 25。

安装基于 nginx 1.23.x、Let’s Encrypt TLS 1.3、MariaDB 10.x、PHP 8.x（php-fpm）、Redis、Fail2ban、ufw 和 Netdata 组件，并最终获得 Nextcloud 和 Qualys SSL Labs 的 A+ 安全评级。

# 安装前准备工作

安装 `ubuntu-keyring` 软件包，`ubuntu-keyring` 软件包包含 Ubuntu 发布团队的 OpenPGP 公钥，这些密钥可用于验证 Ubuntu 发布的软件包的完整性和真实性。在系统上安装了这些密钥之后，APT 软件包管理器将使用这些密钥验证 Ubuntu 发布的软件包。

``` bash
apt install -y ubuntu-keyring
```

使用 `curl` 命令下载 `nginx` 官方的 GPG 签名密钥，然后通过 `gpg` 命令将密钥解密并写入 `/usr/share/keyrings/nginx-archive-keyring.gpg` 文件中，用于验证 `nginx` 软件包的真实性。最后使用 `tee` 命令将解密后的密钥写入 `/usr/share/keyrings/nginx-archive-keyring.gpg` 文件中。

使用 `echo` 命令将 `nginx` 软件包源的地址添加到 `/etc/apt/sources.list.d/nginx.list` 文件中。其中 `[signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg]` 指定了软件包的签名密钥，确保下载的软件包是官方签名的。

使用 `wget` 命令下载 `MariaDB` 的签名密钥，然后通过 `gpg` 命令将密钥解密并写入 `/usr/share/keyrings/mariadb-keyring.gpg` 文件中，用于验证 `MariaDB` 软件包的真实性。最后使用 `tee` 命令将解密后的密钥写入 `/usr/share/keyrings/mariadb-keyring.gpg` 文件中。

使用 `echo` 命令将 `MariaDB` 软件包源的地址添加到 `/etc/apt/sources.list.d/mariadb.list` 文件中。其中 `[signed-by=/usr/share/keyrings/mariadb-keyring.gpg]` 指定了软件包的签名密钥，确保下载的软件包是官方签名的。如需安装其他版本的 MariaDB 的话，只需要把 10.8 替换成你需要的版本即可。

``` bash
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
	| sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
	http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
	| sudo tee /etc/apt/sources.list.d/nginx.list

wget -O- https://mariadb.org/mariadb_release_signing_key.asc \
	| gpg --dearmor | sudo tee /usr/share/keyrings/mariadb-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/mariadb-keyring.gpg] \
	https://mirror.kumi.systems/mariadb/repo/10.8/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/mariadb.list

```

如果系统是 Ubuntu 20.04 那么请添加这个 PHP 源。Ubuntu 22.04，如果想要安装 PHP 7.4 ，也需要添加这个源。👇

``` bash
add-apt-repository -y ppa:ondrej/php
```

更新 apt 的软件包索引，顺便自签个证书，后面会用到。

``` bash
apt update && make-ssl-cert generate-default-snakeoil -y
```

为了确保没有以前安装的遗留物干扰服务器的运行，我们将它们删除（强迫症请原谅🤣）。

``` bash
apt remove nginx nginx-extras nginx-common nginx-full -y --allow-change-held-packages
```

继续强迫症一下，执行以下命令确保 nginx 或者 Apache 既没有激活也没有安装。

``` bash
systemctl stop apache2.service 
systemctl disable apache2.service
```

# 安装 nginx

强迫症结束，现在安装 nginx 。

``` bash
apt install -y nginx
```

设置成系统服务，保证开机自启动。

``` bash
systemctl enable nginx.service
```

备份默认配置文件，并 touch 一个新的 && 用 vi 编辑器带着个新配置文件。

``` bash
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
touch /etc/nginx/nginx.conf && vi /etc/nginx/nginx.conf
```

新配置文件写入以下内容即可。

``` bash
user www-data;
worker_processes auto;
pid /var/run/nginx.pid;
events {
  worker_connections 2048;
  multi_accept on; use epoll;
}
http {
  log_format criegerde escape=json
  '{'
  '"time_local":"$time_local",'
  '"remote_addr":"$remote_addr",'
  '"remote_user":"$remote_user",'
  '"request":"$request",'
  '"status": "$status",'
  '"body_bytes_sent":"$body_bytes_sent",'
  '"request_time":"$request_time",'
  '"http_referrer":"$http_referer",'
  '"http_user_agent":"$http_user_agent"'
  '}';
  server_names_hash_bucket_size 64;
  access_log /var/log/nginx/access.log criegerde;
  error_log /var/log/nginx/error.log warn;
  #set_real_ip_from 127.0.0.1;
  real_ip_header X-Forwarded-For;
  real_ip_recursive on;
  include /etc/nginx/mime.types;
  default_type application/octet-stream;
  sendfile on;
  send_timeout 3600;
  tcp_nopush on;
  tcp_nodelay on;
  open_file_cache max=500 inactive=10m;
  open_file_cache_errors on;
  keepalive_timeout 65;
  reset_timedout_connection on;
  server_tokens off;
  resolver 127.0.0.53 valid=30s;
  resolver_timeout 5s;
  include /etc/nginx/conf.d/*.conf;
}
```

重启 nginx 服务，一切正常的话，此时你就可以访问到默认站点啦。

``` bash
systemctl restart nginx.service
```

准备 SSL 证书所需的文件夹，以及 web 应用程序所需的日志文件夹，并且给他们设置正确的权限。

``` bash
mkdir -p /var/log/nextcloud /var/nc_data /var/www/letsencrypt/.well-known/acme-challenge /etc/letsencrypt/rsa-certs /etc/letsencrypt/ecc-certs
```

``` bash
chown -R www-data:www-data /var/nc_data /var/www /var/log/nextcloud
```

至此，Web 服务器（nginx）就安装完了。

# 安装 PHP

上一节，我们已经配置好了 PHP 所需得资源库，这里我们直接用命令安装即可。

``` bash
apt update && apt install -y php-common \ php8.1-{fpm,gd,curl,xml,zip,intl,mbstring,bz2,ldap,apcu,bcmath,gmp,imagick,igbinary,mysql,redis,smbclient,cli,common,opcache,readline} \ imagemagick --allow-change-held-packages
```

如果计划使用Samba和/或cifs共享或LDAP(s)连接，那么再执行以下命令。

``` bash
apt install -y ldap-utils nfs-common cifs-utils
```

-   `ldap-utils`: 包含了命令行工具和客户端库，用于在 Linux 上进行 Lightweight Directory Access Protocol (LDAP) 操作，如查询和修改 LDAP 数据库，进行身份验证等。
-   `nfs-common`: 包含了用于在 Linux 上访问 Network File System (NFS) 的客户端程序，用于将远程 NFS 共享挂载到本地系统上。
-   `cifs-utils`: 包含了用于在 Linux 上访问 SMB/CIFS 共享的客户端程序，用于将远程 SMB/CIFS 共享挂载到本地系统上。

设置好服务器的时区，以便能正确得输出日志。

``` bash
timedatectl set-timezone Asia/Shanghai
```

接下来准备优化 PHP 得配置，我们先备份必要的文件。

``` bash
cp /etc/php/8.1/fpm/pool.d/www.conf /etc/php/8.1/fpm/pool.d/www.conf.bak
cp /etc/php/8.1/fpm/php-fpm.conf /etc/php/8.1/fpm/php-fpm.conf.bak
cp /etc/php/8.1/cli/php.ini /etc/php/8.1/cli/php.ini.bak
cp /etc/php/8.1/fpm/php.ini /etc/php/8.1/fpm/php.ini.bak
cp /etc/php/8.1/mods-available/apcu.ini /etc/php/8.1/mods-available/apcu.ini.bak
cp /etc/ImageMagick-6/policy.xml /etc/ImageMagick-6/policy.xml.bak
systemctl restart php8.1-fpm.service
```

为了方便操作，也可以写一个 shell 脚本来执行这些重复得操作。

``` bash
#!/bin/bash

# 提供菜单让用户选择 PHP 版本
echo "请选择 PHP 版本:"
echo "1. PHP 7.4"
echo "2. PHP 8.0"
echo "3. PHP 8.1"

read -r -p "请输入数字 [1-3]: " choice

case $choice in
  1)
    PHP_VERSION=7.4
    ;;
  2)
    PHP_VERSION=8.0
    ;;
  3)
    PHP_VERSION=8.1
    ;;
  *)
    echo "输入有误"
    exit 1
    ;;
esac

# 备份文件
cp /etc/php/$PHP_VERSION/fpm/pool.d/www.conf /etc/php/$PHP_VERSION/fpm/pool.d/www.conf.bak
cp /etc/php/$PHP_VERSION/fpm/php-fpm.conf /etc/php/$PHP_VERSION/fpm/php-fpm.conf.bak
cp /etc/php/$PHP_VERSION/cli/php.ini /etc/php/$PHP_VERSION/cli/php.ini.bak
cp /etc/php/$PHP_VERSION/fpm/php.ini /etc/php/$PHP_VERSION/fpm/php.ini.bak
cp /etc/php/$PHP_VERSION/mods-available/apcu.ini /etc/php/$PHP_VERSION/mods-available/apcu.ini.bak
cp /etc/ImageMagick-6/policy.xml /etc/ImageMagick-6/policy.xml.bak

# 重启 PHP-FPM 服务
systemctl restart php$PHP_VERSION-fpm.service
```

为了适应不同得机器，在这里定义了一些变量，方便后续调参。

``` bash
# 获取可用内存大小（单位：MB）
AvailableRAM=$(awk '/MemAvailable/ {printf "%d", $2/1024}' /proc/meminfo)

# 获取 php-fpm8.1 进程的平均占用内存大小（单位：MB）
AverageFPM=$(ps --no-headers -o 'rss,cmd' -C php-fpm8.1 | awk '{ sum+=$1 } END { printf ("%d\n", sum/NR/1024,"M") }')

# 计算每个 php-fpm8.1 进程所需内存大小（单位：MB）
FPMS=$((AvailableRAM/AverageFPM))

# 计算 php-fpm8.1 进程的最大子进程数
PMaxSS=$((FPMS*2/3))

# 计算 php-fpm8.1 进程的最小子进程数
PMinSS=$((PMaxSS/2))

# 计算 php-fpm8.1 进程的起始子进程数
PStartS=$(((PMaxSS+PMinSS)/2))
```

接下来，我们改变一些 配置参数来进行 PHP 部分得优化。

``` bash
sed -i "s/;env\[HOSTNAME\] = /env[HOSTNAME] = /" /etc/php/8.1/fpm/pool.d/www.conf
sed -i "s/;env\[TMP\] = /env[TMP] = /" /etc/php/8.1/fpm/pool.d/www.conf
sed -i "s/;env\[TMPDIR\] = /env[TMPDIR] = /" /etc/php/8.1/fpm/pool.d/www.conf
sed -i "s/;env\[TEMP\] = /env[TEMP] = /" /etc/php/8.1/fpm/pool.d/www.conf
sed -i "s/;env\[PATH\] = /env[PATH] = /" /etc/php/8.1/fpm/pool.d/www.conf

# 可有可无得配置项:
sed -i 's/pm = dynamic/pm = static/' /etc/php/8.1/fpm/pool.d/www.conf
#

sed -i 's/pm.max_children =.*/pm.max_children = '$FPMS'/' /etc/php/8.1/fpm/pool.d/www.conf
sed -i 's/pm.start_servers =.*/pm.start_servers = '$PStartS'/' /etc/php/8.1/fpm/pool.d/www.conf
sed -i 's/pm.min_spare_servers =.*/pm.min_spare_servers = '$PMinSS'/' /etc/php/8.1/fpm/pool.d/www.conf
sed -i 's/pm.max_spare_servers =.*/pm.max_spare_servers = '$PMaxSS'/' /etc/php/8.1/fpm/pool.d/www.conf
sed -i "s/;pm.max_requests =.*/pm.max_requests = 1000/" /etc/php/8.1/fpm/pool.d/www.conf
sed -i "s/allow_url_fopen =.*/allow_url_fopen = 1/" /etc/php/8.1/fpm/php.ini

sed -i "s/output_buffering =.*/output_buffering = 'Off'/" /etc/php/8.1/cli/php.ini
sed -i "s/max_execution_time =.*/max_execution_time = 3600/" /etc/php/8.1/cli/php.ini
sed -i "s/max_input_time =.*/max_input_time = 3600/" /etc/php/8.1/cli/php.ini
sed -i "s/post_max_size =.*/post_max_size = 10240M/" /etc/php/8.1/cli/php.ini
sed -i "s/upload_max_filesize =.*/upload_max_filesize = 10240M/" /etc/php/8.1/cli/php.ini
sed -i "s/;date.timezone.*/date.timezone = Asia\/\Shanghai/" /etc/php/8.1/cli/php.ini
sed -i "s/;cgi.fix_pathinfo.*/cgi.fix_pathinfo=0/" /etc/php/8.1/cli/php.ini

sed -i "s/memory_limit = 128M/memory_limit = 1G/" /etc/php/8.1/fpm/php.ini
sed -i "s/output_buffering =.*/output_buffering = 'Off'/" /etc/php/8.1/fpm/php.ini
sed -i "s/max_execution_time =.*/max_execution_time = 3600/" /etc/php/8.1/fpm/php.ini
sed -i "s/max_input_time =.*/max_input_time = 3600/" /etc/php/8.1/fpm/php.ini
sed -i "s/post_max_size =.*/post_max_size = 10G/" /etc/php/8.1/fpm/php.ini
sed -i "s/upload_max_filesize =.*/upload_max_filesize = 10G/" /etc/php/8.1/fpm/php.ini
sed -i "s/;date.timezone.*/date.timezone = Asia\/\Shanghai/" /etc/php/8.1/fpm/php.ini
sed -i "s/;cgi.fix_pathinfo.*/cgi.fix_pathinfo=0/" /etc/php/8.1/fpm/php.ini
sed -i "s/;session.cookie_secure.*/session.cookie_secure = True/" /etc/php/8.1/fpm/php.ini
sed -i "s/;opcache.enable=.*/opcache.enable=1/" /etc/php/8.1/fpm/php.ini
sed -i "s/;opcache.validate_timestamps=.*/opcache.validate_timestamps=1/" /etc/php/8.1/fpm/php.ini
sed -i "s/;opcache.enable_cli=.*/opcache.enable_cli=1/" /etc/php/8.1/fpm/php.ini
sed -i "s/;opcache.memory_consumption=.*/opcache.memory_consumption=256/" /etc/php/8.1/fpm/php.ini
sed -i "s/;opcache.interned_strings_buffer=.*/opcache.interned_strings_buffer=64/" /etc/php/8.1/fpm/php.ini
sed -i "s/;opcache.max_accelerated_files=.*/opcache.max_accelerated_files=100000/" /etc/php/8.1/fpm/php.ini
sed -i "s/;opcache.revalidate_freq=.*/opcache.revalidate_freq=0/" /etc/php/8.1/fpm/php.ini
sed -i "s/;opcache.save_comments=.*/opcache.save_comments=1/" /etc/php/8.1/fpm/php.ini

sed -i "s|;emergency_restart_threshold.*|emergency_restart_threshold = 10|g" /etc/php/8.1/fpm/php-fpm.conf
sed -i "s|;emergency_restart_interval.*|emergency_restart_interval = 1m|g" /etc/php/8.1/fpm/php-fpm.conf
sed -i "s|;process_control_timeout.*|process_control_timeout = 10|g" /etc/php/8.1/fpm/php-fpm.conf

sed -i '$aapc.enable_cli=1' /etc/php/8.1/mods-available/apcu.ini

sed -i "s/rights=\"none\" pattern=\"PS\"/rights=\"read|write\" pattern=\"PS\"/" /etc/ImageMagick-6/policy.xml
sed -i "s/rights=\"none\" pattern=\"EPS\"/rights=\"read|write\" pattern=\"EPS\"/" /etc/ImageMagick-6/policy.xml
sed -i "s/rights=\"none\" pattern=\"PDF\"/rights=\"read|write\" pattern=\"PDF\"/" /etc/ImageMagick-6/policy.xml
sed -i "s/rights=\"none\" pattern=\"XPS\"/rights=\"read|write\" pattern=\"XPS\"/" /etc/ImageMagick-6/policy.xml
```

这些是针对 MariaDB 得优化项：

``` bash
sed -i '$a[mysql]' /etc/php/8.1/mods-available/mysqli.ini
sed -i '$amysql.allow_local_infile=On' /etc/php/8.1/mods-available/mysqli.ini
sed -i '$amysql.allow_persistent=On' /etc/php/8.1/mods-available/mysqli.ini
sed -i '$amysql.cache_size=2000' /etc/php/8.1/mods-available/mysqli.ini
sed -i '$amysql.max_persistent=-1' /etc/php/8.1/mods-available/mysqli.ini
sed -i '$amysql.max_links=-1' /etc/php/8.1/mods-available/mysqli.ini
sed -i '$amysql.default_port=3306' /etc/php/8.1/mods-available/mysqli.ini
sed -i '$amysql.connect_timeout=60' /etc/php/8.1/mods-available/mysqli.ini
sed -i '$amysql.trace_mode=Off' /etc/php/8.1/mods-available/mysqli.ini
```
