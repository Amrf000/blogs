---
layout: default
title: WSL中使用编译kernel并启用ccache
---


## WSL中使用编译kernel并启用ccache

### WSL2 mnt目录io缓慢问题
* #### 没有找到简单的处理方案，相比起来回退WSL1是一个好主意
问题的现象是在wsl中/mnt/f/目录执行编译会卡住，发现问题是wsl 访问windows的hdfs文件io巨慢，看了一些帖子是wsl使用了一种网络协议处理和宿主windows的io导致性能相对wsl1满了至少一个数量级别，但是又不想只在wsl文件系统内进行相关操作，所以最后回滚会wsl1了；

```
wsl --shutdownbuild/kernel$ ls -al build-tools/path/linux-x86/ccache*

lrwxrwxrwx 1 lyw lyw 57 Sep 3 01:14 build-tools/path/linux-x86/ccache -> ../../../../../prebuilts/build-tools/linux-x86/bin/ccache

lrwxrwxrwx 1 lyw lyw 63 Sep 3 01:17 build-tools/path/linux-x86/ccache-clang -> ../../../../../prebuilts/build-tools/linux-x86/bin/ccache-clang

lrwxrwxrwx 1 lyw lyw 65 Sep 3 01:17 build-tools/path/linux-x86/ccache-clang++ -> ../../../../../prebuilts/build-tools/linux-x86/bin/ccache-clang++

wsl --set-version Ubuntu-20.04 1
wsl --set-default-version 1
wsl -l -v 
wsl
```

