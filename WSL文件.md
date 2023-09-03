### WSL2 mnt目录io缓慢问题
* #### 没有找到简单的处理方案，相比起来回退WSL1是一个好主意
问题的现象是在wsl中/mnt/f/目录执行编译会卡住，发现问题是wsl 访问windows的hdfs文件io巨慢，看了一些帖子是wsl使用了一种网络协议处理和宿主windows的io导致性能相对wsl1满了至少一个数量级别，但是又不想只在wsl文件系统内进行相关操作，所以最后回滚会wsl1了；

```
wsl --shutdown
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
* #### windows目录不区分大小写,会有如下的报错  

```bash
compiler_wrapper.go:250: failed to execute &main.command{Path:"/mnt/f/work_code/android13_kernel1/prebuilts/clang/host/linux-x86/clang-r450784e/bin/clang.real", Args:[]string{"-print-file-name=include"}, EnvUpdates:[]string(nil)}: go_exec.go:22: exec error: exec format error

make[2]: *** /mnt/f/work_code/android13_kernel1/common/Documentation/Kbuild: Is a directory. Stop.

make[1]: *** [/mnt/f/work_code/android13_kernel1/common/Makefile:1960: _clean_Documentation] Error 2

make[1]: Leaving directory '/mnt/f/work_code/android13_kernel1/out/android13-5.15/common'

make: *** [Makefile:244: __sub-make] Error 2

```



