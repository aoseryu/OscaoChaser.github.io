---
layout: post
title: "实验：在CentOS6.9上编译安装LAMP(php模块方式)"
description: "编译安装LAMP，并在其基础上实现WordPress以及加速."
categories: [centos6,service]
tags: [linux,lamp,wordpress]
redirect_from:
  - /2017/10/10/
---
## 实验目的
在centos6.9上编译安装httpd2.4，mariadb5.5，php5.6；并要求基于此基础上安装WordPress，搭建博客平台并编译安装xcache3.2实现缓存加速，并检测结果。
>注：缓存加速xcache支持旧版本的PHP，新版本不适用。

## 第一步：环境准备
<font size=4>
apr-1.6.2.tar.gz
apr-util-1.6.0.tar.gz      
httpd-2.4.27.tar.bz2
mariadb-5.5.57-linux-x86_64.tar.gz            
php-5.6.31.tar.xz 
wordpress-4.8-zh_CN.tar.gz           
xcache-3.2.0.tar.bz2
一台模拟服务器的centos6主机

</font>
## 第二步：源码编译httpd2.4
### （1）安装编译环境包组和包
<pre><font size=4>
[root@centos6 ~]$yum groupinstall "development tools"
[root@centos6 ~]$yum install openssl-devel pcre-devel expat-devel
</font>
</pre>

### （2）解压源码包
<pre><font size=4>
[root@centos6 app]$tar xvf apr-1.6.2.tar.gz 
[root@centos6 app]$tar xvf apr-util-1.6.0.tar.gz
[root@centos6 app]$tar xvf httpd-2.4.27.tar.bz2
</font>
</pre>
>在centos6上源码编译安装httpd2.4是需要依赖apr接口的，因此顺便将apr也进行源码编译安装

### （3）编译安装
<pre><font size=4>
[root@centos6 app]$cp -r apr-1.6.2 httpd-2.4.27/srclib/apr
[root@centos6 app]$cp -r apr-util-1.6.0 httpd-2.4.27/srclib/apr-util
[root@centos6 app]$cd httpd-2.4.27/
[root@centos6 httpd-2.4.27]$./configure \
\>--prefix=/app/httpd24 \  
\>--enable-so  \
\>--enable-ssl \ 
\>--enable-rewrite \ 
\>--with-zlib \
\>--with-pcre  \
\>--with-included-apr \  
\>--enable-modules=most \  
\>--enable-mpms-shared=all \ 
\>--with-mpm=prefork  
[root@centos6 httpd-2.4.27]$make -j 4 && make install
</font>
</pre>

>可以将apr和apr-util同时与httpd一起编译安装，需要将他们两个的解压目录复制或者移动到目录/app/httpd-2.4.27/srclib/下，并分别命名。

### （4）添加httpd的PATH路径
<pre><font size=4>
[root@centos6 ~]$vim /etc/profile.d/lamp.sh
PATH=/app/httpd24/bin/:$PATH
[root@centos6 ~]$. /etc/profile.d/lamp.sh
[root@centos6 ~]$echo $PATH
</font>
</pre>

### （5）准备启动脚本
<pre><font size=4>
[root@centos6 ~]$cp /etc/init.d/httpd  /etc/init.d/httpd24#这里拷贝centos6自带的httpd启动脚本，我们直接再修改一下即可
[root@centos6 ~]$vim /etc/init.d/httpd24
apachectl=/app/httpd24/bin/apachectl #指定编译安装的apachectl位置
httpd=${HTTPD-/app/httpd24/bin/httpd} #指定编译安装的httpd脚本位置
prog=httpd
pidfile=${PIDFILE-/app/httpd24/logs/httpd.pid} #指定编译安装的pidfile位置
lockfile=${LOCKFILE-/var/lock/subsys/httpd24} #指定编译安装的lockfile位置
</font>
</pre>

### （6）开机启动并启动测试
<pre><font size=4>
[root@centos6 ~]$chkconfig --add httpd24
[root@centos6 ~]$chkconfig --list httpd24
[root@centos6 ~]$chkconfig httpd24 on
[root@centos6 ~]$service httpd24 start
[root@centos6 ~]$ss -ntlp
</font>
</pre>

## 第二步：安装MariaDB二进制包
### （1）解压到指定目录
`tar xvf mariadb-5.5.57-linux-x86_64.tar.gz  -C /usr/local/`
`#因为安装的是已经编译好的二进制包，在编译的时候就已经指定了安装目录，所以必须要将包解压到/usr/local/这个目录`
`cd /usr/local/`
`ln -s mariadb-5.5.57-linux-x86_64/ mysql #创建软链接，方便安装`

### （2）为服务准备用户
`useradd -r -m -d /app/mysqldb -s /sbin/nologin mysql` 
`cd mysql/`
`scripts/mysql_install_db --datadir=/app/mysqldb --user=mysql`
`#必须要进入目录后，执行初始换脚本，指定数据库数据存放目录，指定数据库服务的用户。`

### （3）准备服务的配置文件
`mkdir /etc/mysql`
`cp support-files/my-large.cnf   /etc/mysql/my.cnf`
`#创建配置文件存放目录，并将配置文件模板拷贝并修改`
`vim /etc/mysql/my.cnf`
`[mysqld]`
`datadir = /app/mysqldb #指定数据库数据存放目录；该项是必须的`
`innodb_file_per_table = ON  #设置每个表作为文件单独存放；该项可选`
`skip_name_resolve = ON #设置跳过名字解析，加快登录速度；该项可选`

