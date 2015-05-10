title: "Linux/x86中断处理：中断号"
date: 2015-05-04 20:20:00
category: Techniques
tags: [linux, architecture]
---

![](thumbnail.jpg)

(图片来源：http://www.pd4pic.com/)

被问了Linux/x86上有关中断号的问题，仔细一想发现自己也不太清楚，于是就有了下文。

# 架构无关部分

在架构无关部分，中断处理的大致流程很直观：handle_irq（架构无关部分的中断处理入口）拿到一个中断号，找先前在这个中断号上注册过的中断处理例程，每个例程调一遍，完事。所谓“在这个中断号上注册”，指的就是以这个中断号为参数，调用request_irq。

数据结构也很直接：一个中断号对应一个irq_desc，irq_desc里面记录了所有先前注册了这个中断号的中断处理例程（irqaction）。irq_desc里的handle_irq次一级的处理入口，由它一边处理一些通用的细节问题（比如边沿/电平触发的分别处理），一边一个一个地调action链表里注册过的例程。irq_chip是对应中断控制器的结构，里面存放的是一系列函数指针，实现像开关中断这样的由中断控制器提供的中断管理过程。

                                   irq_desc
                  irq == ?? ==> +------------+
                                | handle_irq |
               irq_chip      +--|    chip    |        irqaction
            +------------+<--+  |   action   | ---> +-----------+  +->+-----------+  +-> ...
            |  irq_mask  |      |    name    |      |  handler  |  |  |  handler  |  |
            | irq_unmask |      .            .      | thread_fn |  |  | thread_fn |  |
            .            .      .            .      |   next    |--+  |   next    |--+
            .            .                          .           .     .           .

对于从中断号（上图中的irq）到irq_desc的映射，Linux里默认是用了一个NR_IRQS个单元的irq_desc数组做的，也就是说中断号仅限于0到NR_IRQS-1之间。如果配置选项CONFIG_SPARSE_IRQ开启，那么中断号到irq_desc的映射就会用一个radix tree来维护，那样的话中断号多大都无所谓了。

irq_desc有几个设置函数，用来设置handle_irq、name、chip（严格来说是irq_desc里的irq_data的chip）等，包括：

* irq_set_chip：设置chip
* __irq_set_handler：设置handle_irq和name
* irq_set_chip_and_handler_name：handle_irq、name、chip全包

# 架构相关部分（x86）

特别要注意的是，架构无关部分所用的中断号纯粹是个逻辑值，和硬件上所用的中断号没·有·必·然·联·系！换句话说，只要和架构相关的代码串通好了，我们就完全可以说时钟的中断号是0xbeef，串口的中断号是0xdead，硬盘的中断号是0xbaddad，尽管硬件使用的中断号可能都没超过256。区别起见，以下把架构无关部分用的中断号叫做“逻辑中断号”，硬件使用的中断号（具体到x86上就是用哪个IDT表项）叫做“物理中断号”。对于x86架构来说，物理中断号的范围就是0-255，0号是除0错，14号是缺页等等。

## 物理中断号到逻辑中断号的映射

建立物理中断号到逻辑中断号的映射，是架构相关的中断处理例程需要完成的主要任务之一。对于x86_64来说，汇编部分的IDT表项入口是：

```
ENTRY(irq_entries_start)
        INTR_FRAME
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
        pushq_cfi $(~vector+0x80)    /* Note: always in signed byte range */
    vector=vector+1
        jmp     common_interrupt
        CFI_ADJUST_CFA_OFFSET -8
        .align  8
    .endr
        CFI_ENDPROC
END(irq_entries_start)
```

C部分的IDT建立是：

```
for_each_clear_bit_from(i, used_vectors, first_system_vector) {
        /* IA32_SYSCALL_VECTOR could be used in trap_init already. */
        set_intr_gate(i, irq_entries_start +
                        8 * (i - FIRST_EXTERNAL_VECTOR));
}
```

FIRST_EXTERNAL_VECTOR是0x20（和Intel手册上的内容一致）。进入中断后的一系列操作包括汇编里的：

```
common_interrupt:
        XCPT_FRAME
        ASM_CLAC
        addq $-0x80,(%rsp)              /* Adjust vector to [-256,-1] range */
        interrupt do_IRQ
```

和C里的：
```
unsigned int __irq_entry do_IRQ(struct pt_regs *regs) {
        unsigned vector = ~regs->orig_ax;
        unsigned irq;
		...
		irq = __this_cpu_read(vector_irq[vector]);

        if (!handle_irq(irq, regs))
		...
}
```

其中regs->orig_ax就是irq_entries_start里那些IDT入口push到栈上的值。那么，do_IRQ里的vector（记作v'）和IDT入口里的vector（记作v）的关系就应该是：

    v' = ~((~v + 0x80) + (-0x80)) = v

不就是一样的嘛……

拿到vector之后，接下去还有一个per_cpu的vector_irq，把vector映射成irq，这个才是麻烦的部分，因为对于x86平台上大部分中断，无论是物理中断号还是发给哪个CPU都是可配置的，所以静态没办法决定那个物理中断号、CPU号和逻辑中断号间的关系，只好弄一个per_cpu的vector_irq，遇到一个存一个。

## CPU应该找哪个IDT表项？

一个中断不过是一个管脚上电平的变化，这和应该找哪个IDT表项之间还有一个巨~~~大的差距，因为中间横着lapic/ioapi这两个不省油的灯，先放一张从SDM里抄过来的架构图（这是xAPIC架构，ioapic连到系统总线上，老的APIC架构里ioapic和lapic用一根专用的总线连在一起）。

![lapic/ioapic总体架构](apic.jpg)

我们一个个来看……

### lapic

都说x86中断后会根据一个号码选择相应IDT表项，执行里面注册的处理例程，那这个号码（物理中断号）又是怎么来的？在最近的x86体系结构下，是lapic给的，而且**物理中断号是可编程的**！

由lapic直接管理的中断源包括：
* 时钟（Timer），就是每个CPU自己的时钟；
* 两个lapic自己的中断管脚（LINT0和LINT1），连什么都可以，包括级联其它中断控制器；
* 性能计数器（Performance Counter），就是PMU每次倒数到0时发出来的中断，一般都设成NMI；
* 温度传感器（Thermal Monitor），温度过高时断点用的？
* lapic内部错误（Error）
* 系统自检错（Corrected Machine Check error Interrupt，即CMCI），完全不清楚是个什么东西……
* IPI，核间中断，常常用来让另一个指定的核干点什么事情。

lapic里有一张叫做Local Vector Table（LVT）的表，其实就是映射到MMIO空间的7个32位的寄存器（表里没有IPI的位置），格式差不多像下面这样

     31              23              15              7             0
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                             | | |   | | mode  |     vector    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                   ^ ^     ^
                                   | |     +--- status
                                   | +--- trigger
                                   +--- mask

* vector：该中断被触发后，lapic报给CPU的物理中断号；上电后的默认值是0x10
* (delivery) mode：配置NMI等中断类型；
* (delivery) status：0表示空闲，1表示中断pending；
* trigger (mode)：0表示边沿触发，1表示电平触发；
* mask：0表示可用，1表示被屏蔽。

IDT就256个表项，所以vector用8位足矣。初始情况下，这些寄存器的最后八位都是0x10，所以时钟中断的物理中断号默认情况下是32号（逻辑中断号是0），不过你完全可以改LVT，把时钟中断弄到其它中断号上面去。另一个问题是，不是说有了这8位你就可以发NMI甚至page fault了，发送0到15号中断的尝试会让lapic报错，真要发NMI得要找那个mode。

每个中断的LVT表项稍稍有点不一样，具体在SDM Vol.3 10.5里都找得到。

那IPI怎么发？lapic还有另外一个64位的寄存器ICR，向ICR的低32位写一次就等于发一次IPI。ICR的格式和ioapic的redirection table（见下）很像：

     63              55              47              39            32
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  destination  |                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

     31              23              15              7             0
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       |   |   | |   | | |mode |     vector    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                              ^      ^     ^ ^
           dest. shorthand ---+      |     | +--- dest. mode
                                     |     +--- status
                                     +--- trigger

* vector：该中断被触发后，ioapic报给lapic的号码，也是lapic报给CPU的物理中断号；
* (delivery) mode：同样是配置NMI等中断类型；
* dest. shorthand、dest. mode和destination：指定中断投送的目标CPU，或者指定发送给自己、全体、除自己外全体，详细情况在SDM Vol. 3 10.6节找；
* (delivery) status：0表示空闲，1表示中断pending；
* trigger (mode)：0表示边沿触发，1表示电平触发；

### ioapic

lapic自己的中断清楚了，但事还没完，网卡和显卡们的中断还在ioapic上呢。ioapic的事情SDM已经不灵了，得另外去找82093AA的文档。

简单来说，ioapic（准确来说是82093AA）可以外接24个中断源，每个中断源对应的信息还是用一张表来维护，这张表叫I/O Redirection Table，从ioapic的MMIO空间的0x10处开始（Linux的ioapic驱动这里用了magic number，参见__ioapic_write_entry）。每个中断64位，格式统一如下：

     63              55              47              39            32
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  destination  |                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

     31              23              15              7             0
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                             | | |   | | |mode |     vector    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                   ^ ^     ^ ^
                                   | |     | +--- dest. mode
                                   | |     +--- status
                                   | +--- trigger
                                   +--- mask

* vector：该中断被触发后，ioapic报给lapic的号码，也是lapic报给CPU的物理中断号；
* (delivery) mode：同样是配置NMI等中断类型；
* dest. mode和destination：指定中断投送的目标CPU；
* (delivery) status：0表示空闲，1表示中断pending；
* trigger (mode)：0表示边沿触发，1表示电平触发；
* mask：0表示可用，1表示被屏蔽。

vector号仍然是8位，而且严格限制必须在0x10到0xFE之间（仍然别想法page fault）。ioapic收到中断后，通过PCI，由bridge把消息广播到system bus上，被点名的CPU的lapic就把相应的vector号pending到自己的IRR（一组共256位的寄存器，记录了哪些中断应该交给CPU）里，由lapic“视时机”把中断报给CPU。

### PCI MSI/MSIX

事还~~没完，既然ioapic都到PCI总线上了，没理由其它PCI设备不能给bridge发中断啊！嗯，这就是MSI，有些PCI设备就可以直接用事先给的vector号，给bridge和lapic们发中断消息，最终触发CPU中断。和ioapic的中断一样，中断号、目标CPU、边沿/电平触发之类都是可编程的，具体见SDM Vol.3 10.11。

### 级联中断控制器

事还~~~~没完，ioapic能连在PCI总线上，没道理其它中断控制器不能连到PCI上啊。对于其它连到PCI上的中断控制器，触发中断的办法有两种：

* 连一根脚到ioapic，每次自己这边有中断来，先通过ioapic触发CPU中断，然后在中断处理例程里读自己的寄存器，获得真正的中断源；
* 直接申请一个PCI的MSI中断，每次自己这边有中断来，先通过MSI触发CPU中断，然后……（跟上面一样）

总之怎么看都有点级联的味道……

一个例子是x86 SoC上管理gpio中断的langwell驱动，就先用irq_set_chained_handler向架构无关部分申请了一个中断，然后在lnw_irq_handler里再去看自己的中断又是被哪个管脚触发上来的，找到那个中断号后又调用generic_handle_irq返回到体系无关部分做进一步处理，不过这时完全就换了个逻辑中断号，这样一来在只有256个IDT表项的x86上出现大于256的逻辑中断号也没什么奇怪的了。

### 总结

总的来说，决定一个中断用哪个IDT表项的可以是lapic、ioapic、PCI设备或是其它中断控制器，绝大部分中断所用的物理中断号、中断模式、目标CPU都是可编程的，这也难怪Linux要用一个per_cpu的数组去记物理中断号到逻辑中断号的映射关系。这可真是要乱成一锅粥的节奏……

# procfs & sysfs里的中断相关文件

## /proc/interrupts

这大概是最著名的一个了。文件的创建位置是proc_interrupts_init，内容由show_interrupts函数提供，主要流程是遍历一遍所有有效的逻辑中断号，对于带了action（也就是被注册过的）的中断，打印的域包括：

* 逻辑中断号；
* 每个CPU上中断次数的统计；
* 对应chip的名称，诸如IO-APIC、PCI-MSI之类；
* 中断的名称，即irq_desc里的name，通常是空的；
* 每个action的名称，也就是request_irq时给的name，多个action的话用逗号分隔。

遍历了所有逻辑中断号之后，还会调用arch_show_interrupts获得体系结构相关部分的中断信息，在x86上所有第一列不是数字的部分就是x86的arch_show_interrupts捣腾出来的内容，主要的信息包括中断类型和每个CPU上的中断计数。

### /proc/interrupts有中断看不到？！

这个可以有，因为首先处理中断的总是内核的架构相关部分，如果它默默地自己处理了某些中断，既不用handle_irq（架构无关部分的中断处理入口），也不request_irq，还不在arch_show_interrupts里给你看，那/proc/interrupts里自然就找不到这些中断的痕迹。

x86就有架构相关部分默默处理掉的中断，比如LAPIC时钟。虽说LAPIC时钟中断默认是0x20，但在__setup_APIC_LVTT里就把它改成0xEF了，而0xEF的IDT表项又在apic_intr_init中专门设成了apic_timer_interrupt，而不是irq_entries_start那张表里的通用入口。所以要是在/proc/interrupts里看到这么一行：

             CPU0       CPU1       CPU2       CPU3
    0:        134          0          0          0   IO-APIC-edge      timer

这货才不是运行时的时钟（不排除系统启动的时候暂时用过它）！真正的时钟应该是：

    LOC:  919608278  869138540  868188298  901247206   Local timer interrupts

## /proc/irq/*

这个目录主要是用来控制中断亲和性的，目录下面每个被注册过的逻辑中断号有一个目录，用得比较多的是下面两个文件：

* smp_affinity：bitmap形式的亲和性设置；
* smp_affinity_list：CPU id列表形式的亲和性设置；

其它几个文件只读，其内容的含义待考。

## /sys/.../irq 和 /sys/.../msi_irqs

基本每个device的目录下面都有，irq是每个设备所使用的逻辑中断号，不过奇怪的是有些逻辑中断号没有在/proc/interrupts里出现，原因不明……msi_irqs目录下面是一些以逻辑中断号命名的文件，每个都是这个设备所申请的msi或msix中断，没有申请过msi和msix中断的设备没有这个目录。

# 总结：怎么知道设备的中断号？

绕了这么一大圈下来，可以明显感觉到x86平台为了让一套中断系统适应各种各样奇奇怪怪的环境，在“可配置”这一点上下了多大功夫。这也让搞内核开发的有点头疼：找个中断号怎么就那么麻烦？！

根据x86中断系统的结构，在Linux里找一个设备的中断号大概分这么几步：

1. 确定逻辑中断号：先看/proc/interrupts，有名字跟设备直接对应的最好，看不出的话就去sysfs这个符号链接的迷宫，从总线、功能、驱动模块……等等角度，找那个device对应的目录，看文件irq和目录msi_irqs的内容；
2. 确定物理中断号：插printk，在适当的位置（比如arch_show_interrupts）把vector_irq这个数组打出来，找哪个物理中断号对应了上面确定的逻辑中断号；
3. 确定物理中断号的配置情况：先从/proc/interrupts搞清楚管理这个中断的chip，然后：
  * 如果是IOAPIC，找调用ioapic_write_entry和__ioapic_write_entry（更新I/O Redirection Table的函数）的地方；
  * 如果是MSI，找驱动初始化时申请msi的地方；
  * 如果是其它东西，那一般是一个级联的中断控制器，先翻翻这个中断控制器的驱动，看看有没有设置中断号之类的API再做商榷；
  * 对于中断号用字母缩写的中断，在apic.h里找对应的寄存器，然后找apic_write这个寄存器的地方。

对于其它架构，第一步总是适用的（因为架构无关），第二步需要先找do_IRQ和调用handle_irq的地方，看传给handle_irq的那个irq参数是怎么一步一步被算出来的；至于第三步……这个差距太大了，具体问题具体分析吧……

# References

[Intel SDM](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
[82093AA I/O ADVANCED PROGRAMMABLE INTERRUPT CONTROLLER](http://www.intel.com/design/chipsets/datashts/29056601.pdf)
[gpio: add Intel Moorestown Platform Langwell chip gpio driver](http://lkml.iu.edu/hypermail/linux/kernel/0907.0/00766.html)
