---
layout:     post
title:      "Packet Send to CPU"
subtitle:   " \"Packet from Switch Send to CPU\""
date:       2021-05-13 17:30:00
author:     "taylor"
header-style: "text"
catalog: true
tags:
    - switch
    - CPU
    - Boadcom
---

## OverView
![overview](https://user-images.githubusercontent.com/49511529/118103946-13852700-b40d-11eb-8bae-50e8c301183c.jpg)
## Kernel Network Interface(KNET Interface)
交换芯片收到的报文需要通过KNET Interface进入内核协议栈。  
交换机进行端口初始化的时候会根据/usr/share/sonic/device/……/port_config.ini 文件创建kernel network interface。  
![port_config](https://user-images.githubusercontent.com/49511529/118103972-1bdd6200-b40d-11eb-86b9-b1501216268a.png)   
具体的创建流程如下:  
![create_netif](https://user-images.githubusercontent.com/49511529/118103572-a5406480-b40c-11eb-8b9e-192ae50e1622.jpg)  
创建的KNET interface  
![netif](https://user-images.githubusercontent.com/49511529/118103802-e8023c80-b40c-11eb-9572-68a25173fc78.jpg)  
## CPU Management Inteface Controller(CMIC)
CMIC相当于交换芯片和CPU之间的一个网关，CPU可以通过CMIC模块去配置交换芯片的状态以及和交换芯片互相收发报文。  
![CMIC](https://user-images.githubusercontent.com/49511529/118103470-893cc300-b40c-11eb-8a0c-85f52a504891.jpg)  
## Direct Memory Access(DMA)
### DMA control block(DCB)
#### DCB
DMA通过DCB来接收和发送报文。  
DCB recive packet  
![dcb_recive](https://user-images.githubusercontent.com/49511529/118238314-f1ea7500-b4ca-11eb-99b4-7877b4406c0f.jpg)
#### DCB的作用
1. 当每个独立的DMA engine使能的时候，初始化DMA engine。
2. 在DMA transmit的方向，为DMA engine提供诸如报文在CPU内存的地址，报文的长度，报文的目的端口等信息。
3. 在DMA revice方向，为DMA提供报文应该存放的内存地址空间的指针，当DMA engine收到报文就把报文写到指针指向的地址，  
   然后重写DCB中报文长度等关键信息。
### Transmit Packet DMA
读取CPU的内存，从内存中取出报文后再通过CMIC把报文送到交换芯片，根据DCB中读取的metadata中的SRCPORT、DESTPORT等相关信息决定如何处理报文。
### Revice Packet DMA
收到从交换芯片发出的报文后，把报文内容写进CPU的内存中，写完数据并设置好相关寄存器后，驱动程序(linux_bcm_knet.ko)  
会根据一组过滤规则（匹配自己映射端口或者映射端口所在聚合端口、路由组的报文）将报文送到相应的KNET interface，  
KNET interface收到报文再将报文送到内核协议栈。
## Packet Process 
报文通过KNET interface被送到内核的协议栈后，在用户层可以创建协议相关的套接字，使用BSD socket相关的系统调用接口来接收处理报文。  
![netiftokernel](https://user-images.githubusercontent.com/49511529/118103911-08ca9200-b40d-11eb-8357-ca6b9eaf6006.jpg)  
## Broadcom Linux Driver
在opennsl-moudles.service中加载。  

```  
[Unit]
Description=Opennsl kernel modules init
After=local-fs.target
Before=syncd.service

[Service]
Type=oneshot
ExecStart=-/etc/init.d/opennsl-modules start
# Don't remove opennsl driver when stopping service. Because
# removing knet drivers takes ~30 seconds to delete netdevs.
# This delay cuts too deep into warm reboot time budget.
# We could skip this step because we don't expect stopping
# opennsl service in any context other than rebooting.
# ExecStop=-/etc/init.d/opennsl-modules stop
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
### linux-kernel-bde.ko
Device Enumerator module. This module abstracts the  
platform and system-specific aspects of the physical switch device configuration:  
o PCI config access  
o Register access  
o Interrupt handling  
o DMA memory management  
o Physical/Virtual address translation  
o Note: All the above operations are critical for the operation of SAI and  
requires platform-specific/OS integration knowledge.  
### linux-user-bde.ko
Support module for translating device access from the  
userspace to the kernel. This module is required to run SAI in user space. This  
module is dependent on linux-kernel-bde.ko.
### linux-bcm-knet.ko
In certain applications, it is desirable to have switch packets  
go in and out of the operating system’s network protocol stack. The KNET  
(Kernel Network) infrastructure has been implemented for Linux User Mode  
environment that takes over the DMA and interrupt functions of the  
Transmit/Receive functions. Packets received from the switch are forwarded to  
one or more Linux network devices interfaces based on a set of filter rules.  
Likewise, packets sent out via the Linux network interfaces are sent to the switch  
based on the VLAN/port association.
#### create KNET interfaces
 ```
 bcm_knet_netif_t netif;

 netif.type = BCM_KNET_NETIF_T_TX_LOCAL_PORT; 
 netif.flags = BCM_KNET_NETIF_F_KEEP_RX_TAG; 
 netif.valn = vlan; /* VLAN associated with this interface */
 netif.port = port; /* Egress port*/
 soc_strcpy(netif.name, name); /* Copy the base interface name */ 
 soc_memcpy(netif.mac_addr, 6, baseMac); /* Set Mac address associated with this interface */
 if(BCM_FAILURE(rv = bcm_knet_netif_create(unit, &netif)))
 {
     return rv;
 }

 /* Kernel network interface structure. */
typedef struct bcm_knet_netif_s {
    int id;                             /* Network interface ID. */
    int type;                           /* Network interface type
                                           (BCM_KNET_NETIF_T_XXX). */
    uint32 flags;                       /* BCM_KNET_NETIF_F_XXX flags. */
    bcm_mac_t mac_addr;                 /* MAC address associated with this
                                           network interface. */
    bcm_vlan_t vlan;                    /* VLAN ID associated with this
                                           interface. */
    bcm_port_t port;                    /* Local port associated with this
                                           interface. */
    bcm_cos_queue_t cosq;               /* Cos offset from base queue of port
                                           associated with this interface. */
    char name[BCM_KNET_NETIF_NAME_MAX]; /* Network interface name (assigned by
                                           kernel) */
    uint32 cb_user_data;                /* Netif user data for knet rx cb. */
} bcm_knet_netif_t;
 ```
![netif_ctrl](https://user-images.githubusercontent.com/49511529/118103847-f2bcd180-b40c-11eb-9b29-41801430cae9.jpg)
#### KNET Filters and DMA Channels
1.  可以通过bcm_rx_queue_channel_set(unit, queue, channel)设置不同的CPU队列使用不同的DMA channel。  
2.  每个KNET filter都可以通过设置bcm_knet_filter_t中priority和不同的DMA channel绑定。如果设置了优先级，默认情况下，  
    DMA channel 0 和 优先级为0的filter绑定，DMA channel 1 和 优先级为1的filter绑定，如果  
    优先级大于max_channel - 1，就会和所有的channel绑定，绑定了channel，其他的channel就不会再匹配  
    这个优先级的filter。
3.  通过设置bcm_knet驱动模块里面的 **“num_rx_prio”** 的值，可以改变优先级和DMA channel的关系，优先级0到**num_rx_prio** - 1的  
    filter和DMA channel 0绑定，优先级**num_rx_prio**到2***num_rx_prio** - 1的报文和DMA channel 1绑定……  
    **num_rx_prio**的默认值为1。 
    ```
    if (kf->priority < (num_rx_prio * sinfo->rx_chans)) {
        if (kf->priority < (num_rx_prio * chan) ||
            kf->priority >= (num_rx_prio * (chan + 1))) {
            match = 0;
        }
    }
    ```
4.  数据包进入DMA channel，根据不同的filter送往不同的KNET interface，一个数据包能匹配一个KNET interface绑定的所有filters  
    的情况下，才能进入该KNET interface。

#### create KNET filters
```
/*Add filter for packets from port 3*/
bcm_knet_filter_t filter3;

sal_strcpy(filter3.desc, "Port 3 Packets");
filter3.typpe = BCM_KNET_FILTER_T_RX_PKT;
filter3.priority = 3;
/* Send packet to network interface */
filter3.dest_type = BCM_KNET_DEST_T_NETIF;
filter3.dest_id = netif.id;
filter3.match_flags = BCM_KNET_FILTER_M_INGPORT;
filter3.m_ingport = 3;
BCM_IF_ERROR_RETURN(bcm_knet_filter_create(unit, &filter3));

/* Kernel packet filter structure. */
typedef struct bcm_knet_filter_s {
    int id;                             /* Filter ID. */
    int type;                           /* Filter type (BCM_KNET_FILTER_T_XXX). */
    uint32 flags;                       /* BCM_KNET_FILTER_F_XXX flags. */
    int priority;                       /* Filter priority, lower value has
                                           higher priority, value of RX channels
                                           are reserved to bind filter to
                                           corresponding RX channel by default. */
    int dest_type;                      /* Filter destination type. */
    int dest_id;                        /* Filter destination ID. */
    int dest_proto;                     /* If non-zero this value overrides the
                                           default protocol type when matching
                                           packet is passed to network stack. */
    int mirror_type;                    /* Mirror destination type. */
    int mirror_id;                      /* Mirror destination ID. */
    int mirror_proto;                   /* If non-zero this value overrides the
                                           default protocol type when matching
                                           packet is passed to network stack. */
    uint32 match_flags;                 /* BCM_KNET_FILTER_M_XXX flags. */
    char desc[BCM_KNET_FILTER_DESC_MAX]; /* Filter description (optional) */
    bcm_vlan_t m_vlan;                  /* VLAN ID to match. */
    bcm_port_t m_ingport;               /* Local ingress port to match. */
    int m_src_modport;                  /* Source module port to match. */
    int m_src_modid;                    /* Source module ID to match. */
    bcm_rx_reasons_t m_reason;          /* Copy-to-CPU reason to match. */
    int m_fp_rule;                      /* Filter processor rule to match. */
    int m_cpu_cos;                      /* CPU COS to match. */
    int m_mirrored;                     /* Match on packets mirrored to CPU. */
    int raw_size;                       /* Size of valid raw data and mask. */
    uint8 m_raw_data[BCM_KNET_FILTER_SIZE_MAX]; /* Raw data to match. */
    uint8 m_raw_mask[BCM_KNET_FILTER_SIZE_MAX]; /* Raw data mask for match. */
    uint32 cb_user_data;                /* Filter user data for knet rx cb. */
} bcm_knet_filter_t;
```
![filter_ctrl](https://user-images.githubusercontent.com/49511529/118103643-b8ebcb00-b40c-11eb-8c41-cf352dc59338.jpg)

#### KNET API
1. 初始化
bcm_knet_init()会清除所有存在的filters，同时重新创建默认的BCM filter，  
这个函数在SDK初始化的时候由bcm_init()调用。
```
extern int bcm_knet_init(int unit);
```
2. Clean up
清除所有filters，不会影响KNET interface。
```
extern int bcm_knet_cleanup(int unit);
```
3. 创建Kernel Network Interface
```
extern int bcm_knet_netif_create(int unit, bcm_knet_netif_t *netif);
```
4. 删除Kernel Network Interface
```
extern int bcm_knet_netif_destroy(int unit, int netif_id);
```
5. 获取Kernel Network Interface的配置信息
   通过network interfacce ID获取配置信息。
```
extern int bcm_knet_netif_get(int unit, int netif_id, bcm_knet_netif_t *netif);
```
6. 遍历Kernel Network Interface的对象
   遍历整个Kernel Network Interface列表条目，同时调用回调函数bcm_knet_netif_traverse_cb， 
   可以使用一个指向数据结构的指针，该指针可以传递给回调函数。
```
extern int bcm_knet_netif_traverse(int unit, bcm_knet_netif_traverse_cb trav_fn, void *user_data);
```
7. Kernel Network Interface遍历的回调
```
typedef int(*bcm_knet_netif_traverse_cb)(int unit, bcm_knet_netif_t *netif, void *user_data); 
```
8. 创建Kernel Packet Filter
```
extern int bcm_knet_filter_create(int unit, bcm_knet_filter_t *filter);
```
9.  删除Kernel Packet Filter
```
extern int bcm_knet_filter_destroy(int unit, bcm_knet_filter_t *filter);
```
10. 获取Kernel Packet Filter的配置信息
```
extern int bcm_knet_filter_get(int unit, int filter_id, bcm_knet_filter_t *filter);
```
11. 遍历Kernel Packet Filter对象
```
extern int bcm_knet_filter_traverse(int unit, bcm_knet_filter_traverse_cb trav_fn, void *user_data);
```
12. Kernel Packet Filter遍历的回调
```
typedef int(*bcm_knet_filter_traverse_cb)(int unit, bcm_knet_filter_t *filter, void *user_data);
```
#### Knet Packet Handling
![knet_packet_handling](https://user-images.githubusercontent.com/49511529/118103767-d9b42080-b40c-11eb-9d0e-9711c944c5be.jpg)

### linux-knet-cb.ko
This is an extension callback module for the knet module  
described above and is used to implement custom filtering and action on the  
packets based upon application requirements. Once the knet module has done its  
required processing on the packet it invokes this callback module along with the  
packet buffer and some user data for enabling any further custom processing



