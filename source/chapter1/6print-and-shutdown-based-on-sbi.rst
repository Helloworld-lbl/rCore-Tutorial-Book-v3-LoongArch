基于 UART 完成输出
=================================================================

.. toctree::
   :hidden:
   :maxdepth: 5

本节导读
------------------------------------

本节我们将进行构建“三叶虫”操作系统的最后一个步骤，即基于 UART 完成在屏幕上打印 ``Hello world!`` 。

.. 使用 RustSBI 提供的服务
.. ------------------------------------------

.. 之前我们对 RustSBI 的了解仅限于它会在计算机启动时进行它所负责的环境初始化工作，并将计算机控制权移交给内核。但实际上作为内核的执行环境，它还有另一项职责：即在内核运行时响应内核的请求为内核提供服务。当内核发出请求时，计算机会转由 RustSBI 控制来响应内核的请求，待请求处理完毕后，计算机控制权会被交还给内核。从内存布局的角度来思考，每一层执行环境（或称软件栈）都对应到内存中的一段代码和数据，这里的控制权转移指的是 CPU 从执行一层软件的代码到执行另一层软件的代码的过程。这个过程与我们使用高级语言编程时调用库函数比较类似。

.. 这里展开介绍一些相关术语：从第二章将要讲到的 :ref:`RISC-V 特权级架构 <riscv-priv-arch>` 的视角来看，我们编写的 OS 内核位于 Supervisor 特权级，而 RustSBI 位于 Machine 特权级，也是最高的特权级。类似 RustSBI 这样运行在 Machine 特权级的软件被称为 Supervisor Execution Environment(SEE)，即 Supervisor 执行环境。两层软件之间的接口被称为 Supervisor Binary Interface(SBI)，即 Supervisor 二进制接口。 `SBI Specification <https://github.com/riscv-non-isa/riscv-sbi-doc>`_ （简称 SBI spec）规定了 SBI 接口层要包含哪些功能，该标准由 RISC-V 开源社区维护。RustSBI 按照 SBI spec 标准实现了需要支持的大多数功能，但 RustSBI 并不是 SBI 标准的唯一一种实现，除此之外还有社区中的前辈 OpenSBI 等等。

.. 目前， SBI spec 已经发布了 v2.0-rc8 版本，但本教程基于 2023 年 3 月份发布的 `v1.0.0 版本 <https://github.com/riscv-non-isa/riscv-sbi-doc/releases/download/v1.0.0/riscv-sbi.pdf>`_ 。我们可以来看看里面约定了 SEE 要向 OS 内核提供哪些功能，并寻找我们本节所需的打印到屏幕和关机的接口。可以看到从 Chapter 4 开始，每一章包含了一个 SBI 拓展（Chapter 5 包含多个 Legacy Extension），代表一类功能接口，这有点像 RISC-V 指令集的 IMAFD 等拓展。每个 SBI 拓展还包含若干子功能。其中：

.. - Chapter 5 列出了若干 SBI 遗留接口，其中包括串口的写入（正是我们本节所需要的）和读取接口，分别位于 5.2 和 5.3 小节。在教程第九章我们自己实现串口外设驱动之前，与串口的交互都是通过这两个接口来进行的。顺带一提，第三章开始还会用到 5.1 小节介绍的 set timer 接口。
.. - Chapter 10 包含了若干系统重启相关的接口，我们本节所需的关机接口也在其中。

.. 内核应该如何调用 RustSBI 提供的服务呢？通过函数调用是行不通的，因为内核并没有和 RustSBI 链接到一起，我们仅仅使用 RustSBI 构建后的可执行文件，因此内核无从得知 RustSBI 中的符号或地址。幸而， RustSBI 开源社区的 `sbi_rt <https://github.com/rustsbi/sbi-rt>`_ 封装了调用 SBI 服务的接口，我们直接使用即可。首先，我们在 ``Cargo.toml`` 中引入 sbi_rt 依赖：

.. .. code-block::
..     :linenos:

..     // os/Cargo.toml
..     [dependencies]
..     sbi-rt = { version = "0.0.2", features = ["legacy"] }

