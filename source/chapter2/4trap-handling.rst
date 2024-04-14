.. _term-trap-handle:

实现特权级的切换
===========================

.. toctree::
   :hidden:
   :maxdepth: 5

本节导读
-------------------------------

由于处理器具有硬件级的特权级机制，应用程序在用户态特权级运行时，是无法直接通过函数调用访问处于内核态特权级的批处理操作系统内核中的函数。但应用程序又需要得到操作系统提供的服务，所以应用程序与操作系统需要通过某种合作机制完成特权级之间的切换，使得用户态应用程序可以得到内核态操作系统函数的服务。本节将讲解在 LoongArch 64 处理器提供的 PLV0/PLV3 特权级下，批处理操作系统和应用程序如何相互配合，完成特权级切换的。

LoongArch 特权级切换
---------------------------------------

特权级切换的起因
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
我们知道，批处理操作系统被设计为运行在内核态特权级（LoongArch 的 PLV0 模式），而应用程序被设计为运行在用户态特权级（LoongArch 的 PLV3 模式），被操作系统为核心的执行环境监管起来。在本章中，这个应用程序的执行环境即是由“邓式鱼”批处理操作系统提供的 AEE（Application Execution Environment）。批处理操作系统为了建立好应用程序的执行环境，需要在执行应用程序之前进行一些初始化工作，并监控应用程序的执行，具体体现在：

- 当启动应用程序的时候，需要初始化应用程序的用户态上下文，并能切换到用户态执行应用程序；
- 当应用程序发起系统调用（即发出 Trap）之后，需要到批处理操作系统中进行处理；
- 当应用程序执行出错的时候，需要到批处理操作系统中杀死该应用并加载运行下一个应用； 
- 当应用程序执行结束的时候，需要到批处理操作系统中加载运行下一个应用（实际上也是通过系统调用 ``sys_exit`` 来实现的）。

这些处理都涉及到特权级切换，因此需要应用程序、操作系统和硬件一起协同，完成特权级切换机制。


特权级切换相关的控制状态寄存器
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当从一般意义上讨论 LoongArch 架构的 Trap 机制时，通常需要注意两点：

- 在触发 Trap 之前 CPU 运行在哪个特权级；
- CPU 需要切换到哪个特权级来处理该 Trap ，并在处理完成之后返回原特权级。
  
但本章中我们仅考虑如下流程：当 CPU 在用户态特权级（ LoongArch 的 PLV3 模式）运行应用程序，执行到 Trap，切换到内核态特权级（ LoongArch 的 PLV0 模式），批处理操作系统的对应代码响应 Trap，并执行系统调用服务，处理完毕后，从内核态返回到用户态应用程序继续执行后续指令。

.. _term-s-mod-csr:

.. 在 RISC-V 架构中，关于 Trap 有一条重要的规则：在 Trap 前的特权级不会高于 Trap 后的特权级。因此如果触发 Trap 之后切换到 S 特权级（下称 Trap 到 S），说明 Trap 发生之前 CPU 只能运行在 S/U 特权级。

Trap 到 PLV0 特权级后，操作系统就会使用 PLV0 特权级中与 Trap 相关的 **控制状态寄存器** (CSR, Control and Status Register) 来辅助 Trap 处理。我们在编写运行在 PLV0 特权级的批处理操作系统中的 Trap 处理相关代码的时候，就需要使用如下所示的 CSR 寄存器。

.. list-table:: **Trap 的相关 CSR**
   :header-rows: 1
   :widths: 10 20 100

   * - CSR 名
     - 中文名
     - 该 CSR 与 Trap 相关的功能
   * - CRMD
     - 当前模式信息
     - ``PLV`` 字段表示当前特权等级。其合法的取值范围为 0~3。其中 0 表示最高特权等级，3 表示最低特权等级。当触发例外时，硬件将该域的值置为 0，以确保陷入后处于最高特权等级。
   * - PRMD
     - 例外前模式信息
     - 当触发例外时 ，如果例外类型不是 TLB 重填例外和机器错误例外 ，硬件会将CSR.CRMD 中 PLV 域的旧值记录在该寄存器的 ``PPLV`` 域。当所处理的例外既不是 TLB 重填例外（CSR.TLBRERA.IsTLBR=0）也不是机器错误例外（CSR.ERRCTL.IsMERR=0）时，执行 ERTN 指令从例外处理程序返回时，硬件会将这个域的值恢复到 CSR.CRMD 的 PLV 域。
   * - ERA
     - 例外程序返回地址
     - 当触发例外时，如果例外类型既不是 TLB 重填例外也不是机器错误例外，则触发例外的指令的 PC 将被记录在该寄存器中。
   * - ESTAT
     - 例外状态
     - ``Ecode`` 域和 ``EsubCode`` 域分别是例外类型一级和二级编码。触发例外时：如果是 TLB 重填例外或机器错误例外，该域保持不变；否则，硬件会根据例外类型将 :ref:`例外编码表 <term-exception>` 中的值写入该域。
   * - EENTRY
     - 例外入口地址
     - 该寄存器用于配置普通例外和中断的入口地址。其中 ``VPN`` 字段表示普通例外和中断入口地址所在页的页号。

.. .. note::

..    **S模式下最重要的 sstatus 寄存器**

..    注意 ``sstatus`` 是 S 特权级最重要的 CSR，可以从多个方面控制 S 特权级的 CPU 行为和执行状态。
   
.. chy   
   我们在这里先给出它在 Trap 处理过程中的作用。

