# cgroup v1

## 目录：
## 1.概念

1.1 什么是cgroup？

1.2 为什么需要cgroup？

1.3 cgroup怎么实现的？

1.4 notify_on_release 函数做了什么？

1.5 clone_children 函数做了什么？

1.6 怎样使用cgroup？

## 2.使用示例和语法

2.1 基本使用

2.2 追踪进程

2.3 以名字挂载层

## 内核API

3.1 概述

3.2 同步

3.3 子系统subsystem API

## 4.扩展属性使用

## 5.问题


# 正文

## 1.概念

1.1 什么是cgroup？

cgroup提供了对进程进行聚类或者分割的机制，聚类后的分组具有层级关系，每一层都有特殊的行为.

定义：
cgroup: cgroup将一组进程和一组针对一个或多个subsystem的一组参数联系起来。
subsystem: 充分利用进程分组来区别对待进程分组的模块。

1.2 为什么需要cgroup？

1.3 cgroup怎么实现的？

1.4 notify_on_release 函数做了什么？

1.5 clone_children 函数做了什么？

1.6 怎样使用cgroup？

## 2.使用示例和语法

2.1 基本使用

2.2 追踪进程

2.3 以名字挂载层

## 内核API

3.1 概述

3.2 同步

3.3 子系统subsystem API

## 4.扩展属性使用

## 5.问题

