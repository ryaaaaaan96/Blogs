# pbuf 详解 - LwIP 数据包缓冲区

## 📋 概述

pbuf（Packet Buffer）是 LwIP 中最重要的数据结构之一，用于存储和管理网络数据包。它不仅承载着网络数据，还实现了零拷贝机制和灵活的内存管理策略。

## 🏗️ pbuf 结构体

### 核心结构定义

```c
struct pbuf {
  struct pbuf *next;      // 指向下一个pbuf，实现链表结构
  void *payload;          // 指向实际数据的指针
  u16_t tot_len;          // 当前pbuf及其后续所有pbuf链的总数据长度
  u16_t len;              // 当前pbuf中数据的长度
  u8_t type_internal;     // pbuf类型和分配来源的标志位
  u8_t flags;             // 其他标志位（如是否自定义、是否广播等）
  LWIP_PBUF_REF_T ref;    // 引用计数，表示有多少指针引用此pbuf
  u8_t if_idx;            // 接收数据包时，记录输入网卡的索引
  LWIP_PBUF_CUSTOM_DATA   // 用户自定义数据（宏定义，便于扩展）
};
```

### 设计亮点

1. **链表结构**：通过 `next` 指针支持数据包分片
2. **零拷贝**：`payload` 可指向不同位置，避免数据拷贝
3. **引用计数**：支持多个指针共享同一个 pbuf
4. **类型灵活**：支持多种不同的内存分配策略
5. **网卡索引**：记录数据包来源，便于处理

## 🚩 标志位详解

### flags 标志位

```c
#define PBUF_FLAG_PUSH      0x01U  // TCP PUSH标志，需要立即推送到应用层
#define PBUF_FLAG_IS_CUSTOM 0x02U  // 自定义类型，释放时调用自定义释放函数
#define PBUF_FLAG_MCASTLOOP 0x04U  // UDP组播需要回环到本地
#define PBUF_FLAG_LLBCAST   0x08U  // 链路层广播包
#define PBUF_FLAG_LLMCAST   0x10U  // 链路层组播包
#define PBUF_FLAG_TCP_FIN   0x20U  // TCP FIN标志
```

### type_internal 字段

```c
// 高位标志
#define PBUF_TYPE_FLAG_STRUCT_DATA_CONTIGUOUS   0x80  // 结构体和数据连续
#define PBUF_TYPE_FLAG_DATA_VOLATILE            0x40  // 数据易变，需要复制

// 低4位：分配来源
#define PBUF_TYPE_ALLOC_SRC_MASK                0x0F
#define PBUF_TYPE_ALLOC_SRC_MASK_STD_HEAP       0x00  // 标准堆分配
#define PBUF_TYPE_ALLOC_SRC_MASK_STD_MEMP_PBUF  0x01  // MEMP_PBUF内存池
#define PBUF_TYPE_ALLOC_SRC_MASK_STD_MEMP_PBUF_POOL 0x02  // MEMP_PBUF_POOL内存池

// 分配标志
#define PBUF_ALLOC_FLAG_RX                      0x0100  // 用于接收
#define PBUF_ALLOC_FLAG_DATA_CONTIGUOUS         0x0200  // 需要连续数据区
```

## 🎯 pbuf 类型详解

### PBUF_RAM

**特点**：
- 结构体和数据在一块连续的 RAM 内存
- 适合需要修改数据的场景
- 主要用于发送（TX）

**内存布局**：
```
+----------------+----------------+
| struct pbuf    | payload data   |
+----------------+----------------+
^                ^
pbuf            payload
```

**使用场景**：
- TCP/UDP 发送数据包
- 应用层构造数据包
- 需要修改数据内容的场景

### PBUF_ROM

**特点**：
- 数据在 ROM（只读存储器）中
- 结构体和数据分离
- 适合发送静态数据

**内存布局**：
```
+----------------+     ROM区域
| struct pbuf    |  +-------------+
|   payload ------->| static data |
+----------------+  +-------------+
```

**使用场景**：
- HTTP 服务器发送静态网页
- 发送固定的协议响应
- 常量数据传输

### PBUF_REF

**特点**：
- 数据在外部 RAM 中
- 数据可能被其他地方修改
- 通常需要复制后发送

**内存布局**：
```
+----------------+     外部RAM
| struct pbuf    |  +-------------+
|   payload ------->| extern data |
+----------------+  +-------------+
```

**使用场景**：
- 引用外部缓冲区数据
- DMA 缓冲区数据
- 需要零拷贝但数据易变的场景

### PBUF_POOL

**特点**：
- 从内存池分配，速度快
- 支持链式分片
- 主要用于接收（RX）