.. 这里需要带上 ``legacy`` 的 feature，因为我们需要用到的串口读写接口都属于 SBI 的遗留接口。

.. .. _term-llvm-sbicall:

.. 我们将内核与 RustSBI 通信的相关功能实现在子模块 ``sbi`` 中，因此我们需要在 ``main.rs`` 中加入 ``mod sbi`` 将该子模块加入我们的项目。在 ``os/src/sbi.rs`` 中，我们直接调用 sbi_rt 提供的接口来将输出字符：

.. .. code-block:: rust
..     :linenos:

..     // os/src/sbi.rs
..     pub fn console_putchar(c: usize) {
..         #[allow(deprecated)]
..         sbi_rt::legacy::console_putchar(c);
..     }

.. 注意我们为了简单起见并未用到 ``sbi_call`` 的返回值，有兴趣的同学可以在 SBI spec 中查阅 SBI 服务返回值的含义。到这里，同学们可以试着在 ``rust_main`` 中调用 ``console_putchar`` 来在屏幕上输出 ``OK`` 。接着在 Qemu 上运行一下，我们便可看到由我们自己输出的第一条 log 了。

.. 同样，我们再来实现关机功能：

.. .. code-block:: rust
..     :linenos:

..     // os/src/sbi.rs
..     pub fn shutdown(failure: bool) -> ! {
..         use sbi_rt::{system_reset, NoReason, Shutdown, SystemFailure};
..         if !failure {
..             system_reset(Shutdown, NoReason);
..         } else {
..             system_reset(Shutdown, SystemFailure);
..         }
..         unreachable!()
..     }

.. 这里的参数 ``failure`` 表示系统是否正常退出，这会影响 Qemu 模拟器进程退出之后的返回值，我们则会依此判断系统的执行是否正常。更多内容可以参阅 SBI spec 的 Chapter 10。

.. .. note:: **sbi_rt 是如何调用 SBI 服务的**

..     SBI spec 的 Chapter 3 介绍了服务的调用方法：只需将要调用功能的拓展 ID 和功能 ID 分别放在 ``a7`` 和 ``a6`` 寄存器中，并按照 RISC-V 调用规范将参数放置在其他寄存器中，随后执行 ``ecall`` 指令即可。这会将控制权转交给 RustSBI 并由 RustSBI 来处理请求，处理完成后会将控制权交还给内核。返回值会被保存在 ``a0`` 和 ``a1`` 寄存器中。在本书的第二章中，我们会手动编写汇编代码来实现类似的过程。

通过 UART 实现输入输出
----------------------------------------

串口（Universal Asynchronous Receiver-Transmitter，简称UART）是一种在嵌入式系统中常用的用于传输、接收系列数据的外部设备。串行数据传输是逐位（bit）顺序发送数据的过程。每个UART使用8个I/O字节来访问其寄存器。下表显示了UART中每个寄存器的地址和基本含义。表中使用的 `base` 是串口设备的起始地址。:ref:`前面 <info-mtree>` 我们已经看到在QEMU模拟的virt计算机中串口设备寄存器的MMIO起始地址为 ``0x1fe001e0`` 。

.. note:: **PIO、MMIO 和 PMIO**

   PIO指CPU通过发出I/O指令的方式来进行数据传输。PIO方式可以进一步细分为基于Memory-mapped的PIO（简称MMIO）和Port-mapped的PIO（简称PMIO），MMIO是将I/O设备物理地址映射到内存地址空间，这样CPU就可以通过普通访存指令将数据送到I/O设备在主存上的位置，从而完成数据传输。

   对于采用PMIO方式的I/O设备，它们具有自己独立的地址空间，与内存地址空间分离。CPU若要访问I/O设备，则需要使用特殊的I/O指令，如x86处理器中的 ``IN`` 、``OUT`` 指令，这样CPU直接使用I/O指令，就可以通过PMIO方式访问设备。

.. note:: 关于 UART 等 I/O 设备的管理相关内容将在第九章详细介绍，在此我们只是简单利用 UART 实现最基本的输入输出。

