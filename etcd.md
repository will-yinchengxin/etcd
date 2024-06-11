# [ETCD](https://etcd.io/docs)

## 简述

etcd是一个**非常可靠**的kv存储系统，常在分布式系统中存储着关键的数据。它是由coreos团队开发并开源的分布式键值存储系统，具备以下特点：

- 简单：提供定义明确且面向用户的API
- 安全：支持SSL证书验证
- 性能：基准压测支持1w+/sec写入
- 可靠：采用Raft协议保证分布式系统数据的可用性和一致性。

根据官网的介绍介绍, 读：1w ~ 10w 之间, 写：1w左右

### 对比 Redis

etcd 分布式存储系统，并在k8s 服务注册和服务发现中，为大家所认识。`它偏重的是节点之间的通信和一致性的机制保证，并不强调单节点的读写性能`。

redis 更加`侧重于读写性能和支持更多的存储模式，它不需要care一致性，也不需要侧重事务，因此可以达到很高的读写性能`。

redis常用来做缓存系统，etcd常用来做分布式系统中一些关键数据的存储服务，比如服务注册和服务发现。

## 集群部署

机器准备:  `172.16.27.95`   `172.16.27.96`    `172.16.27.97`

安装后手动启动三台服务

```shell
nohup /usr/local/etcd/etcd --name infra95 --initial-advertise-peer-urls http://172.16.27.95:2380 \
--listen-peer-urls http://172.16.27.95:2380 \
--listen-client-urls http://172.16.27.95:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://172.16.27.95:2379 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster infra95=http://172.16.27.95:2380,infra96=http://172.16.27.96:2380,infra97=http://172.16.27.97:2380 \
--initial-cluster-state new 2>&1 &


nohup /usr/local/etcd/etcd --name infra96 --initial-advertise-peer-urls http://172.16.27.96:2380 \
--listen-peer-urls http://172.16.27.96:2380 \
--listen-client-urls http://172.16.27.96:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://172.16.27.96:2379 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster infra95=http://172.16.27.95:2380,infra96=http://172.16.27.96:2380,infra97=http://172.16.27.97:2380 \
--initial-cluster-state new 2>&1 &


nohup /usr/local/etcd/etcd --name infra97 --initial-advertise-peer-urls http://172.16.27.97:2380 \
--listen-peer-urls http://172.16.27.97:2380 \
--listen-client-urls http://172.16.27.97:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://172.16.27.97:2379 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster infra95=http://172.16.27.95:2380,infra96=http://172.16.27.96:2380,infra97=http://172.16.27.97:2380 \
--initial-cluster-state new 2>&1 &
```

提供 shell 脚本

```shell
case $1 in
"start")
# for i in master1 master2 master3
for i in 172.16.27.95  172.16.27.96  172.16.27.97
 do
 echo " --------启动 $i etcd-------"
 echo "ssh $i nohup /usr/local/etcd/etcd --name $i-etcd --initial-advertise-peer-urls http://$i:2380 \
        --listen-peer-urls http://$i:2380 \
        --listen-client-urls http://$i:2379,http://127.0.0.1:2379 \
        --advertise-client-urls http://$i:2379 \
        --initial-cluster-token etcd-cluster-1 \
        --initial-cluster 172.16.27.95-etcd=http://172.16.27.95:2380,172.16.27.96-etcd=http://172.16.27.96:2380,172.16.27.97-etcd=http://172.16.27.97:2380 \
        --initial-cluster-state new >/dev/null 2>&1 &"
 ssh $i "nohup /usr/local/etcd/etcd --name $i-etcd --initial-advertise-peer-urls http://$i:2380 \
         --listen-peer-urls http://$i:2380 \
         --listen-client-urls http://$i:2379,http://127.0.0.1:2379 \
         --advertise-client-urls http://$i:2379 \
         --initial-cluster-token etcd-cluster-1 \
         --initial-cluster 172.16.27.95-etcd=http://172.16.27.95:2380,172.16.27.96-etcd=http://172.16.27.96:2380,172.16.27.97-etcd=http://172.16.27.97:2380 \
         --initial-cluster-state new >/dev/null 2>&1 &"
 done
 echo "Etcd 集群启动完成"
;;
"stop")
 for i in 172.16.27.95  172.16.27.96  172.16.27.97
 do
 echo " --------关闭 $i etcd-------"
 ssh $i "pkill etcd"
 sleep 1
 done
 echo "Etcd 集群关闭完成"
;;
esac
```

