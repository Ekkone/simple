---
title: zephyr用户模式介绍
published: 2024-11-21
description: ''
updated: ''
tags:
  - zephyr
draft: false
pin: 0
toc: true
lang: ''
abbrlink: ''
---
# 功能综述
[Overview — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/kernel/usermode/overview.html)

> [!NOTE]
> ## 线程模型
> 用户模式线程被 [[Zephyr]] 视为不信任，因此与其他用户模式线程和内核隔离。 有缺陷或恶意的用户模式线程不能泄漏或修改另一个线程或内核的私有数据/资源，并且不能干扰或控制另一个用户模式线程或内核。
> 
> Zephyr 用户模式功能示例：
> 
> - 内核可以防止许多无意的编程错误，否则这些错误可能会悄无声息地或令人震惊地破坏系统。
> - 内核可以对复杂的数据解析器（如解释器、网络协议和文件系统）进行沙箱处理，这样恶意的第三方代码或数据就无法危害内核或其他线程。
> - 内核可以支持多个逻辑 "应用程序 "的概念，每个应用程序都有自己的线程组和私有数据结构，如果其中一个应用程序崩溃或受到其他危害，这些应用程序就会相互隔离。
> ### 设计目标
> 对于在非特权 CPU 状态（以下简称 "用户模式"）下运行的线程，我们的目标是防止以下情况发生：
> 
> - 我们会防止访问未特别授权的内存，或错误访问与策略不兼容的内存，如尝试写入只读区域。
> 	- 对线程堆栈缓冲区的访问将受到策略控制，该策略部分取决于底层内存保护硬件。
> 		- 默认情况下，用户线程可以读/写访问自己的堆栈缓冲区。
> 		- 用户线程默认永远无法访问不属于同一内存域的用户线程栈。
> 		- 默认情况下，用户线程永远无法访问监督线程拥有的线程栈，或用于处理系统调用权限提升、中断或 CPU 异常的线程栈。
> 		- 用户线程可以读/写访问同一内存域中其他用户线程的堆栈，具体取决于硬件。
> 			- 在 MPU 系统上，线程只能访问自己的堆栈缓冲区。
> 			- 在 MMU 系统中，线程可以访问同一内存域中的任何用户线程栈。可移植代码不应假定这一点。
> 	- 默认情况下，所有内核范围内的线程都可以只读方式访问程序文本和只读数据。这一策略可以调整。
> 	- 除上述内容外，用户线程默认不允许访问任何内存。
> - 我们根据每个对象或每个驱动程序实例的权限粒度，防止使用未经特别授权的设备驱动程序或内核对象。
> - 我们会验证带有错误参数的内核或驱动程序 API 调用，否则会导致内核崩溃或内核专用数据结构损坏。这包括：
> 	- 使用错误的内核对象类型。
> 	- 使用超出适当范围或不合理值的参数。
> 	- 根据 API 的语义，传递调用线程没有足够权限读取或写入的内存缓冲区。
> 	- 使用未处于正确初始化状态的内核对象。
> - 我们确保检测和安全处理用户模式堆栈溢出。
> - 防止调用内核配置排除功能的系统调用。
> - 防止禁用或篡改内核定义和硬件执行的内存保护。
> - 防止从用户模式重新进入监管模式，除非通过内核定义的系统调用和中断处理程序。
> - 防止用户模式线程引入新的可执行代码，内核系统调用支持的情况除外。
> 
> **我们特别不防范以下攻击：**
> - 内核本身以及在管理模式下执行的任何线程都被假定为可信的。
> - 假设构建系统使用的工具链和任何补充程序是可信的。
> - 假设内核构建是可信的。有相当多的构建时逻辑用于创建有效内核对象表、定义系统调用和配置中断。在此过程中使用的 .elf 二进制文件都被假定为可信代码。
> - 我们无法防止在内核模式下完成的内存域配置中发生错误，从而将私有内核数据结构暴露给用户线程。内核对象的 RAM 应始终配置为仅限管理程序。
> - 可以对用户模式线程进行顶层声明，并将权限分配给内核对象。一般来说，作为生成 zephyr.elf 的内核构建的一部分的所有 C 和头文件都被假定为可信的。
> - 我们不会通过线程CPU 不足来防止DDOS攻击。 Zephyr 没有线程优先级老化，并且特定优先级的用户线程可以使所有较低优先级的线程挨饿，如果未启用时间分片，则还可以使其他相同优先级的线程挨饿。
> - 对于可以同时活动的线程数量存在构建时定义的限制，在此之后创建新用户线程将失败。
> - 在管理模式下运行的线程的堆栈溢出可能会被捕获，但无法保证系统的完整性。
> ## 策略细节
> 概括地说，我们通过以下机制来实现这些线程级内存保护目标：
> - 任何用户线程都只能访问内存的一个子集：通常是它的堆栈、程序文本、只读数据，以及它所属的内存保护设计中配置的任何分区。对其他内存的访问必须通过系统调用以线程的名义进行，或由监督线程使用内存域应用程序接口专门授予。新创建的线程继承父线程的内存域配置。线程之间可以通过共享相同内存域的成员身份，或通过内核对象（如 semaphores 和管道）进行通信。
> - 用户线程不能直接访问属于内核对象的内存。虽然内核对象的指针可用于引用这些对象，但对内核对象的实际操作是通过系统调用接口完成的。设备驱动程序和线程栈也被视为内核对象。这就确保了内核对象中任何属于内核私有的数据都不会被篡改。
> - 用户线程默认情况下没有权限访问除自己线程对象外的任何内核对象或驱动程序。这种访问权限必须由另一个处于监督模式的线程授予，或者该线程同时拥有接收线程对象和被授予访问权限的内核对象的权限。创建新线程时，可以选择自动继承父线程授予的所有内核对象权限，但父线程本身除外。
> - 出于性能和占用空间的考虑，Zephyr 通常很少或根本不对内核对象或设备驱动程序 API 进行参数错误检查。通过系统调用从用户模式访问涉及到额外的处理函数层，这些处理函数需要严格验证访问权限和对象类型，通过边界检查或其他方法检查其他参数的有效性，并验证对相关内存缓冲区的正确读/写访问。
> - 线程堆栈的定义方式是，超过指定的堆栈空间将产生硬件故障。具体做法因体系结构而异。
> ## 注意
> **如果要在用户模式下使用所有内核对象、线程栈和设备驱动程序实例，则必须在构建时对其进行定义。**内核对象的动态用例需要通过预定义的可用对象池。
> 
> 如果在内核启动后加载执行额外的应用程序二进制数据，则会受到一些限制：
> 
> - 加载对象代码将无法定义任何内核可识别的内核对象。相反，这些代码需要使用 API 从池中请求内核对象。
> - 同样，由于加载的目标代码不是内核构建过程的一部分，因此无论以何种模式运行，这些代码都无法安装中断处理程序、实例化设备驱动程序或定义系统调用。
> - 如果加载的目标代码并非来自经过验证的源代码，则应始终在 CPU 已处于用户模式的情况下输入。

