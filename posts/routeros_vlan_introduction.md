# RouterOS Vlan Introduction

Vlan 可以把一个物理网络划分为多个虚拟局域网,他们之间互相隔离,可以通过路由策略来限制他们之间的访问.比如在家庭网络环境中,我们可以使用 Vlan 来隔离一些设备. 某些无良厂商的摄像头会偷偷的上传用户的信息上传他们自己的服务中, 我们可以把这种设备放到一个 Vlan 中, 然后通过防火墙策略限制这个 Vlan 访问外网. 在比如 Wifi 设置中我们可以设置多个无线信号, 对于访客可以专门设置一个 guest 网络, guest 网络单独对应一个 Vlan, 然后配置这个 Vlan 只能访问外网而不能访问家里的其他设备,比如 NAS, 而且还可以限制 guest 网络的带宽. 上面这些例子是 Vlan 在家庭网络中的一些应用. 下面介绍 Vlan 在 ROS 中具体配置过程.

## Bridge 基础设置
首先需要明确一点,在 ROS 中任何功能都是通过软件实现的,如果一些设备具有专门的芯片,那么 ROS 会利用其中的一些特性来实现 ROS 软件的功能,简单的来说就是硬件加速. 所以同样的功能在不同的设备上可能会有不一样的效率.
在 ROS 中 Vlan 是基于 bridge 的, 所以我们先讲以下 bridge.
bridge 是 ROS 中一个软件功能, 主要完成数据包的转化, bridge 工作在二层. 因为二层交换是一个特别基础的功能,用软件来实现的话效率会特别低,所以 Mikrotik 家的设备大部分都配有一颗专门的交换芯片,以此来达到二层的硬件加速.在 ROS 中, 这叫 [Bridge Hardware Offloading](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-BridgeHardwareOffloading)

下面介绍一些简单命令:

**创建桥**
```shell=
/interface/bridge> add name=bridge1
```



**分配端口到 bridge1 上**
```shell=
/interface/bridge/port> add bridge=bridge1 interface=sfp-sfpplus1
/interface/bridge/port> add bridge=bridge1 interface=sfp-sfpplus2
```

查看端口信息
```shell=
/interface/bridge/port> print
```

> Flags: I - INACTIVE; H - HW-OFFLOAD
>     INTERFACE     BRIDGE   HW   PVID  PRIORITY  HORIZON
> 0 IH sfp-sfpplus1  bridge1  yes     1  0x80      none   
> 1 IH sfp-sfpplus2  bridge1  yes     1  0x80      none  

我们可以看到连个端口都带有 `H` flag, 表明这是硬件加速的端口.
到这里我们成功创建了一个具有两个端口的 bridge, 用户可以通过网线连接这两个端口实现两个设备间的网络通信(一个简单的交换机).

### 查看 bridge 识别的 host mac
```shell=
/interface bridge host print
```

> Flags: X - disabled, I - invalid, D - dynamic, L - local, E - external 
>          MAC-ADDRESS        VID ON-INTERFACE            BRIDGE
>  0   D   B8:69:F4:C9:EE:D7      ether1                  bridge1
>  1   D   B8:69:F4:C9:EE:D8      ether2                  bridge1
>  2   DL  CC:2D:E0:E4:B3:38      bridge1                 bridge1
>  3   DL  CC:2D:E0:E4:B3:39      ether2                  bridge1

带 `L` flag 表示端口本身的 MAC (local). 带 `E` 的表示学习到外部 MAC 地址.

