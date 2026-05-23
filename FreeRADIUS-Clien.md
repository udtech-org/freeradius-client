# FreeRADIUS Client 完整 Code Wiki

## 目录

- [1. 项目概述](#1-项目概述)
  - [1.1 项目简介](#11-项目简介)
  - [1.2 项目结构](#12-项目结构)
- [2. 整体架构](#2-整体架构)
  - [2.1 架构图](#21-架构图)
- [3. 主要模块职责](#3-主要模块职责)
  - [3.1 核心库模块 (lib/)](#31-核心库模块-lib)
    - [3.1.1 config.c](#311-configc)
    - [3.1.2 buildreq.c](#312-buildreqc)
    - [3.1.3 sendserver.c](#313-sendserverc)
    - [3.1.4 avpair.c](#314-avpairc)
    - [3.1.5 dict.c](#315-dictc)
    - [3.1.6 md5.c](#316-md5c)
  - [3.2 工具程序 (src/)](#32-工具程序-src)
    - [3.2.1 radlogin.c](#321-radloginc)
    - [3.2.2 radacct.c](#322-radacctc)
    - [3.2.3 radexample.c](#323-radexamplec)
- [4. 关键类与函数说明](#4-关键类与函数说明)
  - [4.1 关键数据结构](#41-关键数据结构)
    - [4.1.1 rc_handle](#411-rc_handle)
    - [4.1.2 VALUE_PAIR](#412-value_pair)
    - [4.1.3 DICT_ATTR](#413-dict_attr)
  - [4.2 核心函数](#42-核心函数)
    - [4.2.1 rc_new()](#421-rc_new)
    - [4.2.2 rc_read_config()](#422-rc_read_config)
    - [4.2.3 rc_auth()](#423-rc_auth)
    - [4.2.4 rc_acct()](#424-rc_acct)
- [5. 认证、授权、计费详解](#5-认证授权计费详解)
  - [5.1 认证](#51-认证)
    - [5.1.1 支持的认证方式](#511-支持的认证方式)
    - [5.1.2 认证实现流程](#512-认证实现流程)
    - [5.1.3 认证配置](#513-认证配置)
  - [5.2 授权](#52-授权)
  - [5.3 计费](#53-计费)
    - [5.3.1 计费类型](#531-计费类型)
    - [5.3.2 计费实现](#532-计费实现)
    - [5.3.3 计费属性](#533-计费属性)
- [6. 依赖关系](#6-依赖关系)
  - [6.1 内部依赖](#61-内部依赖)
  - [6.2 外部依赖](#62-外部依赖)
- [7. 配置文件详解](#7-配置文件详解)
  - [7.1 radiusclient.conf](#71-radiusclientconf)
  - [7.2 servers文件](#72-servers文件)
  - [7.3 Dictionary文件](#73-dictionary文件)
- [8. 项目运行方式](#8-项目运行方式)
  - [8.1 编译与安装](#81-编译与安装)
  - [8.2 配置步骤](#82-配置步骤)
  - [8.3 使用示例](#83-使用示例)
- [9. 适用场景](#9-适用场景)
  - [9.1 典型应用场景](#91-典型应用场景)
  - [9.2 与FreeRADIUS Server配合](#92-与freeradius-server配合)
- [10. 常见问题与注意事项](#10-常见问题与注意事项)
  - [10.1 安全注意事项](#101-安全注意事项)
  - [10.2 调试](#102-调试)
  - [10.3 超时与重试](#103-超时与重试)
- [11. 参考资料](#11-参考资料)
- [12. 第三方应用集成指南](#12-第三方应用集成指南)
  - [12.1 概述](#121-概述)
  - [12.2 认证原理详解](#122-认证原理详解)
  - [12.3 完整交互流程](#123-完整交互流程)
  - [12.4 具体配置步骤](#124-具体配置步骤)
  - [12.5 代码实现示例](#125-代码实现示例)
  - [12.6 运行与测试](#126-运行与测试)
  - [12.7 常见问题排查](#127-常见问题排查)
  - [12.8 高级功能](#128-高级功能)
  - [12.9 附录：API参考](#129-附录api参考)
- [13. FreeRADIUS-Client工作模式详解](#13-freeradius-client工作模式详解)
  - [13.1 核心问题解答](#131-核心问题解答)
  - [13.2 工作模式概述](#132-工作模式概述)
  - [13.3 主动调用模式详解](#133-主动调用模式详解)
  - [13.4 完整调用流程](#134-完整调用流程)
  - [13.5 代码级原理分析](#135-代码级原理分析)
  - [13.6 两种模式对比](#136-两种模式对比)
  - [13.7 架构图解](#137-架构图解)
  - [13.8 实际应用示例](#138-实际应用示例)
- [14. 端口映射功能详解](#14-端口映射功能详解)
  - [14.1 端口映射的作用](#141-端口映射的作用)
  - [14.2 配置文件格式](#142-配置文件格式)
  - [14.3 工作原理](#143-工作原理)
  - [14.4 实际应用场景](#144-实际应用场景)
  - [14.5 API函数说明](#145-api函数说明)

---

## 1. 项目概述

### 1.1 项目简介
FreeRADIUS Client是一个框架和库，用于开发支持RADIUS协议的客户端应用程序。它包括一个灵活的RADIUS感知登录替换程序、发送RADIUS计费记录的命令行程序，以及查询（Merit）RADIUS服务器状态的工具。

### 1.2 项目结构
```
/workspace/
├── debian/          # Debian打包文件
├── doc/             # 文档
├── etc/             # 配置文件示例
├── include/         # 公共头文件
├── lib/             # 核心库源代码
├── login.radius/    # login.radius程序
├── man/             # 手册页
├── rpm/             # RPM打包文件
├── src/             # 工具程序源代码
└── tests/           # 测试代码和配置
```

---

## 2. 整体架构

### 2.1 架构图
```
┌─────────────────────────────────────────────────────────┐
│                      应用程序层                          │
│  radlogin  radacct  radstatus  custom applications       │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                    核心库层                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐  │
│  │ config  │  │ buildreq│  │ sendserver│ │  dict   │  │
│  └─────────┘  └─────────┘  └──────────┘  └──────────┘  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐  │
│  │ avpair  │  │ md5     │  │ ip_util │ │ util/env │  │
│  └─────────┘  └─────────┘  └─────────┘  └──────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                    网络与配置层                          │
│  ┌─────────────────┐  ┌──────────────────────────────┐ │
│  │ RADIUS服务器    │  │ 配置文件(radiusclient.conf)   │ │
│  └─────────────────┘  └──────────────────────────────┘ │
│  ┌─────────────────┐  ┌──────────────────────────────┐ │
│  │ Dictionary文件  │  │ servers文件(共享密钥)        │ │
│  └─────────────────┘  └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 3. 主要模块职责

### 3.1 核心库模块 (`lib/`)

#### 3.1.1 `config.c`
**职责**：处理配置文件读取和解析
- 从`radiusclient.conf`读取配置
- 管理服务器配置（认证和计费服务器）
- 验证配置完整性
- 提供配置查询接口

**核心数据结构**：
- `struct rc_conf`：包含所有配置项的句柄
- `struct server`：存储服务器信息
- `struct option`：配置项选项

**核心函数**：
- `rc_read_config()`：读取配置文件
- `rc_conf_str()`：获取字符串类型配置
- `rc_conf_int()`：获取整数类型配置
- `rc_conf_srv()`：获取服务器配置

#### 3.1.2 `buildreq.c`
**职责**：构建和发送RADIUS请求
- 构建认证请求
- 构建计费请求
- 处理代理认证
- 检查服务器状态

**核心函数**：
- `rc_auth()`：执行认证
- `rc_acct()`：执行计费
- `rc_auth_proxy()`：代理认证
- `rc_aaa()`：通用AAA请求处理
- `rc_check()`：检查服务器状态

#### 3.1.3 `sendserver.c`
**职责**：底层网络通信
- 将请求打包为RADIUS数据包
- 发送数据包到服务器
- 接收并验证响应
- 密码加密处理

**核心功能**：
- 使用MD5进行响应验证
- 用户密码加密（CHAP和PAP）
- 重试机制和超时处理

#### 3.1.4 `avpair.c`
**职责**：管理属性值对（Attribute-Value Pairs）
- 创建和添加AVP
- 解析和生成AVP列表
- 处理不同类型的AVP
- 从字典查找属性

**核心函数**：
- `rc_avpair_add()`：添加AVP
- `rc_avpair_new()`：创建新AVP
- `rc_avpair_free()`：释放AVP列表
- `rc_avpair_parse()`：解析AVP字符串
- `rc_avpair_tostr()`：转换为字符串

#### 3.1.5 `dict.c`
**职责**：字典文件解析
- 读取标准RADIUS字典
- 支持厂商特定属性（VSA）
- 提供属性查找功能

**字典文件格式**：
```
ATTRIBUTE    User-Name       1       string
ATTRIBUTE    User-Password   2       string
VALUE        Service-Type    Login           1
VENDOR       Cisco           9
BEGIN-VENDOR Cisco
ATTRIBUTE    Cisco-AVPair    1       string
END-VENDOR   Cisco
```

#### 3.1.6 `md5.c`
**职责**：MD5哈希计算
- 用于RADIUS安全机制
- 密码加密
- 响应验证

### 3.2 工具程序 (`src/`)

#### 3.2.1 `radlogin.c`
- 替代系统login程序的RADIUS登录工具
- 支持本地回退登录

#### 3.2.2 `radacct.c`
- 发送RADIUS计费记录
- 支持计费开始和停止

#### 3.2.3 `radexample.c`
- 使用库的简单示例
- 展示基本认证流程

---

## 4. 关键类与函数说明

### 4.1 关键数据结构

#### 4.1.1 `rc_handle` (typedef for `struct rc_conf`)
**定义位置**：`include/freeradius-client.h`

**作用**：整个库的核心句柄，包含所有配置和状态

**字段**：
```c
struct rc_conf {
    struct option         *config_options;   // 配置选项
    struct sockaddr_storage own_bind_addr;    // 绑定地址
    struct map2id_s       *map2id_list;     // 端口映射
    struct dict_attr      *dictionary_attributes; // 属性字典
    struct dict_value     *dictionary_values;    // 值字典
    struct dict_vendor    *dictionary_vendors;   // 厂商字典
    // ...
};
```

#### 4.1.2 `VALUE_PAIR`
**作用**：存储RADIUS属性值对

**字段**：
```c
typedef struct value_pair {
    char          name[NAME_LENGTH + 1];  // 属性名
    uint32_t      vendor;                 // 厂商ID
    uint32_t      attribute;              // 属性ID
    int           type;                   // 属性类型
    uint32_t      lvalue;                 // 数值
    char          strvalue[AUTH_STRING_LEN + 1]; // 字符串值
    struct value_pair *next;              // 下一个
} VALUE_PAIR;
```

#### 4.1.3 `DICT_ATTR`
**作用**：存储字典中的属性定义

### 4.2 核心函数

#### 4.2.1 `rc_new()` - 初始化库
```c
rc_handle *rc_new(void);
```
**返回值**：新的句柄，失败返回NULL

#### 4.2.2 `rc_read_config()` - 读取配置
```c
rc_handle *rc_read_config(char const *filename);
```
**参数**：配置文件路径
**返回值**：新句柄

#### 4.2.3 `rc_auth()` - 执行认证
```c
int rc_auth(rc_handle *rh, uint32_t client_port, 
            VALUE_PAIR *send, VALUE_PAIR **received, char *msg);
```
**参数**：
- `rh`：配置句柄
- `client_port`：客户端端口
- `send`：要发送的AVP
- `received`：接收的AVP
- `msg`：回复消息

**返回值**：
- `OK_RC` (0)：成功
- `REJECT_RC`：拒绝
- `ERROR_RC`：错误
- `TIMEOUT_RC`：超时

#### 4.2.4 `rc_acct()` - 发送计费
```c
int rc_acct(rc_handle *rh, uint32_t client_port, VALUE_PAIR *send);
```

---

## 5. 认证、授权、计费详解

### 5.1 认证（Authentication）

#### 5.1.1 支持的认证方式
1. **PAP（Password Authentication Protocol）**
   - 原理：用户密码被加密后发送
   - 加密：使用MD5和共享密钥
   - 流程：
     ```
     Client        RADIUS Server
       |                |
       |---- Access-Request  --->|
       |    (User-Name,         |
       |     User-Password)     |
       |<--- Access-Accept/Reject --|
       |                |
     ```

2. **CHAP（Challenge-Handshake Authentication Protocol）**
   - 原理：服务器发送挑战，客户端响应
   - 更安全，密码不直接传输

3. **EAP（Extensible Authentication Protocol）**
   - 支持多种EAP方法

#### 5.1.2 认证实现流程
```c
// 1. 初始化
rc_handle *rh = rc_read_config("/etc/radiusclient/radiusclient.conf");
rc_read_dictionary(rh, rc_conf_str(rh, "dictionary"));

// 2. 构建请求
VALUE_PAIR *send = NULL;
rc_avpair_add(rh, &send, PW_USER_NAME, "testuser", -1, 0);
rc_avpair_add(rh, &send, PW_USER_PASSWORD, "testpass", -1, 0);
rc_avpair_add(rh, &send, PW_SERVICE_TYPE, &service, -1, 0);

// 3. 发送认证
VALUE_PAIR *received = NULL;
char msg[1024];
int result = rc_auth(rh, 0, send, &received, msg);
```

#### 5.1.3 认证配置
```ini
# radiusclient.conf
auth_order       radius,local
login_tries      4
login_timeout    60
```

### 5.2 授权（Authorization）
- 通过Access-Accept中的属性实现
- 常见属性：
  - `Session-Timeout`：会话超时
  - `Idle-Timeout`：空闲超时
  - `Framed-IP-Address`：分配IP
  - `Filter-Id`：过滤器

### 5.3 计费（Accounting）

#### 5.3.1 计费类型
- **Start**：会话开始
- **Stop**：会话结束
- **Interim-Update**：定期更新
- **Accounting-On/Off**：设备状态

#### 5.3.2 计费实现
```c
VALUE_PAIR *send = NULL;
int status = PW_STATUS_START;
rc_avpair_add(rh, &send, PW_ACCT_STATUS_TYPE, &status, -1, 0);
rc_avpair_add(rh, &send, PW_ACCT_SESSION_ID, "session123", -1, 0);
rc_acct(rh, 0, send);
```

#### 5.3.3 计费属性
- `Acct-Session-Id`：会话ID
- `Acct-Input-Octets`：入流量
- `Acct-Output-Octets`：出流量
- `Acct-Session-Time`：会话时间
- `Acct-Terminate-Cause`：终止原因

---

## 6. 依赖关系

### 6.1 内部依赖
```
sendserver.c
    ├── md5.c (MD5计算)
    ├── ip_util.c (网络工具)
    └── util.c (通用工具)

buildreq.c
    ├── config.c (配置)
    ├── sendserver.c (发送)
    └── avpair.c (属性)

avpair.c
    ├── dict.c (字典)
    └── util.c (工具)
```

### 6.2 外部依赖
- C标准库
- POSIX网络库（socket, inet等）
- 无第三方库依赖（除了标准C库）

---

## 7. 配置文件详解

### 7.1 `radiusclient.conf`
位置：通常在`/etc/radiusclient/`

```ini
# ---------------------------
# 通用设置
# ---------------------------
auth_order       radius,local    # 认证顺序
login_tries      4                # 最大登录尝试
login_timeout    60               # 登录超时(秒)
nologin          /etc/nologin     # 禁止登录文件
issue            /etc/radiusclient/issue  # 登录提示

# ---------------------------
# RADIUS设置
# ---------------------------
# 认证服务器
authserver       192.168.1.10
authserver       192.168.1.11:1812  # 指定端口

# 计费服务器
acctserver       192.168.1.10
acctserver       192.168.1.11:1813

# 共享密钥文件
servers          /etc/radiusclient/servers

# 字典文件
dictionary       /etc/radiusclient/dictionary

# 登录程序
login_radius     /usr/sbin/login.radius

# 端口映射
mapfile          /etc/radiusclient/port-id-map

# 默认域
default_realm    example.com

# 超时与重试
radius_timeout   10               # 服务器超时(秒)
radius_retries   3                # 重试次数
radius_deadtime  3600             # 服务器故障时间(秒)

# 绑定地址
bindaddr         *                # 所有地址

# ---------------------------
# 本地设置
# ---------------------------
login_local      /bin/login       # 本地登录程序
```

### 7.2 `servers`文件
存储RADIUS服务器和共享密钥
```
# server           secret
192.168.1.10      testing123
192.168.1.11      secret456
```

### 7.3 Dictionary文件
定义RADIUS属性
```
ATTRIBUTE    User-Name           1    string
ATTRIBUTE    User-Password       2    string
ATTRIBUTE    NAS-IP-Address      4    ipaddr
ATTRIBUTE    NAS-Port            5    integer
ATTRIBUTE    Service-Type        6    integer

VALUE        Service-Type        Login               1
VALUE        Service-Type        Framed              2
```

---

## 8. 项目运行方式

### 8.1 编译与安装
```bash
# 配置
./configure --prefix=/usr

# 编译
make

# 安装
make install
```

### 8.2 配置步骤
1. 复制配置文件到`/etc/radiusclient/`
2. 编辑`radiusclient.conf`
3. 编辑`servers`文件添加服务器
4. 确保字典文件存在

### 8.3 使用示例

#### 8.3.1 简单认证示例
```c
#include <freeradius-client.h>

int main() {
    rc_handle *rh = rc_read_config("/etc/radiusclient/radiusclient.conf");
    rc_read_dictionary(rh, rc_conf_str(rh, "dictionary"));
    
    VALUE_PAIR *send = NULL;
    VALUE_PAIR *received = NULL;
    char msg[1024];
    uint32_t service = PW_AUTHENTICATE_ONLY;
    
    rc_avpair_add(rh, &send, PW_USER_NAME, "testuser", -1, 0);
    rc_avpair_add(rh, &send, PW_USER_PASSWORD, "testpass", -1, 0);
    rc_avpair_add(rh, &send, PW_SERVICE_TYPE, &service, -1, 0);
    
    int result = rc_auth(rh, 0, send, &received, msg);
    
    if (result == OK_RC) {
        printf("认证成功\n");
    } else {
        printf("认证失败: %d\n", result);
    }
    
    rc_avpair_free(send);
    rc_avpair_free(received);
    rc_destroy(rh);
    return result;
}
```

#### 8.3.2 编译链接
```bash
gcc -o myapp myapp.c -lfreeradius-client
```

---

## 9. 适用场景

### 9.1 典型应用场景
1. **网络接入设备**：路由器、交换机、VPN网关
2. **ISP登录系统**：宽带接入、拨号上网
3. **企业认证**：Wi-Fi、VPN登录
4. **Web应用**：结合Web服务器进行认证
5. **计费系统**：网络使用量统计和计费

### 9.2 与FreeRADIUS Server配合
```
用户 → NAS/设备 → FreeRADIUS Client → FreeRADIUS Server
                                              ↓
                                         用户数据库
```

---

## 10. 常见问题与注意事项

### 10.1 安全注意事项
1. 保护`servers`文件（权限600）
2. 使用强共享密钥
3. 启用TLS/IPsec保护通信
4. 限制服务器访问来源

### 10.2 调试
- 使用`-DDEBUG`编译启用调试
- 检查日志输出
- 使用`tcpdump`抓包分析

### 10.3 超时与重试
```c
// 调整超时和重试
// 在配置文件中设置：
radius_timeout 15
radius_retries 5
```

---

## 11. 参考资料

- [FreeRADIUS官方文档](https://freeradius.org/)
- [RFC 2865 - RADIUS](https://tools.ietf.org/html/rfc2865)
- [RFC 2866 - RADIUS Accounting](https://tools.ietf.org/html/rfc2866)

---

## 12. 第三方应用集成指南

### 12.1 概述

#### 什么是FreeRADIUS-Client？

FreeRADIUS-Client是一个开源的RADIUS客户端库，它允许第三方应用程序轻松集成RADIUS认证功能。RADIUS（Remote Authentication Dial-In User Service）是一种网络协议，用于提供集中式的用户认证、授权和计费（AAA）服务。

#### 为什么选择FreeRADIUS-Client？

1. **完整的RADIUS支持** - 实现了完整的RADIUS协议
2. **多认证方式** - 支持PAP、CHAP、EAP等多种认证方式
3. **高可定制化** - 灵活的API接口，易于集成
4. **稳定性高** - 被广泛应用于生产环境
5. **跨平台** - 支持Linux、BSD、Solaris等多种系统

---

### 12.2 认证原理详解

#### RADIUS协议核心原理

##### 1. 通信方式
- **协议**：UDP协议
- **默认端口**：
  - 认证：1812（旧标准1645）
  - 计费：1813（旧标准1646）

##### 2. 安全机制
- **共享密钥**：客户端与服务器之间预共享密钥
- **Request Authenticator**：16字节随机数，防止重放攻击
- **密码加密**：用户密码使用MD5哈希加密

##### 3. 认证流程原理

```
┌─────────────┐                    ┌───────────────┐
│  第三方应用  │                    │ RADIUS Server  │
│ (FreeRADIUS  │                    │               │
│  Client)     │                    │               │
└──────┬──────┘                    └───────┬───────┘
       │                                   │
       │  1. 生成16字节随机Request Auth    │
       │                                   │
       │  2. 使用密钥加密密码             │
       │                                   │
       │  Access-Request                  │
       │  ──────────────────────────────> │
       │  User-Name                       │
       │  User-Password(加密)             │
       │  NAS-IP-Address                  │
       │  Service-Type                    │
       │  ...                             │
       │                                   │
       │                                   │ 3. 验证用户凭证
       │                                   │
       │  Access-Accept/Reject            │
       │  <────────────────────────────── │
       │  Reply-Message                   │
       │  Session-Timeout                 │
       │  ...                             │
       │                                   │
```

#### FreeRADIUS-Client内部处理原理

##### 核心数据结构关系

```
rc_handle (配置句柄)
    │
    ├─ config_options (配置选项)
    ├─ dictionary_attributes (属性字典)
    ├─ dictionary_values (值字典)
    └─ dictionary_vendors (厂商字典)

VALUE_PAIR (属性值对链表)
    ├─ attribute (属性ID)
    ├─ type (类型: string/integer/ipaddr)
    ├─ lvalue (数值)
    ├─ strvalue (字符串值)
    └─ next (下一个节点)
```

##### 密码加密原理（PAP方式）

```c
// 伪代码展示加密过程
encrypt_password(password, secret, request_authenticator) {
    // 1. 初始向量使用Request Authenticator
    vector = request_authenticator
    
    // 2. 密码分块（每块16字节）
    password_blocks = split_into_16byte_blocks(password)
    
    // 3. 逐块加密
    for each block in password_blocks {
        hash = MD5(secret + vector)
        encrypted_block = XOR(block, hash)
        vector = encrypted_block  // 下一块使用此块作为向量
    }
    
    return encrypted_password
}
```

---

### 12.3 完整交互流程

#### 一、初始化阶段

```
步骤1: 初始化库
   │
   ▼
步骤2: 读取配置文件
   │
   ▼
步骤3: 加载字典文件
```

#### 二、认证请求构建阶段

```
步骤4: 创建属性值对(VALUE_PAIR)
   │
   ├─ 添加 User-Name
   ├─ 添加 User-Password
   ├─ 添加 Service-Type
   ├─ 添加 NAS-IP-Address
   └─ (可选) 添加其他属性
   │
   ▼
步骤5: 构建RADIUS请求包
   │
   ├─ 生成Request Authenticator
   ├─ 加密User-Password
   └─ 打包所有AVP
```

#### 三、网络传输阶段

```
步骤6: 发送Access-Request
   │
   ├─ 使用UDP协议发送
   ├─ 启动超时计时器
   └─ 支持重试机制
   │
   ▼
步骤7: 接收响应
   │
   ├─ 验证Response Authenticator
   ├─ 解析响应包
   └─ 提取返回的AVP
```

#### 四、结果处理阶段

```
步骤8: 判断认证结果
   │
   ├─ Access-Accept → 认证成功
   ├─ Access-Reject → 认证失败
   └─ Access-Challenge → 挑战响应
   │
   ▼
步骤9: 处理授权属性
   │
   ├─ Session-Timeout
   ├─ Idle-Timeout
   ├─ Framed-IP-Address
   └─ 其他授权属性
   │
   ▼
步骤10: 释放资源
```

#### 时序图

```
第三方应用          FreeRADIUS-Client        RADIUS Server
    │                    │                        │
    │── 初始化 ─────────>│                        │
    │                    │── 读配置 ──>           │
    │                    │<── 配置完成 ──         │
    │                    │── 读字典 ──>           │
    │                    │<── 字典加载 ──         │
    │                    │                        │
    │── 认证请求 ───────>│                        │
    │  (用户名/密码)     │                        │
    │                    │── 构建请求 ──>         │
    │                    │  (加密密码)            │
    │                    │── Access-Request ────>│
    │                    │                        │── 验证凭证
    │                    │                        │
    │                    │<── Access-Accept ─────│
    │                    │  (授权属性)            │
    │<── 认证成功 ───────│                        │
    │  (授权信息)        │                        │
    │                    │                        │
    │── 释放资源 ───────>│                        │
```

---

### 12.4 具体配置步骤

#### 第一步：安装FreeRADIUS-Client库

##### 从源码安装

```bash
# 1. 解压源码
tar -xzf freeradius-client-x.y.z.tar.gz
cd freeradius-client-x.y.z

# 2. 配置
./configure --prefix=/usr

# 3. 编译
make

# 4. 安装
make install
```

##### 从包管理器安装

```bash
# Debian/Ubuntu
apt-get install libradiusclient-ng-dev

# RHEL/CentOS
yum install freeradius-client-devel
```

#### 第二步：配置radiusclient.conf

创建或编辑 `/etc/radiusclient/radiusclient.conf`：

```ini
# /etc/radiusclient/radiusclient.conf

# ==================== 通用配置 ====================
# 认证顺序：先RADIUS，失败再本地
auth_order       radius,local

# 最大登录尝试次数
login_tries      4

# 登录超时时间(秒)
login_timeout    60

# 禁止登录标记文件
nologin          /etc/nologin

# 登录提示文件
issue            /etc/radiusclient/issue


# ==================== RADIUS服务器配置 ====================
# 认证服务器（可以配置多个）
authserver       192.168.1.100
authserver       192.168.1.101:1812

# 计费服务器
acctserver       192.168.1.100
acctserver       192.168.1.101:1813

# 服务器配置文件（包含共享密钥）
servers          /etc/radiusclient/servers

# 字典文件
dictionary       /etc/radiusclient/dictionary

# 登录程序
login_radius     /usr/sbin/login.radius

# 端口映射文件
mapfile          /etc/radiusclient/port-id-map

# 默认域
default_realm    example.com


# ==================== 网络与重试配置 ====================
# RADIUS服务器超时时间(秒)
radius_timeout   10

# 重试次数
radius_retries   3

# 服务器故障时的跳过时间(秒)
radius_deadtime  3600

# 绑定地址（*表示所有地址）
bindaddr         *


# ==================== 本地认证配置 ====================
# 本地登录程序
login_local      /bin/login
```

#### 第三步：配置servers文件

创建 `/etc/radiusclient/servers`：

```
# /etc/radiusclient/servers
# 格式: 服务器地址  共享密钥
# 注意：保护此文件权限为600！

192.168.1.100    mysecretradiuskey123
192.168.1.101    mysecretradiuskey123
```

设置文件权限：

```bash
chmod 600 /etc/radiusclient/servers
chown root:root /etc/radiusclient/servers
```

#### 第四步：配置字典文件

FreeRADIUS-Client自带标准字典，通常无需修改。但如果需要支持厂商特定属性（VSA），可以创建自定义字典：

```
# /etc/radiusclient/dictionary

# 包含标准字典
$INCLUDE /usr/share/freeradius-client/dictionary

# 包含厂商字典
$INCLUDE /usr/share/freeradius-client/dictionary.cisco
$INCLUDE /usr/share/freeradius-client/dictionary.microsoft

# 自定义属性（示例）
VENDOR    MyCompany    12345
BEGIN-VENDOR MyCompany
ATTRIBUTE My-App-Level   1   integer
ATTRIBUTE My-Permission  2   string
END-VENDOR MyCompany
```

#### 第五步：服务端配置（FreeRADIUS-Server示例）

在RADIUS服务器端（如FreeRADIUS），需要配置客户端：

```
# /etc/raddb/clients.conf

client my_application {
    ipaddr = 192.168.1.50
    secret = mysecretradiuskey123
    shortname = myapp
    nastype = other
}
```

配置用户（在 `/etc/raddb/users` 中）：

```
# /etc/raddb/users

testuser    Cleartext-Password := "testpass123"
            Service-Type = Login-User,
            Session-Timeout = 3600,
            Idle-Timeout = 900
```

---

### 12.5 代码实现示例

#### 示例1：最简单的认证程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <freeradius-client.h>

int main(int argc, char *argv[]) {
    rc_handle *rh;
    VALUE_PAIR *send = NULL, *received = NULL;
    char msg[PW_MAX_MSG_SIZE];
    uint32_t service;
    int result;

    // 1. 初始化并读取配置
    rh = rc_read_config("/etc/radiusclient/radiusclient.conf");
    if (!rh) {
        fprintf(stderr, "读取配置失败\n");
        return -1;
    }

    // 2. 加载字典
    if (rc_read_dictionary(rh, rc_conf_str(rh, "dictionary")) != 0) {
        fprintf(stderr, "加载字典失败\n");
        rc_destroy(rh);
        return -1;
    }

    // 3. 构建认证请求
    const char *username = "testuser";
    const char *password = "testpass123";

    // 添加用户名
    if (rc_avpair_add(rh, &send, PW_USER_NAME, 
                      (void *)username, -1, 0) == NULL) {
        goto cleanup;
    }

    // 添加密码
    if (rc_avpair_add(rh, &send, PW_USER_PASSWORD, 
                      (void *)password, -1, 0) == NULL) {
        goto cleanup;
    }

    // 添加服务类型
    service = PW_AUTHENTICATE_ONLY;
    if (rc_avpair_add(rh, &send, PW_SERVICE_TYPE, 
                      &service, -1, 0) == NULL) {
        goto cleanup;
    }

    // 4. 发送认证请求
    result = rc_auth(rh, 0, send, &received, msg);

    // 5. 处理结果
    if (result == OK_RC) {
        printf("认证成功！\n");
        printf("服务器消息: %s\n", msg);

        // 打印返回的授权属性
        printf("\n授权属性:\n");
        VALUE_PAIR *vp = received;
        while (vp) {
            char name[256], value[256];
            rc_avpair_tostr(rh, vp, name, sizeof(name), 
                            value, sizeof(value));
            printf("  %s = %s\n", name, value);
            vp = vp->next;
        }
    } else if (result == REJECT_RC) {
        printf("认证失败: 用户名或密码错误\n");
    } else if (result == TIMEOUT_RC) {
        printf("认证失败: 服务器超时\n");
    } else {
        printf("认证失败: 错误码=%d\n", result);
    }

cleanup:
    // 释放资源
    if (send) rc_avpair_free(send);
    if (received) rc_avpair_free(received);
    if (rh) rc_destroy(rh);

    return (result == OK_RC) ? 0 : 1;
}
```

#### 示例2：带计费功能的完整应用

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <freeradius-client.h>

typedef struct {
    char *username;
    char *session_id;
    rc_handle *rh;
} AuthSession;

// 初始化会话
AuthSession *session_init(const char *config_file) {
    AuthSession *sess = malloc(sizeof(AuthSession));
    if (!sess) return NULL;

    sess->rh = rc_read_config(config_file);
    if (!sess->rh) {
        free(sess);
        return NULL;
    }

    if (rc_read_dictionary(sess->rh, 
                           rc_conf_str(sess->rh, "dictionary")) != 0) {
        rc_destroy(sess->rh);
        free(sess);
        return NULL;
    }

    sess->username = NULL;
    sess->session_id = rc_mksid(sess->rh);
    return sess;
}

// 执行认证
int session_authenticate(AuthSession *sess, 
                         const char *username, 
                         const char *password) {
    VALUE_PAIR *send = NULL, *received = NULL;
    char msg[PW_MAX_MSG_SIZE];
    uint32_t service;
    int result;

    // 构建请求
    rc_avpair_add(sess->rh, &send, PW_USER_NAME, 
                  (void *)username, -1, 0);
    rc_avpair_add(sess->rh, &send, PW_USER_PASSWORD, 
                  (void *)password, -1, 0);
    
    service = PW_FRAMED_USER;
    rc_avpair_add(sess->rh, &send, PW_SERVICE_TYPE, &service, -1, 0);

    // 发送认证
    result = rc_auth(sess->rh, 0, send, &received, msg);

    if (result == OK_RC) {
        sess->username = strdup(username);
    }

    rc_avpair_free(send);
    rc_avpair_free(received);
    return result;
}

// 发送计费开始
int session_start_accounting(AuthSession *sess) {
    VALUE_PAIR *send = NULL;
    int status;

    if (!sess->username) return -1;

    // 计费开始
    status = PW_STATUS_START;
    rc_avpair_add(sess->rh, &send, PW_ACCT_STATUS_TYPE, 
                  &status, -1, 0);
    
    rc_avpair_add(sess->rh, &send, PW_USER_NAME, 
                  (void *)sess->username, -1, 0);
    
    rc_avpair_add(sess->rh, &send, PW_ACCT_SESSION_ID, 
                  sess->session_id, -1, 0);

    int result = rc_acct(sess->rh, 0, send);
    rc_avpair_free(send);
    return result;
}

// 发送计费停止
int session_stop_accounting(AuthSession *sess, 
                            uint32_t session_time,
                            uint32_t input_octets,
                            uint32_t output_octets) {
    VALUE_PAIR *send = NULL;
    int status;

    if (!sess->username) return -1;

    // 计费停止
    status = PW_STATUS_STOP;
    rc_avpair_add(sess->rh, &send, PW_ACCT_STATUS_TYPE, 
                  &status, -1, 0);
    
    rc_avpair_add(sess->rh, &send, PW_USER_NAME, 
                  (void *)sess->username, -1, 0);
    
    rc_avpair_add(sess->rh, &send, PW_ACCT_SESSION_ID, 
                  sess->session_id, -1, 0);
    
    rc_avpair_add(sess->rh, &send, PW_ACCT_SESSION_TIME, 
                  &session_time, -1, 0);
    
    rc_avpair_add(sess->rh, &send, PW_ACCT_INPUT_OCTETS, 
                  &input_octets, -1, 0);
    
    rc_avpair_add(sess->rh, &send, PW_ACCT_OUTPUT_OCTETS, 
                  &output_octets, -1, 0);

    int result = rc_acct(sess->rh, 0, send);
    rc_avpair_free(send);
    return result;
}

// 清理会话
void session_destroy(AuthSession *sess) {
    if (sess) {
        if (sess->username) free(sess->username);
        if (sess->session_id) free(sess->session_id);
        if (sess->rh) rc_destroy(sess->rh);
        free(sess);
    }
}

// 主函数
int main() {
    AuthSession *sess = session_init("/etc/radiusclient/radiusclient.conf");
    if (!sess) {
        fprintf(stderr, "初始化失败\n");
        return 1;
    }

    // 认证
    printf("正在认证...\n");
    int auth_result = session_authenticate(sess, "testuser", "testpass123");
    
    if (auth_result == OK_RC) {
        printf("认证成功！\n");
        
        // 开始计费
        printf("开始计费...\n");
        session_start_accounting(sess);
        
        // 模拟用户使用
        printf("用户使用中...\n");
        sleep(5);  // 模拟5秒使用时间
        
        // 停止计费
        printf("停止计费...\n");
        session_stop_accounting(sess, 5, 10240, 20480);
        
        printf("会话完成！\n");
    } else {
        printf("认证失败！\n");
    }

    session_destroy(sess);
    return 0;
}
```

#### 编译与链接

```bash
# 编译示例1
gcc -o simple_auth simple_auth.c -lfreeradius-client

# 编译示例2
gcc -o full_app full_app.c -lfreeradius-client
```

#### Makefile示例

```makefile
CC = gcc
CFLAGS = -Wall -O2
LDFLAGS = -lfreeradius-client

TARGETS = simple_auth full_app

all: $(TARGETS)

simple_auth: simple_auth.c
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

full_app: full_app.c
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

clean:
	rm -f $(TARGETS)
```

---

### 12.6 运行与测试

#### 第一步：测试配置

```bash
# 使用radexample测试
cd /path/to/freeradius-client/src
./radexample

# 或使用radclient工具（如果已安装）
echo "User-Name = testuser, User-Password = testpass123" | \
radclient 192.168.1.100 auth mysecretradiuskey123
```

#### 第二步：运行自定义应用

```bash
# 编译
make

# 运行
./simple_auth
```

#### 第三步：调试与日志

启用调试模式重新编译：

```bash
# 重新配置，启用调试
./configure --enable-debug
make

# 或在代码中添加
#define CP_DEBUG
```

查看系统日志：

```bash
tail -f /var/log/syslog  # Debian/Ubuntu
tail -f /var/log/messages  # RHEL/CentOS
```

使用tcpdump抓包分析：

```bash
# 抓RADIUS包
tcpdump -i any -n -v port 1812 or port 1813

# 保存到文件
tcpdump -i any -w radius.pcap port 1812
```

---

### 12.7 常见问题排查

#### 问题1：rc_read_config返回NULL

**原因**：
- 配置文件不存在
- 配置文件格式错误
- 权限不足

**解决**：
```bash
# 检查文件是否存在
ls -l /etc/radiusclient/radiusclient.conf

# 检查权限
chmod 644 /etc/radiusclient/radiusclient.conf

# 验证配置语法
cat /etc/radiusclient/radiusclient.conf
```

#### 问题2：认证总是超时

**原因**：
- 网络不通
- 服务器未运行
- 防火墙阻止
- 共享密钥不匹配

**解决**：
```bash
# 测试网络连通性
ping 192.168.1.100

# 测试端口
telnet 192.168.1.100 1812
# 或使用nc
nc -zv 192.168.1.100 1812

# 检查防火墙
iptables -L -n
```

#### 问题3：认证总是被拒绝

**原因**：
- 用户不存在
- 密码错误
- 共享密钥错误
- 服务端配置问题

**解决**：
- 检查服务端日志
- 验证共享密钥
- 测试用户凭证

#### 问题4：字典加载失败

**原因**：
- 字典文件路径错误
- 字典文件格式错误

**解决**：
```c
// 检查字典路径
printf("Dictionary path: %s\n", rc_conf_str(rh, "dictionary"));

// 验证文件存在
access(rc_conf_str(rh, "dictionary"), R_OK);
```

#### 问题5：内存泄漏

**原因**：
- 未释放VALUE_PAIR
- 未销毁rc_handle

**解决**：
```c
// 总是释放成对的资源
if (send) rc_avpair_free(send);
if (received) rc_avpair_free(received);
if (rh) rc_destroy(rh);
```

---

### 12.8 高级功能

#### CHAP认证实现

```c
// CHAP认证需要服务器发送挑战
// 以下是简化的CHAP示例

// 1. 生成CHAP ID
uint8_t chap_id = rc_get_id();

// 2. 生成挑战（或从服务器获取）
uint8_t challenge[16];
// ... 生成挑战 ...

// 3. 计算CHAP响应
uint8_t chap_response[16];
// CHAP-Password = MD5(ID + Password + Challenge)

// 4. 添加到请求
rc_avpair_add(rh, &send, PW_CHAP_PASSWORD, chap_response, 16, 0);
rc_avpair_add(rh, &send, PW_CHAP_CHALLENGE, challenge, 16, 0);
```

#### 代理认证

```c
// 使用rc_auth_proxy进行代理认证
// 这种方式不会自动添加NAS-IP等属性
int rc_auth_proxy(rc_handle *rh, VALUE_PAIR *send, 
                  VALUE_PAIR **received, char *msg);
```

#### 厂商特定属性(VSA)

```c
// 添加Cisco厂商属性
rc_avpair_add(rh, &send, PW_CISCO_AVPAIR, 
              "shell:priv-lvl=15", -1, 9);  // 9是Cisco的厂商ID
```

---

### 12.9 附录：API参考

#### A. 核心API参考

| 函数 | 说明 |
|------|------|
| `rc_new()` | 创建新的配置句柄 |
| `rc_read_config(file)` | 读取配置文件 |
| `rc_read_dictionary(rh, file)` | 加载字典文件 |
| `rc_avpair_add(rh, list, attr, val, len, vendor)` | 添加属性值对 |
| `rc_avpair_free(vp)` | 释放属性值对链表 |
| `rc_auth(rh, port, send, recv, msg)` | 执行认证 |
| `rc_acct(rh, port, send)` | 执行计费 |
| `rc_destroy(rh)` | 销毁配置句柄 |

#### B. 返回码说明

| 码值 | 常量 | 说明 |
|------|------|------|
| 0 | OK_RC | 成功 |
| 1 | TIMEOUT_RC | 超时 |
| 2 | REJECT_RC | 拒绝 |
| -1 | ERROR_RC | 错误 |
| -2 | BADRESP_RC | 响应错误 |

#### C. 常见属性列表

| 属性ID | 属性名 | 类型 | 说明 |
|--------|--------|------|------|
| 1 | User-Name | string | 用户名 |
| 2 | User-Password | string | 用户密码(加密) |
| 3 | CHAP-Password | string | CHAP密码 |
| 4 | NAS-IP-Address | ipaddr | NAS IP地址 |
| 5 | NAS-Port | integer | NAS端口 |
| 6 | Service-Type | integer | 服务类型 |
| 27 | Session-Timeout | integer | 会话超时 |
| 28 | Idle-Timeout | integer | 空闲超时 |
| 40 | Acct-Status-Type | integer | 计费状态类型 |
| 42 | Acct-Input-Octets | integer | 输入字节数 |
| 43 | Acct-Output-Octets | integer | 输出字节数 |
| 44 | Acct-Session-Id | string | 会话ID |
| 46 | Acct-Session-Time | integer | 会话时长 |

---

## 13. FreeRADIUS-Client工作模式详解

### 13.1 核心问题解答

#### 问：FreeRADIUS Client是截获第三方应用的请求，还是第三方应用主动调用它的接口？

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

### 13.2 工作模式概述

#### 13.2.1 什么是主动调用模式？

**主动调用模式**是指：第三方应用程序需要**主动调用**FreeRADIUS Client提供的API函数来发起认证、授权或计费请求。FreeRADIUS Client本身**不会主动监听或截获**任何网络流量。

#### 13.2.2 FreeRADIUS Client的定位

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

#### 13.2.3 FreeRADIUS Client是什么？

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

### 13.3 主动调用模式详解

#### 13.3.1 调用流程图

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

#### 13.3.2 与截获模式的本质区别

**主动调用模式**（FreeRADIUS Client）

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

**截获模式**（传统代理模式）

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

### 13.4 完整调用流程

#### 13.4.1 初始化阶段

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

#### 13.4.2 构建请求阶段

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

#### 13.4.3 发送请求阶段

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

### 13.5 代码级原理分析

#### 13.5.1 rc_auth() 源码分析

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

#### 13.5.2 密码加密原理

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

#### 13.5.3 网络发送原理

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

### 13.6 两种模式对比

#### 13.6.1 主动调用 vs 截获模式

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

#### 13.6.2 何时选择FreeRADIUS Client？

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

### 13.7 架构图解

#### 13.7.1 系统架构

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

#### 13.7.2 数据流架构

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

### 13.8 实际应用示例

#### 13.8.1 完整的Web应用集成示例

**场景：Web应用验证VPN用户**

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

#### 13.8.2 命令行工具集成

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

### 13.9 总结

#### 核心要点

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
