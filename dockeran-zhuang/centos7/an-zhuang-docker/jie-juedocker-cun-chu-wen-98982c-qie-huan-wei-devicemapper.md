1.1创建硬盘

```
parted /dev/sdb
mklabel msdos   (小于1T) #mklabel gpt(大于1t)
mkpart primary 2048s 100%
toggle 1 lvm
quit
reboot
```

1.2 创建PV/VG

```
pvcreate /dev/sdb1  #创建一个物理卷
vgcreate vg_sxf /dev/sdb1 #创建卷组 "vg_sxf".并且将物理卷/dev/sdb1添加到卷组中.创建一个虚拟卷
```

1.3 变更配置

```
vim /etc/sysconfig/docker-storage-setup
# DEVS=/dev/vdc     # 注释该行
加入
VG=vg_sxf
DATA_SIZE=30G  #默认用一半.不设置的情况下
```

1.4 lvg挂载

```
#ls -lsh /var/lib/docker/devicemapper/
0 drwx------ 2 root root 72 Mar 12 12:49 data      #存放数据
0 drwx------ 2 root root 72 Mar 12 12:49 metadata  #存放元数据
```

1.5 生成新卷

```
#service docker stop
#rm -rf /var/lib/docker/ #删除历史docker目录
#docker-storage-setup  #执行setup操作.相关lvm将自动创建

#service docker start

验证会发现：
Data file:
Metadata file:
```

1.6 vim /etc/lvm/profile/vg\_sxf--docker-pool-extend.profile

```
activation {
        thin_pool_autoextend_threshold=60
        thin_pool_autoextend_percent=20
}

DATA_SIZE=40%FREE            #定义创建 DATA thin pool 的大小，默认为 VG 的 40%
MIN_DATA_SIZE=2G             #定义 DATA pool 最小值，默认为 2G，如果 VG 小于 2G 则创建失败
CHUNK_SIZE=512K              #定义 thin pool 的 CHUNK 大小，默认 512k
AUTO_EXTEND_POOL=yes         #定义是否自动扩容 thin pool 大小，默认为自动扩容
POOL_AUTOEXTEND_THRESHOLD=60 #定义自动扩容的百分比，默认为当前 pool 使用 60% 时自动扩容，100 表示 disable，最小为 50 lvmthin — LVM thin provisioning
POOL_AUTOEXTEND_PERCENT=20   #定义每次扩容的大小，默认为 20%，即当前 pool 大小为 100G，那么自动扩容 20G，扩容后大小为 120G， 100 表示 disable
```

1.7 验证监控状态

```
[root@ks8_01 ~]# lvs -o+seg_monitor
  LV          VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Monitor  
  root        cl     -wi-ao---- <35.12g                                                              
  swap        cl     -wi-ao----  <3.88g                                                              
  docker-pool vg_sxf twi-a-t---  30.00g             0.06   0.12                             monitored
```



