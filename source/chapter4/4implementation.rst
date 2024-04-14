管理 LoongArch 多级页表
========================================================

本节导读
--------------------------

上一节我们介绍了 LoongArch 存储管理的硬件机制，本节我们主要讲解基于上述硬件机制的操作系统内存管理。这首先需要对地址和页表项进行数据结构抽象的类型定义，还需进一步管理计算机系统中当前已经使用的或空闲的物理页帧，这样操作系统才能给应用程序动态分配或回收物理地址空间。有了有效的物理内存空间的管理，操作系统就能够在物理内存空间中建立多级页表（页表占用物理内存），为应用程序和操作系统自身建立虚实地址映射关系，从而实现虚拟内存空间，即给应用“看到”的地址空间。

地址与页表项的数据结构抽象与类型定义
------------------------------------------------------------

地址相关的数据结构抽象与类型定义
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

正如本章第一小节所说，在分页内存管理中，地址转换的核心任务在于如何维护虚拟页号到物理页号的映射——也就是页表。不过在具体实现它之前，我们先将地址和页号的概念抽象为 Rust 中的类型，借助 Rust 的类型安全特性来确保它们被正确实现。

首先是这些类型的定义：

.. code-block:: rust

    // os/src/mm/address.rs

    #[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
    pub struct PhysAddr(pub usize);

    #[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
    pub struct VirtAddr(pub usize);

    #[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
    pub struct PhysPageNum(pub usize);

    #[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
    pub struct VirtPageNum(pub usize);

.. _term-type-conversion:

上面分别给出了物理地址、虚拟地址、物理页号、虚拟页号的 Rust 类型声明，它们都是 Rust 的元组式结构体，可以看成 usize 的一种简单包装。我们刻意将它们各自抽象出不同的类型而不是都使用与 LoongArch 64硬件直接对应的 usize 基本类型，就是为了在 Rust 编译器的帮助下，通过多种方便且安全的 **类型转换** (Type Conversion) 来构建页表。

首先，这些类型本身可以和 usize 之间互相转换，以物理地址 ``PhysAddr`` 为例，我们需要：

.. code-block:: rust

    // os/src/mm/address.rs

    pub const PALEN: usize = 48;
    pub const PPN_WIDTH: usize = PALEN - PAGE_SIZE_BITS;

    impl From<usize> for PhysAddr {
        fn from(v: usize) -> Self { Self(v & ((1 << PALEN) - 1)) }
    }
    impl From<usize> for PhysPageNum {
        fn from(v: usize) -> Self { Self(v & ((1 << PPN_WIDTH) - 1)) }
    }

    impl From<PhysAddr> for usize {
        fn from(v: PhysAddr) -> Self { v.0 }
    }
    impl From<PhysPageNum> for usize {
        fn from(v: PhysPageNum) -> Self { v.0 }
    }

前者允许我们从一个 ``usize`` 来生成 ``PhysAddr`` ，即 ``PhysAddr::from(_: usize)`` 将得到一个 ``PhysAddr`` 。注意 LoongArch 架构的不同实现支持的物理地址位宽不同，这里选择 qemu 模拟器模拟的龙芯 3A5000/3B5000 处理器所支持的物理地址位宽 48 位，由于最高位无需存放因此在生成 ``PhysAddr`` 的时候我们仅使用 ``usize`` 较低的 48 位。同理在生成虚拟地址 ``VirtAddr`` 的时候仅使用 ``usize`` 较低的 48 位。反过来，从 ``PhysAddr`` 等类型也很容易得到对应的 ``usize`` 。其实由于我们在声明结构体的时候将字段公开了出来，从物理地址变量 ``pa`` 得到它的 usize 表示的更简便方法是直接 ``pa.0`` 。

.. note::

    **Rust Tips：类型转换之 From 和 Into**

    一般而言，当我们为类型 ``U`` 实现了 ``From<T>`` Trait 之后，可以使用 ``U::from(_: T)`` 来从一个 ``T`` 类型的实例来构造一个 ``U`` 类型的实例；而当我们为类型 ``U`` 实现了 ``Into<T>`` Trait 之后，对于一个 ``U`` 类型的实例 ``u`` ，可以使用 ``u.into()`` 来将其转化为一个类型为 ``T`` 的实例。

    当我们为 ``U`` 实现了 ``From<T>`` 之后，Rust 会自动为 ``T`` 实现 ``Into<U>`` Trait，因为它们两个本来就是在做相同的事情。因此我们只需相互实现 ``From`` 就可以相互 ``From/Into`` 了。

    需要注意的是，当我们使用 ``From`` Trait 的 ``from`` 方法来构造一个转换后类型的实例的时候，``from`` 的参数已经指明了转换前的类型，因而 Rust 编译器知道该使用哪个实现；而使用 ``Into`` Trait 的 ``into`` 方法来将当前类型转化为另一种类型的时候，它并没有参数，因而函数签名中并没有指出要转化为哪一个类型，则我们必须在其它地方 *显式* 指出目标类型。比如，当我们要将 ``u.into()`` 绑定到一个新变量 ``t`` 的时候，必须通过 ``let t: T`` 显式声明 ``t`` 的类型；又或是将 ``u.into()`` 的结果作为参数传给某一个函数，那么由于这个函数的函数签名中指出了传入位置的参数的类型，所以 Rust 编译器也就明确知道转换的类型。

    请注意，解引用 ``Deref`` Trait 是 Rust 编译器唯一允许的一种隐式类型转换，而对于其他的类型转换，我们必须手动调用类型转化方法或者是显式给出转换前后的类型。这体现了 Rust 的类型安全特性，在 C/C++ 中并不是如此，比如两个不同的整数/浮点数类型进行二元运算的时候，编译器经常要先进行隐式类型转换使两个操作数类型相同，而后再进行运算，导致了很多数值溢出或精度损失问题。Rust 不会进行这种隐式类型转换，它会在编译期直接报错，提示两个操作数类型不匹配。