* #### 参考：
[How can I revert from WSL 2 to the earlier version of WSL? (Need to remove conflict with VirtualBox)](https://github.com/MicrosoftDocs/WSL/issues/590)

```
dism.exe /online /disable-feature /featurename:VirtualMachinePlatform /norestart
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-Hypervisor
```
* #### 降级后，wsl不支持systemd，需要手启sshd
```
sudo /usr/sbin/sshd
sudo mkdir /run/sshd
```
注释掉wsl.conf中wsl2才支持的配置
```
sudo vi /etc/wsl.conf
#[boot]
#systemd=true
#[automount]
#options = case = dir
```

### WSL mnt目录中kernel编译
* #### 执行步骤
```
# 如果需要，设置git代理
export http_proxy=http://x.x.x.x:1080
export https_proxy=http://x.x.x.x:1080
git config --global http.proxy http://x.x.x.x:1080
git config --global https.proxy http://x.x.x.x:1080
# 拉取aosp kernel代码
repo init -u https://android.googlesource.com/kernel/manifest -b common-android13-5.15 --depth=1
repo sync --force-sync -j24 --current-branch --no-tags --no-clone-bundle --optimized-fetch --prune
# 创建本地分支
repo start br_local --all
# 尝试编译
BUILD_CONFIG=common-modules/virtual-device/build.config.virtual_device.x86_64 build/build.sh -j24
```
* #### windows目录不区分大小写,会有如下的报错  

```bash
compiler_wrapper.go:250: failed to execute &main.command{Path:"/mnt/f/work_code/android13_kernel1/prebuilts/clang/host/linux-x86/clang-r450784e/bin/clang.real", Args:[]string{"-print-file-name=include"}, EnvUpdates:[]string(nil)}: go_exec.go:22: exec error: exec format error

make[2]: *** /mnt/f/work_code/android13_kernel1/common/Documentation/Kbuild: Is a directory. Stop.

make[1]: *** [/mnt/f/work_code/android13_kernel1/common/Makefile:1960: _clean_Documentation] Error 2

make[1]: Leaving directory '/mnt/f/work_code/android13_kernel1/out/android13-5.15/common'

make: *** [Makefile:244: __sub-make] Error 2

```

>* 解决方案：可以以管理员权限打开powershell，cd到目标代码目录，执行

```
(Get-ChildItem -Recurse -Directory).FullName | ForEach-Object {if (-Not ($_ -like '*node_modules*')) { fsutil.exe file setCaseSensitiveInfo $_ enable } }
```
wsl1中执行上面的设置应该就可以了，wsl2中可能需要修改wsl.conf之后重启wsl
```
[automount]
options = case = dir
```
* #### wsl1编译kernel报错  
clang 14 exec format error  

>* 解决方案：

```
sudo patchelf --set-interpreter /lib64/ld-linux-x86-64.so.2 /mnt/f/work_code/android13_kernel1/prebuilts/clang/host/linux-x86/clang-r450784e/bin/clang.real

```

### 启用CCache功能
```
export CCACHE_DIR=/mnt/f/work_code/android_ccache
export CCACHE_DEBUG=true
export CCACHE_DEBUGDIR=/mnt/f/work_code/android_ccache_debug/
export PATH=$PATH:/mnt/f/work_code/android13_kernel1/build/kernel/build-tools/path/linux-x86
```
vi /mnt/f/work_code/android_ccache/ccache.conf

```
# Set maximum cache size to 10 GB:
max_size = 10G
```
代码修改patch,cd build/kernel
```
git diff _setup_env.sh
diff --git a/_setup_env.sh b/_setup_env.sh
index 6d082bd..541bdcf 100644
--- a/_setup_env.sh
+++ b/_setup_env.sh
@@ -199,9 +199,9 @@ if [[ -n "${LLVM}" ]]; then
   # Reset a bunch of variables that the kernel's top level Makefile does, just
   # in case someone tries to use these binaries in this script such as in
   # initramfs generation below.
-  HOSTCC=clang
-  HOSTCXX=clang++
-  CC=clang
+  HOSTCC=ccache-clang
+  HOSTCXX=ccache-clang++
+  CC=ccache-clang
   LD=ld.lld
   AR=llvm-ar
   NM=llvm-nm

```
添加ccache工具链接到prebuilt仓
```
build/kernel$ ls -al build-tools/path/linux-x86/ccache*

lrwxrwxrwx 1 liuyawu liuyawu 57 Sep 3 01:14 build-tools/path/linux-x86/ccache -> ../../../../../prebuilts/build-tools/linux-x86/bin/ccache

lrwxrwxrwx 1 liuyawu liuyawu 63 Sep 3 01:17 build-tools/path/linux-x86/ccache-clang -> ../../../../../prebuilts/build-tools/linux-x86/bin/ccache-clang

lrwxrwxrwx 1 liuyawu liuyawu 65 Sep 3 01:17 build-tools/path/linux-x86/ccache-clang++ -> ../../../../../prebuilts/build-tools/linux-x86/bin/ccache-clang++
```
common仓修改,Makefile修改
```
/mnt/f/work_code/android13_kernel1/common$ git diff Makefile

diff --git a/Makefile b/Makefile

index ab26272cd..20d0735bd 100644

--- a/Makefile

+++ b/Makefile

@@ -448,9 +448,11 @@ HOST_LFS_CFLAGS := $(shell getconf LFS_CFLAGS 2>/dev/null)

 HOST_LFS_LDFLAGS := $(shell getconf LFS_LDFLAGS 2>/dev/null)

 HOST_LFS_LIBS := $(shell getconf LFS_LIBS 2>/dev/null)

+#$(error "#########$(PATH)")

+

 ifneq ($(LLVM),)

-HOSTCC = clang

-HOSTCXX = clang++

+HOSTCC = ccache-clang

+HOSTCXX = ccache-clang++

 else

 HOSTCC = gcc

 HOSTCXX = g++

@@ -470,7 +472,7 @@ KBUILD_HOSTLDLIBS := $(HOST_LFS_LIBS) $(HOSTLDLIBS)

 # Make variables (CC, etc...)

 CPP = $(CC) -E

 ifneq ($(LLVM),)

-CC = clang

+CC = ccache-clang

 LD = ld.lld

 AR = llvm-ar

 NM = llvm-nm

```
scripts/Kconfig.include
```
common$ git diff scripts/Kconfig.include

diff --git a/scripts/Kconfig.include b/scripts/Kconfig.include

index 0496efd6e..7cc4285ef 100644

--- a/scripts/Kconfig.include

+++ b/scripts/Kconfig.include

@@ -36,7 +36,7 @@ ld-option = $(success,$(LD) -v $(1))

 as-instr = $(success,printf "%b\n" "$(1)" | $(CC) $(CLANG_FLAGS) -c -x assembler -o /dev/null -)

 # check if $(CC) and $(LD) exist

-$(error-if,$(failure,command -v $(CC)),compiler '$(CC)' not found)

+#$(error-if,$(failure,command -v $(CC)),compiler '$(CC)' not found)

 $(error-if,$(failure,command -v $(LD)),linker '$(LD)' not found)

 # Get the compiler name, version, and error out if it is not supported.

```
prebuilts/build-tools添加ccache二进制

```
 linux-x86/bin/ccache
 linux-x86/bin/ccache-clang
 linux-x86/bin/ccache-clang++
```
ccache为4.83的版本amd64的可执行程序，记住添加可执行
ccache-clang是一个wrapper脚本，记住添加可执行
```
#!/bin/bash
exec ccache clang "$@"
```
修改完成即可使用ccache编译kernel，使用ccache -s -p查看命中信息

