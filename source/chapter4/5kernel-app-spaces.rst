内核与应用的地址空间
================================================


本节导读
--------------------------




页表 ``PageTable`` 只能以页为单位帮助我们维护一个虚拟内存到物理内存的地址转换关系，它本身对于计算机系统的整个虚拟/物理内存空间并没有一个全局的描述和掌控。操作系统通过对不同页表的管理，来完成对不同应用和操作系统自身所在的虚拟内存，以及虚拟内存与物理内存映射关系的全面管理。这种管理是建立在 **地址空间** 的抽象上，用来表明正在运行的应用或内核自身所在执行环境中的可访问的内存空间。本节
我们就在内核中通过基于页表的各种数据结构实现地址空间的抽象，并介绍内核和应用的虚拟和物理地址空间中各需要包含哪些内容。

实现地址空间抽象
------------------------------------------

.. _term-vm-map-area:

逻辑段：一段连续地址的虚拟内存
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们以逻辑段 ``MapArea`` 为单位描述一段连续地址的虚拟内存。所谓逻辑段，就是指地址区间中的一段实际可用（即 MMU 通过查多级页表可以正确完成地址转换）的地址连续的虚拟地址区间，该区间内包含的所有虚拟页面都以一种相同的方式映射到物理页帧，具有可读/可写/可执行等属性。

.. code-block:: rust

    // os/src/mm/memory_set.rs

    pub struct MapArea {
        vpn_range: VPNRange,
        data_frames: BTreeMap<VirtPageNum, FrameTracker>,
        map_type: MapType,
        map_perm: MapPermission,
    }

其中 ``VPNRange`` 描述一段虚拟页号的连续区间，表示该逻辑段在地址区间中的位置和长度。它是一个迭代器，可以使用 Rust 的语法糖 for-loop 进行迭代。有兴趣的同学可以参考 ``os/src/mm/address.rs`` 中它的实现。

.. note::

    **Rust Tips：迭代器 Iterator**

    Rust编程的迭代器模式允许你对一个序列的项进行某些处理。迭代器（iterator）是负责遍历序列中的每一项和决定序列何时结束的控制逻辑。对于如何使用迭代器处理元素序列和如何实现 Iterator trait 来创建自定义迭代器的内容，可以参考 `Rust 程序设计语言-中文版第十三章第二节 <https://kaisery.github.io/trpl-zh-cn/ch13-02-iterators.html>`_

``MapType`` 描述该逻辑段内的所有虚拟页面映射到物理页帧的同一种方式，它是一个枚举类型，在内核当前的实现中支持两种方式：

.. code-block:: rust

    // os/src/mm/memory_set.rs

    #[derive(Copy, Clone, PartialEq, Debug)]
    pub enum MapType {
        Identical,
        Framed,
    }

其中 ``Identical`` 表示上一节提到的恒等映射方式；而 ``Framed`` 则表示对于每个虚拟页面都有一个新分配的物理页帧与之对应，虚地址与物理地址的映射关系是相对随机的。恒等映射方式主要是用在启用多级页表之后，内核仍能够在虚存地址空间中访问一个特定的物理地址指向的物理内存。

当逻辑段采用 ``MapType::Framed`` 方式映射到物理内存的时候， ``data_frames`` 是一个保存了该逻辑段内的每个虚拟页面和它被映射到的物理页帧 ``FrameTracker`` 的一个键值对容器 ``BTreeMap`` 中，这些物理页帧被用来存放实际内存数据而不是作为多级页表中的中间节点。和之前的 ``PageTable`` 一样，这也用到了 RAII 的思想，将这些物理页帧的生命周期绑定到它所在的逻辑段 ``MapArea`` 下，当逻辑段被回收之后这些之前分配的物理页帧也会自动地同时被回收。

``MapPermission`` 表示控制该逻辑段的访问方式，它是页表项标志位 ``PTEFlags`` 的一个子集，仅保留 D/PLV_L/PLV_H/W/NR/NX 六个标志位，因为其他的标志位仅与硬件的地址转换机制细节相关，这样的设计能避免引入错误的标志位。

.. code-block:: rust

    // os/src/mm/memory_set.rs

    bitflags! {
        pub struct MapPermission: u64 {
            const D = 1 << 1;
            const PLV_L = 1 << 2;
            const PLV_H = 1 << 3;
            const W = 1 << 8;
            const NR = 1 << 61;
            const NX = 1 << 62;
        }
    }


.. _term-vm-memory-set:

地址空间：一系列有关联的逻辑段
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**地址空间** 是一系列有关联的不一定连续的逻辑段，这种关联一般是指这些逻辑段组成的虚拟内存空间与一个运行的程序（目前把一个运行的程序称为任务，后续会称为进程）绑定，即这个运行的程序对代码和数据的直接访问范围限制在它关联的虚拟地址空间之内。这样我们就有任务的地址空间，内核的地址空间等说法了。地址空间使用 ``MemorySet`` 类型来表示：

.. code-block:: rust

    // os/src/mm/memory_set.rs

    pub struct MemorySet {
        page_table: PageTable,
        areas: Vec<MapArea>,
    }

它包含了该地址空间的多级页表 ``page_table`` 和一个逻辑段 ``MapArea`` 的向量 ``areas`` 。注意 ``PageTable`` 下挂着所有多级页表的节点所在的物理页帧，而每个 ``MapArea`` 下则挂着对应逻辑段中的数据所在的物理页帧，这两部分合在一起构成了一个地址空间所需的所有物理页帧。这同样是一种 RAII 风格，当一个地址空间 ``MemorySet`` 生命周期结束后，这些物理页帧都会被回收。

地址空间 ``MemorySet`` 的方法如下：