其次，地址和页号之间可以相互转换。我们这里仍以物理地址和物理页号之间的转换为例：

.. code-block:: rust
    :linenos:

    // os/src/mm/address.rs

    impl PhysAddr {
        pub fn page_offset(&self) -> usize { self.0 & (PAGE_SIZE - 1) }
    }

    impl From<PhysAddr> for PhysPageNum {
        fn from(v: PhysAddr) -> Self {
            assert_eq!(v.page_offset(), 0);
            v.floor()
        }
    }

    impl From<PhysPageNum> for PhysAddr {
        fn from(v: PhysPageNum) -> Self { Self(v.0 << PAGE_SIZE_BITS) }
    }

其中 ``PAGE_SIZE`` 为 :math:`0x4000` ， ``PAGE_SIZE_BITS`` 为 :math:`14` ，它们均定义在 ``config`` 子模块中，分别表示每个页面的大小和页内偏移的位宽。从物理页号到物理地址的转换只需左移 :math:`14` 位即可，但是物理地址需要保证它与页面大小对齐才能通过右移转换为物理页号。

对于不对齐的情况，物理地址不能通过 ``From/Into`` 转换为物理页号，而是需要通过它自己的 ``floor`` 或 ``ceil`` 方法来进行下取整或上取整的转换。

.. code-block:: rust

    // os/src/mm/address.rs

    impl PhysAddr {
        pub fn floor(&self) -> PhysPageNum { PhysPageNum(self.0 / PAGE_SIZE) }
        pub fn ceil(&self) -> PhysPageNum { PhysPageNum((self.0 + PAGE_SIZE - 1) / PAGE_SIZE) }
    }

我们暂时先介绍这两种最简单的类型转换。

页表项的数据结构抽象与类型定义
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

上一小节我们已经介绍了页表项的具体结构，现在我们使用 Rust 来定义页表项。首先来实现页表项中的标志位 ``PTEFlags`` ：

.. code-block:: rust

    // os/src/main.rs

    #[macro_use]
    extern crate bitflags;

    // os/src/mm/page_table.rs

    use bitflags::*;

    bitflags! {
        /// page table entry flags
        pub struct PTEFlags: u64 {
            const V = 1 << 0;
            const D = 1 << 1;
            const PLV_L = 1 << 2;
            const PLV_H = 1 << 3;
            const MAT_L = 1 << 4;
            const MAT_H = 1 << 5;
            const G = 1 << 6;
            const P = 1 << 7;
            const W = 1 << 8;
            const NR = 1 << 61;
            const NX = 1 << 62;
            const RPLV = 1 << 63;
        }
    }

`bitflags <https://docs.rs/bitflags/1.2.1/bitflags/>`_ 是一个 Rust 中常用来比特标志位的 crate 。它提供了一个 ``bitflags!`` 宏，如上面的代码段所展示的那样，可以将一个 ``u64`` 封装成一个标志位的集合类型，支持一些常见的集合运算。它的一些使用细节这里不展开，请同学自行参考它的官方文档。注意，在使用之前我们需要引入该 crate 的依赖：

.. code-block:: toml

    # os/Cargo.toml

    [dependencies]
    bitflags = "1.2.1"

接下来我们实现页表项 ``PageTableEntry`` ：

.. code-block:: rust
    :linenos:

    // os/src/mm/page_table.rs

    #[derive(Copy, Clone)]
    #[repr(C)]
    pub struct PageTableEntry {
        pub bits: usize,
    }

    impl PageTableEntry {
        pub fn new(ppn: PhysPageNum, flags: PTEFlags) -> Self {
            PageTableEntry {
                bits: ppn.0 << PAGE_SIZE_BITS | flags.bits as usize,
            }
        }
        pub fn empty() -> Self {
            PageTableEntry { bits: 0 }
        }
        pub fn ppn(&self) -> PhysPageNum {
            ((self.bits & ((1usize << PALEN) - 1)) >> PAGE_SIZE_BITS).into()
        }
        pub fn flags(&self) -> PTEFlags {
            unsafe {
                PTEFlags::from_bits_unchecked(self.bits as u64)
            }
        }
    }

- 第 3 行我们让编译器自动为 ``PageTableEntry`` 实现 ``Copy/Clone`` Trait，来让这个类型以值语义赋值/传参的时候不会发生所有权转移，而是拷贝一份新的副本。从这一点来说 ``PageTableEntry`` 就和 usize 一样，因为它也只是后者的一层简单包装，并解释了 usize 各个比特段的含义。
- 第 10 行使得我们可以从一个物理页号 ``PhysPageNum`` 和一个页表项标志位 ``PTEFlags`` 生成一个页表项 ``PageTableEntry`` 实例；而第 18 行和第 21 行则实现了分别可以从一个页表项将它们两个取出的方法。
- 第 15 行中，我们也可以通过 ``empty`` 方法生成一个全零的页表项，注意这隐含着该页表项的 ``V`` 标志位为 0 ，因此它是不合法的。

