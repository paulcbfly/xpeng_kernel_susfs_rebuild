# AI 交接文档 — xpeng_kernel_susfs 编译项目

> 本文件由 AI 编写，用于让**下一个 AI（或人类维护者）无需重新摸索**即可接管本仓库的
> SUSFS + ReSukiSU 内核编译任务。记录了全部踩坑、根因、修复方法和 GitHub Actions 编译流程。

---

## 0. 项目一句话

Motorola xpeng（Edge S30 / G200，代号 xpeng，**5.4 内核**）的编译脚本仓库，
内核集成 **SUSFS v2.2.0** + **ReSukiSU（固定 commit 59c99fdf）**，用 GitHub Actions 自动编译并发布 Release。

## 1. 仓库拓扑（两个仓库，缺一不可）

| 仓库 | 角色 | 分支 |
|------|------|------|
| `paulcbfly/xpeng_kernel_susfs` | **本仓库**：build 脚本 + GitHub Actions workflow | `5.4.302-s3rxc32.33-8-25-ReSukiSU` |
| `paulcbfly/android_kernel_motorola_xpeng` | 内核源码（fork 自 LuoJuly）+ SUSFS 适配 commit | `5.4.302-s3rxc32.33-8-25-susfs` |

- build 仓库**不含内核源码**，workflow 运行时通过 `git clone` 拉取内核仓库指定分支。
- 内核 fork 分支 `5.4.302-s3rxc32.33-8-25-susfs` = 上游 `5.4.302-s3rxc32.33-8-25` + 一个 SUSFS 适配 commit（`b3ecce7eb`）。

### 关键文件修改点（build 仓库）
- `.github/workflows/build-resukisu-edge-s30.yml`（Edge S30）、`build-resukisu-g200.yml`（G200）
  - `KERNEL_URL` → `https://github.com/paulcbfly/android_kernel_motorola_xpeng.git`
  - `KERNEL_BRANCH` → `5.4.302-s3rxc32.33-8-25-susfs`
  - `UPDATE_RESUKISU` → `${{ inputs.update_resukisu || 'false' }}`（默认**不**更新 ReSukiSU，见问题 #1）
- `scripts/ci/build_resukisu_boot.sh`
  - `KERNEL_URL` / `KERNEL_BRANCH` 默认值同上
  - `update_resukisu()` 函数：默认把 ReSukiSU 子模块 **pin 到 `59c99fdf`**（SUSFS v2.2.0 兼容）

## 2. 本次遇到的全部编译问题（按时间顺序）

### 问题 #1（致命，链接失败）：ReSukiSU 更新到 origin/main 后与 SUSFS v2.2.0 不兼容

**现象**：GitHub Actions 编译到最后链接阶段 `ld.lld` 报 4 个 undefined symbol：

```
ld.lld: error: undefined symbol: susfs_set_current_proc_umounted_for_zygote_next
ld.lld: error: undefined symbol: susfs_set_current_proc_no_su
ld.lld: error: undefined symbol: susfs_is_current_proc_no_su
ld.lld: error: undefined symbol: susfs_clear_current_proc_no_su
>>> referenced by vmlinux.o:(ksu_handle_post_execve.cfi_jt) / (mnt_drop_write.cfi_jt)
```

**根因**：
- ReSukiSU 的 `origin/main`（2026-09 以后）在其 `kernel/hook/setuid_hook.c` 中
  调用了 `susfs_set_current_proc_no_su()` 等 4 个新内核符号，并 `#include <linux/susfs_def.h>`。
- 这些符号属于 **SUSFS v2.3.0**（`TIF_PROC_NO_SU=34`、`TIF_PROC_UMOUNTED_FOR_ZYGOTE_NEXT=35`），
  而本内核适配的是 **SUSFS v2.2.0**（只有 `TIF_PROC_UMOUNTED=33`），因此符号未定义。
- ReSukiSU 官方在 `kernel/tools/inline_hook_check.mk` 中明确声明：
  *"We keep tracking simonpunk's latest changes, and don't maintain ANY backward compatibility for old version of susfs."*
  （只跟进最新 susfs，**不向后兼容旧版**）

**修复**：不升级 SUSFS，而是把 ReSukiSU 锁定在 v2.2.0 兼容的 commit `59c99fdf`（2026-08-02）：
- 修改 `scripts/ci/build_resukisu_boot.sh` 的 `update_resukisu()`，默认 `UPDATE_RESUKISU=false` 时
  `git checkout -f 59c99fdf...`；
- workflow 的 `UPDATE_RESUKISU` 改为 `${{ inputs.update_resukisu || 'false' }}`，
  且 `workflow_dispatch` 输入的 `update_resukisu` 默认值改为 `false`，schedule 也不再强制更新。