特权级切换
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当执行一条 Trap 类指令（如 ``syscall`` 时），CPU 发现触发了一个异常并需要进行特殊处理，这涉及到 :ref:`执行环境切换 <term-ee-switch>` 。具体而言，用户态执行环境中的应用程序通过 ``syscall`` 指令向内核态执行环境中的操作系统请求某项服务功能，那么处理器和操作系统会完成到内核态执行环境的切换，并在操作系统完成服务后，再次切换回用户态执行环境，然后应用程序会紧接着 ``syscall`` 指令的后一条指令位置处继续执行，参考 :ref:`图示 <environment-call-flow>` 。

.. chy ？？？: 这条触发 Trap 的指令和进入 Trap 之前执行的最后一条指令不一定是同一条。

.. chy？？？ 下面的内容合并到第零章的os 抽象一节中， 执行环境切换, context等
    _term-execution-of-thread:
    回顾第一章的 :ref:`函数调用与栈 <function-call-and-stack>` ，我们知道在一个固定的 CPU 上，只要有一个栈作为存储空间，我们就能以多种
    普通控制流（顺序、分支、循环结构和多层嵌套函数调用）组合的方式，来一行一行的执行源代码（以编程语言级的视角），也是一条一条的执行汇编指令
    （以汇编语言级的视角）。只考虑普通控制流，那么从某条指令开始记录，该 CPU 可用的所有资源，包括自带的所有通用寄存器（包括虚拟的描述当前执行
    指令地址的寄存器 pc ）和当前特权级可用的 CSR 以及位于内存中的一块栈空间，它们会随着指令的执行而逐渐发生变化。这种局限在普通控制流（相对于 :ref:`异常控制流 <term-ecf>` 而言）之内的
    连续指令执行和与之同步的对相关资源的改变我们用一个新名词 **执行流** (Execution Flow) 来命名。执行流的状态是一个由它衍生出来的
    概念，表示截止到某条指令执行完毕所有相关资源（包括寄存器、栈）的状态集合，它完整描述了自记录起始之后该执行流的指令执行历史。
    .. note::
    实际上 CPU 还有其他资源可用：
    - 内存除了与执行流绑定的栈之外的其他存储空间，比如程序中的数据段；
    - 外围 I/O 设备。
    它们也会在执行期间动态发生变化。但它们可能由多条执行流共享，难以清晰的从中单独区分出某一条执行流的状态变化。因此在执行流概念中，
    我们不将其纳入考虑。

.. chy？？？ 内容与第一段有重复
    让我们通过从 U 特权级 Trap 到 S 特权级的切换过程（这是一个特例，实际上在 Trap 的前后，特权级也可以不变）来分析一下在 Trap 前后发生了哪些事情。首先假设CPU 正处在 U 特权级跑着应用程序的代码。在执行完某一条指令之后， CPU 发现一个
    中断/异常被触发，于是必须将应用执行流暂停，先 Trap 到更高的 S 特权级去执行批处理操作系统提供的相应服务代码，等待执行
    完了之后再回过头来恢复到并继续执行应用程序的执行流。
.. chy？？？  内容与第一段有重复
    我们可以将 CPU 在 S 特权级执行的那一段指令序列也看成一个控制流，因为它全程只是以普通控制流的模式在 S 特权级执行。这个控制流的意义就在于
    处理 Trap ，我们可以将其称之为 **Trap 专用** 控制流，它在 Trap 触发的时候开始，并于 Trap 处理完毕之后结束。于是我们可以从执行环境切换和控制流的角度来看待 
    操作系统对Trap 的整个处理过程： CPU 从用户态执行环境中的应用程序的普通控制流转到内核态执行环境中的 Trap 执行流，然后再切换回去继续运行应用程序。站在应用程序的角度， 由操作系统和CPU协同完成的Trap 机制对它是完全透明的，无论应用程序在它的哪一条指令执行结束后进入 Trap ，它总是相信在 Trap 结束之后 CPU 能够在与被打断的时候"相同"的执行环境中继续正确的运行应用程序的指令。
.. chy？？？ 
    .. note::
.. chy？？？  内容与第一段有重复
        这里所说的相同并不是绝对相同，但是其变化是完全能够被应用程序预知到的。比如应用程序通过 ``ecall`` 指令请求底层高特权级软件的功能，
        由调用规范它知道 Trap 之后 ``a0~a1`` 两个寄存器会被用来保存返回值，所以会发生变化。这个信息是应用程序明确知晓的，但某种程度上
        确实也体现了执行流的变化。

应用程序被切换回来之后需要从发出系统调用请求的执行位置恢复应用程序上下文并继续执行，这需要在切换前后维持应用程序的上下文保持不变。应用程序的上下文包括通用寄存器和栈两个主要部分。由于 CPU 在不同特权级下共享一套通用寄存器，所以在运行操作系统的 Trap 处理过程中，操作系统也会用到这些寄存器，这会改变应用程序的上下文。因此，与函数调用需要保存函数调用上下文/活动记录一样，在执行操作系统的 Trap 处理过程（会修改通用寄存器）之前，我们需要在某个地方（某内存块或内核的栈）保存这些寄存器并在 Trap 处理结束后恢复这些寄存器。

除了通用寄存器之外还有一些可能在处理 Trap 过程中会被修改的 CSR，比如 CPU 所在的特权级。我们要保证它们的变化在我们的预期之内。比如，对于特权级转换而言，应该是 Trap 之前在 PLV3 特权级，处理 Trap 的时候在 PLV0 特权级，返回之后又需要回到 PLV3 特权级。而对于栈问题则相对简单，只要两个应用程序执行过程中用来记录执行历史的栈所对应的内存区域不相交，就不会产生令我们头痛的覆盖问题或数据破坏问题，也就无需进行保存/恢复。

