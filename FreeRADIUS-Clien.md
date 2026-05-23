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

*文档版本：1.0*  
*最后更新：2025*
