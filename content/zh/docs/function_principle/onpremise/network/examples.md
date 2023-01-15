---
title: "典型网络配置"
weight: 90
description: >
  本文介绍几种典型的私有云网络配置
---

## （单网口经典网络）计算节点单个网口，虚拟机使用多VLAN经典网络

这种场景下，每个计算节点只有一个网口(可以为Bonding)，既做管理口，也做业务口。虚拟机之间的业务流量是由宿主机对接的三层交换机转发。

### VPC配置

采用*Default* VPC

### 二层网络配置

在*Default* VPC新建一个二层网络*bcast0*

### IP子网配置

在二层网络*bcast0*创建至少2个IP子网，其中一个IP子网给宿主机管理口使用，需要包含宿主机的管理口IP。另外一个子网给虚拟机用。可以配置更多的IP子网给虚拟机使用，并且这些虚拟机用IP子网可以配置不同的VLAN ID（非1的VLAN ID）。这些IP子网的网关以及对应的VLAN都需要配置在物理机连接的三层交换机上。

### 宿主机配置

宿主机的网口在交换机上需要配置为trunk模式，宿主机管理口*必须*采用默认VLAN ID。一般来说，默认VLAN ID 为 1。但目前交换机也支持为TRUNK口设置非1的VLAN ID，具体请参考交换机的配置手册。

宿主机host.conf的networks配置：

```yaml
networks:
- <host_ifname>/br0/<host_management_ip>
```

例如:

```yaml
networks:
- eth0/br0/10.168.20.2
```

#### 宿主机管理口使用非1的VLAN ID，并且交换机不支持设置TRUNK口的默认VLAN ID，这种情况如何配置？

这种情况需要为宿主机设置一个VLAN子接口，宿主机使用这个VLAN子接口作为管理口，并且虚拟机交换机需要桥接到主接口上。下面以宿主机主接口为bond0，宿主机管理口的VLAN ID为3001为示例说明：

1、在宿主机配置VLAN子接口 bond0.3001，配置如下：

```
# /etc/sysconfig/network-scripts/ifcfg-bond0.3001
VLAN=yes
NAME=bond0.3001
DEVICE=bond0.3001
IPADDR=<management_ip>
PREFIX=<prefix>
DEFROUTE=yes
ONBOOT=yes
```

2. 修改/etc/yunion/host.conf，设置如下:

```yaml
listen_interface: bond0.3001
networks:
- bond0/br0/bcast0
```

需要注意的是，这种配置下，虚拟机将不能使用和宿主机相同的VLAN ID（这里是3001)。

### EIP配置

经典网络模式下，外部能直接访问虚拟机IP，无需配置EIP


## (单网口VPC网络) 计算节点单个网口，虚拟机使用VPC网络

这种场景下，每个计算节点宿主机只有一个网口做管理口。管理口也承载业务流量，但是虚拟机之间的业务流量经过ovn的geneve隧道封装。

### 宿主机网络配置

宿主机的管理口依然接入经典网络。

需要在*Default* VPC下配置二层网络*bcast0*，并在该二层网络下配置宿主机管理口IP所用的IP子网。

宿主机的 host.conf networks配置同*单网口经典网络*模式

例如，宿主机IP子网为 10.128.26.2。则宿主机的配置也是：

```yaml
networks:
- eth0/br0/10.128.26.2
```

### 虚拟机网络配置

虚拟机的网络为VPC管理，用户可以任意创建VPC，并在VPC内分配任意的IP子网。不同VPC之间的IP子网之间相互隔离，无法访问。相同VPC内的IP子网之间无隔离。

### EIP配置

这种模式下，需要配置EIP，用于云平台外访问VPC内部的虚拟机。

最简单的EIP配置是选择一台宿主机，修改/etc/yunion/host.conf，将 sdn_enable_eip_man 设置true。更具体配置方式请参见文档。使用ALL IN ONE模式安装的第一台宿主机自动开启了 sdn_enable_eip_man。

#### EIP网关的路由配置

为了使用EIP，还需要在平台bcast0下增加一条用于EIP分配的IP网段（注意EIP网段应该避免和宿主机的管理IP在同一个网段，也就是网络地址需要不一样，例如宿主机的管理网口是 10.168.22.2/24，则EIP网段是 10.168.23.0/24），在交换机上添加EIP网段的静态路由，NextHop设置为 ALL IN ONE 部署的第一台宿主机的管理口IP。

