# Nginx+lvs+keepalived 

场景引入：当应用服务器配置成集群后，对于一些特大型的网站，性能的瓶颈就来自于nginx了，因为单机的nginx的并发能力是有上限的。而nginx本身是不支持集群模式的，因而此时对nginx的横向扩展就显得尤为重要。但是nginx是不支持横向扩容的，此时nginx就会成为性能瓶颈。

解决方案：lvs是一款负载均衡工具，因而如果我们结合lvs和nginx，那么就可以通过部署多台nginx服务器，通过lvs的负载均衡能力，将请求均衡的分发到各个nginx服务器上，再由nginx服务器分发到各个应用服务器，这样，我们就实现了nginx的横向扩展了。

​		lvs是一款用于四层负载均衡的工具。所谓的四层负载均衡，对应的是网络七层协议，常见的如HTTP协议是建立在七层协议上的，而lvs作用于四层协议上，也即：传输层，网络层，数据链路层和物理层。这里的传输层主要协议有TCP和UDP协议，也就是说lvs主要支持的方式是TCP和UDP。也正是因为lvs是处于四层负载均衡上的，因而其处理请求的能力比常见的服务器要高非常多，比如nginx的请求处理就是建立在网络七层上的，lvs的负载均衡能力是nginx的十倍以上。

​		LVS 的容灾可以通过主备+心跳的方式实现。主 LVS 失去心跳后，备 LVS 可以作为热备立即替换。容灾主要是靠 KeepAlived 来做的。

### 1. 环境准备

```javascript
1. VMware;
2. 4台CentOs7虚拟主机：172.16.28.130, 172.16.28.131, 172.16.28.132, 172.16.28.133
3. 系统服务：LVS, Keepalived
4. Web服务器：nginx
5. 集群搭建：LVS DR模式
```

### 2. 软件安装

​        在四台虚拟机上，我们以如下方式搭建集群：

```javascript
172.16.28.130 lvs+keepalived
172.16.28.131 lvs+keepalived
172.16.28.132 nginx
172.16.28.133 nginx
```

​        这里我们使用`172.16.28.130`和`172.16.28.131`两台机器作为`lvs+keepalived`的工作机器，也就是说这两台机器的作用主要是进行负载均衡和故障检测和下线的；我们使用`172.16.28.132`和`172.16.28.133`两台机器作为应用服务器，主要是对外提供服务的。这四台服务器作为整个后端集群服务，并且对外提供的虚拟ip是`172.16.28.120`。需要说明的是，这里的`keepalived`所检测的服务是两台`lvs`服务器，这两台服务器，一台作为master服务器，一台作为backup服务器，两者在负载均衡的配置上是完全一样的。在正常情况下，客户端请求虚拟ip的时候，`lvs`会将该请求转发到master服务器上，然后master服务器根据配置的负载均衡策略选择一台应用服务器，并且将请求发送给该应用服务器进行处理。如果在某个时刻，lvs的master服务器由于故障宕机了，keepalived就会检测到该故障，并且进行故障下线，然后将backup机器上线用于提供服务，从而实现故障转移的功能。

#### 2.1 lvs+keepalived安装

​        在`172.16.28.130`和`172.16.28.131`上安装ipvs和keepalived：

```javascript
# 安装ipvs
sudo yum install ipvsadm
# 安装keepalived
sudo yum install keepalived
```

​        在`172.16.28.132`和`172.16.28.133`上安装nginx：

```javascript
# 安装nginx
sudo yum install nginx
```

​        需要注意的是，在两台nginx服务器上需要将防火墙关闭，否则lvs+keepalived的两台机器就无法将请求发送到两台nginx服务器上来：

```javascript
# 关闭防火墙
systemctl disable firewalld.service
```

​        查看两台负载均衡机器是否支持lvs：

```javascript
sudo lsmod |grep ip_vs
# 如果看到如下结果，则说明是支持的
[zhangxufeng@localhost ~]$ sudo lsmod|grep ip_vs
ip_vs                 145497  0
nf_conntrack          137239  1 ip_vs
libcrc32c              12644  3 xfs,ip_vs,nf_conntrack
```

