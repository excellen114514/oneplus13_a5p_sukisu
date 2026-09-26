# 每日构建一加平板2Pro与ACE6T的ReSukiSU内核

## 不提供管理器下载

### KPM 已移除，SUSFS / BBR 保留

> ReSukiSU 上游（`ReSukiSU/ReSukiSU` commit `faac309`，2026-06-06）已彻底移除
> KPM（KernelPatch）支持，`CONFIG_KPM` 不再是合法的 Kconfig 选项。
> 因此本仓库不再提供 KPM 开关，也不再对编译出的 `Image` 做 `patch_linux` 二次修补；
> defconfig 中显式写入 `CONFIG_KPM=n`，避免旧配置残留。

## 工作流说明

| 文件 | 设备 | 源码 | 内核 |
|------|------|------|------|
| `oneplus_ace_6t.yml` | 一加 Ace 6T（`macan`，sm8845） | AOSP GKI `common-android16-6.12-2025-09` | 6.12 |
| `oneplus_pad_2_pro.yml` | 一加平板 2 Pro（`sun`，sm8750） | `OnePlusOSS/kernel_manifest` @ `oneplus/sm8750`（`oneplus_pad_2_pro.xml`） | 6.6 (android15) |

### 各设备实际集成的特性

| 特性 | Ace 6T | 平板 2 Pro |
|------|--------|-----------|
| ReSukiSU | ✅ | ✅ |
| SUSFS | ✅ | ✅ |
| BBR (+ `fq`) | ✅ | ✅ |
| zram lz4k / lz4kd | ❌ 未集成 | ✅ |
| HMBird GKI 转换 | ❌ | ✅ |
| sched_ext | ❌ | ✅（可关闭） |

> zram(lz4k·lz4kd) 仅在平板 2 Pro 上集成：`SukiSU_patch` 里的 lz4k 树面向
> 6.6 / 6.1 内核，Ace 6T 走的是 6.12 GKI/Kleaf 构建，故未引入。

两个工作流都支持以下手动触发参数：

| 参数 | 说明 |
|------|------|
| `ksu_setup_branch` | ReSukiSU `kernel/setup.sh` 使用的分支（`main` / `dev`） |
| `ksu_meta` | ReSukiSU 源码，格式 `分支/自定义Tag/可选Commit`，默认 `main/⚡Ultra⚡/` |
| `susfs_meta` | SUSFS 版本：留空=最新，提交哈希或 `HEAD~N`=回滚，`-1`=禁用 SUSFS |
| `kernel_suffix` | 仅 Ace 6T：内核后缀，留空=该 GKI 发行版的原厂后缀 |
| `enable_sched_ext` | 仅平板 2 Pro：是否合入 sched_ext |
| `create_release` | 是否发布 GitHub Release |

### 与旧版的差异

1. **移除 KPM**：删除 `CONFIG_KPM=y`、`enable_feature_x` 开关以及
   `SukiSU_KernelPatch_patch` 的 `patch_linux` 修补步骤。
2. **ReSukiSU 集成方式更新**：继续使用官方 `kernel/setup.sh`，
   并新增软链 / Makefile / Kconfig 接入校验；defconfig 增加
   `CONFIG_KSU_MULTI_MANAGER_SUPPORT=y` 与 `CONFIG_KSU_FULL_NAME_FORMAT`。
3. **SUSFS 打补丁改为先 dry-run 再应用**，并自动探测上游版本号写进
   Release 说明；从 `ZIP` 名称即可看出是否带 SUSFS。
4. **平板 2 Pro 清掉过时 SUSFS 配置**：`CONFIG_KSU_SUSFS_SUS_SU`、
   `HAS_MAGIC_MOUNT`、`TRY_UMOUNT`、`AUTO_ADD_*`、`SUS_OVERLAYFS` 等
   在当前 susfs4ksu 中已不存在，继续写会污染 defconfig。
5. **动作版本更新**：`actions/cache@v6`、`actions/upload-artifact@v7`、
   `softprops/action-gh-release@v3`，运行环境改为 `ubuntu-24.04`。
6. **Ace 6T 内核后缀对齐原厂**：默认使用该 GKI 发行版的原厂
   `uname -r` 后缀，并提供 `HEAD~N`/自定义回退。
7. **构建前后校验**：新增关键 defconfig 条目校验与 Job Summary。
8. **SUSFS 关闭时的占位值**：`susfs_meta=-1` 时也会写入 `SUSVER=noSUSFS` /
   `SUSFS_BRANCH_NAME=none`，避免 Release 标题与摘要渲染成空字符串。

### 首次运行请留意

改写未经实际 CI 运行验证（本地环境无法执行 GitHub Actions），建议先手动
`workflow_dispatch` 跑一次，重点确认两处：

- **Ace 6T 的内核后缀**：`apply_suffix` 依赖 `common/scripts/setlocalversion`
  的具体实现来替换版本号拼接逻辑。若该文件与预期不符，最坏情况只是
  `uname -r` 不完全是原厂形态，不会编译失败；确认后可自行调整或传
  `kernel_suffix` 覆盖。
- **SUSFS 补丁**：`gki-android15-6.6` / `gki-android16-6.12` 分支的补丁若
  与应用基线有偏移，工作流会直接失败并提示「SUSFS 补丁无法应用」，
  此时用 `susfs_meta` 传提交哈希或 `HEAD~N` 回滚到可用版本即可。

### 其他工作流文件未有正常测试过，来自上游

> 注：`oneplus_13.yml.disabled`、`oneplus_13T.yml.disabled`、
> `oneplus_ace5_pro.yml.disabled`、`oneplus_ace5_Ultra.yml.disabled` 未在本次
> 改动范围内，其中仍留有旧的 KPM 逻辑（这几个文件处于禁用状态，不参与构建）。

特别鸣谢：GitHub@HanKuCka（酷安@FutabaWa）
