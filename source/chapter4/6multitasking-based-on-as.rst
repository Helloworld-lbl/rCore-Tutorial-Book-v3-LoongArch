基于地址空间的分时多任务
==============================================================

本节导读
--------------------------

本节我们介绍如何基于地址空间抽象而不是对于物理内存的直接访问来实现支持地址空间隔离的分时多任务系统 -- “头甲龙” [#tutus]_ 操作系统 。这样，我们的应用编写会更加方便，应用与操作系统内核的空间隔离性增强了，应用程序和操作系统自身的安全性也得到了加强。为此，需要对现有的操作系统进行如下的功能扩展：

- 创建内核页表，使能分页机制，建立内核的虚拟地址空间；
- 扩展Trap上下文，在保存与恢复Trap上下文的过程中切换页表（即切换虚拟地址空间）；
- 建立用于内核地址空间与应用地址空间相互切换所需的跳板空间；
- 扩展任务控制块包括虚拟内存相关信息，并在加载执行创建基于某应用的任务时，建立应用的虚拟地址空间；
- 改进Trap处理过程和sys_write等系统调用的实现以支持分离的应用地址空间和内核地址空间。

在扩展了上述功能后，应用与应用之间，应用与操作系统内核之间通过硬件分页机制实现了内存空间隔离，且应用和内核之间还是能有效地进行相互访问，而且应用程序的编写也会更加简单通用。

TLB 重填异常处理
------------------------------

在 :ref:`前面<term-tlbr-handle>` ，我们已经简单介绍了 LoongArch 中 TLB 重填异常的处理方式，并给出了一个简单的样例程序，这里我们基于该样例程序进行简单修改即可完成 TLB 重填异常的处理代码：

.. code-block:: 
    :linenos:

    # os/src/trap/tlbr.S

        .section .tlbrentry
        .globl __tlbrentry
        .align 2
    __tlbrentry:
        csrwr $t0, 0x8b
        csrrd $t0, 0x1b
        lddir $t0, $t0, 3
        srli.w $t0, $t0, 12
        slli.w $t0, $t0, 12
        lddir $t0, $t0, 1
        srli.w $t0, $t0, 12
        slli.w $t0, $t0, 12
        ldpte $t0, 0
        ldpte $t0, 1
        tlbfill
        csrrd $t0, 0x8b
        ertn

相比于样例代码，主要有两处修改：

- 第一，将这段汇编代码放置在了 ``.tlbrentry`` 段中。前面涉及到的普通例外和中断，其入口地址所在页的页号均在 EENTRY 这一 CSR 寄存器中，而用 Ecode 和 EsubCode 来区分不同的例外和中断。而 TLB 重填例外是一个特殊的例外，其入口程序的物理地址被单独设置在一个 CSR 寄存器 TLBRENTRY 中，因此我们需要将上述处理程序单独放在一个物理页中，并将对应的物理地址设置在 TLBRENTRY 寄存器中。为此我们需要在链接脚本中进行相应修改：

  .. code-block:: 
    :linenos:

    # os/src/linker.ld

        stext = .;
        .text : {
            *(.text.entry)
            . = ALIGN(16K);
            strampoline = .;
            *(.text.trampoline);
            . = ALIGN(16K);
            stlbrentry = .;
            *(.tlbrentry);
            . = ALIGN(16K);
            *(.text .text.*)
        }

  代码的第 9 行我们进行了页对齐，确保 ``.tlbrentry`` 被单独放置在一个物理页中，我们还用 ``stlbrentry`` 标记了处理程序的入口地址，方便后续填入 TLBRENTRY 寄存器中。

- 第二，我们在第10、11行，第13、14行均对 ``$t0`` 寄存器的低 12 位进行了清零（通过先右移后左移实现）。笔者发现，直接使用 ``LDDIR rd, rj, level`` 读出的 ``rd`` 寄存器中的内容并没有自动对低 12 位清零，也即会读出页表项中低 12 位里面的标志位，并在之后直接将这没有清零（按页对齐）的地址作为下一级页表的物理基地址去查询下一级页表，这显然会导致错误。

  笔者不清楚这一问题的成因，但目前可以通过两种方式暂时解决这一问题。

  第一种是可以将页目录项和页表项区分开，专门为页目录项定义一种数据结构或将页目录项的所有标志位置 0 。但是这样我们前面多级页表“按需分配”的逻辑就需要进行修改，因为我们是通过页目录项中的 V 标志位来判断该页目录项是否有效（否则就要分配一个物理页用于装载其对应的下一级页表）。

  第二种是直接将 ``rd`` 寄存器的低 12 位进行了清零。这种做法只需通过四行代码（甚至可以更少）就暂时解决这一问题，并且保留了页目录项的标志位，使页目录项和页表项的格式统一，所以笔者选用了这一种方法。

建立并开启基于分页模式的虚拟地址空间
--------------------------------------------

当 BIOS 实现初始化完成后， CPU 将跳转到内核入口点并在 PLV0 特权级上执行，此时还并没有开启分页模式，内核的每次访存是直接的物理内存访问。而在开启分页模式之后，内核代码在访存时只能看到内核地址空间，此时每次访存需要通过 MMU 的地址转换。这两种模式之间的过渡在内核初始化期间完成。

创建内核地址空间和跳板空间
^^^^^^^^^^^^^^^^^^^^^^^^

我们创建内核地址空间和跳板空间的全局实例：

.. code-block:: rust

    // os/src/mm/memory_set.rs

    lazy_static! {
        pub static ref KERNEL_SPACE: Arc<UPSafeCell<MemorySet>> =
            Arc::new(unsafe { UPSafeCell::new(MemorySet::new_kernel()) });
    }
    lazy_static! {
        pub static ref TRAMPOLINE_SPACE: Arc<UPSafeCell<MemorySet>> = 
            Arc::new(unsafe { UPSafeCell::new(MemorySet::new_trampoline()) });
    }

从之前对于 ``lazy_static!`` 宏的介绍可知， ``KERNEL_SPACE`` 和 ``TRAMPOLINE_SPACE`` 在运行期间它第一次被用到时才会实际进行初始化，而它所
占据的空间则是编译期被放在全局数据段中。这里使用 ``Arc<UPSafeCell<T>>`` 组合是因为我们既需要 ``Arc<T>`` 提供的共享
引用，也需要 ``UPSafeCell<T>`` 提供的内部可变引用访问。

在 ``rust_main`` 函数中，我们首先调用 ``mm::init`` 进行内存管理子系统的初始化：

.. code-block:: rust

    // os/src/mm/mod.rs

    pub use memory_set::KERNEL_SPACE;

    pub fn init() {
        heap_allocator::init_heap();
        frame_allocator::init_frame_allocator();
        init_tlb();
        KERNEL_SPACE.exclusive_access().activate();
        TRAMPOLINE_SPACE.exclusive_access().activate_trampoline();
    }

可以看到，我们最先进行了全局动态内存分配器的初始化，因为接下来马上就要用到 Rust 的堆数据结构。接下来我们初始化物理页帧管理器（内含堆数据结构 ``Vec<T>`` ）使能可用物理页帧的分配和回收能力。之后我们需要初始化 TLB 相关的 CSR 寄存器，这通过调用 ``init_tlb`` 函数来实现：

.. code-block:: rust
    :linenos:

    // os/src/mm/mod.rs

    fn init_tlb() {
        extern "C" {
            fn stlbrentry();
        } 
        tlbrentry::write_pa_to_tlbrentry(stlbrentry as usize);

        let mut pwcl = pwcl::read();
        let mut pwch = pwch::read();
        pwcl.set_ptbase(14);
        pwcl.set_ptwidth(11);
        // PMD
        pwcl.set_dir1_base(25);
        pwcl.set_dir1_width(11);
        // PGD
        pwch.set_dir3_base(36);
        pwch.set_dir3_width(11);
        pwcl.write();
        pwch.write();

        let mut stlbps = stlbps::read();
        stlbps.set_ps(0xe);
        stlbps.write();
    }

在该函数中，我们通过配置 ``tlbrentry`` 来设置 TLB 重填异常的入口地址，通过配置 ``pwcl`` 和 ``pwch`` 来定义了操作系统中所采用的页表结构，通过 ``stlbps`` 配置了 STLB 中页的大小。

然后我们创建内核地址空间并让 CPU 开启分页模式， MMU 在地址转换的时候使用内核的多级页表，这一切均在一行之内做到：

- 首先，我们引用 ``KERNEL_SPACE`` ，这是它第一次被使用，就在此时它会被初始化，调用 ``MemorySet::new_kernel`` 创建一个内核地址空间并使用 ``Arc<Mutex<T>>`` 包裹起来；
- 接着使用 ``.exclusive_access()`` 获取一个可变引用 ``&mut MemorySet`` 。需要注意的是这里发生了两次隐式类型转换：

  1.  我们知道 ``exclusive_access`` 是 ``UPSafeCell<T>`` 的方法而不是 ``Arc<T>`` 的方法，由于 ``Arc<T>`` 实现了 ``Deref`` Trait ，当 ``exclusive_access`` 需要一个 ``&UPSafeCell<T>`` 类型的参数的时候，编译器会自动将传入的 ``Arc<UPSafeCell<T>>`` 转换为 ``&UPSafeCell<T>`` 这样就实现了类型匹配；
  2.  事实上 ``UPSafeCell<T>::exclusive_access`` 返回的是一个 ``RefMut<'_, T>`` ，这同样是 RAII 的思想，当这个类型生命周期结束后互斥锁就会被释放。而该类型实现了 ``DerefMut`` Trait，因此当一个函数接受类型为 ``&mut T`` 的参数却被传入一个类型为 ``&mut RefMut<'_, T>`` 的参数的时候，编译器会自动进行类型转换使参数匹配。
- 最后，我们调用 ``MemorySet::activate`` ：

    .. code-block:: rust 
        :linenos:

        // os/src/mm/page_table.rs

        pub fn token(&self) -> usize {
            let root_pa: PhysAddr = self.root_ppn.into();
            root_pa.into()
        }

        // os/src/mm/memory_set.rs

        impl MemorySet {
            pub fn activate(&self) {
                unsafe {
                    let root_pa: PhysAddr = self.page_table.token().into();
                    pgdl::write_pa_to_pgdl(root_pa.into());
                    let mut crmd = crmd::read();
                    crmd.enable_pg();
                    asm!("invtlb 0x0, $r0, $r0");
                }
            }
        }

  ``PageTable::token`` 会返回将当前多级页表的根节点所在的物理地址。在 ``activate`` 中，我们将这个值写入当前 CPU 的 pgdl CSR ，然后修改 crmd CSR 使能分页模式，从这一刻开始分页模式就被启用了，而且 MMU 会使用内核地址空间的多级页表进行地址转换。

  我们必须注意切换 pgdl CSR 是否是一个 *平滑* 的过渡：其含义是指，切换 pgdl 的指令及其下一条指令这两条相邻的指令的虚拟地址是相邻的（由于切换 pgdl 的指令并不是一条跳转指令， pc 只是简单的自增当前指令的字长），而它们所在的物理地址一般情况下也是相邻的，但是它们所经过的地址转换流程却是不同的——切换 pgdl 导致 MMU 查的多级页表是不同的。这就要求前后两个地址空间在切换 pgdl 的指令 *附近* 的映射满足某种意义上的连续性。

  幸运的是，我们做到了这一点。这条写入 pgdl 的指令及之后的指令都在内核内存布局的代码段中，在切换之后是一个恒等映射，而在切换之前是视为物理地址直接取指，也可以将其看成一个恒等映射。这完全符合我们的期待：即使切换了地址空间，指令仍应该能够被连续的执行。

注意到在 ``activate`` 的最后，我们插入了一条汇编指令 ``invtlb 0x0, $r0, $r0`` ，它又起到什么作用呢？

让我们再来回顾一下多级页表：它相比线性表虽然大量节约了内存占用，但是却需要 MMU 进行更多的隐式访存。如果是一个线性表， MMU 仅需单次访存就能找到页表项并完成地址转换，而多级页表（以三级页表为例，不考虑大页）最顺利的情况下也需要三次访存。这些额外的访存和真正访问数据的那些访存在空间上并不相邻，加大了多级缓存的压力，一旦缓存缺失将带来巨大的性能惩罚。如果采用多级页表实现，这个问题会变得更为严重，使得地址空间抽象的性能开销过大。

.. _term-tlb:

为了解决性能问题，一种常见的做法是在 CPU 中利用部分硬件资源额外加入一个 **快表** (TLB, Translation Lookaside Buffer) ， 它维护了部分虚拟页号到页表项的键值对。当 MMU 进行地址转换的时候，首先会到快表中看看是否匹配，如果匹配的话直接取出页表项完成地址转换而无需访存；否则再去查页表并将键值对保存在快表中。一旦我们修改 pgdl 就会切换地址空间，快表中的键值对就会失效（因为快表保存着老地址空间的映射关系，切换到新地址空间后，老的映射关系就没用了）。为了确保 MMU 的地址转换能够及时与 pgdl 的修改同步，我们需要立即使用 ``invtlb 0x0, $r0, $r0`` 指令将快表清空，这样 MMU 就不会看到快表中已经过期的键值对了。

.. note::

    **INVTLB 指令**

    指令格式： ``invtlb  op, rj, rk``

    INVTLB 指令用于无效 TLB 中的内容，以维持 TLB 与内存之间页表数据的一致性。这里给出未实现 LVZ扩展的情况下，INVTLB 指令的功能定义。
    
    指令的三个源操作数中，op 是 5 比特立即数，用于指示操作类型。
    
    通用寄存器 rj 的[9:0]位存放无效操作所需的 ASID 信息（称为“寄存器指定 ASID”），其余比特必须填0。当 op 所指示的操作不需要 ASID 时，应将通用寄存器 rj 设置为 r0。
    
    通用寄存器 rk 中用于存放无效操作所需的虚拟地址信息（称为“寄存器指定 VA”）。当 op 所指示的操作不需要虚拟地址信息时，应将通用寄存器 rk 设置为 r0。
    
    各 op 对应的操作如下表所示，未在表中出现的 op 将触发保留指令例外。

    .. list-table:: 
        :header-rows: 1
        :widths: 10, 90
        :align: center

        * - op
          - 操作
        * - 0x0
          - 清除所有页表项。
        * - 0x1
          - 清除所有页表项。此时操作效果与 op=0 完全一致。
        * - 0x2
          - 清除所有 G=1 的页表项。
        * - 0x3
          - 清除所有 G=0 的页表项。
        * - 0x4
          - 清除所有 G=0，且 ASID 等于寄存器指定 ASID 的页表项。
        * - 0x5
          - 清除 G=0，且 ASID 等于寄存器指定 ASID，且 VA 等于寄存器指定 VA 的页表项。
        * - 0x6
          - 清除所有 G=1 或 ASID 等于寄存器指定 ASID，且 VA 等于寄存器指定 VA 的页表项。

.. note:: 
    
    **ASID 的作用**

    至此同学们应该对 ASID 的作用有所感受。如果我们为不同的地址空间设置不同的 ASID，那么在切换 pgdl CSR 时就不需要无效 TLB 中的所有内容了，因为即使是相同的虚拟地址，若该 TLB 表项中的 ASID 与 CSR.ASID 中 ASID 域的值不相等，该 TLB 表项也是非法的。

    因此，当一个地址空间被换出后又迅速换回时，其之前的 TLB 表项大概率还在 TLB 中，这时就无需触发 TLB 重填异常而去访存了。可见，合理使用 ASID，可以有效提高 TLB 的命中率。

    本章的一道习题就要求同学们实现对 ASID 域的利用。

最后我们创建跳板空间并使能跳板空间，它的 ``activate_trampoline`` 函数实现与内核地址空间的 ``activate`` 类似：

.. code-block:: rust

    // os/src/mm/memory_set.rs

    impl MemorySet {
        pub fn activate_trampoline(&self) {
            unsafe {
                let root_pa: PhysAddr = self.page_table.token().into();
                pgdh::write_pa_to_pgdh(root_pa.into());
                asm!("invtlb 0x0, $r0, $r0");
            }
        }
    }

检查内核地址空间的多级页表设置
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

调用 ``mm::init`` 之后我们就使能了内核动态内存分配、物理页帧管理，还启用了分页模式进入了内核地址空间。之后我们可以通过 ``mm::remap_test`` 来检查内核地址空间的多级页表是否被正确设置：

.. code-block:: rust

    // os/src/mm/memory_set.rs

    pub fn remap_test() {
        let mut kernel_space = KERNEL_SPACE.lock();
        let mid_text: VirtAddr = ((stext as usize + etext as usize) / 2).into();
        let mid_rodata: VirtAddr = ((srodata as usize + erodata as usize) / 2).into();
        let mid_data: VirtAddr = ((sdata as usize + edata as usize) / 2).into();
        assert_eq!(
            kernel_space.page_table.translate(mid_text.floor()).unwrap().writable(),
            false
        );
        assert_eq!(
            kernel_space.page_table.translate(mid_rodata.floor()).unwrap().writable(),
            false,
        );
        assert_eq!(
            kernel_space.page_table.translate(mid_data.floor()).unwrap().executable(),
            false,
        );
        println!("remap_test passed!");
    }

在上述函数的实现中，分别通过手动查内核多级页表的方式验证代码段和只读数据段不允许被写入，同时不允许从数据段上取指执行。

加载和执行应用程序
------------------------------------

扩展任务控制块
^^^^^^^^^^^^^^^^^^^^^^^^^^^

为了让应用在运行时有一个安全隔离且符合编译器给应用设定的地址空间布局的虚拟地址空间，操作系统需要对任务进行更多的管理，所以任务控制块相比第三章也包含了更多内容：

.. code-block:: rust
    :linenos:
    :emphasize-lines: 6,7

    // os/src/task/task.rs

    pub struct TaskControlBlock {
        pub task_status: TaskStatus,
        pub task_cx: TaskContext,
        pub memory_set: MemorySet,
        pub base_size: usize,
    }

除了应用的地址空间 ``memory_set`` 之外，还有 ``base_size`` 统计了应用数据的大小，也就是在应用地址空间中从 :math:`\text{0x0}` 开始到用户栈结束一共包含多少字节。它后续还应该包含用于应用动态内存分配的堆空间的大小，但目前暂不支持。

更新对任务控制块的管理
^^^^^^^^^^^^^^^^^^^^^^^^^^^

下面是任务控制块的创建：

.. code-block:: rust
    :linenos:

    // os/src/config.rs

    /// Return (bottom, top) of a kernel stack in kernel space.
    pub fn kernel_stack_position(app_id: usize) -> (usize, usize) {
        let top = TRAMPOLINE - app_id * (KERNEL_STACK_SIZE + PAGE_SIZE);
        let bottom = top - KERNEL_STACK_SIZE;
        (bottom, top)
    }

    // os/src/task/task.rs

    impl TaskControlBlock {
    pub fn new(elf_data: &[u8], app_id: usize) -> Self {
        // memory_set with elf program headers/trampoline/trap context/user stack
        let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
        let task_status = TaskStatus::Ready;
        // map a kernel-stack in kernel space
        let (kernel_stack_bottom, kernel_stack_top) = kernel_stack_position(app_id);
        TRAMPOLINE_SPACE.exclusive_access().insert_framed_area(
            kernel_stack_bottom.into(),
            kernel_stack_top.into(),
            MapPermission::NX | MapPermission::W | MapPermission::D,
        );
        let trap_cx = kernel_stack_top - size_of::<TrapContext>();
        let task_control_block = Self {
            task_status,
            task_cx: TaskContext::goto_trap_return(trap_cx),
            memory_set,
            base_size: user_sp,
        };
        // prepare TrapContext in user space
        let trap_cx = unsafe { (trap_cx as *mut TrapContext).as_mut().unwrap() };
        let kernel_pgdl = KERNEL_SPACE.exclusive_access().token();
        *trap_cx = TrapContext::app_init_context(
            entry_point,
            user_sp,
            kernel_pgdl,
            trap_handler as usize,
        );
        task_control_block
    }
    }

- 第 15 行，解析传入的 ELF 格式数据构造应用的地址空间 ``memory_set`` 并获得其他信息；
- 第 18 行，根据传入的应用 ID ``app_id`` 调用在 ``config`` 子模块中定义的 ``kernel_stack_position`` 找到
  应用的内核栈预计放在跳板空间 ``TRAMPOLINE_SPACE`` 中的哪个位置，并通过 ``insert_framed_area`` 实际将这个逻辑段
  加入到内核地址空间中；
- 第 24 行，计算出 Trap 上下文的起始地址为内核栈顶减去 TrapContext 所占字节数。

.. _trap-return-intro:

- 第 25~27 行，在应用的内核栈顶压入一个跳转到 ``trap_return`` 而不是 ``__restore`` 的任务上下文，这主要是为了能够支持对该应用的启动并顺利切换到用户地址空间执行。在构造方式上，只是将 ra 寄存器的值设置为 ``trap_return`` 的地址。 ``trap_return`` 是后面要介绍的新版的 Trap 处理的一部分。

..   这里对裸指针解引用成立的原因在于：当前已经进入了内核地址空间，而要操作的内核栈也是在内核地址空间中的；
- 第 26~30 行，用上面的信息来创建并返回任务控制块实例 ``task_control_block``；
- 第 34~39 行，调用 ``TrapContext::app_init_context`` 函数，通过应用的 Trap 上下文的可变引用来对其进行初始化。具体初始化过程如下所示：

  .. code-block:: rust
    :linenos:
    :emphasize-lines: 10,11,19,20

    // os/src/trap/context.rs

    impl TrapContext {
        /// set stack pointer to r_3 reg (sp)
        pub fn set_sp(&mut self, sp: usize) { self.r[3] = sp; }
        /// init app context
        pub fn app_init_context(
            entry: usize,
            sp: usize,
            kernel_pgdl: usize,
            trap_handler: usize,
        ) -> Self {
            let mut prmd = prmd::read(); // CSR sstatus
            prmd.set_pplv(PLV::PLV3); //previous privilege mode: plv3
            let mut cx = Self {
                r: [0; 32],
                prmd,
                era: entry, // entry point of app
                kernel_pgdl, // addr of page table
                trap_handler, // addr of trap_handler function
            };
            cx.set_sp(sp); // app's user stack pointer
            cx // return initial Trap Context of app
        }
    }

  和之前实现相比， ``TrapContext::app_init_context`` 需要补充上让应用在 ``__alltraps`` 能够顺利进入到内核地址空间并跳转到 trap handler 入口点的相关信息。

在内核初始化的时候，需要将所有的应用加载到全局应用管理器中：

.. code-block:: rust
    :linenos:

    // os/src/task/mod.rs

    struct TaskManagerInner {
        tasks: Vec<TaskControlBlock>,
        current_task: usize,
    }

    lazy_static! {
        pub static ref TASK_MANAGER: TaskManager = {
            println!("init TASK_MANAGER");
            let num_app = get_num_app();
            println!("num_app = {}", num_app);
            let mut tasks: Vec<TaskControlBlock> = Vec::new();
            for i in 0..num_app {
                tasks.push(TaskControlBlock::new(
                    get_app_data(i),
                    i,
                ));
            }
            TaskManager {
                num_app,
                inner: RefCell::new(TaskManagerInner {
                    tasks,
                    current_task: 0,
                }),
            }
        };
    }

可以看到，在 ``TaskManagerInner`` 中我们使用向量 ``Vec`` 来保存任务控制块。在全局任务管理器 ``TASK_MANAGER`` 初始化的时候，只需使用 ``loader`` 子模块提供的 ``get_num_app`` 和 ``get_app_data`` 分别获取链接到内核的应用数量和每个应用的 ELF 文件格式的数据，然后依次给每个应用创建任务控制块并加入到向量中即可。将 ``current_task`` 设置为 0 ，表示内核将从第 0 个应用开始执行。

回过头来介绍一下应用构建器 ``os/build.rs`` 的改动：

- 首先，我们在 ``.incbin`` 中不再插入清除全部符号的应用二进制镜像 ``*.bin`` ，而是将应用的 ELF 执行文件直接链接进来；
- 其次，在链接每个 ELF 执行文件之前我们都加入一行 ``.align 3`` 来确保它们对齐到 8 字节，这是由于如果不这样做， ``xmas-elf`` crate 可能会在解析 ELF 的时候进行不对齐的内存读写，例如使用 ``ld`` 指令从内存的一个没有对齐到 8 字节的地址加载一个 64 位的值到一个通用寄存器。

为了方便后续的实现，全局任务管理器还需要提供关于当前应用与地址空间有关的一些信息：

.. code-block:: rust
    :linenos:

    // os/src/task/mod.rs

    impl TaskManager {
        fn get_current_token(&self) -> usize {
            let inner = self.inner.borrow();
            let current = inner.current_task;
            inner.tasks[current].get_user_token()
        }

        fn get_current_trap_cx(&self) -> &mut TrapContext {
            let inner = self.inner.borrow();
            let current = inner.current_task;
            inner.tasks[current].get_trap_cx()
        }
    }

    pub fn current_user_token() -> usize {
        TASK_MANAGER.get_current_token()
    }

    pub fn current_trap_cx() -> &'static mut TrapContext {
        TASK_MANAGER.get_current_trap_cx()
    }