后面我们还为 ``PageTableEntry`` 实现了一些辅助函数(Helper Function)，可以快速判断一个页表项的 ``V/NR/W/NX`` 标志位是否为 1 。以 ``V`` 标志位的判断为例：

.. code-block:: rust

    // os/src/mm/page_table.rs

    impl PageTableEntry {
        pub fn is_valid(&self) -> bool {
            (self.flags() & PTEFlags::V) != PTEFlags::empty()
        }
    }

这里相当于判断两个集合的交集是否为空集，部分说明了 ``bitflags`` crate 的使用方法。

.. _term-manage-phys-frame:

物理页帧管理
-----------------------------------

从前面的介绍可以看出物理页帧的重要性：它既可以用来实际存放应用/内核的数据/代码，也能够用来存储应用/内核的多级页表。当Bootloader把操作系统内核加载到物理内存中后，物理内存上已经有一部分用于放置内核的代码和数据。我们需要将剩下的空闲内存以单个物理页帧为单位管理起来，当需要存放应用数据或扩展应用的多级页表时分配空闲的物理页帧，并在应用出错或退出的时候回收应用占有的所有物理页帧。

可用物理页的分配与回收
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

首先，我们需要知道物理内存的哪一部分是可用的。在 ``os/src/linker.ld`` 中，我们用符号 ``ekernel`` 指明了内核数据的终止物理地址，在它之后的物理内存都是可用的。而在 ``config`` 子模块中：

.. code-block:: rust

    // os/src/config.rs

    pub const MEMORY_END: usize = 0xfffffff;

我们硬编码整块物理内存的终止物理地址为 ``0xfffffff`` 。 而 :ref:`之前 <term-physical-memory>` 提到过在目前的实验环境下可用 RAM 被分为两个部分，即 ``0x0000000-0xfffffff`` 的 ``lowram`` 和 ``0x90000000`` 之后的 ``highram``，在这里我们只使用最低的 :math:`256\text{MiB}` 的 ``lowram`` ，感兴趣的同学可以将 ``highram`` 也纳入内存管理的范围。我们用一个左闭右开的物理页号区间来表示可用的物理内存，则：

- 区间的左端点应该是 ``ekernel`` 的物理地址以上取整方式转化成的物理页号；
- 区间的右端点应该是 ``MEMORY_END`` 以下取整方式转化成的物理页号。

这个区间将被传给我们后面实现的物理页帧管理器用于初始化。

我们声明一个 ``FrameAllocator`` Trait 来描述一个物理页帧管理器需要提供哪些功能：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    trait FrameAllocator {
        fn new() -> Self;
        fn alloc(&mut self) -> Option<PhysPageNum>;
        fn dealloc(&mut self, ppn: PhysPageNum);
    }

即创建一个物理页帧管理器的实例，以及以物理页号为单位进行物理页帧的分配和回收。

我们实现一种最简单的栈式物理页帧管理策略 ``StackFrameAllocator`` ：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    pub struct StackFrameAllocator {
        current: usize,  //空闲内存的起始物理页号
        end: usize,      //空闲内存的结束物理页号
        recycled: Vec<usize>,
    }

其中各字段的含义是：物理页号区间 [ ``current`` , ``end`` ) 此前均 *从未* 被分配出去过，而向量 ``recycled`` 以后入先出的方式保存了被回收的物理页号（注：我们已经自然的将内核堆用起来了）。

初始化非常简单。在通过 ``FrameAllocator`` 的 ``new`` 方法创建实例的时候，只需将区间两端均设为 :math:`0` ，然后创建一个新的向量；而在它真正被使用起来之前，需要调用 ``init`` 方法将自身的 :math:`[\text{current},\text{end})` 初始化为可用物理页号区间：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    impl FrameAllocator for StackFrameAllocator {
        fn new() -> Self {
            Self {
                current: 0,
                end: 0,
                recycled: Vec::new(),
            }
        }
    }

    impl StackFrameAllocator {
        pub fn init(&mut self, l: PhysPageNum, r: PhysPageNum) {
            self.current = l.0;
            self.end = r.0;
        }
    }

接下来我们来看核心的物理页帧分配和回收如何实现：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    impl FrameAllocator for StackFrameAllocator {
        fn alloc(&mut self) -> Option<PhysPageNum> {
            if let Some(ppn) = self.recycled.pop() {
                Some(ppn.into())
            } else {
                if self.current == self.end {
                    None
                } else {
                    self.current += 1;
                    Some((self.current - 1).into())
                }
            }
        }
        fn dealloc(&mut self, ppn: PhysPageNum) {
            let ppn = ppn.0;
            // validity check
            if ppn >= self.current || self.recycled
                .iter()
                .find(|&v| {*v == ppn})
                .is_some() {
                panic!("Frame ppn={:#x} has not been allocated!", ppn);
            }
            // recycle
            self.recycled.push(ppn);
        }
    }

