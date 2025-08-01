## kpatch(1)&#58; 偷偷给内核动手术 - 在Debian上打热补丁

- [环境准备](#环境准备)
  - [环境检查](#环境检查)
  - [安装依赖](#安装依赖)
  - [准备内核源代码](#准备内核源代码)
  - [创建内核配置](#创建内核配置)
- [创建patch](#创建patch)
- [测试patch](#测试patch)


实时补丁可在不重启系统的情况下更新Linux内核,常用于修复严重漏洞.

本教程演示如何用`kpatch`修改`Debian 12`的Linux内核,在不重启的情况下更改`/proc/uptime`(和`uptime`命令)显示的内容, 让系统看起来已经运行了10年.

## 环境准备

### 环境检查
检查内核已支持实时补丁功能. 确认`CONFIG_HAVE_LIVEPATCH`和`CONFIG_LIVEPATCH`是否为 `y`
```
grep LIVEPATCH /boot/config-$(uname -r)
```
检查系统中的`GCC`版本应与内核编译时一致, 否则`kpatch-build`会失败. 虽可用`--skip-gcc-check`跳过检查, 但不推荐.
```
gcc --version
```
查看当前内核的编译所用`GCC`版本
```
cat /proc/version
```
### 安装依赖
安装依赖包
```
apt-get install gawk build-essential libelf-dev -y
```
安装系统的Linux内核的源码
```
apt-get install linux-source
```
安装系统的Linux内核的`vmlinux`文件
```
apt-get install linux-image-$(uname -r)-dbg
```

下载和编译安装 kpatch
```
git clone https://github.com/dynup/kpatch.git
cd kpatch
make && make install
```

### 准备内核源代码

创建新的目录`kernel`, 解压内核源代码
```
cd ~ && mkdir kernel && cd kernel
tar xvf /usr/src/linux-source-6.1.tar.xz
```

### 创建内核配置

当前系统运行的Linux内核是根据发行版提供的配置文件编译的. 需要复制并调整该配置, 以便`kpatch-build`能使用与当前运行内核相同的配置来编译内核.

```
cd linux-source-6.1/
cp /boot/config-$(uname -r) .config
make olddefconfig 
```

检查使用`kpatch`所需的内核配置是否已启用, 所有项应返回`y` 你好

```
scripts/config -s HAVE_LIVEPATCH
scripts/config -s LIVEPATCH
scripts/config -s HAVE_DYNAMIC_FTRACE_WITH_REGS
scripts/config -s DYNAMIC_FTRACE_WITH_REGS
scripts/config -s HAVE_FUNCTION_TRACER
scripts/config -s FUNCTION_TRACER
scripts/config -s HAVE_FENTRY
scripts/config -s KALLSYMS_ALL
scripts/config -s MODULES
scripts/config -s MODULE_SIG
scripts/config -s SYSFS
scripts/config -s SYSTEM_TRUSTED_KEYRING
```

修改一个配置项, 返回到`kernel`目录
```
scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
cd ..
```

## 创建patch
补丁文件是对原始源码和修改后源码运行`diff`命令所生成的输出.
本例修改`uptime`命令输出, 让服务器看起来已经运行了十年

```
cp linux-source-6.1/fs/proc/uptime.c .
```

编辑34行
```
(unsigned long) uptime.tv_sec,
```
修改为

```
(unsigned long) uptime.tv_sec + 315569260,
```
保存修改.
创建patch文件
```
diff -u --label a/fs/proc/uptime.c linux-source-6.1/fs/proc/uptime.c --label b/fs/proc/uptime.c ./uptime.c > uptime.patch
```

创建补丁模块, 首次较慢, 需编译内核源码, 后续构建几分钟即可, 多核CPU可加快速度.
```
kpatch-build -s ~/kernel/linux-source-6.1/  -v /lib/debug/lib/modules/6.1.0-35-amd64/vmlinux ~/kernel/uptime.patch
```

完成后,会生成补丁的内核模块文件(.ko)
```
ls -l *.ko
```

## 测试patch

在加载补丁模块前, 查看当前运行时间
```
cat /proc/uptime && uptime -p
```

加载补丁模块
```
kpatch load livepatch-uptime.ko
```
再次查看运行时间
```
cat /proc/uptime && uptime -p
```
你会看到运行时间显示增加了十年(内部值未变,仅显示内容被修改)

卸载补丁模块
```
kpatch unload livepatch-uptime.ko
```
确认运行时间恢复到原来状态
```
cat /proc/uptime && uptime -p
```