```
[root@master etcd]# ectl --write-out=table --cluster endpoint status
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://172.16.27.97:2379 |  6469dfd7e88421b |  3.4.32 |   20 kB |     false |      false |         6 |          9 |                  9 |        |
| http://172.16.27.96:2379 | 9f9e3875cb6255f4 |  3.4.32 |   20 kB |     false |      false |         6 |          9 |                  9 |        |
| http://172.16.27.95:2379 | efce8cdd54454cbd |  3.4.32 |   20 kB |      true |      false |         6 |          9 |                  9 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

[root@master ~]# ectl --write-out=table member list
+------------------+---------+-------------------+--------------------------+--------------------------+------------+
|        ID        | STATUS  |       NAME        |        PEER ADDRS        |       CLIENT ADDRS       | IS LEARNER |
+------------------+---------+-------------------+--------------------------+--------------------------+------------+
|  6469dfd7e88421b | started | 172.16.27.97-etcd | http://172.16.27.97:2380 | http://172.16.27.97:2379 |      false |
| 9f9e3875cb6255f4 | started | 172.16.27.96-etcd | http://172.16.27.96:2380 | http://172.16.27.96:2379 |      false |
| efce8cdd54454cbd | started | 172.16.27.95-etcd | http://172.16.27.95:2380 | http://172.16.27.95:2379 |      false |
+------------------+---------+-------------------+--------------------------+--------------------------+------------+

```

## [常用交互命令](https://etcd.io/docs/v3.5/tutorials/)

添加环境变量

```shell
vim /etc/profile

# 添加如下内容
export ETCD=/usr/local/etcd
export PATH=$PATH:$ETC
```

```shell
# 这里将 etcdctl 重命名为 ectl, 看个人喜好
[root@master ~]# ll /usr/local/etcd
total 37108
drwx------  3 root root       20 Jun  5 15:32 default.etcd
drwxr-xr-x 14 root root     4096 Apr 26 02:47 Documentation
-rwxr-xr-x  1 root root 16592896 Apr 26 02:47 ectl
-rwxr-xr-x  1 root root 21307392 Apr 26 02:47 etcd
drwx------  3 root root       20 Jun  5 17:27 infra95.etcd
drwx------  3 root root       20 Jun  5 16:43 my-etcd-data
-rw-------  1 root root    24531 Jun  5 18:28 nohup.out
-rw-r--r--  1 root root    43094 Apr 26 02:47 README-etcdctl.md
-rw-r--r--  1 root root     8431 Apr 26 02:47 README.md
-rw-r--r--  1 root root     7855 Apr 26 02:47 READMEv2-etcdctl.md
-rwxr-xr-x  1 root root      473 Jun  5 18:32 start.sh
```

这里提及几个基本的命令

- 设置键值:  `etcdctl put foo bar`
- 读取键值:  `etcdctl get foo`
- 读取前缀的所有键的命令:  `etcdctl get --prefix foo`
- 删除键 :  `etcdctl del foo`

###  [**租约**](https://etcd.io/docs/v3.5/tutorials/how-to-create-lease/)

授予租约: 

```shell
# 授予租约，TTL为120秒
[root@master ~]# ectl lease grant 120
lease 4cbd8fec6893a808 granted with TTL(120s)

# 附加键 a 到租约 4cbd8fec6893a808
[root@master ~]# ectl put a AA --lease=4cbd8fec6893a808 
OK
```

撤销租约

```shell
[root@master ~]# ectl lease revoke 4cbd8fec6893a808
lease 4cbd8fec6893a808 revoked

# 空应答，因为租约撤销导致 a 被删除
[root@master ~]# ectl get a
```

维持租约

```shell
[root@master ~]# ectl lease grant 10
lease 4cbd8fec6893a810 granted with TTL(10s)

[root@master ~]# ectl lease keep-alive  4cbd8fec6893a810
lease 4cbd8fec6893a810 keepalived with TTL(10)
```

查看租约的 TTL, 这里所有续租到该租约上的 key 的 TTL 为同一个值

```shell
[root@master ~]# ectl lease keep-alive  4cbd8fec6893a810
lease 4cbd8fec6893a810 keepalived with TTL(10)

[root@master ~]# ectl lease timetolive  4cbd8fec6893a812
lease 4cbd8fec6893a812 granted with TTL(500s), remaining(485s)

[root@master ~]# ectl lease timetolive --keys 4cbd8fec6893a812
lease 4cbd8fec6893a812 granted with TTL(500s), remaining(337s), attached keys([a b])
```

### [**锁**](https://etcd.io/docs/v3.5/tutorials/how-to-create-locks/)

```shell
[root@master ~]# ectl lock mutex1
```

## 核心 API

















