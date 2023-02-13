**bootstrapNetInit代码解读**

此方法主要初始化网络相关配置，为进程间通信提供前置条件。全局变量bootstrapNetIfAddr和bootstrapNetIfName用来存放满足条件的网卡。
- Step 1: 如果用户设置环境变量NCCL_COMM_ID，则选择指定的网卡。
- Step 2: 若没有设置NCCL_COMM_ID，但设置了NCCL_SOCKET_IFNAME，则根据NCCL_SOCKET_IFNAME选择满足用户指定前缀的网卡。
- Step 3: 若都没有设置，则根据内置规则匹配网卡，查找网卡顺序为：ib，非docker和local loop back（以太网接口等），最后才是docker和local loop back。

```c++
bootstrap.cc:22

ncclResult_t bootstrapNetInit() {
  // 下面判断两次可重入标志bootstrapNetInitDone个人感觉没必要，一次配合加锁即可。
  if (bootstrapNetInitDone == 0) {
    pthread_mutex_lock(&bootstrapNetLock);
    if (bootstrapNetInitDone == 0) {
      // 环境变量NCCL_COMM_ID并未在官网开放，内部用于调试使用。
      // 可以做到更加精确的网络地址匹配。
      // 按照下面的接口语义，个人认为是设置了远程节点的物理地址，所以需要解析出remoteAddr后根据子网继续匹配localAddr，保证连通性。
      // 但是接口实现中注释说明环境变量为用户指定网卡，所以不太理解匹配两次（ncclGetSocketAddrFromString以及ncclFindInterfaceMatchSubnet）的原因，能否直接使用环境变量指定的地址。
      char* env = getenv("NCCL_COMM_ID");
      if (env) {
        union ncclSocketAddress remoteAddr;
        // 根据用户设置的环境变量NCCL_COMM_ID找到对应网络地址。
        if (ncclGetSocketAddrFromString(&remoteAddr, env) != ncclSuccess) {
          WARN("Invalid NCCL_COMM_ID, please use format: <ipv4>:<port> or [<ipv6>]:<port> or <hostname>:<port>");
          return ncclInvalidArgument;
        }
        // 根据用户指定的网络地址，匹配找到在同一网段的网卡。
        if (ncclFindInterfaceMatchSubnet(bootstrapNetIfName, &bootstrapNetIfAddr, &remoteAddr, MAX_IF_NAME_SIZE, 1) <= 0) {
          WARN("NET/Socket : No usable listening interface found");
          return ncclSystemError;
        }
      } else {
        // 根据规则查找可用网卡，规则在函数实现内部，按内置优先级查找网卡，因此没有入参可以指定规则。
        // 返回找到的网卡数量，出参bootstrapNetIfName为找到的一个网卡名，出参bootstrapNetIfAddr为网卡地址。
        int nIfs = ncclFindInterfaces(bootstrapNetIfName, &bootstrapNetIfAddr, MAX_IF_NAME_SIZE, 1);
        if (nIfs <= 0) {
          WARN("Bootstrap : no socket interface found");
          return ncclInternalError;
        }
      }
      char line[SOCKET_NAME_MAXLEN+MAX_IF_NAME_SIZE+2];
      sprintf(line, " %s:", bootstrapNetIfName);
      ncclSocketToString(&bootstrapNetIfAddr, line+strlen(line));
      INFO(NCCL_INIT, "Bootstrap : Using%s", line);
      bootstrapNetInitDone = 1;
    }
    pthread_mutex_unlock(&bootstrapNetLock);
  }
  return ncclSuccess;
}
```

再精读下一些util方法的实现，因为有不少的技巧可以学习，不仅仅是了解流程就足够了。

