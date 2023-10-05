# systemd-boot添加Windows引导项


# 前言
最近把系统换成了CachyOS，用上了ZFS，这个发行版做的一些Tricky还是可以的，例如一些fish的脚本，还有软件与DE的自定义项都比较简洁，比ArcoLinux强上不少。  
我的情况是有两块NVMe的硬盘，一块是Windows系统，一块则是CachyOS。我的期望是用Linux Boot Manager去引导两个操作系统，比用Windows Boot Manager去引导Linux要好操作不少。  
<!--more--> 
# 确定fs alias
看了一下[systemd-boot](https://wiki.archlinux.org/title/systemd-boot)的文档，需要先安装 [edk2-shell](https://archlinux.org/packages/extra/any/edk2-shell/) 来给`systemd-boot` 加个EFI shell:
```
sudo pacman -S edk2-shell && cp /usr/share/edk2-shell/x64/Shell.efi /boot/shellx64.efi
```
然后在boot的时候选择EFI Shell，在shell中，输入map命令，可以得到所有盘的`fs alias`, 这里我们需要得到是另一块盘的EFI分区。  
可以用`blkid`可以看到所有盘or分区的`PARTUUID`，在EFI shell的map命令的输出结果中，可以对应上具体的`fs alias`是哪个盘。  

# 加上Boot脚本
确定了`fs alias`后就好办了，在`/boot/`下面新建一个`windows.nsh`，内容如下：  
```
HD0b:EFI\Microsoft\Boot\Bootmgfw.efi
```
在这里，我的windows EFI分区的`fs alias`是HD0b。  

然后在`/boot/loader/entries/`下面新建一个引导项`windows.conf`：  
```
title  Windows
efi     /shellx64.efi
options -nointerrupt -noconsolein -noconsoleout windows.nsh
```
最后检查一下`windows.nsh`是否在`/boot/`目录下面，用`bootctl list` 看一下boot项的输出，我的输出如下：
```
         type: Boot Loader Specification Type #1 (.conf)
        title: Windows
           id: windows.conf
       source: /boot//loader/entries/windows.conf
          efi: /boot//shellx64.efi
      options: -nointerrupt -noconsolein -noconsoleout windows.nsh

         type: Boot Loader Specification Type #1 (.conf)
        title: Linux Cachyos Lts Lto (default) (selected)
           id: linux-cachyos-lts-lto.conf
       source: /boot//loader/entries/linux-cachyos-lts-lto.conf
        linux: /boot//vmlinuz-linux-cachyos-lts-lto
       initrd: /boot//amd-ucode.img
               /boot//initramfs-linux-cachyos-lts-lto.img
      options: zfs=zpcachyos/ROOT/cos/root rw zswap.enabled=0 nowatchdog

         type: Boot Loader Specification Type #1 (.conf)
        title: Linux Cachyos Lts Lto (Fallback)
           id: linux-cachyos-lts-lto-fallback.conf
       source: /boot//loader/entries/linux-cachyos-lts-lto-fallback.conf
        linux: /boot//vmlinuz-linux-cachyos-lts-lto
       initrd: /boot//amd-ucode.img
               /boot//initramfs-linux-cachyos-lts-lto-fallback.img
      options: zfs=zpcachyos/ROOT/cos/root rw
```

可以确定的是，如果输出正确，那么就可以在引导的时候选择windows了。
PS，如果要改默认选择项，在`/boot/loader/loader.conf` 中可以改掉默认引导项，或者使用`bootctl set-default` 来指定。




