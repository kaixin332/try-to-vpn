# Azure VPN 搭建记录

本文记录部署步骤、连通性测试和排错过程。案例使用 **Azure for Students → Ubuntu 24.04 虚拟机 → WireGuard → Android 客户端**。实测用的是 Malaysia West、`B2ats_v2`、一块标准系统盘和一个静态公网 IPv4。安卓客户端已验证能通过这台 VM 上网。这里的配置全是示例，请在自己的设备上生成密钥。

**开始前**，准备一个可用的 Azure 学生订阅、自己的 SSH 密钥和装有 WireGuard 的手机。先搭一台服务器、连接一台客户端，确认握手和出口后，再添加其他设备。

## 1. 领取 Azure 学生额度

1. 打开 [Azure for Students](https://azure.microsoft.com/zh-cn/free/students/)，按微软页面验证学生资格。这项 US$100 优惠面向满足微软条款的年满 18 岁的学生，按页面提示使用学校邮箱验证。注册这项学生优惠无需信用卡。
2. 登录 [Azure 门户](https://portal.azure.com/)，在「订阅」确认所选订阅名称是 **Azure for Students**，在「教育」查看余额和到期日期。
3. 微软公布的初始额度是 **US$100，12 个月内使用**。它用来支付符合条件的 Azure 资源费用。VPN 是在虚拟机上自行运行的 WireGuard，换成其他可用的 Azure 订阅或 Linux 云主机也能部署。GitHub 学生礼包也列出了 Azure 优惠，领取时仍以微软给你的订阅和条款为准。同一人限领一份 Azure for Students 订阅。若门户的币种标记和官方额度不一致，核对订阅和计费币种，再按实际账单估算。

[微软优惠条款](https://azure.microsoft.com/zh-cn/pricing/offers/ms-azr-0170p/) · [GitHub 学生礼包](https://education.github.com/pack)

## 2. 创建 Ubuntu 虚拟机

1. Azure 门户搜索「虚拟机」，选择「创建」。订阅选 **Azure for Students**，新建一个专用资源组，例如 `rg-wireguard-demo`。
2. 镜像选 **Ubuntu Server 24.04 LTS**。地区和规格按你账户的预检结果选择。本案例新加坡的低价规格当时不可用，**Malaysia West + B2ats_v2** 通过了预检。其他账户的可用性可能不同。
3. 磁盘可选 **Standard HDD**。网络需要一个公网 IPv4。本案例用静态地址，方便客户端持续使用同一个 Endpoint。创建前检查 VM、磁盘和公网 IP 的价格与学生订阅是否适用。
4. 在关联的 **网络安全组（NSG）** 添加入站规则，允许 **UDP 51820**。来源能固定时限定为自己的公网 IP。手机经常切换网络时可以使用 **Any**，由 WireGuard 密钥认证客户端。若用 SSH 安装，临时允许自己公网 IP 的 **TCP 22**、使用 SSH 密钥，完成后移除临时规则。下文的交互式编辑步骤按 **SSH 登录 Ubuntu** 编写。部署后可用 Azure **Run Command** 查看受支持 VM 的服务状态，无需重新开放 SSH。

[Azure Linux VM 创建指南](https://learn.microsoft.com/zh-cn/azure/virtual-machines/linux/quick-create-portal) · [NSG 指南](https://learn.microsoft.com/en-us/azure/virtual-network/tutorial-filter-network-traffic) · [Run Command 文档](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command)

## 3. 在 Ubuntu 上配置 WireGuard

以下命令适用于新建的 Ubuntu VM。示例使用 `10.66.66.0/24` 作为 VPN 网段。若与你现有网络重叠，在服务端和客户端一起换成另一段私有地址。

```bash
sudo apt update
sudo apt install -y wireguard
sudo install -d -m 700 /etc/wireguard
sudo sh -c 'umask 077; set -C; wg genkey > /etc/wireguard/server.key'
sudo sh -c 'wg pubkey < /etc/wireguard/server.key'
ip -4 route show default
```

上面的 `wg pubkey` 输出**服务端公钥**。最后一条命令输出的 `dev` 后面是出口网卡名。接着在手机的 [WireGuard App](https://www.wireguard.com/install/) 点 **＋ → 从头创建**，生成客户端密钥对，复制**客户端公钥**到服务器。手机端私钥留在 App 里。

开启内核 IPv4 转发，并在首次部署时创建权限为仅管理员可读写的配置文件：

```bash
printf '%s\n' 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-wireguard-forward.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard-forward.conf
sudo sh -c 'umask 077; set -C; : > /etc/wireguard/wg0.conf'
sudoedit /etc/wireguard/wg0.conf
```

`set -C` 防止覆盖已有文件。`wg0.conf` 已存在时，跳过创建命令，核对后再编辑。把下列内容写进 `/etc/wireguard/wg0.conf`，替换所有尖括号占位符。服务端私钥来自 `/etc/wireguard/server.key`，只在自己的服务器上查看和填写。`PostUp` 和 `PostDown` 分别在接口启停时添加、移除 IPv4 出站 NAT 规则。

```ini
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = <服务端私钥文件中的值>
PostUp = iptables -t nat -A POSTROUTING -s 10.66.66.0/24 -o <出口网卡> -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.66.66.0/24 -o <出口网卡> -j MASQUERADE

[Peer]
PublicKey = <客户端公钥>
AllowedIPs = 10.66.66.2/32
```

如果 VM 启用了 UFW，允许 **UDP 51820** 和 **从 `wg0` 到出口网卡**的转发。把命令中的 `eth0` 换成刚才查到的网卡名。其他防火墙需要配置等效规则。NAT 规则负责改写源地址，转发权限由防火墙控制。

```bash
sudo ufw allow 51820/udp
sudo ufw route allow in on wg0 out on eth0 from 10.66.66.0/24 to any
```

```bash
sudo systemctl enable --now wg-quick@wg0
sudo systemctl status wg-quick@wg0 --no-pager
sudo wg show wg0
```

[Ubuntu WireGuard 网关配置](https://ubuntu.com/server/docs/how-to/wireguard-vpn/vpn-as-the-default-gateway/) · [Ubuntu 防火墙文档](https://ubuntu.com/server/docs/how-to/security/firewalls/)

## 4. 连接手机并验证

回到手机刚创建的隧道，保留 App 生成的私钥，按下表填其他字段。

| 位置 | 字段 | 值 |
| --- | --- | --- |
| 接口 | Addresses | `10.66.66.2/32` |
| 接口 | DNS servers | 可用的 DNS，例如 `1.1.1.1` |
| 对端 | Public key | 服务端公钥 |
| 对端 | Endpoint | Azure VM 的公网 IPv4 加 `:51820` |
| 对端 | Allowed IPs | `0.0.0.0/0` |
| 对端 | Persistent keepalive | `25` 秒，可用于手机在 NAT 后保持映射 |

1. 先关闭手机上正在运行的其他 VPN，再启用这个隧道。用浏览器打开网页产生流量，检查 App 或服务器 `sudo wg show wg0` 是否出现 **latest handshake**。
2. 打开 [IPv4 检查页](https://api.ipify.org/)，确认显示的是 VM 的公网 IPv4。这份配置只覆盖 **IPv4**。如果客户端有 IPv6，另行检查 IPv6 出口；需要 IPv6 也走隧道时，应配置端到端的 IPv6 路由。
3. **没有握手**时，先核对 Endpoint 的 IP 和端口，然后检查 VM、NSG 和主机防火墙，最后核对双方公钥。**有握手但网页打不开**时，先用 `sysctl net.ipv4.ip_forward` 确认值为 `1`，再检查 NAT 出口网卡、转发规则和 DNS。
4. 每台设备单独生成密钥，并在服务端为它添加独立的 `[Peer]` 和隧道地址。可以把自己的配置通过本地文件或二维码导入，但不要发布含私钥的配置、二维码、订阅链接或门户截图。

[WireGuard 官方快速入门](https://www.wireguard.com/quickstart/)

## 5. 看账单和余额

先看 Azure「教育」中的剩余额度，再看「成本管理」里各资源的实际费用。额度余额和费用明细可能在不同时间更新。

- 分别核对 **VM 算力、系统盘、公网 IPv4、出站流量**。微软学生页面列有部分 B 系列机型的免费时数，是否实际抵扣以自己订阅的计费明细为准。建好后先看「成本管理 → 成本分析」，账单可能延迟出现。
- 关闭手机隧道只会断开连接。Azure VM 显示 **Stopped (Deallocated)** 时通常不再计 VM 算力，仍保留的磁盘、公网 IP 等资源可能继续计费。确定不用时，从资源组逐项核对并删除专为本教程创建的资源。
- 保留学生订阅支出限制，关注教育中心的剩余额度。按微软条款，额度耗尽后订阅会停用，继续使用需要按条款升级。自行估算费用时，以区域、规格和实际流量为准。

[学生权益](https://azure.microsoft.com/en-us/free/students/) · [Azure 定价计算器](https://azure.microsoft.com/zh-cn/pricing/calculator/) · [带宽计费](https://azure.microsoft.com/zh-cn/pricing/details/bandwidth/) · [虚拟机状态与计费](https://learn.microsoft.com/en-us/azure/virtual-machines/states-billing)