通过 ``current_user_token`` 可以获得当前正在执行的应用的地址空间的 token 。同时，该应用地址空间中的 Trap 上下文很关键，内核需要访问它来拿到应用进行系统调用的参数并将系统调用返回值写回，通过 ``current_trap_cx`` 内核可以拿到它访问这个 Trap 上下文的可变引用并进行读写。

改进 Trap 处理的实现
------------------------------------

让我们来看现在 ``trap_handler`` 的改进实现：

.. code-block:: rust
    :linenos:

    // os/src/trap/mod.rs

    fn set_kernel_trap_entry() {
        unsafe {
            eentry::write(trap_from_kernel as usize >> 12);
        }
    }

    #[no_mangle]
    pub fn trap_from_kernel() -> ! {
        panic!("a trap from kernel!");
    }

    #[no_mangle]
    pub fn trap_handler(cx: &mut TrapContext) -> ! {
        set_kernel_trap_entry();
        let estat = estat::read(); // get trap cause
        match estat.cause() {
            ...
        }
        unsafe { asm!("or $sp, $fp, $r0"); }
        trap_return();
    }

注意到，在 ``trap_handler`` 的开头调用了 ``set_kernel_trap_entry`` 将 ``eentry`` 修改为同模块下另一个函数 ``trap_from_kernel`` 的地址。这就是说，一旦进入内核后再次触发到 PLV0 态 Trap，则硬件在设置一些 CSR 寄存器之后，会跳过对通用寄存器的保存过程，直接跳转到 ``trap_from_kernel`` 函数，在这里直接 ``panic`` 退出。这是因为内核和应用的地址空间分离之后，PLV3 --> PLV0 与 PLV0 --> PLV3 的 Trap 上下文保存与恢复实现方式/Trap 处理逻辑有很大差别。这里为了简单起见，弱化了 PLV0 --> PLV0 的 Trap 处理过程：直接 ``panic`` 。