- 在分配 ``alloc`` 的时候，首先会检查栈 ``recycled`` 内有没有之前回收的物理页号，如果有的话直接弹出栈顶并返回；否则的话我们只能从之前从未分配过的物理页号区间 [ ``current`` , ``end`` ) 上进行分配，我们分配它的左端点 ``current`` ，同时将管理器内部维护的 ``current`` 加 ``1`` 代表 ``current`` 已被分配了。在即将返回的时候，我们使用 ``into`` 方法将 usize 转换成了物理页号 ``PhysPageNum`` 。

  注意极端情况下可能出现内存耗尽分配失败的情况：即 ``recycled`` 为空且  ``current`` == ``end`` 。为了涵盖这种情况， ``alloc`` 的返回值被 ``Option`` 包裹，我们返回 ``None`` 即可。
- 在回收 ``dealloc`` 的时候，我们需要检查回收页面的合法性，然后将其压入 ``recycled`` 栈中。回收页面合法有两个条件：

  - 该页面之前一定被分配出去过，因此它的物理页号一定 :math:`<` ``current`` ；
  - 该页面没有正处在回收状态，即它的物理页号不能在栈 ``recycled`` 中找到。

  我们通过 ``recycled.iter()`` 获取栈上内容的迭代器，然后通过迭代器的 ``find`` 方法试图寻找一个与输入物理页号相同的元素。其返回值是一个 ``Option`` ，如果找到了就会是一个 ``Option::Some`` ，这种情况说明我们内核其他部分实现有误，直接报错退出。

下面我们来创建 ``StackFrameAllocator`` 的全局实例 ``FRAME_ALLOCATOR`` ：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    use crate::sync::UPSafeCell;
    type FrameAllocatorImpl = StackFrameAllocator;
    lazy_static! {
        pub static ref FRAME_ALLOCATOR: UPSafeCell<FrameAllocatorImpl> = unsafe {
            UPSafeCell::new(FrameAllocatorImpl::new())
        };
    }

这里我们使用 ``UPSafeCell<T>`` 来包裹栈式物理页帧分配器。每次对该分配器进行操作之前，我们都需要先通过 ``FRAME_ALLOCATOR.exclusive_access()`` 拿到分配器的可变借用。

.. chyyuu
    注意 ``alloc`` 中并没有提供 ``Mutex<T>`` ，它
    来自于一个我们在 ``no_std`` 的裸机环境下经常使用的名为 ``spin`` 的 crate ，它仅依赖 Rust 核心库 
    ``core`` 提供一些可跨平台使用的同步原语，如互斥锁 ``Mutex<T>`` 和读写锁 ``RwLock<T>`` 等。

.. 现在前面已经讲到了

    **Rust Tips：在单核环境下采用 UPSafeCell<T> 而没有采用 Mutex<T> 的原因**

    在编写一个多线程的Rust应用时，一般会通过 Mutex<T> 来包裹数据，并对数据访问进行加锁互斥保护，加锁的目的是为了避免数据竞争，使得里层的共享数据结构同一时间只有一个线程
    在对它进行访问。然而，目前我们的内核运行在单 CPU 上，且 Trap 进入内核之后并没有手动打开中断，这也就
    使得同一时间最多只有一条 Trap 控制流并发访问内核的各数据结构，此时应该是并没有任何数据竞争风险，所以我们基于更简单的 ``RefCell<T>`` 实现了 ``UPSafeCell<T>`` 来支持对全局变量的安全访问，支持在不触及 ``unsafe`` 的情况下实现 ``static mut`` 语义。

    注：这里引入了一些新概念，比如线程，互斥访问、数据竞争等。同学可以先不必深究，暂时有一个初步的概念即可，在后续章节会有进一步深入讲解。


.. chyyuu
    。所以那么
    加锁的原因其实有两点：

    1. 在不触及 ``unsafe`` 的情况下实现 ``static mut`` 语义。如果同学还有印象， 
       :ref:`前面章节 <term-interior-mutability>` 我们使用 ``RefCell<T>`` 提供了内部可变性去掉了
       声明中的 ``mut`` ，然而麻烦的在于 ``static`` ，在 Rust 中一个类型想被实例化为一个全局变量，则
       该类型必须先告知编译器自己某种意义上是线程安全的，这个过程本身是 ``unsafe`` 的。

       因此我们直接使用 ``Mutex<T>`` ，它既通过 ``lock`` 方法提供了内部可变性，又已经在模块内部告知了
       编译器它的线程安全性。这样 ``unsafe`` 就被隐藏在了 ``spin`` crate 之内，我们无需关心。这种风格
       是 Rust 所推荐的。
    2. 方便后续拓展到真正存在数据竞争风险的多核环境下运行。

    这里引入了一些新概念，比如什么是线程，又如何定义线程安全？同学可以先不必深究，暂时有一个初步的概念即可。

在正式分配物理页帧之前，我们需要将物理页帧全局管理器 ``FRAME_ALLOCATOR`` 初始化：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    pub fn init_frame_allocator() {
        extern "C" {
            fn ekernel();
        }
        FRAME_ALLOCATOR
            .exclusive_access()
            .init(PhysAddr::from(ekernel as usize).ceil(), PhysAddr::from(MEMORY_END).floor());
    }

这里我们调用物理地址 ``PhysAddr`` 的 ``floor/ceil`` 方法分别下/上取整获得可用的物理页号区间。


分配/回收物理页帧的接口
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

然后是公开给其他内核模块调用的分配/回收物理页帧的接口：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    pub fn frame_alloc() -> Option<FrameTracker> {
        FRAME_ALLOCATOR
            .exclusive_access()
            .alloc()
            .map(|ppn| FrameTracker::new(ppn))
    }

    fn frame_dealloc(ppn: PhysPageNum) {
        FRAME_ALLOCATOR
            .exclusive_access()
            .dealloc(ppn);
    }