​        如果上述命令没有任何结果，则执行`sudo ipvsadm`命令启动ipvs之后，再通过上述命令进行查看即可。启动ipvs之后，我们就可以在`/etc/keepalived/`目录下编辑`keepalived.conf`文件，我们以`172.16.28.130`机器作为master机器，master节点配置如下：

```javascript
# Global Configuration
global_defs {
  lvs_id director1  # 指定lvs的id
}

# VRRP Configuration
vrrp_instance LVS {
  state MASTER	# 指定当前节点为master节点
  interface ens33	# 这里的ens33是网卡的名称，通过ifconfig或者ip addr可以查看
  virtual_router_id 51	# 这里指定的是虚拟路由id，master节点和backup节点需要指定一样的
  priority 151	# 指定了当前节点的优先级，数值越大优先级越高，master节点要高于backup节点
  advert_int 1	# 指定发送VRRP通告的间隔，单位是秒
  authentication {
    auth_type PASS	# 鉴权，默认通过
    auth_pass 123456	# 鉴权访问密码
  }

  virtual_ipaddress {
    172.16.28.120	# 指定了虚拟ip
  }

}

# Virtual Server Configuration - for www server
# 后台真实主机的配置
virtual_server 172.16.28.120 80 {
  delay_loop 1	# 健康检查的时间间隔
  lb_algo rr	# 负载均衡策略，这里是轮询
  lb_kind DR	# 调度器类型，这里是DR
  persistence_time 1	# 指定了持续将请求打到同一台真实主机的时间长度
  protocol TCP	# 指定了访问后台真实主机的协议类型

  # Real Server 1 configuration
  # 指定了真实主机1的ip和端口
  real_server 172.16.28.132 80 {
    weight 1	# 指定了当前主机的权重
    TCP_CHECK {
      connection_timeout 10	# 指定了进行心跳检查的超时时间
      nb_get_retry 3	# 指定了心跳超时之后的重复次数
      delay_before_retry 3	# 指定了在尝试之前延迟多长时间
    }
  }

  # Real Server 2 Configuration
  real_server 172.16.28.133 80 {
    weight 1	# 指定了当前主机的权重
    TCP_CHECK {
      connection_timeout 10	# 指定了进行心跳检查的超时时间
      nb_get_retry 3	# 指定了心跳超时之后的重复次数
      delay_before_retry 3	# 指定了在尝试之前延迟多长时间
    }
  }
}
```

​        上面是master节点上keepalived的配置，对于backup节点，其配置与master几乎是一致的，只是其state和priority参数不同。如下是backup节点的完整配置：

```javascript
# Global Configuration
global_defs {
  lvs_id director2  # 指定lvs的id
}

# VRRP Configuration
vrrp_instance LVS {
  state BACKUP	# 指定当前节点为master节点
  interface ens33	# 这里的ens33是网卡的名称，通过ifconfig或者ip addr可以查看
  virtual_router_id 51	# 这里指定的是虚拟路由id，master节点和backup节点需要指定一样的
  priority 150	# 指定了当前节点的优先级，数值越大优先级越高，master节点要高于backup节点
  advert_int 1	# 指定发送VRRP通告的间隔，单位是秒
  authentication {
    auth_type PASS	# 鉴权，默认通过
    auth_pass 123456	# 鉴权访问密码
  }

  virtual_ipaddress {
    172.16.28.120	# 指定了虚拟ip
  }

}

# Virtual Server Configuration - for www server
# 后台真实主机的配置
virtual_server 172.16.28.120 80 {
  delay_loop 1	# 健康检查的时间间隔
  lb_algo rr	# 负载均衡策略，这里是轮询
  lb_kind DR	# 调度器类型，这里是DR
  persistence_time 1	# 指定了持续将请求打到同一台真实主机的时间长度
  protocol TCP	# 指定了访问后台真实主机的协议类型

  # Real Server 1 configuration
  # 指定了真实主机1的ip和端口
  real_server 172.16.28.132 80 {
    weight 1	# 指定了当前主机的权重
    TCP_CHECK {
      connection_timeout 10	# 指定了进行心跳检查的超时时间
      nb_get_retry 3	# 指定了心跳超时之后的重复次数
      delay_before_retry 3	# 指定了在尝试之前延迟多长时间
    }
  }

  # Real Server 2 Configuration
  real_server 172.16.28.133 80 {
    weight 1	# 指定了当前主机的权重
    TCP_CHECK {
      connection_timeout 10	# 指定了进行心跳检查的超时时间
      nb_get_retry 3	# 指定了心跳超时之后的重复次数
      delay_before_retry 3	# 指定了在尝试之前延迟多长时间
    }
  }
}
```