特权级切换的具体过程一部分由硬件直接完成，另一部分则需要由操作系统来实现。

.. _trap-hw-mechanism:

特权级切换的硬件控制机制
-------------------------------------

当 CPU 执行完一条指令（如 ``syscall`` ）并准备从用户特权级 陷入（ ``Trap`` ）到 PLV0 特权级的时候，硬件会自动完成如下这些事情：

- 首先，需要记录被异常打断的指令的地址（记为 EPTR） 。异常处理结束后将返回 EPTR 所在地址，重新执行被异常打断的指令。
- 其次，调整 CPU 的权限等级（通常调整至最高特权等级）并关闭中断响应。在 LoongArch 指令系统中，当异常发生时，硬件会将 CSR.PLV 置 0 以进入最高特权等级，并将 CSR.CRMD 的 IE 域置 0 以屏蔽所有中断输入。
- 再次，硬件保存异常发生现场的部分信息。在 LoongArch 指令系统中，异常发生时会将 CSR.CRMD 中的 PLV 和 IE 域的旧值分别记录到 CSR.PRMD 的 PPLV 和 PIE 域中，供后续异常返回时使用。
- 最后，记录异常的相关信息。异常处理程序将利用这些信息完成或加速异常的处理。最常见的如记录异常编号以用于确定异常来源。在 LoongArch 指令系统中，这一信息将被记录在 CSR.ESTAT 的 Ecode 和 EsubCode 域，前者存放异常的一级编号，后者存放异常的二级编号。除此以外，有些情况下还会将引发异常的指令的机器码记录在 CSR.BADI 中，或是将造成异常的访存虚地址记录在 CSR.BADV 中。

不同类型的异常需要各自对应的异常处理。处理器确定异常来源主要有两种方式：一种是将不同的异常进行编号，异常处理程序据此进行区分并跳转到指定的处理入口；另一种是为不同的异常指定不同的异常处理程序入口地址，这样每个入口处的异常处理程序自然知晓待处理的异常来源。X86 由硬件进行异常和中断号的查询，根据编号查询预设好的中断描述符表（Interrupt Descriptor Table，简称 IDT），得到不同异常处理的入口地址，并将 CS/EIP 等压栈。LoongArch 将不同的异常进行编号，其异常处理程序入口地址采用 “入口页号与页内偏移进行按位逻辑或” 的计算方式，入口页号通过 CSR.EENTRY 配置，每个普通异常处理程序入口页内偏移是其异常编号乘以一个可配置间隔（通过 CSR.ECFG 的 VS 域配置）。通过合理配置 EENTRY 和 ECFG 控制状态寄存器中相关的域，可以使得不同异常处理程序入口地址不同。当然，也可以通过配置使得所有异常处理程序入口为同一个地址。为了简单起见，我们在此使所有异常处理程序入口为同一地址。

.. .. note::

..    **stvec 相关细节**

..    在 RV64 中， ``stvec`` 是一个 64 位的 CSR，在中断使能的情况下，保存了中断处理的入口地址。它有两个字段：

..    - MODE 位于 [1:0]，长度为 2 bits；
..    - BASE 位于 [63:2]，长度为 62 bits。

..    当 MODE 字段为 0 的时候， ``stvec`` 被设置为 Direct 模式，此时进入 S 模式的 Trap 无论原因如何，处理 Trap 的入口地址都是 ``BASE<<2`` ， CPU 会跳转到这个地方进行异常处理。本书中我们只会将 ``stvec`` 设置为 Direct 模式。而 ``stvec`` 还可以被设置为 Vectored 模式，有兴趣的同学可以自行参考 RISC-V 指令集特权级规范。

.. note:: 

    **EENTRY 相关细节**

    该寄存器用于配置普通例外和中断的入口地址。

    .. list-table::
        :align: center
        :header-rows: 1
        :widths: 20 20 20 40

        * - 位
          - 名字
          - 读写
          - 描述
        * - 11:0
          - 0
          - R
          - 只读恒为0，写被忽略。
        * - GRLEN-1:12
          - VPN
          - RW
          - 普通例外和中断入口地址所在页的页号。

    从表中可以看出，EENTRY 寄存器默认操作系统对内存采用分页管理，但目前我们还没有实现内存管理模块，所以我们还是直接通过物理地址访问该入口地址。这并不难做到，我们只要将处理程序的入口地址按 4KB 对齐即可，然后截取该物理地址的高位（去掉低 12 位的 0）存入 EENTRY 的 VPN 字段即可。

而当 CPU 完成 Trap 处理准备返回的时候，需要通过一条 PLV0 特权级的特权指令 ``ertn`` 来完成，这一条指令具体完成以下功能：

- CPU 会将当前的特权级按照 ``PRMD`` 的 ``PPLV`` 字段设置为 PLV0 或者 PLV3 ；
- CPU 会跳转到 ``ERA`` 寄存器指向的那条指令，然后继续执行。

这些基本上都是硬件不得不完成的事情，还有一些剩下的收尾工作可以都交给软件，让操作系统能有更大的灵活性。

用户栈与内核栈
--------------------------------

在 Trap 触发的一瞬间， CPU 就会切换到 PLV0 特权级并跳转到 ``EENTRY`` 所指示的位置。但是在正式进入 PLV0 特权级的 Trap 处理之前，上面
提到过我们必须保存原控制流的寄存器状态，这一般通过内核栈来保存。注意，我们需要用专门为操作系统准备的内核栈，而不是应用程序运行时用到的用户栈。

.. 
    chy:我们在一个作为用户栈的特别留出的内存区域上保存应用程序的栈信息，而 Trap 执行流则使用另一个内核栈。

