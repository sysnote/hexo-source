---
title: bcache使用
date: 2016-05-12 17:34:40
tags: [storage,bcache]
categories: storage
---
# 1.编译安装
```bash
# git clone http://evilpiepirate.org/git/bcache-tools.git
```
安装前需要两个依赖包pkg-config和libblkid-dev
```bash
# apt-get install pkg-config libblkid-dev
```
然后编译
```bash
# make
cc -O2 -Wall -g `pkg-config --cflags uuid blkid` make-bcache.c bcache.o `pkg-config --libs uuid blkid` -o make-bcache
cc -O2 -Wall -g `pkg-config --cflags uuid blkid` probe-bcache.c `pkg-config --libs uuid blkid` -o probe-bcache
cc -O2 -Wall -g -std=gnu99 bcache-super-show.c bcache.o `pkg-config --libs uuid` -o bcache-super-show
# make install
install -m0755 make-bcache bcache-super-show	/usr/sbin/
install -m0755 probe-bcache bcache-register	 /lib/udev/
install -m0644 69-bcache.rules	/lib/udev/rules.d/
install -T -m0755 initramfs/hook	/usr/share/initramfs-tools/hooks/bcache
if [ -d /lib/dracut/modules.d ]; \
 then install -D -m0755 dracut/module-setup.sh /lib/dracut/modules.d/90bcache/module-setup.sh; \
 fi
install -m0644 -- *.8 /usr/share/man/man8/
```
<!--more-->
# 2.使用方式
## 2.1 创建bcache设备
```bash
命令：make-bcache -C <cache-device> -B <backing-device>
```
以vde作为缓存盘，vdb和vdc作为后端设备创建bcache设备，有几个后端设备就会生成几个bcache设备。
```bash
# make-bcache -C /dev/vde -B /dev/vdb /dev/vdc
UUID:	 8941a5d1-074e-4cc6-a6dd-56cb1f65aed3
Set UUID:	 7681dbb3-6558-4e60-b062-5fbb648f6665
version:	 0
nbuckets:	 20480
block_size:	 1
bucket_size:	 1024
nr_in_set:	 1
nr_this_dev:	 0
first_bucket:	 1
UUID:	 9e28d09b-942d-487e-be22-9132063da572
Set UUID:	 7681dbb3-6558-4e60-b062-5fbb648f6665
version:	 1
block_size:	 1
data_offset:	 16
UUID:	 1bacf83e-2754-4d6a-a964-09e53e6e679f
Set UUID:	 7681dbb3-6558-4e60-b062-5fbb648f6665
version:	 1
block_size:	 1
data_offset:	 16
root@nbsagent3:~# ls /dev/b
bcache0 bcache1 block/ btrfs-control bus/
root@nbsagent3:~# ls /dev/bcache* -la
brw-rw---T 1 root disk 250, 0 5月 22 17:23 /dev/bcache0
brw-rw---T 1 root disk 250, 1 5月 22 17:23 /dev/bcache1
```
然后可以使用lsblk查看这些设备的对应关系
```bash
# lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda 254:0 0 20G 0 disk
└─vda1 254:1 0 20G 0 part /
vdb 254:16 0 50G 0 disk
└─bcache1 250:1 0 50G 0 disk
vdc 254:32 0 50G 0 disk
└─bcache0 250:0 0 50G 0 disk
vdd 254:48 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vde 254:64 0 10G 0 disk
├─bcache0 250:0 0 50G 0 disk
└─bcache1 250:1 0 50G 0 disk
```
## 2.2 添加一块后端设备（backing device）
```bash
命令：make-bcache -B <backing-device>
# make-bcache -B /dev/vdh
UUID:	 bd1bc116-6f39-401e-a328-3bc0af4ac8d1
Set UUID:	 cd86812e-d24c-46da-b949-683969b59575
version:	 1
block_size:	 1
data_offset:	 16
```
接着看到对应的设备是/dev/bcache3
查看cache set uuid
```bash
# ls -la /sys/fs/bcache/
total 0
drwxr-xr-x 3 root root 0 May 22 17:23 .
drwxr-xr-x 6 root root 0 May 22 17:21 ..
drwxr-xr-x 7 root root 0 May 22 17:26 7681dbb3-6558-4e60-b062-5fbb648f6665
--w------- 1 root root 4096 May 22 17:57 register
--w------- 1 root root 4096 May 22 20:44 register_quiet
```
"attach"后端设备
```bash
命令：echo <cache set uuid> > /sys/block/bcache<N>/bcache/attach
```
attach之后，缓存设备就能够对新加的后端设备缓存数据了。
```bash
# echo 7681dbb3-6558-4e60-b062-5fbb648f6665 > /sys/block/bcache3/bcache/attach
```
添加后使用lsblk查看
```bash
# lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda 254:0 0 20G 0 disk
└─vda1 254:1 0 20G 0 part /
vdb 254:16 0 50G 0 disk
└─bcache1 250:1 0 50G 0 disk
vdc 254:32 0 50G 0 disk
└─bcache0 250:0 0 50G 0 disk
vdd 254:48 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vde 254:64 0 10G 0 disk
├─bcache0 250:0 0 50G 0 disk
├─bcache2 250:2 0 50G 0 disk
└─bcache3 250:3 0 30G 0 disk
vdf 254:80 0 10G 0 disk
vdg 254:96 0 20G 0 disk
vdh 254:112 0 30G 0 disk
└─bcache3 250:3 0 30G 0 disk
```
## 2.3 删除一块后端设备
### 1）detach backing device
```bash
命令：echo <cache set uuid> > /sys/block/bcache<N>/bcache/detach
```
detach后端设备后，对应的bcache<N>设备还是存在的，只不过这个bcache设备是无缓存的，查看其状态可以看到是no cache
比如删除bcache3
```bash
# echo 7681dbb3-6558-4e60-b062-5fbb648f6665 > /sys/block/bcache3/bcache/detach
# cat /sys/block/bcache3/bcache/state
no cache
# lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda 254:0 0 20G 0 disk
└─vda1 254:1 0 20G 0 part /
vdb 254:16 0 50G 0 disk
└─bcache1 250:1 0 50G 0 disk
vdc 254:32 0 50G 0 disk
└─bcache0 250:0 0 50G 0 disk
vdd 254:48 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vde 254:64 0 10G 0 disk
├─bcache0 250:0 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vdf 254:80 0 10G 0 disk
vdg 254:96 0 20G 0 disk
vdh 254:112 0 30G 0 disk
└─bcache3 250:3 0 30G 0 disk
```
### 2)stop backing device
```bash
命令：echo 1 > /sys/block/bcache<N>/bcache/stop
```
detach后端设备后，对应的bcache设备还存在，如果要删除，还需要stop该设备
```bash
# echo 1 > /sys/block/bcache3/bcache/stop
# ls /dev/bcache* -la
brw-rw---T 1 root disk 250, 0 May 22 17:51 /dev/bcache0
brw-rw---T 1 root disk 250, 1 May 22 17:54 /dev/bcache1
brw-rw---T 1 root disk 250, 2 May 22 20:42 /dev/bcache2
# lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda 254:0 0 20G 0 disk
└─vda1 254:1 0 20G 0 part /
vdb 254:16 0 50G 0 disk
└─bcache1 250:1 0 50G 0 disk
vdc 254:32 0 50G 0 disk
└─bcache0 250:0 0 50G 0 disk
vdd 254:48 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vde 254:64 0 10G 0 disk
├─bcache0 250:0 0 50G 0 disk
├─bcache1 250:1 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vdf 254:80 0 10G 0 disk
vdg 254:96 0 20G 0 disk
vdh 254:112 0 30G 0 disk
```
## 2.4 新增一块缓存设备（caching device）
### 1）创建cache设备
```bash
命令：make-bcache -C <cache device>
```
有可能对应的设备已经有一些元数据，需要使用wipefs清理掉
```bash
# make-bcache -C /dev/vdf
Device /dev/vdf already has a non-bcache superblock, remove it using wipefs and wipefs -a
# wipefs -a /dev/vdf
4 bytes were erased at offset 0x27fff0000 (linux_raid_member)
they were: fc 4e 2b a9
# make-bcache -C /dev/vdf
UUID:	 1f8bc71f-0106-4da5-b781-5de7b1517706
Set UUID:	 7bfb0d17-b6d0-4fe9-942b-a1c75a0893ab
version:	 0
nbuckets:	 20480
block_size:	 1
bucket_size:	 1024
nr_in_set:	 1
nr_this_dev:	 0
first_bucket:	 1
# lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda 254:0 0 20G 0 disk
└─vda1 254:1 0 20G 0 part /
vdb 254:16 0 50G 0 disk
└─bcache1 250:1 0 50G 0 disk
vdc 254:32 0 50G 0 disk
└─bcache0 250:0 0 50G 0 disk
vdd 254:48 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vde 254:64 0 10G 0 disk
├─bcache0 250:0 0 50G 0 disk
├─bcache1 250:1 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vdf 254:80 0 10G 0 disk
vdg 254:96 0 20G 0 disk
vdh 254:112 0 30G 0 disk
└─bcache3 250:3 0 30G 0 disk
nbsram_a 251:0 0 16M 0 disk
nbsram_b 251:16 0 16M 0 disk
nbsram_c 251:32 0 16M 0 disk
# ls /sys/fs/bcache/ -la
total 0
drwxr-xr-x 4 root root 0 May 22 17:23 .
drwxr-xr-x 6 root root 0 May 22 17:21 ..
drwxr-xr-x 7 root root 0 May 22 17:26 7681dbb3-6558-4e60-b062-5fbb648f6665
drwxr-xr-x 7 root root 0 May 22 21:05 7bfb0d17-b6d0-4fe9-942b-a1c75a0893ab
--w------- 1 root root 4096 May 22 17:57 register
--w------- 1 root root 4096 May 22 21:05 register_quiet
```
### 2）与bcache设备关联
```bash
命令：echo <cache set uuid> > /sys/block/bcache<N>/bcache/attach
```
缓存设备需要与bcache设备关联后，才能作为对应bcache设备缓存。
```bash
# echo 7bfb0d17-b6d0-4fe9-942b-a1c75a0893ab > /sys/block/bcache3/bcache/attach
# lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda 254:0 0 20G 0 disk
└─vda1 254:1 0 20G 0 part /
vdb 254:16 0 50G 0 disk
└─bcache1 250:1 0 50G 0 disk
vdc 254:32 0 50G 0 disk
└─bcache0 250:0 0 50G 0 disk
vdd 254:48 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vde 254:64 0 10G 0 disk
├─bcache0 250:0 0 50G 0 disk
├─bcache1 250:1 0 50G 0 disk
└─bcache2 250:2 0 50G 0 disk
vdf 254:80 0 10G 0 disk
└─bcache3 250:3 0 30G 0 disk
vdg 254:96 0 20G 0 disk
vdh 254:112 0 30G 0 disk
└─bcache3 250:3 0 30G 0 disk
```
## 2.5 删除cache设备
首先确保没有backing device在使用它，上述的“删除一块后端设备”有说明如何设置取消后端设备对缓存设备的使用。
然后可以使用lsblk来查看是否有盘在引用它。
在在/sys/fs/bcache目录下还有对应的cache set uuid，unregister该set uuid后这个cache设备就被视为删除了。
```bash
命令：echo 1 > /sys/fs/bcache/<cache set uuid>/unregister
```
比如vde已经不作为任何设备的缓存盘了，在/sys/fs/bcache目录下有对应的cache set uuid
```bash
# ls /sys/fs/bcache/ -la
total 0
drwxr-xr-x 4 root root 0 5月 26 09:07 .
drwxr-xr-x 6 root root 0 5月 26 09:06 ..
drwxr-xr-x 7 root root 0 5月 26 09:07 7681dbb3-6558-4e60-b062-5fbb648f6665
drwxr-xr-x 7 root root 0 5月 26 09:07 7bfb0d17-b6d0-4fe9-942b-a1c75a0893ab
--w------- 1 root root 4096 5月 26 09:07 register
--w------- 1 root root 4096 5月 26 09:06 register_quiet
```
然后unregister掉该cache设备
```bash
#echo 1 > /sys/fs/bcache/[SSD bcache UUID]/unregister
```
然后再看/sys/fs/bcache目录下已经没有这个cache设备的uuid了
```bash
# ls /sys/fs/bcache/ -la
total 0
drwxr-xr-x 3 root root 0 5月 26 09:07 .
drwxr-xr-x 6 root root 0 5月 26 09:06 ..
drwxr-xr-x 7 root root 0 5月 26 09:07 7bfb0d17-b6d0-4fe9-942b-a1c75a0893ab
--w------- 1 root root 4096 5月 26 09:07 register
--w------- 1 root root 4096 5月 26 09:06 register_quiet
```
## 2.6 参数调节
### 1）设置缓存模式
```bash
命令：echo <cache mode> > /sys/block/bcache<N>/bcache/cache_mode
```
可以为不同的bcache设备设置不同的缓存策略
并且设置了缓存策略后，机器重启后仍然生效。
另外，机器重启后，bcache设备也能自动重建出来。
bcache还有其他可以调节的参数，这里不作说明，具体参考官方说明。
# 3.参考资料
http://bcache.evilpiepirate.org/
http://lwn.net/Articles/550207/
http://blog.csdn.net/liumangxiong/article/details/18090043
http://unix.stackexchange.com/questions/115764/how-to-remove-cache-device-from-bcache
http://pommi.nethuis.nl/ssd-caching-using-linux-and-bcache/