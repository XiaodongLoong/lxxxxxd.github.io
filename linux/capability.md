容器里面的权限是如何控制的，docker提供了很多的命令和选项：
privileged：给容器进程配置超级用户所拥有的权限，
cap-add:添加linux权限能力
cap-drop:删除linux权限能力

普通容器的权限配置：
```
[root@localhost ~]# docker run -it busybox sh
/ # cat /proc/self/status |grep Cap
CapInh:	00000000a80425fb
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
/ # 
```
超级权限的容器权限配置：
```
[root@localhost ~]# docker run -it --privileged busybox sh
/ # cat /proc/self/status |grep Cap
CapInh:	0000003fffffffff
CapPrm:	0000003fffffffff
CapEff:	0000003fffffffff
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
/ # 
```
