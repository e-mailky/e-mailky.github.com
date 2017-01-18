---
layout: post
title:  linux设备树笔记--dts基本概念及语法
categories: [Linux]
tags: [Linux, DTS,]
description: ""
---
## 概述
&emsp;&emsp;&emsp;&emsp;ARM Device Tree起源于OpenFirmware (OF)，在过去的Linux中，arch/arm/plat-xxx和arch/arm/mach-xxx中充斥着大量的垃圾代码，相当多数的代码只是在描述板级细节，而这些板级细节对于内核来讲，不过是垃圾，如板上的platform设备、resource、i2c_board_info、spi\_board\_info以及各种硬件的platform_data。为了改变这种局面，Linux社区的大牛们参考了PowerPC等体系架构中使用的Flattened
Device Tree（FDT），也采用了Device Tree结构，许多硬件的细节可以直接透过它传递给Linux，而不再需要在kernel中进行大量的冗余编码。

&emsp;&emsp;&emsp;&emsp;Device Tree是一种描述硬件的数据结构，由一系列被命名的结点（node）和属性（property）组成，而结点本身可包含子结点。所谓属性，其实就是成对出现的name和value。在Device Tree中，可描述的信息包括（原先这些信息大多被hard code到kernel中）：CPU的数量和类别，内存基地址和大小，总线和桥，外设连接，中断控制器和中断使用情况，GPIO控制器和GPIO使用情况，Clock控制器和Clock使用情况。
通常由.dts文件以文本方式对系统设备树进行描述，经过Device Tree Compiler(dtc)将dts文件转换成二进制文件binary
device tree blob(dtb)，.dtb文件可由Linux内核解析，有了device tree就可以在不改动Linux内核的情况下，对不同的平台实现无差异的支持，只需更换相应的dts文件，即可满足，当然这样会增加内核的体积。

## 设备树的存储格式
&emsp;&emsp;&emsp;&emsp;简单的说，设备树是一种描述硬件配置的树形数据结构，
有且仅有一个根节点[4]。它包含了有关CPU、物理内存、总线、串口、PHY以及其他外围设备信息等。该树继承了Open Firmware IEEE 1275设备树的定义。操作系统能够在启动时对此结构进行语法分析，以此配置内核，加载相应的驱动。 

&emsp;&emsp;&emsp;&emsp;U-Boot需要将扁平设备树在内存地址传给内核。该树主要由三大部分组成：头（Header）、结构块（Structure block）、字符串块（Strings block）。在内存中分配图如下：

```
+--------------------------+
|                          |
| struct boot_param_header |
|                          |
+--------------------------+
|                          |
|   (alignment gap) (*)    |
|                          |
+--------------------------+
|                          |
|   memory reserve map     |
|                          |
+--------------------------+
|                          |
|   (alignment gap) (*)    |
|                          |
+--------------------------+
|                          |
|                          |
|  device-tree structure   |
|                          |
|                          |
+--------------------------+
|                          |
|   (alignment gap) (*)    |
|                          |
+--------------------------+
|                          |
|                          |
|  device-tree strings     |
|                          |
|                          |
+--------------------------+
```

### 一 头（header）

头主要描述设备树的一些基本信息，例如设备树大小，结构块偏移地址，字符串块偏移地址等。偏移地址是相对于设备树头的起始地址计算的。

```
struct boot_param_header {
__be32 magic; //设备树魔数，固定为0xd00dfeed
__be32 totalsize; //整个设备树的大小
__be32 off_dt_struct; //保存结构块在整个设备树中的偏移
__be32 off_dt_strings; //保存的字符串块在设备树中的偏移
__be32 off_mem_rsvmap; //保留内存区，该区保留了不能被内核动态分配的内存空间
__be32 version; //设备树版本
__be32 last_comp_version; //向下兼容版本号
__be32 boot_cpuid_phys; //为在多核处理器中用于启动的主cpu的物理id
__be32 dt_strings_size; //字符串块大小
__be32 dt_struct_size; //结构块大小
};
```

### 二 结构块（struct block）

&emsp;&emsp;&emsp;&emsp;设备树结构块是一个线性化的结构体，是设备树的主体，以节点node的形式保存了目标单板上的设备信息。
&emsp;&emsp;&emsp;&emsp;在结构块中以宏OF\_DT\_BEGIN\_NODE标志一个节点的开始，以宏OF\_DT\_END\_NODE标识一个节点的结束，整个结构块以宏OF\_DT_END结束。一个节点主要由以下几部分组成

