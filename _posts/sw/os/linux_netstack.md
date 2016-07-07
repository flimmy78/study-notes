<!-- MarkdownTOC -->

- 1. socket 和文件系统都位于 VFS 下一层，对 socket 的操作都要经过 VFS
- 2. netstack 初始化
    - 2.1. `sock_init\(\)`
    - 2.1. skb
        - \(skb 待补充\)
    - 2.2. sockfs 初始化
    - 2.3. 网络协议初始化
    - 2.4. network namespace subsystem

<!-- /MarkdownTOC -->




## 1. socket 和文件系统都位于 VFS 下一层，对 socket 的操作都要经过 VFS

![](./sockfs-vfs.jpg)

- linux 里面每个文件都有唯一的 inode ，inode 会大量使用，为了提高效率会对 inode 进行缓存；
- vfs 要调用具体的文件系统就需要知道每个文件系统的信息，这些信息都放在各自的超级块（super_block) 里，需要文件系统注册（`register_filesystem`）把自己挂到 VFS 的 `file_systems` 全局链表上，然后通过挂载（`kern_mount`）自己、将超级块告知 VFS 。

## 2. netstack 初始化

内核使用 init.h 中定义的初始化宏来进行，即将初始化函数放入特定的代码段去执行：

```
core_initcall(sock_init);
```

相关的宏和初始化函数还包括

```
core_initcall：      sock_init 
fs_initcall：        inet_init 
subsys_initcall：    net_dev_init 
device_initcall:     设备驱动初始化 
```

上面四种宏声明的函数是按顺序执行的。

### 2.1. `sock_init()`

```
static int __init sock_init(void)
{
    int err;
    /*
     *      Initialize the network sysctl infrastructure.
     */
    err = net_sysctl_init();
    if (err)
        goto out;

    /*
     *      Initialize skbuff SLAB cache
     */
    skb_init();

    /*
     *      Initialize the protocols module.
     */

    init_inodecache();

    err = register_filesystem(&sock_fs_type);
    if (err)
        goto out_fs;
    sock_mnt = kern_mount(&sock_fs_type);
    if (IS_ERR(sock_mnt)) {
        err = PTR_ERR(sock_mnt);
        goto out_mount;
    }

    /* The real protocol initialization is performed in later initcalls.
     */

#ifdef CONFIG_NETFILTER
    err = netfilter_init();
    if (err)
        goto out;
#endif

#ifdef CONFIG_NETWORK_PHY_TIMESTAMPING
    skb_timestamping_init();
#endif

out:
    return err;

out_mount:
    unregister_filesystem(&sock_fs_type);
out_fs:
    goto out;
}
```

`sock_init()` 可以分为 4 部分 ： 初始化网络的系统调用（`net_sysctl_init`）、初始化 skb 缓存(`skb_init`)、初始化 VFS 相关(`init_inodecache`、  `register_filesystem` 、 `kern_mount`)、初始化网络过滤模块（`netfilter_init`）。

### 2.1. skb
数据包在应用层称为 data，在 TCP 层称为 segment，在 IP 层称为 packet，在数据链路层称为 frame。 Linux 内核中 `sk_buff` 结构来存放数据。

1. sk_buff 结构体