可以发现， ``frame_alloc`` 的返回值类型并不是 ``FrameAllocator`` 要求的物理页号 ``PhysPageNum`` ，而是将其进一步包装为一个 ``FrameTracker`` 。这里借用了 RAII 的思想，将一个物理页帧的生命周期绑定到一个 ``FrameTracker`` 变量上，当一个 ``FrameTracker`` 被创建的时候，我们需要从 ``FRAME_ALLOCATOR`` 中分配一个物理页帧：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    pub struct FrameTracker {
        pub ppn: PhysPageNum,
    }

    impl FrameTracker {
        pub fn new(ppn: PhysPageNum) -> Self {
            // page cleaning
            let bytes_array = ppn.get_bytes_array();
            for i in bytes_array {
                *i = 0;
            }
            Self { ppn }
        }
    }

我们将分配来的物理页帧的物理页号作为参数传给 ``FrameTracker`` 的 ``new`` 方法来创建一个 ``FrameTracker`` 
实例。由于这个物理页帧之前可能被分配过并用做其他用途，我们在这里直接将这个物理页帧上的所有字节清零。这一过程并不
那么显然，我们后面再详细介绍。

当一个 ``FrameTracker`` 生命周期结束被编译器回收的时候，我们需要将它控制的物理页帧回收到 ``FRAME_ALLOCATOR`` 中：

.. code-block:: rust

    // os/src/mm/frame_allocator.rs

    impl Drop for FrameTracker {
        fn drop(&mut self) {
            frame_dealloc(self.ppn);
        }
    }

这里我们只需为 ``FrameTracker`` 实现 ``Drop`` Trait 即可。当一个 ``FrameTracker`` 实例被回收的时候，它的 ``drop`` 方法会自动被编译器调用，通过之前实现的 ``frame_dealloc`` 我们就将它控制的物理页帧回收以供后续使用了。

.. note::

    **Rust Tips：Drop Trait**

    Rust 中的 ``Drop`` Trait 是它的 RAII 内存管理风格可以被有效实践的关键。之前介绍的多种在堆上分配的 Rust 数据结构便都是通过实现 ``Drop`` Trait 来进行被绑定资源的自动回收的。例如：

    - ``Box<T>`` 的 ``drop`` 方法会回收它控制的分配在堆上的那个变量；
    - ``Rc<T>`` 的 ``drop`` 方法会减少分配在堆上的那个引用计数，一旦变为零则分配在堆上的那个被计数的变量自身也会被回收；
    - ``UPSafeCell<T>`` 的 ``exclusive_access`` 方法会获取内部数据结构的独占借用权并返回一个 ``RefMut<'a, T>`` （实际上来自 ``RefCell<T>`` ），它可以被当做一个 ``&mut T`` 来使用；而 ``RefMut<'a, T>`` 的 ``drop`` 方法会将独占借用权交出，从而允许内核内的其他控制流后续对数据结构进行访问。

    ``FrameTracker`` 的设计也是基于同样的思想，有了它之后我们就不必手动回收物理页帧了，这在编译期就解决了很多潜在的问题。

最后做一个小结：从其他内核模块的视角看来，物理页帧分配的接口是调用 ``frame_alloc`` 函数得到一个 ``FrameTracker`` （如果物理内存还有剩余），它就代表了一个物理页帧，当它的生命周期结束之后它所控制的物理页帧将被自动回收。下面是一段演示该接口使用方法的测试程序：

.. code-block:: rust
    :linenos:
    :emphasize-lines: 9

    // os/src/mm/frame_allocator.rs

    #[allow(unused)]
    pub fn frame_allocator_test() {
        let mut v: Vec<FrameTracker> = Vec::new();
        for i in 0..5 {
            let frame = frame_alloc().unwrap();
            println!("{:?}", frame);
            v.push(frame);
        }
        v.clear();
        for i in 0..5 {
            let frame = frame_alloc().unwrap();
            println!("{:?}", frame);
            v.push(frame);
        }
        drop(v);
        println!("frame_allocator_test passed!");
    }

如果我们将第 9 行删去，则第一轮分配的 5 个物理页帧都是分配之后在循环末尾就被立即回收，因为循环作用域的临时变量 ``frame`` 的生命周期在那时结束了。然而，如果我们将它们 move 到一个向量中，它们的生命周期便被延长了——直到第 11 行向量被清空的时候，这些 ``FrameTracker`` 的生命周期才结束，它们控制的 5 个物理页帧才被回收。这种思想我们立即就会用到。

多级页表管理
-----------------------------------


页表基本数据结构与访问接口
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们知道，LoongArch 多级页表是以节点为单位进行管理的。每个节点恰好存储在一个物理页帧中，它的位置可以用一个物理页号来表示。

.. code-block:: rust
    :linenos:

    // os/src/mm/page_table.rs

    pub struct PageTable {
        root_ppn: PhysPageNum,
        frames: Vec<FrameTracker>,
    }

    impl PageTable {
        pub fn new() -> Self {
            let frame = frame_alloc().unwrap();
            PageTable {
                root_ppn: frame.ppn,
                frames: vec![frame],
            }
        }
    }