* 节点开始标志：一般为OF\_DT\_BEGIN_NODE
* 节点路径或者节点的单元名(version<3以节点路径表示，version>=0x10以节点单元名表示)
* 填充字段（对齐到四字节）
* 节点属性。每个属性以宏OF\_DT_PROP开始，后面依次为属性值的字节长度(4字节)、属性名称在字符串块中的偏移量(4字节)、属性值和填充（对齐到四字节）
* 如果存在子节点，则定义子节点
* 节点结束标志OF\_DT\_END_NODE。

### 三 字符串块
&emsp;&emsp;&emsp;&emsp;通过节点的定义知道节点都有若干属性，而不同的节点的属性又有大量相同的属性名称，因此将这些属性名称提取出一张表，当节点需要应用某个属性名称时直接在属性名字段保存该属性名称在字符串块中的偏移量

### 四  设备树源码 DTS 表示
&emsp;&emsp;&emsp;&emsp;设备树源码文件(.dts)以可读可编辑的文本形式描述系统硬件配置设备树,支持
C/C++方式的注释,该结构有一个唯一的根节点“/”,每个节点都有自己的名字并可以包含多个子节点。设备树的数据格式遵循了
Open Firmware IEEE standard 1275。这个设备树中有很多节点,每个节点都指定了节点单元名称。每一个属性后面都给出相应的值。以双引号引出的内容为 ASCII 字符串,以尖括号给出的是 32 位的16进制值。这个树结构是启动 Linux 内核所需节点和属性简化后的集合,包括了根节点的基本模式信息、CPU
和物理内存布局,它还包括通过/chosen 节点传递给内核的命令行参数信息

### 五 machine_desc结构
&emsp;&emsp;&emsp;&emsp;内核提供了一个重要的结构体struct machine\_desc ，这个结构体在内核移植中起到相当重要的作用，内核通过machine\_desc结构体来控制系统体系架构相关部分的初始化。machine\_desc结构体通过MACHINE\_START宏来初始化，在代码中， 通过在start\_kernel->setup\_arch中调用setup\_machine_fdt来获取

```
struct machine_desc {
    unsigned int nr;    /* architecture number */
    const char *name;   /* architecturename */
    unsigned long atag_offset; /* tagged list (relative) */
    const char *const *dt_compat; /* array of device tree* 'compatible' strings */
    unsigned int nr_irqs; /* number of IRQs */

#ifdef CONFIG_ZONE_DMA
    phys_addr_t dma_zone_size; /* size of DMA-able area */
#endif
    unsigned int video_start;  /* start of video RAM */
    unsigned int video_end;    /* end of video RAM */
    unsigned char reserve_lp0 :1; /* never has lp0 */
    unsigned char reserve_lp1 :1; /* neverhas lp1 */
    unsigned char reserve_lp2 :1; /* never has lp2 */

    enum reboot_mode reboot_mode; /* default restart mode */

    struct smp_operations *smp; /* SMP operations */

    bool (*smp_init)(void);
    void (*fixup)(structtag *, char **,struct meminfo *);
    void (*init_meminfo)(void);
    void (*reserve)(void);/* reserve mem blocks */
    void (*map_io)(void);/* IO mapping function */
    void (*init_early)(void);
    void (*init_irq)(void);
    void (*init_time)(void);
    void (*init_machine)(void);
    void (*init_late)(void);
#ifdef CONFIG_MULTI_IRQ_HANDLER
    void (*handle_irq)(struct pt_regs *);
#endif

    void (*restart)(enum reboot_mode, const char *);
};
```

### 六 设备节点结构体

```
struct device_node {
    const char *name; //设备name
    const char *type; //设备类型
    phandle phandle;
    const char *full_name; //设备全称，包括父设备名
    struct property *properties; //设备属性链表
    struct property *deadprops; //removed properties
    struct device_node *parent; //指向父节点
    struct device_node *child; //指向子节点
    struct device_node *sibling; //指向兄弟节点
    struct device_node *next; //相同设备类型的下一个节点
    struct device_node *allnext; //next in list of all nodes
    struct proc_dir_entry *pde; //该节点对应的proc
    struct kref kref;
    unsigned long _flags;
    void *data;
#if defined(CONFIG_SPARC)
    const char *path_component_name;
    unsigned int unique_id;
    struct of_irq_controller *irq_trans;
#endif

};
```

