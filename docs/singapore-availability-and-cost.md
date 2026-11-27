# 新加坡机型不可用时，怎么查限制和算费用

在 Azure 门户选到新加坡，只说明可以把它填进表单。能否部署还取决于 **订阅、VM 规格、区域或可用区、vCPU 配额，以及镜像和安全类型**。机型旁边显示的价格也不能证明这份订阅能创建它。

## 实查记录

2026-11-27，用 Azure for Students 订阅检查了 **Southeast Asia（新加坡）**，镜像为 Ubuntu 24.04 LTS x64。取消指定可用区后，分别用「受信任启动」和「标准」安全类型检查：

- `B2ats_v2`、`D2as_v5`、`D2as_v6`、`D2s_v5`、`D2s_v3` 均出现在「大小不可用」组。本案例之前还在 `D2s_v3` 上看到过 `NotAvailableForSubscription`。
- 标准安全类型的全规格列表还对部分其他规格分别提示「配额不足 − 系列限制」和「配额不足 − 区域限制」。这些提示和上述五款的「大小不可用」分开显示。门户没有说明这五款不可用的进一步原因。
- 本次只打开规格选择器核对，未创建新加坡资源。实际跑通的是 [Malaysia West 的 WireGuard 部署](vpn-setup.md)。以上结果只代表当时测试的订阅和配置。

## 在自己的订阅上复查

1. 在虚拟机创建页选自己的订阅、目标区域和 Linux 镜像。确认 **Southeast Asia** 是新加坡，别把名称相近的区域混用。
2. 点 **查看所有大小**，搜索想用的规格。先看规格是否可选，再看价格；显示配额不足时，按提示分别检查区域总 vCPU 和对应 VM 系列的 vCPU 配额。
3. 用「查看 + 创建」做部署前验证，保留最后的「创建」按钮不点。即使规格选择器允许选择，最终创建也可能受当时的容量影响。

如果已经装好 Azure CLI 并登录，可用下面的只读查询过滤 D2 规格。先确认 CLI 当前使用的是你要检查的订阅。

```bash
az account show --query name -o tsv
az vm list-skus --location southeastasia --resource-type virtualMachines --size Standard_D2 --all --output table
```

`--all` 会保留受限规格。查看 `Restrictions` 列，区分区域和可用区限制；需要查特定规格时，把 `Standard_D2` 换成完整规格名。若新加坡是硬性要求并且没有可用规格，可向 Azure 支持询问这份订阅在该区域可用的 Linux VM SKU。

[微软的 SKU 不可用排查](https://learn.microsoft.com/en-us/azure/azure-resource-manager/troubleshooting/error-sku-not-available) · [Azure VM vCPU 配额](https://learn.microsoft.com/en-us/azure/virtual-machines/quotas)

## 如果以后有 D 系列可用，US$100 能撑多久？

以下是 **2026-11-27 新加坡 Linux 按量零售价的假设算例**，并非上述订阅的可用规格清单。每月按 730 小时计，假设额度还剩 US$100、只有这一台消耗额度，使用一块 S4 标准 HDD 和一个标准静态公网 IPv4。S4 容量费约 **US$1.54/月**，公网 IPv4 约 **US$3.65/月**，合计约 **US$5.19/月**。表中未计超额流量、磁盘操作和其他资源费用。

| 新加坡规格 | 算力零售价 | 常开总价估算 | US$100 约可覆盖 |
| --- | ---: | ---: | ---: |
| `D2as_v5` | US$0.108/小时 | US$84.03/月 | 36 天 |
| `D2as_v6` | US$0.114/小时 | US$88.41/月 | 34 天 |
| `D2s_v5` | US$0.120/小时 | US$92.79/月 | 33 天 |
| `D2s_v3` | US$0.125/小时 | US$96.44/月 | 约 32 天 |

计算方式：**每月算力 = 小时价 × 实际计费小时**；再加磁盘、静态 IP 和出站流量，最后用**实际剩余额度 ÷ 每月总费用**。还要和额度到期日比较，以先到的为准。适用带宽计费情景下，Azure 列出的首 100 GB/月公网出站免费；是否达到限额以账单为准。

例如 `D2as_v5` 每天用 4 小时，其余时间在 Azure 停机并**解除分配**，仍保留同一块磁盘和公网 IP，约需 **US$18.33/月**，US$100 约够 **5.5 个月**。手机关闭隧道时，Azure VM 仍会继续运行，除非你另行停止 VM。

[Azure 零售价 API](https://learn.microsoft.com/en-us/rest/api/cost-management/retail-prices/azure-retail-prices) · [Azure 带宽定价](https://azure.microsoft.com/en-us/pricing/details/bandwidth/) · [VM 状态与计费](https://learn.microsoft.com/en-us/azure/virtual-machines/states-billing) · [学生优惠条款](https://azure.microsoft.com/zh-cn/pricing/offers/ms-azr-0170p/)