## (双网卡经典网络) 计算节点两个网口，虚拟机使用多VLAN经典网络

这种场景下，每个计算节点有两个网口，一个做管理口，另外一个做业务口。虚拟机之间的业务流量是由宿主机业务口对接的三层交换机转发。但也有部分虚拟机需要对接管理口。

### VPC配置

采用Default VPC

### 二层网络配置

在Default VPC新建两个二层网络：bcast0（管理）和bcast1（业务）

### IP子网配置

在bcast0上创建至少两个IP子网，其中一个IP子网给宿主机管理口使用，需要包含宿主机的管理口IP。另外一个子网给有对接管理网需求的虚拟机使用。

在bcast1上创建1个或者多个IP子网给虚拟机使用，并且这些虚拟机用IP子网可以配置不同的VLAN ID（非1的VLAN ID）。这些IP子网的网关和对应VLAN都需要配置在物理机连接的对应三层交换机上。

### 宿主机配置

宿主机的管理口需配置管理IP，该管理口在交换机上为Access模式。

宿主机的业务口不需要配置IP，但业务网口在交换机上需要配置为trunk模式。

宿主机 /etc/yunion/host.conf 中需要有两条记录，一条给管理口，一条给业务口

```yaml
networks:
- <host_ifname0>/br0/<host_management_ip>
- <host_ifname1>/br1/bcast1
```

例如:

```yaml
networks:
- eth0/br0/10.168.20.2
- eth1/br1/bcast1
```

### EIP配置

这种模式下，外部能直接访问虚拟机IP，无需配置EIP

## (双网口VPC网络) 计算节点两个网口，虚拟机使用VPC网络

这种场景下，每个计算节点宿主机有两个网口，其中一个管理口，承载管理流量。另外一个是业务口，承载业务流量。虚拟机之间的业务流量经过ovn的geneve隧道封装，通过宿主机的业务口转发。

### 宿主机网络配置

同理，宿主机的管理口采用经典网络。

需要在*Default* VPC下配置二层网络*bcast0*，并在该二层网络下配置宿主机管理口所用的IP子网。

例如，宿主机IP子网为 10.128.26.2。则宿主机 /etc/yunion/host.conf 的配置是：

```yaml
networks:
- eth0/br0/10.128.26.2
```

同时，宿主机的业务口也需要配置IP，且该业务口IP和管理口IP不应该在一个二层网络。

为了确保虚拟机之前的ovn流量经过业务口，需要配置 /etc/yunion/host.conf 的 ovn_encap_ip 为宿主机业务口的IP。

例如，宿主机的管理口IP为 10.128.26.2/24，业务口IP为 10.129.26.2/24，则 /etc/yunion/host.conf 配置为：

```yaml
ovn_encap_ip: 10.129.26.2
networks:
- eth0/br0/10.128.26.2
- eth1/br1/bcast1
```

配置后，虚拟机之前的VPC网络流量都会封装为源和目的IP地址为业务网段的geneve流量。网络上只需要确保业务口之前网络互通，宿主机的业务口的VLAN模式无论是TRUNK还是Access都不影响。

### 虚拟机网络配置

同理，这种模式下，虚拟机的网络为VPC管理，用户可以任意创建VPC，并在VPC内分配任意的IP子网。不同VPC之间的IP子网之间相互隔离，无法访问。相同VPC内的IP子网之间无隔离。由于宿主机配置了 ovn_encap_ip 为宿主机的业务口IP，所以虚拟机之前的ovn流量都是通过宿主机的业务口转发。

### EIP配置 

VPC网络模式下，需要配置EIP，用于外部访问指定的内部虚拟机。同理，也需要配置EIP网关和EIP网段。

#### EIP网关的路由配置

和*单网口VPC网络*不同的地方是，为了确保EIP流量也要经过业务口，需要确保如下两个配置：

* 在交换机上配置EIP网段的NextHop时，需要将EIP网关所在节点的业务网口IP作为NextHop IP。(确保进入EIP的流量走业务口)
* 在EIP网关所在节点，需要将业务口设置为缺省路由出口。(确保流出EIP的流量走业务口)
