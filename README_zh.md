# xpeng_kernel_susfs（摩托罗拉 xpeng 内核编译）

> **🌐 语言切换**: [English](README.md) | [**简体中文**](README_zh.md)

> **⚠️ 重要声明：本仓库及内核侧 SUSFS 适配为 100% AI 生成（100% AI-GENERATED）。**
> 编译脚本 fork 自 [LuoJuly/android_kernel_motorola_xpeng_build](https://github.com/LuoJuly/android_kernel_motorola_xpeng_build)；
> SUSFS 及模块适配由 AI 依据 [LuoJuly 的参考提交](https://github.com/LuoJuly/android_kernel_motorola_sm7325/commit/2fa1be6d5a63d3958ab2babd56b61f561f74095b)
> 与 [lineage-23.2-SUSFS 分支](https://github.com/LuoJuly/android_kernel_motorola_sm7325/tree/lineage-23.2-SUSFS) 完成，并经自动化工具全流程调试验证。
> **刷机有风险，后果自负。**

> ⚠️ **AI 接管提示**：接手前请先阅读 [`docs/AI_HANDOVER_zh.md`](docs/AI_HANDOVER_zh.md) —— 记录了全部编译问题及解决方案、GitHub Actions 流程、验证方法。

---

## 📱 项目简介

摩托罗拉 **xpeng**（Edge S30 XT2175-2 / G200 5G XT2175-1）**5.4.302 内核**编译脚本仓库，
内置 **ReSukiSU（KernelSU 分支）+ SUSFS v2.2.0** root 隐藏方案（稳定版）。

| 模块 | 说明 |
|------|------|
| **SUSFS** | Secure User File System（SUS_PATH / SUS_MOUNT / SUS_KSTAT / SPOOF_UNAME / OPEN_REDIRECT / SUS_MAP 等全特性） |

> ⚠️ **回退记录 (2026-09-24 / 2026-09-27)**: Re:Kernel / BBGuard / BBRv3 / DroidSpaces 模块分支已全部回退，
> **SUSFS v2.3 亦已放弃**（实测黑屏）。当前为**稳定的纯 SUSFS v2.2 版本**，仅保留单一内核分支。

---

## 🧠 仓库拓扑（两个仓库）

| 仓库 | 角色 | 分支 |
|------|------|------|
| **paulcbfly/xpeng_kernel_susfs_rebuild**（本仓库） | 编译脚本 + GitHub Actions workflow | `5.4.302-s3rxc32.33-8-25-ReSukiSU` |
| **paulcbfly/android_kernel_motorola_xpeng_rebuild** | 内核源码（fork 自 LuoJuly）+ SUSFS 适配 | `5.4.302-s3rxc32.33-8-25-susfs` |

- 编译仓库**不包含内核源码**，Actions 运行时自动 `git clone` 内核仓库指定分支。
- 内核仓库**只保留唯一分支** `5.4.302-s3rxc32.33-8-25-susfs`（上游 8-25 + SUSFS v2.2 适配）。
- 子模块：`KernelSU`→ReSukiSU @ `59c99fdf`（固定，SUSFS v2.2.0 兼容）。

---

## 🛠️ 使用方法（刷机）

### fastboot 方式

```
fastboot reboot fastboot
fastboot flash boot boot_ksu.img
# 若需要：
fastboot -w
```

### AnyKernel3 方式（推荐）

在 recovery / Kernel Flasher 中刷入 `AnyKernel3-*.zip`。
它会安装内核并把 WiFi `.ko` 推送到 `/vendor/lib/modules/`（`do.modules=1`）。
**刷完 AnyKernel3 后不要再安装 KernelSU WiFi 模块。**

### 独立 WiFi 包

仅当用 fastboot 刷了 `boot_ksu.img` 时需要（该方式不替换 vendor ko）。
首次开机后用 ReSukiSU Manager 安装 WiFi 模块再重启。

### 产物

- `.ci-work/<variant>/release/` 下：`boot_ksu.img`、`Image`、`wlan_crc_match_*.zip`、`AnyKernel3-*.zip`
- GitHub Release 页面也会自动发布（boot_ksu.img / Image / AnyKernel3 / wlan zip）

---

## 🔄 手动触发编译

```bash
# 需要 gh CLI + 登录（环境已配置 GH_TOKEN）
gh workflow run build-resukisu-edge-s30.yml --ref 5.4.302-s3rxc32.33-8-25-ReSukiSU
# G200:
gh workflow run build-resukisu-g200.yml --ref 5.4.302-s3rxc32.33-8-25-ReSukiSU

# 查看状态
gh run list --workflow build-resukisu-edge-s30.yml --limit 3
```

每月 1 日 UTC 00:00（Edge S30）/ 02:00（G200）自动编译（schedule）。

---

## 📦 已合入的谷歌安全补丁（内核）

| 文件 | 修复内容 |
|------|----------|
| `net/packet/af_packet.c` | fanout UAF（NETDEV_UP 竞态，CVE 类） |
| `net/core/skbuff.c` | shared-frag 标记保留/传递 ×2、pskb_carve zerocopy 引用计数 |
| `net/ipv6/icmp.c` | `ip6_err_gen_icmpv6_unreach()` 未清 `skb2->cb[]`（信息泄露） |
| `net/ipv6/ip6_tunnel.c` | `ip4ip6_err()` 未清 `cb[]`（信息泄露） |
| `net/tipc/msg.c` | `tipc_buf_append()` 双重释放 |
| `net/nfc/llcp_core.c` | LLCP_CLOSED 检查缺失 return（UAF） |

---

## 📚 相关文档

- [`docs/AI_HANDOVER_zh.md`](docs/AI_HANDOVER_zh.md) — **AI 交接文档（中文）**：全部编译问题、解决方案、验证方法
- [`docs/AI_HANDOVER.md`](docs/AI_HANDOVER.md) — AI 交接文档（英文版）
- [内核仓库 `5.4.302-s3rxc32.33-8-25-susfs` 分支](https://github.com/paulcbfly/android_kernel_motorola_xpeng_rebuild/tree/5.4.302-s3rxc32.33-8-25-susfs)

---

*README 中文版更新于 2026-09-24。本仓库为 100% AI 生成项目。*