+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+
| I/O port  | Read (DLAB=0)                     | Write (DLAB=0)               | Read (DLAB=1)                     | Write (DLAB=1)             |
+===========+===================================+==============================+===================================+============================+
| base      | **RBR** receiver buffer           | **THR** transmitter holding  | **DLL** divisor latch LSB         | **DLL** divisor latch LSB  |
+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+
| base+1    | **IER** interrupt enable          | **IER** interrupt enable     | **DLM** divisor latch MSB         | **DLM** divisor latch MSB  |
+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+
| base+2    | **IIR** interrupt identification  | **FCR** FIFO control         | **IIR** interrupt identification  | **FCR** FIFO control       |
+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+
| base+3    | **LCR** line control              | **LCR** line control         | **LCR** line control              | **LCR** line control       |
+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+
| base+4    | **MCR** modem control             | **MCR** modem control        | **MCR** modem control             | **MCR** modem control      |
+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+
| base+5    | **LSR** line status               | *factory test*               | **LSR** line status               | *factory test*             |
+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+
| base+6    | **MSR** modem status              | *not used*                   | **MSR** modem status              | *not used*                 |
+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+
| base+7    | **SCR** scratch                   | **SCR** scratch              | **SCR** scratch                   | **SCR** scratch            |
+-----------+-----------------------------------+------------------------------+-----------------------------------+----------------------------+

.. note:: LCR 寄存器中的 DLAB 位设置为 0 或 1 ，会导致CPU访问的是不同的寄存器。比如，DLAB位为0时，读位于 `base` 处的串口寄存器是 `RBR`， DLAB位为1时，读位于 `base` 处的串口寄存器是 `DLL`。

阅读上表可以知道，要输出的字符应放在 THR 寄存器中，而 LSR 寄存器则保存了行状态。因此我们可以通过从 LSR 所映射到的内存地址读取数据得到该寄存器的值，通过不断检查对应的标志位是否有效确定此时是否能够向 THR 寄存器写入数据。当该标志位有效时，就可以向 THR 所映射到的地址写入要写入的字符来输出该字符。这样，我们就实现了单个字符的输出。而通过逐个输出其中的字符我们就可以实现字符串的输出。输出单个字符的代码我们放在 ``uart`` 模块中，其代码如下：

.. code-block:: rust
    :linenos:

    // os/src/uart.rs
    use core::ptr;

    const UART_BASE: usize = 0x1fe001e0;
    const UART0_THR: usize = UART_BASE + 0;
    const UART0_LSR: usize = UART_BASE + 5;
    const LSR_TX_IDLE: u8 = 1 << 5;

    fn io_readb(addr: usize) -> u8 {
        unsafe { ptr::read_volatile(addr as *const u8) }
    }

    fn io_writeb(addr: usize, c: u8) {
        unsafe { ptr::write_volatile(addr as *mut u8, c) };
    }

    pub fn console_putchar(c: usize) {
        let mut lsr = io_readb(UART0_LSR);
        while (lsr & LSR_TX_IDLE) == 0 {
            lsr = io_readb(UART0_LSR);
        }

        io_writeb(UART0_THR, c as u8);
    }

在上述代码中，我们通过给 UART 的 MMIO 起始地址加上对应的偏移量得到对应寄存器的 MMIO 地址从而读写该寄存器。为了实现对指定内存地址的读写，我们使用了 ``core::ptr`` 模块中的 ``read_volatile`` 和 ``write_volatile`` 函数，它们能够实现对指定内存地址的原子读写。由于涉及到使用裸指针对内存地址的直接读写，所以上述代码需要放在 ``unsafe`` 块中。在 ``console_putchar`` 函数中，我们通过 CPU 反复查询寄存器状态的方式来等待输出设备就绪，这实际上是一种忙等，效率非常低下，我们在第九章会进行改进，这里我们只用最简单的方式实现输出功能即可。

实现格式化输出
-----------------------------------------------

