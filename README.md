# PonWrt

PonWrt 基于 [ImmortalWrt](https://github.com/immortalwrt/immortalwrt)，增加了对 Airoha AN7581 和 AN7583 PON 设备的支持。

## ⚠️ 免责声明

PonWrt 是一个用于研究和开发的开源光猫固件项目。

刷写固件或修改 PON 相关配置存在风险，可能导致设备无法启动、配置或设备数据丢失、PON 无法注册等问题。操作前请务必备份原厂固件及设备相关数据。

使用者应自行确保相关操作符合当地法律法规及运营商相关规定。请勿将本项目用于未经授权的网络接入、冒用或复制他人设备身份，或干扰运营商网络正常运行。

因刷写、配置或使用本项目产生的设备故障、网络服务异常及其他后果，由使用者自行承担。

## 支持设备

| SoC | 设备 | Profile | 原厂校准/身份数据分区 |
| --- | --- | --- | --- |
| AN7581 | FiberHome HG5382A | `fiberhome_hg5382a` | `factory` |
| AN7581 | FiberHome HG5585F CT | `fiberhome_hg5585f-ct` | `factory` |
| AN7581 | FiberHome HG5585F CU | `fiberhome_hg5585f-cu` | `factory` |
| AN7581 | Gemtek XG2010G | `gemtek_xg2010g` | `dsd` |
| AN7581 | Nokia XG-040G-MD UBI | `nokia_xg-040g-md-ubi` | `bosa`、`ri` |
| AN7581 | Nokia XG-040G-TF UBI | `nokia_xg-040g-tf-ubi` | `bosa`、`ri` |
| AN7581 | UnionMan UNG00A | `unionman_ung00a` | `reservearea` |
| AN7581 | ZNXT ZN504XG-D | `znxt_zn504xg-d` | `reservearea` |
| AN7581 | ZNXT ZN515XG-D | `znxt_zn515xg-d` | `reservearea` |
| AN7581 | H3C HM2004-DU | `h3c_hm2004-du` | `factory` |
| AN7583 | Nokia XG-040G-MF | `nokia_xg-040g-mf`、`nokia_xg-040g-mf-ubi` | `bosa`、`ri` |

## 编译

```sh
#  安装编译所需的工具链和库
sudo apt update -y
sudo apt full-upgrade -y
sudo apt install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
  bzip2 ccache clang cmake cpio curl device-tree-compiler ecj fastjar flex gawk gettext gcc-multilib \
  g++-multilib git gnutls-dev gperf haveged help2man intltool lib32gcc-s1 libc6-dev-i386 libelf-dev \
  libglib2.0-dev libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev libncurses-dev libpython3-dev \
  libreadline-dev libssl-dev libtool libyaml-dev libz-dev lld llvm lrzsz mkisofs msmtp nano \
  ninja-build p7zip p7zip-full patch pkgconf python3 python3-pip python3-ply python3-docutils \
  python3-pyelftools qemu-utils re2c rsync scons squashfs-tools subversion swig texinfo uglifyjs \
  upx-ucl unzip vim wget xmlto xxd zlib1g-dev zstd

#  或者使用ImmortalWrt一键脚本安装依赖
sudo bash -c 'bash <(curl -s https://build-scripts.immortalwrt.org/init_build_environment.sh)'

#  拉取源码（用 master 分支与上游仓库保持同步）
git clone -b master https://github.com/DenzelVicent/ponwrt.git
cd ponwrt

#  更新并安装 feeds（软件包源）
./scripts/feeds update -a
./scripts/feeds install -a

#  第三方插件：选配置 A 时必须克隆（这两个包不在任何 feed 内，缺了会被静默跳过、不报错）；选配置 B 时按需
#  顺序要求：必须在 feeds install 之后执行（此时 luci-base 等 feed 包已就位，插件依赖才能被解析）
git clone --depth 1 https://github.com/gdy666/luci-app-lucky.git package/luci-app-lucky
git clone --depth 1 https://github.com/DenzelVicent/luci-app-airoha-npu.git package/luci-app-airoha-npu

#  选择配置（A / B 二选一；两者都已默认选中 kmod-airoha-xpon / kmod-airoha-pon-frontend /
#  luci-app-pon / airoha-ponctl·pond·pon-debug，不用再进 menuconfig）
#  A. 复现 GPON 云编译产物：只编 nokia_xg-040g-md-ubi 一个 profile，PON 组件与第三方界面按 CI 全量选中
cp an7581-gpon.config .config
#  B. （可选）编译上游支持的全部 AN7581 设备：an7581.config 选了 12 个 profile，耗时远长于单设备
#     目标是 AN7583 就把下面的 an7581 换成 an7583
# cp configs/an7581.config .config
# ./scripts/kconfig.pl + configs/an7581.config configs/release.config > .config


#  开始编译
# -j$(nproc) 表示用所有 CPU 核心并行编译，加快速度
make defconfig
make download -j8 V=s
make -j$(nproc)


# 修改补丁后可快速编译内核后打包
# 1. 只重建内核（重解压 + 重打 patch + 重编）
make target/linux/clean
make target/linux/compile -j$(nproc)
# 2. 生成固件镜像（打包 kernel + rootfs）
make -j$(nproc)
```

固件位于 `bin/targets/airoha/an7581/` 或 `bin/targets/airoha/an7583/`。

## 刷入

使用 [AN758x-Stock2UBI](https://github.com/pbs05/an758x-stock2ubi) 备份原厂闪存并安装 UBI 布局。启动镜像和 Web 恢复界面由 [AN758x U-Boot](https://github.com/pbs05/uboot-an758x) 提供。

刷入 PonWrt 后，通过 U-Boot Web 或 LuCI 的"网络 → PON → 配置 → PON board data"恢复原厂校准和身份数据。烽火 `factory` 需要先使用 [FiberHome Factory](https://github.com/pbs05/fiberhome-factory) 转换；转换后的烽火数据、`reservearea` 和 `dsd` 写入 PonWrt 的 `factory` 卷；Nokia 的 `bosa` 和 `ri` 写入同名卷。