在 ``trap_handler`` 完成 Trap 处理之后，我们需要调用 ``trap_return`` 返回用户态。注意在调用 ``trap_return`` 之前我们插入了一行汇编代码 ``or $sp, $fp, $r0`` ，我们稍后会解释这行代码的作用。

.. code-block:: rust
    :linenos:

    // os/src/trap/mod.rs

    fn set_user_trap_entry() {
        unsafe {
            eentry::write(TRAMPOLINE as usize >> 12);
        }
    }

    #[no_mangle]
    pub fn trap_return() -> ! {
        set_user_trap_entry();
        let user_pgdl = current_user_token();
        extern "C" {
            fn __alltraps();
            fn __restore();
        }
        let restore_va = __restore as usize - __alltraps as usize + TRAMPOLINE;
        unsafe {
            asm!(
                "ibar 0",
                "or $sp, $fp, $r0",
                "jirl $r0, {restore_va}, 0x0",             // jump to new addr of __restore asm function
                restore_va = in(reg) restore_va,
                in("$a0") user_pgdl,        // a0 = phy addr of usr page table
                options(noreturn),
            );
        }
    }

- 第 11 行，在 ``trap_return`` 的开始处就调用 ``set_user_trap_entry`` ，来让应用 Trap 到 PLV0 的时候可以跳转到 ``__alltraps`` 。注：我们把 ``eentry`` 设置为内核和应用地址空间共享的跳板页面的起始地址 ``TRAMPOLINE`` 而不是编译器在链接时看到的 ``__alltraps`` 的地址。这是因为启用分页模式之后，内核只能通过跳板页面上的虚拟地址来实际取得 ``__alltraps`` 和 ``__restore`` 的汇编代码。
- 第 12 行，准备好 ``__restore`` 需要的参数：要继续执行的应用地址空间的 token 。
  
  最后我们需要跳转到 ``__restore`` ，以执行：切换到应用地址空间、从 Trap 上下文中恢复通用寄存器、 ``ertn`` 继续执行应用。它的关键在于如何找到 ``__restore`` 在内核/应用地址空间中共同的虚拟地址。