.. code-block:: rust
    :linenos:

    // os/src/mm/memory_set.rs

    impl MemorySet {
        pub fn new_bare() -> Self {
            Self {
                page_table: PageTable::new(),
                areas: Vec::new(),
            }
        }
        fn push(&mut self, mut map_area: MapArea, data: Option<&[u8]>) {
            map_area.map(&mut self.page_table);
            if let Some(data) = data {
                map_area.copy_data(&mut self.page_table, data);
            }
            self.areas.push(map_area);
        }
        /// Assume that no conflicts.
        pub fn insert_framed_area(
            &mut self,
            start_va: VirtAddr, end_va: VirtAddr, permission: MapPermission
        ) {
            self.push(MapArea::new(
                start_va,
                end_va,
                MapType::Framed,
                permission,
            ), None);
        }
        pub fn new_kernel() -> Self;
        /// Include sections in elf and trampoline and TrapContext and user stack,
        /// also returns user_sp and entry point.
        pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize);
        pub fn new_trampoline() -> Self;
    }

- 第 4 行， ``new_bare`` 方法可以新建一个空的地址空间；
- 第 10 行， ``push`` 方法可以在当前地址空间插入一个新的逻辑段 ``map_area`` ，如果它是以 ``Framed`` 方式映射到物理内存，还可以可选地在那些被映射到的物理页帧上写入一些初始化数据 ``data`` ；
- 第 18 行， ``insert_framed_area`` 方法调用 ``push`` ，可以在当前地址空间插入一个 ``Framed`` 方式映射到物理内存的逻辑段。注意该方法的调用者要保证同一地址空间内的任意两个逻辑段不能存在交集，从后面即将分别介绍的内核和应用的地址空间布局可以看出这一要求得到了保证；
- 第 29 行， ``new_kernel`` 可以生成内核的地址空间；具体实现将在后面讨论；
- 第 32 行， ``from_elf`` 分析应用的 ELF 文件格式的内容，解析出各数据段并生成对应的地址空间；具体实现将在后面讨论。
- 第 33 行， ``new_trampoline`` 可以生成跳板空间；具体实现将在后面讨论。

在实现 ``push`` 方法在地址空间中插入一个逻辑段 ``MapArea`` 的时候，需要同时维护地址空间的多级页表 ``page_table`` 记录的虚拟页号到页表项的映射关系，也需要用到这个映射关系来找到向哪些物理页帧上拷贝初始数据。这用到了 ``MapArea`` 提供的另外几个方法：

.. code-block:: rust
    :linenos:
    
    // os/src/mm/memory_set.rs

    impl MapArea {
        pub fn new( 
            start_va: VirtAddr,
            end_va: VirtAddr,
            map_type: MapType,
            map_perm: MapPermission
        ) -> Self {
            let start_vpn: VirtPageNum = start_va.floor();
            let end_vpn: VirtPageNum = end_va.ceil();
            Self {
                vpn_range: VPNRange::new(start_vpn, end_vpn),
                data_frames: BTreeMap::new(),
                map_type,
                map_perm,
            }
        }
        pub fn map(&mut self, page_table: &mut PageTable) {
            for vpn in self.vpn_range {
                self.map_one(page_table, vpn);
            }
        }
        pub fn unmap(&mut self, page_table: &mut PageTable) {
            for vpn in self.vpn_range {
                self.unmap_one(page_table, vpn);
            }
        }
        /// data: start-aligned but maybe with shorter length
        /// assume that all frames were cleared before
        pub fn copy_data(&mut self, page_table: &mut PageTable, data: &[u8]) {
            assert_eq!(self.map_type, MapType::Framed);
            let mut start: usize = 0;
            let mut current_vpn = self.vpn_range.get_start();
            let len = data.len();
            loop {
                let src = &data[start..len.min(start + PAGE_SIZE)];
                let dst = &mut page_table
                    .translate(current_vpn)
                    .unwrap()
                    .ppn()
                    .get_bytes_array()[..src.len()];
                dst.copy_from_slice(src);
                start += PAGE_SIZE;
                if start >= len {
                    break;
                }
                current_vpn.step();
            }
        }
    }

- 第 4 行的 ``new`` 方法可以新建一个逻辑段结构体，注意传入的起始/终止虚拟地址会分别被下取整/上取整为虚拟页号并传入迭代器 ``vpn_range`` 中；
- 第 19 行的 ``map`` 和第 24 行的 ``unmap`` 可以将当前逻辑段到物理内存的映射从传入的该逻辑段所属的地址空间的多级页表中加入或删除。可以看到它们的实现是遍历逻辑段中的所有虚拟页面，并以每个虚拟页面为单位依次在多级页表中进行键值对的插入或删除，分别对应 ``MapArea`` 的 ``map_one`` 和 ``unmap_one`` 方法，我们后面将介绍它们的实现；
- 第 31 行的 ``copy_data`` 方法将切片 ``data`` 中的数据拷贝到当前逻辑段实际被内核放置在的各物理页帧上，从而在地址空间中通过该逻辑段就能访问这些数据。调用它的时候需要满足：切片 ``data`` 中的数据大小不超过当前逻辑段的总大小，且切片中的数据会被对齐到逻辑段的开头，然后逐页拷贝到实际的物理页帧。

  从第 36 行开始的循环会遍历每一个需要拷贝数据的虚拟页面，在数据拷贝完成后会在第 48 行通过调用 ``step`` 方法，该方法来自于 ``os/src/mm/address.rs`` 中为 ``VirtPageNum`` 实现的 ``StepOne`` Trait，感兴趣的同学可以阅读代码确认其实现。

  每个页面的数据拷贝需要确定源 ``src`` 和目标 ``dst`` 两个切片并直接使用 ``copy_from_slice`` 完成复制。当确定目标切片 ``dst`` 的时候，第 39 行从传入的当前逻辑段所属的地址空间的多级页表中，手动查找迭代到的虚拟页号被映射到的物理页帧，并通过 ``get_bytes_array`` 方法获取该物理页帧的字节数组型可变引用，最后再获取它的切片用于数据拷贝。

接下来介绍对逻辑段中的单个虚拟页面进行映射/解映射的方法 ``map_one`` 和 ``unmap_one`` 。显然它们的实现取决于当前逻辑段被映射到物理内存的方式：

