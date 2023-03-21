---
layout: post
title: 【Redis】内存模型的源码解析-zmalloc
---

## **1.简介**

zmalloc是一个对内存分配进行简单封装的库。

## **2. API 原型**

1. void *zmalloc(size_t size);
2. void *zcalloc(size_t size);
3. void *zrealloc(void *ptr, size_t size);
4. void zfree(void *ptr);
5. char *zstrdup(const char *s);
6. size_t zmalloc_used_memory(void);
7. void zmalloc_set_oom_handler(void (*oom_handler)(size_t));
8. size_t zmalloc_get_rss(void);
9. int zmalloc_get_allocator_info(size_t *allocated, size_t *active, size_t *resident);
10. size_t zmalloc_get_private_dirty(long pid);
11. size_t zmalloc_get_smap_bytes_by_field(char *field, long pid);
12. size_t zmalloc_get_memory_size(void);
13. void zlibc_free(void *ptr);

## **3. API 速览**

| 名称                            | 功能                                              |
| ------------------------------- | ------------------------------------------------- |
| zmalloc                         | 分配size字节的内存                                |
| zcalloc                         | 分配size字节的内存，并全部初始化为0               |
| zrealloc                        | 对ptr指向的内存块进行大小调整，调整至size字节大小 |
| zfree                           | 释放ptr指向的内存块                               |
| zstrdup                         | 返回一个指向内容和s完全相同的char指针。           |
| zmalloc_used_memory             | 返回通过zmalloc库分配的内存大小。                 |
| zmalloc_set_oom_handler         | 设置OOM (Out Of Memory)的处理函数                 |
| zmalloc_get_rss                 | 获得当前进程RSS (Resident Set Size)的大小         |
| zmalloc_get_allocator_info      | 获得allocator的信息                               |
| zmalloc_get_private_dirty       | 获得smap中private dirty (私有脏页)的大小          |
| zmalloc_get_smap_bytes_by_field | 获得smap中特定字段的大小                          |
| zmalloc_get_memory_size         | 获得系统内存大小                                  |
| zlibc_free                      | 使用libc的free方法进行内存释放                    |
| zmalloc_size                    | 获取已分配内存总大小                              |

## **4. API原理**

### **1. zmalloc**

直接调用分配内存，成功返回对应指针，失败就调用 `zmalloc_oom_handler` 处理

```c
void *zmalloc(size_t size) {
    // 通过malloc分配size的内存，PREFIX_SIZE为该内存块的一些meta-data，详情可见malloc的实现原理
    void *ptr = malloc(size+PREFIX_SIZE);
    // 当ptr为空时调用oom处理函数
    if (!ptr) zmalloc_oom_handler(size);
    // 更新已经分配的内存大小变量used_memory，zmalloc_size会返回ptr指向的内存大小
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
}
```

### **2. zcalloc**

分配并初始化内存，失败直接调用 `zmalloc_oom_handler` 处理。

```c
void *zcalloc(size_t size) {
    // 通过calloc分配size的内存
    void *ptr = calloc(size+PREFIX_SIZE);
    // 当ptr为空时调用oom处理函数
    if (!ptr) zmalloc_oom_handler(size);
    // 更新已经分配的内存大小变量used_memory，zmalloc_size会返回ptr指向的内存大小
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
}
```

### **3. zrealloc**

对ptr指向的内存块进行大小调整，调整至size字节大小，若 ptr为空直接分配返回。

```c
void *zrealloc(void *ptr, size_t size) {
    size_t oldsize;
    void *newptr;
    // 如果ptr是空指针，直接返回一块size大小的内存
    if (ptr == NULL) return zmalloc(size);
    // 先获取ptr指向的内存的大小
    oldsize = zmalloc_size(ptr);
    // 分配size大小的新内存
    newptr = realloc(ptr,size);
    // 分配失败抛出OOM
    if (!newptr) zmalloc_oom_handler(size);
    // 更新used_memory，先减去oldsize，再加上size
    update_zmalloc_stat_free(oldsize);
    update_zmalloc_stat_alloc(zmalloc_size(newptr));
    return newptr;
}
```

### **4. zfree**

 释放ptr指向的内存块

```c
void zfree(void *ptr) {
    if (ptr == NULL) return;
    // 更新used_memory
    update_zmalloc_stat_free(zmalloc_size(ptr));
    // 释放内存
    free(ptr);
}
```

### **5. zstrdup**

返回一个指向内容和s完全相同的char指针。

```c
char *zstrdup(const char *s) {
    // 获取s的长度，并且加上0结尾的长度
    size_t l = strlen(s)+1;
    // 分配l长度的大小
    char *p = zmalloc(l);
    // 把s的内容拷贝到p
    memcpy(p,s,l);
    // p为复制后的结果
    return p;
}
```

