# Nginx+Keepalived集群实战

技术介绍：Nginx负载均衡一般位于整个网站架构的最前端或者中间层，如果为最前端时单台Nginx会存在单点故障，也就是一台Nginx宕机，会影响用户对整个网站的访问。所以需要加入Nginx备份服务器，Nginx主服务器与备份服务器之间形成高可用，一旦发现Nginx主宕机，能快速将网站恢复至备份主机。

环境准备：

| 主机名            | IP地址        | 节点类型 | nginx版本     | Keepalived版本    |
| ----------------- | ------------- | -------- | ------------- | ----------------- |
| Nginx-1（Master） | 192.168.33.8  | 主       | nginx v1.12.0 | keepalived v1.2.1 |
| Nginx-2(backup)   | 192.168.33.10 | 备       | nginx v1.12.0 | keepalived v1.2.1 |

步骤1：   Nginx安装配置，Master、Backup服务器安装Nginx、keepalived，yum install -y pcre-devel  安装perl 兼容的正规表达式库。

```shell
tar -xzf nginx-1.12.0.tar.gz
cd nginx-1.12.0 
sed -i -e 's/1.12.0//g' -e 's/nginx\//TDTWS/g' -e 's/"NGINX"/"TDTWS"/g' src/core/nginx.h 
./configure --prefix=/usr/local/nginx --user=www --group=www  --with-http_stub_status_module --with-http_ssl_module
make
make install

```

步骤二：   Keepalived安装配置

```shell
tar  -xzvf  keepalived-1.2.1.tar.gz
cd keepalived-1.2.1 
./configure --prefix=/usr/local/keepalived/ --with-kernel-dir=/usr/src/kernels/3.10.0-514.el7.x86_64/
make
make install 
DIR=/usr/local/
\cp $DIR/etc/rc.d/init.d/keepalived  /etc/rc.d/init.d/
\cp $DIR/etc/sysconfig/keepalived   /etc/sysconfig/ 
mkdir  -p  /etc/keepalived
\cp   $DIR/sbin/keepalived         /usr/sbin/

```

步骤三：     配置Keepalived，两台服务器keepalived.conf内容都为如下，state均设置为backup，Backup服务器需要修改优先级为90。

```
! Configuration File for keepalived 
 global_defs { 
  notification_email { 
      support@jfedu.net
 } 
    notification_email_from wgkgood@163.com 
    smtp_server 127.0.0.1 
    smtp_connect_timeout 30 
    router_id LVS_DEVEL 
 } 
 
 vrrp_script chk_nginx { 
    script  "/data/sh/check_nginx.sh" 
    interval 2 
    weight 2 
 } 
 # VIP1 
 vrrp_instance VI_1 { 
     state BACKUP 
     interface eth0 
     lvs_sync_daemon_inteface eth0 
     virtual_router_id 151 
     priority 100 
     advert_int 5 
     nopreempt 
     authentication { 
         auth_type  PASS 
         auth_pass  1111
 
     } 
     virtual_ipaddress { 
         192.168.0.198 
     } 
     track_script { 
     chk_nginx 
    } 
 }

```

步骤四：如上配置还需要建立check_nginx脚本，用于检查本地Nginx是否存活，如果不存活，则kill keepalived实现切换。其中check_nginx.sh脚本内容如下：



```shell
#!/bin/bash 
#auto  check  nginx  process 
#2017-5-26 17:47:12
#by  author  jfedu.net 
killall  -0   nginx
if  [[ $? -ne 0 ]]；then 
/etc/init.d/keepalived stop 
fi

```

最后在两台Nginx服务器分别新建index.html测试页面，然后启动Nginx服务测试，访问VIP地址，http://192.168.0.198即可。