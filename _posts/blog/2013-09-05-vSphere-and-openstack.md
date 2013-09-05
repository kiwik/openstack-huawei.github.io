---
layout:     post
title:      【译】vSphere与OpenStack的集成
category: blog
description: 本篇文章是对老外一个系列博客的翻译，讲述OpenStack和VMware如何集成，以及在集成过程中的问题讨论。
---

原文标题：OPENSTACK COMPUTE FOR VSPHERE ADMINS  
原文链接（一共五篇，这里列出第一篇）：http://cloudarchitectmusings.com/2013/06/24/openstack-for-vmware-admins-nova-compute-with-vsphere-part-1/  
翻译者：华为云计算工程师 [吴江](http://weibo.com/119329500?topnav=1&wvr=5&topsug=1)  
团队：Huawei OpenStack Team

-------

在与客户交流中，被问到很多关于OpenStack与VMware比较的问题。如何集成如何选择？如果商业用户选择OpenStack，是否意味着他们会废弃vSphere？这里，我来谈下VMware集成到OpenStack的问题。

## 一、架构描述

VMware之前已经有接入OpenStack的方案。但之前对接ESXi的代码，不是VMware而是Citrix贡献的，有很多功能限制不易维护；现在VMware成为董事后已经主动开始维护了。  
我认为Sphere和OpenStack的深度融合，对于那些之前已使用了VMware系统，但现在又想将应用迁移到OpenStack上的用户来说，是非常重要的。

### 1.1 OpenStack 整体架构

OpenStack采用一种无共享的、基于消息队列的架构，解耦的各模块组合在一起构成了一个统一的IaaS云。架构大家都熟，这里不冗述，需要的请参见[这里](http://www.slideshare.net/kenhui65/getting-started-with-open-stack?ref=http://cloudarchitectmusings.com/2013/06/16/getting-started-with-openstack/)。  
![image001.jpg](/images/blog/openstack-vsphere/image001.jpg)

### 1.2	Nova-Compute

Nova-Compute是Nova中的一个组件，主要负责VM的生命周期管理。与OpenStack中大多数模块相同，Nova也可以部署到单主机或多主机上，各部件的功能这里也不详述，详情参考[这里](http://docs.openstack.org/grizzly/openstack-compute/admin/content/images-and-instances.html)  
![image003.png](/images/blog/openstack-vsphere/image003.png)

很多人都直接将OpenStack与vSphere进行对比，但这样是错误的。从架构来看，OpenStack所处的位置实际上是位于hypervisor层之上的。通常说的OpenStack或者准确点说Nova，实际上类同于VMware中vCloud Director（vCD）+ vCloud Automation Center（vAC），而不是ESXi或者vCenter。事实上很重要的一点是，Nova是不包含hypervisor的，它是通过API+Driver来组织/管理多个hypervisor的，例如KVM、ESXi。目前支持的hypervisor，包括KVM、vSphere、Xen等，详细支持列表可以到[这里](https://wiki.openstack.org/wiki/HypervisorSupportMatrix)查看。  
![image005.png](/images/blog/openstack-vsphere/image005.png)

### 1.3	vSphere与Nova进行集成

目前OpenStack Nova可通过以下两个驱动来支持vSphere 4.1或更高版本：  
- VMwareESXDriver：最早由Citrix提供，后续由VMware更新，允许Nova通过vSphere SDK直接与ESXi主机进行通信。  
- VMwareVCDriver：由VMware在Grizzly版本中加入，允许Nova与管理ESXi主机集群的vCenter进行通信。  
请注意，本文着眼于Nova，暂不讨论存储与网络相关的内容。

### 1.4	EXSi与Nova的集成（VMwareESXDriver）

![image007.png](/images/blog/openstack-vsphere/image007.png)  
逻辑上看，Nova-Compute服务直接与ESXi主机通信。由于未引入vCenter，因此vMotion、HA、DRS等高级功能都不支持：  
- 不像基于内核的hypervisor，VM是跑在ESXi服务器上的，而不是默认的计算节点上。  
- 尽管OpenStack已支持多hypervisor，但每一个计算节点仅能支持一种hypervisor。因此，在multi-hypervisor的OpenStack混合云下，对于每一种hypervisor类型就至少要有一个计算节点来对应。  
- 当前ESXDriver有限制，每个Nova-Compute服务仅能支持一个ESXi主机。

![image009.png](/images/blog/openstack-vsphere/image009.png)

### 1.5	EXSi与Nova的集成（VMwareVCDriver）

逻辑上看，OpenStack与vCenter直接通信，VC管理整个ESXi集群，vMotion、HA、DRS也都能用了。但vCenter从Nova-Compute中抽离出了ESXi主机，Nova-Scheduler将整个自管理的集群作为一个独立主机接入——这会导致一些问题，如在多计算节点环境中，如何正确调度/部署这些VM；这点我会在后面介绍到。大致特点如下：

- 不同于基于内核的hypervisor，vSphere需要一个单独的vCenter主机，VM是运行在ESXi主机上而不是计算节点上。  
- 尽管OpenStack已支持多hypervisor，但一个计算节点同时只能支持一种hypervisor。因此，multi-hypervisor的OpenStack混合云，对于每一种hypervisor类型就至少要有一个计算节点来对应。  
- VCDriver有限制，每个Nova-Compute服务仅能对应一个vSphere集群。当计算节点作为VM运行在同一集群下时，就具有了HA能力。  
- VCDriver中每个cluster都要有一个datastore来进行配置和使用.

![image011.png](/images/blog/openstack-vsphere/image011.png)

## 二、Nova-Scheduler & DRS

### 2.1	vSphere与Nova Compute中的资源调度

上一部分已经给出了vSphere如何集成入OpenStack Nova的概述，在展开阐述之前，我认为有必要先介绍下Nova中的资源调度即Nova-Scheduler，以及vSphere的DRS功能。我会展示DRS与Nova-Scheduler的区别，以及两者如何配合的问题。

- 对于DRS，请直接戳[这里](http://www.amazon.com/vSphere-Clustering-Technical-Deepdive-ebook/dp/B005C1SARM/ref=tmm_kin_title_0)；  
- 对于Nova-Scheduler，[这里](https://www.ibm.com/developerworks/community/blogs/e93514d3-c4f0-4aa0-8844-497f370090f5/entry/openstack_nova_scheduler_and_its_algorithm27?lang=en)有一篇非常好的文章。

### 2.2	OpenStack Nova-Scheduler

![image013.png](/images/blog/openstack-vsphere/image013.png)

Nova使用Nova-Scheduler服务来确定VM需要创建到哪个节点上。Nova-Scheduler，同DRS一样，通过一些具体参数、policy、亲和性规则，来自动选择VM初始创建的位置。但是，Nova-Scheduler在VM创建之后，是不会再进行周期性LB+电源管理的。你可以在nova.conf中找到很多配置项。

> scheduler_driver=nova.scheduler.multi.MultiScheduler
compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
scheduler_available_filters=nova.scheduler.filters.all_filters
scheduler_default_filters=AvailabilityZoneFilter,RamFilter,ComputeFilter
least_cost_functions=nova.scheduler.least_cost.compute_fill_first_cost_fn
compute_fill_first_cost_fn_weight=-1.0

这里有两个关键参数：`schedule_default_filters`和`Least_cost_functions`(其实这个参数在最新版本里已经被deprecated)，它们反映出在确定主机时Filter Scheduler的两个计算过程（还有另两个调度算法分别是Chance Scheduler和Multi Scheduler，但是Filter Scheduler是默认的调度算法并且是适用于绝大多数场景的）。这两个参数一起作用，在创建VM时对整个计算节点下的资源做了负载均衡，和ESXi集群中DRS所使用的Dynamic Entitlements + Resource Allocation Settings 所完成的工作一样。

关于Nova-Scheduler，这里不做过多分析，请参见[孔令贤的博客](http://blog.csdn.net/lynn_kong)。

### 2.3	DRS 与 Filter Scheduler

在阐述了Nova如何调度资源后，我们来与DRS做个对比，然后讨论下这两者如何一起工作。注意，这里我不再讨论Nova与单独ESXi主机如何进行调度，因为这和其他hypervisor是一样的。我这里主要说下Nova+VCDriver，如何与vCenter管理的ESXi主机进行配合。

![image019.png](/images/blog/openstack-vsphere/image019.png)

在之前文章中，已经提到了vCenter将ESXi主机与Nova-Compute服务抽离开了，Nova-Scheduler将整个集群当做一个单独的主机，主机的资源由下面ESXi节点共同组成。这会带来影响：  
Nova-Scheduler一旦选定了一个vSphere集群后，就不在参与集群中具体的选主机工作了；Nova-Compute只是简单的调用vCenter的API。后续vCenter会根据Dynamic Entitlements和Resource Allocation settings来选择一个具体的ESXi主机，并自动进行LB和DRS，这个过程Nova是不感知的。

在上图示例中，节点1有8GB RAM剩余，而节点2看上去有12G RAM。Nova-Scheduler无法区分节点2上的12GB 加和RAM，是否是真实可用的RAM资源。

因此这里就会存在问题，让我们来分析一下两个应用场景下资源的分配情况。环境采用上例中假设的，并且允许pRAM 50%超分配：  
- 用户请求一个10GB RAM的VM。  
Nova-Scheduler根据RamFilter，经过筛选后会认为vSphere计算节点是资源充足的，即使每一个ESXi主机上都没有足够的RAM。一旦vSphere被选出来创建VM，则vCenter会根据之前的DRS规则来判断是否有足够的资源。  
- 用户请求一个4GB RAM的VM。  
Nova-Scheduler根据RamFilter，筛选后会认为这两个节点都满足要求，然后通过权重算法最终倾向于选择RAM更充足的vSphere计算节点来创建。可以看出，当hypervisor/计算节点的RAM资源不充足时，它可能会在创建时被错误的分配到一个cost数值，导致系统变得不再平衡。

所以，如何克服上面的问题？不幸的是，VCDriver刚加入OpenStack，目前并没有好的文档或相关实践数据来解决。

## 三、HA & 迁移

### 3.1	High Availability

HA是vSphere重要的特性之一，它在计算层上提供了弹性，使得源主机故障时VM会自动转移到其他节点上去。这对于那些主机应用本身无法提供应用层HA而仅依赖于基础设置提供保证时，显得尤为重要。因此，这是很多企业在切换hypervisor（如KVM）时必须考虑的特性。所以，客户对于原生OpenStack本身并不具备HA特性，都非常惊讶。

为了代替vSphere HA，当计算节点故障时，OpenStack采用一个被称为“虚拟机疏散（Instance Evacuation）”的功能来进行恢复。请注意，与vSphere不同，Nova的这个计算节点，是同时具备hypervisor以及hypervisor管理节点两个身份的。虚拟机疏散是一个手动的将VM冷迁移到新节点去的过程。

下面是疏散的操作步骤（需管理员身份）：  
- 使用nova host-list查看所有计算节点；  
- 选择一个合适的节点（如果user data也要保留，则目标节点必须具备能够接入与故障节点同一个共享存储的能力）  
- 当调用nova evacuate命令后，Nova读取DB中保存的配置数据，再将VM在选定节点上“rebuild”出来。  
- 恢复后的VM是否与之前一致，取决于**是否部署在共享存储上**：  
1. 共享存储：user data将得到保留，VM密码不变；  
2. 非共享存储：使用原有数据新创一个VM，主机名、ID、IP、UID等保持不变，不过user data不会保留，VM密码也会重新生成。

![image021.png](/images/blog/openstack-vsphere/image021.png)

前文已经提到过，vCenter会管理整个ESXi集群并且将所有ESXi节点与Nova计算节点分离开。当ESXi主机故障时，vSphere HA将被激活并且将所有VM在其他节点上重启，DRS将会保证这些VM的分配平衡。Nova是不感知这些VM移动的，只是vCenter上报的可用资源总数会变少，然后被Nova记录下来，并在下次创建VM时起作用。

![image023.png](/images/blog/openstack-vsphere/image023.png)

这里可能会有读者（尤其是有VMware背景的）问——为什么OpenStack会缺少如此基本的HA特性？我自己认为，有以下几点原因：  
1. OpenStack本身并没有包含特有的hypervisor，它选择将这个能力通过各个hypervisor进行暴露。举例来说，当OpenStack搭配Hyper-V与vSphere时都可以提供HA能力，因为这两者原生就支持HA。  
2. OpenStack最初是作为一个云平台来设计的，它遵循的原则异于传统的虚拟化基础设施。两者的区别，可以从[这篇](http://it20.info/2012/12/vcloud-openstack-pets-and-cattle/)文章中找到答案。

一些“Cloud vs Virtualized Infrastructure”的原理，包含如下内容：  
1. VM和计算节点是提供服务的商品的一部分。如果一个VM或计算节点故障了，干掉再重启个就可以了；  
2. 弹性是多层的，并且要求在应用层+基础设施层都要考虑进行实现。 
3. 横向扩展（Scale-out）比纵向扩展（Scale-up）更好，不仅是因为性能，而且涉及到弹性。通过在多个实例和计算节点之间分配工作负载，将故障实例的影响最小化，为疏散VM争取时间。

值得注意的是，一些厂商如Piston，已经将HA能力集成到他们的OpenStack发行版中了。另外，下一个版本计划推出的新项目Heat，可能就会将HA正式引入OpenStack。

另一方面，对于那些需要应用层弹性支持的既有应用 与OpenStack搭配是否合适，企业可能仍旧存在疑问；而这正是vSphere具备优势的地方。搭建一个包含vSphere的OpenStack混合云，用户可以将新应用部署在KVM或Xen上，而那些需要vSphere HA功能的既有应用，部署到vSphere上。

![image025.png](/images/blog/openstack-vsphere/image025.png)

### 3.2	VM迁移

OpenStack通过它管理下每个hypervisor的原生功能，来实现VM迁移。下面展示了各个hypervisor对迁移的支持情况：

![image027.png](/images/blog/openstack-vsphere/image027.png)

vSphere与表中其他hypervisor最大的差异，就是冷迁移和vMotion不能通过Nova来触发，它只能由vSphere API或CLI来调用。可见，除了vSphere HA功能外，上表其他OpenStack支持的hypervisor与其它相比时都大体类似，而HA和Storage vMotion都为vSphere带来了很大的优势。所以，在OpenStack下个发行版中，无论是其他hypervisor开发的匹配vSphere的新特性，还是VMware如何发展使vSphere成为OpenStack最好的解决方案，都是很值得期待的。

<全文完>