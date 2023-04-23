> 记录一下编译系统内核踩过的坑

# 环境
- 编译系统：Ubuntu 18.04/20.04 WSL1（不同设备不同系统的编译系统有差距，如果后续有环境类的报错可以尝试换系统）
- 设备型号：Pixel3/ Pixel6
- 系统版本：
  - Pixel3: Android 12, kernel 4.9
  - Pixel6: Android 12, kernel 5.10

# 正式编译
## 编译环境安装
主要参考  [官方文档-搭建构建系统](https://source.android.com/docs/setup/start/initializing)
```bash
# install repo
mkdir ~/bin
curl http://gerrit.googlesource.com/git-repo/repo > ~/bin/repo
chmod a+x ~/bin/repo
echo "PATH=~/bin/repo:$PATH"
sudo ln -s /usr/bin/python3  /usr/bin/python

# install essential build environment
sudo apt-get update
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig

# not include in google doc, but it's neccesary
sudo apt-get install libssl-dev bc

```


## 下载源代码
两种方法取其一即可，随着Android源码仓库大小越来越大，推荐使用国内源对其进行下载，主要参考 [官方文档-构建内核](https://source.android.com/docs/setup/build/building-kernels)，确保代码完全同步成功

### 使用代理
```bash
export http_proxy=http://192.168.1.13:7890
export https_proxy=http://192.168.1.13:7890

# change your branch
mkdir ~/code/android-kernel && cd ~/code/android-kernel
repo init -u https://android.googlesource.com/kernel/manifest -b android-msm-crosshatch-4.9-android12

repo sync
```
### 使用国内源
```bash
# replace repo domain, refer to https://www.cooluc.com/archives/1170.html
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
git config --global url.https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/.insteadof https://android.googlesource.com

# change your branch
mkdir ~/code/android-kernel && cd ~/code/android-kernel
repo init -u https://aosp.tuna.tsinghua.edu.cn/kernel/manifest -b android-gs-raviole-5.10-android12L

repo sync
```

## 编译
**建议在编译前阅读 ``buid/build.sh`` 前面的注释部分，包含所有编译脚本支持的参数及参数解释。**

### 正常编译
通常源码根目录下有一个文件 ``build_xxx.sh`` （软连接指向真正运行编译的脚本），直接运行编译即可

#### Pixel 3
```bash
# It's a simple way to build kernel system, if you want to modify source code, you must read buid/build.sh.
./build_bluecross.sh -j8

# build kernel project without clean.
SKIP_MRPROPER=1 ./build_bluecross.sh -j8
```

#### Pixel 6
```bash
# It's a simple way to build kernel system, if you want to modify source code, you must read buid/build.sh.
LTO=full BUILD_KERNEL=1 ./build_slider.sh -j8

# build kernel project without clean.
SKIP_MRPROPER=1 LTO=full BUILD_KERNEL=1 ./build_slider.sh -j8
```

### 编译BOOT镜像
正常编译出来的产物是 ``Image.lz4``，并非可以直接刷机的 ``boot.img``，如果需要编译出直接能刷机的镜像，需要 ``BUILD_BOOT_IMG`` 参数对齐进行打包，具体可以阅读 ``buid/build.sh`` 前半段注释部分。

#### Pixel3
参考 https://blog.seeflower.dev/archives/174/

在 ``build/build.sh`` 中 ``echo " Files copied to ${DIST_DIR}"`` 之前加上复制命令  
```bash
if [ -f "${VENDOR_RAMDISK_BINARY}" ]; then
  cp ${VENDOR_RAMDISK_BINARY} ${DIST_DIR}
fi
```  

下载原始镜像文件解析镜像信息，并编译
```bash
# download image from https://developers.google.cn/android/images
mkdir ~/images && cd ~/images
curl https://dl.google.com/dl/android/aosp/blueline-sp1a.210812.016.b2-factory-6eeb77ae.zip --output blueline-sp1a.210812.016.b2-factory-6eeb77ae.zip
unzip blueline-sp1a.210812.016.b2-factory-6eeb77ae.zip
cd blueline-sp1a.210812.016.c1
unzip image-blueline-sp1a.210812.016.c1.zip

# unpack boot.image info
cd ~/code
git clone https://android.googlesource.com/platform/system/tools/mkbootimg
cd ~/code/mkbootimg
python unpack_bootimg.py --boot_img ~/images/blueline-sp1a.210812.016.c1/boot.img

# get boot.img info
# boot magic: ANDROID!
# kernel_size: 19835242
# kernel load address: 0x00008000
# ramdisk size: 14206170
# ramdisk load address: 0x01000000
# second bootloader size: 0
# second bootloader load address: 0x00000000
# kernel tags load address: 0x00000100
# page size: 4096
# os version: 12.0.0
# os patch level: 2021-10
# boot image header version: 2
# product name:
# command line args: console=ttyMSM0,115200n8 androidboot.console=ttyMSM0 printk.devkmsg=on msm_rtb.filter=0x237 ehci-hcd.park=3 service_locator.enable=1 cgroup.memory=nokmem lpm_levels.sleep_disabled=1 usbcore.autosuspend=7 loop.max_part=7 androidboot.boot_devices=soc/1d84000.ufshc androidboot.super_partition=system buildvariant=user
# additional command line args:
# recovery dtbo size: 0
# recovery dtbo offset: 0x0000000000000000
# boot header size: 1660
# dtb size: 863100
# dtb address: 0x0000000001f00000

cp out/ramdisk ~/code/android-kernel

# pls modify params to be the same as unpack_bootimg output 
BUILD_BOOT_IMG=1 MKBOOTIMG_PATH=../mkbootimg/mkbootimg.py VENDOR_RAMDISK_BINARY=ramdisk KERNEL_BINARY=Image.lz4 BOOT_IMAGE_HEADER_VERSION=2 KERNEL_CMDLINE="console=ttyMSM0,115200n8 androidboot.console=ttyMSM0 printk.devkmsg=on msm_rtb.filter=0x237 ehci-hcd.park=3 service_locator.enable=1 cgroup.memory=nokmem lpm_levels.sleep_disabled=1 usbcore.autosuspend=7 loop.max_part=7 androidboot.boot_devices=soc/1d84000.ufshc androidboot.super_partition=system buildvariant=user" BASE_ADDRESS=0x00000000 PAGE_SIZE=4096 ./build_bluecross.sh

```

#### Pixel6  
Pixel6 ``boot.img`` 文件格式已经和 Pixel3 不同，打包 ``boot.img`` 的方式也不相同。可以通过上面解包的方式确定你需要刷机的 ``boot.img`` 格式版本，并根据不同的版本传入不同的参数进行打包。
> Pixel6: BOOT_IMAGE_HEADER_VERSION=4
> Pixel3: BOOT_IMAGE_HEADER_VERSION=2


```bash
# download image from https://developers.google.cn/android/images
mkdir ~/images && cd ~/images
curl https://dl.google.com/dl/android/aosp/oriole-sq3a.220705.003-factory-4ec7503a.zip --output oriole-sq3a.220705.003-factory-4ec7503a.zip
unzip oriole-sq3a.220705.003-factory-4ec7503a.zip
cd oriole-sq3a.220705.003
unzip image-oriole-sq3a.220705.003.zip


# get boot info, 
cd ~/code
git clone https://android.googlesource.com/platform/system/tools/mkbootimg
cd ~/code/mkbootimg
python unpack_bootimg.py --boot_img ~/images/oriole-sq3a.220705.003/boot.img
cp out/ramdisk ~/code/android-kernel

SKIP_MRPROPER=1 BUILD_BOOT_IMG=1 KERNEL_BINARY=Image.lz4 VENDOR_RAMDISK_BINARY=ramdisk BOOT_IMAGE_HEADER_VERSION=4 LTO=full BUILD_KERNEL=1 ./build_slider.sh
```

### Patch Kernel
以 [rwProcMem33](https://github.com/abcz316/rwProcMem33) 为例子，对内核代码进行修改并重新编译


#### Pixel3
```bash
cd ~/code
git clone https://github.com/abcz316/rwProcMem33

# Pixel3 source code path
cp ~/code/rwProcMem33 ~/code/android-kernel/private/msm-google/drivers
cp ~/code/hwBreakpointProc ~/code/android-kernel/private/msm-google/drivers

# add build params into Makefile
#   obj-m += rwProcMem37/
#   obj-m += hwBreakpointProc1/

# You need fix build error caused by rwProcMem33

./build_bluecross.sh -j8
```


#### Pixel6
```bash
# Pixel6 source code path
cp -r ~/code/rwProcMem33 ~/code/android-kernel/aosp/drivers/rwProcMem37
cp -r ~/code/hwBreakpointProc ~/code/android-kernel/aosp/drivers/hwBreakpointProc1

# add build params into Makefile
#   obj-m += rwProcMem37/
#   obj-m += hwBreakpointProc1/

# You need fix build error caused by rwProcMem33

# You need set KMI_SYMBOL_LIST_STRICT_MODE to skip abi check or use build_abi.sh to update abi info. For more information, pls read https://source.android.com/docs/core/architecture/kernel/abi-monitor 
KMI_SYMBOL_LIST_STRICT_MODE=0 LTO=full BUILD_KERNEL=1 ./build_slider.sh -j8

```


# 其他的一些坑
## 使用 Mac Docker 编译报错
由于 Mac 磁盘默认不区分大小写，导致编译部分代码时找不到文件，解决方案为创建一块区分大小写的磁盘
> 磁盘工具 --> 右上角 "+" 号(新建) --> 格式选择区分大小 --> 在创建的磁盘路径中编译代码

# 参考文章
https://blog.seeflower.dev/archives/174/  
https://www.r4ve1.com/p/brief-android-kernel-exploitation/  
https://github.com/abcz316/rwProcMem33  