**⚠️ 教训**：ReSukiSU 一旦升级到最新 main，内核侧 SUSFS **必须**同步升级到 v2.3.0+，
否则链接必失败。若未来要升级，参考 `cctv18/susfs4oki`（SUSFS v2.3.0，ReSukiSU 配套）：
- 其 `susfs_def.h` 用 `inode->i_mapping->flags` 存 AS_FLAGS（v2.2.0 用 `inode->i_state`，**不兼容，不能只替换头文件**）；
- 需要整体替换 `fs/susfs.c` + `susfs.h` + `susfs_def.h` 并重做 hooks（v2.3.0 无现成 5.4 补丁，需手动移植）。

---

### 问题 #2（补丁上下文不匹配）：参考 commit 直接 `git apply` 部分文件失败

**现象**：`git apply --check` 对参考补丁 `2fa1be6` 报 5 个文件失败。

**根因与处理**：
| 文件 | 原因 | 处理 |
|------|------|------|
| `.gitmodules` / `drivers/kernelsu` / `drivers/Kconfig` / `drivers/Makefile` | 参考 commit 是"从零加 ReSukiSU"，包含子模块+软链接+Kconfig；**本内核已集成 ReSukiSU**（作者 LuoJuly 已做） | **跳过**，已存在 |
| `arch/arm64/configs/vendor/lineage_xpeng.config` | 参考 commit 的目标是 LOS 内核，路径不存在；本内核是 MMI 内核 | 改在 `arch/arm64/configs/vendor/ext_config/moto-lahaina-xpeng.config` 和 `vendor/lahaina-qgki_defconfig` |
| `fs/proc/fd.c` | 参考内核的 `seq_printf` 有 `ino` 字段，MMI 5.4 内核无 | 手动适配：SUS_MOUNT/OPEN_REDIRECT 分支去掉 `ino` 输出，保留 `mnt_id` 伪装逻辑 |
| `fs/proc/task_mmu.c` | 参考内核用 `end = VMA_PAD_START(vma)`，MMI 用 `end = vma->vm_end` | 手动适配，其余 SUSFS 逻辑不变 |
| `kernel/reboot.c` | 本内核已有 `CONFIG_KSU_MANUAL_HOOK` 版本 hook | 保留原有 MANUAL_HOOK 块，新增 `CONFIG_KSU_SUSFS` 块（两者互斥，choice 单选） |

**核验**：手动适配后 `grep -rn "CONFIG_KSU_MANUAL_HOOK" arch/arm64/configs/` 应 0 残留。

---

### 问题 #3（配置切换）：Kconfig choice 互斥，必须把 MANUAL_HOOK 切到 SUSFS

**根因**：ReSukiSU 的 Kconfig 中 hook 方式是一个 **choice**（三选一）：
`CONFIG_KSU_TRACEPOINT_HOOK` / `CONFIG_KSU_MANUAL_HOOK` / `CONFIG_KSU_SUSFS`。
原内核用的是 `MANUAL_HOOK`（作者 resukisu 适配），要启用 SUSFS 必须切换。

**修改**（两处，缺一不可）：
1. `arch/arm64/configs/vendor/ext_config/moto-lahaina-xpeng.config`：
   ```
   CONFIG_KSU=y
   CONFIG_KSU_SUSFS=y          # 取代 CONFIG_KSU_MANUAL_HOOK=y
   CONFIG_KALLSYMS_ALL=y
   ```
   （删掉全部 `CONFIG_KSU_MANUAL_HOOK_AUTO_*`）
2. `arch/arm64/configs/vendor/lahaina-qgki_defconfig`（第 806 行附近）：
   `CONFIG_KSU_MANUAL_HOOK=y` → `CONFIG_KSU_SUSFS=y`

**验证方法**（本地，无需完整编译）：
```bash
# 用 kernel 自带 merge_config 模拟 GKI 合并流程（顺序=base→GKI→QGKI→debugfs→moto ext_config）
scripts/kconfig/merge_config.sh -O /tmp/kmerge -m \
  arch/arm64/configs/gki_defconfig \
  arch/arm64/configs/vendor/lahaina_GKI.config \
  arch/arm64/configs/vendor/lahaina_QGKI.config \
  arch/arm64/configs/vendor/debugfs.config \
  arch/arm64/configs/vendor/ext_config/moto-lahaina-xpeng.config
# 然后 olddefconfig 完整解析（本机是 aarch64，gcc 直接可用；LD=ld 绕过交叉 ld 版本检测）
cp /tmp/kmerge/.config /tmp/kout/.config
make O=/tmp/kout ARCH=arm64 LD=ld olddefconfig
grep -E "^CONFIG_KSU" /tmp/kout/.config
# 期望：CONFIG_KSU=y + CONFIG_KSU_SUSFS=y + 全部 KSU_SUSFS_* 子选项 =y，无 MANUAL_HOOK
```

