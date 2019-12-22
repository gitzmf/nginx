# Keepalived-Nginx实现高可用

场景引入： 由于nginx本质上也是一款应用服务器，因而其也有可能出现宕机的情况。因此实现高可用是非常有必要的。

解决方案：这里结合keepalived就可以实现nginx的故障检测和服务切换。也就是说，通过keepalived+nginx，我们实现了nginx的高可用集群模式。

​		keepalived是一款服务器状态检测和故障切换的工具。在其配置文件中，可以配置主备服务器和该服务器的状态检测请求。也就是说keepalived可以根据配置的请求，在提供服务期间不断向指定服务器发送请求，如果该请求返回的状态码是200，则表示该服务器状态是正常的，如果不正常，那么keepalived就会将该服务器给下线掉，然后将备用服务器设置为上线状态。

## 1、Keepalived环境搭建

| 软件       | 版本   |
| ---------- | ------ |
| Nginx      | 1.10.3 |
| Keepalived | 1.2.18 |

### 1.1   安装步骤

```
# 在官方站下载keepalived-xxxx.tar.gz源码包
# 通过ftp上传keepalived-xxxx.tar.gz到服务器/usr/lcoal/zmf 指定目录下面
[root@localhost ~]# cd /usr/local/zmf
[root@localhost zmf]# tar -zxvf keepalived-xxxx.tar.gz
[root@localhost zmf]# cd keepalived-xxxx/
[root@localhost zmf]# mv keepalived-xxxx keepalived
[root@localhost zmf]# ./configure –-prefix=/usr/local/zmf/keepalived 
[root@localhost zmf]# make && make install
```

### 1.2  Keepalived服务配置

```
[root@localhost zmf]# mkdir /etc/keepalived
[root@localhost zmf]# cp /usr/local/zmf/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/  
# 复制keepalived服务脚本到默认的地址
[root@localhost zmf]# cp /usr/local/zmf/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/ 
[root@localhost zmf]# cp /usr/local/zmf/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
# 设置为系统服务
[root@localhost zmf]# cp /usr/local/zmf/keepalived/sbin/keepalived /usr/sbin/
# 设置服务为开机自启
[root@localhost zmf]# chkconfig keepalived on 
# 启动keepalived服务
[root@localhost zmf]# service keepalived start 
# 查看启动状况
[root@localhost zmf]# ps -ef|grep keepalived
```

### 1.3  nginx主备配置+Keepalived

 注意：因为安装keepalived不是按照默认安装的，所以配置了usr/local/xmf/keepalived下的keepalived.config文件后，还要将配置文件复制到/etc/keepalived下面

编辑keepalived.conf

```
# Configuration File for keepalived
# 全局配置
global_defs {	 
	 router_id # 主机名
}
# 检测nginx状态脚本
vrrp_script check_nginx_alive {
	script "/usr/bin/check_nginx_alive.sh" # 脚本位置 
	interval 3 # 心跳时间
	weight -10 # 权重
}
vrrp_instance VI_1 {
	# 主机配置，从机为BACKUP
	state MASTER
	# 网卡名称
	interface ens37 # 主机正在使用的网卡,通过ip addr或ifconfig可以查看
	virtual_router_id 51 # 与主机ip最后三位相同即可
	mcast_src_ip  主机ip
	# 权重值,值越大，优先级越高，backup设置比master小,这样就能在master宕机后讲backup变为master,而master回复后就可以恢复.
	priority 100
	advert_int 1 #指定发送VRRP通告的间隔。单位是秒。
	# 结点间认证方式
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	# 虚拟地址
	virtual_ipaddress {
		## 同一网段虚拟IP
		192.168.1.100
	}
	# 检测脚本 名字指定为上面定义的检测脚本块
	track_script {
		check_nginx_alive
	}
}
```

检测脚本

```
#!/bin/bash	 
A=`ps -C nginx --no-header |wc -l`	 
if [ $A -eq 0 ]
   then
	 /usr/local/zmf/nginx/sbin/nginx
	 sleep 2
	 if [ $A -eq 0 ]
		echo 'nginx server is died'
		killall keepalived
	fi
fi
```

### 1.4 nginx主备切换测试

1.修改主机nginx/html/index.html 添加主机标识信息“Thank you for using nginx-1”

2.修改从机nginx/html/index.html 添加从机标识信息“Thank you for using nginx-2”

3.访问Keepalived配置的虚拟IP，会出现主机的欢迎信息，把主机停掉后，在访问虚拟IP，会出现从机的欢迎信息

主机启动后，再次访问，又会切换到主机。

Next->[Keepalived配置文件讲解](./Keepalived配置文件详解.md)

Next->[Nginx+lvs讲解](./Nginx+LVS.md)

