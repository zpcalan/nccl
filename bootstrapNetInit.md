**bootstrapNetInit代码解读**

此方法主要初始化网络相关配置，为进程间通信提供前置条件。

```c++
bootstrap.cc:22

ncclResult_t bootstrapNetInit() {
  // 下面判断两次可重入标志bootstrapNetInitDone个人感觉没必要，一次配合加锁即可。
  if (bootstrapNetInitDone == 0) {
    pthread_mutex_lock(&bootstrapNetLock);
    if (bootstrapNetInitDone == 0) {
      // 环境变量NCCL_COMM_ID并未在官网开放，内部用于调试使用。
      // 可以做到更加精确的网络地址匹配。
      char* env = getenv("NCCL_COMM_ID");
      if (env) {
        union ncclSocketAddress remoteAddr;
        // 根据设置的环境变量NCCL_COMM_ID找到对应网络地址。
        if (ncclGetSocketAddrFromString(&remoteAddr, env) != ncclSuccess) {
          WARN("Invalid NCCL_COMM_ID, please use format: <ipv4>:<port> or [<ipv6>]:<port> or <hostname>:<port>");
          return ncclInvalidArgument;
        }
        // 根据网络地址找到对应的网卡。
        if (ncclFindInterfaceMatchSubnet(bootstrapNetIfName, &bootstrapNetIfAddr, &remoteAddr, MAX_IF_NAME_SIZE, 1) <= 0) {
          WARN("NET/Socket : No usable listening interface found");
          return ncclSystemError;
        }
      } else {
        // 根据规则查找可用网卡。返回找到的网卡数量，出参bootstrapNetIfName为找到的一个网卡名，bootstrapNetIfAddr为网卡地址。
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

int ncclFindInterfaceMatchSubnet(char* ifNames, union ncclSocketAddress* localAddrs, union ncclSocketAddress* remoteAddr, int ifNameMaxSize, int maxIfs) {
#ifdef ENABLE_TRACE
  char line[SOCKET_NAME_MAXLEN+1];
#endif
  char line_a[SOCKET_NAME_MAXLEN+1];
  int found = 0;
  // 调用内核函数获取全量网卡列表。
  struct ifaddrs *interfaces, *interface;
  getifaddrs(&interfaces);
  for (interface = interfaces; interface && !found; interface = interface->ifa_next) {
    if (interface->ifa_addr == NULL) continue;

    // 判断网卡协议，只支持IPv4和IPv6。
    int family = interface->ifa_addr->sa_family;
    if (family != AF_INET && family != AF_INET6)
      continue;

    // 和用户设置的网卡配置比较是否处在同一网段。具体实现不在此展开。
    // TODO：为什么只要匹配子网？
    if (!matchSubnet(*interface, remoteAddr)) {
      continue;
    }

    // Store the local IP address
    int salen = (family == AF_INET) ? sizeof(struct sockaddr_in) : sizeof(struct sockaddr_in6);
    memcpy(localAddrs+found, interface->ifa_addr, salen);

    // Store the interface name
    strncpy(ifNames+found*ifNameMaxSize, interface->ifa_name, ifNameMaxSize);

    TRACE(NCCL_INIT|NCCL_NET,"NET : Found interface %s:%s in the same subnet as remote address %s", interface->ifa_name, ncclSocketToString(localAddrs+found, line), ncclSocketToString(remoteAddr, line_a));
    found++;
    if (found == maxIfs) break;
  }

  if (found == 0) {
    WARN("Net : No interface found in the same subnet as remote address %s", ncclSocketToString(remoteAddr, line_a));
  }
  freeifaddrs(interfaces);
  return found;
}
```