.. code-block:: rust
    :linenos:

    // os/src/mm/memory_set.rs

    impl MapArea {
        pub fn map_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
            let ppn: PhysPageNum;
            match self.map_type {
                MapType::Identical => {
                    ppn = PhysPageNum(vpn.0);
                }
                MapType::Framed => {
                    let frame = frame_alloc().unwrap();
                    ppn = frame.ppn;
                    self.data_frames.insert(vpn, frame);
                }
            }
            let pte_flags = PTEFlags::from_bits(self.map_perm.bits).unwrap();
            page_table.map(vpn, ppn, pte_flags);
        }
        pub fn unmap_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
            match self.map_type {
                MapType::Framed => {
                    self.data_frames.remove(&vpn);
                }
                _ => {}
            }
            page_table.unmap(vpn);
        }
    }

- 对于第 4 行的 ``map_one`` 来说，在虚拟页号 ``vpn`` 已经确定的情况下，它需要知道要将一个怎么样的页表项插入多级页表。页表项的标志位来源于当前逻辑段的类型为 ``MapPermission`` 的统一配置，只需将其转换为 ``PTEFlags`` ；而页表项的物理页号则取决于当前逻辑段映射到物理内存的方式：

  - 当以恒等映射 ``Identical`` 方式映射的时候，物理页号就等于虚拟页号；
  - 当以 ``Framed`` 方式映射时，需要分配一个物理页帧让当前的虚拟页面可以映射过去，此时页表项中的物理页号自然就是
    这个被分配的物理页帧的物理页号。此时还需要将这个物理页帧挂在逻辑段的 ``data_frames`` 字段下。

  当确定了页表项的标志位和物理页号之后，即可调用多级页表 ``PageTable`` 的 ``map`` 接口来插入键值对。
- 对于第 19 行的 ``unmap_one`` 来说，基本上就是调用 ``PageTable`` 的 ``unmap`` 接口删除以传入的虚拟页号为键的键值对即可。然而，当以 ``Framed`` 映射的时候，不要忘记同时将虚拟页面被映射到的物理页帧 ``FrameTracker`` 从 ``data_frames`` 中移除，这样这个物理页帧才能立即被回收以备后续分配。

内核地址空间
------------------------------------------

.. _term-isolation:

在本章之前，内核和应用代码的访存地址都被视为一个物理地址，并直接访问物理内存，而在分页模式开启之后，CPU先拿到虚存地址，需要通过 MMU 的地址转换变成物理地址，再交给 CPU 的访存单元去访问物理内存。地址空间抽象的重要意义在于 **隔离** (Isolation) ，当内核让应用执行前，内核需要控制 MMU 使用这个应用的多级页表进行地址转换。由于每个应用地址空间在创建的时候也顺带设置好了多级页表，使得只有那些存放了它的代码和数据的物理页帧能够通过该多级页表被映射到，这样它就只能访问自己的代码和数据而无法触及其他应用或内核的内容。

.. _term-trampoline-first:

启用分页模式下，内核代码的访存地址也会被视为一个虚拟地址并需要经过 MMU 的地址转换，因此我们也需要为内核对应构造一个地址空间，从而允许内核的各数据段能够被正常访问。当然，还需要包含所有应用的内核栈以及一个 **跳板** (Trampoline) ，我们会在本章的后续部分再深入介绍 :ref:`跳板空间的实现 <term-trampoline>` 。这里我们先实现内核地址空间的低半合法部分，将其实现为一个 ``MemorySet`` 。

下面则给出了内核地址空间的低半合法部分的布局：

.. image:: kernel-as-low.png
    :align: center
    :height: 400

内核的四个逻辑段 ``.text/.rodata/.data/.bss`` 被恒等映射到物理内存，这使得我们在无需调整内核内存布局 ``os/src/linker.ld`` 的情况下就仍能象启用页表机制之前那样访问内核的各个段。注意我们借用页表机制对这些逻辑段的访问方式做出了限制，这都是为了在硬件的帮助下能够尽可能发现内核中的 bug ，在这里：

- 四个逻辑段的 PLV_L 和 PLV_H 标志位均未被设置，使得 CPU 只能在处于 PLV0 特权级时访问它们；
- 代码段 ``.text`` 不允许被修改；
- 只读数据段 ``.rodata`` 不允许被修改，也不允许从它上面取指执行；
- ``.data/.bss`` 均允许被读写，但是不允许从它上面取指执行。

此外， :ref:`之前 <modify-page-table>` 提到过内核地址空间中需要存在一个恒等映射到内核数据段之外的可用物理页帧的逻辑段，这样才能在启用页表机制之后，内核仍能以纯软件的方式读写这些物理页帧。它们的标志位仅包含 rw ，意味着该逻辑段只能在 PLV0 特权级访问，并且只能读写。

下面我们给出创建内核地址空间的方法 ``new_kernel`` ：

.. code-block:: rust
    :linenos:

    // os/src/mm/memory_set.rs

    extern "C" {
        fn stext();
        fn etext();
        fn srodata();
        fn erodata();
        fn sdata();
        fn edata();
        fn sbss_with_stack();
        fn ebss();
        fn ekernel();
        fn strampoline();
    }

    impl MemorySet {
        /// Without kernel stacks.
        pub fn new_kernel() -> Self {
            let mut memory_set = Self::new_bare();
            // map kernel sections
            println!(".text [{:#x}, {:#x})", stext as usize, etext as usize);
            println!(".rodata [{:#x}, {:#x})", srodata as usize, erodata as usize);
            println!(".data [{:#x}, {:#x})", sdata as usize, edata as usize);
            println!(".bss [{:#x}, {:#x})", sbss_with_stack as usize, ebss as usize);
            println!("mapping .text section");
            memory_set.push(
                MapArea::new(
                    (stext as usize).into(),
                    (etext as usize).into(),
                    MapType::Identical,
                    MapPermission::from_bits(0).unwrap(),
                ),
                None,
            );
            println!("mapping .rodata section");
            memory_set.push(
                MapArea::new(
                    (srodata as usize).into(),
                    (erodata as usize).into(),
                    MapType::Identical,
                    MapPermission::NX,
                ),
                None,
            );
            println!("mapping .data section");
            memory_set.push(
                MapArea::new(
                    (sdata as usize).into(),
                    (edata as usize).into(),
                    MapType::Identical,
                    MapPermission::NX | MapPermission::W | MapPermission::D,
                ),
                None,
            );
            println!("mapping .bss section");
            memory_set.push(
                MapArea::new(
                    (sbss_with_stack as usize).into(),
                    (ebss as usize).into(),
                    MapType::Identical,
                    MapPermission::NX | MapPermission::W | MapPermission::D,
                ),
                None,
            );
            println!("mapping physical memory");
            memory_set.push(
                MapArea::new(
                    (ekernel as usize).into(),
                    MEMORY_END.into(),
                    MapType::Identical,
                    MapPermission::NX | MapPermission::W | MapPermission::D,
                ),
                None,
            );
            println!("mapping memory-mapped registers");
            for pair in MMIO {
                memory_set.push(
                    MapArea::new(
                        (*pair).0.into(),
                        ((*pair).0 + (*pair).1).into(),
                        MapType::Identical,
                        MapPermission::NX | MapPermission::W | MapPermission::D,
                    ),
                    None,
                );
            }
            memory_set
        }
    }

