# ImmortalWrt 固件在线构建 · GL.iNet GL-MT3600BE

这个仓库**不存源码，只放 GitHub Actions 工作流**。它的唯一作用：在 GitHub 的机器上云端编译
[chasey-dev/immortalwrt-mt798x-rebase](https://github.com/chasey-dev/immortalwrt-mt798x-rebase)
（ImmortalWrt 25.12 + MediaTek MTK OpenWrt Feeds 闭源驱动补丁），编完直接在 Actions 页面下载固件。

- 不需要自己的电脑装 Linux、不需要 30GB 磁盘、不需要跑 3 小时
- 不需要 fork 上游仓库，也不会碰上游代码
- 配置全部通过 Run workflow 的参数调整，不用改文件

> 固件来源是上游仓库，本仓库只负责 CI，不修改任何源码。

<!--
想加构建状态徽章：把 <用户名> 和 <仓库名> 换成你自己的，再删掉这行的注释符号
![Wi-Fi 版](https://github.com/<用户名>/<仓库名>/actions/workflows/build-mt3600be.yml/badge.svg)
![有线版](https://github.com/<用户名>/<仓库名>/actions/workflows/build-mt3600be-wired.yml/badge.svg)
-->

## 目录

- [支持的机型](#支持的机型)
- [两个版本](#两个版本)
- [快速开始](#快速开始)
- [编译参数](#编译参数)
- [拿到固件之后怎么刷](#拿到固件之后怎么刷)
- [已知问题与注意事项](#已知问题与注意事项)
- [常见问题](#常见问题)
- [进阶用法](#进阶用法)
- [目录结构](#目录结构)
- [免责声明](#免责声明)

## 支持的机型

默认打的就是这台，本仓库所有 workflow 的默认参数都指向它：

| 项目        | 规格                                                         |
| ----------- | ------------------------------------------------------------ |
| 型号        | GL.iNet GL-MT3600BE（Beryl 7）                               |
| SoC         | MediaTek MT7987A，4×Cortex-A53 @ 2.0GHz                      |
| 无线        | MT7990 / MT7993，双频 Wi-Fi 7，BE3600（2.4G 688 + 5G 2882 Mbps） |
| 内存 / 闪存 | DDR4 512MB / SPI-NAND 512MB                                  |
| 网口        | 2.5G WAN ×1 + 2.5G LAN ×1（内建 i2p5gbe PHY）                |
| 其它        | USB 3.0、PWM 风扇、Type-C 供电                               |

SoC 同为 MT7987/MT7981/MT7986 的其它机型，只要上游有对应的 device profile，改几个参数就能编，见
[进阶用法 · 换机型](#换机型)。

## 两个版本

| 工作流                     | 版本         | 无线                                           | 适合场景                | 产物名                                       |
| -------------------------- | ------------ | ---------------------------------------------- | ----------------------- | -------------------------------------------- |
| `build-mt3600be.yml`       | **Wi-Fi 版** | MTK 闭源 `mt_wifi7`（MT7993）                  | 当无线路由器用          | `immortalwrt-glinet_gl-mt3600be-<run>`       |
| `build-mt3600be-wired.yml` | **有线版**   | 整套移除（驱动 + `wpad`/`hostapd` + 配置工具） | 当纯有线路由器 / 主路由 | `immortalwrt-glinet_gl-mt3600be-wired-<run>` |

两版的共同点：

- 内核 6.12，ImmortalWrt 25.12 分支
- **保留 HNAT / PPE 硬件加速**（`kmod-mediatek_hnat`、`mtkhnat_util`、`luci-app-turboacc-mtk`）
- 保留 2.5G PHY 固件（`kmod-mt798x-2p5g-phy`、`mt798x-2p5g-phy-firmware-internal`）
- 保留风扇控制、USB 3.0、zram、TCP BBR
- 使用 512MB 低内存精简配置（`defconfig/low-mem-512m/mt7987-mt7993-be3600.config`）

有线版额外做了两道保险：`make defconfig` 后立即校验无线包是否真的关掉（没关掉就提前失败，不等 3 小时），
编译完成后再用固件内的 `manifest` 复核一遍。

## 快速开始

### 方式一：网页操作（推荐）

1. 用 GitHub 网页新建一个仓库，**建议选 Public**（公开仓库的 Actions 分钟数不计费）
   仓库名随意，例如 `mt3600be-firmware`
2. 在仓库里新建文件 `.github/workflows/build-mt3600be.yml`（注意是三层路径，
   在网页上输入文件名时直接把 `.github/workflows/build-mt3600be.yml` 整段粘进去即可），
   内容粘贴本仓库里的 `build-mt3600be.yml`
3. 想要有线版就再来一次，文件名换成 `.github/workflows/build-mt3600be-wired.yml`
4. 打开仓库顶部的 **Actions**，左侧会看到
   `Build ImmortalWrt (GL-MT3600BE)` 和 `Build ImmortalWrt (GL-MT3600BE · 有线版)`
5. 点右侧 **Run workflow** → 分支选 `main`（默认分支）→ **Run workflow**
6. 等跑完（页面顶部会显示绿勾），在 run 页面底部 **Artifacts** 里下载 zip

> ⚠️ workflow 文件必须放在**默认分支**上，否则左侧列表里不会出现 `Run workflow` 按钮。
> 但源码分支（`source_branch`）可以随便选，两者互不影响。

### 方式二：命令行

```bash
# 1. 建一个空仓库（网页建更简单），然后
git clone https://github.com/<你的用户名>/mt3600be-firmware.git
cd mt3600be-firmware

# 2. 放入两个 workflow
mkdir -p .github/workflows
cp /path/to/build-mt3600be.yml       .github/workflows/
cp /path/to/build-mt3600be-wired.yml .github/workflows/

# 3. 推送
git add .github/workflows
git commit -m "ci: add firmware build workflows"
git push origin main
```

推送后回到网页 Actions 页面手动触发即可。

## 编译参数

每次 **Run workflow** 时可以填这些参数（不填就用默认值）：

| 参数             | 默认值                                                       | 说明                                                         |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `source_repo`    | `https://github.com/chasey-dev/immortalwrt-mt798x-rebase.git` | 源码仓库。想编自己的 fork 就换成自己的 URL                   |
| `source_branch`  | `25.12-dev-wifi7`                                            | 源码分支。**Wi-Fi 7 支持只在这个分支**，`25.12` 是主线稳定分支 |
| `defconfig`      | `defconfig/low-mem-512m/mt7987-mt7993-be3600.config`         | 基础配置在源码树里的相对路径                                 |
| `device`         | `glinet_gl-mt3600be`                                         | 设备 profile 名（对应 `target_config` 里的 `CONFIG_TARGET_DEVICE_mediatek_filogic_DEVICE_<名字>`） |
| `extra_packages` | 空                                                           | 额外软件包，空格分隔，例如 `luci-app-ttyd luci-app-sqm`      |
| `verbose`        | `false`                                                      | 打开后加 `V=s` 输出详细日志，编译更慢但排错有用              |

### 为什么会话不用 fork

跨仓库 checkout 时默认的 `GITHUB_TOKEN` 只有当前仓库的权限，容易报 `repository not found`，要配 PAT 才行。
所以 workflow 直接用 `git clone` 匿名拉取公共仓库，既不需要 PAT，也不需要 fork —— 你改 `source_repo`
就能编译任何公开仓库。

## 拿到固件之后怎么刷

Artifact 解压后主要看这一个文件：

```
immortalwrt-mediatek-filogic-glinet_gl-mt3600be-squashfs-sysupgrade.bin
```

同目录还有 `.manifest`（固件内实际安装的包清单，可用来核对）和 `sha256sums`（校验用）。

### 刷机步骤

1. **先备份**：进原厂 GL.iNet 系统 `ssh root@192.168.8.1`，把 Factory/art 分区导出备份。
   无线校准数据和 MAC 地址在里面，丢了很难恢复。
2. **进 U-Boot 恢复模式**：断电，按住 Reset，插电，保持约 5 秒松开，
   电脑网卡设为 DHCP，浏览器打开 `http://192.168.1.1`
3. **上传固件**：用 U-Boot 页面的固件更新功能上传上面的 `sysupgrade.bin`
4. 等待自动重启（约 2–3 分钟），默认管理地址 `192.168.1.1`，无密码

> ⚠️ **不要在 GL.iNet 原厂 Web UI 里直接升级**。本项目只产出 `sysupgrade.bin`，没有 `factory.bin`，
> 原厂 UI 不认这种镜像。首次刷入必须走 U-Boot 恢复页。

想刷回原厂固件，同样用 U-Boot 恢复页上传 GL.iNet 官方镜像即可。

## 已知问题与注意事项

### 1. LAN 口重启后随机不通（2.5G PHY 固件缺失）

上游 [#101](https://github.com/chasey-dev/immortalwrt-mt798x-rebase/issues/101)：
`kmod-mt798x-2p5g-phy` 和 `mt798x-2p5g-phy-firmware-internal` 如果只编译成模块（`M`）而没装进固件，
LAN 口会随机没连接。本仓库的两个 workflow 都显式把这两个包设为 `=y`，并清掉了可能把
`DEVICE_PACKAGES` 覆盖成空的配置项，正常情况下不会踩到。

如果你自己改了配置，刷完记得确认：

```sh
apk list | grep 2p5g
```

### 2. 5G 160MHz 需要等 DFS CAC

上游 [#110](https://github.com/chasey-dev/immortalwrt-mt798x-rebase/issues/110)：
MT7993 默认配置 `/etc/wireless/mediatek/mt7993.b1.dat` 里 `DfsEnable=1`，开启 160MHz 要等 DFS 检测
（约 1 分钟起）。上游作者明确表示不会默认关掉 DFS（法规要求），想改只能自己 fork 改模板。

### 3. 有线版的两个副作用

- LuCI 里的 **Wireless 菜单会消失**，`/etc/config/wireless` 也不会生成，属正常现象
- DTS 里的 PCIe `mt7990` 节点仍然存在但没有驱动，启动日志可能有一条未绑定 PCIe 设备，**不影响功能**

### 4. 不要勾选 `kmod-mt76*`

这个 fork 为了用 MTK 官方闭源驱动 + HNAT，已经把 `package/kernel/mt76` 整个移除了，
无线走 `mt_wifi` / `mt_wifi7`。如果手动勾了任何 `kmod-mt76*` 或 `kmod-mac80211`，会冲突或编不过。

### 5. 只有 512MB 内存

`extra_packages` 别塞太多。塞到内存不够，轻则 OOM、重则无线驱动加载失败。

## 常见问题

<details>
<summary><b>编译要多久？</b></summary>


- 首次（无缓存）：**2.5–4 小时**
- 之后（ccache 命中）：**40–70 分钟**
- `dl/` 源码包缓存命中还能再省 10 分钟左右
  </details>

<details>
<summary><b>会消耗 Actions 额度吗？</b></summary>


- **Public 仓库：不计量分钟数**，随便跑（受并发数限制）
- **Private 仓库：按分钟计费**，一次约 150–250 分钟。免费额度 2000 分钟/月 ≈ 跑 8–13 次

另外 GitHub 单个 job 硬上限是 **6 小时**，本仓库设的是 350 分钟超时。
</details>

<details>
<summary><b>失败了怎么查？</b></summary>


1. 看失败的那个步骤的日志；失败的 run 会自动上传 `build-log-<run号>` artifact，里面有完整 `build.log`
2. 重新 Run 一次，把 `verbose` 设为 `true`，日志会详细很多
3. `make download` 阶段偶发失败（GitHub 限流）通常重跑就好，workflow 里已经加了一次重试
   </details>

<details>
<summary><b>Actions 页面看不到 Run workflow 按钮？</b></summary>


workflow 文件没在默认分支上。把它提交到 `main`（或你的默认分支）即可。
</details>

<details>
<summary><b>固件跑完没用 / 编译成功但无线不工作？</b></summary>


先看 manifest 里 `kmod-mt_wifi7`、`kmod-mt_hwifi`、`wifi-profile`、`wpad-openssl` 是否都在。
Wi-Fi 版缺了这些说明配置被覆盖了。另外确认 `source_branch` 是 `25.12-dev-wifi7`——`25.12` 分支的
MT3600BE profile 还在引用这个 fork 已移除的旧包名。
</details>

<details>
<summary><b>能同时编两个版本吗？</b></summary>


可以，它们是两个独立 workflow。但两者共用同一个 `concurrency` 组，**不会同时跑**，
避免两个 3 小时的重编译抢资源、写坏同一份缓存。
</details>

## 进阶用法

### 加软件包

Run workflow 时在 `extra_packages` 填，空格分隔：

```
luci-app-ttyd luci-app-sqm luci-app-wireguard luci-app-ddns
```

包里想关掉某个默认包，可以在 workflow 的 `生成 .config` 步骤里加 `# CONFIG_PACKAGE_<包名> is not set`。

### 换机型

三处一起改（在 Run workflow 的参数里填，或改 workflow 默认值）：

```
defconfig = defconfig/low-mem-512m/mt7987-mt7993-be3600.config   # 换成目标机型对应的
device    = glinet_gl-mt3600be                                   # 换成对应的 profile 名
```

profile 名可以在源码树里查：

```bash
grep -rh 'define Device/' target/linux/mediatek/image/*.mk | sed 's/define Device\///'
```

注意同一个机型的 `device` 名要和 image 定义、defconfig 三者一致，否则会编出空包。

### 编自己的改动

1. 把上游 Fork 一份，在 `25.12-dev-wifi7` 分支上改
2. Run workflow 时把 `source_repo` 换成你自己的 URL：
   `https://github.com/<你的用户名>/immortalwrt-mt798x-rebase.git`

workflow 文件不用动。

### 使用自己维护的 `.config`

默认从源码树里取 defconfig。如果想把这个 `.config` 放在本仓库里版本化，
需要多加一个 checkout 步骤（把本仓库拉到 runner 上）：

```yaml
      - name: 取本仓库的配置
        uses: actions/checkout@v7
        with:
          path: ci

      # 然后生成 .config 的那一步里改成：
      #   cp "$GITHUB_WORKSPACE/ci/configs/mt3600be.config" .config
```

### 打标签自动发 Release

两个 workflow 都配了 tag 触发，产物会挂到 Release 上：

```bash
git tag v1.0            # 触发 Wi-Fi 版
git push origin v1.0

git tag v1.0-wired      # 触发有线版
git push origin v1.0-wired
```

tag 触发时 `source_branch` 参数为空，会用默认的 `25.12-dev-wifi7`，也就是「发版用最新代码」。

### 想改成每天定时自动编

在 workflow 的 `on:` 下加：

```yaml
  schedule:
    - cron: "0 18 * * 5"   # 每周五 UTC 18:00，约北京时间周六 02:00
```

## 目录结构

```
.
├── .github/
│   └── workflows/
│       ├── build-mt3600be.yml         # Wi-Fi 版
│       └── build-mt3600be-wired.yml   # 有线版（去掉无线）
└── README.md
```

源码不会被克隆进本仓库——运行时 `git clone --depth 1` 到 runner 的临时目录，跑完即销毁。

## 致谢

- 固件来源：[chasey-dev/immortalwrt-mt798x-rebase](https://github.com/chasey-dev/immortalwrt-mt798x-rebase)
  （ImmortalWrt + MTK OpenWrt Feeds 补丁）
- 上游项目：[ImmortalWrt](https://github.com/immortalwrt/immortalwrt)、[MTK OpenWrt Feeds](https://github.com/mediatek/mtk-openwrt-feeds)
- 磁盘清理：[jlumbroso/free-disk-space](https://github.com/jlumbroso/free-disk-space)
- 机型规格：[GL.iNet 官方产品页](https://www.gl-inet.com/products/gl-mt3600be)

## 免责声明

- 固件由社区维护的上游仓库构建，**不是 GL.iNet 或 ImmortalWrt 官方发布**
- 无线驱动为 MediaTek 闭源二进制，遵循其各自许可
- **刷机有风险**：变砖、数据丢失、失去保修均需自行承担。刷前务必备份 Factory/art 分区
- 请遵守当地无线电法规，尤其是 DFS / 发射功率相关限制
