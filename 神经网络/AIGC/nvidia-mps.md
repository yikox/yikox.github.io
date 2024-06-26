# Multi-Process Service (MPS)
> 基于[mps](https://docs.nvidia.com/deploy/mps/index.html#topic_2)文档翻译
## 1、简介
Multi-Process Service (MPS) is an alternative, binary-compatible implementation of the CUDA Application Programming Interface (API). 
> MPS是CUDA应用程序编程接口（API）的替代二进制兼容实现。
The MPS runtime architecture is designed to transparently enable co-operative multi-process CUDA applications, typically MPI jobs, to utilize Hyper-Q capabilities on the latest NVIDIA (Kepler and later) GPUs. Hyper-Q allows CUDA kernels to be processed concurrently on the same GPU;
> MPS运行时架构旨在透明地使合作多进程CUDA应用程序（通常是MPI作业）能够在最新的NVIDIA（Kepler及更高版本）图形处理器上利用Hyper-Q功能。Hyper-Q允许在同一个图形处理器上并发处理CUDA内核;
this can benefit performance when the GPU compute capacity is underutilized by a single application process.
> 当单个应用程序进程利用不足时，这可以提高性能。

### 1.1.2、Volta MPS
Volta 架构 引入新的MPS功能，与Volta之前版本的MPS相比，Volta MPS提供了一些关键改进：
- Volta MPS客户端无需通过MPS服务器直接将工作提交到图形处理器。
- 每个Volta MPS客户端都拥有自己的图形处理器地址空间，而不是与所有其他MPS客户端共享图形处理器地址空间。
- Volta MPS 支持为服务质量（QoS）限制有限的执行资源配置
本文档将介绍新功能，并指出Volta MPS和Volta之前的图形处理器上MPS之间的差异。在Volta上运行MPS将自动启用新功能。
## 1.3、概念
### 1.3.2、MPS 是什么
MPS是CUDA API的二进制兼容客户端-服务器运行时实现，由多个组件组成。
- Control Daemon Process - 控制守护进程负责启动和停止服务器，以及协调客户端和服务器之间的连接。
- Client Runtime - MPS客户端运行时内置在CUDA驱动程序库中，任何CUDA应用程序都可以透明地使用。
- Server Process  - 服务器是客户端到图形处理器的共享连接，并提供客户端之间的并发性。

## 2、何时使用 MPS

### 2.1、使用 MPS 的好处
- GPU利用率
单个进程可能无法利用图形处理器上可用的所有计算和内存带宽容量。MPS允许来自不同进程的内核和memCopy操作在图形处理器上重叠，从而实现更高的利用率和更短的运行时间。
- 减少了图形处理器上的上下文存储
在没有MPS的情况下，每个使用图形处理器的CUDA进程都会在图形处理器上分配单独的存储和调度资源。相比之下，MPS服务器分配一份由其所有客户端共享的图形处理器存储和调度资源副本。Volta MPS支持MPS客户端之间的隔离，因此资源减少的程度要小得多。
- 减少了图形处理器上下文切换
如果没有MPS，当进程共享图形处理器时，它们的调度资源必须在图形处理器中交换和交换。MPS服务器在其所有客户端之间共享一组调度资源，从而消除了当图形处理器在这些客户端之间进行调度时交换的负担。

### 2.2、什么程序合适使用 mps
当每个应用程序进程不会产生足够的工作来使GPU饱和时，MPS非常有用。使用MPS，每个节点可以运行多个进程，以实现更多的并发。这样的应用程序通过每个网格具有少量块来标识。
此外，如果应用程序由于每个网格的线程数量较少而导致GPU占用率较低，则可以使用MP来实现性能改进。建议在内核调用中使用较少的每个网格的块和更多的每个块的线程来增加每个块的占用。MPS允许从其他进程运行的CUDA内核占用剩余的GPU容量。
这些情况发生在可伸缩性很强的情况下，计算容量(节点、CPU核心和/或GPU计数)增加，而问题大小保持不变。尽管计算工作总量保持不变，但每个进程的工作减少，并且可能在应用程序运行时未充分利用可用计算能力。有了MPS，GPU将允许从不同进程启动的内核同时运行，并从计算中删除不必要的串行化点。

### 2.3、考虑事项
#### 限制
- 仅Linux操作系统支持MPS。当在非Linux操作系统上启动时，MPS服务器无法启动。
- Tegra平台不支持MPS。在Tegra平台上启动时，MPS服务器将无法启动。
- MPS需要具备3.5或更高计算能力的GPU。如果应用CUDA_Visible_Device后可见的其中一个GPU不具有3.5或更高的计算能力，则MPS服务器将无法启动。
- CUDA的统一虚拟寻址(UVA)功能必须可用，这是在具有计算功能2.0版或更高版本的GPU上运行的任何64位CUDA程序的默认设置。如果UVA不可用，MPS服务器将无法启动。
- MPS客户端可以分配的分页锁定主机内存量受tmpfs文件系统(/dev/shm)大小的限制。
- 独占模式限制应用于MPS服务器，而不是MPS客户端。
- 一个系统上只有一个用户可以具有活动的MPS服务器。
- MPS控制守护程序将来自不同用户的MPS服务器激活请求排队，从而导致用户之间对GPU的串行化独占访问，而不考虑GPU独占设置。
- 所有MPS客户端行为都将通过系统监控和记账工具(例如NVIDIA-SMI、NVML API)归因于MPS服务器进程。

## 图形处理器计算模式
通过nvidia-smi中访问的设置支持三种计算模式。
- `PROHIBITED` - 图形处理器不适用于计算应用程序。
- `EXCLUSSIVE_PROCESS` - 图形处理器一次仅分配给一个进程，各个进程线程可以同时向图形处理器提交工作。
- `DEFAULT` - 多个进程可以同时使用图形处理器。每个进程的各个线程可以同时将工作提交到图形处理器。
使用 mps 使得 `EXCLUSSIVE_PROCESS` 像 `DEFAULT` 一样对于多个 mps 客户端。MPS 允许多个客户端通过MPS服务器使用图形处理器。
使用MPS时，建议使用 `EXCLUSIVE_PROCESS` 模式以确保只有一个MPS服务器正在使用图形处理器，这提供了额外的保证，即MPS服务器是该图形处理器的所有CUDA进程之间的单点仲裁。

## 应用考虑
- 在Pre-Volta MPS客户端的MPS下不支持NVIDIA Codec SDK：https://developer.nvidia.com/nvidia-video-codec-sdk。
- 仅支持64位应用程序。如果CUDA应用不是64位，则MPS服务器无法启动。MPS客户端CUDA初始化失败。
- 如果应用程序使用CUDA驱动程序API，则它必须使用CUDA 4.0或更高版本中的标头(即，它不能是通过将CUDA_FORCE_API_VERSION设置为较早版本来构建的)。如果上下文版本早于4.0，则客户端中的上下文创建将失败。
- 不支持动态并行。如果模块使用动态并行功能，则CUDA模块加载将失败。
- MPS服务器仅支持使用与服务器相同的UID运行的客户端。如果服务器未使用相同的UID运行，则客户端应用程序将无法初始化。
- Pre-Volta MPS客户端不支持流回调。调用任何流回调接口都会返回错误。
- 在Pre-Volta MPS客户端上的MPS下不支持带有主机节点的CUDA图。
- Pre-Volta MPS客户端应用程序可以分配的页面锁定主机内存量受tmpfs文件系统(/dev/shm)大小的限制。尝试使用任何相关的CUDA API分配比允许大小更多的页面锁定内存将失败。
- 在不与所有未完成的GPU工作同步的情况下终止MPS客户端(通过Ctrl-C/程序异常，如段故障/信号等)可能会使MPS服务器和其他MPS客户端处于未定义状态，这可能会导致挂起、意外故障或损坏。
- 在Volta MPS下，支持由作为MPS客户端运行的进程创建的CUDA上下文和由不作为MPS客户端运行的进程创建的CUDA上下文之间的CUDA IPC。

## 内存保护和错误遏制
仅建议MPS用于有效地作为单个应用程序运行协作流程，例如同一MPI作业的多个队列，以便以下内存保护和错误遏制限制的严重性是可以接受的。
### 存储器保护
- Volta MPS客户端进程具有完全隔离的图形处理器地址空间。
- Pre-Volta MPS客户端进程从同一图形处理器虚拟地址空间的不同分区分配内存。结果：
    - CUDA内核中的超范围写入可以修改另一个进程的CUDA可访问内存状态，并且不会触发错误。
    - CUDA内核中的超范围读取可以访问由另一个进程修改的CUDA可访问内存，并且不会触发错误，从而导致未定义的行为。
Volta MPS之前的这种行为受到来自CUDA内核内指针的内存访问的限制。任何CUDA API都会限制MPS客户端访问该MPS客户端内存分区之外的任何资源。例如，无法使用cudaMemcpy（）API覆盖另一个MPS客户端的内存。
### 错误抑制
Volta MPS支持有限的错误遏制：
- 由Volta MPS客户端进程生成的致命GPU故障将包含在所有客户端与导致致命故障的GPU之间共享的GPU子集内。
- 由Volta MPS客户端进程生成的致命GPU故障将报告给在包含致命故障的GPU子集上运行的所有客户端，但不会指明哪个客户端生成了该错误。请注意，在收到致命的GPU故障通知后，受影响的客户端有责任退出。
- 在其他GPU上运行的客户端保持不受致命故障的影响，并将照常运行，直到完成。
- 一旦观察到致命故障，MPS服务器将等待与受影响的GPU相关联的所有客户端退出，从而禁止连接到这些GPU的新客户端加入。MPS服务器的状态从‘ACTIVE’变为‘FAULT’。当与受影响的GPU相关联的所有现有客户端都已退出时，MPS服务器将在受影响的GPU上重新创建GPU上下文，并恢复处理对这些GPU的客户端请求。MPS服务器的状态变回‘ACTIVE’，表明它能够处理新的客户端。
> 例如，如果您的系统具有设备0、1和2，并且如果有四个客户端客户端A、客户端B、客户端C和客户端D连接到MPS服务器：客户端A在设备0上运行，客户端B在设备0和1上运行，客户端C在设备1上运行，客户端D在设备2上运行。如果客户端A触发致命的图形处理器故障：
由于设备0和设备1共享注释客户端客户端B，因此致命的GPU故障包含在设备0和1中。
致命的GPU故障将报告给设备0和1上运行的所有客户端，即客户端A、客户端B和客户端C。
在设备2上运行的客户端D保持不受致命故障的影响，并继续正常运行。
MPS服务器将等待客户端A、客户端B和客户端C退出并拒绝任何新的客户端请求，并在服务器状态为‘FAULT’时拒绝错误CUDA_ERROR_MPS_SERVER_NOT_READY。在客户端A、客户端B和客户端C退出后，服务器在设备0和设备1上重新创建GPU上下文，然后恢复在所有设备上接受客户端请求。服务器状态再次变为“Active”。
将记录有关致命GPU故障包含的信息，包括：
如果致命的GPU故障是致命的内存故障，则是触发致命的GPU内存故障的客户端的PID。
受此致命GPU故障影响的设备的设备ID。
受此致命GPU故障影响的客户端的ID。每个受影响的客户端的状态变为‘非活动’，并且MPS服务器的状态变为‘故障’。
这些消息指示在所有受影响的客户端退出后已成功重新创建受影响的设备。

Pre-Volta MPS客户端进程共享图形处理器上调度和错误报告资源。结果：
- 任何客户端生成的图形处理器错误都会报告给所有客户端，而不会指示是哪个客户端生成了错误。
- 由一个客户端触发的致命图形处理器故障将终止MPS服务器和所有客户端的图形处理器活动。
在CUDA收件箱或CUDA驱动程序中的中央处理器上生成的CUDA API错误仅传递给调用客户端。

## MPS on Multi-GPU Systems
MPS服务器支持使用多个图形处理器。在具有多个图形处理器的系统上，您可以使用`CUDA_VISIBLE_DEVICES`来列举您要使用的图形处理器。更多详细信息，请参阅第4.2节。
在混合使用Volta / Pre-Volta图形处理器的系统上，如果MPS服务器设置为列举任何Volta图形处理器，则它将丢弃所有Pre-Volta图形处理器。换句话说，MPS服务器要么仅在Volta图形处理器上运行并暴露Volta功能，要么仅在Volta之前的图形处理器上运行。

## 性能
### 客户端 — 服务器 连接限制
Volta MPS服务器之前的每个设备同时支持最多16个客户端CUDA上下文。Volta MPS服务器支持每个设备48个客户端CUDA上下文。这些上下文可能分布在多个进程上。如果超过连接限制，CUDA应用程序将无法创建CUDA上下文，并从cuCtxCreate（）或触发上下文创建的第一个CUDA SEARCH API调用返回API错误。MPS服务器将记录失败的连接尝试。
### Volta MPS执行资源配置
Volta MPS支持有限的执行资源配置。客户端上下文可以设置为仅使用可用线程的一部分。配置功能通常用于实现两个目标：
- 减少客户端内存占用：由于每个MPS客户端进程都具有完全隔离的地址空间，因此每个客户端上下文都会分配独立的上下文存储和调度资源。这些资源随着客户端可用的线程数量而扩展。默认情况下，每个MPS客户端都有所有可用的线程。由于MPS通常与同时运行的多个进程一起使用，因此让每个客户端都可以访问所有线程通常是不必要的，因此分配完整的上下文存储是浪费的。减少可用线程数量将有效减少上下文存储分配大小。
- 提高QoS：供应机制可以用作经典的QoS机制来限制可用的计算带宽。减少可用线程的比例还将集中客户提交给一组SM的工作，从而减少对其他客户提交的工作的破坏性干扰。
设置限制不会为任何MPS客户端上下文保留专用资源。它只是限制客户端上下文可以使用的资源量。从不同MPS客户端上下文启动的内核可以在同一SM上执行，具体取决于负载平衡。
默认情况下，每个客户端都被设置为可以访问所有可用线程。这将允许最大程度的调度自由度，但由于执行资源分配浪费而导致内存占用增加。每个客户端进程的内存使用情况可以通过nvidia-smi查询。
可以通过几种不同的机制针对不同的效果设置准备金限制。这些机制分为两种机制：活动线程百分比和编程接口。特别是，通过活动线程百分比进行分区分为两种策略：均匀分区和非均匀分区。
受统一活动线程百分比约束的限制是在客户端进程启动时为其配置的，并且之后无法为客户端进程更改该限制。已执行的限制通过设备属性cudaDevAttrMultiProcess计数反映，该属性的值在整个客户端进程中保持不变。
- MPS控制实用程序提供2组命令来设置/查询所有未来MPS客户端的限制。更多详细信息，请参阅第4.1.1节。
- 或者，可以通过为MPS控制流程设置环境变量CUDA_MPS_Active_THRAD_PERCENTTAGE来设置所有未来MPS客户端的限制。有关更多详细信息，请参阅4.2.5.1部分。
- 通过仅为客户端进程设置环境变量CUDA_MPS_Active_THRAD_PERCENTAGE，可以进一步限制新客户端的限制。有关更多详细信息，请参阅4.2.5.2部分。
受非统一活动线程百分比约束的限制是为每个客户端CUDA上下文配置的，并且可以在整个客户端进程中更改。已执行的限制通过设备属性cudaDevAttrMultiProcess计数反映，该属性的值返回调用线程当前的客户端CUDA上下文可以使用的可用线程部分。
- 通过结合环境变量 CUDA_MPS_Active_THRAD_PERCENTAGE 和环境变量 CUDA_MPS_ENABLE_PER_CTX_DEVICE_MULTIPROCESSOR_PARTITIONING ，可以进一步限制新客户端CUDA上下文的受统一分区机制约束的限制。有关更多详细信息，请参阅4.2.5.3和4.2.6部分。
由编程分区约束的限制是为通过cuCtxCreate_v3()创建的客户端CUDA上下文配置的，该客户端CUDA上下文具有执行亲和性CUexecAffinityParam，它指定上下文被限制使用的SMS的数量。可以通过cuCtxGetExecAffity()查询上下文的执行限制。有关详细信息，请参见第5.2节。
一种常见的供应策略是将可用线程均匀地划分给每个MPS客户端进程(即，对于N个预期的MPS客户端进程，将活动线程百分比设置为100%/n)。此策略将分配接近最小执行资源量，但它可能会限制偶尔使用空闲资源的客户端的性能。
一种更优的策略是将部分按预期客户端数的一半进行统一分区(即将活动线程百分比设置为100%/0.5n)，以便负载均衡器在有空闲资源时更自由地在客户端之间重叠执行。
接近最优的配置策略是根据每个MPS客户端的负载对可用的线程进行非均匀分区(即，如果客户端1和客户端2的负载比例为30%：70%，则将客户端1的活动线程百分比设置为30%，将客户端2的活动线程百分比设置为70%)。该策略将不同客户提交的工作集中到互不相交的短信集合中，有效地减少不同客户提交工作之间的干扰。
最优的供应策略是在知道每个客户端的执行资源需求的情况下，精确地限制每个MPS客户端使用的短消息的数量(即，在具有84个短消息的设备上，用于客户端1的24个短消息和用于客户端2的60个短消息)。与活动线程百分比相比，此策略提供了对工作将在其上运行的SM集的更细粒度和更灵活的控制。
如果使用活动线程百分比进行分区，则该限制将在内部四舍五入到最接近的硬件支持的线程数限制。如果使用编程接口进行分区，则该限制将在内部四舍五入到最接近的硬件支持的SM计数限制。

### 线程和Linux调度
在Volta之前的图形处理器上，启动的MPS客户端数量超过了计算机上可用的逻辑核心，将导致启动延迟增加，并且通常会由于线程如何被Linux CPS（完全公平的处理器）调度而减慢客户端与服务器的通信。对于使用多个图形处理器并使用MPS控制守护程序和每个图形处理器启动的服务器的设置，我们建议将每个MPS服务器固定到不同的核心。这可以通过使用实用程序“taskset”来实现，该实用程序允许将正在运行的程序绑定到多个核心或在它们上启动新的程序。要使用MPS实现此目标，请启动绑定到特定核心的控制守护程序，例如，' taskset -c 0 nvidia-cuda-mps-control -d '。MPS服务器启动时将继承进程相关性。

### Volta MPS设备内存限制
在Volta MPS上，用户可以强制客户端遵守分配设备内存的预设限制。此机制提供了在特定GPU上运行的MPS客户端上细分GPU内存的工具，这使调度和部署系统能够根据客户端的内存使用情况做出决策。如果客户端尝试分配超过预设限制的内存，则CUDA内存分配调用将返回内存不足错误。特定的内存限制还将考虑CUDA内部设备分配，这将帮助用户做出调度决策，以实现最佳的GPU利用率。这可以通过用于用户限制MPS客户端上的固定设备存储器的控制机制的层级来实现。默认的限制设置将在产生的所有未来MPS服务器的所有MPS客户端上强制实施设备内存限制。每台服务器的限制设置允许对内存资源限制进行更精细的控制，从而用户可以选择使用服务器ID以及服务器的所有客户端来有选择地设置内存限制。此外，MPS客户端可以使用CUDA_MPS_PINNED_DEVICE_MEM_LIMIT环境变量进一步约束服务器的内存限制设置。

## Interaction with Tools

### 调试与CUDA-GDB
在Volta MPS上，可以使用CUDA-GDB生成和调试GPU核心转储。有关使用说明，请参阅CUDA-GDB文档。
在某些情况下，从CUDA-GDB(或任何与CUDA兼容的调试器，如Allinea DDT)中调用的应用程序可以在不使用MPS的情况下自动运行，即使当MPS自动配置处于活动状态时也是如此。为了利用这种自动回退，此时可能没有其他MPS客户端应用程序在运行。这样，无需修改系统的MPS配置即可调试CUDA应用程序。
以下是它的工作原理：
CUDA-GDB尝试运行应用程序，并识别该应用程序将成为MPS客户端。
在cuda-gdb下运行的应用程序在cuInit()中阻塞，并等待所有活动的MPS客户端进程退出(如果有正在运行的进程)。
一旦所有客户端进程终止，MPS服务器将允许CUDA-GDB和正在调试的应用程序继续。
任何尝试连接到MPS守护程序的新客户端进程都将正常配置服务器。
### Cuda-Memcheck
MPS支持Cuda-Memcheck工具。有关使用说明，请参阅Cuda-Memcheck文档。
### Profiling
MPS支持CUDA性能分析工具(如nvprof和nvidia视觉性能分析器)和基于CUPTI的性能分析器。有关使用说明，请参阅探查器文档。

### Client 提前终止
不支持通过CTRL-C或Signals终止MPS客户端，这将导致未定义的行为。在可以终止MPS客户端之前，用户必须通过在所有流上调用cudaDeviceSynchronize或cudaStreamSynchronize来保证MPS客户端空闲。如果在不同步所有未完成的GPU工作的情况下提前终止MPS客户端，可能会使MPS服务器处于未定义的状态，并导致意外故障、损坏或挂起；因此，必须重新启动受影响的MPS服务器及其所有客户端。

在Volta MPS上，用户可以通过使用控制命令‘Terminate_Client<SERVER PID><CLIENT PID>’来指示MPS服务器终止MPS客户端进程的CUDA上下文，而不管CUDA上下文是否空闲。此机制使用户能够终止给定MPS客户端进程的CUDA上下文，即使CUDA上下文是非空闲的，也不会影响MPS服务器或其其他MPS客户端。控制命令‘Terminate_Client’向MPS服务器发送请求，MPS服务器代表用户终止目标MPS客户端进程的CUDA上下文，并在MPS服务器完成请求后返回。如果目标MPS客户端进程的CUDA上下文已成功终止，则返回值为CUDA_SUCCESS；否则，返回描述失败状态的CUDA错误。当MPS服务器开始处理该请求时，在目标MPS客户端进程中运行的每个MPS客户端上下文变为‘非活动’；状态改变将由MPS服务器记录。在客户端终止成功完成后，目标MPS客户端进程将观察到粘性错误CUDA_ERROR_MPS_CLIENT_TERMINATED，并且利用诸如SIGKILL的信号终止目标MPS客户端进程变得安全，而不影响MPS服务器的其余部分及其MPS客户端。注意，在设置粘性错误之后，MPS服务器不负责终止目标MPS客户端进程，因为目标MPS客户端进程可能想要：
对其GPU或CPU状态执行清理。这可能包括设备重置。继续剩余的CPU工作。
继续剩余的CPU工作。

如果用户想要终止在不同于MPS控制的PID命名空间的PID命名空间内运行的MPS客户端进程的GPU工作，例如容器内的MPS客户端进程，则用户必须使用转换成MPS控制的PID命名空间的目标MPS客户端进程的PID。例如，容器内的一个MPS客户端进程的PID值为6，该MPS客户端进程在宿主PID命名空间中的PID值为1024；用户必须使用1024来终止目标MPS客户端进程的GPU工作。
终止客户端应用程序‘nbody’的常见工作流程如下：
使用控制命令‘ps’获取当前活动MPS客户端的状态
```shell
$echo "ps" |nvidia-cuda-mps-control
PID ID SERVER DEVICE NAMESPACE COMMAND
9741 0 6472 GPU-cb1213a3-d6a4-be7f 4026531836 ./nbody
9743 0 6472 GPU-cb1213a3-d6a4-be7f 4026531836 ./matrixMul
使用‘ps’报告的主机PID命名空间中的‘nbody’终止：
$echo "terminate_client 6472 9741" | nvidia-cuda-mps-control
#等待TERMINATE_CLIENT返回
#终止成功返回0
0
现在可以安全地杀掉‘nbody’了
$kill -9 9741
```

### Client 优先级控制
在编写程序时，用户通常只能通过使用cudaStreamCreateWithPriority()API来控制其内核的GPU优先级别。在Volta MPS上，用户可以使用控制命令`set_default_client_priority <Priority Level>`将给定客户端的流优先级映射到不同范围的内部CUDA优先级。在打开与服务器的下一个客户端连接之前，对此设置的更改不会生效。用户还可以在启动控制守护程序或任何给定客户端进程以设置此值之前设置 `CUDA_MPS_CLIENT_PRIORITY` 环境变量。
在此版本中，允许的优先级值为‘0’(正常)和‘1’(低于正常)。较低的数字映射到较高的优先级，以匹配Linux内核调度程序的行为。
注意：CUDA优先级并不是执行顺序的保证-它们只是对CUDA驱动程序的性能提示。
例如：
进程A以正常优先级启动，并且仅使用最低优先级为0的默认CUDA流。
进程B以低于正常优先级的优先级启动，并使用具有自定义流优先级值的流，例如-3。
如果没有此功能，来自进程B的流将首先由CUDA驱动程序执行。但是，使用客户端优先级功能时，来自进程A的流将优先。

## 架构
### background
CUDA是一个通用的并行计算平台和编程模型，它利用NVIDIA图形处理器中的并行计算引擎，以比在CPU上更高效的方式解决许多复杂的计算问题。

CUDA程序从为特定GPU显式使用驱动程序API或隐式使用运行时API创建CUDA上下文开始。该上下文封装了程序能够管理内存并在该GPU上启动工作所需的所有硬件资源。

在GPU上启动工作通常涉及将数据复制到GPU内存中先前分配的区域，运行对该数据进行操作的CUDA内核，然后将结果从GPU内存复制回系统内存。CUDA内核由在GPU计算引擎上并行执行的线程组的层次结构组成。

在使用CUDA启动的GPU上的所有工作要么显式启动到CUDA流中，要么使用默认流隐式启动。流是代表命令序列的软件抽象，命令序列可以是按顺序执行的内核、副本和其他命令的混合。在两个不同的流中启动的工作可以同时执行，从而实现粗粒度并行。

驱动程序将CUDA流别名到GPU上的一个或多个‘工作队列’上。工作队列是硬件资源，表示要由GPU上的特定引擎执行的流中命令子集的有序序列，例如内核执行或内存副本。带有Hyper-Q的GPU有一个并发调度程序来调度属于单个CUDA环境的工作队列中的工作。从属于同一CUDA上下文的工作队列启动到计算引擎的工作可以在GPU上并发执行。

GPU还具有时间片调度器，用于调度属于不同CUDA上下文的工作队列中的工作。从属于不同CUDA上下文的工作队列启动到计算引擎的工作不能同时执行。如果从单个CUDA上下文启动的工作不足以用完它可用的所有资源，这可能会导致GPU的计算资源未得到充分利用。

此外，在软件层内，为了从OS接收异步通知并代表应用程序执行异步CPU工作，CUDA驱动程序可以创建内部线程：Up Call处理程序线程和潜在的用户回调执行程序线程。
### Client-Server 架构
![Client-server Architecture](https://yikox.github.io/imgs/clinet-server-architecture.png)

此图显示了在运行由多个OS进程组成的MPI应用程序时，CUDA内核的可能调度。请注意，虽然每个MPI进程内的CUDA内核可以被并发调度，但每个MPI进程都被分配了整个图形处理器上的连续调度时间片。

![pre-volta-clinet-server-architecture](https://yikox.github.io/imgs/pre-volta-clinet-server-architecture.png)

使用 pre Volta MPS 的版本时，服务器管理与单个CUDA上下文关联的硬件资源。属于MPS客户的CUDA上下文通过MPS服务器进行工作。这允许客户端CUDA上下文绕过与时间切片调度相关的硬件限制，并允许其CUDA内核同时执行。

Volta提供新的硬件功能来减少MPS服务器必须管理的硬件资源类型。客户端CUDA上下文管理Volta上的大部分硬件资源，并将工作直接提交给硬件。Volta MPS服务器调解确保同时安排各个客户端提交的工作所需的剩余共享资源，并远离关键执行路径。

MPS客户端和MPS服务器之间的通信完全封装在CUDA API后面的CUDA驱动程序中。因此，MPS对MPI程序是透明的。

MPS客户端CUDA上下文保留其向上调用处理程序线程和任何同步执行程序线程。MPS服务器创建一个额外的向上调用处理程序线程，并为每个客户端创建一个工作线程。


### Provisioning Sequence
![Provisioning Sequence](https://yikox.github.io/imgs/provisioning-sequence.png)
具有多个用户的系统范围预配。
#### 服务器
MPS控制守护进程负责MPS服务器的启动和关闭。控制守护程序一次最多允许一个MPS服务器处于活动状态。当MPS客户端连接到控制守护程序时，如果没有活动的服务器，则该守护程序启动MPS服务器。以与MPS客户端相同的用户ID启动MPS服务器。
如果存在已经活动的MPS服务器，并且服务器和客户端的用户ID匹配，则控制守护程序允许客户端继续连接到服务器。如果MPS服务器已经处于活动状态，但服务器和客户端是使用不同的用户ID启动的，则一旦所有客户端都断开连接，控制守护进程就会请求现有服务器关闭。现有服务器关闭后，控制守护进程将启动一个新服务器，其用户ID与新用户的客户端进程的用户ID相同。这如上图所示，其中用户Bob在服务器可用之前启动了客户端C‘。只有当用户Alice的客户端退出时，才会为用户Bob和客户端C‘创建服务器。
如果没有挂起的客户端请求，则MPS控制守护程序不会关闭活动服务器。这意味着即使所有活动客户端退出，活动MPS服务器进程也将持续。当使用与活动MPS服务器不同的用户ID启动的新MPS客户端连接到控制守护程序时，或者当客户端启动的工作已经导致故障时，活动服务器被关闭。这如上面的示例所示，其中，控制守护进程仅在用户Bob启动客户端C时向Alice的服务器发出服务器退出请求，即使Alice的所有客户端都已退出。
活动MPS服务器可能处于以下状态之一：正在初始化、活动或故障。正在初始化状态指示MPS服务器正忙于初始化，并且MPS控制将在其队列中保持新的客户端请求。活动状态指示MPS服务器能够处理新的客户端请求。故障状态表示MPS服务器因客户端导致的致命故障而被阻止。任何新的客户端请求都将被拒绝，并显示错误CUDA_ERROR_MPS_SERVER_NOT_READY。
新启动的MPS服务器将首先处于正在初始化状态。初始化成功后，MPS服务器进入活动状态。当客户端遇到致命故障时，MPS服务器将从“活动”状态转变为“故障”状态。在Pre-Volta MPS上，MPS服务器在遇到致命故障后关闭。在Volta MPS上，在所有故障客户端断开连接后，MPS服务器再次变为活动状态。
控制守护程序可执行文件还支持交互模式，在该模式下，具有足够权限的用户可以发出命令，例如查看服务器和客户端的当前列表及其状态或手动启动和关闭服务器。
#### 客户端连接/断开
在程序中首次初始化CUDA时，CUDA驱动程序会尝试连接到MPS控制守护程序。如果连接尝试失败，程序在没有MPS的情况下将继续正常运行。然而，如果连接尝试成功，则MPS控制守护进程继续以确保以与连接客户端的用户ID相同的用户ID启动的MPS服务器在返回客户端之前是活动的。然后，MPS客户端继续连接到服务器。
MPS客户端、MPS控制守护进程和MPS服务器之间的所有通信都使用命名管道和UNIX域套接字完成。MPS服务器启动工作线程以接收来自客户端的命令。当客户端状态变为‘ACTIVE’时，MPS服务器将记录成功的客户端连接。在客户端进程退出时，服务器将销毁客户端进程未显式释放的任何资源，并终止工作线程。客户端退出事件将由MPS服务器记录。

## 附录：工具和界面参考
以下实用程序和环境变量用于管理MPS执行环境。下面将介绍它们以及标准CUDA编程环境的其他相关部分。
### Utilities and Daemons
#### nvidia-cuda-mps-Control
此控制守护程序通常存储在Linux系统上的/usr/bin下，并且通常以超级用户特权运行，用于管理下一节中描述的nvidia-cuda-mps-server。以下是相关用例：
```shell
man nvidia-cuda-mps-control # Describes usage of this utility.

nvidia-cuda-mps-control -d # Start daemon in background process.

ps -ef | grep mps # See if the MPS daemon is running.

echo quit | nvidia-cuda-mps-control # Shut the daemon down.

nvidia-cuda-mps-control -f # Start daemon in foreground

nvidia-cuda-mps-control -v           # Print version of control daemon executable (applicable on Tegra platforms only).
```
控制守护程序创建nvidia-cuda-mps-control.pid文件，其中包含CUDA_MPS_PIPE_DIRECTORY中控制守护程序进程的ID。当控制守护程序的多个实例并行运行时，可以通过在相应的CUDA_MPS_PIPE_DIRECTORY中查找特定实例的ID来针对特定实例。如果未设置CUDA_MPS_PIPE_DIRECTORY，则将在默认管道目录/tmp/nvidia-mps中创建nvidia-cuda-mps-control.pid文件。
在交互模式下使用时，可用的命令包括:
- get_server_list - 这将打印出服务器实例的所有PID的列表。
- get_server_status<PID> - 这将打印出具有给定<PID>的服务器的状态。
- start_server -uid<user id>-这将使用给定的用户ID手动启动nvidia-cuda-mps-server的新实例。
- get_client_list <PID> - 列出连接到分配给给定ID的服务器实例的客户端应用程序的ID
- quit - 终止nvidia-cuda-mps-control后台进程

可用于Volta MPS控制的命令：
- get_device_client_list [<PID>] - 列出设备和列举此设备的客户端应用程序的PID。它可选地获取服务器实例的PID。
- set_default_active_thread_percentage <percentage> - 这将覆盖MPS服务器的默认活动线程百分比。如果已经派生了一台服务器，则此命令将仅影响下一台服务器。如果执行退出命令，设置的值将丢失。默认值为100。
- get_default_active_thread_percentage - 查询当前默认可用线程百分比。
- set_active_thread_percentage <PID> <percentage> - 这将覆盖给定PID的MPS服务器实例的活动线程百分比。- 之后使用该服务器创建的所有客户端都将遵守新限制。现有客户端不受影响。
- get_active_thread_percentage <PID> - 查询给定ID的MPS服务器实例的当前可用线程百分比。
- set_default_device_pinned_mem_limit <dev> <value> - 设置每个MPS客户端的默认设备固定内存限制。如果已经派生了一台服务器，则此命令将仅影响下一台服务器。如果执行退出命令，设置的值将丢失。该值的形式必须是一个整数，后跟一个限定符，即分别以GB或MB为单位指定值的“G”或“M”。例如：为了将设备0的限制设置为10 GB，使用的命令为：
- set_default_device_pinned_mem_limit 0 10G。
- 默认情况下，内存限制处于禁用状态。
- get_default_device_pinned_mem_limit <dev> - 查询设备的当前默认固定内存限制。
- set_device_pinned_mem_limit <PID> <dev> <Value> - 这将覆盖MPS服务器的设备固定内存限制。这为设备设备的- 给定ID的MPS服务器实例的每个客户端设置设备固定的存储器限制。之后使用该服务器创建的所有客户端都将遵守新限制。现有客户端不受影响。将设备的ID为1024的服务器的内存限制设置为900MB的用法示例。 set_Device_pinned_mem_limit 1024 0900M
- get_device_pinned_mem_limit <PID> <dev> - 查询设备设备的给定ID的MPS服务器实例的当前设备固定内存限制。
每个节点应仅运行nvidia-cuda-mps-Control守护程序的一个实例
- terminate_client <server PID> <client PID> - 终止在由 <server PID> 表示的MPS服务器上运行的MPS客户端进程的所有未完成的GPU工作。用于终止在PID123的MPS服务器上运行的PID1024的MPS客户端进程的未完成的GPU工作的示例用法：terminate_client 123 1024
- ps[-p PID] - 报告当前客户端进程的快照。它可选地获取服务器实例的ID。它显示了ID、服务器分配的唯一标识符、关联设备的部分UUID、连接的服务器的ID、命名空间ID和客户端的命令行。
- set_default_client_priority [priority] - 设置将用于新客户端的默认客户端优先级。该值不适用于现有客户端。优先级值应被视为对CUDA驱动程序的提示，而不是保证。允许值为0[NORMAL]和1[BELOW NORMAL]。如果执行退出命令，设置的值将丢失。默认值为0[NORMAL]。
- get_default_client_priority - 查询将用于新客户端的当前优先级值。
#### nvidia-cuda-mps-server
此守护程序通常存储在Linux系统上的 /usr/bin下，与在节点上运行的客户端应用程序在相同的$UID下运行。的.当客户端应用程序连接到控制守护程序时，nvidia-cuda-mps-server实例按需创建。不应直接调用服务器二进制文件，而是应使用控制守护程序来管理服务器的启动和关闭。
nvidia-cuda-mps-server 进程拥有图形处理器上的CUDA上下文，并使用它为其客户端应用程序进程执行图形处理器操作。因此，当通过nvidia-smi（或任何基于NVML的应用程序）查询活动进程时，nvidia-cuda-mps-server将显示为活动的CUDA进程，而不是任何客户端进程。
在Tegra平台上，nvidia-cuda-mps-server可执行文件的版本可以打印：
```shell
nvidia-cuda-mps-server -v
```
#### nvidia-smi
通常存储在Linux系统上的/usr/bin下，用于配置节点上的图形处理器。以下用例与管理MPS相关：
```shell
man nvidia-smi # Describes usage of this utility.

nvidia-smi -L # List the GPU's on node.

nvidia-smi -q # List GPU state and configuration information.

nvidia-smi -q -d compute # Show the compute mode of each GPU.

nvidia-smi -i 0 -c EXCLUSIVE_PROCESS # Set GPU 0 to exclusive mode, run as root.

nvidia-smi -i 0 -c DEFAULT # Set GPU 0 to default mode, run as root. (SHARED_PROCESS)

nvidia-smi -i 0 -r # Reboot GPU 0 with the new setting.
```
### 环境变量
#### CUDA_VISIBLE_DEVICES
CUDA_VISIBLE_DEVICES用于指定哪些图形处理器应该对CUDA应用程序可见。只有序列中存在索引或UID的设备才能对CUDA应用程序可见，并且它们按序列的顺序列举。
当在启动控制守护程序之前设置了CUDA_VISIBLE_DEVICES时，MPS服务器将重新映射设备。这意味着，如果您的系统具有设备0、1和2，并且如果CUDA_VISIBLE_DEVICES设置为“0，2”，那么当客户端连接到服务器时，它将看到重新映射的设备-设备0和设备1。因此，在启动客户端时将CUDA_VISIBLE_DEVICES设置为“0，2”将导致错误。
如果任何可见设备是Volta+，MPS控制守护程序将进一步过滤掉任何Volta之前的设备。
为了避免这种模糊性，我们建议使用UID而不是索引。这些可以通过启动nvidia-smi -q来查看。启动服务器或应用程序时，您可以将CUDA_VISIBLE_DEVICES设置为“UUID_1，UUID_2”，其中UUID_1和UUID_2是GDPUID。当您指定UID的前几个字符（包括“图形处理器-”）而不是完整的UID时，它也会起作用。
如果应用CUDA_VISIBLE_DEQUICES后可见不兼容的设备，MPS服务器将无法启动。
#### CUDA_MPS_PIPE_DIRECTORY
MPS控制守护程序、MPS服务器和关联的MPS客户端通过命名管道和UNix域插槽相互通信。这些管道和插座的默认目录是/tmp/nvidia-mps。环境变量CUDA_MPS_PIPE_DIRECTORY可用于覆盖这些管道和插座的位置。此环境变量的值在共享同一MPS服务器和MPS控制守护程序的所有MPS客户端之间应该一致。
包含这些命名管道和域插槽的目录的建议位置是本地文件夹，例如/tmp。如果指定的位置存在于共享的多节点文件系统中，则每个节点的路径必须是唯一的，以防止多个MPS服务器或MPS控制守护进程使用相同的管道和插槽。当按用户配置MPS时，目录应设置为一个位置，以便不同的用户最终不会使用同一目录。
#### CUDA_MPS_LOG_DIRECTORY