使用两个不同的栈主要是为了安全性：如果两个控制流（即应用程序的控制流和内核的控制流）使用同一个栈，在返回之后应用程序就能读到 Trap 控制流的历史信息，比如内核一些函数的地址，这样会带来安全隐患。于是，我们要做的是，在批处理操作系统中添加一段汇编代码，实现从用户栈切换到内核栈，并在内核栈上保存应用程序控制流的寄存器状态。

我们声明两个类型 ``KernelStack`` 和 ``UserStack`` 分别表示内核栈和用户栈，它们都只是字节数组的简单包装：

.. code-block:: rust
    :linenos:

    // os/src/batch.rs

    const USER_STACK_SIZE: usize = 4096 * 2;
    const KERNEL_STACK_SIZE: usize = 4096 * 2;

    #[repr(align(4096))]
    struct KernelStack {
        data: [u8; KERNEL_STACK_SIZE],
    }

    #[repr(align(4096))]
    struct UserStack {
        data: [u8; USER_STACK_SIZE],
    }

    static KERNEL_STACK: KernelStack = KernelStack { data: [0; KERNEL_STACK_SIZE] };
    static USER_STACK: UserStack = UserStack { data: [0; USER_STACK_SIZE] };

常数 ``USER_STACK_SIZE`` 和 ``KERNEL_STACK_SIZE`` 指出用户栈和内核栈的大小分别为 :math:`8\text{KiB}` 。两个类型是以全局变量的形式实例化在批处理操作系统的 ``.bss`` 段中的。

我们为两个类型实现了 ``get_sp`` 方法来获取栈顶地址。由于在 LoongArch 中栈是向下增长的，我们只需返回包裹的数组的结尾地址，以用户栈类型 ``UserStack`` 为例：

.. code-block:: rust
    :linenos:

    impl UserStack {
        fn get_sp(&self) -> usize {
            self.data.as_ptr() as usize + USER_STACK_SIZE
        }
    }

于是换栈是非常简单的，只需将 ``sp`` 寄存器的值修改为 ``get_sp`` 的返回值即可。

.. _term-trap-context:

接下来是Trap上下文（即数据结构 ``TrapContext`` ），类似前面提到的函数调用上下文，即在 Trap 发生时需要保存的物理资源内容，并将其一起放在一个名为 ``TrapContext`` 的类型中，定义如下：

.. code-block:: rust
    :linenos:

    // os/src/trap/context.rs

    #[repr(C)]
    pub struct TrapContext {
        pub r: [usize; 32],
        pub prmd: Prmd,
        pub era: usize,
    }

可以看到里面包含所有的通用寄存器 ``r0~r31`` ，还有 ``PRMD`` 和 ``ERA`` 。那么为什么需要保存它们呢？

- 对于通用寄存器而言，两条控制流（应用程序控制流和内核控制流）运行在不同的特权级，所属的软件也可能由不同的编程语言编写，虽然在 Trap 控制流中只是会执行 Trap 处理相关的代码，但依然可能直接或间接调用很多模块，因此很难甚至不可能找出哪些寄存器无需保存。既然如此我们就只能全部保存了。但这里也有一些例外，如 ``r0`` 被硬编码为 0 ，它自然不会有变化；还有 ``tp(r2)`` 寄存器，除非我们手动出于一些特殊用途使用它，否则一般也不会被用到。虽然它们无需保存，但我们仍然在 ``TrapContext`` 中为它们预留空间，主要是为了后续的实现方便。
- 对于 CSR 而言，我们知道进入 Trap 的时候，硬件会立即覆盖掉 ``ESTAT/PRMD/ERA`` 的全部或是其中一部分。 ``ESTAT`` 的情况是：它总是在 Trap 处理的第一时间就被使用或者是在其他地方保存下来了，因此它没有被修改并造成不良影响的风险。而对于 ``PRMD/ERA`` 而言，它们会在 Trap 处理的全程有意义（在 Trap 控制流最后 ``ertn`` 的时候还用到了它们），而且确实会出现 Trap 嵌套的情况使得它们的值被覆盖掉。所以我们需要将它们也一起保存下来，并在 ``ertn`` 之前恢复原样。


Trap 管理
-------------------------------

特权级切换的核心是对 Trap 的管理。这主要涉及到如下一些内容：

- 应用程序通过 ``syscall`` 进入到内核状态时，操作系统保存被打断的应用程序的 Trap 上下文；
- 操作系统根据 Trap 相关的 CSR 寄存器内容，完成系统调用服务的分发与处理；
- 操作系统完成系统调用服务后，需要恢复被打断的应用程序的 Trap 上下文，并通 ``ertn`` 让应用程序继续执行。

接下来我们具体介绍上述内容。

Trap 上下文的保存与恢复
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

首先是具体实现 Trap 上下文保存和恢复的汇编代码。

.. _trap-context-save-restore: 

在批处理操作系统初始化的时候，我们需要修改 ``EENTRY`` 寄存器来指向正确的 Trap 处理入口点。

.. code-block:: rust
    :linenos:

    // os/src/trap/mod.rs

    global_asm!(include_str!("trap.S"));

    pub fn init() {
        extern "C" { fn __alltraps(); }
        unsafe {
            eentry::write(__alltraps as usize >> 12);
        }
    }

这里我们引入了一个外部符号 ``__alltraps`` ，并将 ``__alltraps`` 的地址右移 12 位来模拟页号。我们在 ``os/src/trap/trap.S`` 中实现 Trap 上下文保存/恢复的汇编代码，分别用外部符号 ``__alltraps`` 和 ``__restore`` 标记为函数，并通过 ``global_asm!`` 宏将 ``trap.S`` 这段汇编代码插入进来。