### （5）准备服务的启动脚本
`cp support-files/mysql.server /etc/init.d/mysqld`
`#将服务脚本拷贝到规定目录下并改名为mysqld，这样便可以利用service命令和chkconfig命令控制。`

### （6）设置服务的PATH路径
`vim /etc/profile.d/lamp.sh` 
`PATH=/app/httpd24/bin/:/usr/local/mysql/bin/:$PATH`
`. /etc/profile.d/lamp.sh`

### （7）将服务设置为开机启动并启动查看端口
`chkconfig --add mysqld`
`chkconfig --list` 
`service mysqld start`
`ss -ntlp`

### （8）执行安全初始化脚本
`mysql_secure_installation`
`#根据提示一步步操作，重要的是设置root密码，以及删除临时用户`

### （9）为WordPress创建数据库用户
`mysql -uroot -p123456`
`CREATE datebase wpdb;`
`GRANT all ON wpdb.* TO wpuser@'%' IDENTIFIED BY '123456';`
`#%表示所有的主机都可以用这个账户链接，不太安全，如果是为独立的数据库服务器，那么最好设定好IP地址`

## 第三步：源码编译PHP
### （1）安装编译PHP时的依赖包
`yum -y install libxml2-devel bzip2-devel libmcrypt-devel`
`#注意配置epel源`

### （2）解压并编译安装
`tar xvf php-5.6.31.tar.xz `
`cd php-5.6.31`		
`./configure --prefix=/app/php 
--with-apxs2=/app/httpd24/bin/apxs #该项指定php作为httpd24的一个模块工作
--with-mysql=/usr/local/mysql  #指定要连接的mysql位置
--with-mysqli=/usr/local/mysql/bin/mysql_config  #指定链接数据库接口文件的位置 
--with-openssl 
--enable-mbstring 
--with-png-dir 
--with-jpeg-dir 
--with-freetype-dir 
--with-zlib 
--with-libxml-dir=/usr 
--enable-xml 
--enable-sockets 
--with-mcrypt 
--with-config-file-path=/etc 
--with-config-file-scan-dir=/etc/php.d 
-with-bz2`
`make -j 4 && make install`

### （3）配置服务的PATH路径
`vim /etc/profile.d/lamp.sh` 
`PATH=/app/php/bin:/app/httpd24/bin/:/usr/local/mysql/bin/:$PATH`
`.  /etc/profile.d/lamp.sh`

### （4）设置php和httpd的配置文件
`cp /app/php-5.6.31php.ini-production /etc/php.ini`
`#我们可以直接拷贝模板改名，内容不需要修改`
`vim /app/httpd24/conf/httpd.conf`
`#在文件尾部加两行`
`AddType application/x-httpd-php .php`
`AddType application/x-httpd-php-source .phps`
`#修改下面行,指明访问的主站点文件`
`<IfModule dir_module>`
`DirectoryIndex index.php index.html`
`</IfModule>`

### （5）设置完成重启服务
`service httpd24 restart`

## 第四步：测试php是否正常
### （1）编写一个简单的PHP脚本来检测能否在httpd上运行，以及能否链接数据库
`vim /app/httpd24/htdocs/index.php`
`<html><body><h1>It works!</h1></body></html>`
`<?php`
`$mysqli=new mysqli("localhost","root","123456");`
`if(mysqli_connect_errno()){`
`echo "连接数据库失败!";`
`$mysqli=null;`
`exit;`
`}`
`echo "连接数据库成功!";`
`$mysqli->close();`
`phpinfo();`
`?>`

### （2）在浏览器访问测试即可
在浏览器地址栏输入我们的httpd服务器的IP地址即可

## 第五步：安装配置WordPress
### （1）解压到指定目录
`tar xvf wordpress-4.8.1-zh_CN.tar.gz  -C /app/httpd24/htdocs`

### （2）将目录改名为blog
`cd /app/httpd24/htdocs`
`mv wordpress/ blog`

### （3）修改配置文件
`cd /app/httpd24/htdocs/blog/`
`cp wp-config-sample.php  wp-config.php`
`vim wp-config.php`
`/** MySQL数据库的名称*/`
`define('DB_NAME', 'wpdb');`
`/** MySQL数据库用户名 */`
`define('DB_USER', 'wpuser');`
`/** MySQL数据库密码 */`
`define('DB_PASSWORD', '123456');`
`/** MySQL主机 */`
`define('DB_HOST', 'localhost');`

## 第六步：登录并测试WordPress
### （1）登录
在浏览器地址栏输入http://websrv/blog

### （2）测试性能
`ab -c 10 -n 100 http://websrv/blog/`
`#主要观察Requests per second这项的数值，越大性能越高`

## 第七步：编译安装xcache实现PHP加速
### （1）解压并编译安装
`tar xvf xcache-3.2.0.tar.bz2 `
`cd xcache-3.2.0`
`phpize `
`./configure  --enable-xcache --with-php-config=/app/php/bin/php-config` 
`make -j 4 && make install`

### （2）准备xcache的配置文件
`mkdir /etc/php.d/`
`cp xcache.ini  /etc/php.d/`
`vim /etc/php.d/xcache.ini `
`extension = /app/php/lib/php/extensions/no-debug-non-zts-20131226/xcache.so `
`#xcache是作为php的一个模块发挥 作用的，因此要创建一个php的子配置文件，在其中指明要加载的模块位置`

### （3）重启httpd服务
`service httpd24 restart`

## 第八步：测试加速效果
### （1）登录页面查看xcache功能是否开启
### （2）检测加速效果
`ab -c 10 -n 100 http://websrv/blog/`
`#和之前微加速前比较二者之间的Requests per second这项的数值，一般成功加速数值会变大，性能会提升`