**内存布局**：
```
+----------------+----------------+     +----------------+----------------+
| struct pbuf    | payload data   | --> | struct pbuf    | payload data   |
+----------------+----------------+     +----------------+----------------+
```

**使用场景**：
- 网络接口驱动接收数据
- 大数据包的分片接收
- 需要高效内存分配的场景

## 🔧 pbuf 分配函数

### pbuf_alloc 函数

```c
struct pbuf *pbuf_alloc(pbuf_layer layer, u16_t length, pbuf_type type)
```

**参数说明**：
- `layer`：协议层类型，决定预留的头部空间
- `length`：payload 数据长度
- `type`：pbuf 类型

**协议层类型**：
```c
typedef enum {
  PBUF_TRANSPORT,    // 传输层（TCP/UDP头部后）
  PBUF_IP,           // 网络层（IP头部后）
  PBUF_LINK,         // 链路层（以太网头部后）
  PBUF_RAW_TX,       // 原始发送（包含所有头部）
  PBUF_RAW           // 原始接收
} pbuf_layer;
```

### 分配流程

**PBUF_RAM 分配**：
```c
case PBUF_RAM: {
  // 计算总大小：结构体 + 头部偏移 + 数据长度
  mem_size_t alloc_len = LWIP_MEM_ALIGN_SIZE(SIZEOF_STRUCT_PBUF + offset) + 
                         LWIP_MEM_ALIGN_SIZE(length);
  
  // 从内存堆分配
  p = (struct pbuf*)mem_malloc(alloc_len);
  if (p == NULL) {
    return NULL;
  }
  
  // 设置 payload 指针
  p->payload = LWIP_MEM_ALIGN((void *)((u8_t *)p + SIZEOF_STRUCT_PBUF + offset));
  
  // 初始化结构体
  pbuf_init_alloced_pbuf(p, p->payload, length, length, type, 0);
  break;
}
```

**PBUF_POOL 分配**：
```c
case PBUF_POOL: {
  struct pbuf *q, *last;
  u16_t rem_len = length;
  
  // 可能需要多个 pbuf 来存储数据
  do {
    // 从内存池分配一个 pbuf
    q = (struct pbuf *)memp_malloc(MEMP_PBUF_POOL);
    if (q == NULL) {
      // 分配失败，释放已分配的链表
      if (p) pbuf_free(p);
      return NULL;
    }
    
    // 计算当前 pbuf 可存储的数据长度
    u16_t qlen = LWIP_MIN(rem_len, 
                         (u16_t)(PBUF_POOL_BUFSIZE_ALIGNED - 
                                LWIP_MEM_ALIGN_SIZE(offset)));
    
    // 初始化 pbuf
    pbuf_init_alloced_pbuf(q, 
                          LWIP_MEM_ALIGN((void *)((u8_t *)q + SIZEOF_STRUCT_PBUF + offset)),
                          rem_len, qlen, type, 0);
    
    // 链接到链表
    if (p == NULL) {
      p = q;  // 第一个 pbuf
    } else {
      last->next = q;  // 链接到链表尾部
    }
    
    last = q;
    rem_len = (u16_t)(rem_len - qlen);
    offset = 0;  // 后续 pbuf 不需要头部偏移
  } while (rem_len > 0);
  
  break;
}
```

## 🔄 pbuf 引用计数

### 引用计数机制

```c
// 增加引用计数
void pbuf_ref(struct pbuf *p)
{
  if (p != NULL) {
    SYS_ARCH_PROTECT(old_level);
    ++(p->ref);
    SYS_ARCH_UNPROTECT(old_level);
  }
}

// 减少引用计数并可能释放
u8_t pbuf_free(struct pbuf *p)
{
  u8_t alloc_src;
  struct pbuf *q;
  u8_t count;
  
  if (p == NULL) {
    return 0;
  }
  
  count = 0;
  while (p != NULL) {
    SYS_ARCH_PROTECT(old_level);
    --(p->ref);
    if (p->ref == 0) {
      // 引用计数为0，可以释放
      SYS_ARCH_UNPROTECT(old_level);
      
      alloc_src = pbuf_get_allocsrc(p);
      q = p->next;
      
      // 根据分配来源选择释放方法
      if (alloc_src == PBUF_TYPE_ALLOC_SRC_MASK_STD_HEAP) {
        mem_free(p);  // 释放到内存堆
      } else {
        memp_free(MEMP_PBUF_POOL, p);  // 释放到内存池
      }
      
      count++;
      p = q;
    } else {
      SYS_ARCH_UNPROTECT(old_level);
      // 还有其他引用，不能释放
      break;
    }
  }
  
  return count;
}
```

## 🎨 pbuf 操作函数

