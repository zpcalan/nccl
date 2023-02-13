
**NCCL作为Nvidia GPU侧唯一的通信软件，其性能和易用性经过了众多开源框架和工业场景的验证。本系列文章首先沿着NCCL实现一次集合通信的链路，对其源码进行精度，从而帮助自己对GPU物理拓扑，NCCL通信框架有着更深刻的理解。然后详细分析其中的特性模块：如通信，拓扑，集合通信算法等。总体来说，其中的编码规范（全局变量的使用，代码可靠性等）做的不是特别好，但其中编码细节处理还是有很多值得我们去学习的地方。**  
NCCL作为一个轻量级的通信框架，其版本发布周期约为2到4个月一个大版本，先简单说下其目录分布：

1. ext-net：网络通信扩展插件目录，可忽略。
2. makefiles：编译脚本。
3. pkg：各类操作系统下出包脚本。
4. src：源码目录。

其中src目录有若干文件夹：
1. collectives：所有集合通信的cuda实现。
2. graph：根据硬件选择通信拓扑实现。比如ring，tree等。
3. include：头文件目录。
4. misc：各种其他模块需要用到一堆的utils。
5. transport：跨进程通信的基本实现，包括socket，ib，shared-mem等。
其余不在文件夹下面的文件是一些分散的实现，在从初始化到集合通信的过程中都会被调用到。

首先从多进程多卡场景入手，依次阅读和理解每个接口实现。  
除开第三方做的host侧数据同步代码，做一次AllReduce的接口调用，NCCL的官网示例如下：

```c++
  if (myRank == 0) ncclGetUniqueId(&id);
  CUDACHECK(cudaSetDevice(localRank));
  CUDACHECK(cudaMalloc(&sendbuff, size * sizeof(float)));
  CUDACHECK(cudaMalloc(&recvbuff, size * sizeof(float)));
  CUDACHECK(cudaStreamCreate(&s));
  // Broadcast unique id to all processes with different ranks.
  NCCLCHECK(ncclCommInitRank(&comm, nRanks, id, myRank));
  NCCLCHECK(ncclAllReduce((const void*)sendbuff, (void*)recvbuff, size, ncclFloat, ncclSum, comm, s));
```

<h1>1.ncclGetUniqueId</h1>
第一个步骤为调用`ncclGetUniqueId`接口：

```c++
if (myRank == 0) ncclGetUniqueId(&id);
```

代表此接口只在root进程被调用，用于生成本通信域的唯一ID，阅读`ncclGetUniqueId`声明：  

```c++
nccl.h.in:75

/* Generates an Id to be used in ncclCommInitRank. ncclGetUniqueId should be
 * called once and the Id should be distributed to all ranks in the
 * communicator before calling ncclCommInitRank. */
ncclResult_t  ncclGetUniqueId(ncclUniqueId* uniqueId);
```

每创建一个通信域，此接口只需调用一次，且是在此通信域或者说`group`中rank id为0的根节点调用即可。生成唯一ID后，通过Host侧进程间通信同步到本通信域中其他所有进程，注意此能力是NCCL没有内置的，因此需要依赖第三方组件进行同步，如OpenMPI（官网使用）等：```MPI_Bcast((void *)&id, sizeof(id), MPI_BYTE, 0, MPI_COMM_WORLD)```

下面来看看`ncclGetUniqueId`做了什么：

```c++
init.cc:104

ncclResult_t ncclGetUniqueId(ncclUniqueId* out) {
  NCCLCHECK(ncclInit());
  NCCLCHECK(PtrCheck(out, "GetUniqueId", "out"));
  ncclResult_t res = bootstrapGetUniqueId(out);
  TRACE_CALL("ncclGetUniqueId(0x%llx)", (unsigned long long)hashUniqueId(*out));
  return res;
}
```

1. 首先调用`ncclInit`，从接口无入参来看应该是一些stateless变量的初始化：

```c++
init.cc:77

static ncclResult_t ncclInit() {
  if (__atomic_load_n(&initialized, __ATOMIC_ACQUIRE)) return ncclSuccess;
  // 保证线程安全，因为此函数还可支持多线程多卡等场景调用。
  pthread_mutex_lock(&initLock);
  // 可重入标志。
  if (!initialized) {
    // 加载nccl.conf中的一些默认环境变量配置。
    initEnv();
    // 判断是否需要使能GDRDMA，默认关闭。
    // GDRDMA一般通过安装Mellanox/nv_peer_memory使能，需要动态加载libgdrapi.so库。
    initGdrCopy();
    // TODO:此函数还不是很理解作用。
    maxLocalSizeBytes = ncclKernMaxLocalSize();
    // TODO:为cuda核函数设置共享内存切片？
    int carveout = ncclParamL1SharedMemoryCarveout();
    if (carveout) ncclKernSetSharedMemoryCarveout(carveout);
    // Always initialize bootstrap network
    // 初始化网络配置。
    NCCLCHECK(bootstrapNetInit());
    // 初始化网络通信插件。
    NCCLCHECK(ncclNetPluginInit());

    __atomic_store_n(&initialized, true, __ATOMIC_RELEASE);
  }
  pthread_mutex_unlock(&initLock);
  return ncclSuccess;
}
```

> 笔者搜索了此接口在`ncclCommInitRankDev`也会被调用，因此非root进程的也会正常初始化这些无状态的变量。

重点关注`bootstrapNetInit`和`ncclNetPluginInit`两个接口实现：
- bootstrapNetInit: 根据用户配置和内置规则查找对应的网卡，初始化核心全局变量bootstrapNetIfAddr和bootstrapNetIfName，作为host侧的通信网卡，在后续逻辑中会用到。
- ncclNetPluginInit



2. ID作为`bootstrapGetUniqueId`出参由其生成：