​        将master和backup配置成完全一样的原因是，在master宕机时，可以根据backup的配置进行服务的无缝切换。

​        在lvs+keepalived机器配置完成之后，我们下面配置两台应用服务器的nginx配置。这里我们是将nginx作为应用服务器，在其配置文件中配置返回状态码为200，并且会将当前主机的ip返回，如下是其配置：

```javascript
worker_processes auto;
# pid /run/nginx.pid;

events {
  worker_connections 786;
}

http {
  server {
    listen 80;

    # 这里是直接返回200状态码和一段文本
    location / {
      default_type text/html;
      return 200 "Hello, Nginx! Server zhangxufeng@172.16.28.132\n";
    }
  }
}
worker_processes auto;
# pid /run/nginx.pid;

events {
  worker_connections 786;
}

http {
  server {
    listen 80;

    # 这里是直接返回200状态码和一段文本
    location / {
      default_type text/html;
      return 200 "Hello, Nginx! Server zhangxufeng@172.16.28.133\n";
    }
  }
}
```

​        可以看到，两台机器返回的文本中主机ip是不一样的。nginx配置完成后，可以通过如下命令进行启动：

```javascript
sudo nginx
```

​        在启动nginx之后，我们需要配置虚拟ip，这是因为我们使用的lvs调度器是DR模式，前面我们讲到过，这种模式下，对客户端的响应是真实服务器直接返回给客户端的，而真实服务器需要将响应报文中的源ip修改为虚拟ip，这里配置的虚拟ip就是起这个作用的。我们编辑`/etc/init.d/lvsrs`文件，写入如下内容：

```javascript
#!/bin/bash

ifconfig lo:0 172.16.28.120 netmask 255.255.255.255 broadcast 172.16.28.120 up
route add -host 172.16.28.120 dev lo:0

echo "0" > /proc/sys/net/ipv4/ip_forward
echo "1" > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" > /proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce

exit 0
```

- lo：表示当前主机真实网卡的名称；
- 172.16.28.120：表示虚拟ip；

​        编写完成后运行该脚本文件即可。然后将两台lvs+keepalived机器上的keepalived服务启动起来即可：

```javascript
sudo service keepalived start
```

​        最后可以通过如下命令查看配置的lvs+keepalived的策略：

```javascript
[zhangxufeng@localhost keepalived]$ sudo ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.16.28.120:80 rr
  -> 172.16.28.132:80             Route   1      0          0
```

#### 2.2 集群测试

​        根据上述步骤，我们配置完成了一个lvs+keepalived+nginx的集群。在浏览器中，我们可以访问`http://172.16.28.120`即可看到如下响应：

```javascript
Hello, Nginx! Server zhangxufeng@172.16.28.132
```

​        多次刷新浏览器之后，可以看到浏览器中显示的文本切换如下，这是因为lvs的负载均衡策略产生的：

```javascript
Hello, Nginx! Server zhangxufeng@172.16.28.133
```

### 3. 小结

​        本文首先对lvs和keepalived的工作原理进行了讲解，分别介绍了其工作的几种模式，然后对lvs+keepalived+nginx搭建nginx集群的方式进行详细讲解，并且说明了其中所需要注意的问题。