# 6.828环境配置（tools）

### 写在前面

整个环境搭建主要是根据[官方网站]([6.828 / Fall 2018 (mit.edu)](https://pdos.csail.mit.edu/6.828/2018/tools.html))作为主要参考完成的，总体过程也非常麻烦，中间报错不断，最后也是终于在查了好多博客才完成。

我使用的环境是VM+Ubuntu（20.04.6LTS）作为参考。

我由于之前课程的原因，所以原本的虚拟环境中有预先装有qemu，害怕版本的问题，所以提前进行了卸载（卸载的博客可以轻松查到）

同时,在网站中有这样一句话`Unfortunately, QEMU's debugging facilities, while powerful, are somewhat immature, so we highly recommend you use our patched version of QEMU instead of the stock version that may come with your distribution. `,，所以也是毫不犹豫删除了。

## 前面的后面（遇到的主要问题汇总）

所有的问题全部都出现在qemu的环境搭建中，中间不断出现问题。

- ` ./configure --disable-kvm --disable-werror [--prefix=PFX] [--target-list="i386-softmmu x86_64-softmmu"]`中的两个disable都不要少，否则后面会出问题

  ​	注：上面的两个方框表示选项的意思，不要忘了删

- On Linux, you may need to install several libraries. We have successfully built 6.828 QEMU on Debian/Ubuntu 16.04 after installing the following packages: libsdl1.2-dev, libtool-bin, libglib2.0-dev, libz-dev, and libpixman-1-dev.

  这里工具链的安装，问chatgpt后得知`libz-dev` 包在 Ubuntu 中通常是 `zlib1g-dev`。

  可直接复制下面的话

  ```shell
  sudo apt install libsdl1.2-dev libtool-bin libglib2.0-dev zlib1g-dev libpixman-1-dev
  ```

- 最后对qemu编译出现问题

  解决方案可以看这篇[博客]([MIT6.828实验环境配置_mit6.828配置-CSDN博客](https://blog.csdn.net/qq_43012789/article/details/106343268))和我遇到了很多相近的问题。

  也即通过在qga/commands-posix.c中添加一个头文件，具体就是<sys/types.h>后加入<sys/sysmacros.h>

  重新编译，解决问题。
  
- 最后的最后，运行qemu出现问题。报错代码如下

  ```shell
  *** Error: Couldn't find a working QEMU executable.
  *** Is the directory containing the qemu binary in your PATH
  *** or have you tried setting the QEMU variable in conf/env.mk?
  ```

  这里检查自己是否开始cpu虚拟化

  ```shell
  grep -Eoc '(vmx|svm)' /proc/cpuinfo
  ```

  如果没有开启会输出0，其余为大于0的数，即cpu核心数目。

  下面安装KVM和其他虚拟化管理包

  ```shell
  sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
  ```

  这里安装之后，libvirt守护进程自动启动，可以使用systemctl查看

  ```shell
  sudo systemctl is-active libvirtd
  ```

  最后，为了能够创建和管理虚拟机，需要将用户添加到libvirt和kvm组中，运行下面的代码

  ```shell
  sudo usermod -aG libvirt $USER
  sudo usermod -aG kvm $USER
  ```

  具体可以参考下面这个[博客]([path - ERROR: Couldn't find a working QEMU executable - Stack Overflow](https://stackoverflow.com/questions/56507764/error-couldnt-find-a-working-qemu-executable))

### 中间的最后

至此，JOS就可以运行起来了