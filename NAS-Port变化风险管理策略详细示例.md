# NAS-Port变化风险管理策略 - 详细示例

## 目录

- [策略1: 使用稳定的设备标识符](#策略1-使用稳定的设备标识符)
  - [1.1 问题分析](#11-问题分析)
  - [1.2 解决方案](#12-解决方案)
  - [1.3 完整代码示例](#13-完整代码示例)
  - [1.4 测试验证](#14-测试验证)
- [策略2: 锁定设备到端口的映射](#策略2-锁定设备到端口的映射)
  - [2.1 问题分析](#21-问题分析)
  - [2.2 解决方案](#22-解决方案)
  - [2.3 完整配置示例](#23-完整配置示例)
  - [2.4 自动化脚本](#24-自动化脚本)
- [策略3: 使用端口范围池](#策略3-使用端口范围池)
  - [3.1 问题分析](#31-问题分析)
  - [3.2 解决方案](#32-解决方案)
  - [3.3 完整配置示例](#33-完整配置示例)
  - [3.4 RADIUS策略配置](#34-radius策略配置)
- [策略4: 会话级端口绑定](#策略4-会话级端口绑定)
  - [4.1 问题分析](#41-问题分析)
  - [4.2 解决方案](#42-解决方案)
  - [4.3 完整代码示例](#43-完整代码示例)
  - [4.4 数据库设计](#44-数据库设计)
- [策略5: 端口变化监控](#策略5-端口变化监控)
  - [5.1 问题分析](#51-问题分析)
  - [5.2 解决方案](#52-解决方案)
  - [5.3 完整监控脚本](#53-完整监控脚本)
  - [5.4 告警集成](#54-告警集成)
- [策略6: 回滚机制](#策略6-回滚机制)
  - [6.1 问题分析](#61-问题分析)
  - [6.2 解决方案](#62-解决方案)
  - [6.3 完整脚本示例](#63-完整脚本示例)
- [策略7: 双映射配置](#策略7-双映射配置)
  - [7.1 问题分析](#71-问题分析)
  - [7.2 解决方案](#72-解决方案)
  - [7.3 增强的映射加载](#73-增强的映射加载)
- [综合示例: 完整的生产环境部署](#综合示例-完整的生产环境部署)

---

## 策略1: 使用稳定的设备标识符

### 1.1 问题分析

#### 问题场景

```bash
# ❌ 问题：使用TTY名称（可能变化）
const char *tty = ttyname(0);  // 返回 "/dev/ttyUSB0"
#                                              ↑
#                                              可能变成 ttyUSB1！

client_port = rc_map2id(rh, tty);  // 返回 20（如果映射存在）
```

#### 变化示例

```bash
# 初始状态
$ ls -l /dev/ttyUSB*
crw-rw---- 1 root dialout 188, 0 /dev/ttyUSB0

$ ttyname(0)
/dev/ttyUSB0

$ cat /etc/radiusclient/port-id-map
/dev/ttyUSB0    20

# USB设备重新插入（可能变成ttyUSB1）
$ ls -l /dev/ttyUSB*
crw-rw---- 1 root dialout 188, 1 /dev/ttyUSB1  # 变化了！

$ ttyname(0)
/dev/ttyUSB1  # 也变化了

# 查询映射
$ rc_map2id(rh, "/dev/ttyUSB1")
# 返回 0 (未找到！) ❌
```

### 1.2 解决方案

#### 使用稳定的标识符

```
┌─────────────────────────────────────────────────────────────┐
│                    设备标识符对比                           │
│                                                             │
│  不稳定的标识符（❌）                                       │
│  /dev/ttyUSB0      ← 可能变成 ttyUSB1                     │
│  /dev/ttyS0        ← 可能重新编号                         │
│  pts/0             ← 可能变化                             │
│                                                             │
│  稳定的标识符（✓）                                         │
│  /dev/serial/by-id/usb-Prolific_xxx-port0  ← 基于序列号    │
│  /dev/serial/by-path/pci-xxx-usb-xxx-port0  ← 基于硬件路径│
│  /dev/serial/by-path/platform-xxx              ← 固定路径  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 查看稳定标识符

```bash
# 1. 查看USB设备的序列号标识符
$ ls -l /dev/serial/by-id/
total 0
lrwxrwxrwx 1 root root  44 10月  1 10:00 
  ../by-id/usb-Prolific_Technology_Inc._USB-Serial_Controller_D123456789-if00-port0 
  -> ../../ttyUSB0
lrwxrwxrwx 1 root root  44 10月  1 10:00 
  ../by-id/usb-Silicon_Labs_CP2102_xxxxx-if00-port0 
  -> ../../ttyUSB1

# 2. 查看硬件路径标识符
$ ls -l /dev/serial/by-path/
total 0
lrwxrwxrwx 1 root root  36 10月  1 10:00 
  ../by-path/pci-0000:00:14.0-usb-0:1:1.0-port0 
  -> ../../ttyUSB0
lrwxrwxrwx 1 root root  36 10月  1 10:00 
  ../by-path/pci-0000:00:14.0-usb-0:1:1.0-port1 
  -> ../../ttyUSB1

# 3. 查看设备序列号
$ udevadm info -a -n /dev/ttyUSB0 | grep -E "(serial|SERIAL)"
    ATTR{serial}=="D123456789"
```

### 1.3 完整代码示例

#### 示例1: 使用stable-identifier的RADIUS客户端

```c
/**
 * stable_radius_client.c
 * 使用稳定的设备标识符的RADIUS客户端
 * 
 * 编译: gcc -o stable_radius_client stable_radius_client.c -lfreeradius-client -lpthread
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <freeradius-client.h>
#include <sys/stat.h>

#define CONFIG_FILE "/etc/radiusclient/radiusclient.conf"
#define DEVICE_INFO_FILE "/var/lib/radiusclient/device-info.json"

/* 设备信息结构 */
typedef struct {
    char stable_id[256];      /* 稳定的设备ID (by-id) */
    char by_path_id[256];    /* 硬件路径ID (by-path) */
    char tty_name[64];       /* 当前TTY名称 */
    uint32_t port_id;        /* NAS-Port值 */
    time_t last_updated;     /* 最后更新时间 */
} DeviceInfo;

/* 全局配置 */
static rc_handle *g_rh = NULL;
static DeviceInfo g_device_info = {0};

/**
 * 获取稳定的设备标识符
 * @return 0成功，-1失败
 */
int get_stable_device_identifier(char *stable_id, size_t stable_id_size,
                                char *by_path_id, size_t by_path_size) {
    const char *tty = ttyname(STDIN_FILENO);
    char link_buf[512];
    ssize_t len;
    
    if (!tty) {
        fprintf(stderr, "无法获取TTY名称\n");
        return -1;
    }
    
    strncpy(g_device_info.tty_name, tty, sizeof(g_device_info.tty_name) - 1);
    
    /* 尝试读取 by-id 链接 */
    snprintf(link_buf, sizeof(link_buf), "/dev/serial/by-id");
    DIR *dir = opendir(link_buf);
    if (dir) {
        struct dirent *entry;
        while ((entry = readdir(dir)) != NULL) {
            if (entry->d_type == DT_LNK) {
                /* 构建完整路径 */
                char full_link_path[512];
                char target_path[512];
                ssize_t target_len;
                
                snprintf(full_link_path, sizeof(full_link_path),
                        "%s/%s", link_buf, entry->d_name);
                
                target_len = readlink(full_link_path, target_path, sizeof(target_path) - 1);
                if (target_len > 0) {
                    target_path[target_len] = '\0';
                    
                    /* 检查是否指向当前TTY */
                    if (strstr(target_path, basename(tty)) != NULL) {
                        strncpy(stable_id, entry->d_name, stable_id_size - 1);
                        printf("找到稳定的by-id标识: %s\n", stable_id);
                        closedir(dir);
                        return 0;
                    }
                }
            }
        }
        closedir(dir);
    }
    
    /* 尝试读取 by-path 链接 */
    snprintf(link_buf, sizeof(link_buf), "/dev/serial/by-path");
    dir = opendir(link_buf);
    if (dir) {
        struct dirent *entry;
        while ((entry = readdir(dir)) != NULL) {
            if (entry->d_type == DT_LNK) {
                char full_link_path[512];
                char target_path[512];
                ssize_t target_len;
                
                snprintf(full_link_path, sizeof(full_link_path),
                        "%s/%s", link_buf, entry->d_name);
                
                target_len = readlink(full_link_path, target_path, sizeof(target_path) - 1);
                if (target_len > 0) {
                    target_path[target_len] = '\0';
                    
                    if (strstr(target_path, basename(tty)) != NULL) {
                        strncpy(by_path_id, entry->d_name, by_path_size - 1);
                        printf("找到稳定的by-path标识: %s\n", by_path_id);
                        closedir(dir);
                        return 0;
                    }
                }
            }
        }
        closedir(dir);
    }
    
    /* 都找不到，使用TTY名称作为后备 */
    strncpy(stable_id, tty, stable_id_size - 1);
    fprintf(stderr, "警告: 未找到稳定的设备标识符，使用TTY: %s\n", tty);
    return 0;
}

/**
 * 加载设备信息（从持久化存储）
 */
int load_device_info() {
    FILE *fp = fopen(DEVICE_INFO_FILE, "r");
    if (!fp) {
        return -1;
    }
    
    char line[512];
    while (fgets(line, sizeof(line), fp)) {
        char key[128], value[256];
        if (sscanf(line, "%127[^=]=%255[^\n]", key, value) == 2) {
            if (strcmp(key, "stable_id") == 0) {
                strncpy(g_device_info.stable_id, value, sizeof(g_device_info.stable_id) - 1);
            } else if (strcmp(key, "by_path_id") == 0) {
                strncpy(g_device_info.by_path_id, value, sizeof(g_device_info.by_path_id) - 1);
            } else if (strcmp(key, "tty_name") == 0) {
                strncpy(g_device_info.tty_name, value, sizeof(g_device_info.tty_name) - 1);
            } else if (strcmp(key, "port_id") == 0) {
                g_device_info.port_id = atoi(value);
            }
        }
    }
    
    fclose(fp);
    return 0;
}

/**
 * 保存设备信息（持久化）
 */
int save_device_info() {
    /* 确保目录存在 */
    mkdir("/var/lib/radiusclient", 0755);
    
    FILE *fp = fopen(DEVICE_INFO_FILE, "w");
    if (!fp) {
        fprintf(stderr, "无法保存设备信息到 %s\n", DEVICE_INFO_FILE);
        return -1;
    }
    
    fprintf(fp, "stable_id=%s\n", g_device_info.stable_id);
    fprintf(fp, "by_path_id=%s\n", g_device_info.by_path_id);
    fprintf(fp, "tty_name=%s\n", g_device_info.tty_name);
    fprintf(fp, "port_id=%u\n", g_device_info.port_id);
    fprintf(fp, "last_updated=%ld\n", time(NULL));
    
    fclose(fp);
    printf("设备信息已保存\n");
    return 0;
}

/**
 * 查询NAS-Port（优先使用稳定的标识符）
 */
uint32_t query_nas_port_using_stable_id() {
    char stable_id[256] = {0};
    char by_path_id[256] = {0};
    
    /* 1. 尝试获取稳定的设备标识符 */
    if (get_stable_device_identifier(stable_id, sizeof(stable_id),
                                     by_path_id, sizeof(by_path_id)) != 0) {
        return 0;  /* 失败返回0 */
    }
    
    /* 2. 保存稳定标识符 */
    strncpy(g_device_info.stable_id, stable_id, sizeof(g_device_info.stable_id) - 1);
    strncpy(g_device_info.by_path_id, by_path_id, sizeof(g_device_info.by_path_id) - 1);
    
    /* 3. 尝试使用stable_id查询NAS-Port */
    if (stable_id[0] != '\0') {
        uint32_t port = rc_map2id(g_rh, stable_id);
        if (port > 0) {
            printf("通过stable_id找到NAS-Port: %u\n", port);
            g_device_info.port_id = port;
            return port;
        }
    }
    
    /* 4. 尝试使用by_path_id查询 */
    if (by_path_id[0] != '\0') {
        uint32_t port = rc_map2id(g_rh, by_path_id);
        if (port > 0) {
            printf("通过by_path_id找到NAS-Port: %u\n", port);
            g_device_info.port_id = port;
            return port;
        }
    }
    
    /* 5. 最后回退到TTY名称 */
    if (g_device_info.tty_name[0] != '\0') {
        uint32_t port = rc_map2id(g_rh, g_device_info.tty_name);
        if (port > 0) {
            printf("通过tty_name找到NAS-Port: %u\n", port);
            g_device_info.port_id = port;
            return port;
        }
    }
    
    printf("警告: 所有查询方式都未找到NAS-Port，使用默认值0\n");
    return 0;
}

/**
 * 完整的认证流程
 */
int authenticate_with_stable_port(const char *username, const char *password) {
    VALUE_PAIR *send = NULL, *received = NULL;
    char msg[PW_MAX_MSG_SIZE];
    int result;
    
    /* 1. 获取NAS-Port（使用稳定的标识符） */
    uint32_t nas_port = query_nas_port_using_stable_id();
    
    /* 2. 构建认证请求 */
    rc_avpair_add(g_rh, &send, PW_USER_NAME, (void *)username, -1, 0);
    rc_avpair_add(g_rh, &send, PW_USER_PASSWORD, (void *)password, -1, 0);
    
    /* 3. 添加NAS-Port（从稳定的标识符查询） */
    if (nas_port > 0) {
        rc_avpair_add(g_rh, &send, PW_NAS_PORT, &nas_port, sizeof(nas_port), 0);
    }
    
    /* 4. 发送认证请求 */
    result = rc_auth(g_rh, nas_port, send, &received, msg);
    
    /* 5. 处理结果 */
    if (result == OK_RC) {
        printf("✓ 认证成功！\n");
        if (msg[0]) {
            printf("服务器消息: %s\n", msg);
        }
        
        /* 打印授权属性 */
        VALUE_PAIR *vp = received;
        while (vp) {
            char attr_name[64], attr_value[256];
            rc_avpair_tostr(g_rh, vp, attr_name, sizeof(attr_name),
                          attr_value, sizeof(attr_value));
            printf("  %s = %s\n", attr_name, attr_value);
            vp = vp->next;
        }
    } else if (result == REJECT_RC) {
        printf("✗ 认证被拒绝: %s\n", msg);
    } else {
        printf("✗ 认证失败: 错误码 %d\n", result);
    }
    
    /* 6. 清理资源 */
    rc_avpair_free(send);
    rc_avpair_free(received);
    
    /* 7. 保存设备信息（持久化） */
    save_device_info();
    
    return result;
}

/**
 * 主函数
 */
int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "用法: %s <用户名> <密码>\n", argv[0]);
        return 1;
    }
    
    const char *username = argv[1];
    const char *password = argv[2];
    
    /* 1. 初始化RADIUS客户端 */
    g_rh = rc_read_config(CONFIG_FILE);
    if (!g_rh) {
        fprintf(stderr, "错误: 无法读取RADIUS配置\n");
        return 1;
    }
    
    if (rc_read_dictionary(g_rh, rc_conf_str(g_rh, "dictionary")) != 0) {
        fprintf(stderr, "错误: 无法加载字典\n");
        rc_destroy(g_rh);
        return 1;
    }
    
    /* 2. 加载持久化的设备信息 */
    if (load_device_info() == 0) {
        printf("已加载之前保存的设备信息\n");
        printf("  stable_id: %s\n", g_device_info.stable_id);
        printf("  port_id: %u\n", g_device_info.port_id);
    }
    
    /* 3. 执行认证 */
    int result = authenticate_with_stable_port(username, password);
    
    /* 4. 清理 */
    rc_destroy(g_rh);
    
    return (result == OK_RC) ? 0 : 1;
}
```

### 1.4 测试验证

#### 测试脚本

```bash
#!/bin/bash
# test_stable_identifier.sh
# 测试稳定的设备标识符

echo "===== 设备标识符测试 ====="

echo ""
echo "1. 当前TTY信息:"
TTY=$(ttyname 0)
echo "   TTY: $TTY"

echo ""
echo "2. by-id 稳定标识符:"
if [ -d /dev/serial/by-id ]; then
    ls -l /dev/serial/by-id/ 2>/dev/null | grep -v "total" || echo "   无"
else
    echo "   /dev/serial/by-id 不存在"
fi

echo ""
echo "3. by-path 稳定标识符:"
if [ -d /dev/serial/by-path ]; then
    ls -l /dev/serial/by-path/ 2>/dev/null | grep -v "total" | head -5 || echo "   无"
else
    echo "   /dev/serial/by-path 不存在"
fi

echo ""
echo "4. 设备序列号:"
if command -v udevadm &> /dev/null && [ -e "$TTY" ]; then
    udevadm info -a -n "$TTY" 2>/dev/null | grep -E "ATTR{serial}|{id}" | head -3
else
    echo "   无法获取"
fi

echo ""
echo "5. 测试程序运行:"
if [ -x ./stable_radius_client ]; then
    echo "   编译成功，可以运行测试"
    echo "   用法: ./stable_radius_client <用户名> <密码>"
else
    echo "   未编译，请运行: gcc -o stable_radius_client stable_radius_client.c -lfreeradius-client"
fi

echo ""
echo "===== 测试完成 ====="
```

---

## 策略2: 锁定设备到端口的映射

### 2.1 问题分析

#### 问题场景

```bash
# port-id-map 文件
/dev/ttyUSB0    20
/dev/ttyUSB1    21

# 问题：
# 1. ttyUSB0拔出后重新插入，可能变成ttyUSB1
# 2. ttyUSB0和ttyUSB1同时插入，设备名可能互换
# 3. 系统重启后，USB枚举顺序可能变化
```

### 2.2 解决方案

#### 使用稳定的设备路径

```bash
# 使用序列号路径（非常稳定）
/dev/serial/by-id/usb-Prolific_Technology_Inc._USB-Serial_Controller_D123456789-if00-port0    20

# 使用硬件路径（稳定）
/dev/serial/by-path/pci-0000:00:14.0-usb-0:1:1.0-port0    20
```

### 2.3 完整配置示例

#### 示例配置: stable-port-id-map

```bash
# /etc/radiusclient/stable-port-id-map
# 使用稳定的设备标识符映射NAS-Port
# 自动生成，不要手动编辑

# 更新时间: 2026-05-23 10:00:00
# 生成方式: auto-generate.sh

# ===== 物理串口 (固定设备) =====
# 内置串口，通常固定不变
/dev/ttyS0    1
/dev/ttyS1    2
/dev/ttyS2    3
/dev/ttyS3    4

# ===== USB转串口 (使用by-id稳定标识符) =====
# Prolific USB转串口适配器 (序列号: D123456789)
/dev/serial/by-id/usb-Prolific_Technology_Inc._USB-Serial_Controller_D123456789-if00-port0    20

# Silicon Labs CP2102 (序列号: 0001)
/dev/serial/by-id/usb-Silicon_Labs_CP2102_USB_to_UART_Bridge_Controller_0001-if00-port0    21

# FTDI FT232R (序列号: FT12345678)
/dev/serial/by-id/usb-FTDI_FT232R_USB_UART_FT12345678-if00-port0    22

# ===== USB转串口 (使用by-path稳定标识符，作为备用) =====
# 当by-id不可用时，使用by-path
/dev/serial/by-path/pci-0000:00:14.0-usb-0:1:1.0-port0    20
/dev/serial/by-path/pci-0000:00:14.0-usb-0:1:1.0-port1    21
/dev/serial/by-path/pci-0000:00:14.0-usb-0:1:1.1-port0    22

# ===== 物理终端 =====
/dev/tty1    101
/dev/tty2    102
/dev/tty3    103
/dev/tty4    104

# ===== 虚拟终端 (用于SSH等) =====
# 注意: pts编号可能会变化，仅作为参考
# 建议使用范围策略而非具体编号
pts/0        1001
pts/1        1002
pts/2        1003
```

#### 对比: 传统配置 vs 稳定配置

```bash
# ===== 传统配置 (不稳定) =====
# /etc/radiusclient/port-id-map.legacy
/dev/ttyUSB0    20      # ❌ 可能变成ttyUSB1
/dev/ttyUSB1    21      # ❌ 可能变成ttyUSB0
pts/0           1001    # ❌ 可能变化

# ===== 稳定配置 (推荐) =====
# /etc/radiusclient/stable-port-id-map
/dev/serial/by-id/usb-Prolific_Technology_Inc._USB-Serial_Controller_D123456789-if00-port0    20  # ✓ 稳定
/dev/serial/by-id/usb-Silicon_Labs_CP2102_0001-if00-port0    21  # ✓ 稳定
pts/0           1001    # ⚠ 可能变化，但使用范围策略可缓解
```

### 2.4 自动化脚本

#### 自动生成稳定映射脚本

```bash
#!/bin/bash
# generate-stable-port-map.sh
# 自动生成稳定的port-id-map

set -e

OUTPUT_FILE="/etc/radiusclient/stable-port-id-map"
BACKUP_FILE="/etc/radiusclient/stable-port-id-map.backup"
LOG_FILE="/var/log/radiusclient/port-map-generator.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# 备份旧文件
if [ -f "$OUTPUT_FILE" ]; then
    cp "$OUTPUT_FILE" "$BACKUP_FILE"
    log "备份旧配置文件到 $BACKUP_FILE"
fi

# 开始生成
{
    echo "#"
    echo "# 稳定的端口映射文件 - 自动生成"
    echo "# 生成时间: $(date '+%Y-%m-%d %H:%M:%S')"
    echo "# 不要手动编辑此文件，修改后会被自动覆盖"
    echo "#"
    echo ""
    
    # ===== 物理串口 =====
    echo "# ===== 物理串口 (固定设备) ====="
    port_num=1
    for dev in /dev/ttyS*; do
        if [ -e "$dev" ] && [ ! -L "$dev" ]; then
            # 跳过符号链接
            echo "$dev    $port_num"
            port_num=$((port_num + 1))
        fi
    done
    echo ""
    
    # ===== USB转串口 (优先使用by-id) =====
    echo "# ===== USB转串口 (使用by-id稳定标识符) ====="
    
    # 创建临时映射表
    declare -A port_mapping
    port_num=20
    
    # 扫描by-id目录
    if [ -d /dev/serial/by-id ]; then
        for link in /dev/serial/by-id/*; do
            if [ -L "$link" ]; then
                target=$(readlink -f "$link" 2>/dev/null)
                if [[ "$target" == *ttyUSB* ]] || [[ "$target" == *ttyACM* ]]; then
                    basename_link=$(basename "$link")
                    echo "# $(basename "$target") -> $basename_link"
                    echo "$basename_link    $port_num"
                    port_num=$((port_num + 1))
                fi
            fi
        done
    fi
    
    # 如果by-id不可用，使用by-path
    if [ $port_num -eq 20 ]; then
        echo "#"
        echo "# by-id不可用，使用by-path作为备用"
        echo "#"
        
        if [ -d /dev/serial/by-path ]; then
            for link in /dev/serial/by-path/*; do
                if [ -L "$link" ]; then
                    target=$(readlink -f "$link" 2>/dev/null)
                    if [[ "$target" == *ttyUSB* ]] || [[ "$target" == *ttyACM* ]]; then
                        basename_link=$(basename "$link")
                        echo "# $(basename "$target") -> $basename_link"
                        echo "$basename_link    $port_num"
                        port_num=$((port_num + 1))
                    fi
                fi
            done
        fi
    fi
    
    echo ""
    
    # ===== 物理终端 =====
    echo "# ===== 物理终端 ====="
    port_num=101
    for dev in /dev/tty[0-9]*; do
        if [ -e "$dev" ] && [ ! -L "$dev" ]; then
            # 只处理tty1-tty63
            devname=$(basename "$dev")
            if [[ "$devname" =~ ^tty[0-9]{1,2}$ ]]; then
                echo "$dev    $port_num"
                port_num=$((port_num + 1))
            fi
        fi
    done
    echo ""
    
    # ===== 伪终端 (注释说明) =====
    echo "# ===== 伪终端 ====="
    echo "# 注意: pts编号动态变化，建议使用范围策略"
    echo "# pts/0-99 -> 范围 1000-1099"
    
} > "$OUTPUT_FILE"

# 设置权限
chmod 644 "$OUTPUT_FILE"
chown root:root "$OUTPUT_FILE"

log "成功生成稳定的port-id-map: $OUTPUT_FILE"
log "共映射了 $(grep -c '^/' "$OUTPUT_FILE") 个设备"

# 显示生成结果
echo ""
echo "===== 生成结果 ====="
grep -v '^#' "$OUTPUT_FILE" | grep -v '^$' | head -20
echo "..."
echo "总计: $(grep -c '^/' "$OUTPUT_FILE") 个设备映射"

# 验证
echo ""
echo "===== 验证映射 ====="
errors=0
while IFS=$' \t' read -r tty port; do
    if [[ "$tty" =~ ^/dev/ ]] && [ -n "$port" ]; then
        if [ -e "/dev/serial/by-id/$tty" ] || [ -e "/dev/serial/by-path/$tty" ] || [ -e "$tty" ]; then
            echo "✓ $tty -> $port"
        else
            echo "✗ $tty -> $port (设备不存在)"
            errors=$((errors + 1))
        fi
    fi
done < <(grep -v '^#' "$OUTPUT_FILE" | grep -v '^$')

if [ $errors -gt 0 ]; then
    echo ""
    echo "警告: 发现 $errors 个设备不存在"
    exit 1
else
    echo ""
    echo "✓ 所有设备映射验证通过"
fi
```

#### 设置自动更新cron

```bash
# 将以下内容添加到crontab
# 每小时自动更新一次
0 * * * * /usr/local/bin/generate-stable-port-map.sh >> /var/log/radiusclient/cron.log 2>&1

# 或者系统启动时更新
# @reboot /usr/local/bin/generate-stable-port-map.sh >> /var/log/radiusclient/cron.log 2>&1
```

---

## 策略3: 使用端口范围池

### 3.1 问题分析

#### 问题场景

```bash
# 传统策略：精确匹配
user_admin   NAS-Port == 9
             Reply-Message = "管理员权限"

# 问题：如果port-id-map中/dev/ttyS0从9变成10
# 管理员即使通过ttyS0登录，也无法获得管理员权限！
```

### 3.2 解决方案

#### 使用范围池

```bash
# 端口范围分配
范围 1-999:    固定设备 (串口、终端)
范围 1000-1999: 伪终端 (SSH、Telnet)
范围 5000-5999: VPN会话
范围 6000-6999: IoT设备
```

### 3.3 完整配置示例

#### 示例: port-id-map-with-ranges

```bash
# /etc/radiusclient/port-id-map.ranges
# 使用端口范围池的映射配置

# ===== 范围1: 固定设备 (1-999) =====
# ----- 子范围 1-99: 控制台端口 -----
/dev/tty1    1
/dev/tty2    2
/dev/tty3    3
/dev/tty4    4
/dev/tty5    5
/dev/tty6    6

# ----- 子范围 10-99: 串口端口 -----
/dev/ttyS0    10
/dev/ttyS1    11
/dev/ttyS2    12
/dev/ttyS3    13
/dev/ttyS4    14
/dev/ttyS5    15
/dev/ttyS6    16
/dev/ttyS7    17

# ----- 子范围 100-199: 固定USB设备 -----
/dev/serial/by-id/usb-Prolific_Technology_Inc._USB-Serial_Controller_D123456789-if00-port0    100
/dev/serial/by-id/usb-Silicon_Labs_CP2102_0001-if00-port0    101
/dev/serial/by-id/usb-FTDI_FT232R_FT12345678-if00-port0    102

# ===== 范围2: 动态终端 (1000-1999) =====
# 伪终端，SSH/Telnet等
# 预留范围，不在映射文件中
# pts/0-999 对应 1000-1999

# ===== 范围3: VPN会话 (5000-5999) =====
# VPN设备，动态分配
# 预留范围

# ===== 范围4: IoT设备 (6000-6999) =====
# 物联网网关
# 预留范围

# ===== 特殊说明 =====
# 伪终端映射规则:
# NAS-Port = 1000 + (pts编号)
# 例如: pts/0 -> 1000, pts/99 -> 1099
```

### 3.4 RADIUS策略配置

#### FreeRADIUS 3.x 配置

```perl
# /etc/raddb/mods-enabled/port_ranges
# 端口范围策略定义

# 定义端口范围组
group fixed_ports {
    # 固定设备: 范围 1-999
    (&NAS-Port >= 1) && (&NAS-Port <= 999)
}

group console_ports {
    # 控制台: 范围 1-6
    (&NAS-Port >= 1) && (&NAS-Port <= 6)
}

group serial_ports {
    # 串口: 范围 10-99
    (&NAS-Port >= 10) && (&NAS-Port <= 99)
}

group fixed_usb_ports {
    # 固定USB设备: 范围 100-199
    (&NAS-Port >= 100) && (&NAS-Port <= 199)
}

group dynamic_terminals {
    # 动态终端: 范围 1000-1999
    (&NAS-Port >= 1000) && (&NAS-Port <= 1999)
}

group vpn_sessions {
    # VPN会话: 范围 5000-5999
    (&NAS-Port >= 5000) && (&NAS-Port <= 5999)
}

group iot_devices {
    # IoT设备: 范围 6000-6999
    (&NAS-Port >= 6000) && (&NAS-Port <= 6999)
}
```

#### 用户策略配置

```
# /etc/raddb/users
# 基于端口范围的用户授权

# 管理员 - 可访问控制台和所有固定端口
admin_user    NAS-Port-Group := "console_ports",
              Reply-Message = "管理员权限"

# 操作员 - 可访问串口和固定USB设备
operator_user    NAS-Port-Group := "serial_ports",
                 NAS-Port-Group += "fixed_usb_ports",
                 Reply-Message = "操作员权限"

# 普通用户 - 仅可访问固定USB设备
normal_user    NAS-Port-Group := "fixed_usb_ports",
               Reply-Message = "普通用户权限"

# 远程用户 - 通过SSH/Telnet访问
remote_user    NAS-Port-Group := "dynamic_terminals",
               Reply-Message = "远程访问权限"

# VPN用户 - VPN会话
vpn_user    NAS-Port-Group := "vpn_sessions",
            Reply-Message = "VPN访问权限"
```

#### check配置文件

```ini
# /etc/raddb/check
# 基于端口范围检查

# admin_user
NAS-Port >= 1
NAS-Port <= 6
Cleartext-Password := "admin123"

# operator_user  
NAS-Port >= 10
NAS-Port <= 199
Cleartext-Password := "operator123"

# normal_user
NAS-Port >= 100
NAS-Port <= 199
Cleartext-Password := "user123"
```

#### 完整策略示例

```
# /etc/raddb/users - 完整策略示例

# ========================================
# 管理员配置 - 所有固定端口
# ========================================
admin  Cleartext-Password := "admin_secure_pass"
       NAS-Port >= 1,
       NAS-Port <= 999,
       Reply-Message = "欢迎, 管理员!",
       Service-Type = Administrative-User,
       Fall-Through = No

# ========================================
# 操作员配置 - 串口和USB设备
# ========================================
operator01  Cleartext-Password := "operator01_pass"
            NAS-Port >= 10,
            NAS-Port <= 199,
            Reply-Message = "欢迎, 操作员!",
            Service-Type = Login-User,
            Fall-Through = No

# ========================================
# 普通用户配置 - 仅固定USB设备
# ========================================
user01  Cleartext-Password := "user01_pass"
        NAS-Port >= 100,
        NAS-Port <= 199,
        Reply-Message = "欢迎, 用户!",
        Service-Type = Login-User,
        Fall-Through = No

# ========================================
# 远程用户配置 - SSH/Telnet
# ========================================
remote_user  Cleartext-Password := "remote_pass"
             NAS-Port >= 1000,
             NAS-Port <= 1999,
             Reply-Message = "远程访问已授权",
             Service-Type = Login-User,
             Fall-Through = No

# ========================================
# VPN用户配置 - VPN会话
# ========================================
vpn_user01  Cleartext-Password := "vpn_pass"
            NAS-Port >= 5000,
            NAS-Port <= 5999,
            Reply-Message = "VPN会话已建立",
            Framed-Protocol = PPP,
            Framed-IP-Address = 10.8.0.2,
            Framed-IP-Netmask = 255.255.255.0,
            Session-Timeout = 7200,
            Idle-Timeout = 600,
            Fall-Through = No

# ========================================
# 默认拒绝 - 不在范围内
# ========================================
DEFAULT  Auth-Type := Reject
         Reply-Message = "访问被拒绝: 端口不在允许范围内"
```

---

## 策略4: 会话级端口绑定

### 4.1 问题分析

```c
// 问题：认证和计费可能使用不同的NAS-Port

// 认证时
tty = ttyname(0);  // "/dev/pts/0"
port1 = rc_map2id(rh, tty);  // 101
rc_auth(rh, port1, send, ...);

// ... 设备变化 ...

// 计费时
tty = ttyname(0);  // 可能变成 "/dev/pts/1"
port2 = rc_map2id(rh, tty);  // 102 (变化了!)
rc_acct(rh, port2, send, ...);

// 结果：认证port=101，计费port=102，不匹配！
```

### 4.2 解决方案

```c
// 会话级端口绑定

// 1. 认证时分配并锁定端口
SessionContext ctx;
ctx.port_id = rc_map2id(rh, ttyname(0));  // 分配
save_session_binding(&ctx);                  // 持久化

// 2. 整个会话使用锁定的端口
rc_auth(rh, ctx.port_id, send, ...);  // 认证
rc_acct(rh, ctx.port_id, send, ...);   // 计费（使用相同端口）
```

### 4.3 完整代码示例

```c
/**
 * session_bound_radius.c
 * 会话级端口绑定的RADIUS客户端
 * 
 * 特点：
 * 1. 首次认证时分配并锁定NAS-Port
 * 2. 会话生命周期内使用相同的NAS-Port
 * 3. 持久化会话绑定关系
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
#include <pthread.h>
#include <freeradius-client.h>

#define CONFIG_FILE "/etc/radiusclient/radiusclient.conf"
#define SESSION_DB_FILE "/var/lib/radiusclient/sessions.db"

/* 会话绑定结构 */
typedef struct {
    char session_id[64];          /* 唯一会话ID */
    char username[64];            /* 用户名 */
    uint32_t port_id;             /* 锁定的NAS-Port */
    char ttyname[64];             /* 原始TTY名称 */
    char stable_id[256];          /* 稳定设备ID */
    time_t created;               /* 创建时间 */
    time_t last_activity;         /* 最后活动时间 */
    int active;                   /* 是否活跃 */
    pthread_mutex_t mutex;        /* 互斥锁 */
} SessionBinding;

/* 全局变量 */
static rc_handle *g_rh = NULL;
static FILE *g_session_db = NULL;
static pthread_mutex_t g_db_mutex = PTHREAD_MUTEX_INITIALIZER;

/**
 * 生成唯一会话ID
 */
char* generate_session_id() {
    static char session_id[64];
    struct timespec ts;
    
    clock_gettime(CLOCK_REALTIME, &ts);
    
    snprintf(session_id, sizeof(session_id),
             "%ld%03ld-%d",
             ts.tv_sec, ts.tv_nsec / 1000000,
             getpid());
    
    return session_id;
}

/**
 * 创建会话绑定
 */
SessionBinding* session_create(const char *username, const char *tty) {
    SessionBinding *session = malloc(sizeof(SessionBinding));
    if (!session) {
        return NULL;
    }
    
    /* 初始化 */
    memset(session, 0, sizeof(SessionBinding));
    strncpy(session->username, username, sizeof(session->username) - 1);
    strncpy(session->session_id, generate_session_id(), 
            sizeof(session->session_id) - 1);
    
    /* 获取TTY名称 */
    if (tty) {
        strncpy(session->ttyname, tty, sizeof(session->ttyname) - 1);
    } else {
        const char *current_tty = ttyname(STDIN_FILENO);
        if (current_tty) {
            strncpy(session->ttyname, current_tty, 
                    sizeof(session->ttyname) - 1);
        }
    }
    
    /* 查询NAS-Port */
    if (session->ttyname[0] != '\0') {
        session->port_id = rc_map2id(g_rh, session->ttyname);
    }
    
    if (session->port_id == 0) {
        /* 未找到，使用默认端口 */
        fprintf(stderr, "警告: 未找到TTY %s 的映射，使用默认端口0\n",
                session->ttyname[0] ? session->ttyname : "(unknown)");
        session->port_id = 0;
    }
    
    /* 设置时间戳 */
    session->created = time(NULL);
    session->last_activity = session->created;
    session->active = 1;
    
    /* 初始化互斥锁 */
    pthread_mutex_init(&session->mutex, NULL);
    
    printf("创建会话绑定: session_id=%s, port_id=%u, tty=%s\n",
           session->session_id, session->port_id, session->ttyname);
    
    return session;
}

/**
 * 保存会话绑定到数据库
 */
int session_save_to_db(SessionBinding *session) {
    pthread_mutex_lock(&g_db_mutex);
    
    /* 确保目录存在 */
    mkdir("/var/lib/radiusclient", 0755);
    
    FILE *fp = fopen(SESSION_DB_FILE, "a");
    if (!fp) {
        pthread_mutex_unlock(&g_db_mutex);
        return -1;
    }
    
    fprintf(fp, "%s|%s|%u|%s|%s|%ld|%ld|%d\n",
            session->session_id,
            session->username,
            session->port_id,
            session->ttyname,
            session->stable_id,
            (long)session->created,
            (long)session->last_activity,
            session->active);
    
    fclose(fp);
    
    pthread_mutex_unlock(&g_db_mutex);
    return 0;
}

/**
 * 从数据库加载会话
 */
SessionBinding* session_load_from_db(const char *session_id) {
    pthread_mutex_lock(&g_db_mutex);
    
    FILE *fp = fopen(SESSION_DB_FILE, "r");
    if (!fp) {
        pthread_mutex_unlock(&g_db_mutex);
        return NULL;
    }
    
    char line[512];
    SessionBinding *session = NULL;
    
    while (fgets(line, sizeof(line), fp)) {
        char stored_session_id[64];
        char stored_username[64];
        uint32_t stored_port_id;
        char stored_ttyname[64];
        char stored_stable_id[256];
        time_t stored_created, stored_last_activity;
        int stored_active;
        
        if (sscanf(line, "%63[^|]|%63[^|]|%u|%63[^|]|%255[^|]|%ld|%ld|%d",
                   stored_session_id,
                   stored_username,
                   &stored_port_id,
                   stored_ttyname,
                   stored_stable_id,
                   &stored_created,
                   &stored_last_activity,
                   &stored_active) == 8) {
            
            if (strcmp(stored_session_id, session_id) == 0) {
                session = malloc(sizeof(SessionBinding));
                if (session) {
                    memset(session, 0, sizeof(SessionBinding));
                    
                    strncpy(session->session_id, stored_session_id, 
                            sizeof(session->session_id) - 1);
                    strncpy(session->username, stored_username,
                            sizeof(session->username) - 1);
                    strncpy(session->ttyname, stored_ttyname,
                            sizeof(session->ttyname) - 1);
                    strncpy(session->stable_id, stored_stable_id,
                            sizeof(session->stable_id) - 1);
                    
                    session->port_id = stored_port_id;
                    session->created = stored_created;
                    session->last_activity = stored_last_activity;
                    session->active = stored_active;
                    
                    pthread_mutex_init(&session->mutex, NULL);
                }
                break;
            }
        }
    }
    
    fclose(fp);
    pthread_mutex_unlock(&g_db_mutex);
    
    return session;
}

/**
 * 更新会话活跃状态
 */
int session_update_activity(SessionBinding *session) {
    pthread_mutex_lock(&session->mutex);
    session->last_activity = time(NULL);
    pthread_mutex_unlock(&session->mutex);
    
    /* 更新数据库 */
    return session_save_to_db(session);
}

/**
 * 销毁会话
 */
void session_destroy(SessionBinding *session) {
    if (!session) return;
    
    pthread_mutex_lock(&session->mutex);
    session->active = 0;
    pthread_mutex_unlock(&session->mutex);
    
    /* 更新数据库 */
    session_save_to_db(session);
    
    /* 释放互斥锁 */
    pthread_mutex_destroy(&session->mutex);
    
    free(session);
}

/**
 * 使用锁定的端口执行认证
 * 
 * 关键点：始终使用 session->port_id，而不是重新查询
 */
int session_authenticate(SessionBinding *session,
                        const char *username,
                        const char *password) {
    VALUE_PAIR *send = NULL, *received = NULL;
    char msg[PW_MAX_MSG_SIZE];
    int result;
    
    pthread_mutex_lock(&session->mutex);
    
    /* 始终使用锁定的端口 */
    uint32_t locked_port = session->port_id;
    
    printf("使用锁定的端口进行认证: port_id=%u\n", locked_port);
    
    /* 构建认证请求 */
    rc_avpair_add(g_rh, &send, PW_USER_NAME, (void *)username, -1, 0);
    rc_avpair_add(g_rh, &send, PW_USER_PASSWORD, (void *)password, -1, 0);
    
    /* 添加NAS-Port - 使用锁定的端口，而不是重新查询 */
    rc_avpair_add(g_rh, &send, PW_NAS_PORT, 
                  &locked_port, sizeof(locked_port), 0);
    
    /* 发送认证请求 */
    result = rc_auth(g_rh, locked_port, send, &received, msg);
    
    /* 更新活跃时间 */
    session->last_activity = time(NULL);
    
    pthread_mutex_unlock(&session->mutex);
    
    /* 处理结果 */
    if (result == OK_RC) {
        printf("✓ 认证成功！\n");
        
        /* 打印授权属性 */
        VALUE_PAIR *vp = received;
        while (vp) {
            char attr_name[64], attr_value[256];
            rc_avpair_tostr(g_rh, vp, attr_name, sizeof(attr_name),
                          attr_value, sizeof(attr_value));
            printf("  %s = %s\n", attr_name, attr_value);
            vp = vp->next;
        }
    } else if (result == REJECT_RC) {
        printf("✗ 认证被拒绝: %s\n", msg);
    } else {
        printf("✗ 认证失败: 错误码 %d\n", result);
    }
    
    /* 保存会话到数据库 */
    session_save_to_db(session);
    
    /* 清理 */
    rc_avpair_free(send);
    rc_avpair_free(received);
    
    return result;
}

/**
 * 使用锁定的端口执行计费
 * 
 * 关键点：与认证使用相同的端口
 */
int session_accounting_start(SessionBinding *session) {
    VALUE_PAIR *send = NULL;
    int result;
    
    pthread_mutex_lock(&session->mutex);
    
    /* 始终使用锁定的端口 */
    uint32_t locked_port = session->port_id;
    
    printf("使用锁定的端口进行计费开始: port_id=%u\n", locked_port);
    
    /* 构建计费请求 */
    int status = PW_STATUS_START;
    rc_avpair_add(g_rh, &send, PW_ACCT_STATUS_TYPE, &status, sizeof(status), 0);
    rc_avpair_add(g_rh, &send, PW_USER_NAME, session->username, -1, 0);
    rc_avpair_add(g_rh, &send, PW_ACCT_SESSION_ID, session->session_id, -1, 0);
    
    /* 添加NAS-Port - 使用锁定的端口 */
    rc_avpair_add(g_rh, &send, PW_NAS_PORT,
                  &locked_port, sizeof(locked_port), 0);
    
    /* 更新活跃时间 */
    session->last_activity = time(NULL);
    
    pthread_mutex_unlock(&session->mutex);
    
    /* 发送计费请求 */
    result = rc_acct(g_rh, locked_port, send);
    
    if (result == OK_RC) {
        printf("✓ 计费开始成功\n");
    } else {
        printf("✗ 计费开始失败: 错误码 %d\n", result);
    }
    
    /* 保存会话到数据库 */
    session_save_to_db(session);
    
    /* 清理 */
    rc_avpair_free(send);
    
    return result;
}

/**
 * 会话计费停止
 */
int session_accounting_stop(SessionBinding *session,
                           uint32_t session_time,
                           uint32_t input_octets,
                           uint32_t output_octets) {
    VALUE_PAIR *send = NULL;
    int result;
    
    pthread_mutex_lock(&session->mutex);
    
    /* 始终使用锁定的端口 */
    uint32_t locked_port = session->port_id;
    
    printf("使用锁定的端口进行计费停止: port_id=%u\n", locked_port);
    
    /* 构建计费请求 */
    int status = PW_STATUS_STOP;
    rc_avpair_add(g_rh, &send, PW_ACCT_STATUS_TYPE, &status, sizeof(status), 0);
    rc_avpair_add(g_rh, &send, PW_USER_NAME, session->username, -1, 0);
    rc_avpair_add(g_rh, &send, PW_ACCT_SESSION_ID, session->session_id, -1, 0);
    
    /* 添加NAS-Port - 使用锁定的端口 */
    rc_avpair_add(g_rh, &send, PW_NAS_PORT,
                  &locked_port, sizeof(locked_port), 0);
    
    /* 添加会话统计 */
    rc_avpair_add(g_rh, &send, PW_ACCT_SESSION_TIME,
                  &session_time, sizeof(session_time), 0);
    rc_avpair_add(g_rh, &send, PW_ACCT_INPUT_OCTETS,
                  &input_octets, sizeof(input_octets), 0);
    rc_avpair_add(g_rh, &send, PW_ACCT_OUTPUT_OCTETS,
                  &output_octets, sizeof(output_octets), 0);
    
    pthread_mutex_unlock(&session->mutex);
    
    /* 发送计费请求 */
    result = rc_acct(g_rh, locked_port, send);
    
    if (result == OK_RC) {
        printf("✓ 计费停止成功\n");
    } else {
        printf("✗ 计费停止失败: 错误码 %d\n", result);
    }
    
    /* 清理 */
    rc_avpair_free(send);
    
    return result;
}

/**
 * 完整会话流程演示
 */
int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "用法: %s <用户名> <密码>\n", argv[0]);
        return 1;
    }
    
    const char *username = argv[1];
    const char *password = argv[2];
    
    /* 1. 初始化RADIUS客户端 */
    g_rh = rc_read_config(CONFIG_FILE);
    if (!g_rh) {
        fprintf(stderr, "错误: 无法读取RADIUS配置\n");
        return 1;
    }
    
    if (rc_read_dictionary(g_rh, rc_conf_str(g_rh, "dictionary")) != 0) {
        fprintf(stderr, "错误: 无法加载字典\n");
        rc_destroy(g_rh);
        return 1;
    }
    
    /* 2. 创建会话绑定（分配并锁定端口） */
    SessionBinding *session = session_create(username, NULL);
    if (!session) {
        fprintf(stderr, "错误: 无法创建会话\n");
        rc_destroy(g_rh);
        return 1;
    }
    
    /* 3. 保存会话到数据库 */
    session_save_to_db(session);
    
    /* 4. 执行认证（使用锁定的端口） */
    int auth_result = session_authenticate(session, username, password);
    
    if (auth_result == OK_RC) {
        /* 5. 计费开始 */
        session_accounting_start(session);
        
        /* 6. 模拟会话活动 */
        printf("\n会话进行中...\n");
        printf("会话ID: %s\n", session->session_id);
        printf("锁定的端口: %u\n", session->port_id);
        
        /* 7. 模拟会话结束 */
        printf("\n会话结束...\n");
        
        /* 8. 计费停止 */
        session_accounting_stop(session, 300, 10240, 20480);
    }
    
    /* 9. 销毁会话 */
    session_destroy(session);
    
    /* 10. 清理 */
    rc_destroy(g_rh);
    
    return (auth_result == OK_RC) ? 0 : 1;
}
```

### 4.4 数据库设计

```sql
-- session_bindings.sql
-- 会话绑定数据库表设计

-- 会话绑定表
CREATE TABLE IF NOT EXISTS session_bindings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id VARCHAR(64) UNIQUE NOT NULL,
    username VARCHAR(64) NOT NULL,
    port_id INTEGER NOT NULL,
    ttyname VARCHAR(64),
    stable_id VARCHAR(256),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_activity TIMESTAMP,
    terminated_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'active',
    
    -- 索引
    INDEX idx_username (username),
    INDEX idx_port_id (port_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- 会话事件表
CREATE TABLE IF NOT EXISTS session_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (session_id) REFERENCES session_bindings(session_id),
    INDEX idx_session_id (session_id),
    INDEX idx_event_type (event_type)
);

-- 示例查询

-- 1. 查询用户的活动会话
SELECT * FROM session_bindings 
WHERE username = 'user01' 
AND status = 'active';

-- 2. 查询特定端口的活动会话
SELECT * FROM session_bindings 
WHERE port_id = 101 
AND status = 'active';

-- 3. 统计每个端口的使用情况
SELECT port_id, COUNT(*) as session_count 
FROM session_bindings 
GROUP BY port_id 
ORDER BY session_count DESC;

-- 4. 查询会话历史
SELECT 
    sb.session_id,
    sb.username,
    sb.port_id,
    sb.ttyname,
    sb.created_at,
    sb.terminated_at,
    se.event_type,
    se.event_data
FROM session_bindings sb
LEFT JOIN session_events se ON sb.session_id = se.session_id
WHERE sb.username = 'user01'
ORDER BY sb.created_at DESC;
```

---

## 策略5: 端口变化监控

### 5.1 问题分析

```
问题：
1. port-id-map文件被修改
2. 设备插入/拔出导致映射变化
3. 系统配置变更
4. 映射文件损坏或格式错误

影响：
- 认证失败
- 计费错误
- 审计追踪困难
```

### 5.2 解决方案

```
┌──────────────────────────────────────────────┐
│          端口映射监控系统架构                  │
│                                              │
│  ┌─────────────┐                            │
│  │ 定时检查    │ ──每5分钟──┐                │
│  └─────────────┘           │                │
│                           ▼                │
│  ┌─────────────┐    ┌─────────────┐        │
│  │ 计算校验和  │───>│ 对比上次值  │        │
│  └─────────────┘    └──────┬──────┘        │
│                             │                │
│              ┌──────────────┴──────────────┐│
│              │                             ││
│              ▼                             ▼│
│        ┌─────────┐                  ┌─────────┐│
│        │ 变化    │                  │ 未变化  ││
│        └────┬────┘                  └────┬────┘│
│             │                            │     │
│             ▼                            ▼     │
│       ┌──────────┐                 (忽略)     │
│       │ 告警通知 │                            │
│       └────┬─────┘                            │
│            │                                  │
│            ▼                                  │
│       ┌──────────┐                           │
│       │ 记录日志 │                           │
│       └──────────┘                           │
│                                              │
└──────────────────────────────────────────────┘
```

### 5.3 完整监控脚本

```bash
#!/bin/bash
# monitor-port-map.sh
# 端口映射监控脚本
# 
# 功能：
# 1. 检测port-id-map文件变化
# 2. 检测设备可用性
# 3. 验证映射一致性
# 4. 发送告警通知

set -e

# ===== 配置 =====
CONFIG_FILE="/etc/radiusclient/radiusclient.conf"
PORT_MAP_FILE="/etc/radiusclient/stable-port-id-map"
CHECKSUM_FILE="/var/lib/radiusclient/port-map-checksum.sha256"
STATE_FILE="/var/lib/radiusclient/port-map-state.json"
LOG_DIR="/var/log/radiusclient"
LOG_FILE="$LOG_DIR/port-monitor.log"
ALERT_SCRIPT="/usr/local/bin/port-map-alert.sh"

# 检查间隔（秒）
CHECK_INTERVAL=300  # 5分钟

# 最大历史记录
MAX_HISTORY=100

# ===== 初始化 =====
init() {
    mkdir -p "$LOG_DIR"
    mkdir -p "$(dirname "$CHECKSUM_FILE")"
    
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] 监控初始化" >> "$LOG_FILE"
}

# ===== 日志记录 =====
log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" >> "$LOG_FILE"
}

# ===== 计算文件校验和 =====
calculate_checksum() {
    if [ -f "$PORT_MAP_FILE" ]; then
        sha256sum "$PORT_MAP_FILE" | awk '{print $1}'
    else
        echo "FILE_NOT_FOUND"
    fi
}

# ===== 加载上次状态 =====
load_last_state() {
    if [ -f "$CHECKSUM_FILE" ]; then
        cat "$CHECKSUM_FILE"
    else
        echo ""
    fi
}

# ===== 保存当前状态 =====
save_current_state() {
    calculate_checksum > "$CHECKSUM_FILE"
}

# ===== 检测文件变化 =====
check_file_change() {
    local current_sum=$(calculate_checksum)
    local last_sum=$(load_last_state)
    
    if [ "$current_sum" != "$last_sum" ]; then
        if [ -n "$last_sum" ]; then
            log "WARN" "检测到port-id-map文件变化"
            log "WARN" "  旧校验和: $last_sum"
            log "WARN" "  新校验和: $current_sum"
            return 1  # 变化
        else
            log "INFO" "首次运行，初始化校验和: $current_sum"
            save_current_state
            return 0  # 首次，无变化
        fi
    fi
    
    return 0  # 无变化
}

# ===== 检测设备可用性 =====
check_device_availability() {
    local errors=0
    local warnings=0
    
    log "INFO" "检查设备可用性..."
    
    while IFS=$' \t' read -r device port; do
        # 跳过注释和空行
        [[ "$device" =~ ^# ]] && continue
        [[ -z "$device" ]] && continue
        
        # 检查设备是否存在
        if [[ "$device" =~ ^/dev/ ]]; then
            if [ ! -e "$device" ]; then
                # 尝试构建完整路径
                if [ -e "/dev/serial/by-id/$device" ]; then
                    log "INFO" "设备 $device 可通过by-id访问"
                elif [ -e "/dev/serial/by-path/$device" ]; then
                    log "INFO" "设备 $device 可通过by-path访问"
                else
                    log "ERROR" "设备不存在: $device -> port $port"
                    errors=$((errors + 1))
                fi
            else
                log "DEBUG" "设备正常: $device -> port $port"
            fi
        elif [[ "$device" =~ ^pci- ]] || [[ "$device" =~ ^usb- ]]; then
            # by-path设备
            if [ -e "/dev/serial/by-path/$device" ]; then
                log "DEBUG" "设备正常: /dev/serial/by-path/$device -> port $port"
            elif [ -e "/dev/serial/by-id/$device" ]; then
                log "DEBUG" "设备正常: /dev/serial/by-id/$device -> port $port"
            else
                log "ERROR" "设备不存在: $device -> port $port"
                errors=$((errors + 1))
            fi
        fi
    done < <(grep -v '^#' "$PORT_MAP_FILE" | grep -v '^$')
    
    return $errors
}

# ===== 验证映射一致性 =====
verify_mapping_consistency() {
    log "INFO" "验证映射一致性..."
    
    local ports=()
    local duplicates=()
    
    while IFS=$' \t' read -r device port; do
        [[ "$device" =~ ^# ]] && continue
        [[ -z "$device" ]] && continue
        [[ -z "$port" ]] && continue
        
        # 检查端口重复
        for p in "${ports[@]}"; do
            if [ "$p" == "$port" ]; then
                log "ERROR" "发现重复端口: $port (设备: $device)"
                duplicates+=("$port")
            fi
        done
        ports+=("$port")
        
    done < <(grep -v '^#' "$PORT_MAP_FILE" | grep -v '^$')
    
    if [ ${#duplicates[@]} -gt 0 ]; then
        log "ERROR" "发现 ${#duplicates[@]} 个重复端口"
        return 1
    fi
    
    log "INFO" "映射一致性验证通过"
    return 0
}

# ===== 生成状态报告 =====
generate_status_report() {
    local report_file="$STATE_FILE"
    
    cat > "$report_file" << EOF
{
    "timestamp": "$(date -Iseconds)",
    "checksum": "$(calculate_checksum)",
    "device_count": $(grep -c '^/' "$PORT_MAP_FILE" 2>/dev/null || echo 0),
    "last_check": "$(date '+%Y-%m-%d %H:%M:%S')",
    "monitor_status": "running"
}
EOF
    
    log "INFO" "状态报告已生成: $report_file"
}

# ===== 发送告警 =====
send_alert() {
    local alert_type=$1
    local message=$2
    
    log "ALERT" "发送告警: $alert_type - $message"
    
    # 调用告警脚本（如果存在）
    if [ -x "$ALERT_SCRIPT" ]; then
        "$ALERT_SCRIPT" "$alert_type" "$message"
    else
        # 默认告警方式：写入专用告警文件
        local alert_file="$LOG_DIR/port-map-alerts.log"
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$alert_type] $message" >> "$alert_file"
    fi
}

# ===== 显示帮助 =====
show_help() {
    cat << EOF
用法: $(basename "$0") [选项]

选项:
    -c, --check          执行单次检查
    -d, --daemon         以守护进程模式运行
    -v, --verbose        详细输出
    -h, --help           显示帮助

示例:
    $(basename "$0") -c              # 单次检查
    $(basename "$0") -d              # 守护进程模式
    $(basename "$0") -c -v          # 详细输出
EOF
}

# ===== 单次检查 =====
run_check() {
    local failed=0
    
    log "INFO" "===== 开始端口映射检查 ====="
    
    # 1. 检查文件变化
    if ! check_file_change; then
        send_alert "FILE_CHANGE" "port-id-map文件已变更"
        failed=1
    else
        log "INFO" "文件无变化"
    fi
    
    # 2. 检查设备可用性
    if ! check_device_availability; then
        send_alert "DEVICE_UNAVAILABLE" "部分设备不可用"
        failed=1
    fi
    
    # 3. 验证映射一致性
    if ! verify_mapping_consistency; then
        send_alert "MAPPING_ERROR" "映射配置存在错误"
        failed=1
    fi
    
    # 4. 生成状态报告
    generate_status_report
    
    # 5. 保存当前状态
    save_current_state
    
    log "INFO" "===== 检查完成 ====="
    
    if [ $failed -eq 0 ]; then
        log "INFO" "所有检查通过"
        return 0
    else
        log "ERROR" "部分检查失败"
        return 1
    fi
}

# ===== 守护进程模式 =====
run_daemon() {
    log "INFO" "启动守护进程模式，检查间隔: ${CHECK_INTERVAL}秒"
    
    while true; do
        run_check
        sleep "$CHECK_INTERVAL"
    done
}

# ===== 主函数 =====
main() {
    init
    
    # 解析参数
    case "${1:-}" in
        -c|--check)
            run_check
            ;;
        -d|--daemon)
            run_daemon
            ;;
        -v|--verbose)
            export DEBUG=1
            run_check
            ;;
        -h|--help)
            show_help
            exit 0
            ;;
        *)
            echo "用法: $(basename "$0") [-c|--check] [-d|--daemon] [-v|--verbose] [-h|--help]"
            echo "运行 '$(basename "$0") -h' 查看帮助"
            exit 1
            ;;
    esac
}

main "$@"
```

#### 告警脚本

```bash
#!/bin/bash
# /usr/local/bin/port-map-alert.sh
# 端口映射告警通知脚本

ALERT_TYPE=$1
MESSAGE=$2

# 日志
echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$ALERT_TYPE] $MESSAGE" >> /var/log/radiusclient/alerts.log

# 邮件告警（如果配置）
if [ -n "$ALERT_EMAIL" ]; then
    echo "$MESSAGE" | mail -s "[RADIUS告警] $ALERT_TYPE" "$ALERT_EMAIL"
fi

# Webhook告警（如果配置）
if [ -n "$WEBHOOK_URL" ]; then
    curl -X POST "$WEBHOOK_URL" \
         -H "Content-Type: application/json" \
         -d "{\"alert_type\":\"$ALERT_TYPE\",\"message\":\"$MESSAGE\",\"timestamp\":\"$(date -Iseconds)\"}" \
         2>/dev/null
fi

# 日志告警（如果配置）
if [ -n "$SYSLOG_HOST" ]; then
    logger -t "port-monitor" -p user.warn "$ALERT_TYPE: $MESSAGE"
fi
```

### 5.4 告警集成

#### Cron配置

```bash
# /etc/cron.d/radius-port-monitor
# 端口映射监控cron配置

# 每5分钟检查一次
*/5 * * * * root /usr/local/bin/monitor-port-map.sh -c >> /var/log/radiusclient/cron.log 2>&1

# 每天凌晨3点生成报告
0 3 * * * root /usr/local/bin/port-map-report.sh >> /var/log/radiusclient/report.log 2>&1
```

#### systemd服务配置

```ini
# /etc/systemd/system/port-monitor.service
[Unit]
Description=Port Mapping Monitor
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/monitor-port-map.sh -d
Restart=always
RestartSec=10

User=root
Group=root

# 日志配置
StandardOutput=append:/var/log/radiusclient/port-monitor-stdout.log
StandardError=append:/var/log/radiusclient/port-monitor-stderr.log

[Install]
WantedBy=multi-user.target
```

```bash
# 启用服务
systemctl enable port-monitor
systemctl start port-monitor
systemctl status port-monitor
```

---

## 策略6: 回滚机制

### 6.1 问题分析

```
问题：
1. port-id-map被误修改
2. 自动生成脚本出错
3. 配置变更导致认证失败

需要：
1. 自动备份
2. 版本控制
3. 一键回滚
```

### 6.2 解决方案

```
备份策略：
- 每次修改前自动备份
- 保留多个历史版本
- 按时间自动清理旧备份
```

### 6.3 完整脚本示例

```bash
#!/bin/bash
# /usr/local/bin/port-map-manager.sh
# 端口映射配置管理器
# 
# 功能：
# 1. 自动备份
# 2. 版本控制
# 3. 一键回滚
# 4. 配置验证

set -e

# ===== 配置 =====
PORT_MAP_FILE="/etc/radiusclient/stable-port-id-map"
BACKUP_DIR="/var/backups/radiusclient/port-maps"
MAX_BACKUPS=10
DATE_FORMAT="%Y%m%d_%H%M%S"

# ===== 初始化 =====
init() {
    mkdir -p "$BACKUP_DIR"
}

# ===== 日志 =====
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# ===== 备份当前配置 =====
backup() {
    local description="${1:-manual_backup}"
    local timestamp=$(date +"$DATE_FORMAT")
    local backup_file="$BACKUP_DIR/${timestamp}_${description}.map"
    
    if [ -f "$PORT_MAP_FILE" ]; then
        cp "$PORT_MAP_FILE" "$backup_file"
        
        # 计算并保存校验和
        sha256sum "$backup_file" > "${backup_file}.sha256"
        
        log "✓ 已备份到: $backup_file"
        
        # 清理旧备份
        cleanup_old_backups
        
        return 0
    else
        log "✗ 源文件不存在: $PORT_MAP_FILE"
        return 1
    fi
}

# ===== 列出备份 =====
list_backups() {
    log "===== 可用的备份 ====="
    
    local count=0
    for backup in "$BACKUP_DIR"/*.map; do
        if [ -f "$backup" ]; then
            local filename=$(basename "$backup")
            local size=$(stat -f%z "$backup" 2>/dev/null || stat -c%s "$backup")
            local modified=$(stat -f%Sm "$backup" 2>/dev/null || stat -c%y "$backup" | cut -d' ' -f1)
            
            echo "  [$count] $filename"
            echo "      大小: $size 字节"
            echo "      修改: $modified"
            echo ""
            
            count=$((count + 1))
        fi
    done
    
    if [ $count -eq 0 ]; then
        echo "  无可用备份"
    fi
    
    echo "总计: $count 个备份"
}

# ===== 回滚到指定版本 =====
rollback() {
    local backup_index=${1:-0}
    
    # 获取备份文件
    local backups=("$BACKUP_DIR"/*.map)
    local backup_file="${backups[$backup_index]}"
    
    if [ ! -f "$backup_file" ]; then
        log "✗ 无效的备份索引: $backup_index"
        list_backups
        return 1
    fi
    
    # 验证备份完整性
    if [ -f "${backup_file}.sha256" ]; then
        if ! sha256sum -c "${backup_file}.sha256" > /dev/null 2>&1; then
            log "✗ 备份文件校验失败: $backup_file"
            return 1
        fi
        log "✓ 备份文件校验通过"
    fi
    
    # 创建回滚前备份
    backup "pre_rollback"
    
    # 执行回滚
    cp "$backup_file" "$PORT_MAP_FILE"
    
    log "✓ 已回滚到: $(basename "$backup_file")"
    log "  当前配置已备份为回滚前版本"
}

# ===== 清理旧备份 =====
cleanup_old_backups() {
    local backups=("$BACKUP_DIR"/*.map)
    local count=${#backups[@]}
    
    if [ $count -gt $MAX_BACKUPS ]; then
        log "清理旧备份（保留最近 $MAX_BACKUPS 个）..."
        
        # 按修改时间排序，删除最旧的
        while [ $count -gt $MAX_BACKUPS ]; do
            local oldest="${backups[0]}"
            rm -f "$oldest" "${oldest}.sha256"
            log "  删除: $(basename "$oldest")"
            
            # 重新获取列表
            backups=("$BACKUP_DIR"/*.map)
            count=${#backups[@]}
        done
    fi
}

# ===== 验证配置 =====
verify() {
    log "===== 验证配置 ====="
    
    local errors=0
    
    # 检查文件是否存在
    if [ ! -f "$PORT_MAP_FILE" ]; then
        log "✗ 配置文件不存在: $PORT_MAP_FILE"
        return 1
    fi
    
    # 检查格式
    while IFS=$' \t' read -r line; do
        [[ "$line" =~ ^# ]] && continue
        [[ -z "$line" ]] && continue
        
        # 提取设备和端口
        device=$(echo "$line" | awk '{print $1}')
        port=$(echo "$line" | awk '{print $2}')
        
        if [ -z "$device" ] || [ -z "$port" ]; then
            log "✗ 格式错误: $line"
            errors=$((errors + 1))
        fi
    done < "$PORT_MAP_FILE"
    
    # 检查重复端口
    local ports=$(grep -v '^#' "$PORT_MAP_FILE" | grep -v '^$' | awk '{print $2}' | sort)
    local duplicates=$(echo "$ports" | uniq -d)
    
    if [ -n "$duplicates" ]; then
        log "✗ 发现重复端口: $duplicates"
        errors=$((errors + 1))
    fi
    
    if [ $errors -eq 0 ]; then
        log "✓ 配置验证通过"
        return 0
    else
        log "✗ 发现 $errors 个错误"
        return 1
    fi
}

# ===== 比较配置 =====
diff_backups() {
    local index1=${1:-0}
    local index2=${2:-$((index1 + 1))}
    
    local backups=("$BACKUP_DIR"/*.map)
    local file1="${backups[$index1]}"
    local file2="${backups[$index2]}"
    
    if [ ! -f "$file1" ] || [ ! -f "$file2" ]; then
        log "✗ 无效的备份索引"
        return 1
    fi
    
    log "比较: $(basename "$file1") vs $(basename "$file2")"
    diff -u "$file1" "$file2" || true
}

# ===== 显示帮助 =====
show_help() {
    cat << EOF
端口映射配置管理器

用法: $(basename "$0") <命令> [参数]

命令:
    backup [描述]       备份当前配置
                        示例: $(basename "$0") backup "修改前"
    
    list               列出所有备份
    
    rollback [索引]    回滚到指定版本
                        示例: $(basename "$0") rollback 0
    
    verify             验证当前配置
    
    diff [索引1] [索引2]  比较两个备份
                        示例: $(basename "$0") diff 0 1
    
    restore <文件>     从指定文件恢复
                        示例: $(basename "$0") restore /path/to/backup.map

示例:
    $(basename "$0") backup                        # 创建备份
    $(basename "$0") list                          # 查看备份
    $(basename "$0") rollback 0                    # 回滚到最新备份
    $(basename "$0") verify                        # 验证配置
EOF
}

# ===== 主函数 =====
main() {
    init
    
    local command=${1:-}
    shift
    
    case "$command" in
        backup)
            backup "$@"
            ;;
        list)
            list_backups
            ;;
        rollback)
            rollback "$@"
            ;;
        verify)
            verify
            ;;
        diff)
            diff_backups "$@"
            ;;
        restore)
            if [ -z "$1" ]; then
                echo "错误: 请指定备份文件"
                exit 1
            fi
            backup "pre_manual_restore"
            cp "$1" "$PORT_MAP_FILE"
            log "✓ 已恢复: $1"
            ;;
        help|--help|-h)
            show_help
            ;;
        *)
            if [ -n "$command" ]; then
                echo "未知命令: $command"
            fi
            show_help
            exit 1
            ;;
    esac
}

main "$@"
```

---

## 策略7: 双映射配置

### 7.1 问题分析

```
问题：
1. 同一个物理设备可能有多个标识符
2. by-id和by-path可能指向同一设备
3. 需要灵活的映射关系

解决方案：
支持别名映射，一个端口可以对应多个设备标识符
```

### 7.2 解决方案

```bash
# 双映射配置示例
# 格式: 主设备 别名1 别名2 ... -> 端口

# 主设备（by-id）指向端口20
/dev/serial/by-id/usb-Prolific_xxx-port0 -> 20
# 别名1（by-path）也指向端口20
/dev/serial/by-path/pci-xxx-usb-xxx-port0 -> 20
# 别名2（ttyUSB）也指向端口20
/dev/ttyUSB0 -> 20
```

### 7.3 增强的映射加载

```c
/**
 * enhanced_mapfile.c
 * 增强的端口映射加载器
 * 
 * 支持功能：
 * 1. 别名映射
 * 2. 自动别名生成
 * 3. 优先级配置
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>
#include <freeradius-client.h>

/* 别名映射结构 */
typedef struct {
    char *device;           /* 设备标识符 */
    uint32_t port_id;       /* 端口ID */
    int is_primary;         /* 是否主映射 */
    struct alias_s *next;   /* 指向下一个别名 */
} AliasMapping;

/* 别名组结构 */
typedef struct {
    uint32_t port_id;       /* 端口ID */
    AliasMapping *primary;   /* 主映射 */
    AliasMapping *aliases;  /* 别名列表 */
    struct alias_group_s *next;
} AliasGroup;

/* 全局别名组列表 */
static AliasGroup *g_alias_groups = NULL;

/**
 * 解析别名映射行
 * 格式: device1 [device2] [device3] ... -> port_id
 * 或:   device1|device2|device3 -> port_id
 */
int parse_alias_line(const char *line, char ***devices, uint32_t *port_id) {
    static char *dev_list[32];
    int dev_count = 0;
    
    /* 跳过注释和空行 */
    if (line[0] == '#' || line[0] == '\0' || line[0] == '\n') {
        return 0;
    }
    
    /* 查找端口分隔符 */
    const char *port_sep = strstr(line, "->");
    if (!port_sep) {
        /* 标准格式: device port_id */
        char buffer[256];
        strncpy(buffer, line, sizeof(buffer) - 1);
        
        char *device = strtok(buffer, " \t\n");
        char *port_str = strtok(NULL, " \t\n");
        
        if (device && port_str) {
            dev_list[dev_count++] = strdup(device);
            *port_id = atoi(port_str);
        }
    } else {
        /* 别名格式 */
        char buffer[512];
        strncpy(buffer, line, sizeof(buffer) - 1);
        
        char *devices_part = strtok(buffer, "->");
        char *port_str = strtok(NULL, "->");
        
        if (devices_part && port_str) {
            /* 解析设备列表 */
            char *device = strtok(devices_part, " \t|");
            while (device && dev_count < 32) {
                dev_list[dev_count++] = strdup(device);
                device = strtok(NULL, " \t|");
            }
            
            /* 解析端口 */
            *port_id = atoi(port_str);
        }
    }
    
    /* 复制设备列表 */
    *devices = malloc(sizeof(char*) * dev_count);
    for (int i = 0; i < dev_count; i++) {
        (*devices)[i] = dev_list[i];
    }
    
    return dev_count;
}

/**
 * 加载增强映射文件
 */
int load_enhanced_mapfile(rc_handle *rh, const char *filename) {
    FILE *fp = fopen(filename, "r");
    if (!fp) {
        fprintf(stderr, "无法打开映射文件: %s\n", filename);
        return -1;
    }
    
    char line[512];
    AliasGroup *last_group = NULL;
    
    while (fgets(line, sizeof(line), fp)) {
        /* 去除换行符 */
        line[strcspn(line, "\n")] = 0;
        
        /* 解析行 */
        char **devices = NULL;
        uint32_t port_id = 0;
        int dev_count = parse_alias_line(line, &devices, &port_id);
        
        if (dev_count > 0 && devices && port_id > 0) {
            /* 创建别名组 */
            AliasGroup *group = malloc(sizeof(AliasGroup));
            memset(group, 0, sizeof(AliasGroup));
            group->port_id = port_id;
            
            /* 第一个设备是主映射 */
            group->primary = malloc(sizeof(AliasMapping));
            memset(group->primary, 0, sizeof(AliasMapping));
            group->primary->device = devices[0];
            group->primary->port_id = port_id;
            group->primary->is_primary = 1;
            
            /* 后续设备是别名 */
            AliasMapping *last_alias = group->primary;
            for (int i = 1; i < dev_count; i++) {
                AliasMapping *alias = malloc(sizeof(AliasMapping));
                memset(alias, 0, sizeof(AliasMapping));
                alias->device = devices[i];
                alias->port_id = port_id;
                alias->is_primary = 0;
                
                last_alias->next = alias;
                last_alias = alias;
                
                group->aliases = alias;
            }
            
            /* 添加到列表 */
            if (last_group) {
                last_group->next = group;
            } else {
                g_alias_groups = group;
            }
            last_group = group;
            
            printf("加载映射: 主设备=%s, 端口=%u, 别名数=%d\n",
                   group->primary->device, port_id, dev_count - 1);
        }
        
        /* 清理 */
        if (devices) {
            for (int i = 0; i < dev_count; i++) {
                free(devices[i]);
            }
            free(devices);
        }
    }
    
    fclose(fp);
    return 0;
}

/**
 * 查询NAS-Port（支持别名）
 */
uint32_t query_nas_port_with_aliases(const char *device) {
    AliasGroup *group = g_alias_groups;
    
    while (group) {
        /* 检查主映射 */
        if (group->primary && 
            strcmp(group->primary->device, device) == 0) {
            printf("通过主映射找到: %s -> %u\n", device, group->port_id);
            return group->port_id;
        }
        
        /* 检查别名 */
        AliasMapping *alias = group->aliases;
        while (alias) {
            if (strcmp(alias->device, device) == 0) {
                printf("通过别名找到: %s -> %u\n", device, group->port_id);
                return group->port_id;
            }
            alias = alias->next;
        }
        
        group = group->next;
    }
    
    printf("未找到设备 %s 的映射\n", device);
    return 0;
}

/**
 * 清理别名组
 */
void free_alias_groups() {
    AliasGroup *group = g_alias_groups;
    
    while (group) {
        AliasMapping *alias = group->primary;
        while (alias) {
            AliasMapping *next = alias->next;
            free(alias->device);
            free(alias);
            alias = next;
        }
        
        AliasGroup *next = group->next;
        free(group);
        group = next;
    }
    
    g_alias_groups = NULL;
}

/**
 * 自动生成别名
 */
int auto_generate_aliases() {
    printf("===== 自动生成别名 =====\n");
    
    /* 扫描 /dev/serial/by-id */
    DIR *dir = opendir("/dev/serial/by-id");
    if (dir) {
        struct dirent *entry;
        while ((entry = readdir(dir)) != NULL) {
            if (entry->d_type == DT_LNK) {
                char link_path[512];
                char target_path[512];
                
                snprintf(link_path, sizeof(link_path),
                        "/dev/serial/by-id/%s", entry->d_name);
                
                ssize_t len = readlink(link_path, target_path, 
                                      sizeof(target_path) - 1);
                if (len > 0) {
                    target_path[len] = '\0';
                    
                    /* 提取tty名称 */
                    char *tty_name = basename(target_path);
                    
                    printf("发现别名映射: %s -> %s\n", 
                           entry->d_name, tty_name);
                }
            }
        }
        closedir(dir);
    }
    
    return 0;
}

/* 示例配置格式说明 */
void show_config_format() {
    printf("\n===== 双映射配置文件格式 =====\n\n");
    
    printf("格式1: 标准格式\n");
    printf("  device_name    port_id\n\n");
    
    printf("格式2: 别名格式 (空格分隔)\n");
    printf("  device1 device2 device3    port_id\n\n");
    
    printf("格式3: 别名格式 (管道符分隔)\n");
    printf("  device1|device2|device3    port_id\n\n");
    
    printf("格式4: 显式箭头格式\n");
    printf("  device1 device2 -> port_id\n\n");
    
    printf("示例:\n");
    printf("  /dev/ttyUSB0    20\n");
    printf("  /dev/ttyUSB0 /dev/serial/by-id/xxx    20\n");
    printf("  /dev/ttyUSB0|/dev/ttyUSB1|/dev/pts/0    20\n");
}
```

---

## 综合示例: 完整的生产环境部署

### 部署架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    生产环境部署架构                               │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   设备层                                 │    │
│  │                                                          │    │
│  │   /dev/ttyS0-3 (内置串口)                                │    │
│  │   /dev/ttyUSB0-5 (USB转串口)                            │    │
│  │   /dev/pts/0-999 (伪终端)                               │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              稳定标识符层                                 │    │
│  │                                                          │    │
│  │   /dev/serial/by-id/*  (序列号，优先)                   │    │
│  │   /dev/serial/by-path/* (硬件路径，备用)                 │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              端口映射层 (port-id-map)                    │    │
│  │                                                          │    │
│  │   范围1: 1-999   固定设备                                │    │
│  │   范围2: 1000-1999  动态终端                             │    │
│  │   范围3: 5000-5999  VPN会话                              │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              会话绑定层                                   │    │
│  │                                                          │    │
│  │   SessionContext {                                      │    │
│  │     session_id: "xxx",                                  │    │
│  │     port_id: 100,  // 锁定                              │    │
│  │     ...                                                 │    │
│  │   }                                                     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              RADIUS客户端应用                             │    │
│  │                                                          │    │
│  │   rc_auth(rh, session.port_id, ...)  // 认证            │    │
│  │   rc_acct(rh, session.port_id, ...)  // 计费            │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              监控与回滚层                                 │    │
│  │                                                          │    │
│  │   monitor-port-map.sh  // 监控                          │    │
│  │   port-map-manager.sh  // 回滚                          │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 部署步骤

```bash
#!/bin/bash
# deploy-stable-port-mapping.sh
# 完整的生产环境部署脚本

set -e

echo "===== FreeRADIUS Client 稳定端口映射部署 ====="

# 1. 安装依赖
echo "[1/8] 安装依赖..."
apt-get update
apt-get install -y freeradius-client libfreeradius-client-dev

# 2. 创建目录结构
echo "[2/8] 创建目录结构..."
mkdir -p /etc/radiusclient
mkdir -p /var/lib/radiusclient
mkdir -p /var/log/radiusclient
mkdir -p /var/backups/radiusclient/port-maps
mkdir -p /usr/local/bin

# 3. 生成稳定的端口映射
echo "[3/8] 生成稳定的端口映射..."
cat > /etc/radiusclient/stable-port-id-map << 'EOF'
# 固定串口 (范围 1-99)
/dev/ttyS0    1
/dev/ttyS1    2
/dev/ttyS2    3
/dev/ttyS3    4

# USB串口 - by-id优先 (范围 100-199)
/dev/serial/by-id/usb-Prolific_Technology_Inc._USB-Serial_Controller_0001-if00-port0    100
/dev/serial/by-id/usb-Silicon_Labs_CP2102_0001-if00-port0    101
/dev/serial/by-id/usb-FTDI_FT232R_0001-if00-port0    102

# USB串口 - by-path备用
/dev/serial/by-path/pci-0000:00:14.0-usb-0:1:1.0-port0    100
/dev/serial/by-path/pci-0000:00:14.0-usb-0:1:1.1-port0    101
/dev/serial/by-path/pci-0000:00:14.0-usb-0:1:1.2-port0    102

# 物理终端 (范围 200-299)
/dev/tty1    200
/dev/tty2    201
/dev/tty3    202
EOF

# 4. 配置radiusclient.conf
echo "[4/8] 配置radiusclient.conf..."
cat > /etc/radiusclient/radiusclient.conf << 'EOF'
auth_order       radius,local
login_tries      4
login_timeout    60
nologin          /etc/nologin
servers          /etc/radiusclient/servers
dictionary       /etc/radiusclient/dictionary
mapfile          /etc/radiusclient/stable-port-id-map
radius_timeout   10
radius_retries   3
EOF

# 5. 设置权限
echo "[5/8] 设置权限..."
chmod 644 /etc/radiusclient/*.map
chmod 644 /etc/radiusclient/radiusclient.conf
chmod 600 /etc/radiusclient/servers

# 6. 部署监控脚本
echo "[6/8] 部署监控脚本..."
cp monitor-port-map.sh /usr/local/bin/
cp port-map-manager.sh /usr/local/bin/
chmod +x /usr/local/bin/*.sh

# 7. 配置定时任务
echo "[7/8] 配置定时任务..."
echo "*/5 * * * * root /usr/local/bin/monitor-port-map.sh -c >> /var/log/radiusclient/cron.log 2>&1" > /etc/cron.d/radius-port-monitor

# 8. 验证部署
echo "[8/8] 验证部署..."
/usr/local/bin/port-map-manager.sh verify

echo ""
echo "===== 部署完成 ====="
echo ""
echo "下一步:"
echo "1. 配置RADIUS服务器: /etc/radiusclient/servers"
echo "2. 配置用户: /etc/raddb/users"
echo "3. 测试认证: ./stable_radius_client testuser testpass"
echo "4. 启动监控: /usr/local/bin/monitor-port-map.sh -d"
echo "5. 查看状态: /usr/local/bin/port-map-manager.sh list"
```

### 完整文件清单

```
/etc/radiusclient/
├── radiusclient.conf          # 主配置文件
├── stable-port-id-map         # 稳定端口映射
├── servers                    # RADIUS服务器和密钥
└── dictionary                 # 属性字典

/var/lib/radiusclient/
├── sessions.db                # 会话绑定数据库
├── port-map-checksum.sha256   # 校验和
├── device-info.json           # 设备信息
└── port-map-state.json        # 状态文件

/var/log/radiusclient/
├── port-monitor.log           # 监控日志
├── port-monitor-alerts.log    # 告警日志
└── port-map-generator.log    # 生成日志

/var/backups/radiusclient/port-maps/
├── 20260523_100000_manual_backup.map
├── 20260523_110000_auto.map
└── ...

/usr/local/bin/
├── stable_radius_client       # 稳定标识符客户端
├── session_bound_radius       # 会话绑定客户端
├── monitor-port-map.sh        # 监控脚本
└── port-map-manager.sh        # 管理脚本
```

---

*文档版本：1.0*  
*最后更新：2026-05-23*