每个应用的地址空间都对应一个不同的多级页表，这也就意味这不同页表的起始地址（即页表根节点的地址）是不一样的。因此 ``PageTable`` 要保存它根节点的物理页号 ``root_ppn`` 作为页表唯一的区分标志。此外，向量 ``frames`` 以 ``FrameTracker`` 的形式保存了页表所有的节点（包括根节点）所在的物理页帧。这与物理页帧管理模块的测试程序是一个思路，即将这些 ``FrameTracker`` 的生命周期进一步绑定到 ``PageTable`` 下面。当 ``PageTable`` 生命周期结束后，向量 ``frames`` 里面的那些 ``FrameTracker`` 也会被回收，也就意味着存放多级页表节点的那些物理页帧被回收了。

当我们通过 ``new`` 方法新建一个 ``PageTable`` 的时候，它只需有一个根节点。为此我们需要分配一个物理页帧 ``FrameTracker`` 并挂在向量 ``frames`` 下，然后更新根节点的物理页号 ``root_ppn`` 。

多级页表并不是被创建出来之后就不再变化的，为了 MMU 能够通过地址转换正确找到应用地址空间中的数据实际被内核放在内存中位置，操作系统需要动态维护一个虚拟页号到页表项的映射，支持插入/删除键值对，其方法签名如下：

.. code-block:: rust

    // os/src/mm/page_table.rs

    impl PageTable {
        pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags);
        pub fn unmap(&mut self, vpn: VirtPageNum);
    }

- 通过 ``map`` 方法来在多级页表中插入一个键值对，注意这里将物理页号 ``ppn`` 和页表项标志位 ``flags`` 作为不同的参数传入；
- 通过 ``unmap`` 方法来删除一个键值对，在调用时仅需给出作为索引的虚拟页号即可。

.. _modify-page-table:

在上述操作的过程中，内核需要能访问或修改多级页表节点的内容。即在操作某个多级页表或管理物理页帧的时候，操作系统要能够读写与一个给定的物理页号对应的物理页帧上的数据。这是因为，在多级页表的架构中，每个节点都被保存在一个物理页帧中，一个节点所在物理页帧的物理页号其实就是指向该节点的“指针”。

在尚未启用分页模式之前，内核和应用的代码都可以通过物理地址直接访问内存。而在打开分页模式之后，运行在 PLV0 特权级的内核与运行在 PLV3 特权级的应用在访存上都会受到影响，它们的访存地址会被视为一个当前地址空间（ ``pgdl`` 和 ``pgdh`` CSR 给出当前多级页表根节点的物理地址）中的一个虚拟地址，需要 MMU 查相应的多级页表完成地址转换变为物理地址，即地址空间中虚拟地址指向的数据真正被内核放在的物理内存中的位置，然后才能访问相应的数据。此时，如果想要访问一个特定的物理地址 ``pa`` 所指向的内存上的数据，就需要 **构造** 对应的一个虚拟地址 ``va`` ，使得当前地址空间的页表存在映射 :math:`\text{va}\rightarrow\text{pa}` ，且页表项中的保护位允许这种访问方式。于是，在代码中我们只需访问地址 ``va`` ，它便会被 MMU 通过地址转换变成 ``pa`` ，这样我们就做到了在启用分页模式的情况下也能正常访问内存。

.. _term-identical-mapping:

这就需要提前扩充多级页表维护的映射，让每个物理页帧的物理页号 ``ppn`` ，均存在一个对应的虚拟页号 ``vpn`` ，这需要建立一种映射关系。这里我们采用一种最简单的 **恒等映射** (Identical Mapping) ，即对于物理内存上的每个物理页帧，我们都在多级页表中用一个与其物理页号相等的虚拟页号来映射。

.. _term-recursive-mapping:

.. note::

    **其他的映射方式**

    为了达到这一目的还存在其他不同的映射方式，例如比较著名的 **页表自映射** (Recursive Mapping) 等。有兴趣的同学
    可以进一步参考 `BlogOS 中的相关介绍 <https://os.phil-opp.com/paging-implementation/#accessing-page-tables>`_ 。

这里需要说明的是，在下一节中我们可以看到，应用和内核的地址空间是隔离的。而直接访问物理页帧的操作只会在内核中进行，应用无法看到物理页帧管理器和多级页表等内核数据结构。因此，上述的恒等映射只需被附加到内核地址空间即可。


内核中访问物理页帧的方法
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _access-frame-in-kernel-as:


于是，我们来看看在内核中应如何访问一个特定的物理页帧：

.. code-block:: rust

    // os/src/mm/address.rs

    impl PhysPageNum {
        pub fn get_pte_array(&self) -> &'static mut [PageTableEntry] {
            let pa: PhysAddr = (*self).into();
            unsafe { core::slice::from_raw_parts_mut(pa.0 as *mut PageTableEntry, TABLE_ENTRY_NUM) }
        }
        pub fn get_bytes_array(&self) -> &'static mut [u8] {
            let pa: PhysAddr = (*self).into();
            unsafe { core::slice::from_raw_parts_mut(pa.0 as *mut u8, PAGE_SIZE) }
        }
        pub fn get_mut<T>(&self) -> &'static mut T {
            let pa: PhysAddr = (*self).into();
            unsafe { (pa.0 as *mut T).as_mut().unwrap() }
        }
    }