### **6. zmalloc_used_memory**

返回通过zmalloc库分配的内存大小。

```c
size_t zmalloc_used_memory(void) {
    size_t um;
    // 获得原子变量used_memory的值
    atomicGet(used_memory,um);
    return um;
}
```

### **7. zmalloc_set_oom_handler**

设置OOM (Out Of Memory)的处理函数

```c
void zmalloc_set_oom_handler(void (*oom_handler)(size_t)) {
    // 设置OOM处理函数
    zmalloc_oom_handler = oom_handler;
}
```

### **8. zmalloc_get_rss**

同操作系统的特殊方式获得当前进程RSS (Resident Set Size)的大小（实际使用物理内存大小）

```c
size_t zmalloc_get_rss(void) {
    // 首先获取系统页面大小
    int page = sysconf(_SC_PAGESIZE);
    size_t rss;
    char buf[4096];
    char filename[256];
    int fd, count;
    char *p, *x;
    // 拼接/proc/<pid>/stat字符串
    snprintf(filename,256,"/proc/%d/stat",getpid());
    if ((fd = open(filename,O_RDONLY)) == -1) return 0;
    // 从/proc/<pid>/stat中读取最多4096字节的数据
    if (read(fd,buf,4096) <= 0) {
        close(fd);
        return 0;
    }
    close(fd);

    p = buf;
    count = 23; /* RSS is the 24th field in /proc/<pid>/stat */
    // 当p还没到末尾的时候，搜索以空格为间隔的第一次出现的地方
    // 因为stat文件的格式是这样的（以bash的stat为例）：
    // 732 (bash) S 731 732 732 34817 2112 4210944 23406 380524 1 451 86 19 1271 282 20 0 1 0 71934 24113152 1423 18446744073709551615 94563676422144 94563677484040 140736568454640 0 0 0 65536 3670020 1266777851 1 0 0 17 4 0 0 2 0 0 94563679583632 94563679630692 94563688689664 140736568461822 140736568461828 140736568461828 140736568463342 0
    // 而官方注释里有说，RSS字段是第24个值
    while(p && count--) {
        p = strchr(p,' ');
        // 如果p不是末尾，+1跳过该空格
        if (p) p++;
    }
    if (!p) return 0;
    // 获得RSS的字符串表示形式
    x = strchr(p,' ');
    // 没找到，说明stat不存在或异常
    if (!x) return 0;
    // 截断p，使得p只是一个数字的字符串表示形式
    *x = '\0';
    // 将p转化为long long的形式
    rss = strtoll(p,NULL,10);
    // 由于rss只是表示页数，需要乘以每页大小才是整个RSS的大小
    rss *= page;
    return rss;
}
```

### **9.zmalloc_get_allocator_info**

  获得allocator内存分配器的信息（Reids用的是JEMalloc或者TCMalloc来申请）

```c
// 定义USE_JEMALLOC时 使用 Jemalloc 内存模型管理
int zmalloc_get_allocator_info(size_t *allocated,
                               size_t *active,
                               size_t *resident) {
    uint64_t epoch = 1;
    size_t sz;
    *allocated = *resident = *active = 0;
    // 更新mallctl缓存中的统计
    sz = sizeof(epoch);
    je_mallctl("epoch", &epoch, &sz, &epoch, sz);
    sz = sizeof(size_t);
    /* Unlike RSS, this does not include RSS from shared libraries and other non
     * heap mappings. */
    je_mallctl("stats.resident", resident, &sz, NULL, 0);
    /* Unlike resident, this doesn't not include the pages jemalloc reserves
     * for re-use (purge will clean that). */
    je_mallctl("stats.active", active, &sz, NULL, 0);
    /* Unlike zmalloc_used_memory, this matches the stats.resident by taking
     * into account all allocations done by this process (not only zmalloc). */
    je_mallctl("stats.allocated", allocated, &sz, NULL, 0);
    return 1;
}

// 未使用 Jemalloc时
int zmalloc_get_allocator_info(size_t *allocated,
                               size_t *active,
                               size_t *resident) {
    // 使用libc的前提下该函数并没有实质意义上的实现
    *allocated = *resident = *active = 0;
    return 1;
}
```

### **10. zmalloc_get_private_dirty**

获得smap中private dirty (私有脏页)的大小

```c
size_t zmalloc_get_private_dirty(long pid) {
    // 获取smap中Private_Dirty: 字段的大小
    return zmalloc_get_smap_bytes_by_field("Private_Dirty:",pid);
}
```

### **11. zmalloc_get_smap_bytes_by_field**

获得smap中特定字段的大小