### 头部操作

```c
// 在 pbuf 头部添加空间（向前移动 payload）
u8_t pbuf_add_header(struct pbuf *p, size_t header_size_increment)
{
  u16_t increment_magnitude = (u16_t)header_size_increment;
  void *payload = (u8_t *)p->payload - increment_magnitude;
  
  // 检查是否有足够空间
  if ((u8_t *)payload < (u8_t *)p + SIZEOF_STRUCT_PBUF) {
    return 1;  // 空间不足
  }
  
  // 更新 payload 和长度
  p->payload = payload;
  p->len = (u16_t)(p->len + increment_magnitude);
  p->tot_len = (u16_t)(p->tot_len + increment_magnitude);
  
  return 0;
}

// 移除 pbuf 头部（向后移动 payload）
u8_t pbuf_remove_header(struct pbuf *p, size_t header_size)
{
  void *payload = (u8_t *)p->payload + header_size;
  
  // 检查是否超出数据区
  if (header_size > p->len) {
    return 1;  // 移除长度超出数据长度
  }
  
  // 更新 payload 和长度
  p->payload = payload;
  p->len = (u16_t)(p->len - header_size);
  p->tot_len = (u16_t)(p->tot_len - header_size);
  
  return 0;
}
```

### 数据复制

```c
// 从 pbuf 链表复制数据到缓冲区
u16_t pbuf_copy_partial(const struct pbuf *buf, void *dataptr, u16_t len, u16_t offset)
{
  const struct pbuf *p;
  u16_t left = 0;
  u16_t buf_copy_len;
  u16_t copied_total = 0;
  
  // 找到偏移位置
  for (p = buf; len != 0 && p != NULL; p = p->next) {
    if ((offset != 0) && (offset >= p->len)) {
      offset = (u16_t)(offset - p->len);
    } else {
      // 复制当前 pbuf 中的数据
      buf_copy_len = (u16_t)(p->len - offset);
      if (buf_copy_len > len) {
        buf_copy_len = len;
      }
      
      MEMCPY(&((char *)dataptr)[left], &((char *)p->payload)[offset], buf_copy_len);
      copied_total = (u16_t)(copied_total + buf_copy_len);
      left = (u16_t)(left + buf_copy_len);
      len = (u16_t)(len - buf_copy_len);
      offset = 0;
    }
  }
  
  return copied_total;
}
```

## 📊 内存池配置

### 相关内存池

LwIP 初始化两个与 pbuf 相关的内存池：

1. **MEMP_PBUF**：
   - 用于存放 pbuf 结构体
   - 主要供 PBUF_ROM、PBUF_REF 类型使用
   - 不包含数据区

2. **MEMP_PBUF_POOL**：
   - 用于存放 pbuf 结构体和数据区
   - 主要供 PBUF_POOL 类型使用
   - 包含连续的数据存储空间

### 配置参数

```c
/* pbuf 相关配置 */
#define MEMP_NUM_PBUF              16    // MEMP_PBUF 内存池大小
#define MEMP_NUM_PBUF_POOL         16    // MEMP_PBUF_POOL 内存池大小
#define PBUF_POOL_BUFSIZE          592   // PBUF_POOL 每个元素的数据区大小

/* MTU 相关配置 */
#define PBUF_LINK_ENCAPSULATION_HLEN  14   // 以太网头部长度
#define PBUF_LINK_HLEN                14   // 链路层头部长度
```

## 🎯 最佳实践

### 选择合适的 pbuf 类型

1. **发送数据**：优先使用 `PBUF_RAM`，便于修改和管理
2. **接收数据**：使用 `PBUF_POOL`，分配速度快且支持分片
3. **静态数据**：使用 `PBUF_ROM`，避免不必要的内存复制
4. **外部数据**：使用 `PBUF_REF`，但注意数据生命周期

### 内存管理建议

1. **及时释放**：使用 `pbuf_free()` 及时释放不需要的 pbuf
2. **引用计数**：理解引用计数机制，避免重复释放
3. **链表处理**：正确处理 pbuf 链表，避免内存泄漏
4. **大小配置**：根据应用需求合理配置内存池大小

### 性能优化

1. **零拷贝**：尽量使用指针操作，避免数据复制
2. **批量处理**：对于大数据，使用 pbuf 链表分片处理
3. **内存对齐**：确保数据对齐，提高访问效率
4. **缓存友好**：连续的内存访问模式提高缓存命中率

---

> **总结**：pbuf 是 LwIP 数据管理的核心，通过灵活的类型设计和高效的内存管理，实现了在嵌入式环境下的高性能网络数据处理。理解 pbuf 的设计思想对于深入掌握 LwIP 至关重要。