``new_kernel`` 将映射低半合法空间中的内核逻辑段。第 3 行开始，我们从 ``os/src/linker.ld`` 中引用了很多表示各个段位置的符号，而后在 ``new_kernel`` 中，我们从低地址到高地址依次创建 4 个逻辑段并通过 ``push`` 方法将它们插入到内核地址空间中，上面我们已经详细介绍过这 4 个逻辑段。

.. _term-vm-app-addr-space:

应用地址空间
------------------------------------------

现在我们来介绍如何创建应用的地址空间。在前面的章节中，我们直接将丢弃了所有符号信息的应用二进制镜像链接到内核，在初始化的时候内核仅需将他们加载到正确的初始物理地址就能使它们正确执行。但本章中，我们希望效仿内核地址空间的设计，同样借助页表机制使得应用地址空间的各个逻辑段也可以有不同的访问方式限制，这样可以提早检测出应用的错误并及时将其终止以最小化它对系统带来的恶劣影响。

在第三章中，每个应用链接脚本中的起始地址被要求是不同的，这样它们的代码和数据存放的位置才不会产生冲突。但这是一种对于应用开发者很不方便的设计。现在，借助地址空间的抽象，我们终于可以让所有应用程序都使用同样的起始地址，这也意味着所有应用可以使用同一个链接脚本了：

.. code-block:: 
    :linenos:

    /* user/src/linker.ld */

    OUTPUT_ARCH(loongarch)
    ENTRY(start)

    BASE_ADDRESS = 0x10000;

    SECTIONS
    {
        . = BASE_ADDRESS;
        .text : {
            *(.text.entry)
            *(.text .text.*)
        }
        . = ALIGN(16K);
        .rodata : {
            *(.rodata .rodata.*)
            *(.srodata .srodata.*)
        }
        . = ALIGN(16K);
        .data : {
            *(.data .data.*)
            *(.sdata .sdata.*)
        }
        .bss : {
            *(.bss .bss.*)
            *(.sbss .sbss.*)
        }
        /DISCARD/ : {
            *(.eh_frame)
            *(.debug*)
        }
    }

我们将起始地址 ``BASE_ADDRESS`` 设置为 :math:`\text{0x10000}` （我们这里并不设置为 :math:`\text{0x0}` ，因为它一般代表空指针），它是一个地址空间中的虚拟地址。事实上由于我们将入口汇编代码段放在最低的地方，这也是整个应用的入口点。我们只需清楚这一事实即可，而无需像之前一样将其硬编码到代码中。此外，在 ``.text`` 和 ``.rodata`` 中间以及 ``.rodata`` 和 ``.data`` 中间我们进行了页面对齐，因为前后两个逻辑段的访问方式限制是不同的，由于我们只能以页为单位对这个限制进行设置，因此就只能将下一个逻辑段对齐到下一个页面开始放置。而 ``.data`` 和 ``.bss`` 两个逻辑段由于访问限制相同（可读写），它们中间则无需进行页面对齐。

下图展示了应用地址空间低半合法部分的布局：

.. image:: app-as-full.png
    :align: center
    :height: 400
    
从 :math:`\text{0x10000}` 开始向高地址放置应用内存布局中的各个逻辑段，最后放置带有一个保护页面的用户栈。这些逻辑段都是以 ``Framed`` 方式映射到物理内存的，从访问方式上来说都加上了 PLV_L 和 PLV_H 标志位代表 CPU 可以在 PLV3 特权级也就是执行应用代码的时候访问它们。

在 ``os/src/build.rs`` 中，我们不再将丢弃了所有符号的应用二进制镜像链接进内核，因为在应用二进制镜像中，内存布局中各个逻辑段的位置和访问限制等信息都被裁剪掉了。我们直接使用保存了逻辑段信息的 ELF 格式的应用可执行文件。这样 ``loader`` 子模块的设计实现也变得精简：

.. code-block:: rust

    // os/src/loader.rs

    pub fn get_num_app() -> usize {
        extern "C" { fn _num_app(); }
        unsafe { (_num_app as usize as *const usize).read_volatile() }
    }

    pub fn get_app_data(app_id: usize) -> &'static [u8] {
        extern "C" { fn _num_app(); }
        let num_app_ptr = _num_app as usize as *const usize;
        let num_app = get_num_app();
        let app_start = unsafe {
            core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1)
        };
        assert!(app_id < num_app);
        unsafe {
            core::slice::from_raw_parts(
                app_start[app_id] as *const u8,
                app_start[app_id + 1] - app_start[app_id]
            )
        }
    }

它仅需要提供两个函数： ``get_num_app`` 获取链接到内核内的应用的数目，而 ``get_app_data`` 则根据传入的应用编号取出对应应用的 ELF 格式可执行文件数据。它们和之前一样仍是基于 ``build.rs`` 生成的 ``link_app.S`` 给出的符号来确定其位置，并实际放在内核的数据段中。 ``loader`` 模块中原有的内核和用户栈则分别作为逻辑段放在跳板空间和用户地址空间中，我们无需再去专门为其定义一种类型。

在创建应用地址空间的时候，我们需要对 ``get_app_data`` 得到的 ELF 格式数据进行解析，找到各个逻辑段所在位置和访问限制并插入进来，最终得到一个完整的应用地址空间：