- 第 17 行，展示了计算 ``__restore`` 虚地址的过程：由于 ``__alltraps`` 是对齐到地址空间跳板页面的起始地址 ``TRAMPOLINE`` 上的， 则 ``__restore`` 的虚拟地址只需在 ``TRAMPOLINE`` 基础上加上 ``__restore`` 相对于 ``__alltraps`` 的偏移量即可。这里 ``__alltraps`` 和 ``__restore`` 都是指编译器在链接时看到的内核内存布局中的地址。

- 第 20-27 行，首先需要使用 ``ibar 0`` 指令清空指令缓存 i-cache 。这是因为，在内核中进行的一些操作可能导致一些原先存放某个应用代码的物理页帧如今用来存放数据或者是其他应用的代码，i-cache 中可能还保存着该物理页帧的错误快照。因此我们直接将整个 i-cache 清空避免错误。最后使用 ``jirl`` 指令完成了跳转到 ``__restore`` 的任务。
  注意 ``jirl`` 指令前又插入了一条 ``or $sp, $fp, $r0`` ，这里它的作用与上面类似。这行代码的作用是将 ``$fp`` 的值赋给 ``$sp`` ，即手动收回函数的栈帧。一般情况下，编译器会在函数返回时自动插入一行指令来回收给该函数分配的栈帧，但是这里执行 ``jirl`` 后就会跳转到 ``__restore`` 然后返回用户空间。也就是说，这一函数永远不会返回（前面的 ``trap_handler`` 也是），所以编译器的自动回收栈帧功能就实效了，此时需要我们手动回收栈帧，才能保证在进入 ``__restore`` 时 ``sp`` 指向的是 TrapContext 而不是 ``trap_return`` 或 ``trap_handler`` 的栈帧。

