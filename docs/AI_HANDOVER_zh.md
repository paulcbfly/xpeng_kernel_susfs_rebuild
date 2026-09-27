# AI 交接文档（中文版）— xpeng_kernel_susfs 编译项目

> 本文档由 AI 编写，用于让**下一个 AI（或人类维护者）无需重新摸索**即可接管本仓库的
> SUSFS + ReSukiSU + 可选模块内核编译任务。记录了全部踩坑、根因、修复方法和 GitHub Actions 编译流程。

---

## 0. 项目一句话

Motorola xpeng（Edge S30 / G200，代号 xpeng，**5.4 内核**）的编译脚本仓库，
内核集成 **SUSFS v2.2.0** + **ReSukiSU（固定 commit 59c99fdf）** +
**Re:Kernel + Baseband-guard + BBRv3 + DroidSpaces**（可选项，编译时可切换），
用 GitHub Actions 自动编译并发布 Release。

## 1. 仓库拓扑（两个仓库，缺一不可）

| 仓库 | 角色 | 分支 |
|------|------|------|
| `paulcbfly/xpeng_kernel_susfs` | **编译仓库**：build 脚本 + GitHub Actions workflow | `5.4.302-s3rxc32.33-8-25-ReSukiSU` |
| `paulcbfly/android_kernel_motorola_xpeng` | 内核源码（fork 自 LuoJuly）+ 全部适配 commit | `5.4.302-s3rxc32.33-8-25-modules` |

- 编译仓库**不含内核源码**，workflow 运行时 `git clone` 内核仓库指定分支。
- 内核分支 `5.4.302-s3rxc32.33-8-25-modules`：
  - 基座 = 上游 `5.4.302-s3rxc32.33-8-25`
  - + SUSFS 适配 commit `b3ecce7eb`（SUSFS v2.2.0 + ReSukiSU 子模块 pin 59c99fdf）
  - + 模块扩展 commit `8f74e34f6`（Re:Kernel、BBGuard、BBRv3、DroidSpaces、谷歌安全补丁）
- 子模块：`KernelSU` → ReSukiSU @ `59c99fdf`；`Baseband-guard` → vc-teahouse @ `cef0daa`

### 关键文件修改点（编译仓库）
- `.github/workflows/build-resukisu-edge-s30.yml`（Edge S30）、`build-resukisu-g200.yml`（G200）
  - `KERNEL_URL` → `https://github.com/paulcbfly/android_kernel_motorola_xpeng.git`
  - `KERNEL_BRANCH` → `5.4.302-s3rxc32.33-8-25-modules`
  - `UPDATE_RESUKISU` → `${{ inputs.update_resukisu || 'false' }}`（默认不更新）
  - 新增 5 个模块输入：`enable_susfs` / `enable_rekernel` / `enable_bbguard` / `enable_bbrv3` / `enable_droidspaces`（默认全 true）
- `scripts/ci/build_resukisu_boot.sh`
  - `KERNEL_URL` / `KERNEL_BRANCH` 默认值同上
  - `update_resukisu()`：默认 pin ReSukiSU 到 `59c99fdf`
  - `build_kernel()`：defconfig 生成后、olddefconfig 前，用 `scripts/config` 按 `ENABLE_*` 环境变量调整 `.config`

## 2. 编译问题与解决方案（按时间顺序）

### 问题 #1（致命，链接失败）：ReSukiSU 更新到 origin/main 后与 SUSFS v2.2.0 不兼容

**现象**：GitHub Actions 链接阶段 `ld.lld` 报 4 个 undefined symbol：

```
ld.lld: error: undefined symbol: susfs_set_current_proc_umounted_for_zygote_next
ld.lld: error: undefined symbol: susfs_set_current_proc_no_su
ld.lld: error: undefined symbol: susfs_is_current_proc_no_su
ld.lld: error: undefined symbol: susfs_clear_current_proc_no_su
>>> referenced by vmlinux.o:(ksu_handle_post_execve.cfi_jt) / (mnt_drop_write.cfi_jt)
```

