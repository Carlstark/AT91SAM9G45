### AT91SAM9G45双片选SDRAM
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