```
struct sk_buff {
    /* These two members must be first. */
    struct sk_buff      *next;
    struct sk_buff      *prev;

    ktime_t         tstamp;

    struct sock     *sk;
    struct net_device   *dev;

    /*
     * This is the control buffer. It is free to use for every
     * layer. Please put your private variables there. If you
     * want to keep them across layers you have to do a skb_clone()
     * first. This is owned by whoever has the skb queued ATM.
     */
#ifdef CONFIG_AS_FASTPATH
    char            cb[96] __aligned(8);
#else
    char            cb[48] __aligned(8);
#endif
    unsigned long       _skb_refdst;
#ifdef CONFIG_XFRM
    struct  sec_path    *sp;
#endif
    unsigned int        len,
                data_len;
    __u16           mac_len,
                hdr_len;
    union {
        __wsum      csum;
        struct {
            __u16   csum_start;
            __u16   csum_offset;
        };
    };
    __u32           priority;
    kmemcheck_bitfield_begin(flags1);
    __u8            local_df:1,
                cloned:1,
                ip_summed:2,
                nohdr:1,
                nfctinfo:3;
    __u8            pkt_type:3,
                fclone:2,
                ipvs_property:1,
                peeked:1,
                nf_trace:1;
    kmemcheck_bitfield_end(flags1);
    __be16          protocol;

    void            (*destructor)(struct sk_buff *skb);
#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
    struct nf_conntrack *nfct;
#endif
#ifdef CONFIG_BRIDGE_NETFILTER
    struct nf_bridge_info   *nf_bridge;
#endif

    int         skb_iif;

    __u32           rxhash;

    __be16          vlan_proto;
    __u16           vlan_tci;

#ifdef CONFIG_NET_SCHED
    __u16           tc_index;   /* traffic control index */
#ifdef CONFIG_NET_CLS_ACT
    __u16           tc_verd;    /* traffic control verdict */
#endif
#endif

    __u16           queue_mapping;
    kmemcheck_bitfield_begin(flags2);
#ifdef CONFIG_IPV6_NDISC_NODETYPE
    __u8            ndisc_nodetype:2;
#endif
    __u8            pfmemalloc:1;
    __u8            ooo_okay:1;
    __u8            l4_rxhash:1;
    __u8            wifi_acked_valid:1;
    __u8            wifi_acked:1;
    __u8            no_fcs:1;
    __u8            head_frag:1;
    /* Encapsulation protocol and NIC drivers should use
     * this flag to indicate to each other if the skb contains
     * encapsulated packet or not and maybe use the inner packet
     * headers if needed
     */
    __u8            encapsulation:1;
    /* 6/8 bit hole (depending on ndisc_nodetype presence) */
    kmemcheck_bitfield_end(flags2);

#if defined CONFIG_NET_DMA || defined CONFIG_NET_RX_BUSY_POLL
    union {
        unsigned int    napi_id;
        dma_cookie_t    dma_cookie;
    };
#endif
#ifdef CONFIG_NETWORK_SECMARK
    __u32           secmark;
#endif
    union {
        __u32       mark;
        __u32       dropcount;
        __u32       reserved_tailroom;
    };

    __be16          inner_protocol;
    __u16           inner_transport_header;
    __u16           inner_network_header;
#if defined(CONFIG_GIANFAR) && defined(CONFIG_AS_FASTPATH)
    __u8            owner;
    struct sk_buff      *new_skb;
#endif
    __u16           inner_mac_header;
    __u16           transport_header;
    __u16           network_header;
    __u16           mac_header;
    /* These elements must be at the end, see alloc_skb() for details.  */
    sk_buff_data_t      tail;
    sk_buff_data_t      end;
    unsigned char       *head,
                *data;
    unsigned int        truesize;
    atomic_t        users;
};
```

其中几个主要的成员是 ： 

```
struct sk_buff      *next;      //sk_buff 是以链表组织起来的，需要知道前后两个 sk_buff 的位置
struct sk_buff      *prev;
struct net_device   *dev;       //数据报所属的网络设备
unsigned int        len,        //全部数据的长度
                data_len;       //当前 sk_buff 的分片数据长度
__be16          protocol;       //所属报文的协议类型
__u8            pkt_type:3;     //该数据包的类型
unsigned char   *data;          //保存的数据 
atomic_t        users;          //每引用或“克隆”一次 sk_buff 的时候，都自加 1
```

协议类型

宏                  |值         |说明 
:--                 |:--        |:--
ETH_P_802_2         |4          |真正的 802.2 LLC，当报文长度小于 1536 时 
ETH_P_LOOP          |0x0060     |以太网环回报文 
ETH_P_IP            ||0x0800    |IP 报文 
ETH_P_ARP           |0x0806     |ARP 报文 
BOND_ETH_P_LACPDU   |0x8809     |LACP 协议报文 
ETH_P_8021Q         |0x8100     |VLAN 报文 
ETH_P_MPLS_UC       |0x8847     |MPLS 单播报文 

数据包类型

宏                      |值      |说明 
:--                     |:--     |:--
PACKET_HOST             |0       |该报文的目的地是本机 
PACKET_BROADCAST        |1       |广播数据包，该 报文的目的地是所有主机
PACKET_MULTICAST        |2       |组播数据包 
PACKET_OTHERHOST        |3       |到其他主机的数据包，在 VLAN 接口接收数据时有用 
PACKET_OUTGOING         |4       |它不是“发送到外部主机的报文”，而是指接收的类型，这
种类型用在 AF_PACKET 的套接字上，这是 Linux 的扩展
PACKET_LOOPBACK         |5       |MC/BRD 的 loopback 帧（用户层不可见） 

2. `skb_init()`

`skb_init()` 创建了两个缓存 `skbuff_head_cache` 和 `skbuff_fclone_cache` ，协议栈中所使用到的所有的 sk_buff 结构都是从这两个后备高速缓存中分配出来的，两者的区别在于前者是以 `sizeof(struct sk_buff)` 为单位创建的，是用来存放单纯的 sk_buff ，后者是以 `2*sizeof(struct sk_buff)+sizeof(atomic_t)` 为单位创建的，这一对 sk_buff 是克隆的，即它们指向同一个数据缓冲区，引用计数值是 0，1 或 2， 表示这一对
中有几个 sk_buff 已被使用。