**根因**：
- ReSukiSU 的 `origin/main`（2026-09 后）在其 `kernel/hook/setuid_hook.c` 中调用
  `susfs_set_current_proc_no_su()` 等 4 个新内核符号，并 `#include <linux/susfs_def.h>`。
- 这些符号属于 **SUSFS v2.3.0**（`TIF_PROC_NO_SU=34`、`TIF_PROC_UMOUNTED_FOR_ZYGOTE_NEXT=35`），
  本内核适配的是 **SUSFS v2.2.0**（只有 `TIF_PROC_UMOUNTED=33`），符号未定义。
- ReSukiSU 官方声明：*"We keep tracking simonpunk's latest changes, and don't maintain ANY backward compatibility for old version of susfs."*

**修复**：不升级 SUSFS，把 ReSukiSU 锁定在 v2.2.0 兼容 commit `59c99fdf`（2026-08-02）：
- `update_resukisu()` 默认 `UPDATE_RESUKISU=false` 时 `git checkout -f 59c99fdf...`
- workflow 的 `UPDATE_RESUKISU` 改为 `${{ inputs.update_resukisu || 'false' }}`，输入默认 `false`

**⚠️ 教训**：ReSukiSU 一旦升级到 main，内核侧 SUSFS **必须**同步升级 v2.3.0+，否则链接必失败。
升级参考 `cctv18/susfs4oki`（v2.3.0）：其 `susfs_def.h` 用 `inode->i_mapping->flags` 存 AS_FLAGS
（v2.2.0 用 `inode->i_state`，**不兼容，不能只换头文件**），需整体替换 `fs/susfs.c` + `susfs.h` + `susfs_def.h` 并重做 hooks。

---

### 问题 #2（补丁上下文不匹配）：参考 commit `git apply` 部分文件失败

| 文件 | 原因 | 处理 |
|------|------|------|
| `.gitmodules` / `drivers/kernelsu` / `drivers/Kconfig` / `drivers/Makefile` | 参考 commit 是"从零加 ReSukiSU"；本内核已集成 | **跳过** |
| `arch/arm64/configs/vendor/lineage_xpeng.config` | LOS 内核路径；本内核是 MMI | 改 `ext_config/moto-lahaina-xpeng.config` + `lahaina-qgki_defconfig` |
| `fs/proc/fd.c` | 参考内核 seq_printf 有 `ino`，MMI 无 | 手动适配：去掉 ino 输出，保留 mnt_id 伪装 |
| `fs/proc/task_mmu.c` | 参考用 `VMA_PAD_START`，MMI 用 `vma->vm_end` | 手动适配 |
| `kernel/reboot.c` | 本内核已有 MANUAL_HOOK 版本 | 保留 MANUAL_HOOK 块，新增 KSU_SUSFS 块（choice 互斥） |

---

### 问题 #3（配置切换）：Kconfig choice 互斥，MANUAL_HOOK 必须切 SUSFS

ReSukiSU 的 Kconfig 中 hook 方式是一个 **choice**（三选一）：
`KSU_TRACEPOINT_HOOK` / `KSU_MANUAL_HOOK` / `KSU_SUSFS`。原内核用 MANUAL_HOOK，不切换则 SUSFS 不生效。

**修改**（两处）：
1. `arch/arm64/configs/vendor/ext_config/moto-lahaina-xpeng.config`：
   ```
   CONFIG_KSU=y
   CONFIG_KSU_SUSFS=y          # 取代 CONFIG_KSU_MANUAL_HOOK=y
   CONFIG_KALLSYMS_ALL=y
   ```
   （删除全部 `CONFIG_KSU_MANUAL_HOOK_AUTO_*`）
2. `arch/arm64/configs/vendor/lahaina-qgki_defconfig`（~806 行）：
   `CONFIG_KSU_MANUAL_HOOK=y` → `CONFIG_KSU_SUSFS=y`

**模块可选切换时注意**：`ENABLE_SUSFS=false` 会把 `.config` 里 `KSU_SUSFS` 关闭并开 `KSU_MANUAL_HOOK`，
此时走 MANUAL_HOOK 模式（不需要 fs/susfs.c，因为 `CONFIG_KSU_SUSFS` 未定义，susfs.o 不编）。

