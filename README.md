Phytium DWMAC Ethernet Driver (DKMS)

这是专为国产飞腾平台（飞腾 FTC663 / FT-2000/4 / D2000 / 长城等整机）适配现代 Linux（如 Arch Linux ARM，内核 6.x 和 7.x+）的千兆有线网卡驱动。

本仓库在原作者驱动的基础上，追加了一个**针对长城 GW-001M1A-FTF 整机（飞腾 FT-2000/4，ACPI HID `FTGM0001`）**的补丁，用于修复该机型在 Ubuntu 上板载有线网卡无法启用的问题，见「本仓库追加的补丁」一节。

主要修复的问题：
1. 架构识别：修复内核升级后将千兆 GMAC 误识别为百兆老网卡、随机生成 MAC 地址以及 MDIO 注册失败的问题。
2. 内核 ABI 兼容：自动适配新内核（Linux 6.19+ 及 7.x）中断结构体偏移变更，解决启动时申请中断失败（error -22）。
3. 千兆丢包根治：采用 RGMII_RXID 模式，消除 1000M 高频下 MAC 与 PHY 双重发送延时叠加冲突，实现千兆线速 0 丢包、0 延迟卡顿。
4. 电源时钟优化：修复休眠挂起时时钟双重释放报警。

一键安装：
git clone https://github.com/bin2yx/phytium-dwmac-dkms.git
cd phytium-dwmac-dkms
sudo ./install.sh

一键卸载：
sudo ./uninstall.sh

Arch Linux 用户也可以直接打包安装：
makepkg -si

注意：如果系统启用了引导镜像（initramfs），安装后建议执行一次更新镜像，防止开机从旧镜像加载旧驱动：
sudo mkinitcpio -P

## 本仓库追加的补丁：长城 GW-001M1A-FTF（飞腾 FT-2000/4）DMA 修复

适用机型：**长城 GW-001M1A-FTF**（飞腾 **FT-2000/4**，板载千兆 GMAC 的 ACPI HID 为 **`FTGM0001`**，PHY 为 Qualcomm Atheros AR8035）。

问题现象：驱动能绑定、能识别 PHY 和 MAC，但 `ip link set <iface> up` 报 `RTNETLINK answers: Cannot allocate memory`，内核日志报：

```
dwmac-phytium FTGM0001:00 enaftgm1i0: stmmac_setup_dma_desc: DMA descriptors allocation failed
```

根因：`FTGM0001` 在 ACPI DSDT 中既没有 `_DMA` 属性也没有 `_CCA`（本机型实测 DSDT 中 `_CCA` 出现 0 次），也不在 IORT 描述范围内。因此 arm64 上 `acpi_dma_supported()` 返回 false（内核开了 `CONFIG_ACPI_CCA_REQUIRED=y`），ACPI 核心按 `DEV_DMA_NOT_SUPPORTED` 给设备安装了 `dma_dummy_ops`。该 ops 的 `dma_supported()` 恒返回 0、且没有 `->alloc()`，于是：

- `dma_set_mask()` 返回 `-EIO`
- `dma_alloc_coherent()` 静默返回 `NULL`

最终 `stmmac` 的 DMA 描述符分配失败，接口无法打开（`-ENOMEM`）。

补丁内容：在 `dwmac_phytium_probe()` 中丢弃这套 dummy ops，回落到通用的 dma-direct 实现，并设置 32 位一致性掩码：

```c
#include <linux/dma-map-ops.h>
...
	/* FTGM0001 has no _DMA property and no IORT entry, ... */
	set_dma_ops(&pdev->dev, NULL);
	if (!pdev->dev.dma_mask)
		pdev->dev.dma_mask = &pdev->dev.coherent_dma_mask;
	dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
```

验证环境：Ubuntu 26.04 / 内核 `7.0.0-30-generic`（aarch64），长城 GW-001M1A-FTF（FT-2000/4）。打补丁后接口正常 up，`Link is Up - 1Gbps/Full`，可正常收发报文，`rx_errors` / `tx_errors` / `rx_crc_errors` 全为 0。

> 更「正统」的修法是让固件在 DSDT 中为 `FTGM0001` 补上 `_CCA (1)`，内核便会走 `DEV_DMA_COHERENT` 正常路径；但改固件不现实，故在驱动侧兜底。
>
> 注：飞腾官方内核里的 `dwmac-phytium` 驱动匹配的 ACPI HID 是 `PHYT0004`，与本机型 DSDT 中的 `FTGM0001` 不同，官方原版在这类整机上不会绑定。