3. sk_buff 的使用

分配 `alloc_skb()`

销毁 `kfree_skb()`

**分包/组包** ： 

##### (skb 待补充)

### 2.2. sockfs 初始化

网络通信可以被看作对文件的操作，socket 也是一种文件。网络初始化首先就要初始化 网络文件系统（sockfs）。

第一步是初始化 inode 缓冲(`init_inodecache`)，为 sockfs 的 inode 分配一片高速缓存 ：

```
static int init_inodecache(void)
{
    sock_inode_cachep = kmem_cache_create("sock_inode_cache",
                          sizeof(struct socket_alloc),
                          0,
                          (SLAB_HWCACHE_ALIGN |
                           SLAB_RECLAIM_ACCOUNT |
                           SLAB_MEM_SPREAD),
                          init_once);
    if (sock_inode_cachep == NULL)
        return -ENOMEM;
    return 0;
}
```

接着注册 sockfs 这种文件系统类型到 VFS 并将 sockfs 注册到 **super_blocks** ：

```
...
init_inodecache();

err = register_filesystem(&sock_fs_type);

sock_mnt = kern_mount(&sock_fs_type);
...
```

这样以后创建 socket 就是在 sockfs 文件系统里创建一个特殊的文件，而文件系统的 `super_block` 里面有一个成员变量 `struct super_operations   *s_op;` 记录了文件系统支持的操作函数，而这些操作函数都是让 VFS 来调用的，这样一来 socket 的表现就更像一个普通文件，支持大部分操作接口比如 write 、read 、close 等。

### 2.3. 网络协议初始化

按照上文所述的协议栈初始化顺序， 网络文件系统初始化(`sock_init`) 结束之后就开始进入 网络协议初始化(由宏 `fs_initcall` 修饰的`inet_init`)。


```
static int __init inet_init(void)
{
...
    sysctl_local_reserved_ports = kzalloc(65536 / 8, GFP_KERNEL);
    if (!sysctl_local_reserved_ports)
        goto out;

    rc = proto_register(&tcp_prot, 1);
    if (rc)
        goto out_free_reserved_ports;

    rc = proto_register(&udp_prot, 1);
    if (rc)
        goto out_unregister_tcp_proto;

    rc = proto_register(&raw_prot, 1);
    if (rc)
        goto out_unregister_udp_proto;

    rc = proto_register(&ping_prot, 1);
    if (rc)
        goto out_unregister_raw_proto;

...
    /*
     *  Add all the base protocols.
     */

    if (inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) < 0)
        pr_crit("%s: Cannot add ICMP protocol\n", __func__);
    if (inet_add_protocol(&udp_protocol, IPPROTO_UDP) < 0)
        pr_crit("%s: Cannot add UDP protocol\n", __func__);
    if (inet_add_protocol(&tcp_protocol, IPPROTO_TCP) < 0)
        pr_crit("%s: Cannot add TCP protocol\n", __func__);
#ifdef CONFIG_IP_MULTICAST
    if (inet_add_protocol(&igmp_protocol, IPPROTO_IGMP) < 0)
        pr_crit("%s: Cannot add IGMP protocol\n", __func__);
#endif

    /* Register the socket-side information for inet_create. */
    for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
        INIT_LIST_HEAD(r);

    for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
        inet_register_protosw(q);

    /*
     *  Set the ARP module up
     */

    arp_init();

    /*
     *  Set the IP module up
     */

    ip_init();

    tcp_v4_init();

    /* Setup TCP slab cache for open requests. */
    tcp_init();

    /* Setup UDP memory threshold */
    udp_init();

    /* Add UDP-Lite (RFC 3828) */
    udplite4_register();

    ping_init();

    /*
     *  Set the ICMP layer up
     */

    if (icmp_init() < 0)
        panic("Failed to create the ICMP control socket.\n");

    /*
     *  Initialise the multicast router
     */
#if defined(CONFIG_IP_MROUTE)
    if (ip_mr_init())
        pr_crit("%s: Cannot init ipv4 mroute\n", __func__);
#endif
    /*
     *  Initialise per-cpu ipv4 mibs
     */

    if (init_ipv4_mibs())
        pr_crit("%s: Cannot init ipv4 mibs\n", __func__);

    ipv4_proc_init();

    ipfrag_init();

    dev_add_pack(&ip_packet_type);

    rc = 0;
...
}
```

`inet_init()` 主要工作就是注册各种网络协议（如 icmp 、 tcp 、 udp 等）和初始化基础功能模块（如 arp 、 ip 等）。

1. 协议初始化
  - ICMP
  - UDP
  - TCP
  - IGMP
2. 模块初始化
  - ARP
  - ipv4
  - tcp/udp/udplite ？
  - ping ？
  - icmp ？

### 2.4. network namespace subsystem


