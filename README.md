# try-to-vpn

记录 Azure VPN 搭建与测试过程。

这是一份中文实作教程，介绍如何使用 **Azure for Students** 的云资源额度，在 **Ubuntu 24.04** 虚拟机上搭建 **WireGuard VPN**，并用手机验证 IPv4 出口。案例使用 Azure 的 **Malaysia West** 区域和 **B2ats_v2** 实例。

**[阅读完整搭建教程](docs/vpn-setup.md)**

教程覆盖学生资格与余额查询、虚拟机与端口设置、WireGuard 服务端和客户端配置、连通性排查，以及算力、磁盘、公网 IP 和出站流量的计费。

Azure 学生优惠的官方初始额度是 **US$100**，有效期为 12 个月。额度用来抵扣符合条件的 Azure 资源费。规格是否可用、费用怎样抵扣，以自己的订阅和账单为准。 [微软优惠条款](https://azure.microsoft.com/zh-cn/pricing/offers/ms-azr-0170p/)