---

### 问题 #4（本地验证陷阱）：gcc-wrapper.py 把 warning 当 error（CI 不踩，本地会）

- MMI 内核 `Makefile:466` 把 CC 包了 `scripts/gcc-wrapper.py`，**非白名单警告一律视为错误**。
- SUSFS v2.2.0 在 <6.1 分支日志格式有 `%u` 格式化 `long long`（上游已知），gcc 报 `forbidden warning`。
- **CI 不炸**：CI 用 Android clang + `-Wno-format`，警告被关。
- **结论**：不要为本地 gcc 报错去改 susfs.c 格式串；本地验证用 `make CC=clang`。

**本地快速验证**：
```bash
make O=/tmp/kout ARCH=arm64 prepare
make O=/tmp/kout ARCH=arm64 CC=clang fs/susfs.o fs/exec.o fs/namei.o \
  drivers/input/input.o kernel/reboot.o kernel/sys.c ...   # 逐个编译改过的文件
```

---

### 问题 #5（hook 强制检查）：编译 KernelSU 驱动会校验 hooks

`kernel/tools/inline_hook_check.mk` grep 校验 7 个必要 hook，缺失即编译失败：

| hook | 所在文件 |
|------|----------|
| `ksu_handle_setresuid` | kernel/sys.c |
| `ksu_handle_execveat` | fs/exec.c |
| `ksu_handle_faccessat` | fs/open.c |
| `ksu_handle_sys_read` | fs/read_write.c |
| `ksu_handle_stat` | fs/stat.c |
| `ksu_handle_sys_reboot` | kernel/reboot.c |
| `ksu_handle_input_handle_event` | drivers/input/input.c |

旧 hook（`ksu_vfs_read_hook`/`ksu_input_hook`/`ksu_execveat_hook`/`ksu_init_rc_hook`）不得存在。
日志出现 `-- SUSFS_VERSION: v2.2.0` 即通过。`WARNING: Detected KSU_MANUAL_HOOK guard` 属正常。

---

## 3. GitHub Actions 编译流程

### 触发方式
```bash
gh workflow run build-resukisu-edge-s30.yml --ref 5.4.302-s3rxc32.33-8-25-ReSukiSU
gh workflow run build-resukisu-g200.yml --ref 5.4.302-s3rxc32.33-8-25-ReSukiSU
gh run list --workflow build-resukisu-edge-s30.yml --limit 3
gh run view <RUN_ID> --log-failed     # 失败日志
```
- 每月 1 日 UTC 00:00（Edge S30）/ 02:00（G200）自动跑（schedule）。
- 认证：环境变量 `GH_TOKEN`（已配置，含 `repo` + `workflow` 权限）。
- 网页触发：Actions 页 → Run workflow → 勾选/取消模块选项（见 README_zh.md）。

### workflow 内部流程（约 20-60 分钟）
1. `actions/checkout` 编译仓库 `5.4.302-s3rxc32.33-8-25-ReSukiSU`
2. 缓存/下载工具链：`clang-r383902b1`（AOSP）+ GCC 4.9（Lineage 19.1）+ magiskboot
3. `build_resukisu_boot.sh`：
   - `fetch_kernel`：clone 内核 fork 的 `5.4.302-s3rxc32.33-8-25-modules`（--recursive）
   - `update_resukisu`：pin KernelSU 子模块到 59c99fdf（默认）
   - `setup_toolchain`、`build_kernel`（generate_defconfig → **模块开关** → olddefconfig → Image）
   - `build_wlan_modules`（WiFi ko）
   - `repack_boot`（magiskboot 打包 boot_ksu.img）
   - `pack_anykernel3`（Image + WiFi kos）
4. 上传 artifact + 创建 GitHub Release

### 产物命名
```
AnyKernel3-xpeng-EdgeS30-ReSukiSU-5.4.302-v4.1.0-1332-g59c99fdf-S3RXC32.33-8-25.zip
boot_ksu.img   # fastboot: fastboot flash boot boot_ksu.img
```