.. code-block:: rust
    :linenos:

    // os/src/mm/memory_set.rs

    impl MemorySet {
        /// Include sections in elf and trampoline and TrapContext and user stack,
        /// also returns user_sp and entry point.
        pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize) {
            let mut memory_set = Self::new_bare();
            // map program headers of elf, with PLV flags
            let elf = xmas_elf::ElfFile::new(elf_data).unwrap();
            let elf_header = elf.header;
            let magic = elf_header.pt1.magic;
            assert_eq!(magic, [0x7f, 0x45, 0x4c, 0x46], "invalid elf!");
            let ph_count = elf_header.pt2.ph_count();
            let mut max_end_vpn = VirtPageNum(0);
            for i in 0..ph_count {
                let ph = elf.program_header(i).unwrap();
                if ph.get_type().unwrap() == xmas_elf::program::Type::Load {
                    let start_va: VirtAddr = (ph.virtual_addr() as usize).into();
                    let end_va: VirtAddr = ((ph.virtual_addr() + ph.mem_size()) as usize).into();
                    let mut map_perm = MapPermission::PLV_L | MapPermission::PLV_H;
                    let ph_flags = ph.flags();
                    if !ph_flags.is_read() {
                        map_perm |= MapPermission::NR;
                    }
                    if ph_flags.is_write() {
                        map_perm |= MapPermission::W | MapPermission::D;
                    }
                    if !ph_flags.is_execute() {
                        map_perm |= MapPermission::NX;
                    }
                    let map_area = MapArea::new(start_va, end_va, MapType::Framed, map_perm);
                    max_end_vpn = map_area.vpn_range.get_end();
                    memory_set.push(
                        map_area,
                        Some(&elf.input[ph.offset() as usize..(ph.offset() + ph.file_size()) as usize]),
                    );
                }
            }
            // map user stack with PLV flags
            let max_end_va: VirtAddr = max_end_vpn.into();
            let mut user_stack_bottom: usize = max_end_va.into();
            // guard page
            user_stack_bottom += PAGE_SIZE;
            let user_stack_top = user_stack_bottom + USER_STACK_SIZE;
            memory_set.push(
                MapArea::new(
                    user_stack_bottom.into(),
                    user_stack_top.into(),
                    MapType::Framed,
                    MapPermission::NX | MapPermission::W | MapPermission::D | MapPermission::PLV_L | MapPermission::PLV_H,
                ),
                None,
            );
            ( memory_set, user_stack_top, elf.header.pt2.entry_point() as usize, )
        }
    }

- 第 9 行，我们使用外部 crate ``xmas_elf`` 来解析传入的应用 ELF 数据并可以轻松取出各个部分。:ref:`此前 <term-elf>` 我们简要介绍过 ELF 格式的布局。第 12 行，我们取出 ELF 的魔数来判断它是不是一个合法的 ELF 。 
  
  第 13 行，我们可以直接得到 program header 的数目，然后遍历所有的 program header 并将合适的区域加入到应用地址空间中。这一过程的主体在第 15~38 行之间。第 17 行我们确认 program header 的类型是 ``LOAD`` ，这表明它有被内核加载的必要，此时不必理会其他类型的 program header 。接着通过 ``ph.virtual_addr()`` 和 ``ph.mem_size()`` 来计算这一区域在应用地址空间中的位置，通过 ``ph.flags()`` 来确认这一区域访问方式的限制并将其转换为 ``MapPermission`` 类型（注意它默认包含 PLV_L 和 PLV_H 标志位）。最后我们在第 31 行创建逻辑段 ``map_area`` 并在第 33 行 ``push`` 到应用地址空间。在 ``push`` 的时候我们需要完成数据拷贝，当前 program header 数据被存放的位置可以通过 ``ph.offset()`` 和 ``ph.file_size()`` 来找到。 注意当存在一部分零初始化的时候， ``ph.file_size()`` 将会小于 ``ph.mem_size()`` ，因为这些零出于缩减可执行文件大小的原因不应该实际出现在 ELF 数据中。
- 我们从第 39 行开始处理用户栈。注意在前面加载各个 program header 的时候，我们就已经维护了 ``max_end_vpn`` 记录目前涉及到的最大的虚拟页号，只需紧接着在它上面再放置一个保护页面和用户栈即可。
- 第 54 行返回的时候，我们不仅返回应用地址空间 ``memory_set`` ，也同时返回用户栈虚拟地址 ``user_stack_top`` 以及从解析 ELF 得到的该应用入口点地址，它们将被我们用来创建应用的任务控制块。


小结一下，本节讲解了 **地址空间** 这一抽象概念的含义与对应的具体数据结构设计与实现，并进一步介绍了在分页机制的帮助下，内核和应用各自的地址空间的基本组成和创建这两种地址空间的基本方法。接下来，需要考虑如何把地址空间与之前的分时多任务结合起来，实现一个更加安全和强大的内核，这还需要进一步拓展内核功能 -- 建立具体的内核虚拟地址空间和应用虚拟地址空间、实现不同地址空间的切换，即能切换不同应用之间的地址空间，以及应用与内核之间的地址空间。在下一节，我们将讲解如何构建基于地址空间的分时多任务操作系统 -- “头甲龙”。

.. hint::
    
    **内核如何访问应用的数据？** 

    应用应该不能直接访问内核的数据，但内核可以访问应用的数据，这是如何做的？由于内核要管理应用，所以它负责构建自身和其他应用的多级页表。如果内核获得了一个应用数据的虚地址，内核就可以通过查询应用的页表来把应用的虚地址转换为物理地址，内核直接访问这个地址（注：内核自身的虚实映射是恒等映射），就可以获得应用数据的内容了。

.. _term-trampoline:

跳板空间的实现
------------------------------------

上一小节我们曾多次提到跳板地址空间。那么跳板究竟起什么作用呢？