我们构造可变引用来直接访问一个物理页号 ``PhysPageNum`` 对应的物理页帧，不同的引用类型对应于物理页帧上的一种不同的内存布局，如 ``get_pte_array`` 返回的是一个页表项定长数组的可变引用，代表多级页表中的一个节点；而 ``get_bytes_array`` 返回的是一个字节数组的可变引用，可以以字节为粒度对物理页帧上的数据进行访问，前面进行数据清零就用到了这个方法； ``get_mut`` 是个泛型函数，可以获取一个恰好放在一个物理页帧开头的类型为 ``T`` 的数据的可变引用。

在实现方面，都是先把物理页号转为物理地址 ``PhysAddr`` ，然后再转成 usize 形式的物理地址。接着，我们直接将它转为裸指针用来访问物理地址指向的物理内存。在分页机制开启前，这样做自然成立；而开启之后，虽然裸指针被视为一个虚拟地址，但是上面已经提到，基于恒等映射，虚拟地址会映射到一个相同的物理地址，因此在也是成立的。注意，我们在返回值类型上附加了静态生命周期泛型 ``'static`` ，这是为了绕过 Rust 编译器的借用检查，实质上可以将返回的类型也看成一个裸指针，因为它也只是标识数据存放的位置以及类型。但与裸指针不同的是，无需通过 ``unsafe`` 的解引用访问它指向的数据，而是可以像一个正常的可变引用一样直接访问。

.. note::
    
    **unsafe 真的就是“不安全”吗？**

    下面是笔者关于 unsafe 一点较为深入的讨论，不感兴趣的同学可以跳过。

    当我们在 Rust 中使用 unsafe 的时候，并不仅仅是为了绕过编译器检查，更是为了告知编译器和其他看到这段代码的程序员：“ **我保证这样做是安全的** ” 。尽管，严格的 Rust 编译器暂时还不能确信这一点。从规范 Rust 代码编写的角度，我们需要尽可能绕过 unsafe ，因为如果 Rust 编译器或者一些已有的接口就可以提供安全性，我们当然倾向于利用它们让我们实现的功能仍然是安全的，可以避免一些无谓的心智负担；反之，就只能使用 unsafe ，同时最好说明如何保证这项功能是安全的。

    这里简要从内存安全的角度来分析一下 ``PhysPageNum`` 的 ``get_*`` 系列方法的实现中 ``unsafe`` 的使用。首先需要指出的是，当需要访问一个物理页帧的时候，我们需要从它被绑定到的 ``FrameTracker`` 中获得其物理页号 ``PhysPageNum`` 随后再调用 ``get_*`` 系列方法才能访问物理页帧。因此， ``PhysPageNum`` 介于 ``FrameTracker`` 和物理页帧之间，也可以看做拥有部分物理页帧的所有权。由于 ``get_*`` 返回的是引用，我们可以尝试检查引用引发的常见问题：第一个问题是 use-after-free 的问题，即是否存在 ``get_*`` 返回的引用存在期间被引用的物理页帧已被回收的情形；第二个问题则是注意到 ``get_*`` 返回的是可变引用，那么就需要考虑对物理页帧的访问读写冲突的问题。

    为了解决这些问题，我们在编写代码的时候需要额外当心。对于每一段 unsafe 代码，我们都需要认真考虑它会对其他无论是 unsafe 还是 safe 的代码造成的潜在影响。比如为了避免第一个问题，我们需要保证当完成物理页帧访问之后便立即回收掉 ``get_*`` 返回的引用，至少使它不能超出 ``FrameTracker`` 的生命周期；考虑第二个问题，目前每个 ``FrameTracker`` 仅会出现一次（在它所属的进程中），因此它只会出现在一个上下文中，也就不会产生冲突。但是当内核态打开（允许）中断时，或内核支持在单进程中存在多个线程时，情况也许又会发生变化。

    当编译器不能介入的时候，我们很难完美的解决这些问题。因此重新设计数据结构和接口，特别是考虑数据的所有权关系，将建模进行转换，使得 Rust 有能力检查我们的设计会是一种更明智的选择。这也可以说明为什么要尽量避免使用 unsafe 。事实上，我们目前 ``PhysPageNum::get_*`` 接口并非一个好的设计，如果同学有兴趣可以试着对设计进行改良，让 Rust 编译器帮助我们解决上述与引用相关的问题。
    
.. _term-create-pagetable:

建立和拆除虚实地址映射关系
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

接下来介绍建立和拆除虚实地址映射关系的 ``map`` 和 ``unmap`` 方法是如何实现的。它们都依赖于一个很重要的过程，即在多级页表中找到一个虚拟地址对应的页表项。找到之后，只要修改页表项的内容即可完成键值对的插入和删除。在寻找页表项的时候，可能出现页表的中间级节点还未被创建的情况，这个时候我们需要手动分配一个物理页帧来存放这个节点，并将这个节点接入到当前的多级页表的某级中。


