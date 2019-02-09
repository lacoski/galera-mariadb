# Triển khai Haproxy Pacemaker cho Cluster Galera 3 node
---
## Phần 1. Chuẩn bị
### Phân hoạch
| Hostname | Hardware                      | Interface                                               |
|----------|-------------------------------|---------------------------------------------------------|
| node1    | 2 Cpu - 2gb Ram - 100 gb Disk | ens160: 10.10.10.86 (MNGT) - ens192: 10.10.11.86 (REPL) |
| node2    | 2 Cpu - 2gb Ram - 100 gb Disk | ens160: 10.10.10.87 (MNGT) - ens192: 10.10.11.87 (REPL) |
| node3    | 2 Cpu - 2gb Ram - 100 gb Disk | ens160: 10.10.10.88 (MNGT) - ens192: 10.10.11.88 (REPL) |

### Mô hình

![](/images/pic1.png)

### Thiết lập ban đầu

Thực hiện cấu hình theo docs triển galera 3 node

## Phần 2. Cài đặt Haproxy bản 1.8

> Thực hiện trên tất cả các node

Cài đặt
```
sudo yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y
```

Tạo bản backup cho cấu hình mặc định và chỉnh sửa cấu hình HAproxy
```
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```

Cầu hình Haproxy
```
echo 'global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen stats
    bind :8080
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics

listen galera 10.10.10.89:3306
    balance source
    mode tcp
    option tcpka
    option tcplog
    option clitcpka
    option srvtcpka
    timeout client 28801s
    timeout server 28801s 
    option mysql-check user haproxy
    server node1 10.10.10.86:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3 
    server node2 10.10.10.87:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3 backup
    server node3 10.10.10.88:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3 backup' > /etc/haproxy/haproxy.cfg
```

Cấu hình Log cho HAProxy
```
sed -i "s/#\$ModLoad imudp/\$ModLoad imudp/g" /etc/rsyslog.conf
sed -i "s/#\$UDPServerRun 514/\$UDPServerRun 514/g" /etc/rsyslog.conf
echo '$UDPServerAddress 127.0.0.1' >> /etc/rsyslog.conf

echo 'local2.*    /var/log/haproxy.log' > /etc/rsyslog.d/haproxy.conf

systemctl restart rsyslog
```

Bổ sung cấu hình cho phép kernel có thể binding tới IP VIP
```
echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf
```

Kiểm tra
```
$ sysctl -p

net.ipv4.ip_nonlocal_bind = 1
```

## Phần 3. Triển khai Cluster Pacemaker

### Bước 1: Cài đặt pacemaker corosync

__Lưu ý: Thực hiện trên tất cả các node__

Cài đặt gói pacemaker pcs
```
yum -y install pacemaker pcs

systemctl start pcsd 
systemctl enable pcsd
```

Thiết lập mật khẩu user `hacluster`
```
passwd hacluster
```

Lưu ý: Nhập chính xác và nhớ mật khẩu user hacluster, đồng bộ mật khẩu trên tất cả các node



### Bước 2: Tạo Cluster

Chứng thực cluster (Chỉ thực thiện trên cấu hình trên một node duy nhất, trong bài sẽ thực hiện trên `node1`), nhập chính xác tài khoản user hacluster
```
$ pcs cluster auth node1 node2 node3

Username: hacluster
Password: *********

```
