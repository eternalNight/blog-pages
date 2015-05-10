title: "Linux/x86中断处理：OS怎么结束中断？"
date: 2015-05-09 11:18:22
category: Techniques
tags: [linux, architecture]
---

![](thumbnail.jpg)

(图片来源：http://www.pd4pic.com/)

# x86中断种类

在x86上，无论是lapic、ioapic还是msi中断，在相应的配置表项/寄存器里都有delivery mode的概念。对于lapic直接管理的中断而言，delivery mode有五种：

* Fixed（000）：普通中断，物理中断号使用vector域（配置表项/寄存器的最低8位）；
* Lowest Priority（001）：把中断发给dest.指定的那些CPU中优先级最低的那个，其它和Fixed一样，似乎可以用来避免打断正在执行重要工作的CPU或中断，考虑实时性的时候会有点用，但实际上Linux里除了kvm没人用到这个……
* SMI（010）：系统管理中断（System Management Interrupt），暂时没看到Linux怎么用的；
* NMI（100）：给CPU发一个不可屏蔽中断（Non-maskable Interrupt），vector域的内容直接忽略；
* INIT（101）：强制CPU重新初始化（101还可以是INIT Level De-assert，Xeon处理器不支持，这里按下不提）；
* Start-up（110）：仅限于IPI，让目标CPU从指定位置开始执行，用于CPU初始化；
* ExtINT（111）：用于级联8259A，涉及到lapic如何在总线上相应中断源；在正常使用ioapic的Linux里没有用到这个。

各种中断支持的mode如下表：

|              | Fixed | Lowest Priority | SMI | NMI | INIT | Start-up | ExtINT |
|:-------------|:-----:|:---------------:|:---:|:---:|:----:|:--------:|:------:|
|lapic（除IPI）|   *   |                 |  *  |  *  |  *   |          |   *    |
|lapic（IPI）  |   *   |        *        |  *  |  *  |  *   |    *     |        |
|ioapic        |   *   |        *        |  *  |  *  |  *   |          |   *    |
|msi           |   *   |        *        |  *  |  *  |  *   |          |   *    |

Linux里用不同前缀的宏来表示这些mode，包括APIC_DM_、APIC_EILVT_MSG_、APIC_MODE_和一个枚举类型ioapic_irq_destination_types，APIC_DM_似乎是用于IPI的配置，其它的不甚清楚。实际上因为各处的delivery mode编码都是一样的，所以理论上混用也不会有问题，有没有混用的情况就不知道了。

# 各类中断的处理流程

以下讨论lapic、ioapic和msi中断流程时，主要考虑最最常用的Fixed中断，其它非主流大多不考虑。

## lapic

### Fixed中断

lapic里有三个256位的寄存器来表示每一个中断当前的状态，分别是：

     255                                                16 15       0
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                     |reserved |  IRR
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                     |reserved |  ISR
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                     |reserved |  TMR
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

IRR和ISR构成一个中断处理的“流水线”；TMR记录中断是边沿触发还是电平触发，用于决定在EOI（End of Interrupt）时进行相应处理。

一个中断的“生命周期”大致是这样的：

1. lapic收到普通中断（从自身管脚上的还是从处理器的内部总线上收到），将vector号对应的IRR中的那一位置1；如果IRR中的对应为已经是1的话，那么这两次中断相当于被合并为一次。
2. 当CPU允许中断（EFLAGS的IF位为1），且以下两个条件中的任意一个满足时：（1） ISR为0，或（2）ISR最高位的1对应的vector的优先级小于IRR最高位的1对应的vector的优先级（一个vector的优先级就是这个8位vector号的高四位，值越大表示优先级越高，好绕……），lapic向CPU发送中断，物理中断号为IRR中最高位对应的vector号，同时将IRR最高位的1清除，设置对应ISR的那一位为1。
3. 当CPU写EOI寄存器（另一个lapic的寄存器）时，lapic清除ISR里最高位的1；如果该中断为电平触发（TMR里有记录），那么lapic还会向ioapic发送EOI。

总之，管你是边沿触发还是电平触发，普通中断无论如何都是要发EOI的！不然一票中断（同优先级和更低优先级的那些）你就甭想再收到了……换句话说，lapic“lapic为地”把它和CPU之间的普通中断都弄成边沿触发了，中断信号线就是ISR里对应的那个bit。

Linux里负责发EOI的是ack_lapic_irq和ack_APIC_irq（实际上都是后者），架构无关代码里可以通过chip（也就是中断控制器）结构里的函数指针irq_ack或irq_eoi来发出EOI。
在各种handle_irq函数里，发出EOI的时机和使用的函数不尽相同：

* handle_edge_irq里是在调用各个irqaction之前，调用irq_ack；
* handle_fasteoi_irq里是在结束各个irqaction之后，在一定条件下调用irq_eoi。

Linux里lapic中断全部设置为边沿触发，所以lapic_chip结构里只有irq_ack；使用ioapic的中断有edge和fasteoi两类，所以ioapic_chip里同时存在irq_ack和irq_eoi。

### NMI等非Fixed中断

这些中断的处理简单粗暴，lapic收到一个就往CPU扔一个，没有EOI什么事（实际上这时候OS不能发EOI），没什么可说的。

## ioapic

SDM里对lapic工作机制描述的很详尽，但一到ioapic就得找专门的ioapic芯片（比如82093AA）的手册，那里描述的细致程度就差得多了（至少82093AA是这样）。以下是根据手册和Linux代码的一些“猜测”。

首先，ioapic没有像lapic那样给专门开三个寄存器（ISR、IRR和TMR）来保存每一个中断的当前状态，而是把中断的当前状态同样存到配置表项（也就是I/O Redirection Table）里，delivery status（DELIVS）对应lapic里的IRR，remote IRR对应lapic里的ISR。一个中断的“生命周期”差不多是：

1. 中断触发后，ioapic设置相应配置表项的DELIVS位为1；
2. 当目标lapic空闲且apic总线可用时，将中断发到目标apic上。在lapic接受了中断请求之后，清除DELIVS位；如果这一中断是电平触发的，那么进一步设置remote IRR为1；
3. 等lapic那边和CPU交涉一圈，发回来EOI（上面提到过lapic什么时候给ioapic发EOI）时，ioapic将对应的remote IRR清空，中断处理完毕。

根据82093AA手册的描述，remote IRR只对电平触发的中断有意义，如此看来，对于边沿触发的中断，要么这些外设会自己恢复中断线上的信号，要么ioapic在收到中断后就自己ack了外设，这样边沿触发的ack不需要向上级部门申请审批或者报备，remote IRR自然也就不需要了（声明：纯属个人猜测）。

## msi

msi保存中断当前状态的方式和ioapic很类似，在配置寄存器里也有一个仅对电平触发中断有效，表示当前中断状态的level位，感觉处理流程和ioapic不会差太多，但目前手头缺少PCIe的手册，细节不甚清楚。

没有去细抠msi的另一个原因，是Linux的msi全用边沿触发（参见setup_msi_irq），所以msi的“中断控制器”结构msi_chip里也只有的irq_ack，换句话说EOI只到lapic为止，估计msi也自己暗地里处理掉了边沿触发的EOI，所以没有OS什么事。

# x86里的handle_fasteoi_irq vs. handle_edge_irq，以及irq_ack vs. irq_eoi

这两组函数在讨论lapic的时候就冒出来了，而且其区别不甚了了。这里不讨论在一般情况下有什么差异，只看看在x86架构下，这两组函数到底是什么货色。

关键在下面这段代码：

```
static void ioapic_register_intr(unsigned int irq, struct irq_cfg *cfg,
                                 unsigned long trigger)
{
        struct irq_chip *chip = &ioapic_chip;
        irq_flow_handler_t hdl;
        bool fasteoi;

        if ((trigger == IOAPIC_AUTO && IO_APIC_irq_trigger(irq)) ||
            trigger == IOAPIC_LEVEL) {
                irq_set_status_flags(irq, IRQ_LEVEL);
                fasteoi = true;
        } else {
                irq_clear_status_flags(irq, IRQ_LEVEL);
                fasteoi = false;
        }

        if (setup_remapped_irq(irq, cfg, chip))
                fasteoi = trigger != 0;

        hdl = fasteoi ? handle_fasteoi_irq : handle_edge_irq;
        irq_set_chip_and_handler_name(irq, chip, hdl,
                                      fasteoi ? "fasteoi" : "edge");
}
```

这是注册ioapic中断时必须调用的一个函数，其中fasteoi决定了使用哪种类型的中断，fasteoi的取值又取决于trigger是不是IOAPIC_LEVEL。换句话说，就是电平触发用fasteoi，边沿触发用edge。

接着是irq_ack和irq_eoi，对于ioapic来说分别是apic_ack_edge和ack_ioapic_level，从名字上看……不就是一个edge一个level吗？！对于边沿触发的中断，lapic不需要告诉ioapic什么时候EOI，所以OS只需要给lapic发EOI就可以；对于电平触发的中断，虽说lapic也会给ioapic发EOI，但是免不了一些corner case下出现混乱的情况（例如运行时CPU offline之类，ack_ioapic_level里有好几个story），所以ack_ioapic_level除了给lapic发EOI以外，还会额外check一遍要不要给ioapic发EOI。

总的来说，handle_edge_irq和irq_ack一组，处理边沿触发的中断；handle_fasteoi_irq和irq_eoi一组，处理电平触发的中断。x86里的edge和fasteoi，本质上就是edge和level的区别嘛……

# 总结

废话那么多，总结下来就是，

* lapic的Fixed中断无论如何__需要__EOI，不然中断只来一次；
* lapic的非Fixed中断无论如何__不需要__EOI；
* ioapic的边沿触发中断不需要EOI，电平触发中断需要EOI，而且不能太过依赖lapic，需要OS再lapic EOI之后再check一次；
* msi的边沿触发中断不需要EOI，电平触发中断不明（Linux没有用到）。

然后Linux x86里，handle_edge_irq和irq_ack用于边沿触发，handle_fasteoi_irq和irq_eoi用于电平触发。

没了。

# References

[Intel SDM](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
[82093AA I/O ADVANCED PROGRAMMABLE INTERRUPT CONTROLLER](http://www.intel.com/design/chipsets/datashts/29056601.pdf)