```c++
misc/socket.cc: 197

// 'ua'为出参，根据'ip_port_pair'地址匹配。
ncclResult_t ncclGetSocketAddrFromString(union ncclSocketAddress* ua, const char* ip_port_pair) {
  // ...

  bool ipv6 = ip_port_pair[0] == '[';
  /* Construct the sockaddress structure */
  if (!ipv6) {
    struct netIf ni;
    // 只解析出指定地址列表中的第一对ip+端口。
    // parse <ip_or_hostname>:<port> string, expect one pair
    if (parseStringList(ip_port_pair, &ni, 1) != 1) {
      WARN("Net : No valid <IPv4_or_hostname>:<port> pair found");
      return ncclInvalidArgument;
    }

    struct addrinfo hints, *p;
    int rv;
    memset(&hints, 0, sizeof(hints));
    // 这里hints是用作选择addrinfo的条件，设置了ai_family为未指定，ai_socktype为TCP协议。
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    // 使用getaddrinfo获取对应地址的属性，支持ipv4和ipv6，参考链接https://linux.die.net/man/3/getaddrinfo。
    // 有的时候只传了主机名，而不是具体ip等，所以需要用getaddrinfo来获取地址信息。
    // 返回的p是一个链表，如果主机名ni.prefix有多个地址可以使用，或者指定了多种协议ai_socktype，则可能有多个地址信息。
    if ( (rv = getaddrinfo(ni.prefix, NULL, &hints, &p)) != 0) {
      WARN("Net : error encountered when getting address info : %s", gai_strerror(rv));
      return ncclInvalidArgument;
    }

    // 只使用链表中的第一个地址信息设置到出参ua。
    // use the first
    if (p->ai_family == AF_INET) {
      struct sockaddr_in& sin = ua->sin;
      memcpy(&sin, p->ai_addr, sizeof(struct sockaddr_in));
      sin.sin_family = AF_INET;                        // IPv4
      //inet_pton(AF_INET, ni.prefix, &(sin.sin_addr));  // IP address
      sin.sin_port = htons(ni.port);                   // port
    } else if (p->ai_family == AF_INET6) {
      struct sockaddr_in6& sin6 = ua->sin6;
      memcpy(&sin6, p->ai_addr, sizeof(struct sockaddr_in6));
      sin6.sin6_family = AF_INET6;                     // IPv6
      sin6.sin6_port = htons(ni.port);                 // port
      sin6.sin6_flowinfo = 0;                          // needed by IPv6, but possibly obsolete
      sin6.sin6_scope_id = 0;                          // should be global scope, set to 0
    } else {
      WARN("Net : unsupported IP family");
      return ncclInvalidArgument;
    }

    freeaddrinfo(p); // all done with this structure

  } else {
    // 直接解析ipv6地址。不在此展开
  }
  return ncclSuccess;
}
```


```c++
misc/socket.cc: 157

// localAddrs为出参，可能存在多个匹配上的地址；remoteAddr为入参，用户指定地址。
int ncclFindInterfaceMatchSubnet(char* ifNames, union ncclSocketAddress* localAddrs, union ncclSocketAddress* remoteAddr, int ifNameMaxSize, int maxIfs) {
#ifdef ENABLE_TRACE
  char line[SOCKET_NAME_MAXLEN+1];
#endif
  char line_a[SOCKET_NAME_MAXLEN+1];
  int found = 0;
  // 调用内核函数获取全量网卡列表，进行网段匹配。
  struct ifaddrs *interfaces, *interface;
  getifaddrs(&interfaces);
  for (interface = interfaces; interface && !found; interface = interface->ifa_next) {
    if (interface->ifa_addr == NULL) continue;

    // 判断网卡协议，只支持IPv4和IPv6。非此两协议则匹配下一网卡。
    /* We only support IPv4 & IPv6 */
    int family = interface->ifa_addr->sa_family;
    if (family != AF_INET && family != AF_INET6)
      continue;

    // 和用户设置的网卡配置比较是否处在同一网段。具体实现不在此展开。
    // check against user specified interfaces
    if (!matchSubnet(*interface, remoteAddr)) {
      continue;
    }

    // Store the local IP address
    int salen = (family == AF_INET) ? sizeof(struct sockaddr_in) : sizeof(struct sockaddr_in6);
    // TODO: 没看到有对localAddrs分配额外地址，'found'大于0时如何copy？
    memcpy(localAddrs+found, interface->ifa_addr, salen);

    // Store the interface name
    // TODO: 同上，ifNames为bootstrapNetIfName，最大为17字节，'found'大于0时如何copy？
    strncpy(ifNames+found*ifNameMaxSize, interface->ifa_name, ifNameMaxSize);

    TRACE(NCCL_INIT|NCCL_NET,"NET : Found interface %s:%s in the same subnet as remote address %s", interface->ifa_name, ncclSocketToString(localAddrs+found, line), ncclSocketToString(remoteAddr, line_a));
    found++;
    // 大部分调用此接口的地方maxIfs设置为了1，因此查找到一个网卡后就会退出，不会发生越界问题，但代码仍存在隐患，待提交issue确认。
    if (found == maxIfs) break;
  }

  if (found == 0) {
    WARN("Net : No interface found in the same subnet as remote address %s", ncclSocketToString(remoteAddr, line_a));
  }
  freeifaddrs(interfaces);
  return found;
}
```


