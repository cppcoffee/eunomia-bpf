# talk in yunxi

## 1/2、开始

大家好！我是来自浙江大学的郑昱笙，今天为大家带来的演讲主题是：eunomia-bpf：一个 eBPF 轻量级开发框架，希望和大家分享一下 eunomia-bpf 项目作为一个为了简化 eBPF 程序的开发、分发、运行而设计的轻量级 eBPF 开发框架的背景和目标，然后通过一些简单的实例，展示一下 eunomia-bpf 是如何从云端一行命令下载运行 eBPF 程序、只编写内核态代码即可运行和导出事件，以及和 WebAssembly 的结合等功能，最后简要阐述一下 eunomia-bpf 的原理和设计实现的思路，探讨一下接下来的发展方向。

## 3、背景概况

eunomia-bpf 起源于今年七八月份 2022 年全国大学生操作系统大赛的一个 idea，希望将 eBPF 程序作为服务运行，把 eBPF 程序打包为一个 JSON 对象，通过 HTTP 请求即可动态插拔运行任意一个可重定位的 eBPF 程序，并且可以适应不同内核版本和架构。比赛结束之后，在高校的几位老师和社区中的一些伙伴的帮助和指导下，尤其是要感谢西安邮电大学陈莉君老师团队和龙蜥社区的毛文安老师，逐步把这些想法变成了一个初具雏形的开源项目，目前除了我之外，也主要是陈莉君老师那边的一些团队，和一些社区的朋友在一起协作。

## 4、设计目标

eunomia-bpf 想要解决的问题，或者说个人理解的 eBPF 程序的痛点主要有两个：

1. 一方面，对于新手而言，搭建和开发 ebpf 程序是一个门槛比较高、比较复杂的工作，必须同时关注内核态和用户态两个方面的交互和信息处理，还需要编写不少的用户态加载代码
2. 另外一方面，如何跨架构和内核版本，方便又快捷的打包、分发、发布 eBPF 程序也是一个挑战，之前在陈老师的 LMP 项目那边，我们就发现使用或者集成个人开发者编写的各种各样不同类型的 ebpf 程序是一个很困难工作，它们可能使用多种用户态语言开发（go，rust，c/cpp 等等），有各种各样不同的接口，也没办法很轻松以插件的方式集成到大型的可观测性或其他系统中，一般必须修改代码并重新编译整个可观测性的框架然后部署上线，才能更新其中的 bpf 探针，同时如果引入第三方代码且没有很好的做代码审查的话，也可能会引入额外的安全隐患。

针对上面两个问题，我们分别有两个方向的解决思路：

1. 第一个方面，对于初学者或者不需要 wasm 用户态运行时的简单应用来说，比如一个纯粹的 ebpf 探针，可以只编写内核态 ebpf 代码，也可以在源代码中通过一些类似 SEC 或者 doxygen 那种类型的注释的方式，添加一些额外的辅助加载信息，例如 uprobe 具体的 attach point，编写内核代码后即可自动完成 ebpf 程序的动态加载，和自动获取内核态导出数据；这样可以降低 eBPF 程序的学习成本，同时提高开发效率。

2. 第二个方面，基于 libbpf 信息一次编译，到处运行的特性， ebpf 编译和加载的过程完全分离，并且可以通过标准化的 JSON 或 wasm 模块的方式进行分发，在移植部署的时候不需要类似 bcc 一样重新编译，ebpf 应用启动占用资源也非常少；

> WebAssembly(缩写 Wasm)是基于堆栈虚拟机的二进制指令格式。Wasm 是为了一个可移植的目标而设计的，可作为 C/C+/RUST 等高级语言的编译目标，使客户端和服务器应用程序能够在 Web 上部署。到现在为止，Wasm 已经发展成为一个轻量级、高性能、跨平台和多语种的软件沙盒环境，被运用于云原生软件组件，可以在非浏览器环境下运行。wasm 的设计思路和 ebpf 也有不少相似之处。

3. 只编写内核态代码的时候，使用 JSON 即可完成分发、加载、打包的过程，对于完整的、需要用户态和内核态进行交互的 ebpf 应用或工具，可以在 wasm 中编写复杂的用户态处理程序进行控制和处理，并且将编译好的 ebpf 字节码嵌入在 wasm 模块中一同分发，在目标机器上动态加载运行；

> 和 Wasm 生态项结合可以给 eBPF 程序带来许多特性，同时和 eBPF 程序原本的设计思路也不谋而合，比如可移植、隔离性、安全性，它也是一个跨语言、轻量级的运行环境等等；同时也可以借助 wasm 的相关工具完成 ebpf 程序的 OCI 镜像的存储和分发，最近 docker 官方也推出了一个基于 wasm 的分发工具。

以上三个部分，就是 eunomia-bpf 的核心特性。现在和大家一起来看几个例子。

## 5、从网页端下载预编译 eBPF 程序一键运行的过程

