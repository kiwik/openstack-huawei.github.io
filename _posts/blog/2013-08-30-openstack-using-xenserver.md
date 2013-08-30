---
layout:     post
title:      在OpenStack中使用XenServer资源池浅析
category: blog
description: 在OpenStack中使用XenServer资源池浅析
---

OpenStack中的Xen driver类：nova/virt/xenapi/driver.py中的XenAPIDriver类，该类继承的ComputeDriver是所有driver的基类，是所有虚拟机相关功能集合，而XenAPIDriver实现的方法是ComputeDriver类中方法的子集。

### 创建资源池并添加主机

使用xenserver资源池（支持虚拟机热迁移）前提：  
1. 有符合创建资源池的xenserver主机，已部署openstack（nova-compute）  
2. 有共享存储

步骤：  
1. 在master节点（注意:此时还没有资源池，master节点是我们预定义的某节点）上配置共享存储为默认SR  
2. 配置所有的slave节点使用该默认SR，通过配置项`sr_matching_filter=default-sr:true`  
3. 创建host aggregate。aggregate是上层逻辑概念，不对外公开，是管理员的配置对象。aggregate最初就是为了使用xenserver资源池功能而出现，但现在可以通过给aggregate对象添加key-value对，给nova-scheduler提供一种高级调度机制。host和aggregate是多对多的关系。

    nova aggregate-create <name-for-pool> <availability-zone>
    
上述操作只是操作数据表。  
4. 为配合xenserver的资源池的使用，需要给aggregate提供两个metadata

    nova aggregate-set-metadata <aggregate-id> hypervisor_pool=true
    nova aggregate-set-metadata <aggregate-id> operational_state=created
    
上述操作只是操作数据表。  
5. 将master节点加入aggregate

    nova aggregate-add-host <aggregate-id> <name-of-master-compute>
    
向主机组添加主机的流程如下：  
![流程图](/images/blog/openstack-using-xenserver/1.png)  
6. 加入其它slave节点

    nova aggregate-add-host <aggregate-id> <compute-host-name>
    
slave主机加入资源池后，在每个主机上的nova-compute虚拟机会被关机，待xenserver主机完成加入池的操作后，再把nova-compute虚拟机启动。

上述流程图中的最后一步理解的不是很清楚，不知道为何需要在master节点执行命令。XenServer官方文档中将一个主机加入资源池，是在预加入xenserver主机上执行（而不是在master节点执行）：

    xe pool-join master-address=<host1> master-username=<administrators_username> master-password=<password>

资源池创建成功后，就可以根据aggregate_metadata创建flavor，使用flavor创建的虚拟机就可以运行在XenServer资源池内的主机上，同时支持手动迁移和热迁移等高级特性。