Trap 处理的总体流程如下：首先通过 ``__alltraps`` 将 Trap 上下文保存在内核栈上，然后跳转到使用 Rust 编写的 ``trap_handler`` 函数完成 Trap 分发及处理。当 ``trap_handler`` 返回之后，使用 ``__restore`` 从保存在内核栈上的 Trap 上下文恢复寄存器。最后通过一条 ``ertn`` 指令回到应用程序执行。

首先是保存 Trap 上下文的 ``__alltraps`` 的实现：

.. code-block:: 
    :linenos:

    # os/src/trap/trap.S

    .macro SAVE_GP n
        st.d $r\n, $sp, \n*8
    .endm

        .section .text.trap

    __alltraps:
        csrwr $sp, 0x30
        # now sp->kernel stack, save0->user stack
        # allocate a TrapContext on kernel stack
        addi.d $sp, $sp, -34*8
        # save general-purpose registers
        st.d $r1, $sp, 1*8
        # skip tp(r2), application does not use it
        # skip sp(r3), we will save it later
        # save r4~r31
        .set n, 4
        .rept 28
            SAVE_GP %n
            .set n, n+1
        .endr
        # we can use t0/t1/t2 freely, because they were saved on kernel stack
        csrrd $t0, 0x1
        csrrd $t1, 0x6
        st.d $t0, $sp, 32*8
        st.d $t1, $sp, 33*8
        # read user stack from save0 and save it on the kernel stack
        csrrd $r2, 0x30
        st.d $r2, $sp, 3*8
        # set input argument of trap_handler(cx: &mut TrapContext)
        move $a0, $sp
        bl trap_handler

.. - 第 7 行我们使用 ``.align`` 将 ``__alltraps`` 的地址 4 字节对齐，这是 RISC-V 特权级规范的要求；
- 第 10 行的 ``csrwr`` 原型是 :math:`\text{csrwr rd, csr_num}` 。该指令将通用寄存器 ``rd`` 中的旧值写入到指定 CSR 中，同时将指定 CSR 的旧值更新到通用寄存器 ``rd`` 中。因此这里起到的是交换 0x30 号 CSR 和 sp 的效果。该寄存器名为 save0，在这一行之前 sp 指向用户栈， save0 指向内核栈（原因稍后说明），现在 sp 指向内核栈， save0 指向用户栈。
- 第 13 行，我们准备在内核栈上保存 Trap 上下文，于是预先分配 :math:`34\times 8` 字节的栈帧，这里改动的是 sp ，说明确实是在内核栈上。
- 第 14~23 行，保存 Trap 上下文的通用寄存器 r0~r31，跳过 r0 和 tp(r2)，原因之前已经说明。我们在这里也不保存 sp(r3)，因为我们要基于它来找到每个寄存器应该被保存到的正确的位置。实际上，在栈帧分配之后，我们可用于保存 Trap 上下文的地址区间为 :math:`[\text{sp},\text{sp}+8\times34)` ，按照  ``TrapContext`` 结构体的内存布局，基于内核栈的位置（sp所指地址）来从低地址到高地址分别按顺序放置 r0~r31 这些通用寄存器，最后是 prmd 和 era 。因此通用寄存器 rn 应该被保存在地址区间 :math:`[\text{sp}+8n,\text{sp}+8(n+1))` 。

  为了简化代码，r4~r31 这 28 个通用寄存器我们通过类似循环的 ``.rept`` 每次使用 ``SAVE_GP`` 宏来保存，其实质是相同的。注意我们需要在 ``trap.S`` 开头加上 ``.altmacro`` 才能正常使用 ``.rept`` 命令。虽然目前 LoongArch 架构中的 r21 为 Reserved，但我们仍将其保存，以防止之后该寄存器成为一个需要保存的寄存器。
- 第 25~28 行，我们将 CSR prmd 和 era 的值分别读到寄存器 t0 和 t1 中然后保存到内核栈对应的位置上。指令 :math:`\text{csrrd rd, csr_num}`  的功能就是将 CSR 的值读到寄存器 :math:`\text{rd}` 中。这里我们不用担心 t0 和 t1 被覆盖，因为它们刚刚已经被保存了。
- 第 30~31 行专门处理 sp 的问题。首先将 save0 的值读到寄存器 t2 并保存到内核栈上，注意： save0 的值是进入 Trap 之前的 sp 的值，指向用户栈。而现在的 sp 则指向内核栈。
- 第 33 行令 :math:`\text{a}_0\leftarrow\text{sp}`，让寄存器 a0 指向内核栈的栈指针也就是我们刚刚保存的 Trap 上下文的地址，这是由于我们接下来要调用 ``trap_handler`` 进行 Trap 处理，它的第一个参数 ``cx`` 由调用规范要从 a0 中获取。而 Trap 处理函数 ``trap_handler`` 需要 Trap 上下文的原因在于：它需要知道其中某些寄存器的值，比如在系统调用的时候应用程序传过来的 syscall ID 和对应参数。我们不能直接使用这些寄存器现在的值，因为它们可能已经被修改了，因此要去内核栈上找已经被保存下来的值。

最后还需注意，为了让上面的代码模拟按页存储，需要将其首行代码地址按 4KB（页大小）对齐。为此，我们定义上述代码的 section 为 ``.text.trap`` ，并对链接脚本进行如下修改：

.. code-block:: 

    .text : {
        *(.text.entry)
        . = ALIGN(4k);
        *(.text.trap)
        *(.text .text.*)
    }

这样，上述代码的第一条指令的地址就按 4KB 对齐了， ``__alltraps`` 所指示的地址自然也是按 4KB 对齐的，我们将该地址右移 12 位就可以得到所谓的页号。