这里我们使用的是一个简单的 eBPF 程序，它可以获取当前系统的进程间的 signal 信号传递的事件。就像图上显示的那样，下载对应的二进制命令行工具之后，

1. 一行命令即可从云端运行任意最新版本的 eBPF 程序
2. 使用 WebAssembly 模块或 JSON 对象进行标准化 eBPF 程序的分发
3. 一次编译到处运行：部署时不需要重新编译

之前这个 PPT 上演示的其实是把它放在一个静态文件的 url 里面，已经有一点过时了，接下来我们可以换成使用 wasm 的 OCI 镜像来进行分发，就像 docker 推出的那个 wasm 的工具一样，可以带来类似于 docker 的体验，进行一个镜像的 push 或 pull。

## 6、编写代码的 Hello World

这里演示的只是一个简单的版本，具体复杂的例子，包含如何使用注解，如何从内核态往用户态导出信息，可以参考这个项目的示例部分。

## 7、WebAssembly

一般来说，一个完整的 eBPF 应用程序分为用户空间程序和内核程序两部分，用户空间程序负责加载 BPF 字节码至内核，或负责读取内核回传的统计信息或者事件详情，进行相关的数据处理和控制。

我们可以在 Wasm 中编写用户态辅助程序，来完成安全、高效的用户态数据处理和控制逻辑，它同样具备 eBPF 的特性，例如安全性（wasm 和 ebpf 一样也是个沙盒环境，在用户态运行的时候即使 wasm 模块崩溃了，也不会造成宿主程序的异常退出）、可移植性、轻量级、模块化等等，也可以作为插件使用，添加新的数据处理逻辑时，也不需要更改原本的代码。注意 wasm 是可选而不是必须的，对于一些简单的应用而言，编写内核态代码就足够了。

实际上，我们是用 C 语言编写代码，然后打包生成 Wasm 模块。之后我们可以：

- 借助 WebAssembly 的相关生态帮助分发、管理 eBPF 程序，例如 docker-wasm
- 可嵌入大型应用中作为 eBPF 可编程模块或插件使用

这里演示的是一个简单的 wasm 模块，它可以获取当前系统的进程间的 signal 信号传递的事件。它可以接受一些命令行参数，并且对上报的信息进行处理。

这个图是之前截的，目前来看，我们已经可以基本上不用进行代码修改，就可以直接把 BCC/libbpf-tools 里面的程序编译为 wasm 模块。开发体验来说，也可以做到和使用 C 语言开发 libbpf 的 eBPF 程序完全相同，之后也可以引入别的语言的开发 SDK。

把 wasm 和 ebpf 结合起来主要的困难在于，Wasm 的内存布局和 eBPF 程序并不一样，C 语言的结构体并不能直接映射，所以传递结构体必须要经过序列化操作；同时，Wasm 对于访问系统资源，例如文件、网络等等，也有不少限制，很多标准库是缺失的，所以我们需要在 wasm 模块中进行一些特殊的处理和移植。

## 8、eunomia-bpf 项目现在的组成部分

（从下往上）

依赖的部分：

- 内核的基础设施：ebpf 虚拟机
- 用户态的基础设施：libbpf 库

除了这两个东西以外，运行 eunomia-bpf 的时候就没有额外的依赖了。eunomia-bpf 的核心部分主要有两个:

- 一个 ebpf 动态加载运行时库，用来根据 JSON 信息往内核态动态加载代码，它和 wasm 无关，只是一个通用的 ebpf 动态加载库，也可以独立使用；
- 开发工具链，用来编译生成 JSON 信息，以及编译打包 wasm 模块等等。

在这两个部分之上，我们构建了和 wasm 相关的一些工具，例如：

- 一些对应的 API 规范，用于在 wasm 模块中扩展 WASI，来加载 eBPF 程序；
- 一个基于 wasm 定制的 libbpf 库，和一些移植的辅助程序，比如命令行解析器和序列化库，用于在 wasm 模块中加载基于 libbpf 的 eBPF 程序；
- 一个示例的运行时库，用于在 wasm 模块中加载 eBPF 程序；你也可以轻松地换成其他实现了 wasi 的 wasm 运行时，例如 wasmEdge，在其他大型项目中作为插件运行时使用。

在这些之外，还有一些其他的工具，例如：

- 一个使用 Rust 编写的可观测性数据收集器，通过 HTTP 请求即可动态插拔运行任意一个 eBPF 探针；
- 一个命令行工具，用来下载运行 eBPF 程序；
- 还有和陈莉君老师的 LMP 项目那边合作，提供更多的 eBPF 程序实例，以及一个包管理器；

## 9、eunomia-bpf 项目的编译运行流程

## 12、总结陈述

今天就为大家分享到这里，感谢大家聆听和关注，我们的项目也还是在一个刚起步的阶段，非常希望能得到各位老师和专家们的建议和反馈，谢谢！