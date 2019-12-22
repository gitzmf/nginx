## **lvs-dr实现**

实验拓扑如下： 

![lvs_dr拓扑.jpg-98.4kB](images/lvs_dr%E6%8B%93%E6%89%91.jpg)

```
后端两台real server搭建httpd服务（默认已搭建完成并启动）,各节点iptables和selinux均已关闭
```

 step1在director上执行如下操作：

```
#ifconfig eno16777736:0 192.168.2.11/32 broadcast 192.168.2.11 up
#route add -host 192.168.2.11 dev eno16777736:0
```

step2在real server1上执行如下操作：

```
#echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
#echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
#echo 1 > /proc/sys/net/ipv4/conf/eno16777736/arp_ignore
# echo 2 > /proc/sys/net/ipv4/conf/eno16777736/arp_announce
#添加如上内核参数，注意要在director节点添加ipvs规则前做此步操作
#ifconfig lo:0 192.168.2.11/32 broadcast 192.168.2.11 up
#route add -host 192.168.2.11 dev lo:0
#echo "<h1>This is Real Server 1 </h1>" > /var/www/html/index.html
```

 step3在real server2上执行如下操作：

```
#echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
#echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
#echo 1 > /proc/sys/net/ipv4/conf/eno16777736/arp_ignore
# echo 2 > /proc/sys/net/ipv4/conf/eno16777736/arp_announce
#添加如上内核参数，注意要在director节点添加ipvs规则前做此步操作
#ifconfig lo:0 192.168.2.11/32 broadcast 192.168.2.11 up
#route add -host 192.168.2.11 dev lo:0
#echo "<h1>This is Real Server 2 </h1>" > /var/www/html/index.html
```

step4在director节点执行如下操作：

```
  #ipvsadm -A -t 192.168.2.11:80 -s rr
 #ipvsadm -a -t 192.168.2.11:80 -r 192.168.2.117 -g
 #ipvsadm -a -t 192.168.2.11:80 -r 192.168.2.135 -g
```

测试： 


 ![QQ图片20161212213930.png-30.4kB](images/QQ%E5%9B%BE%E7%89%8720161212213930.png) 

 ![QQ图片20161212213938.png-40.2kB](images/QQ%E5%9B%BE%E7%89%8720161212213938.png)