# 例程说明
```shell
ekko@work: ~/zephyrproject/zephyr/samples/userspace dev!
$ tree                                                                                                               
.
├── hello_world_user
│   ├── CMakeLists.txt
│   ├── README.rst
│   ├── prj.conf
│   ├── sample.yaml
│   └── src
│       └── main.c
├── malloc
│   ├── CMakeLists.txt
│   ├── prj.conf
│   └── src
│       └── main.c
├── prod_consumer
│   ├── CMakeLists.txt
│   ├── README.rst
│   ├── prj.conf
│   ├── sample.yaml
│   └── src
│       ├── app_a.c
│       ├── app_a.h
│       ├── app_b.c
│       ├── app_b.h
│       ├── app_shared.c
│       ├── app_shared.h
│       ├── app_syscall.c
│       ├── app_syscall.h
│       ├── main.c
│       ├── sample_driver.h
│       ├── sample_driver_foo.c
│       └── sample_driver_handlers.c
├── shared_mem
│   ├── CMakeLists.txt
│   ├── README.rst
│   ├── boards
│   │   ├── nucleo_f746zg.overlay
│   │   └── ok3568.conf
│   ├── prj.conf
│   ├── sample.yaml
│   └── src
│       ├── enc.c
│       ├── enc.h
│       ├── main.c
│       └── main.h
└── syscall_perf
    ├── CMakeLists.txt
    ├── README.rst
    ├── prj.conf
    ├── sample.yaml
    └── src
        ├── main.c
        ├── test_supervisor.c
        ├── test_user.c
        └── thread_def.h

11 directories, 43 files
```
官方共有四个用例，测试libc增加了一个用例，各用例内容：

| 用例               | 功能                                                                 |
| ---------------- | ------------------------------------------------------------------ |
| hello_world_user | 基础用例，在用户模式线程打印信息                                                   |
| prod_consumer    | 生产者消费者用例，较完整的流程，包括了内存域配置、资源池配置、内核对象权限传递、系统调用等用户模式基本操作，用户模式开发可参考该例程 |
| shared_mem       | 共享内存用例，主要靠内存域配置实现多用户线程的通信                                          |
| syscall_perf     | 系统调用用例，用于测试特权模式和用户模式系统调用的性能差异，但使用的riscv的csr(控制状态)寄存器系统调用           |
| malloc           | 用户模式使用libc中malloc示例                                                |
# 开发指南
本节讲解使用用户模式需要哪些步骤，涉及到哪些操作，只做基本描述，详细步骤请参考prod_consumer例程。
## 1. 工程配置
需要在zephyr工程文件中开启用户空间
> [!prj.conf]
> CONFIG_USERSPACE=y

