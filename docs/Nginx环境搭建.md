# Nginx环境搭建

## 1、系统环境

| 软件  | 版本   |
| ----- | ------ |
| Nginx | 1.10.3 |
| Cenos | 7      |



```
[root@localhost ~]#  cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
[root@localhost ~]# uname -a
Linux localhost.localdomain 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```
## 2、安装步骤
###  2.1 安装基础依赖包

- 安装 pcre
- 安装 openssl-devel

```
# pcre 安装
# 安装 pcre库是为了使 nginx 支持具备 URI 重写功能的 rewrite 模块，如果不安装 pcre 库，则 nginx 无法使用 rewrite 模块功能
[root@localhost zmf]# yum -y install pcre pcre-devel
[root@localhost zmf]# rpm -qa pcre pcre-devel
pcre-devel-8.32-17.el7.x86_64
pcre-8.32-17.el7.x86_64

# openssl-devel 安装
# nginx 在使用HTTPS服务的时候要用到此模块，如果不安装 openssl 相关包，安装 nginx 的过程中会报错。openssl 系统默认已经安装，只需要安装 openssl-devel 即可
[root@localhost zmf]# yum -y install openssl-devel
[root@localhost zmf]# rpm -qa openssl-devel openssl
openssl-devel-1.0.2k-19.el7.x86_64
openssl-1.0.2k-19.el7.x86_64
```

### 2.2 安装 nginx
```
# 创建软件包存放目录
[root@localhost zmf]# mkdir -p /data/tools
[root@localhost zmf]# cd /data/tools/

# 下载 nginx 的稳定版本 1.10.3
[root@localhost zmf]# wget http://nginx.org/download/nginx-1.10.3.tar.gz
# 创建 nginx 用户 可省略
[root@localhost zmf]# useradd nginx -s /sbin/nologin -M

[root@localhost zmf]# tar -zxf nginx-1.10.3.tar.gz 
[root@localhost zmf]# cd nginx-1.10.3
[root@localhost zmf]# ./configure --prefix=/usr/lcoal/zmf/nginx-1.10.3 --with-http_ssl_module
[root@localhost zmf]# make
[root@localhost zmf]# make install

# 可省略
[root@localhost zmf]# ln -s /data/application/nginx-1.10.3/ /etc/nginx
[root@localhost zmf]# ln -s /data/application/nginx-1.10.3/sbin/nginx /usr/local/sbin/

# 使用 nginx -V 可以查看编译是的参数
[root@localhost zmf]# ./nginx -V
nginx version: nginx/1.10.3
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/usr/local/zmf/nginx --conf-path=/usr/local/zmf/nginx/conf/nginx.conf  --with-http_ssl_module

# 检查配置文件语法，可以防止因配置错误导致网站重启或重新加载配置等对用户的影响
[root@localhost zmf]# ./nginx -t
nginx: the configuration file /usr/local/zmf/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/zmf/nginx/conf/nginx.conf test is successful

# 启动 nginx 服务
[root@localhost zmf]# ./nginx /usr/local/zmf/nginx/nginx.conf

# 查看是否启动成功
[root@localhost zmf]#  ps -ef | grep nginx 
root      6820     1  0 11:27 ?        00:00:00 nginx: master process ./nginx
nobody    6821  6820  0 11:27 ?        00:00:00 nginx: worker process
root      6823  6762  0 11:27 pts/0    00:00:00 grep --color=auto nginx
# 停止nginx服务
[root@localhost sbin]# ./nginx -s stop
```



Next->[Nginx+Keepalived讲解](./Nginx+Keepalived.md)