---

### 问题 #4（本地验证陷阱）：gcc-wrapper.py 把 warning 当 error（CI 不会踩，本地会）

**现象**：本地 `make fs/susfs.o` 报
`error, forbidden warning: kern_levels.h:5 ... format '%u' expects ... 'long long int'`。

**根因**：MMI 内核的 `Makefile:466` 把 CC 包了一层 `scripts/gcc-wrapper.py`，
该脚本把**非白名单的编译警告一律视为错误**（`sys.exit(1)`）。SUSFS v2.2.0 上游代码在
内核 < 6.1 分支的日志格式串里用了 `%u` 格式化 `long long` 类型的 `spoofed_size`。

**为什么 CI 不炸**：CI 用 Android clang 编译，内核会把 `-Wno-format` 加进 CFLAGS
（`make V=1` 可见），clang 的 format 警告被整体关掉；而 gcc 本地的格式检查关不干净。
**结论**：不要为了本地 gcc 报错去改 susfs.c 的格式串；CI（clang）下无警告。
本地验证请换用 `make CC=clang`。

**本地单文件/子系统快速验证**（已验证通过）：
```bash
make O=/tmp/kout ARCH=arm64 prepare          # 生成本地头文件（缺编译器的坑：先 make prepare 到报错前即可）
make O=/tmp/kout ARCH=arm64 CC=clang fs/susfs.o fs/exec.o fs/namei.o fs/namespace.o \
  fs/open.o fs/notify/fdinfo.o fs/proc/base.o fs/proc/cmdline.o fs/proc/fd.o \
  fs/proc/task_mmu.o fs/proc_namespace.o fs/read_write.o fs/readdir.o fs/stat.o fs/statfs.o \
  drivers/input/input.o kernel/kallsyms.o kernel/reboot.o kernel/sys.o mm/memory.o security/selinux/avc.o
```

---

### 问题 #5（hook 强制检查）：编译 KernelSU 驱动时 ReSukiSU 会校验内核 hooks

**现象/机制**：编译 `drivers/kernelsu/` 时，`kernel/tools/inline_hook_check.mk`
用 grep 校验 7 个必要 hook 必须存在于内核源码，缺失即 `$(error)` 编译失败。

**要求清单**（SUSFS 模式下）：
| hook | 所在文件 |
|------|----------|
| `ksu_handle_setresuid` | kernel/sys.c |
| `ksu_handle_execveat` | fs/exec.c |
| `ksu_handle_faccessat` | fs/open.c |
| `ksu_handle_sys_read` | fs/read_write.c |
| `ksu_handle_stat` | fs/stat.c |
| `ksu_handle_sys_reboot` | kernel/reboot.c |
| `ksu_handle_input_handle_event` | drivers/input/input.c |

同时检查**旧版不兼容 hook 不得存在**：`ksu_vfs_read_hook`、`ksu_input_hook`、
`ksu_execveat_hook`、`ksu_init_rc_hook`。

**验证**：编译 `drivers/kernelsu/` 时日志出现 `-- ReSukiSU/susfs_inline: ksu_handle_* found` 且
`-- SUSFS_VERSION: v2.2.0` 即通过。若有 `WARNING: Detected KSU_MANUAL_HOOK guard` 属正常
（源码中保留了 MANUAL_HOOK 块，choice 互斥下不参与编译，只是 grep 能搜到）。

---

### 问题 #6（defconfig 里 CONFIG_KSU 的来龙去脉）

- `lahaina-qgki_defconfig` 里的 KSU 配置是 **generate_defconfig 流程生成的产物**
  （`scripts/gki/generate_defconfig.sh` 会把合并结果写回该文件），不是手写的。
- 真正的输入是 `ext_config/moto-lahaina-xpeng.config`（MOTO_REQUIRED_CONFIG 最后合并，优先级最高）。
- workflow 构建时会 `git checkout HEAD -- lahaina-qgki_defconfig` 恢复，所以两处都改了才能自洽。

## 3. GitHub Actions 编译流程（下一个 AI 直接照此操作）