## 2. 用户模式全局变量定义
用户模式对内存权限有要求，因此使用到的全局变量需要手动规划到具体的分区：
```c
K_APPMEM_PARTITION_DEFINE(app_a_partition);//定义a app使用到的分区

#define APP_A_DATA  K_APP_DMEM(app_a_partition)//data数据段，存放初始不为0的全局/静态数据
#define APP_A_BSS   K_APP_BMEM(app_a_partition)//bss段，存放初始化为0的全局/静态数据

//后续定义全局变量需要在前面加上数据段修饰
APP_A_BSS unsigned int count;
APP_A_DATA uint32_t baudrate = 115200;
```
## 3. 创建用户线程
和普通创建线程流程相同，差异点在option选择K_USER，用户模式，以及线程运行延时为无限，因为后续对该线程还有其他操作
```c
k_tid_t user_tid = k_thread_create(&user_thread, user_stack, STACKSIZE,
			thread_entry, NULL, NULL, NULL,
			-1, K_USER,
			K_FOREVER);
```
## 4. 内存域配置
用户模式下的线程只有权限操作内存域中的分区，这一步也就把前面全局变量的分区权限给到了用户线程，内存域可配多个分区，也可动态修改，多个用户线程可通过某一个大家都有权限的分区实现共享内存通信和数据交互
```c
#include <zephyr/app_memory/app_memdomain.h>
struct k_mem_domain user_domain;//定义一个内存域

//a app的分区列表
struct k_mem_partition *app_a_parts[] = {
		&app_a_partition,
		&z_libc_partition,//libc 全局变量分区
		&z_malloc_partition//libc malloc分区
	};
//初始化内存域
k_mem_domain_init(&user_domain, ARRAY_SIZE(app_a_parts), app_a_parts);

//将用户线程添加到内存域
k_mem_domain_add_thread(&user_domain, user_tid);
```
## 5. 资源池配置
资源池就是一个k_heap的指针，也就是一个可以申请内存的堆，在一些api函数中需要用到这个资源池来进行操作。而创建的线程默认是没有配置资源池的，特权模式的线程可以使用系统本身的资源池，而用户模式的线程就需要手动绑定一个资源池：
```c
//全局变量定义资源池
K_HEAP_DEFINE(app_a_resource_pool, 256 * 5 + 128);
//将资源池绑定线程
k_thread_heap_assign(user_tid, &app_a_resource_pool);
```
## 6. 授权内核对象
内核对象包括以下类型，其中`device`为驱动设备，即所有驱动设备都属于内核对象：
```python
kobjects = OrderedDict([
    ("k_mem_slab", (None, False, True)),
    ("k_msgq", (None, False, True)),
    ("k_mutex", (None, False, True)),
    ("k_pipe", (None, False, True)),
    ("k_queue", (None, False, True)),
    ("k_poll_signal", (None, False, True)),
    ("k_sem", (None, False, True)),
    ("k_stack", (None, False, True)),
    ("k_thread", (None, False, True)), # But see #
    ("k_timer", (None, False, True)),
    ("z_thread_stack_element", (None, False, False)),
    ("device", (None, False, False)),
    ("NET_SOCKET", (None, False, False)),
    ("net_if", (None, False, False)),
    ("sys_mutex", (None, True, False)),
    ("k_futex", (None, True, False)),
    ("k_condvar", (None, False, True)),
    ("k_event", ("CONFIG_EVENTS", False, True)),
    ("ztest_suite_node", ("CONFIG_ZTEST", True, False)),
    ("ztest_suite_stats", ("CONFIG_ZTEST", True, False)),
    ("ztest_unit_test", ("CONFIG_ZTEST", True, False)),
    ("ztest_test_rule", ("CONFIG_ZTEST", True, False)),
    ("rtio", ("CONFIG_RTIO", False, False)),
    ("rtio_iodev", ("CONFIG_RTIO", False, False)),
    ("sensor_decoder_api", ("CONFIG_SENSOR_ASYNC_API", True, False))
])
```
默认内核对象的定义都用到了系统堆，用户模式是没有权限进行这种操作的，所以需要在全局变量创建（功能综述最后的注意中有提到），然后通过授权方式传递给用户线程：
```c
//消息队列
K_MSGQ_DEFINE(mqueue, SAMPLE_DRIVER_MSG_SIZE, MAX_MSGS, 4);
//队列
K_QUEUE_DEFINE(queue);
//驱动
APP_A_BSS const struct device *sample_device;
sample_device = device_get_binding(SAMPLE_DRIVER_NAME_0);
//授权内核对象列表
k_thread_access_grant(user_tid, &mqueue, &queue, sample_device);
```
## 7.运行用户线程
终于，到了最后一步，之后该线程就运行在了用户模式
```c
k_thread_start(user_tid);
```
## 8.运行用户函数
除了上述方式创建用户模式线程外，还可以直接调用函数使当前线程进入用户模式
```c
void user_func()
{
	printf("hello world\n");
}
...
//调用该函数后进入user_func并切换到用户模式
k_thread_user_mode_enter(user_func, NULL, NULL, NULL);
```
## 9.动态创建内核对象
如果用户线程需要创建自己内部使用的一些内核对象，比如信号量，队列，锁等，而不是通过授权内核对象的方式获取全局内核对象的话，则按照以下步骤进行创建：
### 1. 工程配置
首先需要开启动态创建对象功能，该功能是用户模式才有的，在工程配置文件添加以下选项：
```config
CONFIG_DYNAMIC_OBJECTS=y
```
### 2.确保资源池已配置
也就是前面提到的第5步，资源池配置，资源池除了用于已授权内核对象部分api使用，也可以用于动态创建内核对象
### 3.创建使用内核对象
```c
/*
K_OBJ_MEM_SLAB,
K_OBJ_MSGQ,
K_OBJ_MUTEX,
K_OBJ_PIPE,
K_OBJ_QUEUE,
K_OBJ_POLL_SIGNAL,
K_OBJ_SEM,
K_OBJ_STACK,
K_OBJ_THREAD,
K_OBJ_TIMER,
K_OBJ_THREAD_STACK_ELEMENT,等等类型
*/
	//队列
	struct k_queue *queue = k_object_alloc(K_OBJ_QUEUE);
	printf("queue : %p\n",queue);
	k_queue_init(queue);
	printf("k_queue_init finish\n");
	int test = 10;
	

	k_queue_alloc_append(queue,&test);
	printf("k_queue_alloc_append finish\n");

	int *get = k_queue_get(queue,K_NO_WAIT);
	printf("get %p : %d\n",get,*get);

	//锁
	struct k_mutex *mutex = k_object_alloc(K_OBJ_MUTEX);
	printf("mutex : %p\n",mutex);

	k_mutex_init(mutex);
	printf("k_mutex_init finish\n");

	k_mutex_lock(mutex,K_FOREVER);
	printf("k_mutex_lock finish\n");

	k_mutex_unlock(mutex);
	printf("k_mutex_unlock finish\n");

	//消息队列
	struct k_msgq *msgq = k_object_alloc(K_OBJ_MSGQ);
	printf("msgq : %p\n",msgq);

	k_msgq_alloc_init(msgq,1,5);
	printf("k_msgq_alloc_init finish\n");

	k_msgq_put(msgq,&test,K_NO_WAIT);
	printf("k_msgq_put finish\n");
	int recv = 0;
	k_msgq_get(msgq,&recv,K_NO_WAIT);
	printf("recv : %d\n",recv);
	//信号量
	struct k_sem *sem = k_object_alloc(K_OBJ_SEM);
	printf("sem : %p\n",sem);

	k_sem_init(sem,0,10);
	printf("k_sem_init finish\n");

	k_sem_give(sem);
	printf("k_sem_give finish\n");

	k_sem_take(sem,K_NO_WAIT);
	printf("k_sem_take finish\n");

	//栈
	struct k_stack *stack = k_object_alloc(K_OBJ_STACK);
	printf("stack : %p\n",stack);

	k_stack_alloc_init(stack,10);
	printf("k_sem_take finish\n");

	k_stack_push(stack,1);
	printf("k_sem_take finish\n");

	stack_data_t pop;
	k_stack_pop(stack,&pop,K_NO_WAIT);
	printf("pop : %ld\n",pop);
```
# 实现原理
## 内存保护
### 内存域介绍
在zephyr下用户线程不能随意访问内存空间，用户线程之间除了通过内核交换数据外，还可以通过内存域的方法共享内存交换数据。  
内存域的主要设计规则如下:
- partition: 内存的基本单元，规定了内存的起始地址和大小
- domain: partition的集合，一个domain可以包含多个partition，同一个partition可以出现在多个domain中
- 一个用户线程只能绑定一个domain，用户线程拥有绑定domain内所有partition的访问权限  从上面的设计规则可以看到，当一个partition同时属于2个domain，那么这两个domain的拥有者thread，就可以通过这个partition交换数据。下图摘自ELC2019 Europe,Andrew Boie的slider说明了thread1和thread2通过共享蓝色区域的partition交换数据。
-   ![domain](https://lgl88911.github.io/2019/10/28/Zephyr%E7%94%A8%E6%88%B7%E6%A8%A1%E5%BC%8F-%E5%86%85%E5%AD%98%E4%BF%9D%E6%8A%A4/domain.png)
内存域相关结构体：
```c
/**
 * @brief Memory Partition
 *
 * A memory partition is a region of memory in the linear address space
 * with a specific access policy.
 *
 * The alignment of the starting address, and the alignment of the size
 * value may have varying requirements based on the capabilities of the
 * underlying memory management hardware; arbitrary values are unlikely
 * to work.
 */
struct k_mem_partition {
	/** start address of memory partition */
	uintptr_t start;
	/** size of memory partition */
	size_t size;
	/** attribute of memory partition */
	k_mem_partition_attr_t attr;
};
/**
 * @brief Memory Domain
 *
 * A memory domain is a collection of memory partitions, used to represent
 * a user thread's access policy for the linear address space. A thread
 * may be a member of only one memory domain, but any memory domain may
 * have multiple threads that are members.
 *
 * Supervisor threads may also be a member of a memory domain; this has
 * no implications on their memory access but can be useful as any child
 * threads inherit the memory domain membership of the parent.
 *
 * A user thread belonging to a memory domain with no active partitions
 * will have guaranteed access to its own stack buffer, program text,
 * and read-only data.
 */
struct k_mem_domain {
#ifdef CONFIG_ARCH_MEM_DOMAIN_DATA
	struct arch_mem_domain arch;
#endif /* CONFIG_ARCH_MEM_DOMAIN_DATA */
	/** partitions in the domain */
	struct k_mem_partition partitions[CONFIG_MAX_DOMAIN_PARTITIONS];
	/** Doubly linked list of member threads */
	sys_dlist_t mem_domain_q;
	/** number of active partitions in the domain */
	uint8_t num_partitions;
};
```
### 内存分区定义
```c
#ifdef Z_LIBC_PARTITION_EXISTS
K_APPMEM_PARTITION_DEFINE(z_libc_partition);
#endif /* Z_LIBC_PARTITION_EXISTS */

K_APPMEM_PARTITION_DEFINE(app_a_partition);
```
### 内存域配置
```c
#include <zephyr/app_memory/app_memdomain.h>
struct k_mem_domain user_domain;//定义一个内存域

//a app的分区列表
struct k_mem_partition *app_a_parts[] = {
		&app_a_partition,
		&z_libc_partition,//libc 全局变量分区
		&z_malloc_partition//libc malloc分区
	};
//初始化内存域
k_mem_domain_init(&user_domain, ARRAY_SIZE(app_a_parts), app_a_parts);

//将用户线程添加到内存域
k_mem_domain_add_thread(&user_domain, user_tid);
```
zephyr.map:
```map
 .app_regions.z_libc_partition
                0x00000000400557b0       0x10 zephyr/kernel/libkernel.a(userspace.c.obj)
                0x00000000400557b0                z_libc_partition_region
 .data.z_libc_partition
                0x0000000040151c90       0x18 zephyr/kernel/libkernel.a(userspace.c.obj)
                0x0000000040151c90                z_libc_partition
 .app_regions.app_a_partition
                0x00000000400557a0       0x10 app/libapp.a(main.c.obj)
                0x00000000400557a0                app_a_partition_region
```
### MPU/MMU功能差异
1. **MPU（Memory Protection Unit，内存保护单元）功能**
   - **内存保护**
     - MPU主要用于保护内存区域。它可以将内存划分为不同的区域，为每个区域设置访问权限，如可读、可写、可执行等权限组合。例如，在一个嵌入式系统中，操作系统的内核代码所在的内存区域可以被设置为只读和可执行，这样就可以防止应用程序意外地修改内核代码，从而提高系统的稳定性和安全性。
     - 对于不同的任务或进程，MPU能够确保它们只能访问自己被授权的内存区域。比如，在一个多任务的嵌入式设备中，任务A和任务B有各自的内存空间，MPU可以防止任务A非法访问任务B的内存，避免数据损坏和系统崩溃。
   - **资源划分**
     - MPU能够有效地划分系统的内存资源。它可以定义内存块的大小和起始地址，使系统能够更合理地利用内存。例如，在一个具有有限内存的微控制器系统中，通过MPU可以将内存划分为不同大小的缓冲区用于存储数据、代码和堆栈等。可以为数据缓冲区设置合适的读写权限，为代码区设置为只读和可执行，确保系统按照设计的方式正确地使用内存资源。
2. **MMU（Memory Management Unit，内存管理单元）功能**
   - **虚拟内存管理**
     - MMU的核心功能是实现虚拟内存到物理内存的映射。在现代操作系统中，每个进程都有自己独立的虚拟地址空间。例如，在一个32位的操作系统中，每个进程理论上可以有4GB的虚拟地址空间。MMU负责将这些虚拟地址转换为实际的物理地址。这样做的好处是，多个进程可以独立地使用相同的虚拟地址范围，而MMU会根据不同的进程将其映射到不同的物理内存区域，使得每个进程都感觉自己独占了整个地址空间，提高了系统的资源利用率和进程的隔离性。
   - **地址转换**
     - MMU使用页表（Page Table）来进行地址转换。页表是一种数据结构，记录了虚拟页号和物理页号之间的对应关系。当CPU发出一个虚拟地址时，MMU会根据页表将其转换为物理地址。例如，如果一个虚拟地址的页号为10，通过查询页表发现它对应的物理页号为20，MMU就可以正确地将虚拟地址转换为物理地址，从而让CPU能够访问到正确的物理内存位置。
   - **内存访问控制和缓存管理**
     - MMU还可以对内存访问进行控制，如设置不同的访问权限，包括读、写、执行等权限，这一点和MPU有些类似，但在虚拟内存环境下。同时，一些MMU还与缓存（Cache）系统协同工作，控制缓存的操作，例如决定哪些数据应该被缓存，以及如何更新缓存等内容，以提高内存访问的效率。
3. **功能差异总结**
   - **保护范围和目的**
     - MPU侧重于对物理内存的保护和资源划分，主要目的是防止不同的任务或模块在物理内存层面的非法访问，它是在物理内存的基础上进行权限管理。而MMU更关注虚拟内存环境下的地址转换和进程间的隔离，通过虚拟内存管理，使得多个进程能够安全、高效地共享物理内存资源，其保护范围涉及整个虚拟内存空间到物理内存的映射过程。
   - **应用场景差异**
     - MPU通常应用于一些资源相对有限、对实时性要求较高的嵌入式系统中，这些系统可能不需要复杂的虚拟内存机制，但需要对内存进行简单而有效的保护。例如，汽车电子控制系统中的微控制器，通过MPU可以防止不同的电子控制单元（ECU）之间的内存干扰。MMU则主要应用于复杂的操作系统环境，如桌面操作系统（Windows、Linux等）和大型服务器系统，这些系统需要处理多个进程同时运行的情况，MMU通过虚拟内存管理来实现进程的隔离和内存的高效利用。
### MMU保护
每个线程绑定对应的转换页表，在上下文切换时动态修改来达到各个线程访问各自权限分区的功能。
```c
int arch_mem_domain_partition_add(struct k_mem_domain *domain,
				  uint32_t partition_id)
{
	struct arm_mmu_ptables *domain_ptables = &domain->arch.ptables;
	struct k_mem_partition *ptn = &domain->partitions[partition_id];

	return private_map(domain_ptables, "partition", ptn->start, ptn->start,
			   ptn->size, ptn->attr.attrs | MT_NORMAL);
}

int arch_mem_domain_thread_add(struct k_thread *thread)
{
	struct arm_mmu_ptables *old_ptables, *domain_ptables;
	struct k_mem_domain *domain;
	bool is_user, is_migration;
	int ret = 0;

	domain = thread->mem_domain_info.mem_domain;
	domain_ptables = &domain->arch.ptables;
	old_ptables = thread->arch.ptables;

	is_user = (thread->base.user_options & K_USER) != 0;
	is_migration = (old_ptables != NULL) && is_user;

	if (is_migration) {
		ret = map_thread_stack(thread, domain_ptables);
	}

	thread->arch.ptables = domain_ptables;
	if (thread == _current) {
		//切换转换页表
		z_arm64_swap_ptables(thread);
	} else {
#ifdef CONFIG_SMP
		/* the thread could be running on another CPU right now */
		z_arm64_mem_cfg_ipi();
#endif
	}

	if (is_migration) {
		ret = reset_map(old_ptables, __func__, thread->stack_info.start,
				thread->stack_info.size);
	}

	return ret;
}

GTEXT(z_arm64_context_switch)
SECTION_FUNC(TEXT, z_arm64_context_switch)
#if defined(CONFIG_USERSPACE) || defined(CONFIG_ARM64_STACK_PROTECTION)
	str     lr, [sp, #-16]!
	bl      z_arm64_swap_mem_domains
	ldr     lr, [sp], #16
#endif

static void z_arm64_swap_ptables(struct k_thread *incoming)
{
	struct arm_mmu_ptables *ptables = incoming->arch.ptables;
	uint64_t curr_ttbr0 = read_ttbr0_el1();
	uint64_t new_ttbr0 = ptables->ttbr0;

	if (curr_ttbr0 == new_ttbr0) {
		return; /* Already the right tables */
	}

	MMU_DEBUG("TTBR0 switch from %#llx to %#llx\n", curr_ttbr0, new_ttbr0);
	z_arm64_set_ttbr0(new_ttbr0);

	if (get_asid(curr_ttbr0) == get_asid(new_ttbr0)) {
		invalidate_tlb_all();
	}
}
```
### MPU保护
提供内存域保护以及安全溢出栈保护
## 内核对象授权
每一个内核对象中都有一个权限数组，存储每个线程对该对象的访问权限，大多数情况都是使用`K_SYSCALL_OBJ`宏来检查当前线程是否有该对象的权限。
```c
struct k_object {
	void *name;
	uint8_t perms[CONFIG_MAX_THREAD_BYTES];
	uint8_t type;
	uint8_t flags;
	union k_object_data data;
} __packed __aligned(4);
```
```c
/**
 * Grant a thread access to a kernel object
 *
 * The thread will be granted access to the object if the caller is from
 * supervisor mode, or the caller is from user mode AND has permissions
 * on both the object and the thread whose access is being granted.
 *
 * @param object Address of kernel object
 * @param thread Thread to grant access to the object
 */
__syscall void k_object_access_grant(const void *object,
				     struct k_thread *thread);
```
```c
void z_impl_k_object_access_grant(const void *object, struct k_thread *thread)
{
	struct k_object *ko = k_object_find(object);

	if (ko != NULL) {
		k_thread_perms_set(ko, thread);
	}
}
void k_thread_perms_set(struct k_object *ko, struct k_thread *thread)
{
	int index = thread_index_get(thread);

	if (index != -1) {
		sys_bitfield_set_bit((mem_addr_t)&ko->perms, index);
	}
}

void k_object_access_all_grant(const void *object)
{
	struct k_object *ko = k_object_find(object);

	if (ko != NULL) {
		ko->flags |= K_OBJ_FLAG_PUBLIC;
	}
}
```
开启用户模式后，内核对象的调用接口会多一层封装来进行权限判断
```c
#ifdef CONFIG_USERSPACE
static inline int32_t z_vrfy_k_queue_alloc_append(struct k_queue *queue,
						  void *data)
{
	K_OOPS(K_SYSCALL_OBJ(queue, K_OBJ_QUEUE));
	return z_impl_k_queue_alloc_append(queue, data);
}
#include <zephyr/syscalls/k_queue_alloc_append_mrsh.c>
#endif /* CONFIG_USERSPACE */
```
```c
/**
 * @brief Runtime check kernel object pointer for non-init functions
 *
 * Calls k_object_validate and triggers a kernel oops if the check fails.
 * For use in system call handlers which are not init functions; a fatal
 * error will occur if the object is not initialized.
 *
 * @param ptr Untrusted kernel object pointer
 * @param type Expected kernel object type
 * @return 0 on success, nonzero on failure
 * @note This is an internal API. Do not use unless you are extending
 *       functionality in the Zephyr tree.
 */
#define K_SYSCALL_OBJ(ptr, type) \
	K_SYSCALL_IS_OBJ(ptr, type, _OBJ_INIT_TRUE)
#define K_SYSCALL_IS_OBJ(ptr, type, init) \
	K_SYSCALL_VERIFY_MSG(k_object_validation_check(			\
				     k_object_find((const void *)(ptr)),	\
				     (const void *)(ptr),		\
				     (type), (init)) == 0, "access denied")
				     
static inline int k_object_validation_check(struct k_object *ko,
					 const void *obj,
					 enum k_objects otype,
					 enum _obj_init_check init)
{
	int ret;
	ret = k_object_validate(ko, otype, init);

#ifdef CONFIG_LOG
	if (ret != 0) {
		k_object_dump_error(ret, obj, ko, otype);
	}
#else
	ARG_UNUSED(obj);
#endif

	return ret;
}
int k_object_validate(struct k_object *ko, enum k_objects otype,
		       enum _obj_init_check init)
{
	if (unlikely((ko == NULL) ||
		((otype != K_OBJ_ANY) && (ko->type != otype)))) {
		return -EBADF;
	}

	/* Manipulation of any kernel objects by a user thread requires that
	 * thread be granted access first, even for uninitialized objects
	 */
	if (unlikely(thread_perms_test(ko) == 0)) {
		return -EPERM;
	}

	/* Initialization state checks. _OBJ_INIT_ANY, we don't care */
	if (likely(init == _OBJ_INIT_TRUE)) {
		/* Object MUST be initialized */
		if (unlikely((ko->flags & K_OBJ_FLAG_INITIALIZED) == 0U)) {
			return -EINVAL;
		}
	} else if (init == _OBJ_INIT_FALSE) { /* _OBJ_INIT_FALSE case */
		/* Object MUST NOT be initialized */
		if (unlikely((ko->flags & K_OBJ_FLAG_INITIALIZED) != 0U)) {
			return -EADDRINUSE;
		}
	} else {
		/* _OBJ_INIT_ANY */
	}

	return 0;
}

static int thread_perms_test(struct k_object *ko)
{
	int index;

	if ((ko->flags & K_OBJ_FLAG_PUBLIC) != 0U) {
		return 1;
	}

	index = thread_index_get(_current);
	if (index != -1) {
		return sys_bitfield_test_bit((mem_addr_t)&ko->perms, index);
	}
	return 0;
}


```
## 系统调用
利用svc进入特权模式再执行系统调用
![[Pasted image 20241113164119.png]]
```c
static inline int z_sys_mutex_kernel_lock(struct sys_mutex * mutex, k_timeout_t timeout)
{
#ifdef CONFIG_USERSPACE
	if (z_syscall_trap()) {
		union { uintptr_t x; struct sys_mutex * val; } parm0 = { .val = mutex };
		union { uintptr_t x; k_timeout_t val; } parm1 = { .val = timeout };
		return (int) arch_syscall_invoke2(parm0.x, parm1.x, K_SYSCALL_Z_SYS_MUTEX_KERNEL_LOCK);
	}
#endif
	compiler_barrier();
	return z_impl_z_sys_mutex_kernel_lock(mutex, timeout);
}

static inline uintptr_t arch_syscall_invoke2(uintptr_t arg1, uintptr_t arg2,
					     uintptr_t call_id)
{
	register uint64_t ret __asm__("x0") = arg1;
	register uint64_t r1 __asm__("x1") = arg2;
	register uint64_t r8 __asm__("x8") = call_id;

	__asm__ volatile("svc %[svid]\n"
			 : "=r"(ret)
			 : [svid] "i" (_SVC_CALL_SYSTEM_CALL),
			   "r" (ret), "r" (r1), "r" (r8)
			 : "memory");

	return ret;
}
/*
 * Synchronous exceptions handler
 *
 * The service call (SVC) is used in the following occasions:
 * - Cooperative context switching
 * - IRQ offloading
 */
GTEXT(z_arm64_sync_exc)
SECTION_FUNC(TEXT, z_arm64_sync_exc)
#ifdef CONFIG_USERSPACE
	cmp	x1, #_SVC_CALL_SYSTEM_CALL
	beq	z_arm64_do_syscall
#endif 

GTEXT(z_arm64_do_syscall)
SECTION_FUNC(TEXT, z_arm64_do_syscall)
	/* Recover the syscall parameters from the ESF */
	ldp	x0, x1, [sp, ___esf_t_x0_x1_OFFSET]
	ldp	x2, x3, [sp, ___esf_t_x2_x3_OFFSET]
	ldp	x4, x5, [sp, ___esf_t_x4_x5_OFFSET]

	/* Use the ESF as SSF */
	mov	x6, sp

	/* Recover the syscall ID */
	ldr	x8, [sp, ___esf_t_x8_x9_OFFSET]

	/* Check whether the ID is valid */
	ldr	x9, =K_SYSCALL_LIMIT
	cmp	x8, x9
	blo	valid_syscall_id

	/* Save the bad ID for handler_bad_syscall() */
	mov	x0, x8
	ldr	x8, =K_SYSCALL_BAD

valid_syscall_id:
	ldr	x9, =_k_syscall_table
	ldr	x9, [x9, x8, lsl #3]

	/* Jump into the syscall */
	msr	daifclr, #(DAIFSET_IRQ_BIT)
	blr	x9
	msr	daifset, #(DAIFSET_IRQ_BIT)

	/* Save the return value into the ESF */
	str	x0, [sp, ___esf_t_x0_x1_OFFSET]

	/* Return from exception */
	b	z_arm64_exit_exc

```
# 优缺点分析
## 优点
前面的功能综述中已经对用户模式下的各种特性优势做了说明，这部分就说一下在实际使用中可能的几个场景：
1. 用户线程权限有限，多团队多用户APP开发即使出错不会影响到其他部分，沙盒性隔离性比较好
2. 内存权限明晰，不同功能线程内存安全性高，且不会对内核进行越权操作
## 劣势
1. 流程较为复杂，门槛变高
2. 权限受限，有些posix相关接口封装了很多内核对象，目前还未找到好的方式来授权，只能使用zephyr定义好的内核对象进行授权操作，对zephyr内核对象使用依赖性变高
3. 性能损失，前面提到的系统调用性能测试用例是RSICV平台的，以下输出供参考，列出了时钟耗费及指令数量的差异：
       User thread:		       18012 cycles	       748 instructions
 	   Supervisor thread:	       7 cycles	       4 instructions
 	   User thread:		       20136 cycles	       748 instructions
 	   Supervisor thread:	       7 cycles	       4 instructions
 	   User thread:		       18014 cycles	       748 instructions
 	   Supervisor thread:	       7 cycles	       4 instructions
# 参考资料
[User Mode — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/kernel/usermode/index.html)

[Zephyr用户模式-简介 | Half Coder](https://lgl88911.github.io/2019/10/02/Zephyr%E7%94%A8%E6%88%B7%E6%A8%A1%E5%BC%8F-%E7%AE%80%E4%BB%8B/)

[Zephyr用户模式-技术基础 | Half Coder](https://lgl88911.github.io/2019/10/02/Zephyr%E7%94%A8%E6%88%B7%E6%A8%A1%E5%BC%8F-%E6%8A%80%E6%9C%AF%E5%9F%BA%E7%A1%80/)

[Zephyr用户模式-系统调用 | Half Coder](https://lgl88911.github.io/2019/10/04/Zephyr%E7%94%A8%E6%88%B7%E6%A8%A1%E5%BC%8F-%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/)

[Zephyr用户模式-内存保护 | Half Coder](https://lgl88911.github.io/2019/10/28/Zephyr%E7%94%A8%E6%88%B7%E6%A8%A1%E5%BC%8F-%E5%86%85%E5%AD%98%E4%BF%9D%E6%8A%A4/)

[Zephyr用户模式-内核对象 | Half Coder](https://lgl88911.github.io/2019/10/26/Zephyr%E7%94%A8%E6%88%B7%E6%A8%A1%E5%BC%8F-%E5%86%85%E6%A0%B8%E5%AF%B9%E8%B1%A1/)