.. _term-atomic-instruction:

.. note::

    **CSR 相关原子指令**

    LoongArch 中读写 CSR 的指令是一类能不会被打断地完成多个读写操作的指令。这种不会被打断地完成多个操作的指令被称为 **原子指令** (Atomic Instruction)。这里的 **原子** 的含义是“不可分割的最小个体”，也就是说指令的多个操作要么都不完成，要么全部完成，而不会处于某种中间状态。

    另外，LoongArch 架构中常规的数据处理和访存类指令只能操作通用寄存器而不能操作 CSR 。因此，当想要对 CSR 进行操作时，需要先使用读取 CSR 的指令将 CSR 读到一个通用寄存器中，而后操作该通用寄存器，最后再使用写入 CSR 的指令将该通用寄存器的值写入到 CSR 中。

当 ``trap_handler`` 返回之后会从调用 ``trap_handler`` 的下一条指令开始执行，也就是从栈上的 Trap 上下文恢复的 ``__restore`` ：

.. _code-restore:

.. code-block:: 
    :linenos:

    # os/src/trap/trap.S

    .macro LOAD_GP n
        ld.d $r\n, $sp, \n*8
    .endm

    __restore:
        # case1: start running app by __restore
        # case2: back to PLV3 after handling trap
        move $sp, $a0
        # now sp->kernel stack(after allocated), save0->user stack
        # restore prmd/era
        ld.d $t0, $sp, 32*8
        ld.d $t1, $sp, 33*8
        ld.d $t2, $sp, 3*8
        csrwr $t0, 0x1
        csrwr $t1, 0x6
        csrwr $t2, 0x30
        # restore general-purpuse registers except sp/tp
        ld.d $r1, $sp, 1*8
        # ld.d $r3, $sp, 3*8
        .set n, 4
        .rept 28
            LOAD_GP %n
            .set n, n+1
        .endr
        # release TrapContext on kernel stack
        addi.d $sp, $sp, 34*8
        # now sp->kernel stack, save0->user stack
        csrwr $sp, 0x30
        ertn

- 第 10 行比较奇怪我们暂且不管，假设它从未发生，那么 sp 仍然指向内核栈的栈顶。
- 第 13~26 行负责从内核栈顶的 Trap 上下文恢复通用寄存器和 CSR 。注意我们要先恢复 CSR 再恢复通用寄存器，这样我们使用的三个临时寄存器才能被正确恢复。
- 在第 28 行之前，sp 指向保存了 Trap 上下文之后的内核栈栈顶， save0 指向用户栈栈顶。我们在第 28 行在内核栈上回收 Trap 上下文所占用的内存，回归进入 Trap 之前的内核栈栈顶。第 30 行，再次交换 save0 和 sp，现在 sp 重新指向用户栈栈顶，save0 也依然保存进入 Trap 之前的状态并指向内核栈栈顶。
- 在应用程序控制流状态被还原之后，第 31 行我们使用 ``ertn`` 指令回到 PLV3 特权级继续运行应用程序控制流。

.. note::

    **SAVE CSR 的用途**

    在特权级切换的时候，我们需要将 Trap 上下文保存在内核栈上，因此需要一个寄存器暂存内核栈地址，并以它作为基地址指针来依次保存 Trap 上下文的内容。但是所有的通用寄存器都不能够用作基地址指针，因为它们都需要被保存，如果覆盖掉它们，就会影响后续应用控制流的执行。

    事实上我们缺少了一个重要的中转寄存器，这时我们就可以利用 CSR 中的 SAVE 寄存器。SAVE 寄存器，即数据保存控制状态寄存器，用于给系统软件暂存数据。每个数据保存寄存器可以存放一个通用寄存器的数据。数据保存寄存器最少实现 1 个，最多实现 16 个。具体实现的个数软件可以从 CSR.PRCFG1.SAVENum 中获知。从 SAVE0 开始，各个 SAVE 寄存器的地址依次为 0x30、0x31、……、0x30+SAVENum-1。除执行 CSR 指令外，硬件不会修改该寄存器的内容。
    
    在这里，我们使用了 SAVE0 寄存器。从上面的汇编代码中可以看出，在保存 Trap 上下文的时候，它起到了两个作用：首先是保存了内核栈的地址，其次它可作为一个中转站让 ``sp`` （目前指向的用户栈的地址）的值可以暂时保存在 ``save0`` 。这样仅需一条 ``csrwr $sp, 0x30`` 指令（交换对 ``sp`` 和 ``save0`` 两个寄存器内容）就完成了从用户栈到内核栈的切换，这是一种极其精巧的实现。

Trap 分发与处理
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Trap 在使用 Rust 实现的 ``trap_handler`` 函数中完成分发和处理：

.. code-block:: rust
    :linenos:

    // os/src/trap/mod.rs

    #[no_mangle]
    pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext {
        let estat = estat::read();
        match estat.cause() {
            Trap::Exception(Exception::SYS) => {
                cx.era += 4;
                cx.r[4] = syscall(cx.r[11], [cx.r[4], cx.r[5], cx.r[6]]) as usize;
            }
            Trap::Exception(Exception::PIS) => {
                println!("[kernel] Trap::Exception(Exception::PIS) Invalid store operation page exception in application, kernel killed it.");
                run_next_app();
            }
            Trap::Exception(Exception::IPE) => {
                println!("[kernel] Trap::Exception(Exception::IPE) Instruction privilege level exception in application, kernel killed it.");
                run_next_app();
            }
            _ => {
                panic!(
                    "Unsupported trap {:?}!",
                    estat.cause()
                );
            }
        }
        cx
    }