### 触发方式
```bash
# 手动触发 Edge S30 编译（默认不更新 ReSukiSU = 锁定 59c99fdf）
gh workflow run build-resukisu-edge-s30.yml --ref 5.4.302-s3rxc32.33-8-25-ReSukiSU

# 手动触发 G200
gh workflow run build-resukisu-g200.yml --ref 5.4.302-s3rxc32.33-8-25-ReSukiSU

# 查看状态/日志
gh run list --workflow build-resukisu-edge-s30.yml --limit 3
gh run view <RUN_ID> --log-failed     # 失败日志
```
- 仓库每月 1 日 UTC 00:00（S30）/ 02:00（G200）自动跑（schedule）。
- 认证：环境变量 `GH_TOKEN`（已配置，权限含 `repo` + `workflow`）。

### workflow 内部流程（约 20-60 分钟）
1. `actions/checkout` build 仓库 `5.4.302-s3rxc32.33-8-25-ReSukiSU`
2. 缓存/下载工具链：`clang-r383902b1`（AOSP）+ GCC 4.9（Lineage 19.1）+ magiskboot
3. `build_resukisu_boot.sh`：
   - `fetch_kernel`：clone 内核 fork 的 `5.4.302-s3rxc32.33-8-25-susfs`（--recursive）
   - `update_resukisu`：**把 KernelSU 子模块 pin 到 59c99fdf**（因 SUSFS v2.2.0）
   - `setup_toolchain`、`build_kernel`（generate_defconfig → Image）
   - `build_wlan_modules`（WiFi ko，vermagic 匹配）
   - `repack_boot`（magiskboot 解包 boot_oem.img 换内核重打包成 boot_ksu.img）
   - `pack_anykernel3`（Image + vendor WiFi kos 打成 AnyKernel3 zip）
4. 上传 artifact + 创建 GitHub **Release**（boot_ksu.img / Image / AnyKernel3.zip / wlan zip）

### 产物命名
```
AnyKernel3-xpeng-EdgeS30-ReSukiSU-5.4.302-v4.1.0-1332-g59c99fdf-S3RXC32.33-8-25.zip
boot_ksu.img   # fastboot: fastboot flash boot boot_ksu.img
```

## 4. 本地如何复现构建（可选）

```bash
export VARIANT=edge-s30    # 或 g200（ENABLE_NFC=true）
export ENABLE_NFC=false
export UPDATE_RESUKISU=false
export KERNEL_URL=https://github.com/paulcbfly/android_kernel_motorola_xpeng.git
export KERNEL_BRANCH=5.4.302-s3rxc32.33-8-25-susfs
./scripts/ci/build_resukisu_boot.sh
# 产物：.ci-work/edge-s30/release/
```
> 完整本地构建需要下载 clang-r383902b1（约 1-2GB），磁盘需 >20GB 空闲。

## 5. 踩坑速查表（TL;DR）

| 症状 | 原因 | 处置 |
|------|------|------|
| ld.lld undefined `susfs_*_no_su` | ReSukiSU 被更新到 main，需 SUSFS v2.3+ | 锁回 `59c99fdf`；或整体升 SUSFS v2.3.0 |
| 本地 gcc `forbidden warning %u` | MMI `gcc-wrapper.py` 把 warning 当 error | 无视；验证用 `CC=clang` |
| Kconfig 里 `KSU_SUSFS` 不生效 | choice 与 MANUAL_HOOK 互斥，残留未清 | 两处 defconfig 都改：`ext_config/moto-lahaina-xpeng.config` + `lahaina-qgki_defconfig` |
| hook 检查 `$(error)` | 7 个 hooks 缺一个 | 对照第 2 节问题 #5 清单补齐 |
| `fs/proc/fd.c` patch 不适用 | 参考内核有 `ino` 字段，MMI 无 | 去掉 ino 输出即可，保留 mnt_id 伪装 |
| `compile.h not found`（本地单文件编译） | 未完整 `make prepare` | 本地验证可忽略，CI 全量编译会生成 |

## 6. 关于 SUSFS v2.3 / 模块扩展

**结论：已全部放弃，不再尝试。**

- SUSFS v2.3：曾基于 AstideLabs v2.3 补丁做过分支移植，**实测开机黑屏**，已回退。
- 模块扩展（Re:Kernel / BBGuard / BBRv3 / DroidSpaces）：BBRv3 的无条件 TCP 改动导致卡机，已回退。
- 相关内核分支（`feat/*`、`ci/merge-*`）**已全部删除**，内核仓库只保留唯一分支
  `5.4.302-s3rxc32.33-8-25-susfs`。

> 复现附录中的 `ld.lld undefined susfs_*_no_su` 时，直接对照「问题 #1」处理：
> 保持 ReSukiSU 锁定在 `59c99fdf`（`UPDATE_RESUKISU=false`），不要拉到 `origin/main`。

---

*文档生成时间：2026-09-23，最后修订 2026-09-27。作者：AI 助手（100% AI-generated project）。*
