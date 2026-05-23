# FreeRADIUS Client 工作模式详解

## 目录

- [核心问题解答](#核心问题解答)
- [工作模式概述](#工作模式概述)
- [主动调用模式详解](#主动调用模式详解)
- [完整调用流程](#完整调用流程)
- [代码级原理分析](#代码级原理分析)
- [两种模式对比](#两种模式对比)
- [架构图解](#架构图解)
- [实际应用示例](#实际应用示例)

---

## 核心问题解答

### 问：FreeRADIUS Client是截获第三方应用的请求，还是第三方应用主动调用它的接口？

**答：FreeRADIUS Client是供第三方应用主动调用的客户端库，不是监听截获模式。**

**工作流程：**
```
第三方应用  ──主动调用──>  FreeRADIUS Client库  ──发送──>  RADIUS Server
    ↑                              ↑
  被动等待                      被动等待
```

**不是这样的：**
```
❌ 第三方应用 ──发送请求──>  [FreeRADIUS Client截获] ──转发──> RADIUS Server
```

---

## 工作模式概述

### 1.1 什么是主动调用模式？

**主动调用模式**是指：第三方应用程序需要**主动调用**FreeRADIUS Client提供的API函数来发起认证、授权或计费请求。FreeRADIUS Client本身**不会主动监听或截获**任何网络流量。

### 1.2 FreeRADIUS Client的定位

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   第三方应用程序（Your Application）                 │
│   - Web应用                                         │
│   - 桌面软件                                         │
│   - 嵌入式系统                                       │
│   - 网络设备                                         │
│                                                     │
│   【主动调用】 ↓ 调用API                             │
│                                                     │
│   ┌──────────────────────────────────────┐          │
│   │   FreeRADIUS Client 库                │          │
│   │   (freeradius-client.so/.a)           │          │
│   │                                       │          │
│   │   提供API接口：                       │          │
│   │   - rc_auth()  认证                  │          │
│   │   - rc_acct()  计费                  │          │
│   │   - rc_avpair_add() 添加属性         │          │
│   │                                       │          │
│   └──────────────────────────────────────┘          │
│                                                     │
│   【主动发送】 ↓ UDP网络                             │
│                                                     │
│   RADIUS Server (1812/1813端口)                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 1.3 FreeRADIUS Client是什么？

**FreeRADIUS Client是一个客户端库（Client Library）**，而不是：
- ❌ 代理服务器（Proxy）
- ❌ 守护进程（Daemon）
- ❌ 监听服务（Listener）
- ❌ 截获工具（Interceptor）

**它是一个SDK/开发库，类似于：**
- MySQL的libmysqlclient
- PostgreSQL的libpq
- HTTP的libcurl

---

## 主动调用模式详解

### 2.1 调用流程图

```
┌────────────────────────────────────────────────────────────────┐
│                     第三方应用                                  │
│                                                                │
│   1. 初始化                                                     │
│      rc_handle *rh = rc_read_config("radiusclient.conf");      │
│                     ↓                                           │
│   2. 构建请求                                                    │
│      VALUE_PAIR *send = NULL;                                   │
│      rc_avpair_add(rh, &send, PW_USER_NAME, "user", -1, 0);   │
│      rc_avpair_add(rh, &send, PW_USER_PASSWORD, "pass", -1, 0);│
│                     ↓                                           │
│   3. 调用API发送认证                                             │
│      int result = rc_auth(rh, 0, send, &received, msg);        │
│                     ↓                                           │
│   4. 处理结果                                                    │
│      if (result == OK_RC) { 认证成功 }                         │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                           ↓
                           【FreeRADIUS Client库内部处理】
                           ↓
┌────────────────────────────────────────────────────────────────┐
│                     FreeRADIUS Client                          │
│                                                                │
│   lib/buildreq.c:                                              │
│     - rc_auth() 接收应用传入的参数                              │
│     - 构建RADIUS请求包                                          │
│     - 添加NAS-IP等属性                                          │
│                                                                │
│   lib/sendserver.c:                                            │
│     - 加密密码（MD5）                                           │
│     - 生成Request Authenticator                                 │
│     - 通过UDP socket发送请求                                     │
│                                                                │
│   lib/avpair.c:                                                │
│     - 解析和打包属性值对                                         │
│     - 编码/解码属性                                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                           ↓
                           【UDP网络传输】
                           ↓
┌────────────────────────────────────────────────────────────────┐
│                   RADIUS Server                                 │
│                                                                │
│   监听 1812(认证) / 1813(计费) 端口                            │
│                                                                │
│   接收请求 → 验证 → 查询用户数据库 → 返回响应                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 与截获模式的本质区别

#### **主动调用模式**（FreeRADIUS Client）

```
时序：
T1: 第三方应用准备好认证数据
T2: 第三方应用调用 rc_auth()
T3: FreeRADIUS Client库内部处理
T4: FreeRADIUS Client发送UDP包到RADIUS Server
T5: 等待响应
T6: FreeRADIUS Client返回结果给第三方应用
T7: 第三方应用处理结果
```

**特点：**
- ✅ 第三方应用完全控制何时发送请求
- ✅ 第三方应用可以添加任意属性
- ✅ 第三方应用处理返回的授权属性
- ✅ 需要在代码中集成API调用

#### **截获模式**（传统代理模式）

```
时序：
T1: 用户发送请求到网络
T2: 代理服务截获请求
T3: 代理服务调用RADIUS认证
T4: 代理服务决定是否转发请求
T5: 返回响应给用户
```

**特点：**
- ❌ 第三方应用不知道认证过程
- ❌ 无法添加自定义属性
- ❌ 无法处理授权属性
- ❌ 需要配置网络路由或透明代理

---

## 完整调用流程

### 3.1 初始化阶段

```c
// 步骤1: 创建配置句柄
rc_handle *rh = rc_new();

// 步骤2: 读取配置文件
rh = rc_read_config("/etc/radiusclient/radiusclient.conf");
if (!rh) {
    fprintf(stderr, "配置读取失败\n");
    return ERROR;
}

// 步骤3: 加载属性字典
if (rc_read_dictionary(rh, rc_conf_str(rh, "dictionary")) != 0) {
    fprintf(stderr, "字典加载失败\n");
    return ERROR;
}

printf("初始化完成，可以调用认证API了\n");
```

**在这个阶段发生了什么？**

```
第三方应用
    ↓ 调用 rc_read_config()
FreeRADIUS Client库
    ↓ 读取配置文件
    ↓ 解析servers文件（共享密钥）
    ↓ 加载dictionary文件（属性定义）
    ↓ 返回配置句柄 rh
第三方应用
    ↓ 获得rh，可以继续调用API
```

### 3.2 构建请求阶段

```c
// 步骤4: 创建属性值对链表
VALUE_PAIR *send = NULL;
VALUE_PAIR *received = NULL;

// 步骤5: 添加必需的属性
const char *username = "testuser";
const char *password = "testpass123";

rc_avpair_add(rh, &send, PW_USER_NAME, 
              (void *)username, -1, 0);

rc_avpair_add(rh, &send, PW_USER_PASSWORD, 
              (void *)password, -1, 0);

// 步骤6: 添加可选属性
uint32_t service = PW_AUTHENTICATE_ONLY;
rc_avpair_add(rh, &send, PW_SERVICE_TYPE, 
              &service, -1, 0);

printf("请求构建完成，准备发送\n");
```

**内部数据结构：**

```
VALUE_PAIR链表结构：
┌────────────┐    ┌────────────┐    ┌────────────┐
│  Node 1    │    │  Node 2    │    │  Node 3    │
├────────────┤    ├────────────┤    ├────────────┤
│ name:      │    │ name:      │    │ name:      │
│ "User-Name"│    │ "User-     │    │ "Service-  │
│            │    │ Password"  │    │ Type"      │
├────────────┤    ├────────────┤    ├────────────┤
│ value:     │    │ value:     │    │ value:     │
│ "testuser" │    │ "testpass" │    │ 1          │
├────────────┤    ├────────────┤    ├────────────┤
│ next ──────┼───>│ next ──────┼───>│ next: NULL │
└────────────┘    └────────────┘    └────────────┘
```

### 3.3 发送请求阶段

```c
// 步骤7: 调用认证API
char msg[PW_MAX_MSG_SIZE];
int result = rc_auth(rh, 0, send, &received, msg);

// 步骤8: 处理返回结果
if (result == OK_RC) {
    printf("认证成功！\n");
    printf("服务器消息: %s\n", msg);
    
    // 处理授权属性
    VALUE_PAIR *vp = received;
    while (vp) {
        printf("属性: %s = ", vp->name);
        if (vp->type == PW_TYPE_INTEGER) {
            printf("%u\n", vp->lvalue);
        } else {
            printf("%s\n", vp->strvalue);
        }
        vp = vp->next;
    }
} else if (result == REJECT_RC) {
    printf("认证被拒绝: %s\n", msg);
} else if (result == TIMEOUT_RC) {
    printf("认证超时\n");
} else {
    printf("认证失败，错误码: %d\n", result);
}
```

**rc_auth() 函数的完整执行流程：**

```c
// 这是rc_auth()的简化伪代码
int rc_auth(rc_handle *rh, uint32_t client_port, 
            VALUE_PAIR *send, VALUE_PAIR **received, 
            char *msg) {
    
    // 1. 从配置中获取服务器信息
    SERVER *authserver = rc_conf_srv(rh, "authserver");
    char *secret = get_secret_for_server(authserver);
    
    // 2. 创建发送数据结构
    SEND_DATA data;
    data.send_pairs = send;
    data.server = authserver->addr;
    data.secret = secret;
    data.code = PW_ACCESS_REQUEST;
    
    // 3. 自动添加NAS-IP等属性（如果需要）
    add_nas_attributes(rh, &data);
    
    // 4. 发送到RADIUS服务器（内部调用sendserver.c）
    result = send_rc_request(&data, received, msg);
    
    // 5. 返回结果
    return result;
}
```

---

## 代码级原理分析

### 4.1 rc_auth() 源码分析

查看 `/workspace/lib/buildreq.c` 中的 `rc_auth()` 函数：

```c
// 简化版rc_auth()逻辑
int rc_auth(rc_handle *rh, uint32_t client_port, 
            VALUE_PAIR *send, VALUE_PAIR **received, 
            char *msg) {
    
    // 步骤1: 检查是否需要添加NAS-Port
    if (add_nas_port != 0) {
        rc_avpair_add(rh, &(data.send_pairs), 
                      PW_NAS_PORT, &client_port, 0, 0);
    }
    
    // 步骤2: 获取认证服务器和共享密钥
    SERVER *aaaserver = rc_conf_srv(rh, "authserver");
    char *secret = rc_get_server_secret(rh, aaaserver);
    
    // 步骤3: 构建请求
    rc_buildreq(rh, &data, PW_ACCESS_REQUEST,
                aaaserver->addr, 1812,
                secret, 
                rc_conf_int(rh, "radius_timeout"),
                rc_conf_int(rh, "radius_retries"));
    
    // 步骤4: 发送请求并等待响应
    result = rc_send_server(rh, &data, received, msg);
    
    return result;
}
```

### 4.2 密码加密原理

查看 `/workspace/lib/sendserver.c` 中的密码加密：

```c
// PAP密码加密伪代码
void encrypt_password(const char *password, 
                      const char *secret,
                      unsigned char *auth_vector,
                      unsigned char *output) {
    
    // PAP加密使用MD5(secret + auth_vector)
    unsigned char md5buf[256];
    size_t secretlen = strlen(secret);
    
    // 密码分块（每块16字节）
    int password_len = strlen(password);
    int padded_len = (password_len + 15) & ~15;
    
    unsigned char password_block[128] = {0};
    memcpy(password_block, password, password_len);
    
    unsigned char *vector = auth_vector;
    
    // 逐块加密
    for (int i = 0; i < padded_len; i += 16) {
        // MD5(secret + vector)
        memcpy(md5buf, secret, secretlen);
        memcpy(md5buf + secretlen, vector, 16);
        unsigned char hash[16];
        rc_md5_calc(hash, md5buf, secretlen + 16);
        
        // XOR
        for (int j = 0; j < 16; j++) {
            output[i + j] = password_block[i + j] ^ hash[j];
        }
        
        // 下一块使用当前输出作为向量
        vector = &output[i];
    }
}
```

### 4.3 网络发送原理

```c
// 简化版网络发送逻辑
int rc_send_server(rc_handle *rh, SEND_DATA *data,
                   VALUE_PAIR **received, char *msg) {
    
    // 1. 创建UDP socket
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    
    // 2. 绑定本地地址（可选）
    struct sockaddr_in local_addr;
    bind(sock, (struct sockaddr *)&local_addr, sizeof(local_addr));
    
    // 3. 准备服务器地址
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(data->svc_port);
    server_addr.sin_addr.s_addr = inet_addr(data->server);
    
    // 4. 生成随机认证向量
    unsigned char auth_vector[16];
    rc_random_vector(auth_vector);
    
    // 5. 打包请求包
    AUTH_HDR *request = malloc(MAX_PACKET_SIZE);
    build_radius_packet(request, data, auth_vector);
    
    // 6. 发送请求
    int retries = data->retries;
    while (retries > 0) {
        sendto(sock, request, request->length, 0,
                (struct sockaddr *)&server_addr, 
                sizeof(server_addr));
        
        // 7. 等待响应（带超时）
        fd_set readfds;
        FD_ZERO(&readfds);
        FD_SET(sock, &readfds);
        struct timeval timeout = {data->timeout, 0};
        
        int ready = select(sock + 1, &readfds, NULL, NULL, &timeout);
        
        if (ready > 0) {
            // 8. 接收响应
            recvfrom(sock, response, MAX_PACKET_SIZE, 0, ...);
            
            // 9. 验证响应
            if (verify_response(response, data->secret, 
                               auth_vector) == 0) {
                // 验证成功
                parse_attributes(response, received);
                close(sock);
                return OK_RC;
            }
        }
        
        retries--;
    }
    
    close(sock);
    return TIMEOUT_RC;
}
```

---

## 两种模式对比

### 5.1 主动调用 vs 截获模式

| 特性 | 主动调用（FreeRADIUS Client） | 截获模式（代理） |
|------|-----------------------------|----------------|
| **工作方式** | 应用主动调用API | 代理截获网络流量 |
| **控制权** | 应用完全控制 | 代理控制，应用不知情 |
| **代码集成** | 需要编写代码 | 不需要改应用代码 |
| **属性定制** | 可以添加任意属性 | 无法定制 |
| **响应处理** | 应用处理授权属性 | 代理处理，应用无法获得 |
| **延迟** | 低（直接调用） | 中等（多一跳） |
| **部署复杂度** | 中等（需要集成SDK） | 低（只需配置网络） |
| **适用场景** | 应用本身需要认证 | 网络设备透明认证 |

### 5.2 何时选择FreeRADIUS Client？

**适合使用FreeRADIUS Client的场景：**

```
✓ Web应用需要验证用户VPN登录
✓ 桌面软件需要验证WiFi密码
✓ 嵌入式设备需要RADIUS认证
✓ 应用程序需要获取授权属性（IP、带宽限制等）
✓ 需要完全控制认证流程
✓ 需要在一个应用内完成认证
```

**不适合使用FreeRADIUS Client的场景：**

```
✗ 需要透明截获所有HTTP请求（应该用透明代理）
✗ 无法修改现有应用代码（应该用代理网关）
✗ 只需要简单的IP限制（应该用防火墙）
```

---

## 架构图解

### 6.1 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户空间                                  │
│                                                                 │
│  ┌──────────────────┐      ┌──────────────────────────┐         │
│  │   第三方应用       │      │   FreeRADIUS Client库    │         │
│  │                  │      │                          │         │
│  │  main() {        │      │  ┌──────────────────┐   │         │
│  │    // 调用API     │──────>│  │   API层          │   │         │
│  │    rc_auth(...);  │      │  │   - rc_auth()    │   │         │
│  │  }                │      │  │   - rc_acct()    │   │         │
│  │                  │      │  │   - rc_avpair_*()│   │         │
│  │                  │      │  └────────┬─────────┘   │         │
│  │                  │      │           │             │         │
│  │                  │      │  ┌────────▼─────────┐   │         │
│  │                  │      │  │   协议层         │   │         │
│  │                  │      │  │   - buildreq    │   │         │
│  │                  │      │  │   - sendserver  │   │         │
│  │                  │      │  │   - avpair      │   │         │
│  │                  │      │  └────────┬─────────┘   │         │
│  │                  │      │           │             │         │
│  └──────────────────┘      └───────────┼─────────────┘         │
│                                         │                      │
│                                         │ UDP                  │
│                                         ↓                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                       内核空间                              ││
│  │                                                             ││
│  │   ┌─────────────────────────────────────────────────────┐   ││
│  │   │              网络协议栈                              │   ││
│  │   │   UDP → IP → Ethernet                               │   ││
│  │   └─────────────────────────────────────────────────────┘   ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                                    ↓ 网络
┌─────────────────────────────────────────────────────────────────┐
│                        外部网络                                  │
│                                                                 │
│      RADIUS Server (192.168.1.100:1812)                        │
│      - 验证用户凭证                                             │
│      - 查询用户数据库                                           │
│      - 返回授权属性                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 数据流架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    第三方应用                                    │
│                                                                 │
│  1. 用户输入: username="alice"  password="secret123"            │
│                                                                 │
│  2. 调用API:                                                    │
│     rc_avpair_add(rh, &vp, PW_USER_NAME, "alice", ...)         │
│     rc_avpair_add(rh, &vp, PW_USER_PASSWORD, "secret123", ...)  │
│                                                                 │
│  3. 调用rc_auth():                                              │
│     result = rc_auth(rh, 0, vp, &response, msg);              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓ 函数调用
┌─────────────────────────────────────────────────────────────────┐
│                 FreeRADIUS Client库                              │
│                                                                 │
│  【buildreq.c】                                                 │
│  rc_auth() ──────────────────────────────────┐                 │
│       │                                       │                 │
│       ├─> 从配置获取服务器地址                 │                 │
│       ├─> 从配置获取共享密钥                   │                 │
│       └─> 创建SEND_DATA结构                   │                 │
│                                                ↓                 │
│  【avpair.c】                                  【buildreq.c】   │
│  rc_avpair_add()                               rc_buildreq()    │
│       │                                             │           │
│       ├─> 创建VALUE_PAIR节点                       │           │
│       ├─> 分配内存                                 │           │
│       ├─> 填充属性值                               │           │
│       └─> 插入链表                               │           │
│                                                   ↓             │
│  【sendserver.c】                              【sendserver.c】 │
│  rc_send_server()                              rc_send_server()│
│       │                                             │           │
│       ├─> 生成16字节随机认证向量                     │           │
│       ├─> 打包RADIUS包                             │           │
│       ├─> PAP加密密码                             │           │
│       └─> 创建UDP socket                          │           │
│                                                   ↓             │
│  【网络层】                                                          │
│  sendto(sock, packet, length, 0,                   │             │
│          &server_addr, sizeof(server_addr))       │             │
│       │                                                           │
│       └──────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────┘
                              ↓ UDP包
┌─────────────────────────────────────────────────────────────────┐
│                    RADIUS Server                                 │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  1. 接收UDP包 (1812端口)                                  │   │
│  │  2. 验证Response Authenticator                          │   │
│  │  3. 解密User-Password                                    │   │
│  │  4. 查询用户数据库 (LDAP/SQL/files)                      │   │
│  │  5. 生成Access-Accept/Reject响应                        │   │
│  │  6. 添加授权属性 (Session-Timeout, Framed-IP等)          │   │
│  │  7. 发送UDP响应                                          │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓ UDP响应
┌─────────────────────────────────────────────────────────────────┐
│                 FreeRADIUS Client库                              │
│                                                                 │
│  【sendserver.c】                                                │
│  recvfrom() ◀──────────────────────────────────────────────────│
│       │                                                         │
│       ├─> 验证Authenticator                                     │
│       ├─> 解析属性值对                                          │
│       ├─> 返回结果码                                            │
│       └─> 填充response链表                                      │
│                                                                 │
│  【返回给应用】                                                   │
│  return OK_RC / REJECT_RC / TIMEOUT_RC                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓ 函数返回
┌─────────────────────────────────────────────────────────────────┐
│                    第三方应用                                    │
│                                                                 │
│  if (result == OK_RC) {                                         │
│      // 认证成功                                                │
│      // 处理response中的授权属性                                 │
│      VALUE_PAIR *vp = response;                                 │
│      while (vp) {                                               │
│          if (vp->attribute == PW_FRAMED_IP_ADDRESS) {          │
│              printf("分配IP: %s\n", vp->strvalue);              │
│          }                                                      │
│          vp = vp->next;                                         │
│      }                                                          │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 实际应用示例

### 7.1 完整的Web应用集成示例

#### 场景：Web应用验证VPN用户

```c
// web_auth.c - Web应用的RADIUS认证模块

#include <freeradius-client.h>
#include <httpd.h>  // Web服务器头文件
#include <mysql.h>   // 数据库头文件

typedef struct {
    char username[64];
    char password[128];
    char vpn_ip[16];
} VPNUser;

// 初始化RADIUS客户端
int init_radius_client(void) {
    rc_handle *rh = rc_new();
    
    // 读取配置
    rh = rc_read_config("/etc/radiusclient/radiusclient.conf");
    if (!rh) {
        fprintf(stderr, "RADIUS配置失败\n");
        return -1;
    }
    
    // 加载字典
    if (rc_read_dictionary(rh, 
                          rc_conf_str(rh, "dictionary")) != 0) {
        fprintf(stderr, "字典加载失败\n");
        return -1;
    }
    
    // 保存到全局或应用上下文
    g_radius_handle = rh;
    
    return 0;
}

// 验证VPN用户
int verify_vpn_user(const char *username, const char *password) {
    VALUE_PAIR *send = NULL;
    VALUE_PAIR *received = NULL;
    char msg[PW_MAX_MSG_SIZE];
    
    // 添加用户名
    rc_avpair_add(g_radius_handle, &send, PW_USER_NAME,
                  (void *)username, -1, 0);
    
    // 添加密码
    rc_avpair_add(g_radius_handle, &send, PW_USER_PASSWORD,
                  (void *)password, -1, 0);
    
    // 添加服务类型：Framed-User (PPP/VPN)
    uint32_t service = PW_FRAMED_USER;
    rc_avpair_add(g_radius_handle, &send, PW_SERVICE_TYPE,
                  &service, -1, 0);
    
    // 添加Framed协议：PPP
    uint32_t protocol = PW_PPP;
    rc_avpair_add(g_radius_handle, &send, PW_FRAMED_PROTOCOL,
                  &protocol, -1, 0);
    
    // 调用RADIUS认证
    int result = rc_auth(g_radius_handle, 0, send, 
                        &received, msg);
    
    // 释放发送数据
    rc_avpair_free(send);
    
    VPNUser user;
    memset(&user, 0, sizeof(user));
    strncpy(user.username, username, sizeof(user.username) - 1);
    
    // 处理授权属性
    VALUE_PAIR *vp = received;
    while (vp) {
        switch (vp->attribute) {
            case PW_FRAMED_IP_ADDRESS:
                // 服务器分配的IP
                inet_ntop(AF_INET, &vp->lvalue, 
                         user.vpn_ip, sizeof(user.vpn_ip));
                break;
            case PW_SESSION_TIMEOUT:
                printf("会话超时: %u秒\n", vp->lvalue);
                break;
            case PW_IDLE_TIMEOUT:
                printf("空闲超时: %u秒\n", vp->lvalue);
                break;
            case PW_REPLY_MESSAGE:
                printf("服务器消息: %s\n", vp->strvalue);
                break;
        }
        vp = vp->next;
    }
    
    rc_avpair_free(received);
    
    if (result == OK_RC) {
        // 认证成功，保存用户信息到数据库
        save_user_to_database(&user);
        return 0;
    } else if (result == REJECT_RC) {
        log_auth_failure(username, msg);
        return -1;
    } else {
        log_auth_error(username, result);
        return -1;
    }
}

// Web请求处理函数 (伪代码)
void handle_vpn_login_request(http_request_t *req) {
    const char *username = get_post_param(req, "username");
    const char *password = get_post_param(req, "password");
    
    if (!username || !password) {
        send_error_response(req, 400, "缺少参数");
        return;
    }
    
    int auth_result = verify_vpn_user(username, password);
    
    if (auth_result == 0) {
        send_json_response(req, 200, 
            "{\"status\": \"success\", "
            "\"message\": \"认证成功\", "
            "\"vpn_ip\": \"%s\"}", user.vpn_ip);
    } else {
        send_json_response(req, 401,
            "{\"status\": \"failed\", "
            "\"message\": \"认证失败\"}");
    }
}
```

### 7.2 命令行工具集成

```bash
#!/bin/bash
# 使用FreeRADIUS Client库的命令行认证工具

# 编译
gcc -o radius_auth radius_auth.c -lfreeradius-client -Wall

# 使用示例
./radius_auth testuser testpass123
```

```c
// radius_auth.c - 命令行RADIUS认证工具

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <freeradius-client.h>

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "用法: %s <用户名> <密码>\n", argv[0]);
        return 1;
    }
    
    const char *username = argv[1];
    const char *password = argv[2];
    
    // 初始化
    rc_handle *rh = rc_read_config("/etc/radiusclient/radiusclient.conf");
    if (!rh) {
        fprintf(stderr, "错误: 无法读取RADIUS配置\n");
        return 1;
    }
    
    if (rc_read_dictionary(rh, rc_conf_str(rh, "dictionary")) != 0) {
        fprintf(stderr, "错误: 无法加载字典\n");
        rc_destroy(rh);
        return 1;
    }
    
    // 构建请求
    VALUE_PAIR *send = NULL;
    rc_avpair_add(rh, &send, PW_USER_NAME, (void *)username, -1, 0);
    rc_avpair_add(rh, &send, PW_USER_PASSWORD, (void *)password, -1, 0);
    
    uint32_t service = PW_AUTHENTICATE_ONLY;
    rc_avpair_add(rh, &send, PW_SERVICE_TYPE, &service, -1, 0);
    
    // 发送认证
    VALUE_PAIR *received = NULL;
    char msg[PW_MAX_MSG_SIZE];
    
    printf("正在验证用户 %s...\n", username);
    
    int result = rc_auth(rh, 0, send, &received, msg);
    
    // 清理
    rc_avpair_free(send);
    
    // 处理结果
    switch (result) {
        case OK_RC:
            printf("✓ 认证成功!\n");
            if (msg[0]) {
                printf("服务器消息: %s\n", msg);
            }
            
            // 显示授权属性
            if (received) {
                printf("\n授权属性:\n");
                VALUE_PAIR *vp = received;
                while (vp) {
                    char name[64], value[256];
                    rc_avpair_tostr(rh, vp, name, sizeof(name),
                                   value, sizeof(value));
                    printf("  %s = %s\n", name, value);
                    vp = vp->next;
                }
            }
            rc_avpair_free(received);
            rc_destroy(rh);
            return 0;
            
        case REJECT_RC:
            printf("✗ 认证被拒绝: %s\n", msg);
            rc_destroy(rh);
            return 2;
            
        case TIMEOUT_RC:
            printf("✗ 认证超时: RADIUS服务器无响应\n");
            rc_destroy(rh);
            return 3;
            
        default:
            printf("✗ 认证失败: 错误码 %d\n", result);
            rc_destroy(rh);
            return 4;
    }
}
```

---

## 总结

### 核心要点

1. **FreeRADIUS Client是主动调用的库**
   - 不是守护进程
   - 不是代理服务
   - 不是截获工具

2. **第三方应用必须主动调用API**
   - 调用 `rc_auth()` 发起认证
   - 调用 `rc_acct()` 发起计费
   - 调用 `rc_avpair_add()` 构建属性

3. **调用流程清晰**
   ```
   应用 → 初始化 → 构建请求 → 调用API → 库处理 → 网络发送 → 等待响应 → 返回结果 → 应用处理
   ```

4. **库负责协议处理**
   - 密码加密
   - 请求打包
   - 网络通信
   - 响应验证

5. **应用负责业务逻辑**
   - 获取用户输入
   - 处理认证结果
   - 使用授权属性

---

*文档版本：1.0*
*最后更新：2026-05-23*