```c
size_t zmalloc_get_smap_bytes_by_field(char *field, long pid) {
    char line[1024];
    size_t bytes = 0;
    int flen = strlen(field);
    FILE *fp;
    // 如果pid是-1，说明是个子进程，读取父进程的smaps即可
    if (pid == -1) {
        fp = fopen("/proc/self/smaps","r");
    } else {
        // 是个父进程，打开对应pid的smaps
        char filename[128];
        snprintf(filename,sizeof(filename),"/proc/%ld/smaps",pid);
        fp = fopen(filename,"r");
    }

    if (!fp) return 0;
    // 逐行读取
    // 因为smaps的特征就是每行一个字段，所以逐行读取可以获得对应字段的信息
    while(fgets(line,sizeof(line),fp) != NULL) {
        // 判断field是否在当前行
        if (strncmp(line,field,flen) == 0) {
            // 获得k这个计量单位前的数字
            char *p = strchr(line,'k');
            if (p) {
                // p有效，截断p以转换数字
                *p = '\0';
                // 前面的空格不需要关心，因为strtol可以自动忽略数字前面所有的空格
                // *1024是因为返回的是字节数，smaps上面的单位是kB
                bytes += strtol(line+flen,NULL,10) * 1024;
            }
        }
    }
    fclose(fp);
    return bytes;
}
```

### **12. zmalloc_get_memory_size**

```c
size_t zmalloc_get_memory_size(void) {
    /* FreeBSD, Linux, OpenBSD, and Solaris. -------------------- */
    // 页数*页面大小=总内存
    return (size_t)sysconf(_SC_PHYS_PAGES) * (size_t)sysconf(_SC_PAGESIZE);
}
```

### **13. zlibc_free**

```c
void zlibc_free(void *ptr) {
    // 提供一种可以访问标准C库的free方法的封装，以方便释放由backtrace_symbols维护的内存，并且给使用非libc分配方法的其他分配器提供了一种访问标准库的方法。
    free(ptr);
}
```

### 14.zmalloc_size

获取已分配内存总大小，一个 `malloc` 本身不提供的功能，就是为了在每个分配内存的第一部分字节存储一个信息头（当前分配内存总大小）。照调用来看，应该是已分配内存可使用大小。仅在 `HAVE_MALLOC_SIZE` 未定义时定义。

```c
size_t zmalloc_size(void *ptr) {
    void *realptr = (char*)ptr-PREFIX_SIZE; // 偏移去掉前缀，指向实际指针
    size_t size = *((size_t*)realptr); // 获取实际指针关联内存大小
    return size+PREFIX_SIZE; // ptr实际分配内存=实际关联内存大小+前缀大小
}
```



## **5.宏函数**

### **5.1 update_zmalloc_stat_alloc**

```c
#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    //用于字节对齐
    //等价于  if(_n&7) _n += 8 - (_n&7);
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \ 
    //检查线程环境是否安全
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_add(_n); \
    } else { \
    	//安全时使用
        used_memory += _n; \
    } \
} while(0)
```

这个宏函数是用于更新原子类静态变量 used_memory（内存总使用大小）的值，互斥操作，其中__n为分配的内存字节数，if的作用是用于补齐__n到sizeof(long)

### **5.2 update_zmalloc_stat_free**

```c
#define update_zmalloc_stat_free(__n) do { 
    size_t _n = (__n); 
    //用于字节对齐
    //等价于  if(_n&7) _n += 8 - (_n&7);
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); 
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_sub(_n); 
    } else { \
       	//安全时使用
        used_memory -= _n; 
    } \
} while(0)
```

和上述一样。

### 5.3 update_zmalloc_stat_add()

确保线程安全，加了个线程锁。

```c
#define update_zmalloc_stat_add(__n) do { 
    pthread_mutex_lock(&used_memory_mutex); 
    used_memory += (__n); \
    pthread_mutex_unlock(&used_memory_mutex); 
} while(0)
```

### 5.4 update_zmalloc_stat_sub()

```c
#define update_zmalloc_stat_sub(__n) do { 
    pthread_mutex_lock(&used_memory_mutex); 
    used_memory -= (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)
```





## 6、总结

zmalloc 是对内存分配基于malloc 做了一层封装，为了在每个分配内存的第一部分字节存储一个信息头（类似SDS，可以直接获取 *ptr 指向内存的大小），另外实现了 Redis机制所需的函数，例如zmalloc_used_memory（获取通过zmalloc库分配的内存大小）、 zmalloc_get_private_dirty（获取每个线程私有脏页的大小），为什么会有私有脏页的说法，原因是还有关乎更底层的操作系统到底如何去分配内存的，Redis 使用内存分配器的实现（JEMalloc或者TCMalloc），下篇详解TCMalloc！