```c++
misc/socket.cc: 276

// 出参为ifNames和ifAddrs。查找网卡通用接口，值得细看。
int ncclFindInterfaces(char* ifNames, union ncclSocketAddress *ifAddrs, int ifNameMaxSize, int maxIfs) {
  static int shownIfName = 0;
  int nIfs = 0;
  // Allow user to force the INET socket family selection
  // 支持外部指定socket协议类型IPv4或者v6。
  int sock_family = envSocketFamily();
  // User specified interface
  // 用户指定网卡名，大部分初始化失败网络不同问题可以通过设置此环境变量解决。
  char* env = getenv("NCCL_SOCKET_IFNAME");
  if (env && strlen(env) > 1) {
    INFO(NCCL_ENV, "NCCL_SOCKET_IFNAME set by environment to %s", env);
    // Specified by user : find or fail
    if (shownIfName++ == 0) INFO(NCCL_NET, "NCCL_SOCKET_IFNAME set to %s", env);
    nIfs = findInterfaces(env, ifNames, ifAddrs, sock_family, ifNameMaxSize, maxIfs);
  } else {
    // 外部没有指定的情况下socket协议类型'sock_family'传-1。
    // 查找网卡顺序为：ib，非docker和local loop back（以太网接口等），最后才是docker和local loop back。
    // Try to automatically pick the right one
    // Start with IB
    nIfs = findInterfaces("ib", ifNames, ifAddrs, sock_family, ifNameMaxSize, maxIfs);
    // else see if we can get some hint from COMM ID
    if (nIfs == 0) {
      // 逻辑和设置了环境变量'NCCL_COMM_ID'后一致。
    }
    // Then look for anything else (but not docker or lo)
    if (nIfs == 0) nIfs = findInterfaces("^docker,lo", ifNames, ifAddrs, sock_family, ifNameMaxSize, maxIfs);
    // Finally look for docker, then lo.
    if (nIfs == 0) nIfs = findInterfaces("docker", ifNames, ifAddrs, sock_family, ifNameMaxSize, maxIfs);
    if (nIfs == 0) nIfs = findInterfaces("lo", ifNames, ifAddrs, sock_family, ifNameMaxSize, maxIfs);
  }
  return nIfs;
}
```


```c++
misc/socket.cc: 54

// names和addrs为出参。根据代表前缀的正则表达式匹配网卡名列表。
static int findInterfaces(const char* prefixList, char* names, union ncclSocketAddress *addrs, int sock_family, int maxIfNameSize, int maxIfs) {
#ifdef ENABLE_TRACE
  char line[SOCKET_NAME_MAXLEN+1];
#endif
  struct netIf userIfs[MAX_IFS];
  // '^'代表不能匹配对应前缀网卡名。
  bool searchNot = prefixList && prefixList[0] == '^';
  // const char *代表指向内容不变，指针本身可以被修改偏移。
  if (searchNot) prefixList++;
  // '='代表以'prefixList'严格匹配网卡名
  bool searchExact = prefixList && prefixList[0] == '=';
  if (searchExact) prefixList++;
  // 因为可能指定多个前缀，所以需要逐一解析前缀列表。
  int nUserIfs = parseStringList(prefixList, userIfs, MAX_IFS);

  int found = 0;
  struct ifaddrs *interfaces, *interface;
  // 遍历所有网卡。
  getifaddrs(&interfaces);
  for (interface = interfaces; interface && found < maxIfs; interface = interface->ifa_next) {
    if (interface->ifa_addr == NULL) continue;
    // 先把所有异常场景过滤后再进行匹配，一些复杂逻辑可以遵循此原则。
    /* We only support IPv4 & IPv6 */
    int family = interface->ifa_addr->sa_family;
    if (family != AF_INET && family != AF_INET6)
      continue;

    TRACE(NCCL_INIT|NCCL_NET,"Found interface %s:%s", interface->ifa_name, ncclSocketToString((union ncclSocketAddress *) interface->ifa_addr, line));

    /* Allow the caller to force the socket family type */
    // 设置-1就不匹配socket协议类型。
    if (sock_family != -1 && family != sock_family)
      continue;

    /* We also need to skip IPv6 loopback interfaces */
    if (family == AF_INET6) {
      struct sockaddr_in6* sa = (struct sockaddr_in6*)(interface->ifa_addr);
      if (IN6_IS_ADDR_LOOPBACK(&sa->sin6_addr)) continue;
    }

    // 和根据用户配置获取的网卡列表进行匹配，看是否匹配上。
    // check against user specified interfaces
    if (!(matchIfList(interface->ifa_name, -1, userIfs, nUserIfs, searchExact) ^ searchNot)) {
      continue;
    }

    // 保留匹配到的网卡，同样有'found'大于0时的越界问题。
    // Check that this interface has not already been saved
    // getifaddrs() normal order appears to be; IPv4, IPv6 Global, IPv6 Link
    bool duplicate = false;
    for (int i = 0; i < found; i++) {
      if (strcmp(interface->ifa_name, names+i*maxIfNameSize) == 0) { duplicate = true; break; }
    }

    if (!duplicate) {
      // Store the interface name
      strncpy(names+found*maxIfNameSize, interface->ifa_name, maxIfNameSize);
      // Store the IP address
      int salen = (family == AF_INET) ? sizeof(struct sockaddr_in) : sizeof(struct sockaddr_in6);
      memcpy(addrs+found, interface->ifa_addr, salen);
      found++;
    }
  }

  freeifaddrs(interfaces);
  return found;
}
```