当每个应用第一次获得 CPU 使用权即将进入用户态执行的时候，它的内核栈顶放置着我们在 :ref:`内核加载应用的时候 <trap-return-intro>` 构造的一个任务上下文：

.. code-block:: rust

    // os/src/task/context.rs

    impl TaskContext {
        pub fn goto_trap_return(kstack_ptr: usize) -> Self {
            Self {
                ra: trap_return as usize,
                sp: kstack_ptr,
                s: [0; 10],
            }
        }
    }

在 ``__switch`` 切换到该应用的任务上下文的时候，内核将会跳转到 ``trap_return`` 并返回用户态开始该应用的启动执行。

改进 sys_write 的实现
------------------------------------

类似Trap处理的改进，由于内核和应用地址空间的隔离， ``sys_write`` 不再能够直接访问位于应用空间中的数据，而需要手动查页表才能知道那些数据被放置在哪些物理页帧上并进行访问。

为此，页表模块 ``page_table`` 提供了将应用地址空间中一个缓冲区转化为在内核空间中能够直接访问的形式的辅助函数：

.. code-block:: rust
    :linenos:

    // os/src/mm/page_table.rs

    pub fn translated_byte_buffer(
        token: usize,
        ptr: *const u8,
        len: usize
    ) -> Vec<&'static [u8]> {
        let page_table = PageTable::from_token(token);
        let mut start = ptr as usize;
        let end = start + len;
        let mut v = Vec::new();
        while start < end {
            let start_va = VirtAddr::from(start);
            let mut vpn = start_va.floor();
            let ppn = page_table
                .translate(vpn)
                .unwrap()
                .ppn();
            vpn.step();
            let mut end_va: VirtAddr = vpn.into();
            end_va = end_va.min(VirtAddr::from(end));
            if end_va.page_offset() == 0 {
                v.push(&mut ppn.get_bytes_array()[start_va.page_offset()..]);
            } else {
                v.push(&mut ppn.get_bytes_array()[start_va.page_offset()..end_va.page_offset()]);
            }
            start = end_va.into();
        }
        v
    }