回忆曾在第二章介绍过的 :ref:`Trap 上下文保存与恢复 <trap-context-save-restore>` 。当一个应用 Trap 到内核时，``save0`` 已指向该应用的内核栈栈顶，我们用一条指令即可从用户栈切换到内核栈，然后直接将 Trap 上下文压入内核栈栈顶。当 Trap 处理完毕返回用户态的时候，将 Trap 上下文中的内容恢复到寄存器上，最后将保存着应用用户栈顶的 ``save0`` 与 sp 进行交换，也就从内核栈切换回了用户栈。在这个过程中， ``save0`` 起到了非常关键的作用，它使得我们可以在不破坏任何通用寄存器的情况下，完成用户栈与内核栈的切换，以及位于内核栈顶的 Trap 上下文的保存与恢复。

然而，一旦使能了分页机制，一切就并没有这么简单了，我们必须在这个过程中同时完成地址空间的切换。具体来说，当 ``__alltraps`` 保存 Trap 上下文的时候，我们必须通过修改 pgdl 从应用地址空间切换到内核地址空间，因为 trap handler 只有在内核地址空间中才能访问；同理，在 ``__restore`` 恢复 Trap 上下文的时候，我们也必须从内核地址空间切换回应用地址空间，因为应用的代码和数据只能在它自己的地址空间中才能访问，应用是看不到内核地址空间的。这样就要求地址空间的切换不能影响指令的连续执行，即要求应用和内核地址空间在切换地址空间指令附近是平滑的。

.. _term-meltdown:

.. note::

    **内核与应用地址空间的隔离**

    目前我们的设计思路 A 是：对内核建立唯一的内核地址空间存放内核的代码、数据，同时对于每个应用维护一个它们自己的用户地址空间，因此在 Trap 的时候就需要进行地址空间切换，而在任务切换的时候无需进行（因为这个过程全程在内核内完成）。

    另外的一种设计思路 B 是：让每个应用都有一个包含应用和内核的地址空间，并将其中的逻辑段分为内核和用户两部分，分别映射到内核/用户的数据和代码，且分别在 CPU 处于 S/U 特权级时访问。此设计中并不存在一个单独的内核地址空间。

    设计方式 B 的优点在于： Trap 的时候无需切换地址空间，而在任务切换的时候才需要切换地址空间。相对而言，设计方式B比设计方式A更容易实现，在应用高频进行系统调用的时候，采用设计方式B能够避免频繁地址空间切换的开销，这通常源于快表或 cache 的失效问题。但是设计方式B也有缺点：即内核的逻辑段需要在每个应用的地址空间内都映射一次，这会带来一些无法忽略的内存占用开销，并显著限制了嵌入式平台的任务并发数。此外，设计方式 B 无法防御针对处理器电路设计缺陷的侧信道攻击（如 `熔断 (Meltdown) 漏洞 <https://cacm.acm.org/magazines/2020/6/245161-meltdown/fulltext>`_ ），使得恶意应用能够以某种方式间接“看到”内核地址空间中的数据，使得用户隐私数据有可能被泄露。将内核与地址空间隔离便是修复此漏洞的一种方法。

    经过权衡，在本教程中我们参考 MIT 的教学 OS `xv6 <https://github.com/mit-pdos/xv6-riscv>`_ ，采用内核和应用地址空间隔离的设计。

问题的关键在于特权级的切换和地址空间的切换不是同步的。具体来说，当用户程序执行时，CPU 的特权级为 PLV3，而地址空间为应用地址空间（即 pgdl 中存放的是应用地址空间根页表的物理地址）。当用户程序陷入异常或中断的时候，CPU 的特权级会在发生异常或中断时自动被硬件设置为 PLV0 且自动跳转到对应的处理程序的首地址然后开始执行该处理程序，但是硬件不会自动完成地址空间的切换（也就是 pgdl 不会被硬件自动设置为内核地址空间根页表的物理地址），即特权级的切换和地址空间的切换不是同步的！

因此，我们需要在合适的地方手动实现地址空间的切换（即寄存器的修改），这当然应该放在 ``__alltraps`` 和 ``__restore`` 当中。但是，这会造成什么问题呢？考虑下面的场景：假设我们将 ``__alltraps`` 代码映射到应用地址空间的虚拟地址与映射到内核地址空间的虚拟地址不同，那么当修改 pgdl 寄存器的指令执行后，PC 寄存器中保存的是该指令的下一条指令在应用地址空间当中的虚拟地址，而此时地址空间已经切换为内核地址空间，使用这一虚拟地址显然不能在内核地址空间中取到本应执行的下一行代码。

问题出在在应用地址空间和内核地址空间中相同的虚拟地址不一定映射到相同的物理地址。所以这个问题其实不难解决，我们只需要在内核和所有应用的地址空间中约定一个统一的虚拟地址（虚拟页号）均映射到上面的代码所在的物理地址即可正确取到下一条指令。但是妨碍地址空间平滑过渡的不只是 pc，还有 sp 等，在地址空间切换前，sp 指向应用地址空间中内核栈的栈顶的虚拟地址，而地址空间切换后，这一虚拟地址在内核地址空间中则可能已经指向了其它地址。这显然无法直接通过让内核与应用地址空间约定同一个虚拟地址为内核栈栈顶来解决，因为不同的应用在内核地址空间中的内核栈不能重叠。但是，如果以内核地址空间为标准，不同的应用在其地址空间中完成内核地址空间中该应用的内核栈所对应的页表映射关系的拷贝，也能解决这一问题。

我们发现，上述问题的解决主要需要完成一项工作，即在不同的地址空间中使相同的虚拟地址映射到相同的物理地址，这样才能够实现地址空间转换前后的平滑过渡。如何完成这一目标？我们可以在构建各个地址空间时就完成“相同的虚拟地址映射到相同的物理地址”的工作，但实际上，LoongArch 给我们提供了 pgdl 和 pgdh 两个寄存器，我们也可以让所有地址空间的高半合法部分都使用同一个映射关系，即将上面提到的代码和所有应用的内核栈都放在 pgdh 所维护的高半合法地址空间中，而 pgdh 在内核和所有应用间共享，自始至终不发生切换。这样，无论是在应用还是在内核，用高半合法空间中的同一个虚拟地址一定访问到的是同一个物理地址。

