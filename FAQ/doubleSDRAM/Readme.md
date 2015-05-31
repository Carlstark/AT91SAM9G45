### **AT91SAM9G45双片选SDRAM**
---
AT91SAM9G45芯片手册描述支持挂载双SDRAM:

* DDRSDRC1 | 0x20000000
* DDRSDRC0 | 0x70000000

官方源码中将DDRSDRC0用作主内存, 而DDRSDRC1并未在Linux系统下被使能.

那么问题来了, 加入硬件设计中在DDRSDRC1挂有SDRAM岂不是造成极大浪费 ? 是的, 本文仅讲述如何减小浪费, 暂时仍然未能充分利用该片选上的内存空间.

减小浪费的方法即: 将DDRSDRC1中的部分空间分配给LCD的Framebuffer.

以linux-2.6.30为例, 修改文件`arch/arm/mach-at91/at91sam9g45_devices.c`
```
static struct resource lcdc_resources[] = {
    [0] = {
        .start  = AT91SAM9G45_LCDC_BASE,
        .end    = AT91SAM9G45_LCDC_BASE + SZ_4K - 1,
        .flags  = IORESOURCE_MEM,
    },
    [1] = {
        .start  = 0x20000000,
        .end    = 0x20000000 + 0x3A9800 - 1 ,
        .flags  = IORESOURCE_MEM,
    },
    [2] = {
        .start  = AT91SAM9G45_ID_LCDC,
        .end    = AT91SAM9G45_ID_LCDC,
        .flags  = IORESOURCE_IRQ,
    },
};
```
找到上列结构体, 改成这样即可.
其中成员[1]描述了Framebuffer的空间, 长度为`0x3A9800 = 800 * 600 * 4 * 2`, 即最大适合800*600 32bpp 双buffer显示.

#### 修改前后对比
---
修改前:
>
atmel_lcdfb atmel_lcdfb.0: 750KiB frame buffer at 77900000 (mapped at ffa00000)
atmel_lcdfb atmel_lcdfb.0: fb0: Atmel LCDC at 0x00500000 (mapped at c886a000), irq

>
[root@:/]# free
              total         used         free       shared      buffers
  Mem:       126380         6724       119656            0            0
 Swap:            0            0            0
Total:       126380         6724       119656

修改后
>
atmel_lcdfb atmel_lcdfb.0: 3750KiB frame buffer at 20000000 (mapped at c8c00000)
atmel_lcdfb atmel_lcdfb.0: fb0: Atmel LCDC at 0x00500000 (mapped at c886a000), irq 23

>
[root@:/]# free
              total         used         free       shared      buffers
  Mem:       125916         5944       119972            0            0
 Swap:            0            0            0
Total:       125916         5944       119972

明显, 修改后系统占用的内存空间减小了750KB左右, 释放的空间即原来占用的Framebuffer. 若您能力足够, 可看到内存地址0x20000000处存储的内容和显示内容是一致的.

不过回收700多KB相对于DDRSDRC1的内存空间来讲显得微不足道.


#### **充分利用剩下的物理内存**
---
##### **原理**

将DDRSDRC1上的SDRAM存储空间虚拟成磁盘, 故在Linux系统下可将其当作文件系统保存临时文件, 以获得110MB左右的有效快速的临时存储空间.

1. 应用补丁

* `cd linux-2.6.30;`
* `patch -p1 < <YOUR_PATH>/patch/patch/linux-2.6.30_sdram.patch`

2. 重新编译内核后将uImage烧录到ARM板

3. 启动ARM板Linux, 加载驱动

* `insmod atmel_memdisk.ko`
成功后将生成`/dev/memdisk`节点.

4. 使用下列脚本初始化该虚拟磁盘
```
dd if=/dev/zero of=/dev/memdisk bs=1024 count=3
fdisk /dev/memdisk << EOF
n
p
1


w
EOF
[ -e "/dev/memdisk1" ] && mkfs.ext2 /dev/memdisk1
! [ -d "/dev/shm" ] && mkdir /dev/shm
mount -t ext2 /dev/memdisk1 /dev/shm
```
执行成功后, 该虚拟磁盘被挂载到`/dev/shm`.

##### **测试**

查看存储情况

>
[root@9G45:/]# df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/root               246.4M     14.5M    231.9M   6% /
tmpfs                    61.6M      4.0K     61.6M   0% /dev
tmpfs                    61.6M         0     61.6M   0% /tmp
tmpfs                    61.6M     16.0K     61.6M   0% /var
/dev/memdisk1           119.1M    111.9M      1.0M  99% /dev/shm
[root@9G45:/]#

写入文件

>
[root@9G45:/]# dd if=/dev/zero of=/dev/shm/testFile bs=1M count=110  
110+0 records in  
110+0 records out  

查看内存消耗

>
[root@9G45:/]# free  
              total         used         free       shared      buffers  
  Mem:       126144       123228         2916            0         3596  
 Swap:            0            0            0  
Total:       126144       123228         2916  

内存消耗巨大, 是不是文件被保存到了主内存中呢 ? 接着往下看

清理内存

>
[root@9G45:/]# echo 1 > /proc/sys/vm/drop_caches

再次查看内存

>
[root@9G45:/]# free  
              total         used         free       shared      buffers  
  Mem:       126144         4340       121804            0           36  
 Swap:            0            0            0  
Total:       126144         4340       121804  

OK, 正常了. 再看看`/dev/shm/`下的测试文件是否还在, 内容是否被修改.

是的, 虚拟磁盘中的内容没有丢失.