- 第 4 行声明返回值为 ``&mut TrapContext`` 并在第 26 行实际将传入的 Trap 上下文 ``cx`` 原样返回，因此在 ``__restore`` 的时候 ``a0`` 寄存器在调用 ``trap_handler`` 前后并没有发生变化，仍然指向分配 Trap 上下文之后的内核栈栈顶，和此时 ``sp`` 的值相同，这里的 :math:`\text{sp}\leftarrow\text{a}_0` 并不会有问题；
- 第 6 行根据 ``estat`` 寄存器所保存的 Trap 的原因进行分发处理。由于在整个系统实现中我们经常会涉及到类似于读写各种控制状态寄存器以及操作内存管理部件等直接操作硬件的情况，所以我们将这些直接操作硬件的代码封装在一个名为 loongarch 的 Rust 包当中，感兴趣的同学可以阅读并共建该项目。在这里我们直接引入该库使用即可，为此，我们需要在配置文件中添加依赖：

  .. code-block:: toml

      # os/Cargo.toml
      
      [dependencies]
      loongarch = { git = "https://github.com/Helloworld-lbl/loongarch", features = ["inline-asm"] }
    
- 第 7~10 行，发现触发 Trap 的原因是来自 PLV3 特权级的 Environment Call，也就是系统调用。这里我们首先修改保存在内核栈上的 Trap 上下文里面 era，让其增加 4。这是因为我们知道这是一个由 ``syscall`` 指令触发的系统调用，在进入 Trap 的时候，硬件会将 era 设置为这条 ``syscall`` 指令所在的地址（因为它是进入 Trap 之前最后一条执行的指令）。而在 Trap 返回之后，我们希望应用程序控制流从 ``syscall`` 的下一条指令开始执行。因此我们只需修改 Trap 上下文里面的 era，让它增加 ``syscall`` 指令的码长，也即 4 字节。这样在 ``__restore`` 的时候 era 在恢复之后就会指向 ``syscall`` 的下一条指令，并在 ``ertn`` 之后从那里开始执行。

  用来保存系统调用返回值的 a0 寄存器也会同样发生变化。我们从 Trap 上下文取出作为 syscall ID 的 a7 和系统调用的三个参数 a0~a2 传给 ``syscall`` 函数并获取返回值。 ``syscall`` 函数是在 ``syscall`` 子模块中实现的。 这段代码是处理正常系统调用的控制逻辑。
- 第 11~18 行，分别处理应用程序出现 store 操作页无效例外和指令特权等级错例外的情形。此时需要打印错误信息并调用 ``run_next_app`` 直接切换并运行下一个应用程序。
- 第 20 行开始，当遇到目前还不支持的 Trap 类型的时候，“邓式鱼” 批处理操作系统整个 panic 报错退出。



实现系统调用功能
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于系统调用而言， ``syscall`` 函数并不会实际处理系统调用，而只是根据 syscall ID 分发到具体的处理函数：

.. code-block:: rust
    :linenos:

    // os/src/syscall/mod.rs

    pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
        match syscall_id {
            SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
            SYSCALL_EXIT => sys_exit(args[0] as i32),
            _ => panic!("Unsupported syscall_id: {}", syscall_id),
        }
    }

这里我们会将传进来的参数 ``args`` 转化成能够被具体的系统调用处理函数接受的类型。它们的实现都非常简单：

.. code-block:: rust
    :linenos:

    // os/src/syscall/fs.rs

    const FD_STDOUT: usize = 1;

    pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize {
        match fd {
            FD_STDOUT => {
                let slice = unsafe { core::slice::from_raw_parts(buf, len) };
                let str = core::str::from_utf8(slice).unwrap();
                print!("{}", str);
                len as isize
            },
            _ => {
                panic!("Unsupported fd in sys_write!");
            }
        }
    }

    // os/src/syscall/process.rs

    pub fn sys_exit(xstate: i32) -> ! {
        println!("[kernel] Application exited with code {}", xstate);
        run_next_app()
    }

- ``sys_write`` 我们将传入的位于应用程序内的缓冲区的开始地址和长度转化为一个字符串 ``&str`` ，然后使用批处理操作系统已经实现的 ``print!`` 宏打印出来。注意这里我们并没有检查传入参数的安全性，即使会在出错严重的时候 panic，还是会存在安全隐患。这里我们出于实现方便暂且不做修补。
- ``sys_exit`` 打印退出的应用程序的返回值并同样调用 ``run_next_app`` 切换到下一个应用程序。

.. _ch2-app-execution:

执行应用程序
-------------------------------------

当批处理操作系统初始化完成，或者是某个应用程序运行结束或出错的时候，我们要调用 ``run_next_app`` 函数切换到下一个应用程序。此时 CPU 运行在 PLV0 特权级，而它希望能够切换到 PLV3 特权级。在 LoongArch 架构中，我们可以通过执行 Trap 返回的特权指令使得 CPU 特权级下降，即 ``ertn`` 。事实上，在从操作系统内核返回到运行应用程序之前，要完成如下这些工作：


- 构造应用程序开始执行所需的 Trap 上下文；
- 通过 ``__restore`` 函数，从刚构造的 Trap 上下文中，恢复应用程序执行的部分寄存器；
- 设置 ``era`` CSR的内容为应用程序入口点 ``0x300000``；
- 切换 ``save0`` 和 ``sp`` 寄存器，设置 ``sp`` 指向应用程序用户栈；
- 执行 ``sret`` 从 PLV0 特权级切换到 PLV3 特权级。


