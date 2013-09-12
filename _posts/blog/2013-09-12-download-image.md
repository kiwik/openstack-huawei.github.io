---
layout:     post
title:      第一次创建虚拟机耗时原因验证及优化
category: blog
description: OpenStack在第一次创建虚拟机时，很大一部分时间都在下载镜像，如果镜像较大，则创建虚拟机耗时较长，用户体验不好；同时，对于在线迁移，在目的节点也需要下载虚拟机镜像（如果之前该镜像没有被下载过），这样就导致迁移时间过长，增加了虚拟机的停顿时间。
---

## 第一次创建虚拟机耗时原因验证及优化

Author: Liu Sheng  
Date: 2013/9/12 13:27:42 

OpenStack在第一次创建虚拟机时，很大一部分时间都在下载镜像，如果镜像较大，则创建虚拟机耗时较长，用户体验不好；同时，对于在线迁移，在目的节点也需要下载虚拟机镜像（如果之前该镜像没有被下载过），这样就导致迁移时间过长，增加了虚拟机的停顿时间。试验如下：

1.指定节点在之前使用相同镜像已创建过虚拟机的计算节点创建一个VM（之前已经创建过虚拟机，在_base目录下有镜像文件），日志记录从libvirt的create image到创建成功如下：

![](/images/blog/download-image/image001.png)

![](/images/blog/download-image/image003.png)

用时：5s

2.将_base目录下镜像移除：

![](/images/blog/download-image/image005.png)

创建虚拟机，日志记录从libvirt的create image到创建成功如下：

![](/images/blog/download-image/image007.png)

![](/images/blog/download-image/image009.png)

用时：22s

3.使用脚本将镜像下载到_base目录下

![](/images/blog/download-image/image011.png)

![](/images/blog/download-image/image013.png)

再次创建虚拟机，日志记录从libvirt的create image到创建成功如下：

![](/images/blog/download-image/image015.png)

![](/images/blog/download-image/image017.png)

用时：4s  
由此，可以看出预先用脚本将镜像下载到_base目录下会缩短创建第一个虚拟机的时间，这里使用ubuntu的镜像（241M）做实验，若使用一个尺寸更大的镜像，效果也会更加明显。

代码参见：<https://github.com/openstack-huawei/download-image.git>