### 七 属性结构体
```
struct property {
    char *name; //属性名
    int length; //属性值长度
    void *value; //属性值
    struct property *next; //指向下一个属性
    unsigned long _flags; //标志
    unsigned int unique_id;
};
````

## 基本数据格式
&emsp;&emsp;&emsp;&emsp;device tree是一个简单的节点和属性树，属性是键值对，节点可以包含属性和子节点。下面是一个.dts格式的简单设备树。

```
/ { 
    node1 { 
        a-string-property = "A string"; 
        a-string-list-property = "first string", "second string"; 
        a-byte-data-property = [0x01 0x23 0x34 0x56]; 

        child-node1 { 
            first-child-property; 
            second-child-property = <1>; 
            a-string-property = "Hello, world"; 
        }; 

        child-node2 { 

        }; 
    }; 

    node2 { 

        an-empty-property; 
        a-cell-property = <1 2 3 4>; /* each number (cell) is a uint32 */ 
        child-node1 { 
        }; 
    }; 
}; 
```

&emsp;&emsp;&emsp;&emsp;该树并未描述任何东西，也不具备任何实际意义，但它却揭示了节点和属性的结构。即：

一个的根节点:'/'，两个子节点：node1和node2；node1的子节点：child-node1和child-node2，一些属性分散在树之间

&emsp;&emsp;&emsp;&emsp;属性是一些简单的键值对（key-value pairs）：value可以为空也可以包含任意的字节流。而数据类型并没有编码成数据结构，有一些基本数据表示可以在device tree源文件中表示

文本字符串（null 终止）用双引号来表示：string-property = "a string"

“Cells”是由尖括号分隔的32位无符号整数：cell-property = <0xbeef 123 0xabcd1234>

二进制数据是用方括号分隔：binary-property = [0x01 0x23 0x45 0x67];

不同格式的数据可以用逗号连接在一起：mixed-property = "a string", [0x01 0x23 0x45 0x67], <0x12345678>;

逗号也可以用来创建字符串列表：string-list = "red fish", "blue fish";

## 基本概念
&emsp;&emsp;&emsp;&emsp;为了帮助理解device tree的用法，我们从一个简单的计算机开始， 手把手创建一个device tree来描述它

### 例子
&emsp;&emsp;&emsp;&emsp;假设有这样一台计算机（基于ARM Versatile），由“Acme”制造并命名为"Coyote's Revenge"：

&emsp;&emsp;&emsp;&emsp;一个32位的ARM CPU连接到内存映射串行端口的处理器本地总线(processor local bus),spi bus controller, i2c controller, interrupt controller, 和external bus bridge

* 256MB SDRAM，起始地址为0
* 两个串口起始地址为0x101F1000，0x101F2000
* GPIO controller，起始地址为0x101F3000
* SPI controller起始地址为0x10170000，并挂载以下设备：
    + MMC插槽（SS管脚连接到GPIO #1）
* External bus bridge，挂载以下设备：
    + SMC SMC91111以太网设备连接到external bus，基地址0x10100000
    + i2c controller起始地址为0x10160000，并挂载以下设备：
        - Maxim DS1338 real time clock.响应从机地址1101000 (0x58)
        - 64MB NOR flash，基地址0x30000000


#### 初始结构
&emsp;&emsp;&emsp;&emsp;第一步，先构建一个计算机的基本架构，即一个有效设备树的最小架构。在这一步，要唯一地标志这台计算机

```
/ { 
    compatible = "acme,coyotes-revenge"; 
}; 
```
&emsp;&emsp;&emsp;&emsp;compatible属性以"<manufacturer>,<model>"的格式指定了系统名称。指定了具体设备和制造商名称来避免命名空间的冲突是很重要的，因为一个操作系统可以使用compatible值来决定如何运行这台计算机，在该属性中填入正确的数据很重要。理论上，compatible是一个OS唯一地识别一台计算机所需要的所有数据。如果计算机的所有细节都被硬编码，那么OS可以在顶层compatible属性中专门查找"acme,coyotes-revenge"。

#### CPU
&emsp;&emsp;&emsp;&emsp;接下来就要描述各个CPU了。先添加一个“cpus”容器节点，再将每个CPU作为子节点添加。在本例中，系统是基于ARM的双核Cortex A9系统

```
/ { 
    compatible = "acme,coyotes-revenge"; 
    cpus { 
        cpu@0 { 
            compatible = "arm,cortex-a9"; 
        }; 
        cpu@1 { 
            compatible = "arm,cortex-a9"; 
        }; 
    }; 
}; 
```
&emsp;&emsp;&emsp;&emsp;各个CPU节点的compatible属性是一个字符串，与顶层compatible属性类似，该字符串以“<manufacturer>,<model>”的格式指定了CPU的确切型号。
随后更多的属性被添加到cpu节点，但首先我们需要先了解一些基本概念。

#### 节点命名
&emsp;&emsp;&emsp;&emsp;花些时间谈谈命名习惯是值得的。每个节点都必须有一个<name>[@<unit-address>]格式的名称。<name>是一个简单的ascii字符串，最长为31个字符，总的来说，节点命名是根据它代表什么设备。比如说，一个代表3com以太网适配器的节点应该命名为ethernet，而不是3com509。

&emsp;&emsp;&emsp;&emsp;如果节点描述的设备有地址的话，就应该加上unit-address，unit-address通常是用来访问设备的主地址，并在节点的reg属性中被列出。后面我们将谈到reg属性。

&emsp;&emsp;&emsp;&emsp;同级节点的命名必须是唯一，但多个节点的通用名称可以相同，只要地址不同就行。（即serial@101f1000 & serial@101f2000）

关于节点命名的全部细节请参考ePAPR规范2.2.1节

#### 设备
&emsp;&emsp;&emsp;&emsp;系统中的每个设备由device tree的一个节点来表示，接下来将为设备树添加设备节点。在我们讲到如何寻址和如何处理中断之前，暂时将新节点置空

```
/ { 
    compatible = "acme,coyotes-revenge"; 
    cpus { 
        cpu@0 { 
            compatible = "arm,cortex-a9"; 
        }; 
        cpu@1 { 
            compatible = "arm,cortex-a9"; 
        }; 
    }; 
    
    serial@101F0000 { 
        compatible = "arm,pl011"; 
    };
    
    serial@101F2000 { 
        compatible = "arm,pl011"; 
    };
    
    gpio@101F3000 { 
        compatible = "arm,pl061"; 
    };

    interrupt-controller@10140000 { 
        compatible = "arm,pl190"; 
    }; 

    spi@10115000 {
        compatible = "arm,pl022"; 
    };

    external-bus { 
        ethernet@0,0 { 
            compatible = "smc,smc91c111"; 
        };
        i2c@1,0 { 
            compatible = "acme,a1234-i2c-bus"; 
            rtc@58 { 
                compatible = "maxim,ds1338"; 
            };
  
        flash@2,0 { 
            compatible = "samsung,k8f1315ebm", "cfi-flash"; 
        };
    };  
}; 
```
&emsp;&emsp;&emsp;&emsp;在上面的设备树中，系统中的设备节点已经添加进来，树的层次结构反映了设备如何连到系统中。外部总线上的设备就是外部总线节点的子节点，i2c设备是i2c总线控制节点的子节点。总的来说，层次结构表现的是从CPU视角来看的系统视图。在这里这棵树是依然是无效的。它缺少关于设备之间的连接信息。稍后将添加这些数据

&emsp;&emsp;&emsp;&emsp;设备树中应当注意：每个设备节点有一个compatible属性。flash节点的compatible属性有两个字符串。请阅读下一节以了解更多内容。 之前提到的，节点命名应当反映设备的类型，而不是特定型号。请参考ePAPR规范2.2.2节的通用节点命名，应优先使用这些命名。

#### compatible 属性
&emsp;&emsp;&emsp;&emsp;树中的每一个代表了一个设备的节点都要有一个compatible属性。compatible是OS用来决定绑定到设备的设备驱动的关键

&emsp;&emsp;&emsp;&emsp;compatible是字符串的列表。列表中的第一个字符串指定了"<manufacturer>,<model>"格式的节点代表的确切设备，第二个字符串代表了与该设备兼容的其他设备。例如，Freescale MPC8349 SoC有一个串口设备实现了National Semiconductor ns16550寄存器接口。因此MPC8349串口设备的compatible属性为：compatible
= "fsl,mpc8349-uart", "ns16550"。在这里，fsl,mpc8349-uart指定了确切的设备，ns16550表明它与National Semiconductor 16550 UART是寄存器级兼容的。

*注：*由于历史原因，ns16550没有制造商前缀，所有新的compatible值都应使用制造商的前缀。这种做法使得现有的设备驱动程序可以绑定到一个新设备上，同时仍能唯一准确的识别硬件。

*警告：*不要使用通配符compatible值，如"fsl,mpc83xx-uart"等类似表达，芯片厂商总会改变并打破你的通配符假设，到时候再想修改就为时已晚了。相反，你应当选择一个特定的芯片实现，并与所有后续芯片保持兼容。

#### 编址
可编址的设备使用下列属性来将地址信息编码进设备树：

```
reg
#address-cells
#size-cells
```

每个可寻址的设备有一个reg属性，即以下面形式表示的元组列表：reg = <address1 length1 [address2 length2] [address3 length3] ... >

每个元组表示该设备的地址范围。每个地址值由一个或多个32位整数列表组成，被称做cells。同样地，长度值可以是cells列表，也可以为空。

既然address和length字段是大小可变的变量，父节点的#address-cells和#size-cells属性用来说明各个子节点有多少个cells。换句话说，正确解释一个子节点的reg属性需要父节点的#address-cells和#size-cells值。让我们从CPU开始，添加编址属性到示例设备树。

#### CPU编址
Each CPU is assigned a single unique ID, and there is no size associated with CPU ids.

谈到编址，最简单的例子就是CPU节点。每个CPU被分配了一个唯一的ID，并且不存在与CPU ids的相关大小信息。

```
/ { 
    compatible = "acme,coyotes-revenge"; 
    #address-cells = <1>; 
    #size-cells = <1>; 
    cpus { 
        #address-cells = <1>;
        #size-cells = <0>; 
        cpu@0 { 
            compatible = "arm,cortex-a9"; 
            reg = <0>; 
        }; 
        cpu@1 { 
            compatible = "arm,cortex-a9"; 
            reg = <1>; 
        }; 
    }; 
    
    serial@101F0000 { 
        compatible = "arm,pl011"; 
        reg = <0x101f0000 0x1000>; 
    };
    
    serial@101F2000 { 
        compatible = "arm,pl011"; 
        reg = <0x101f2000 0x1000>; 
    };
    
    gpio@101F3000 { 
        compatible = "arm,pl061"; 
        reg = <0x101f3000 0x1000 0x101f4000 0x0010>
    };

    interrupt-controller@10140000 { 
        compatible = "arm,pl190"; 
        reg = <0x10140000 0x1000 >; 
    }; 

    spi@10115000 {
        compatible = "arm,pl022"; 
        reg = <0x10115000 0x1000 >; 
    };

    external-bus { 
        ethernet@0,0 { 
            compatible = "smc,smc91c111"; 
        };
        i2c@1,0 { 
            compatible = "acme,a1234-i2c-bus"; 
            rtc@58 { 
                compatible = "maxim,ds1338"; 
            };
        flash@2,0 { 
            compatible = "samsung,k8f1315ebm", "cfi-flash"; 
        };   
    };
}; 
```
在cpus节点中，#address-cells为1，#size-cells为0，这意味着子寄存器值是一个uint32，
是一个不包含大小字段的地址。在本例中，两个CPU被分配为地址0和1。cpu节点的#size-cells为0，
是因为只为每个CPU分配了地址值。你一定注意到了，reg值与节点名中的值是匹配。
按照惯例，*如果一个节点有reg属性，则节点名称必须包含unit-address属性，unit-address属性值是reg属性中的第一个地址值。*

注：ePAPR中对cell的定义是”一个包含32bit信息的单元“。

#### 内存映射设备
&emsp;&emsp;&emsp;&emsp;与CPU节点中的单一地址值不同，内存映射设备会被分配一个它能响应的地址范围。#size-cells用来说明每个子节点种reg元组的长度大小。在下面的示例中，每个地址值是1 cell (32位) ，并且每个的长度值也为1 cell，这在32位系统中是非常典型的。64位计算机可以在设备树中使用2作为#address-cells和#size-cells的值来实现64位寻址。

```
/ { 
    compatible = "acme,coyotes-revenge"; 
    #address-cells = <1>; 
    #size-cells = <1>; 
    cpus { 
        #address-cells = <1>;
        #size-cells = <0>; 
        cpu@0 { 
            compatible = "arm,cortex-a9"; 
            reg = <0>; 
        }; 
        cpu@1 { 
            compatible = "arm,cortex-a9"; 
            reg = <1>; 
        }; 
    }; 
    
    serial@101F0000 { 
        compatible = "arm,pl011"; 
        reg = <0x101f0000 0x1000>; 
    };
    
    serial@101F2000 { 
        compatible = "arm,pl011"; 
        reg = <0x101f2000 0x1000>; 
    };
    
    gpio@101F3000 { 
        compatible = "arm,pl061"; 
        reg = <0x101f3000 0x1000 0x101f4000 0x0010>
    };

    interrupt-controller@10140000 { 
        compatible = "arm,pl190"; 
        reg = <0x10140000 0x1000 >; 
    }; 

    spi@10115000 {
        compatible = "arm,pl022"; 
        reg = <0x10115000 0x1000 >; 
    };

    external-bus { 
        #address-cells = <2>;
        #size-cells = <1>;
        ethernet@0,0 { 
            compatible = "smc,smc91c111"; 
            reg = <0 0 0x1000>; 
        };
        i2c@1,0 { 
            compatible = "acme,a1234-i2c-bus"; 
            reg = <1 0 0x1000>; 
            rtc@58 { 
                compatible = "maxim,ds1338"; 
            };
        flash@2,0 { 
            compatible = "samsung,k8f1315ebm", "cfi-flash"; 
            reg = <2 0 0x4000000>; 
        };   
    };
}; 
```
每个设备都被分配了一个基地址及该区域大小。本例中的GPIO设备地址被分成两个地址范围:0x101f3000~0x101f3fff和0x101f4000~0x101f400f。

有些挂载于总线上的设备有不同的编址方案。例如，设备也可以通过独立片选线连接到外部总线。因为父节点定义了它的子节点的地址范围，可根据需要选择地址映射来最佳地描述该系统。下面的代码显示了连接到外部总线并将芯片片选编码进地址的设备地址分配。

&emsp;&emsp;&emsp;&emsp;外部总线用了2个cells来表示地址值;
一个是片选号，一个是基于片选的偏移量。长度字段还是一个cell，这是因为只有地址的偏移部分需要一个范围。
所以，在本例中，每个reg条目包含3个cell；片选号码，偏移，长度。由于地址范围包含节点及其子节点，父节点可以自由定义任何对总线而言有意义的编址方案。直接父节点和子节点之外的其他节点，通常不需要关心本地节点地址域

#### 非内存映射设备
&emsp;&emsp;&emsp;&emsp;其他设备没有映射到处理器总线上。虽然这些设备可以有地址范围，但是不能直接被CPU访问，而是由父设备的驱动代表CPU来执行间接访问
以I2C设备为例，每个设备分配一个地址，但没有与它相关的长度或范围，这与CPU地址分配很相似。

```
/ { 
    compatible = "acme,coyotes-revenge"; 
    #address-cells = <1>; 
    #size-cells = <1>; 
    cpus { 
        #address-cells = <1>;
        #size-cells = <0>; 
        cpu@0 { 
            compatible = "arm,cortex-a9"; 
            reg = <0>; 
        }; 
        cpu@1 { 
            compatible = "arm,cortex-a9"; 
            reg = <1>; 
        }; 
    }; 
    
    serial@101F0000 { 
        compatible = "arm,pl011"; 
        reg = <0x101f0000 0x1000>; 
    };
    
    serial@101F2000 { 
        compatible = "arm,pl011"; 
        reg = <0x101f2000 0x1000>; 
    };
    
    gpio@101F3000 { 
        compatible = "arm,pl061"; 
        reg = <0x101f3000 0x1000 0x101f4000 0x0010>
    };

    interrupt-controller@10140000 { 
        compatible = "arm,pl190"; 
        reg = <0x10140000 0x1000 >; 
    }; 

    spi@10115000 {
        compatible = "arm,pl022"; 
        reg = <0x10115000 0x1000 >; 
    };

    external-bus { 
        #address-cells = <2>;
        #size-cells = <1>;
        ethernet@0,0 { 
            compatible = "smc,smc91c111"; 
            reg = <0 0 0x1000>; 
        };
        i2c@1,0 { 
            compatible = "acme,a1234-i2c-bus"; 
            #address-cells = <1>; 
            #size-cells = <0>; 
            reg = <1 0 0x1000>; 
            rtc@58 { 
                compatible = "maxim,ds1338"; 
                reg = <58>; 
            };
        flash@2,0 { 
            compatible = "samsung,k8f1315ebm", "cfi-flash"; 
            reg = <2 0 0x4000000>; 
        };   
    };
}; 
```
#### ranges（地址翻译）
&emsp;&emsp;&emsp;&emsp;我们已经讨论过如何分配地址给设备，但在这些地址只是设备节点可见的，还没有描述如何将这些地址映射成CPU可使用的地址。根节点总是从CPU的角度描述地址空间。
&emsp;&emsp;&emsp;&emsp;如果根节点的子节点已经使用了CPU地址域，就不需要任何显式映射了，例如，串口serial@101f0000被直接分配到地址0x101f0000
```
/ { 
    compatible = "acme,coyotes-revenge"; 
    #address-cells = <1>; 
    #size-cells = <1>; 
    ... 
    external-bus { 
        #address-cells = <2> 
        #size-cells = <1>; 
        ranges = <0 0 0x10100000 0x10000 // Chipselect 1, Ethernet 
            1 0 0x10160000 0x10000 // Chipselect 2, i2c controller 
            2 0 0x30000000 0x1000000>; // Chipselect 3, NOR Flash 

        ethernet@0,0 { 
            compatible = "smc,smc91c111"; 
            reg = <0 0 0x1000>; 
        }
        
        i2c@1,0 { 
            compatible = "acme,a1234-i2c-bus"; 
            #address-cells = <1>; 
            #size-cells = <0>; 
            reg = <1 0 0x1000>; 
            rtc@58 { 
            compatible = "maxim,ds1338"; 
            reg = <58>; 
            }; 
        };

        flash@2,0 { 
            compatible = "samsung,k8f1315ebm", "cfi-flash"; 
            reg = <2 0 0x4000000>; 
        }; 
    }; 
}; 

```
ranges是地址翻译表，由3个数组成，即<子地址，父地址，区域大小>，分别对应子节点的#address-cells值，父节点的#address-cells值，子节点的#size-cells值确定。对于本例中的外部总线，子节点的地址是2个单元，父节点的地址是1个单元，子节点的区域大小是1个单元。

3个ranges被转换：

从片选0偏移0被映射到地址范围0x10100000~0x1010ffff

从片选1偏移0被映射到地址范围0x10160000~0x1016ffff

从片选2偏移0被映射到地址范围为0x30000000~0x30ffffff

另外，如果父节点和子节点的地址空间是相同的，那么一个节点可以添加一个空的ranges属性。一个空的ranges属性的存在意味着子节点地址空间的地址1：1地映射到父地址空间。你可能会问，你可能会问，为如果可以都用一一映射，为什么还需要地址转换？这是因为一些总线（如PCI ）具有完全不同的地址空间，其细节需要暴露给操作系统。其他有DMA engine的计算机需要知道总线上的真实地址。有时设备需要组合在一起，因为他们都有着相同的软件可编程的物理地址映射。是否需要一一映射取决于操作系统所需要的信息，以及硬件设计。

你也应该注意到，在i2c@1,0节点上没有ranges属性。这样做的原因是，不像外部总线，I2C总线上的设备不是内存映射到CPU地址域的。相反，CPU通过i2c@1,0设备间接访问rtc@58设备。ranges属性为空意味着设备不能被除了父设备以外的任何设备直接访问。

另举一例说明ranges属性
```
soc {
    compatible = "simple-bus";
    #address-cells = <1>;
    #size-cells = <1>;
    ranges = <0x0 0xe0000000 0x00100000>;
    serial {
        device_type = "serial";
        compatible = "ns16550";
        reg = <0x4600 0x100>;
        clock-frequency = <0>;
        interrupts = <0xA 0x8>;
        interrupt-parent = < &ipic >;
    }
}
```

soc的ranges属性指定了，从物理地址为0x0大小为1024KB的子节点映射到了物理地址为0xe0000000的父地址空间，有个这层映射关系，串口设备节点就可以通过load/store地址0xe0004600来访问，即0x4600的偏移+在ranges中指定的映射0xe0000000。

当然在64位操作系统中也会看到这样的映射，不要感到吃惊啦，"0xf 0x00000000"一起组成父节点地址即f00000000

dcsr: dcsr@f00000000 {

ranges = <0x0 0xf 0x00000000 0x01072000>;

};

#### 中断如何工作
&emsp;&emsp;&emsp;&emsp;与地址范围转换遵循树的天然结构不同，一台计算机的任何设备都可以发起和终止中断信号。不像设备编址，中断信号表现为独立于树的节点之间的链接。描述中断连接有4个属性：

interrupt-controller - 一个空的属性定义该节点为接收中断信号的设备
#interrupt-cells - 这是中断控制器节点的属性。它声明了中断控制器的中断说明符有多少个cell（类似#address-cells和#size-cells） 。
interrupt-parent - 设备节点的属性，包含一个指向该设备所连接中断控制器的pHandle。那些没有interrupt-parent属性节点则从它们的父节点继承该属性。
interrupts - 设备节点属性，包含中断说明符列表，对应于该设备上的每个中断输出信号。

中断说明符是一个或多个cell的数据（由#interrupt-cells指定），指定设备连接到哪些中断输入。下面的例子中，大多数设备只有一个中断输出，但也有一个设备上有多个中断输出的情况。一个中断符的含义完全取决于绑定的中断控制器设备。每个中断控制器可以决定它需要多少cell来唯一地确定一个中断输入。

下面的代码将中断添加到我们的Coyote's Revenge：
```
/ { 
    compatible = "acme,coyotes-revenge"; 
    #address-cells = <1>; 
    #size-cells = <1>; 
    interrupt-parent = <&intc>; 
    cpus { 
        #address-cells = <1>; 
        #size-cells = <0>; 
        cpu@0 { 
            compatible = "arm,cortex-a9"; 
            reg = <0>; 
        }; 

        cpu@1 { 
            compatible = "arm,cortex-a9"; 
            reg = <1>; 
        }; 
    }; 

    serial@101f0000 { 
        compatible = "arm,pl011"; 
        reg = <0x101f0000 0x1000 >; 
        interrupts = < 1 0 >; 
    }; 

    serial@101f2000 { 
        compatible = "arm,pl011"; 
        reg = <0x101f2000 0x1000 >; 
        interrupts = < 2 0 >; 
    }; 

    gpio@101f3000 { 
        compatible = "arm,pl061"; 
        reg = <0x101f3000 0x1000 
        0x101f4000 0x0010>; 
        interrupts = < 3 0 >; 
    }; 

    intc: interrupt-controller@10140000 { 
        compatible = "arm,pl190"; 
        reg = <0x10140000 0x1000 >; 
        interrupt-controller; 
        #interrupt-cells = <2>; 
    }; 

    spi@10115000 { 
        compatible = "arm,pl022"; 
        reg = <0x10115000 0x1000 >; 
        interrupts = < 4 0 >; 
    }; 

    external-bus { 
        #address-cells = <2> 
        #size-cells = <1>; 
        ranges = <0 0 0x10100000 0x10000 // Chipselect 1, Ethernet 
            1 0 0x10160000 0x10000 // Chipselect 2, i2c controller 
            2 0 0x30000000 0x1000000>; // Chipselect 3, NOR Flash 

        ethernet@0,0 { 
            compatible = "smc,smc91c111"; 
            reg = <0 0 0x1000>; 
            interrupts = < 5 2 >; 
        }; 

        i2c@1,0 { 
            compatible = "acme,a1234-i2c-bus"; 
            #address-cells = <1>; 
            #size-cells = <0>; 
            reg = <1 0 0x1000>; 
            interrupts = < 6 2 >; 
            rtc@58 { 
                compatible = "maxim,ds1338"; 
                reg = <58>; 
                interrupts = < 7 3 >; 
            }; 
        }; 

        flash@2,0 { 
            compatible = "samsung,k8f1315ebm", "cfi-flash"; 
            reg = <2 0 0x4000000>; 
        }; 
    }; 

}; 

```
#### 设备特有的数据
&emsp;&emsp;&emsp;&emsp;除了公共属性，任意属性和子节点都可以作为节点添加。只要遵循一些规则，操作系统所需要的任何数据可以被添加。首先，设备新的特有属性名应当使用制造商前缀，这样它们不会与现有的标准属性名称冲突。第二，属性和子节点的含义必须有文档来约束，这样一个设备驱动程序的作者才能知道如何解释这些数据。约束记录了一个特定的​​compatible值代表什么，它应该有什么样的属性，它可能有什么子节点，代表什么设备。每个唯一的compatible值应该有自己的约束（或者声明与其他compatible值的兼容性）。绑定新设备都记录在本wiki页。第三，发布新的约束到devicetree-discuss@lists.ozlabs.org邮件列表上进行审查。审查新的约束的确发现​了很多未来会导致问题的常见错误。

#### 特殊节点

#### 别名节点
&emsp;&emsp;&emsp;&emsp;特定节点通常通过完整路径引用，如/external-bus/ethernet@0,0，但当用户真正想要知道的是哪个设备是eth0时，这很不具有易读性，别名节点可分配一个短的alias给一个完整的设备路径。例如：
```
aliases { 
    ethernet0 = &ethernet0; 
    serial0 = &serial0; 
}; 
```
分配标识符给设备时，使用别名是受操作系统欢迎的。

这里使用了一个新的语法property = &label;该语法指定通过标签引用的完整节点路径为一个字符串属性。这与phandle = <&label>;不同，它是把一个pHandle值插入到一个cell。

#### 可选节点
&emsp;&emsp;&emsp;&emsp;可选节点并不代表真正的设备，而是作为固件和操作系统之间传递数据的地方，如启动参数。选择节点中的数据并不代表硬件。通常情况下，选择节点在DTS源文件中为空，并在开机时填充。
在我们的示例系统中，固件可以添加以下选择节点
```
chosen { 

    bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0,115200"; 

} 
```

{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [Warp Drive 的时间隧道][Markdown简明教程]
{% endhighlight %}

[helloanthea](http://my.csdn.net/helloanthea)
[http://blog.csdn.net/helloanthea/article/details/26677559](http://blog.csdn.net/helloanthea/article/details/26677559)
[wangbaolin719 ](http://blog.chinaunix.net/uid/27717694.html)
[wangbaolin719 ](http://blog.chinaunix.net/uid-27717694-id-4274992.html%3C/b%3E)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
