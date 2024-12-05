# 基于cgroup和Cache Allocation Technology (CAT)实现资源隔离
以下流程在Ubuntu22下测试通过
## 锁定cpu频率
```
sudo cpupower frequency-set --governor userspace
sudo cpupower frequency-set --freq 2250MHz
cpufreq-info
```

## 限制所有用户的资源使用

创建配置
```
sudo mkdir -p /etc/systemd/system/user-.slice.d
sudo nano /etc/systemd/system/user-.slice.d/limitcpus.conf
```
在limitcpus.conf中输入以下内容来控制所有用户对cpu资源的访问
```
[Slice]
AllowedCPUs=0-15,64-79
```
使其生效后，所有用户将被限制使用0-31和64-95的cpu逻辑核。
```
sudo systemctl daemon-reload
```

## cgroup v2 创建自己的group使其能访问所有cpu或访问指定cpu
```
sudo cgcreate -a $USER:$USER -t $USER:$USER -g cpuset:mygroup
sudo cgset -r cpuset.cpus=0-127 mygroup
```

使用cgexec来为cmd指定group从而访问所有cpu
```
sudo cgexec -g cpuset:mygroup cmd
```
## NUMA设定
先在bios中打开numa，通过numa节点划分来实现内存带宽隔离。
然后使用以下命令禁用 NUMA 自动均衡
```
sudo sysctl kernel.numa_balancing=0
```

## Cache Allocation Technology (CAT)
安装intel-cmt-cat

如果使用pqos命令出现初始化错误如下：
> API lock initialization error!  
> Allocation: Error initializing PQoS library!

使用以下命令解决
```
sudo rm /var/lock/libpqos
```

使用pqos限制COS1的资源，并绑定cpu核
```
sudo pqos -e "llc:1=0x000f;"
sudo pqos -a "llc:1=0-15,64-79;"
sudo pqos -e "mba:1=10;"
```
使用rdtset分配资源运行命令，同时使用numactl --membind来指定numa内存节点
```
sudo cgexec -g cpuset:mygroup rdtset -t 'l3=0xfff0;mba=90;cpu=32-63,96-127' -c 32-63,96-127 -k numactl --membind=1 cmd
```
重置
```
sudo pqos -R
```
