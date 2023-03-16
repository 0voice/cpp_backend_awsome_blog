# 【NO.405】FFmpeg源码分析：内存管理系统

FFmpeg有专门的内存管理系统，包括：内存分配、内存拷贝、内存释放。其中内存分配包含分配内存与对齐、内存分配与清零、分配指定大小的内存块、重新分配内存块、快速分配内存、分配指定最大值的内存、分配数组内存、快速分配数组内存、重新分配数组内存。

FFmpeg的内存管理位于libavutil/mem.c，相关函数如下图所示：

![img](https://pic4.zhimg.com/80/v2-fad0b0128d7c6776305bf526da87d37b_720w.webp)

## 1.内存分配

### **1.1 av_malloc**

av_malloc()内存分配，并且内存对齐，方便系统快速访问内存。代码如下：

```text
void *av_malloc(size_t size)
{
    void *ptr = NULL;
 
    if (size > max_alloc_size)
        return NULL;
 
#if HAVE_POSIX_MEMALIGN
    if (size)
    if (posix_memalign(&ptr, ALIGN, size))
        ptr = NULL;
#elif HAVE_ALIGNED_MALLOC
    ptr = _aligned_malloc(size, ALIGN);
#elif HAVE_MEMALIGN
#ifndef __DJGPP__
    ptr = memalign(ALIGN, size);
#else
    ptr = memalign(size, ALIGN);
#endif
    /* Why 64?
     * Indeed, we should align it:
     *   on  4 for 386
     *   on 16 for 486
     *   on 32 for 586, PPro - K6-III
     *   on 64 for K7 (maybe for P3 too).
     * Because L1 and L2 caches are aligned on those values.
     * But I don't want to code such logic here!
     */
    /* Why 32?
     * For AVX ASM. SSE / NEON needs only 16.
     * Why not larger? Because I did not see a difference in benchmarks ...
     */
    /* benchmarks with P3
     * memalign(64) + 1          3071, 3051, 3032
     * memalign(64) + 2          3051, 3032, 3041
     * memalign(64) + 4          2911, 2896, 2915
     * memalign(64) + 8          2545, 2554, 2550
     * memalign(64) + 16         2543, 2572, 2563
     * memalign(64) + 32         2546, 2545, 2571
     * memalign(64) + 64         2570, 2533, 2558
     *
     * BTW, malloc seems to do 8-byte alignment by default here.
     */
#else
    ptr = malloc(size);
#endif
    if(!ptr && !size) {
        size = 1;
        ptr= av_malloc(1);
    }
#if CONFIG_MEMORY_POISONING
    if (ptr)
        memset(ptr, FF_MEMORY_POISON, size);
#endif
    return ptr;
}
```

### **1.2 av_mallocz**

av_mallocz()是在av_malloc()基础上，调用memset()进行内存清零：

```text
void *av_mallocz(size_t size)
{
    void *ptr = av_malloc(size);
    if (ptr)
        memset(ptr, 0, size);
    return ptr;
}
```

### **1.3 av_malloc_array**

av_malloc_array()先计算数组所需要内存块大小，然后用av_malloc()分配数组内存：

```text
void *av_malloc_array(size_t nmemb, size_t size)
{
    size_t result;
    if (av_size_mult(nmemb, size, &result) < 0)
        return NULL;
    return av_malloc(result);
}
```

### **1.4 av_mallocz_array**

av_mallocz_array()先计算数组所需要内存块大小，然后用av_mallocz()分配数组内存：

```text
void *av_mallocz_array(size_t nmemb, size_t size)
{
    size_t result;
    if (av_size_mult(nmemb, size, &result) < 0)
        return NULL;
    return av_mallocz(result);
}
```

### **1.5 av_calloc**

av_calloc()操作与av_mallocz_array()，先计算内存大小再用av_mallocz()分配内存：

```text
void *av_calloc(size_t nmemb, size_t size)
{
    size_t result;
    if (av_size_mult(nmemb, size, &result) < 0)
        return NULL;
    return av_mallocz(result);
}
```

### **1.6 av_max_alloc**

av_max_alloc()主要是指定分配内存的最大值：

```text
static size_t max_alloc_size= INT_MAX;
 
void av_max_alloc(size_t max)
{
    max_alloc_size = max;
}
```

在av_malloc()用于判断size是否超出最大值：

```text
void *av_malloc(size_t size)
{
    void *ptr = NULL;
 
    if (size > max_alloc_size)
        return NULL;
 
    ......
}
```

### **1.7 av_realloc**

av_realloc()是对系统的realloc函数进行封装，重新分配内存块：

```text
void *av_realloc(void *ptr, size_t size)
{
    if (size > max_alloc_size)
        return NULL;
 
#if HAVE_ALIGNED_MALLOC
    return _aligned_realloc(ptr, size + !size, ALIGN);
#else
    return realloc(ptr, size + !size);
#endif
}
```

### **1.8 av_realloc_array**

av_realloc_array()先计算内存块大小，然后用av_realloc()重新分配数组内存：

```text
void *av_realloc_array(void *ptr, size_t nmemb, size_t size)
{
    size_t result;
    if (av_size_mult(nmemb, size, &result) < 0)
        return NULL;
    return av_realloc(ptr, result);
}
```

### **1.9 av_fast_realloc**

av_fast_realloc()快速重新分配内存，如果原始内存块足够大直接复用：

```text
void *av_fast_realloc(void *ptr, unsigned int *size, size_t min_size)
{
    if (min_size <= *size)
        return ptr;
 
    if (min_size > max_alloc_size) {
        *size = 0;
        return NULL;
    }
 
    min_size = FFMIN(max_alloc_size, FFMAX(min_size + min_size / 16 + 32, min_size));
 
    ptr = av_realloc(ptr, min_size);
    /* we could set this to the unmodified min_size but this is safer
     * if the user lost the ptr and uses NULL now
     */
    if (!ptr)
        min_size = 0;
 
    *size = min_size;
 
    return ptr;
}
```

### **1.10 av_fast_malloc**

av_fast_malloc()快速分配内存：

```text
void av_fast_malloc(void *ptr, unsigned int *size, size_t min_size)
{
    ff_fast_malloc(ptr, size, min_size, 0);
}
```

其中ff_fast_malloc()代码位于libavutil/mem_internal.h:

```text
static inline int ff_fast_malloc(void *ptr, unsigned int *size, size_t min_size, int zero_realloc)
{
    void *val;
 
    memcpy(&val, ptr, sizeof(val));
    if (min_size <= *size) {
        av_assert0(val || !min_size);
        return 0;
    }
    min_size = FFMAX(min_size + min_size / 16 + 32, min_size);
    av_freep(ptr);
    val = zero_realloc ? av_mallocz(min_size) : av_malloc(min_size);
    memcpy(ptr, &val, sizeof(val));
    if (!val)
        min_size = 0;
    *size = min_size;
    return 1;
}
```

### **1.11 av_fast_mallocz**

av_fast_mallocz()快速分配内存，并且内存清零：

```text
void av_fast_mallocz(void *ptr, unsigned int *size, size_t min_size)
{
    ff_fast_malloc(ptr, size, min_size, 1);
}
```

## 2.内存拷贝

### **2.1 av_strdup**

av_strdup()用于重新分配内存与拷贝字符串：

```text
char *av_strdup(const char *s)
{
    char *ptr = NULL;
    if (s) {
        size_t len = strlen(s) + 1;
        ptr = av_realloc(NULL, len);
        if (ptr)
            memcpy(ptr, s, len);
    }
    return ptr;
}
```

### **2.2 av_strndup**

av_strndup()用于分配指定大小内存与拷贝字符串，先用memchr()获取有效字符串长度，然后使用av_realloc()重新分配内存，再用memcpy()拷贝字符串：

```text
char *av_strndup(const char *s, size_t len)
{
    char *ret = NULL, *end;
 
    if (!s)
        return NULL;
 
    end = memchr(s, 0, len);
    if (end)
        len = end - s;
 
    ret = av_realloc(NULL, len + 1);
    if (!ret)
        return NULL;
 
    memcpy(ret, s, len);
    ret[len] = 0;
    return ret;
}
```

### **2.3 av_memdup**

av_memdup()用于内存分配与内存拷贝，先用av_malloc()分配内存，再用memcpy()拷贝内存：

```text
void *av_memdup(const void *p, size_t size)
{
    void *ptr = NULL;
    if (p) {
        ptr = av_malloc(size);
        if (ptr)
            memcpy(ptr, p, size);
    }
    return ptr;
}
```

### **2.4 av_memcpy_backptr**

av_memcpy_backptr()用于内存拷贝，与系统提供的memcpy()类似，并且考虑16位、24位、32位内存对齐：

```text
void av_memcpy_backptr(uint8_t *dst, int back, int cnt)
{
    const uint8_t *src = &dst[-back];
    if (!back)
        return;
 
    if (back == 1) {
        memset(dst, *src, cnt);
    } else if (back == 2) {
        fill16(dst, cnt);
    } else if (back == 3) {
        fill24(dst, cnt);
    } else if (back == 4) {
        fill32(dst, cnt);
    } else {
        if (cnt >= 16) {
            int blocklen = back;
            while (cnt > blocklen) {
                memcpy(dst, src, blocklen);
                dst       += blocklen;
                cnt       -= blocklen;
                blocklen <<= 1;
            }
            memcpy(dst, src, cnt);
            return;
        }
        if (cnt >= 8) {
            AV_COPY32U(dst,     src);
            AV_COPY32U(dst + 4, src + 4);
            src += 8;
            dst += 8;
            cnt -= 8;
        }
        if (cnt >= 4) {
            AV_COPY32U(dst, src);
            src += 4;
            dst += 4;
            cnt -= 4;
        }
        if (cnt >= 2) {
            AV_COPY16U(dst, src);
            src += 2;
            dst += 2;
            cnt -= 2;
        }
        if (cnt)
            *dst = *src;
    }
}
```

## 3.内存释放

### **3.1 av_free**

av_free()用于释放内存块，主要是调用系统free()进行释放。如果宏定义了对齐分配，那么要对齐释放：

```text
void av_free(void *ptr)
{
#if HAVE_ALIGNED_MALLOC
    _aligned_free(ptr);
#else
    free(ptr);
#endif
}
```

### **3.2 av_freep**

av_freep()用于释放内存指针，先备份内存指针，然后把指针地址清空，再释放内存：

```text
void av_freep(void *arg)
{
    void *val;
 
    memcpy(&val, arg, sizeof(val));
    memcpy(arg, &(void *){ NULL }, sizeof(val));
    av_free(val);
}
```

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/450437498