.. code-block:: rust
    :linenos:

    // os/src/mm/address.rs

    impl VirtPageNum {
        pub fn indexes(&self) -> [usize; 3] {
            let mut vpn = self.0;
            let mut idx = [0usize; 3];
            for i in (0..3).rev() {
                idx[i] = vpn & (TABLE_ENTRY_NUM - 1);
                vpn >>= TABLE_ENTRY_NUM_BITS;
            }
            idx
        }
    }

    // os/src/mm/page_table.rs

    impl PageTable {
        fn find_pte_create(&mut self, vpn: VirtPageNum) -> Option<&mut PageTableEntry> {
            let idxs = vpn.indexes();
            let mut ppn = self.root_ppn;
            let mut result: Option<&mut PageTableEntry> = None;
            for i in 0..3 {
                let pte = &mut ppn.get_pte_array()[idxs[i]];
                if i == 2 {
                    result = Some(pte);
                    break;
                }
                if !pte.is_valid() {
                    let frame = frame_alloc().unwrap();
                    *pte = PageTableEntry::new(frame.ppn, PTEFlags::V);
                    self.frames.push(frame);
                }
                ppn = pte.ppn();
            }
            result
        }
        fn find_pte(&self, vpn: VirtPageNum) -> Option<&mut PageTableEntry> {
            let idxs = vpn.indexes();
            let mut ppn = self.root_ppn;
            let mut result: Option<&mut PageTableEntry> = None;
            for i in 0..3 {
                let pte = &mut ppn.get_pte_array()[idxs[i]];
                if i == 2 {
                    result = Some(pte);
                    break;
                }
                if !pte.is_valid() {
                    return None;
                }
                ppn = pte.ppn();
            }
            result
        }
    }

- ``VirtPageNum`` 的 ``indexes`` 可以取出虚拟页号的三级页索引，并按照从高到低的顺序返回。注意这里我们是用来在多级页表上进行遍历，因此只取出其包裹的 ``usize`` 的低 :math:`33` 位。
- ``PageTable::find_pte_create`` 在多级页表找到一个虚拟页号对应的页表项的可变引用。如果在遍历的过程中发现有节点尚未创建则会新建一个节点。

  变量 ``ppn`` 表示当前节点的物理页号，最开始指向多级页表的根节点。随后每次循环通过 ``get_pte_array`` 将取出当前节点的页表项数组，并根据当前级页索引找到对应的页表项。如果当前节点是一个叶节点，那么直接返回这个页表项的可变引用；否则尝试向下走。走不下去的话就新建一个节点，更新作为下级节点指针的页表项，并将新分配的物理页帧移动到向量 ``frames`` 中方便后续的自动回收。注意在更新页表项的时候，不仅要更新物理页号，还要将标志位 V 置 1，不然硬件在查多级页表的时候，会认为这个页表项不合法，从而触发 Page Fault 而不能向下走。
- ``PageTable::find_pte`` 与 ``find_pte_create`` 的不同在于当找不到合法叶子节点的时候不会新建叶子节点而是直接返回 ``None`` 即查找失败。因此，它不会尝试对页表本身进行修改，但是注意它返回的参数类型是页表项的可变引用，也即它允许我们修改页表项。从 ``find_pte`` 的实现还可以看出，即使找到的页表项不合法，还是会将其返回回去而不是返回 ``None`` 。这说明在目前的实现中，页表和页表项是相对解耦合的。

于是， ``map/unmap`` 就非常容易实现了：

.. code-block:: rust

    // os/src/mm/page_table.rs

    impl PageTable {
        pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags) {
            let pte = self.find_pte_create(vpn).unwrap();
            assert!(!pte.is_valid(), "vpn {:?} is mapped before mapping", vpn);
            *pte = PageTableEntry::new(ppn, flags | PTEFlags::V);
        }
        pub fn unmap(&mut self, vpn: VirtPageNum) {
            let pte = self.find_pte(vpn).unwrap();
            assert!(pte.is_valid(), "vpn {:?} is invalid before unmapping", vpn);
            *pte = PageTableEntry::empty();
        }
    }

只需根据虚拟页号找到页表项，然后修改或者直接清空其内容即可。

.. warning::

    目前的实现方式并不打算对物理页帧耗尽的情形做任何处理而是直接 ``panic`` 退出。因此在前面的代码中能够看到很多 ``unwrap`` ，这种使用方式并不为 Rust 所推荐，只是由于简单起见暂且这样做。

为了方便后面的实现，我们还需要 ``PageTable`` 提供一种类似 MMU 操作的手动查页表的方法：

.. code-block:: rust
    :linenos:

    // os/src/mm/page_table.rs

    impl PageTable {
        /// Temporarily used to get arguments from user space.
        pub fn from_token(pgdl: usize) -> Self {
            Self {
                root_ppn: PhysPageNum::from(pgdl >> PAGE_SIZE_BITS),
                frames: Vec::new(),
            }
        }
        pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry> {
            self.find_pte(vpn).map(|pte| *pte)
        }
    }

- 第 5 行的 ``from_token`` 可以临时创建一个专用来手动查页表的 ``PageTable`` ，它仅有一个从传入的 ``satp`` token 中得到的多级页表根节点的物理页号，它的 ``frames`` 字段为空，也即不实际控制任何资源；
- 第 11 行的 ``translate`` 调用 ``find_pte`` 来实现，如果能够找到页表项，那么它会将页表项拷贝一份并返回，否则就返回一个 ``None`` 。

之后，当遇到需要查一个特定页表（非当前正处在的地址空间的页表时），便可先通过 ``PageTable::from_token`` 新建一个页表，再调用它的 ``translate`` 方法查页表。

小结一下，上一节和本节讲解了如何基于 LoongArch 64 的分页机制建立多级页表，并实现基于虚存地址空间的内存使用环境。这样，一旦启用分页机制，操作系统和应用都只能在虚拟地址空间中访问数据了，只是操作系统可以通过页表机制来限制应用访问的实际物理内存范围。这就要在后续小节中，进一步看看操作系统内核和应用程序是如何在虚拟地址空间中进行代码和数据访问的。