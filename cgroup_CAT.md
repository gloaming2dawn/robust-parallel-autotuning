# 基于cgroup和Cache Allocation Technology (CAT)实现资源隔离
以下流程在Ubuntu22下测试通过

## 限制所有用户的资源使用

创建配置
```
sudo mkdir -p /etc/systemd/system/user-.slice.d
sudo nano /etc/systemd/system/user-.slice.d/limitcpus.conf
```
在limitcpus.conf中输入以下内容来控制所有用户对cpu资源的访问
```
[Slice]
AllowedCPUs=0-31,64-95
```
使其生效后，所有用户将被限制使用0-31和64-95的cpu逻辑核。
```
systemctl daemon-reload
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

## Cache Allocation Technology (CAT)
安装intel-cmt-cat
如果命令出现初始化错误如下：
> API lock initialization error!  
> Allocation: Error initializing PQoS library!

使用以下命令解决
```
sudo rm /var/lock/libpqos
```