## 4. 配置开关（仅 SUSFS）

`build_kernel()` 中，在 `make vendor/lahaina-qgki_defconfig` 之后、`olddefconfig` 之前：

```bash
"${KERNEL_DIR}/scripts/config" --file "${OUT_DIR}/.config" --enable/--disable/--set-str <CONFIG> ...
```

- `ENABLE_SUSFS=false` → `--disable KSU_SFS` + `--enable KSU_MANUAL_HOOK`（AUTO_* 子项由 olddefconfig 补齐）

> 模块开关（Re:Kernel / BBGuard / BBRv3 / DroidSpaces）**已全部移除**，不再存在。

## 5. 本地复现构建（可选）

```bash
export VARIANT=edge-s30     # 或 g200（ENABLE_NFC=true）
export ENABLE_NFC=false
export UPDATE_RESUKISU=false
export KERNEL_URL=https://github.com/paulcbfly/android_kernel_motorola_xpeng_rebuild.git
export KERNEL_BRANCH=5.4.302-s3rxc32.33-8-25-susfs
./scripts/ci/build_resukisu_boot.sh
# 产物：.ci-work/edge-s30/release/
```
> 完整本地构建需下载 clang-r383902b1（约 1-2GB），磁盘需 >20GB 空闲。

## 6. 踩坑速查表（TL;DR）

| 症状 | 原因 | 处置 |
|------|------|------|
| ld.lld undefined `susfs_*_no_su` | ReSukiSU 更新到 main，需 SUSFS v2.3+ | 锁回 59c99fdf；或整体升 SUSFS v2.3.0 |
| 本地 `forbidden warning %u` | MMI `gcc-wrapper.py` 警告当错误 | 验证用 `CC=clang` |
| `KSU_SUSFS` 不生效 | choice 与 MANUAL_HOOK 互斥残留 | 两处 defconfig 都改 |
| hook 检查 `$(error)` | 7 个 hooks 缺一个 | 对照问题 #5 清单补齐 |
| `can't open file "drivers/net/rekernel/Kconfig"` | 旧 netlink 版残留 | 删 drivers/net/Kconfig + Makefile 引用 |
| `fs/proc/fd.c` patch 不适用 | 参考内核有 ino，MMI 无 | 去掉 ino 输出 |
| `compile.h not found`（本地单文件） | 未完整 prepare | 本地验证可忽略，CI 全量编译会生成 |

## 7. 关于 SUSFS v2.3 / 模块扩展

**结论：已全部放弃，不再尝试。**

- SUSFS v2.3：曾基于 AstideLabs v2.3 补丁做过分支移植，**实测开机黑屏**，已回退。
- 模块扩展（Re:Kernel / BBGuard / BBRv3 / DroidSpaces）：BBRv3 的无条件 TCP 改动导致卡机，已回退。
- 相关内核分支（`feat/*`、`ci/merge-*`）**已全部删除**，内核仓库只保留唯一分支
  `5.4.302-s3rxc32.33-8-25-susfs`。

> 再次出现 `ld.lld undefined susfs_*_no_su` 时，直接对照「问题 #1」处理：
> 保持 ReSukiSU 锁定 `59c99fdf`（`UPDATE_RESUKISU=false`），不要拉到 `origin/main`。

## 8. 本次合入的谷歌安全补丁清单

| commit 内容 | 文件 |
|-------------|------|
| fanout UAF（NETDEV_UP 竞态） | net/packet/af_packet.c |
| shared-frag 保留 + 传递 + pskb_carve zerocopy | net/core/skbuff.c |
| `ip6_err_gen_icmpv6_unreach()` cb[] 清理 | net/ipv6/icmp.c |
| `ip4ip6_err()` cb[] 清理 | net/ipv6/ip6_tunnel.c |
| `tipc_buf_append()` 双重释放 | net/tipc/msg.c |
| LLCP_CLOSED 缺 return（UAF） | net/nfc/llcp_core.c |

---

*文档生成时间：2026-09-24，最后修订 2026-09-27。作者：AI 助手（100% AI-generated project）。*