参数中的 ``token`` 是某个应用地址空间的 token ， ``ptr`` 和 ``len`` 则分别表示该地址空间中的一段缓冲区的起始地址和长度(注：这个缓冲区的应用虚拟地址范围是连续的)。 ``translated_byte_buffer`` 会以向量的形式返回一组可以在内核空间中直接访问的字节数组切片（注：这个缓冲区的内核虚拟地址范围有可能是不连续的），具体实现在这里不再赘述。

进而我们可以完成对 ``sys_write`` 系统调用的改造：

.. code-block:: rust

    // os/src/syscall/fs.rs

    pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize {
        match fd {
            FD_STDOUT => {
                let buffers = translated_byte_buffer(current_user_token(), buf, len);
                for buffer in buffers {
                    print!("{}", core::str::from_utf8(buffer).unwrap());
                }
                len as isize
            },
            _ => {
                panic!("Unsupported fd in sys_write!");
            }
        }
    }

上述函数尝试将按应用的虚地址指向的缓冲区转换为一组按内核虚地址指向的字节数组切片构成的向量，然后把每个字节数组切片转化为字符串``&str`` 然后输出即可。



小结
-------------------------------------

这一章内容很多，讲解了 **地址空间** 这一抽象概念是如何在一个具体的“头甲龙”操作系统中实现的。这里面的核心内容是如何建立基于页表机制的虚拟地址空间。为此，操作系统需要知道并管理整个系统中的物理内存；需要建立虚拟地址到物理地址映射关系的页表；并基于页表给操作系统自身和每个应用提供一个虚拟地址空间；并需要对管理应用的任务控制块进行扩展，确保能对应用的地址空间进行管理；由于应用和内核的地址空间是隔离的，需要有一个跳板来帮助完成应用与内核之间的切换执行；并导致了对异常、中断、系统调用的相应更改。这一系列的改进，最终的效果是编写应用更加简单了，且应用的执行或错误不会影响到内核和其他应用的正常工作。为了得到这些好处，我们需要比较费劲地进化我们的操作系统。如果同学结合阅读代码，编译并运行应用+内核，读懂了上面的文档，那完成本章的实验就有了一个坚实的基础。

如果同学能想明白如何插入/删除页表；如何在 ``trap_handler`` 下处理 ``LoadPageFault`` ；以及 ``sys_get_time`` 在使能页机制下如何实现，那就会发现下一节的实验练习也许 **就和lab1一样** 。

.. [#tutus] 头甲龙最早出现在1.8亿年以前的侏罗纪中期，是身披重甲的食素恐龙，尾巴末端的尾锤，是防身武器。