所以，这一机制实际上是让所有合法地址空间（内核的或应用的）的高半部分使用同一个映射关系，为此，我们可以基于 **跳板** 的概念，提出一个 **跳板地址空间** ，用于抽象出这一在各个地址空间中共享的高半合法部分，即用一个单独的 ``MemorySet`` 来管理这一部分空间。下图展示了这一地址空间的布局：

.. image:: kernel-as-high.png
    :name: kernel-as-high
    :align: center
    :height: 400

可以看到，跳板放在最高的一个虚拟页面中。接下来则是从高到低放置每个应用的内核栈，内核栈的大小由 ``config`` 子模块的 ``KERNEL_STACK_SIZE`` 给出。它们的映射方式为 ``MapPermission`` 中的 rw 两个标志位，意味着这个逻辑段仅允许 CPU 处于内核态访问，且只能读或写。

.. _term-guard-page:

注意相邻两个内核栈之间会预留一个 **保护页面** (Guard Page) ，它是内核地址空间中的空洞，多级页表中并不存在与它相关的映射。它的意义在于当内核栈空间不足（如调用层数过多或死递归）的时候，代码会尝试访问空洞区域内的虚拟地址，然而它无法在多级页表中找到映射，便会触发异常，此时控制权会交给内核 trap handler 函数进行异常处理。由于编译器会对访存顺序和局部变量在栈帧中的位置进行优化，我们难以确定一个已经溢出的栈帧中的哪些位置会先被访问，但总的来说，空洞区域被设置的越大，我们就能越早捕获到这一可能覆盖其他重要数据的错误异常。由于我们的内核非常简单且内核栈的大小设置比较宽裕，在当前的设计中我们仅将空洞区域的大小设置为单个页面。

.. note:: **关于一些概念的说明**

    **跳板** 与 **跳板空间**

    在本书的 RISC-V 版本中，跳板指的是放置 ``__alltraps`` 和 ``__restore`` 代码的页面，跳板的一个显著的特点就是“相同的虚拟地址映射到相同的物理地址”。而在本书中，我们借鉴了这一思想，使对所有应用的内核栈的映射也具有了这一特点，为了区别于跳板这一概念，我们使用了跳板空间这一表述。不严谨地讲，本书所谓的“跳板空间”是包含“跳板”和所有应用的内核栈的。

    **地址空间**

    在本书中，我们将跳板空间也认为是一个地址空间，其实这一说法从传统的地址空间的含义上来说并不严谨，跳板空间并不是一块独立的地址空间，它只是各个地址空间的高半合法部分。实际上，我们这样定义也与 LoongArch 的硬件设计有关，LoongArch 为地址空间提供了 pgdl 和 pgdh 两个寄存器用于分别存储低半合法部分和高半合法部分的根页表的物理地址，如果按照传统的地址空间的概念，一个地址空间（我们实现为 ``MemorySet``）就应该包含两个多级页表，但在本实验中更合适的实现是每个 ``MemorySet`` 只包含一个多级页表，因此，我们称跳板空间也是一个地址空间主要是因为它也包含了一个多级页表，也刚好可以用一个 ``MemorySet`` 来实现。在本书中，地址空间在不同的语境下有不同的含义，有时指传统的地址空间，有时指一个 ``MemorySet``，但我们认为这并不影响读者对相关内容的理解，所以不对这一概念进行严格区分。

.. note:: **一定需要跳板空间吗？**

    前面提到，我们需要跳板空间，最根本的原因在于特权级的切换和地址空间的切换不是同步的。实际上，如果硬件能够实现特权级和地址空间的同步切换，那么上述问题都无需考虑了，跳板空间也不必存在。请读者自行思考推演在特权级和地址空间同步切换的前提下用户态和内核态切换的具体过程，即可感受到确实无需再建立跳板空间。

    实际上，LoongArch 的 :ref:`直接映射模式 <term-direct-map-mode>` 能够从硬件上做到特权级和地址空间的同步切换。具体来说，我们可以配置一个映射整个物理地址空间的映射窗口（实际上相当于前面的恒等映射）并将其特权级设置为 PLV0。这样，只有在 PLV0 特权级下该映射窗口才可用，才能实现直接通过与物理地址相同的虚拟地址访问对应的物理地址。并且，在特权级转换的瞬间，对映射窗口的访问使能也同时转换，相当于实现了特权级和地址空间的同步切换。

    可以看到，通过配置该映射窗口，不仅无需再实现跳板空间，甚至整个内核空间也无需再实现，毕竟内核空间的主要目的就是使内核能直接通过与物理地址相同的虚拟地址来访问对应的物理地址，这会使得我们的内核实现大大简化！

    但是，这一方式也有其缺点，即它不能够实现细粒度的存储访问类型控制。虽然映射窗口能够实现基本的读写执行等存储访问类型控制，但是它只能对它所映射的整个空间整体进行这样的限制，而这个粒度是很粗的。使用这种方式实现的内核空间，可以说几乎没有提供内核对不同区域的存储访问类型的控制，这不利于我们发现我们内核实现的缺陷，也不能实现用户程序内核栈间的 ``guard page`` 等。如果只是使用映射窗口而不添加额外的措施来进行存储访问类型控制，不能称得上是一个较好的实践。

    但是作为一个实验性质的教学操作系统，我们可以不必过于追求操作系统的健壮性，所以鼓励大家尝试用这种方式实现内核对物理地址的访问，我们将此作为本章的一道课后习题。

扩展 Trap 上下文
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

为了方便实现，我们在 Trap 上下文中包含更多内容（和我们关于上下文的定义有些不同，它们在初始化之后便只会被读取而不会被写入，并不是每次都需要保存/恢复）：

.. code-block:: rust
    :linenos:
    :emphasize-lines: 8,9

    // os/src/trap/context.rs

    #[repr(C)]
    pub struct TrapContext {
        pub r: [usize; 32],
        pub prmd: Prmd,
        pub era: usize,
        pub kernel_pgdl: usize,
        pub trap_handler: usize,
    }

在多出的三个字段中：

- ``kernel_pgdl`` 表示内核地址空间的 token ，即内核页表的起始物理地址；
- ``trap_handler`` 表示内核中 trap handler 入口点的虚拟地址。