``console_putchar`` 的功能过于受限，如果想打印一行 ``Hello world!`` 的话需要进行多次调用。能否像本章第一节那样使用 ``println!`` 宏一行就完成输出呢？因此我们尝试自己编写基于 ``console_putchar`` 的 ``println!`` 宏。

.. code-block:: rust
    :linenos:

    // os/src/main.rs
    #[macro_use]
    mod console;

    // os/src/console.rs
    use crate::sbi::console_putchar;
    use core::fmt::{self, Write};

    struct Stdout;

    impl Write for Stdout {
        fn write_str(&mut self, s: &str) -> fmt::Result {
            for c in s.chars() {
                console_putchar(c as usize);
            }
            Ok(())
        }
    }

    pub fn print(args: fmt::Arguments) {
        Stdout.write_fmt(args).unwrap();
    }

    #[macro_export]
    macro_rules! print {
        ($fmt: literal $(, $($arg: tt)+)?) => {
            $crate::console::print(format_args!($fmt $(, $($arg)+)?));
        }
    }

    #[macro_export]
    macro_rules! println {
        ($fmt: literal $(, $($arg: tt)+)?) => {
            $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
        }
    }

我们在 ``console`` 子模块中编写 ``println!`` 宏。结构体 ``Stdout`` 不包含任何字段，因此它被称为类单元结构体（Unit-like structs，请参考 [#unit-like-structs]_ ）。 ``core::fmt::Write`` trait 包含一个用来实现 ``println!`` 宏很好用的 ``write_fmt`` 方法，为此我们准备为结构体 ``Stdout`` 实现 ``Write`` trait 。在 ``Write`` trait 中， ``write_str`` 方法必须实现，因此我们需要为 ``Stdout`` 实现这一方法，它并不难实现，只需遍历传入的 ``&str`` 中的每个字符并调用 ``console_putchar`` 就能将传入的整个字符串打印到屏幕上。

在此之后 ``Stdout`` 便可调用 ``Write`` trait 提供的 ``write_fmt`` 方法并进而实现 ``print`` 函数。在声明宏（Declarative macros，参考 [#declarative-macros]_ ） ``print!`` 和 ``println!`` 中会调用 ``print`` 函数完成输出。

现在我们可以在 ``rust_main`` 中使用 ``print!`` 和 ``println!`` 宏进行格式化输出了，如有兴趣的话可以输出 ``Hello, world!`` 试一下。

.. note::

    **Rust Tips：Rust Trait**

    在 Rust 语言中，trait（中文翻译：特质、特征）是一种类型，用于描述一组方法的集合。trait 可以用来定义接口（interface），并可以被其他类型实现。
    举个例子，假设我们有一个简单的Rust程序，其中有一个名为 Shape 的 trait，用于描述形状：

    .. code-block:: rust
        :linenos:

        trait Shape {
            fn area(&self) -> f64;
        }


    我们可以使用这个 trait 来定义一个圆形类型：

    .. code-block:: rust
        :linenos:

        struct Circle {
            radius: f64,
        }

        impl Shape for Circle {
            fn area(&self) -> f64 {
                3.14 * self.radius * self.radius
            }
        }

    这样，我们就可以使用 Circle 类型的实例调用 area 方法了。

    .. code-block:: rust
        :linenos:

        let c = Circle { radius: 1.0 };
        println!("Circle area: {}", c.area());  // 输出: Circle area: 3.14    



处理致命错误
-----------------------------------------------

错误处理是编程的重要一环，它能够保证程序的可靠性和可用性，使得程序能够从容应对更多突发状况而不至于过早崩溃。不同于 C 的返回错误编号 ``errno`` 模型和 C++/Java 的 ``try-catch`` 异常捕获模型，Rust 将错误分为可恢复和不可恢复错误两大类。这里我们主要关心不可恢复错误。和 C++/Java 中一个异常被抛出后始终得不到处理一样，在 Rust 中遇到不可恢复错误，程序会直接报错退出。例如，使用 ``panic!`` 宏便会直接触发一个不可恢复错误并使程序退出。不过在我们的内核中，目前不可恢复错误的处理机制还不完善：

.. code-block:: rust
    :linenos:

    // os/src/lang_items.rs
    use core::panic::PanicInfo;

    #[panic_handler]
    fn panic(_info: &PanicInfo) -> ! {
        loop {}
    }

可以看到，在目前的实现中，当遇到不可恢复错误的时候，被标记为语义项 ``#[panic_handler]`` 的 ``panic`` 函数将会被调用，然而其中只是一个死循环，会使得计算机卡在这里。借助前面实现的 ``println!`` 宏，我们可以在 ``panic`` 函数中打印错误信息：

.. code-block:: rust
    :linenos:

    // os/src/main.rs
    #![feature(panic_info_message)]

    // os/src/lang_item.rs
    use crate::sbi::shutdown;
    use core::panic::PanicInfo;

    #[panic_handler]
    fn panic(info: &PanicInfo) -> ! {
        if let Some(location) = info.location() {
            println!(
                "Panicked at {}:{} {}",
                location.file(),
                location.line(),
                info.message().unwrap()
            );
        } else {
            println!("Panicked: {}", info.message().unwrap());
        }
        loop {}
    }

我们尝试打印更加详细的信息，包括 panic 所在的源文件和代码行数。我们尝试从传入的 ``PanicInfo`` 中解析这些信息，如果解析成功的话，就和 panic 的报错信息一起打印出来。我们需要在 ``main.rs`` 开头加上 ``#![feature(panic_info_message)]`` 才能通过 ``PanicInfo::message`` 获取报错信息。

为了测试我们的实现是否正确，我们将 ``rust_main`` 改为：

.. code-block:: rust
    :linenos:

    // os/src/main.rs
    #[no_mangle]
    pub fn rust_main() -> ! {
        clear_bss();
        println!("Hello, world!");
        panic!("Shutdown machine!");
    }

使用 Qemu 运行我们的内核，运行结果为：

.. code-block::

    ......
    
    entry kernel ...
    Hello, world!
    Panicked at src/main.rs:26 Shutdown machine!

可以看到，panic 所在的源文件和代码行数被正确报告，这将为我们后续章节的开发和调试带来很大方便。到这里，我们就实现了一个可以在Qemu模拟的计算机上运行的裸机应用程序，其具体内容就是上述的 `rust_main` 函数，而其他部分，如 `entry.asm` 、 `lang_items.rs` 、`console.rs` 、 `sbi.rs` 则形成了支持裸机应用程序的寒武纪“三叶虫”操作系统 -- LibOS 。

.. note::

    **Rust Tips：Rust 可恢复错误**

    在有可能出现错误时，Rust 函数的返回值可以属于一种特殊的类型，该类型可以涵盖两种情况：要么函数正常退出，则函数返回正常的返回值；要么函数执行过程中出错，则函数返回出错的类型。Rust 的类型系统保证这种返回值不会在程序员无意识的情况下被滥用，即程序员必须显式对其进行分支判断或者强制排除出错的情况。如果不进行任何处理，那么无法从中得到有意义的结果供后续使用或是无法通过编译。这样，就杜绝了很大一部分因程序员的疏忽产生的错误（如不加判断地使用某函数返回的空指针）。

    在 Rust 中有两种这样的特殊类型，它们都属于枚举结构：

    - ``Option<T>`` 既可以有值 ``Option::Some<T>`` ，也有可能没有值 ``Option::None``；
    - ``Result<T, E>`` 既可以保存某个操作的返回值 ``Result::Ok<T>`` ，也可以表明操作过程中出现了错误 ``Result::Err<E>`` 。

    我们可以使用 ``Option/Result`` 来保存一个不能确定存在/不存在或是成功/失败的值。之后可以通过匹配 ``if let`` 或是在能够确定
    的场合直接通过 ``unwrap`` 将里面的值取出。详细的内容可以参考 Rust 官方文档 [#recoverable-errors]_ 。


.. [#unit-like-structs] https://doc.rust-lang.org/book/ch05-01-defining-structs.html#unit-like-structs-without-any-fields
.. [#declarative-macros] https://doc.rust-lang.org/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming
.. [#recoverable-errors] https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html