1.关掉selinux

```
#vim /etc/selinux/config
SELINUX=disabled
```

2.设置固定IP

```
vim /etc/sysconfig/network-scripts/ifcfg-eth0
BOOTPROTO=static
IPADDR=192.168.234.4
NETMASK=255.255.255.0
GATEWAY=192.168.234.2
DNS1=16.110.135.52
DNS2=16.110.135.51
```

3.设置hostname

```
hostnamectl --static set-hostname master
```

4.设置网络

```

```

5.设置防火墙

```

```