**需要注意的是 ROS 中对 bridge 的使用有一些限制**:
- 大部分设备只有一个交换芯片, 并且在这个芯片上只能创建一个硬件加速的 bridge, 你可以创建多个 bridge, 但是只会有一个是硬件加速的. 这里有一个具体的[案例](https://help.mikrotik.com/docs/display/ROS/Layer2+misconfiguration#Layer2misconfiguration-Bridgesonasingleswitchchip)
- 有一些端口可能是管理端口, 并不是交换芯片上引出的,这种端口不能用于硬件加速
- 有些设备具有多个交换芯片, 创建 bridge 的时候不能混口不同芯片的上的端口.

Mikrotik 产品页面上会有对应设备的 `Block Diagram`, 这个图会展示出设备的芯片信息以及对应的端口, 大家可以根据这个图来创建对应的 bridge.

我的建议是大家只买只有一个交换芯片的设备, 然后只创建一个 bridge.


## Vlan 基础设置
上面说道 Vlan 是基于 bridge 的, 在创建好 bridge 之后, 我们就可以用 Vlan 给这个 bridge 划分多个网络了. 在 ROS 中, 这叫做 [Bridge VLAN Filtering](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-BridgeVLANFiltering).

注意有的设备在开启 `Bridge VLAN Filtering` 之后会失去 bridge 硬件加速功能.
> Currently, CRS3xx, CRS5xx series switches, CCR2116, CCR2216 routers and RTL8367, 88E6393X, 88E6191X, 88E6190, MT7621 and MT7531 switch chips (since RouterOS v7) are capable of using bridge VLAN filtering and hardware offloading at the same time, other devices will not be able to use the benefits of a built-in switch chip when bridge VLAN filtering is enabled. Other devices should be configured according to the method described in the Basic VLAN switching guide. If an improper configuration method is used, your device can cause throughput issues in your network.

可以参考[这个表](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-BridgeHardwareOffloading)查看芯片支持的硬件功能

ROS 中 Vlan 的配置主要在相面两个地方
- `/interface bridge port` 主要用来配置端口的 VID, 设置 ingress 规则
- `/interface bridge vlan` 配置 vlan table, 设置 egress 规则

### vlan table
vlan table 概念上有3个属性:
- vlan id
- tagged ports: 端口数组
- untagged ports: 端口数组

前面说过 vlan table 是用于配置 egress 规则, 当一个带有 vid 的包经过某个端口:
```
vid in (tagged ports) => 放行该包,不修改 vlan tag, 这种端口一般是 trunk port
vid in (untagged ports) => 放行该包, 移除 vlan tag, 这种端口一般是 access port
不符合上面条件的包直接丢弃, 说明此端口和包不在一个 Vlan
```

### vlan port
vlan port 通过 `pvid`, `frame-types` 两个属性配置 ingress 规则
```
无 vlan tag, frame-types=admit-only-untagged-and-priority-tagged => 放行该包进入端口, 打上 vlan tag, vid=pvid, 
无 vlan tag, frame-types=admit-all => 放行该包进入端口, 打上 vlan tag, vid=pvid
有 vlan tag, frame-types=admit-only-vlan-tagged, 并且 vid in (tagged ports) => 放行该包进入端口, vlan tag 不变.
有 vlan tag, frame-types=admit-all, 并且 vid in (tagged ports) => 放行该包进入端口, vlan tag 不变.
不符合上面条件的包直接丢弃
```
frame-types 设置为 admit-all 可以同时让带 vlan tag 和不带 vlan tag 的包进入, 这种端口一般为 Hybrid Ports

vlan port 中一般来说一个 port 对应一个物理接口 (bonding 接口也可以用在vlan 中, 这里暂不讨论). 但是一个端口比较特殊,
那就是 cpu port, 这个接口主要用作管理端口, 同时也具备路由, 防火墙过滤等功能. 这个端口通过 `/interface bridge` 实现.
比如
```
/interface bridge set bridge1 frame-types=admit-only-vlan-tagged
```

## 使用 CRS504 配置 vlan 的一个 🌰

```shell=
# model = CRS504-4XQ
# CRS504 一个有 4 个物理接口,每个接口由4个 ethernet interface 组成
# 这里在每个物理接口(4各interface) 上创建一个 vlan, 通过给每个 vlan 分配 interface
# 并且分配 ip 地址来实现 inter vlan routing, vlan 结构如下:
# vlan name, vlanid, ip
# vlan100, 100, 10.1.1.1/24
# vlan200, 200, 10.1.2.1/24
# vlan300, 300, 10.1.3.1/24
# vlan400, 400, 10.1.4.1/24

:local mybridge "mybridge"


# 创建 bridge
/interface bridge
:put "create bridge: $($mybridge)"
add frame-types=admit-only-vlan-tagged name=$mybridge vlan-filtering=yes

# 给 bridge 创建 4 个 vlan
:foreach i in={1;2;3;4} do={
    :local vlanid ($i * 100)
    :local vname "vlan$($vlanid)"
    :put "vlanid=$($vlanid), vlanname=$($vname)"
    # 每个 vlan 由 4各端口组成
    :foreach j in={1;2;3;4} do= {
        :local inname ("qsfp28-$($i)-$($j)")
        :put ("add bridge port interface=$($inname) pvid=$($vlanid)")
        /interface bridge port
        add bridge=$mybridge frame-types=admit-only-untagged-and-priority-tagged interface=$inname pvid=$vlanid
    }
    :put "create bridge vlan"
    /interface bridge vlan
    add bridge=$mybridge tagged=$mybridge vlan-ids=$vlanid

    :put "create interface vlan"
    /interface vlan
    add interface=$mybridge name=$vname vlan-id=$vlanid mtu=$mymtu

    :put "assign ip to vlan"
    /ip/address
    add address="10.1.$($i).1/24" interface=$vname
}

```

## 更多🌰

- [Basic VLAN switching](https://help.mikrotik.com/docs/display/ROS/Basic+VLAN+switching)
- [VLAN Example - Trunk and Access Ports](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-VLANExample-TrunkandAccessPorts)
- [VLAN Example - Trunk and Hybrid Ports](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-VLANExample-TrunkandHybridPorts)
- [VLAN Example - InterVLAN Routing by Bridge](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-VLANExample-InterVLANRoutingbyBridge)

## 总结
我们首先讲解了 ROS 中 bridge 相关的概念, 然后在 bridge 的基础上讲解了 vlan 划分. Vlan 主要通过 vlan table 和
vlan port 来配置进出端口的规则. 最后使用 CRS504 做了一个简单的 Vlan 间路由的例子. 通过这个🌰可以发现配置起来其实不算
复杂, 但是其中的概念还算比较多, 文章的一个初衷是方便自己以后配置新的设备的时候能够快速的回忆起这些基础概念, 如果这篇文章
对于屏幕前的你也有一点点作用,那就再好不过了.最后一点是 ROS 中对于不同的硬件设备可能会有不同的配置,大家配置之前务必了解
清楚设备的 `block diagram`.


## 参考
- [Bridging and Switching](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-BridgeSettings)
- [Bridge VLAN Table](https://help.mikrotik.com/docs/display/ROS/Bridge+VLAN+Table)
- [Using RouterOS to VLAN your network](https://forum.mikrotik.com/viewtopic.php?f=13&t=143620)
- [Layer2 misconfiguration](https://help.mikrotik.com/docs/display/ROS/Layer2+misconfiguration#Layer2misconfiguration-Bridgesonasingleswitchchip)
