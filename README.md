# 用 Azure 学生额度搭建 WireGuard VPN

记录 Azure VPN 搭建与测试过程。本文用一台普通的 Linux 虚拟机做 VPN 出口，手机通过 WireGuard 隧道连接。案例在 **Azure Malaysia West** 使用 **Ubuntu 24.04 LTS** 和 **B2ats_v2**，已在 Android 上验证 IPv4 出口。

**[从 Azure 学生认证开始，阅读完整教程](docs/vpn-setup.md)**

- **部署** · 领取 Azure for Students 额度，创建 Ubuntu VM，配置公网 IPv4 和 UDP 入站规则。
- **连接** · 配置 WireGuard，在手机上检查握手和出口 IP，定位连接故障。
- **费用** · 核对算力、磁盘、公网 IPv4 和出站流量，区分停机与释放资源。

Azure for Students 的官方初始额度为 **US$100**，在注册后的 12 个月内可用于符合条件的 Azure 资源。实例是否可选、免费时数是否抵扣，都以实际订阅和账单为准。 [微软优惠条款](https://azure.microsoft.com/zh-cn/pricing/offers/ms-azr-0170p/)

公开示例中的密钥和端点均为占位符。阅读教程时请在自己的设备上生成密钥。