它们可以通过复用 ``__restore`` 的代码来更容易的实现上述工作。我们只需要在内核栈上压入一个为启动应用程序而特殊构造的 Trap 上下文，再通过 ``__restore`` 函数，就能让这些寄存器到达启动应用程序所需要的上下文状态。

.. code-block:: rust
    :linenos:

    // os/src/trap/context.rs

    impl TrapContext {
        pub fn set_sp(&mut self, sp: usize) { self.r[3] = sp; }
        pub fn app_init_context(entry: usize, sp: usize) -> Self {
            let mut prmd = prmd::read();
            prmd.set_pplv(PLV::PLV3);
            let mut cx = Self {
                r: [0; 32],
                prmd,
                era: entry,
            };
            cx.set_sp(sp);
            cx
        }
    }

为 ``TrapContext`` 实现 ``app_init_context`` 方法，修改其中的 era 寄存器为应用程序入口点 ``entry``， sp 寄存器为我们设定的一个栈指针，并将 prmd 寄存器的 ``PPLV`` 字段设置为 PLV3 。

在 ``run_next_app`` 函数中我们能够看到：

.. code-block:: rust
    :linenos:
    :emphasize-lines: 14,15,16,17,18

    // os/src/batch.rs

    pub fn run_next_app() -> ! {
        let mut app_manager = APP_MANAGER.exclusive_access();
        let current_app = app_manager.get_current_app();
        unsafe {
            app_manager.load_app(current_app);
        }
        app_manager.move_to_next_app();
        drop(app_manager);
        // before this we have to drop local variables related to resources manually
        // and release the resources
        extern "C" { fn __restore(cx_addr: usize); }
        unsafe {
            __restore(KERNEL_STACK.push_context(
                TrapContext::app_init_context(APP_BASE_ADDRESS, USER_STACK.get_sp())
            ) as *const _ as usize);
        }
        panic!("Unreachable in batch::run_current_app!");
    }

在高亮行所做的事情是在内核栈上压入一个 Trap 上下文，其 ``era`` 是应用程序入口地址 ``0x300000`` ，其 ``sp`` 寄存器指向用户栈，其 ``prmd`` 的 ``PPLV`` 字段被设置为 PLV3 。 ``push_context`` 的返回值是内核栈压入 Trap 上下文之后的栈顶，它会被作为 ``__restore`` 的参数（回看 :ref:`__restore 代码 <code-restore>` ，这时我们可以理解为何 ``__restore`` 函数的起始部分会完成 :math:`\text{sp}\leftarrow\text{a}_0` ），这使得在 ``__restore`` 函数中 ``sp`` 仍然可以指向内核栈的栈顶。这之后，就和执行一次普通的 ``__restore`` 函数调用一样了。

.. note::

    有兴趣的同学可以思考： save0 是何时被设置为内核栈顶的？



.. 
   马老师发生甚么事了？
   --
   这里要说明目前只考虑从 U Trap 到 S ，而实际上 Trap 的要素就有：Trap 之前在哪个特权级，Trap 在哪个特权级处理。这个对于中断和异常
   都是如此，只不过中断可能跟特权级的关系稍微更紧密一点。毕竟中断的类型都是跟特权级挂钩的。但是对于 Trap 而言有一点是共同的，也就是触发 
   Trap 不会导致优先级下降。从中断/异常的代理就可以看出从定义上就不允许代理到更低的优先级。而且代理只能逐级代理，目前我们能操作的只有从 
   M 代理到 S，其他代理都基本只出现在指令集拓展或者硬件还不支持。中断的情况是，如果是属于某个特权级的中断，不能在更低的优先级处理。事实上
   这个中断只可能在 CPU 处于不会更高的优先级上收到（否则会被屏蔽），而 Trap 之后优先级不会下降（Trap 代理机制决定），这样就自洽了。
   --
   之前提到异常是说需要执行环境功能的原因与某条指令的执行有关。而 Trap 的定义更加广泛一些，就是在执行某条指令之后发现需要执行环境的功能，
   如果是中断的话 Trap 回来之后默认直接执行下一条指令，如果是异常的话硬件会将 sepc 设置为 Trap 发生之前最后执行的那条指令，而异常发生
   的原因不一定和这条指令的执行有关。应该指出的是，在大多数情况下都是和最后这条指令的执行有关。但在缓存的作用下也会出现那种特别极端的情况。
   --
   然后是 Trap 到 S，就有 S 模式的一些相关 CSR，以及从 U Trap 到 S，硬件会做哪些事情（包括触发异常的一瞬间，以及处理完成调用 sret 
   之后）。然后指出从用户的视角来看，如果是 ecall 的话， Trap 回来之后应该从 ecall 的下一条指令开始执行，且执行现场不能发生变化。
   所以就需要将应用执行环境保存在内核栈上（还需要换栈！）。栈存在的原因可能是 Trap handler 是一条新的运行在 S 特权级的执行流，所以
   这个可以理解成跨特权级的执行流切换，确实就复杂一点，要保存的内容也相对多一点。而下一章多任务的任务切换是全程发生在 S 特权级的执行流
   切换，所以会简单一点，保存的通用寄存器大概率更少（少在调用者保存寄存器），从各种意义上都很像函数调用。从不同特权级的角度来解释换栈
   是出于安全性，应用不应该看到 Trap 执行流的栈，这样做完之后，虽然理论上可以访问，但应用不知道内核栈的位置应该也有点麻烦。
   --
   然后是 rust_trap 的处理，尤其是奇妙的参数传递，内部处理逻辑倒是非常简单。
   --
   最后是如何利用 __restore 初始化应用的执行环境，包括如何设置入口点、用户栈以及保证在 U 特权级执行。