它们在应用初始化的时候由内核写入应用地址空间中的 TrapContext 的相应位置，此后就不再被修改。

切换地址空间
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

让我们来看一下现在的 ``__alltraps`` 和 ``__restore`` 各是如何在保存和恢复 Trap 上下文的同时也切换地址空间的：

.. code-block:: loongarch
    :linenos:

    # os/src/trap/trap.S

        .section .text.trampoline
        .globl __alltraps
        .globl __restore
        .align 2
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
        # load kernel_pgdl into t0
        ld.d $t0, $sp, 34*8
        # load trap_handler into t1
        ld.d $t1, $sp, 35*8
        # switch to kernel space
        csrwr $t0, 0x19
        invtlb 0x0, $r0, $r0
        # jump to trap_handler
        move $a0, $sp
        jirl $r0, $t1, 0x0

    __restore:
        # a0: user space token
        csrwr $a0, 0x19
        invtlb 0x0, $r0, $r0
        # now sp->kernel stack, start restoring based on it
        # restore prmd/era
        ld.d $t0, $sp, 32*8
        ld.d $t1, $sp, 33*8
        ld.d $t2, $sp, 3*8
        csrwr $t0, 0x1
        csrwr $t1, 0x6
        csrwr $t2, 0x30
        # restore general-purpuse registers except sp/tp
        ld.d $r1, $sp, 1*8
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

- 当应用 Trap 进入内核的时候，硬件会设置一些 CSR 并在 PLV0 特权级下跳转到 ``__alltraps`` 保存 Trap 上下文。此时 sp 寄存器仍指向用户栈，但 ``save0`` 则被设置为跳板空间中该应用内核栈的栈顶。随后，就像之前一样，我们 ``csrrw`` 交换 sp 和 ``save0`` ，并基于指向内核栈栈顶的 sp 开始保存通用寄存器和一些 CSR ，这个过程在第 29 行结束。到这里，我们就完成了保存 Trap 上下文的工作。
  
- 接下来该考虑切换到内核地址空间并跳转到 trap handler 了。

  - 第 31 行将内核地址空间的 token 载入到 t0 寄存器中；
  - 第 33 行将 trap handler 入口点的虚拟地址载入到 t1 寄存器中；

  注：这两条信息均是内核在初始化该应用的时候就已经设置好的。

  - 第 35~36 行将 pgdl 修改为内核地址空间的 token 并使用 ``invtlb 0x0, $r0, $r0`` 刷新快表，这就切换到了内核地址空间；
  - 第 39 行 最后通过 ``jirl`` 指令跳转到 t1 寄存器所保存的 trap handler 入口点的地址。

  注：这里我们不能像之前的章节那样直接 ``call trap_handler`` ，原因稍后解释。

- 当内核将 Trap 处理完毕准备返回用户态的时候会 *调用* ``__restore`` （符合LoongArch函数调用规范），它有一个参数：即将回到的应用的地址空间的 token ，在 a0 寄存器中传递。

  - 第 43~44 行先切换回应用地址空间。
  - 第 63 行交换 ``sp`` 和 ``save0``。
  - 第 64 行最后通过 ``ertn`` 指令返回用户态。


建立跳板页面
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


接下来还需要考虑切换地址空间前后指令能否仍能连续执行。可以看到我们将 ``trap.S`` 中的整段汇编代码放置在 ``.text.trampoline`` 段，并在调整内存布局的时候将它对齐到代码段的一个页面中：

.. code-block:: diff
    :linenos:

    # os/src/linker.ld

        stext = .;
        .text : {
            *(.text.entry)
    +        . = ALIGN(16K);
    +        strampoline = .;
    +        *(.text.trampoline);
    +        . = ALIGN(16K);
            *(.text .text.*)
        }

这样，这段汇编代码放在一个物理页帧中，且 ``__alltraps`` 恰好位于这个物理页帧的开头，其物理地址被外部符号 ``strampoline`` 标记。在开启分页模式之后，内核和应用代码都只能看到各自的虚拟地址空间，而在它们的视角中，这段汇编代码都被放在它们各自地址空间的最高虚拟页面上，由于这段汇编代码在执行的时候涉及到地址空间切换，故而被称为跳板页面。

建立跳板空间
^^^^^^^^^^^^^^^^^^^^

和建立内核地址空间和应用地址空间一样，我们也需要一个函数来建立跳板空间：

.. code-block:: rust
    :linenos:

    // os/src/config.rs

    pub const TRAMPOLINE: usize = usize::MAX - PAGE_SIZE + 1;

    // os/src/mm/memory_set.rs

    impl MemorySet {
        pub fn new_trampoline() -> Self {
            let mut memory_set = Self::new_bare();
            memory_set.page_table.map(
                VirtAddr::from(TRAMPOLINE).into(),
                PhysAddr::from(strampoline as usize).into(),
                PTEFlags::from_bits(0).unwrap(),
            );
            memory_set
        }
    }

这里我们为了实现方便并没有新增逻辑段 ``MemoryArea`` 而是直接在多级页表中插入一个从地址空间的最高虚拟页面映射到跳板汇编代码所在的物理页帧的键值对，访问权限与代码段相同，即可读可执行。

最后可以解释为何我们在 ``__alltraps`` 中需要借助寄存器 ``jr`` 而不能直接 ``call trap_handler`` 了。因为在内存布局中，这条 ``.text.trampoline`` 段中的跳转指令和 ``trap_handler`` 都在代码段之内，汇编器（Assembler）和链接器（Linker）会根据链接脚本中的地址布局描述，设定跳转指令的地址，并计算二者地址偏移量，让跳转指令的实际效果为当前 pc 自增这个偏移量。但实际上由于我们设计的缘故，这条跳转指令在被执行的时候，它的虚拟地址被操作系统内核设置在地址空间中的最高页面之内，所以加上这个偏移量并不能正确的得到 ``trap_handler`` 的入口地址。

**问题的本质可以概括为：跳转指令实际被执行时的虚拟地址和在编译器/汇编器/链接器进行后端代码生成和链接形成最终机器码时设置此指令的地址是不同的。** 
