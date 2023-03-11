# 【NO.124】一文掌握 Linux 内存管理

以下源代码来自 linux-5.10.3 内核代码，主要以 x86-32 为例。

Linux 内存管理是一个很复杂的“工程”，它不仅仅是对物理内存的管理，也涉及到虚拟内存管理、内存交换和内存回收等

## 1.物理内存的探测

Linux 内核通过 detect_memory()函数实现对物理内存的探测

```
void detect_memory(void){ detect_memory_e820(); detect_memory_e801(); detect_memory_88();}
```

这里主要介绍一下 detect_memory_e820()，detect_memory_e801()和 detect_memory_88()是针对较老的电脑进行兼容而保留的

```
static void detect_memory_e820(void){ int count = 0; struct biosregs ireg, oreg; struct boot_e820_entry *desc = boot_params.e820_table; static struct boot_e820_entry buf; /* static so it is zeroed */ initregs(&ireg); ireg.ax  = 0xe820; ireg.cx  = sizeof(buf); ireg.edx = SMAP; ireg.di  = (size_t)&buf; do {  intcall(0x15, &ireg, &oreg);  ireg.ebx = oreg.ebx; /* for next iteration... */  if (oreg.eflags & X86_EFLAGS_CF)   break;  if (oreg.eax != SMAP) {   count = 0;   break;  }  *desc++ = buf;  count++; } while (ireg.ebx && count < ARRAY_SIZE(boot_params.e820_table)); boot_params.e820_entries = count;}
```

**detect_memory_e820()实现内核从 BIOS 那里获取到内存的基础布局**，之所以叫 e820 是因为内核是通过 0x15 中断向量，并在 AX 寄存器中指定 0xE820，中断调用后将会返回被 BIOS 保留的内存地址范围以及系统可以使用的内存地址范围，所有通过中断获取的数据将会填充在 boot_params.e820_table 中，具体 0xE820 的详细用法感兴趣的话可以上网查……这里获取到的 e820_table 里的数据是未经过整理，linux 会通过 setup_memory_map 去整理这些数据

```
start_kernel() -> setup_arch() -> setup_memory_map()void __init e820__memory_setup(void){ char *who; BUILD_BUG_ON(sizeof(struct boot_e820_entry) != 20); who = x86_init.resources.memory_setup(); memcpy(e820_table_kexec, e820_table, sizeof(*e820_table_kexec)); memcpy(e820_table_firmware, e820_table, sizeof(*e820_table_firmware)); pr_info("BIOS-provided physical RAM map:\n"); e820__print_table(who);}
```

x86_init.resources.memory_setup()指向了 e820__memory_setup_default()，会将 boot_params.e820_table 转换为内核自己使用的 e820_table，转换之后的**e820 表记录着所有物理内存的起始地址、长度以及类型**，然后通过 memcpy 将 e820_table 复制到 e820_table_kexec、e820_table_firmware

```
struct x86_init_ops x86_init __initdata = { .resources = {  .probe_roms  = probe_roms,  .reserve_resources = reserve_standard_io_resources,  .memory_setup  = e820__memory_setup_default, }, ......}char *__init e820__memory_setup_default(void){ char *who = "BIOS-e820"; /*  * Try to copy the BIOS-supplied E820-map.  *  * Otherwise fake a memory map; one section from 0k->640k,  * the next section from 1mb->appropriate_mem_k  */ if (append_e820_table(boot_params.e820_table, boot_params.e820_entries) < 0) {  u64 mem_size;  /* Compare results from other methods and take the one that gives more RAM: */  if (boot_params.alt_mem_k < boot_params.screen_info.ext_mem_k) {   mem_size = boot_params.screen_info.ext_mem_k;   who = "BIOS-88";  } else {   mem_size = boot_params.alt_mem_k;   who = "BIOS-e801";  }  e820_table->nr_entries = 0;  e820__range_add(0, LOWMEMSIZE(), E820_TYPE_RAM);  e820__range_add(HIGH_MEMORY, mem_size << 10, E820_TYPE_RAM); } /* We just appended a lot of ranges, sanitize the table: */ e820__update_table(e820_table); return who;}
```

内核使用的**e820_table 结构描述**如下：

```
enum e820_type { E820_TYPE_RAM  = 1, E820_TYPE_RESERVED = 2, E820_TYPE_ACPI  = 3, E820_TYPE_NVS  = 4, E820_TYPE_UNUSABLE = 5, E820_TYPE_PMEM  = 7, E820_TYPE_PRAM  = 12, E820_TYPE_SOFT_RESERVED = 0xefffffff, E820_TYPE_RESERVED_KERN = 128,};struct e820_entry { u64   addr; u64   size; enum e820_type  type;} __attribute__((packed));struct e820_table { __u32 nr_entries; struct e820_entry entries[E820_MAX_ENTRIES];};
```

### 1.1 memblock 内存分配器

linux x86 内存映射主要存在两种方式：**段式映射和页式映射**。linux 首次进入保护模式时会用到段式映射（加电时，运行在实模式，任意内存地址都能执行代码，可以被读写，这非常不安全，CPU 为了提供限制/禁止的手段，提出了保护模式），根据段寄存器（以 8086 为例，段寄存器有 CS（Code Segment）：代码段寄存器；DS（Data Segment）：数据段寄存器；SS（Stack Segment）：堆栈段寄存器；ES（Extra Segment）：附加段寄存器）查找到对应的**段描述符**（这里其实就是用到了段描述符表，即段表），段描述符指明了此时的环境可以通过段访问到内存基地址、空间大小和访问权限。访问权限则点明了哪些内存可读、哪些内存可写。

```
typedef struct Descriptor{    unsigned int base;  // 段基址    unsigned int limit; // 段大小    unsigned short attribute;   // 段属性、权限}
```

linux 在段描述符表准备完成之后会通过汇编跳转到保护模式

事实上，在上面这个过程中，linux 并没有明显地去区分每个段，所以这里并没有很好地起到保护作用，linux 最终使用的还是内存分页管理（开启页式映射可以参考/arch/x86/kernel/head_32.S）

#### **1.1.1 memblock 算法**

memblock 是 linux 内核初始化阶段使用的一个内存分配器，实现较为简单，负责**页分配器初始化之前的内存管理和分配请求**，memblock 的结构如下

```
struct memblock_region { phys_addr_t base; phys_addr_t size; enum memblock_flags flags;#ifdef CONFIG_NEED_MULTIPLE_NODES int nid;#endif};struct memblock_type { unsigned long cnt; unsigned long max; phys_addr_t total_size; struct memblock_region *regions; char *name;};struct memblock { bool bottom_up;  /* is bottom up direction? */ phys_addr_t current_limit; struct memblock_type memory; struct memblock_type reserved;};
```

**bottom_up**：用来表示分配器分配内存是自低地址向高地址还是自高地址向低地址

**current_limit**：用来表示用来限制 memblock_alloc()和 memblock_alloc_base()的内存申请

**memory**：用于指向系统**可用物理内存区**，这个内存区维护着系统所有可用的物理内存，即系统 DRAM 对应的物理内存

**reserved**：用于指向系统预留区，也就是这个内存区的内存**已经分配**，在释放之前不能再次分配这个区内的内存区块

**memblock_type**中的**cnt**用于描述该类型内存区中的**内存区块数**，这有利于 MEMBLOCK 内存分配器动态地知道某种类型的内存区还有多少个内存区块

memblock_type 中的**max**用于描述该类型内存区**最大可以含有多少个内存区块**，当往某种类型的内存区添加 内存区块的时候，如果内存区的内存区块数超过 max 成员，那么 memblock 内存分配器就会增加内存区的容量，以此维护更多的内存区块

memblock_type 中的**total_size**用于统计内存区总共含有的**物理内存数**

memblock_type 中的**regions**是一个**内存区块链表**，用于维护属于这类型的所有内存区块（包括基址、大小和内存块标记等），

**name** ：用于指明这个**内存区的名字**，MEMBLOCK 分配器目前支持的内存区名字有：**“memory”, “reserved”, “physmem”**

具体关系可以参考下图：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171713176389130.png)
内核启动后会为 MEMBLOCK 内存分配器创建了一些私有的 section，这些 section 用于存放于 MEMBLOCK 分配器有关的函数和数据，即 **init_memblock** 和 **initdata_memblock**。在创建完 init_memblock section 和 initdata_memblock section 之后，memblock 分配器会开始创建 struct memblock 实例，这个实例此时作为最原始的 MEMBLOCK 分配器，描述了系统的物理内存的初始值

```
#define MEMBLOCK_ALLOC_ANYWHERE (~(phys_addr_t)0) //即0xFFFFFFFF#define INIT_MEMBLOCK_REGIONS   128#ifndef INIT_MEMBLOCK_RESERVED_REGIONS# define INIT_MEMBLOCK_RESERVED_REGIONS  INIT_MEMBLOCK_REGIONS#endifstatic struct memblock_region memblock_memory_init_regions[INIT_MEMBLOCK_REGIONS] __initdata_memblock;static struct memblock_region memblock_reserved_init_regions[INIT_MEMBLOCK_RESERVED_REGIONS] __initdata_memblock;struct memblock memblock __initdata_memblock = { .memory.regions  = memblock_memory_init_regions, .memory.cnt  = 1, /* empty dummy entry */ .memory.max  = INIT_MEMBLOCK_REGIONS, .memory.name  = "memory", .reserved.regions = memblock_reserved_init_regions, .reserved.cnt  = 1, /* empty dummy entry */ .reserved.max  = INIT_MEMBLOCK_RESERVED_REGIONS, .reserved.name  = "reserved", .bottom_up  = false, .current_limit  = MEMBLOCK_ALLOC_ANYWHERE,};
```

内核在 setup_arch(char **cmdline_p)中会调用**e820__memblock_setup()**对 MEMBLOCK 内存分配器进行**初始化**启动

```
void __init e820__memblock_setup(void){ int i; u64 end; memblock_allow_resize(); for (i = 0; i < e820_table->nr_entries; i++) {  struct e820_entry *entry = &e820_table->entries[i];  end = entry->addr + entry->size;  if (end != (resource_size_t)end)   continue;  if (entry->type == E820_TYPE_SOFT_RESERVED)   memblock_reserve(entry->addr, entry->size);  if (entry->type != E820_TYPE_RAM && entry->type != E820_TYPE_RESERVED_KERN)   continue;  memblock_add(entry->addr, entry->size); } /* Throw away partial pages: */ memblock_trim_memory(PAGE_SIZE); memblock_dump_all();}
```

memblock_allow_resize() 仅是用于置 memblock_can_resize 的值，里面的 for 则是用于循环遍历 e820 的内存布局信息，然后进行 memblock_add 的操作

```
int __init_memblock memblock_add(phys_addr_t base, phys_addr_t size){ phys_addr_t end = base + size - 1; memblock_dbg("%s: [%pa-%pa] %pS\n", __func__,       &base, &end, (void *)_RET_IP_); return memblock_add_range(&memblock.memory, base, size, MAX_NUMNODES, 0);}
```

memblock_add()主要封装了 memblock_add_region()，且它操作对象是 memblock.memory，即系统可用物理内存区。

```
static int __init_memblock memblock_add_range(struct memblock_type *type,    phys_addr_t base, phys_addr_t size,    int nid, enum memblock_flags flags){ bool insert = false; phys_addr_t obase = base;    //调整size大小，确保不会越过边界 phys_addr_t end = base + memblock_cap_size(base, &size); int idx, nr_new; struct memblock_region *rgn; if (!size)  return 0; /* special case for empty array */ if (type->regions[0].size == 0) {  WARN_ON(type->cnt != 1 || type->total_size);  type->regions[0].base = base;  type->regions[0].size = size;  type->regions[0].flags = flags;  memblock_set_region_node(&type->regions[0], nid);  type->total_size = size;  return 0; }repeat: /*  * The following is executed twice.  Once with %false @insert and  * then with %true.  The first counts the number of regions needed  * to accommodate the new area.  The second actually inserts them.  */ base = obase; nr_new = 0; for_each_memblock_type(idx, type, rgn) {  phys_addr_t rbase = rgn->base;  phys_addr_t rend = rbase + rgn->size;  if (rbase >= end)   break;  if (rend <= base)   continue;  /*   * @rgn overlaps.  If it separates the lower part of new   * area, insert that portion.   */  if (rbase > base) {#ifdef CONFIG_NEED_MULTIPLE_NODES   WARN_ON(nid != memblock_get_region_node(rgn));#endif   WARN_ON(flags != rgn->flags);   nr_new++;   if (insert)    memblock_insert_region(type, idx++, base,             rbase - base, nid,             flags);  }  /* area below @rend is dealt with, forget about it */  base = min(rend, end); } /* insert the remaining portion */ if (base < end) {  nr_new++;  if (insert)   memblock_insert_region(type, idx, base, end - base,            nid, flags); } if (!nr_new)  return 0; /*  * If this was the first round, resize array and repeat for actual  * insertions; otherwise, merge and return.  */ if (!insert) {  while (type->cnt + nr_new > type->max)   if (memblock_double_array(type, obase, size) < 0)    return -ENOMEM;  insert = true;  goto repeat; } else {  memblock_merge_regions(type);  return 0; }}
```

**如果 memblock 算法管理的内存为空时，将当前空间添加进去**

不为空的情况下，for_each_memblock_type(idx, type, rgn)这个循环会检查是否存在内存重叠的情况，如果有的话，则剔除重叠部分，然后将其余非重叠的部分插入 memblock。内存块的添加主要分为四种情况（其余情况与这几种类似），可以参考下图：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171713323044606.png)

如果 region 空间不够，则通过 memblock_double_array()添加新的空间，然后重试。

最后 memblock_merge_regions()会将紧挨着的内存进行合并（节点号、flag 等必须一致，节点号后面再进行介绍）。

##### **1.1.2 memblock 内存分配与回收**

到这里，memblock 内存管理的初始化基本完成，后面还有一些关于 memblock.memory 的修正，这里就不做介绍了，最后也简单介绍一下**memblock 的内存分配和回收**，即 memblock_alloc()和 memblock_free()。

```
//size即分配区块的大小，align用于字节对齐,表示分配区块的对齐大小//这里NUMA_NO_NODE指任何节点（没有节点），关于节点后面会介绍，这里节点还没初始化//MEMBLOCK_ALLOC_ACCESSIBLE指分配内存块时仅受memblock.current_limit的限制#define NUMA_NO_NODE (-1)#define MEMBLOCK_LOW_LIMIT 0#define MEMBLOCK_ALLOC_ACCESSIBLE 0static inline void * __init memblock_alloc(phys_addr_t size,  phys_addr_t align){ return memblock_alloc_try_nid(size, align, MEMBLOCK_LOW_LIMIT,          MEMBLOCK_ALLOC_ACCESSIBLE, NUMA_NO_NODE);}void * __init memblock_alloc_try_nid(   phys_addr_t size, phys_addr_t align,   phys_addr_t min_addr, phys_addr_t max_addr,   int nid){ void *ptr; memblock_dbg("%s: %llu bytes align=0x%llx nid=%d from=%pa max_addr=%pa %pS\n",       __func__, (u64)size, (u64)align, nid, &min_addr,       &max_addr, (void *)_RET_IP_); ptr = memblock_alloc_internal(size, align,        min_addr, max_addr, nid, false); if (ptr)  memset(ptr, 0, size); return ptr;}static void * __init memblock_alloc_internal(    phys_addr_t size, phys_addr_t align,    phys_addr_t min_addr, phys_addr_t max_addr,    int nid, bool exact_nid){ phys_addr_t alloc; /*  * Detect any accidental use of these APIs after slab is ready, as at  * this moment memblock may be deinitialized already and its  * internal data may be destroyed (after execution of memblock_free_all)  */ if (WARN_ON_ONCE(slab_is_available()))  return kzalloc_node(size, GFP_NOWAIT, nid); if (max_addr > memblock.current_limit)  max_addr = memblock.current_limit; alloc = memblock_alloc_range_nid(size, align, min_addr, max_addr, nid,     exact_nid); /* retry allocation without lower limit */ if (!alloc && min_addr)  alloc = memblock_alloc_range_nid(size, align, 0, max_addr, nid,      exact_nid); if (!alloc)  return NULL; return phys_to_virt(alloc);}
```

memblock_alloc_internal 返回的是分配到的内存块的虚拟地址，为 NULL 表示分配失败，关于 phys_to_virt 的实现后面再介绍，这里主要看 memblock_alloc_range_nid 的实现。

```
phys_addr_t __init memblock_alloc_range_nid(phys_addr_t size,     phys_addr_t align, phys_addr_t start,     phys_addr_t end, int nid,     bool exact_nid){ enum memblock_flags flags = choose_memblock_flags(); phys_addr_t found; if (WARN_ONCE(nid == MAX_NUMNODES, "Usage of MAX_NUMNODES is deprecated. Use NUMA_NO_NODE instead\n"))  nid = NUMA_NO_NODE; if (!align) {  /* Can't use WARNs this early in boot on powerpc */  dump_stack();  align = SMP_CACHE_BYTES; }again: found = memblock_find_in_range_node(size, align, start, end, nid,         flags); if (found && !memblock_reserve(found, size))  goto done; if (nid != NUMA_NO_NODE && !exact_nid) {  found = memblock_find_in_range_node(size, align, start,          end, NUMA_NO_NODE,          flags);  if (found && !memblock_reserve(found, size))   goto done; } if (flags & MEMBLOCK_MIRROR) {  flags &= ~MEMBLOCK_MIRROR;  pr_warn("Could not allocate %pap bytes of mirrored memory\n",   &size);  goto again; } return 0;done: if (end != MEMBLOCK_ALLOC_KASAN)  kmemleak_alloc_phys(found, size, 0, 0); return found;}
```

kmemleak 是一个检查内存泄漏的工具，这里就不做介绍了。

memblock_alloc_range_nid()首先对 align 参数进行检测，如果为零，则警告。接着函数调用 memblock_find_in_range_node() 函数**从可用内存区中找一块大小为 size 的物理内存区块**， 然后调用 memblock_reseve() 函数在找到的情况下，将这块物理内存区块加入到预留区内。

```
static phys_addr_t __init_memblock memblock_find_in_range_node(phys_addr_t size,     phys_addr_t align, phys_addr_t start,     phys_addr_t end, int nid,     enum memblock_flags flags){ phys_addr_t kernel_end, ret; /* pump up @end */ if (end == MEMBLOCK_ALLOC_ACCESSIBLE ||     end == MEMBLOCK_ALLOC_KASAN)  end = memblock.current_limit; /* avoid allocating the first page */ start = max_t(phys_addr_t, start, PAGE_SIZE); end = max(start, end); kernel_end = __pa_symbol(_end); if (memblock_bottom_up() && end > kernel_end) {  phys_addr_t bottom_up_start;  /* make sure we will allocate above the kernel */  bottom_up_start = max(start, kernel_end);  /* ok, try bottom-up allocation first */  ret = __memblock_find_range_bottom_up(bottom_up_start, end,            size, align, nid, flags);  if (ret)   return ret;  WARN_ONCE(IS_ENABLED(CONFIG_MEMORY_HOTREMOVE),     "memblock: bottom-up allocation failed, memory hotremove may be affected\n"); } return __memblock_find_range_top_down(start, end, size, align, nid,           flags);}
```

memblock 分配器分配内存是支持自低地址向高地址和自高地址向低地址的，如果 memblock_bottom_up() 函数返回 true，表示 MEMBLOCK 从低向上分配，而前面初始化的时候这个返回值其实是 false（某些情况下不一定），所以这里主要看**memblock_find_range_top_down 的实现（**memblock_find_range_bottom_up 的实现是类似的）。

```
static phys_addr_t __init_memblock__memblock_find_range_top_down(phys_addr_t start, phys_addr_t end,          phys_addr_t size, phys_addr_t align, int nid,          enum memblock_flags flags){ phys_addr_t this_start, this_end, cand; u64 i; for_each_free_mem_range_reverse(i, nid, flags, &this_start, &this_end,     NULL) {  this_start = clamp(this_start, start, end);  this_end = clamp(this_end, start, end);  if (this_end < size)   continue;  cand = round_down(this_end - size, align);  if (cand >= this_start)   return cand; } return 0;}
```

Clamp 函数可以将随机变化的数值限制在一个给定的区间[min, max]内。

***_memblock\*find_range_top_down()**通过使用 for_each_free_mem_range_reverse 宏封装调用*_next*free_mem_range_rev()函数，该函数逐一将 memblock.memory 里面的内存块信息提取出来与 memblock.reserved 的各项信息进行检验，确保返回的 this_start 和 this_end 不会与 reserved 的内存存在交叉重叠的情况。判断大小是否满足，满足的情况下，将自末端向前（因为这是 top-down 申请方式）的 size 大小的空间的起始地址（前提该地址不会超出 this_start）返回回去，至此满足要求的内存块找到了。

最后看一下**memblock_free**的实现：

```
int __init_memblock memblock_free(phys_addr_t base, phys_addr_t size){ phys_addr_t end = base + size - 1; memblock_dbg("%s: [%pa-%pa] %pS\n", __func__,       &base, &end, (void *)_RET_IP_); kmemleak_free_part_phys(base, size); return memblock_remove_range(&memblock.reserved, base, size);}
```

这里直接看最核心的部分：

```
static int __init_memblock memblock_remove_range(struct memblock_type *type,       phys_addr_t base, phys_addr_t size){ int start_rgn, end_rgn; int i, ret; ret = memblock_isolate_range(type, base, size, &start_rgn, &end_rgn); if (ret)  return ret; for (i = end_rgn - 1; i >= start_rgn; i--)  memblock_remove_region(type, i); return 0;}static void __init_memblock memblock_remove_region(struct memblock_type *type, unsigned long r){ type->total_size -= type->regions[r].size; memmove(&type->regions[r], &type->regions[r + 1],  (type->cnt - (r + 1)) * sizeof(type->regions[r])); type->cnt--; /* Special case for empty arrays */ if (type->cnt == 0) {  WARN_ON(type->total_size != 0);  type->cnt = 1;  type->regions[0].base = 0;  type->regions[0].size = 0;  type->regions[0].flags = 0;  memblock_set_region_node(&type->regions[0], MAX_NUMNODES); }}
```

其主要功能是**将指定下标索引的内存项从 memblock.reserved 中移除**。

在*_memblock*remove()里面，**memblock_isolate_range()主要作用是将要移除的物理内存区从 reserved 内存区中分离出来**，将 start_rgn 和 end_rgn（该内存区块的起始、结束索引号）返回回去。

memblock_isolate_range()返回后，接着调用 memblock_remove_region() 函数将这些索引对应的内存区块从内存区中移除，这里具体做法为调用 memmove 函数将 r 索引之后的内存区块全部往前挪一个位置，这样 r 索引对应的内存区块就被移除了，如果移除之后，内存区不含有任何内存区块，那么就初始化该内存区。

##### **1.1.3 memblock 内存管理总结**

memblock 内存区管理算法将可用可分配的内存用 memblock.memory 进行管理，已分配的内存用 memblock.reserved 进行管理，只要内存块加入到 memblock.reserved 里面就表示该内存被申请占用了，另外，在内存申请的时候，memblock 仅是把被申请到的内存加入到 memblock.reserved 中，而没有对 memblock.memory 进行任何删除或改动操作，因此申请和释放的操作都集中在 memblock.reserved。这个算法效率不高，但是却是合理的，因为在内核初始化阶段并没有太多复杂的内存操作场景，而且很多地方都是申请的内存都是永久使用的。

------

## 2.内核页表

上面有提到，内核会通过/arch/x86/kernel/head_32.S 开启页式映射，不过这里建立的页表只是临时页表，而内核真正需要建立的是内核页表。

### 2.1 **内核虚拟地址空间与用户虚拟地址空间**

为了合理地利用 4G 的内存空间，Linux 采用了 3：1 的策略，即内核占用 1G 的线性地址空间，用户占用 3G 的线性地址空间，由于历史原因，用户进程的地址范围从 0~~3G，内核地址范围从 3G~~4G，而内核的那 1GB 地址空间又称为内核虚拟地址（逻辑地址）空间，虽然内核在虚拟地址中是在高地址的，但是在物理地址中是从 0 开始的。

内核虚拟地址与用户虚拟地址，这两者都是虚拟地址，都需要经过 MMU 的翻译转换为物理地址，从硬件层面上来看，所谓的内核虚拟地址和用户虚拟地址只是权限不一样而已，但在软件层面上看，就大不相同了，当进程需要知道一个用户空间虚拟地址对应的物理地址时，linux 内核需要通过页表来得到它的物理地址，而内核空间虚拟地址是所有进程共享的，也就是说，内核在初始化时，就可以创建内核虚拟地址空间的映射，并且这里的映射就是线性映射，它基本等同于物理地址，只是它们之间有一个固定的偏移，当内核需要获取该物理地址时，可以绕开页表翻译直接通过偏移计算拿到，当然这是从软件层面上来看的，当内核去访问该页时, 硬件层面仍然走的是 MMU 翻译的全过程。

至于为什么用户虚拟地址空间不能也像内核虚拟地址空间这么做，原因是用户地址空间是随进程创建才产生的，无法事先给它分配一块连续的内存

内核通过内核虚拟地址可以直接访问到对应的物理地址，那内核如何使用其它的用户虚拟地址（0~3G）？

Linux 采用的一种折中方案是只对 1G 内核空间的前 896 MB 按**线性映射**, 剩下的 128 MB 采用动态映射，即走多级页表翻译，这样，内核态能访问空间就更多了。这里 linux 内核把这 896M 的空间称为 NORMAL 内存，剩下的 128M 称为高端内存，即 highmem。在 64 位处理器上，内核空间大大增加，所以也就不需要高端内存了，但是仍然保留了动态映射。

**动态映射**不全是为了内核空间可以访问更多的物理内存，还有一个重要原因：如果内核空间全线性映射，那么很可能就会出现内核空间碎片化而满足不了很多连续页面分配的需求（这类似于内存分段与内存分页）。因此内核空间也必须有一部分是非线性映射，从而在这碎片化物理地址空间上，用页表构造出连续虚拟地址空间（虚拟地址连续、物理地址不连续），这就是所谓的 vmalloc 空间。

到这里，可以大致知道**linux 虚拟内存的构造**：

### 2.2 **linux 内存分页**

linux 内核主要是通过内存分页来管理内存的，这里先介绍两个重要的变量：max_pfn 和 max_low_pfn。**max_pfn 为最大物理内存页面帧号，max_low_pfn 为低端内存区的最大可用页帧号**，它们的初始化如下：

```
void __init setup_arch(char **cmdline_p){    ...... max_pfn = e820__end_of_ram_pfn(); .....#ifdef CONFIG_X86_32 /* max_low_pfn get updated here */ find_low_pfn_range();#else check_x2apic(); /* How many end-of-memory variables you have, grandma! */ /* need this before calling reserve_initrd */ if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))  max_low_pfn = e820__end_of_low_ram_pfn(); else  max_low_pfn = max_pfn; high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;#endif ......    e820__memblock_setup();    ......}
```

其中 e820__end_of_ram_pfn 的实现如下，其中 E820_TYPE_RAM 代表可用物理内存类型

```
#define PAGE_SHIFT  12#ifdef CONFIG_X86_32# ifdef CONFIG_X86_PAE#  define MAX_ARCH_PFN  (1ULL<<(36-PAGE_SHIFT))# else//32位系统，1<<20，即0x100000，代表4G物理内存的最大页面帧号#  define MAX_ARCH_PFN  (1ULL<<(32-PAGE_SHIFT))# endif#else /* CONFIG_X86_32 */# define MAX_ARCH_PFN MAXMEM>>PAGE_SHIFT#endifunsigned long __init e820__end_of_ram_pfn(void){ return e820_end_pfn(MAX_ARCH_PFN, E820_TYPE_RAM);}/* * Find the highest page frame number we have available */static unsigned long __init e820_end_pfn(unsigned long limit_pfn, enum e820_type type){ int i; unsigned long last_pfn = 0; unsigned long max_arch_pfn = MAX_ARCH_PFN; for (i = 0; i < e820_table->nr_entries; i++) {  struct e820_entry *entry = &e820_table->entries[i];  unsigned long start_pfn;  unsigned long end_pfn;  if (entry->type != type)   continue;  start_pfn = entry->addr >> PAGE_SHIFT;  end_pfn = (entry->addr + entry->size) >> PAGE_SHIFT;  if (start_pfn >= limit_pfn)   continue;  if (end_pfn > limit_pfn) {   last_pfn = limit_pfn;   break;  }  if (end_pfn > last_pfn)   last_pfn = end_pfn; } if (last_pfn > max_arch_pfn)  last_pfn = max_arch_pfn; pr_info("last_pfn = %#lx max_arch_pfn = %#lx\n",  last_pfn, max_arch_pfn); return last_pfn;}
```

**e820__end_of_ram_pfn**其实就是遍历**e820_table**，得到内存块的起始地址以及内存块大小，将起始地址右移 PAGE_SHIFT，算出其起始地址对应的页面帧号，同时根据内存块大小可以算出结束地址的页号，如果结束页号大于 limit_pfn，则设置该页号为为 limit_pfn，然后通过比较得到一个 last_pfn，即系统真正的最大物理页号。

**max_low_pfn**的计算则调用到了**find_low_pfn_range**：

```
#define PFN_UP(x) (((x) + PAGE_SIZE-1) >> PAGE_SHIFT)#define PFN_DOWN(x) ((x) >> PAGE_SHIFT)#define PFN_PHYS(x) ((phys_addr_t)(x) << PAGE_SHIFT)#ifndef __pa#define __pa(x)  __phys_addr((unsigned long)(x))#endif#define VMALLOC_RESERVE  SZ_128M#define VMALLOC_END  (CONSISTENT_BASE - PAGE_SIZE)#define VMALLOC_START  ((VMALLOC_END) - VMALLOC_RESERVE)#define VMALLOC_VMADDR(x) ((unsigned long)(x))#define MAXMEM   __pa(VMALLOC_START)#define MAXMEM_PFN  PFN_DOWN(MAXMEM)void __init find_low_pfn_range(void){ /* it could update max_pfn */ if (max_pfn <= MAXMEM_PFN)  lowmem_pfn_init(); else  highmem_pfn_init();}
```

**PFN_DOWN(x)是用来返回小于 x 的最后一个物理页号，PFN_UP(x)是用来返回大于 x 的第一个物理页号，这里 x 即物理地址，而 PFN_PHYS(x)返回的是物理页号 x 对应的物理地址**。

**__pa 其实就是通过虚拟地址计算出物理地址**，这一块后面再做讲解。

将 MAXMEM 展开一下可得

```
#ifdef CONFIG_HIGHMEM#define CONSISTENT_BASE  ((PKMAP_BASE) - (SZ_2M))#define CONSISTENT_END  (PKMAP_BASE)#else#define CONSISTENT_BASE  (FIXADDR_START - SZ_2M)#define CONSISTENT_END  (FIXADDR_START)#endif#define SZ_2M    0x00200000#define SZ_128M    0x08000000#define MAXMEM              __pa(VMALLOC_END – PAGE_OFFSET – __VMALLOC_RESERVE)//进一步展开#define MAXMEM              __pa(CONSISTENT_BASE - PAGE_SIZE – PAGE_OFFSET – SZ_128M)//再进一步展开#define MAXMEM              __pa((PKMAP_BASE) - (SZ_2M) - PAGE_SIZE – PAGE_OFFSET – SZ_128M)
```

下面这一部分就涉及到高端内存的构成了，其中**PKMAP_BASE 是持久映射空间（KMAP 空间，持久映射区）的起始地址，LAST_PKMAP 则是持久映射空间的映射页面数，而 FIXADDR_TOP 是固定映射区（临时内核映射区）的末尾，FIXADDR_START 是固定映射区起始地址**，其中的*_end*of_permanent_fixed_addresses 是固定映射的一个标志（一个枚举值，具体可以参考\arch\x86\include\asm\fixmap.h 里的 enum fixed_addresses）。最后的 VMALLOC_END 即为动态映射区的末尾。

```
//临时映射//-4096(4KB) -> 0xfffff000#define __FIXADDR_TOP (-PAGE_SIZE)#define FIXADDR_TOP ((unsigned long)__FIXADDR_TOP)#define FIXADDR_SIZE  (__end_of_permanent_fixed_addresses << PAGE_SHIFT)#define FIXADDR_START  (FIXADDR_TOP - FIXADDR_SIZE)#define FIXADDR_TOT_SIZE (__end_of_fixed_addresses << PAGE_SHIFT)#define FIXADDR_TOT_START (FIXADDR_TOP - FIXADDR_TOT_SIZE)//持久内核映射#ifdef CONFIG_X86_PAE#define LAST_PKMAP 512#else#define LAST_PKMAP 1024#endif#define CPU_ENTRY_AREA_BASE \ ((FIXADDR_TOT_START - PAGE_SIZE*(CPU_ENTRY_AREA_PAGES+1)) & PMD_MASK)#define LDT_BASE_ADDR  \ ((CPU_ENTRY_AREA_BASE - PAGE_SIZE) & PMD_MASK)#define LDT_END_ADDR  (LDT_BASE_ADDR + PMD_SIZE)#define PKMAP_BASE  \ ((LDT_BASE_ADDR - PAGE_SIZE) & PMD_MASK)//动态映射//0xffffff80<<20 -> 0xf8000000 -> 4,160,749,568 -> 3948MB -> 3GB+896MB 与上述一致#define high_memory (-128UL << 20)//8MB#define VMALLOC_OFFSET (8 * 1024 * 1024)#define VMALLOC_START ((unsigned long)high_memory + VMALLOC_OFFSET)#ifdef CONFIG_HIGHMEM# define VMALLOC_END (PKMAP_BASE - 2 * PAGE_SIZE)#else# define VMALLOC_END (LDT_BASE_ADDR - 2 * PAGE_SIZE)#endif
```

直接看图~

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171714069763974.png)

PAGE_OFFSET 代表的是内核空间和用户空间对虚拟地址空间的划分，对不同的体系结构不同。比如在 32 位系统中 3G-4G 属于内核使用的内存空间，所以 PAGE_OFFSET = 0xC0000000

内核空间如上图，可分为直接内存映射区和高端内存映射区，其中直接内存映射区是指 3G 到 3G+896M 的线性空间，直接对应物理地址就是 0 到 896M（前提是有超过 896M 的物理内存），其中 896M 是 high_memory 值，使用 kmalloc()/kfree()接口申请释放内存；而高端内存映射区则是超过 896M 物理内存的空间，**它又分为动态映射区、持久映射区和固定映射区**。

动态内存映射区，又称之为 vmalloc 映射区或非连续映射区，是指 VMALLOC_START 到 VMALLOC_END 的地址空间，申请释放内存的接口是 vmalloc()/vfree()，通常用于将非连续的物理内存映射为连续的线性地址内存空间；

而持久映射区，又称之为 KMAP 区或永久映射区，是指自 PKMAP_BASE 开始共 LAST_PKMAP 个页面大小的空间，操作接口是 kmap()/kunmap()，用于将高端内存长久映射到内存虚拟地址空间中；

最后的固定映射区，有时候也称为临时映射区，是指 FIXADDR_START 到 FIXADDR_TOP 的地址空间，操作接口是 kmap_atomic()/kummap_atomic()，用于解决持久映射不能用于中断处理程序而增加的临时内核映射。

上面的**MAXMEM_PFN 其实就是用来判断是否初始化（启用）高端内存，当内存物理页数本来就小于低端内存的最大物理页数时，就没有高端地址映射**。

这里接着看 max_low_pfn 的初始化，进入 highmem_pfn_init(void)。

```
static void __init highmem_pfn_init(void){ max_low_pfn = MAXMEM_PFN; if (highmem_pages == -1)  highmem_pages = max_pfn - MAXMEM_PFN; if (highmem_pages + MAXMEM_PFN < max_pfn)  max_pfn = MAXMEM_PFN + highmem_pages; if (highmem_pages + MAXMEM_PFN > max_pfn) {  printk(KERN_WARNING MSG_HIGHMEM_TOO_SMALL,   pages_to_mb(max_pfn - MAXMEM_PFN),   pages_to_mb(highmem_pages));  highmem_pages = 0; }#ifndef CONFIG_HIGHMEM /* Maximum memory usable is what is directly addressable */ printk(KERN_WARNING "Warning only %ldMB will be used.\n", MAXMEM>>20); if (max_pfn > MAX_NONPAE_PFN)  printk(KERN_WARNING "Use a HIGHMEM64G enabled kernel.\n"); else  printk(KERN_WARNING "Use a HIGHMEM enabled kernel.\n"); max_pfn = MAXMEM_PFN;#else /* !CONFIG_HIGHMEM */#ifndef CONFIG_HIGHMEM64G if (max_pfn > MAX_NONPAE_PFN) {  max_pfn = MAX_NONPAE_PFN;  printk(KERN_WARNING MSG_HIGHMEM_TRIMMED); }#endif /* !CONFIG_HIGHMEM64G */#endif /* !CONFIG_HIGHMEM */}
```

highmem_pfn_init 的主要工作其实就是**把 max_low_pfn 设置为 MAXMEM_PFN，将 highmem_pages 设置为 max_pfn – MAXMEM_PFN**，至此，max_pfn 和 max_low_pfn 初始化完毕。

### 2.3 **低端内存初始化**

回到 setup_arch 函数：

```
void __init setup_arch(char **cmdline_p){    ...... max_pfn = e820__end_of_ram_pfn(); .....#ifdef CONFIG_X86_32 /* max_low_pfn get updated here */ find_low_pfn_range();#else check_x2apic(); /* How many end-of-memory variables you have, grandma! */ /* need this before calling reserve_initrd */ if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))  max_low_pfn = e820__end_of_low_ram_pfn(); else  max_low_pfn = max_pfn; high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;#endif ...... early_alloc_pgt_buf(); //<-------------------------------------    /*  * Need to conclude brk, before e820__memblock_setup()  *  it could use memblock_find_in_range, could overlap with  *  brk area.  */ reserve_brk(); //<-------------------------------------------    ......    e820__memblock_setup(); ......}
```

**early_alloc_pgt_buf()即申请页表缓冲区**

```
#define INIT_PGD_PAGE_COUNT      6#define INIT_PGT_BUF_SIZE (INIT_PGD_PAGE_COUNT * PAGE_SIZE)RESERVE_BRK(early_pgt_alloc, INIT_PGT_BUF_SIZE);void  __init early_alloc_pgt_buf(void){ unsigned long tables = INIT_PGT_BUF_SIZE; phys_addr_t base; base = __pa(extend_brk(tables, PAGE_SIZE)); pgt_buf_start = base >> PAGE_SHIFT; pgt_buf_end = pgt_buf_start; pgt_buf_top = pgt_buf_start + (tables >> PAGE_SHIFT);}
```

pgt_buf_start：标识该缓冲空间的起始物理内存页框号；

pgt_buf_end：初始化时和 pgt_buf_start 是同一个值，但是它是用于表示该空间未被申请使用的空间起始页框号；

pgt_buf_top：则是用来表示缓冲空间的末尾，存放的是该末尾的页框号

INIT_PGT_BUF_SIZE 即 24KB，这里直接看最关键的部分：**extend_brk**

```
unsigned long _brk_start = (unsigned long)__brk_base;unsigned long _brk_end   = (unsigned long)__brk_base;void * __init extend_brk(size_t size, size_t align){ size_t mask = align - 1; void *ret; BUG_ON(_brk_start == 0); BUG_ON(align & mask); _brk_end = (_brk_end + mask) & ~mask; BUG_ON((char *)(_brk_end + size) > __brk_limit); ret = (void *)_brk_end; _brk_end += size; memset(ret, 0, size); return ret;}
```

BUG_ON()函数是内核标记 bug、提供断言并输出信息的常用手段

*_brk*base 相关的初始化可以参考 arch\x86\kernel\vmlinux.lds.S

在 setup_arch()中，紧接着 early_alloc_pgt_buf()还有 reserve_brk()函数

```
static void __init reserve_brk(void){ if (_brk_end > _brk_start)  memblock_reserve(__pa_symbol(_brk_start),     _brk_end - _brk_start); /* Mark brk area as locked down and no longer taking any    new allocations */ _brk_start = 0;}
```

这个地方主要是**将 early_alloc_pgt_buf()申请的空间在 membloc 中做 reserved 保留操作**，避免被其它地方申请使用而引发异常

回到 setup_arch 函数：

```
void __init setup_arch(char **cmdline_p){    ...... max_pfn = e820__end_of_ram_pfn(); //max_pfn初始化 ...... find_low_pfn_range(); //max_low_pfn、高端内存初始化 ...... ...... early_alloc_pgt_buf(); //页表缓冲区分配 reserve_brk(); //缓冲区加入memblock.reserve    ......    e820__memblock_setup(); //memblock.memory空间初始化 启动 ......    init_mem_mapping(); //<-----------------------------    ......}
```

**init_mem_mapping()即低端内存内核页表初始化的关键函数**

```
#define ISA_END_ADDRESS  0x00100000 //1MBvoid __init init_mem_mapping(void){ unsigned long end; pti_check_boottime_disable(); probe_page_size_mask(); setup_pcid();#ifdef CONFIG_X86_64 end = max_pfn << PAGE_SHIFT;#else end = max_low_pfn << PAGE_SHIFT;#endif /* the ISA range is always mapped regardless of memory holes */ init_memory_mapping(0, ISA_END_ADDRESS, PAGE_KERNEL); /* Init the trampoline, possibly with KASLR memory offset */ init_trampoline(); /*  * If the allocation is in bottom-up direction, we setup direct mapping  * in bottom-up, otherwise we setup direct mapping in top-down.  */ if (memblock_bottom_up()) {  unsigned long kernel_end = __pa_symbol(_end);  memory_map_bottom_up(kernel_end, end);  memory_map_bottom_up(ISA_END_ADDRESS, kernel_end); } else {  memory_map_top_down(ISA_END_ADDRESS, end); }#ifdef CONFIG_X86_64 if (max_pfn > max_low_pfn) {  /* can we preseve max_low_pfn ?*/  max_low_pfn = max_pfn; }#else early_ioremap_page_table_range_init();#endif load_cr3(swapper_pg_dir); __flush_tlb_all(); x86_init.hyper.init_mem_mapping(); early_memtest(0, max_pfn_mapped << PAGE_SHIFT);}
```

probe_page_size_mask()主要作用是初始化直接映射变量（直接映射区相关）以及根据配置来控制 CR4 寄存器的置位，用于后面分页时页面大小的判定。

上面 init_memory_mapping 的参数 ISA_END_ADDRESS 表示 ISA 总线上设备的末尾地址。

**init_memory_mapping(0, ISA_END_ADDRESS, PAGE_KERNEL)初始化 0 ~ 1MB 的物理地址**，一般内核启动时被安装在 1MB 开始处，这里初始化完成之后会调用到 memory_map_bottom_up 或者 memory_map_top_down，后面就是初始化 1MB ~ 内核结束地址 这块物理地址区域 ，最后也会回归到 init_memory_mapping 的调用，因此这里不做过多的介绍，直接看 init_memory_mapping()：

```
#ifdef CONFIG_X86_32#define NR_RANGE_MR 3#else /* CONFIG_X86_64 */#define NR_RANGE_MR 5#endifstruct map_range { unsigned long start; unsigned long end; unsigned page_size_mask;};unsigned long __ref init_memory_mapping(unsigned long start,     unsigned long end, pgprot_t prot){ struct map_range mr[NR_RANGE_MR]; unsigned long ret = 0; int nr_range, i; pr_debug("init_memory_mapping: [mem %#010lx-%#010lx]\n",        start, end - 1); memset(mr, 0, sizeof(mr)); nr_range = split_mem_range(mr, 0, start, end); for (i = 0; i < nr_range; i++)  ret = kernel_physical_mapping_init(mr[i].start, mr[i].end,         mr[i].page_size_mask,         prot); add_pfn_range_mapped(start >> PAGE_SHIFT, ret >> PAGE_SHIFT); return ret >> PAGE_SHIFT;}
```

struct map_range，该结构是用来保存内存段信息，其包含了一个段的起始地址、结束地址，以及该段是按多大的页面进行分页(4K、2M、1G，1G 是 64 位的，所以这里不提及)。

```
static int __meminit split_mem_range(struct map_range *mr, int nr_range,         unsigned long start,         unsigned long end){ unsigned long start_pfn, end_pfn, limit_pfn; unsigned long pfn; int i; //返回小于...的最后一个物理页号 limit_pfn = PFN_DOWN(end); pfn = start_pfn = PFN_DOWN(start); if (pfn == 0)  end_pfn = PFN_DOWN(PMD_SIZE); else  end_pfn = round_up(pfn, PFN_DOWN(PMD_SIZE)); if (end_pfn > limit_pfn)  end_pfn = limit_pfn; if (start_pfn < end_pfn) {  nr_range = save_mr(mr, nr_range, start_pfn, end_pfn, 0);  pfn = end_pfn; } /* big page (2M) range */ start_pfn = round_up(pfn, PFN_DOWN(PMD_SIZE)); end_pfn = round_down(limit_pfn, PFN_DOWN(PMD_SIZE)); if (start_pfn < end_pfn) {  nr_range = save_mr(mr, nr_range, start_pfn, end_pfn,    page_size_mask & (1<<PG_LEVEL_2M));  pfn = end_pfn; } /* tail is not big page (2M) alignment */ start_pfn = pfn; end_pfn = limit_pfn; nr_range = save_mr(mr, nr_range, start_pfn, end_pfn, 0); if (!after_bootmem)  adjust_range_page_size_mask(mr, nr_range); /* try to merge same page size and continuous */ for (i = 0; nr_range > 1 && i < nr_range - 1; i++) {  unsigned long old_start;  if (mr[i].end != mr[i+1].start ||      mr[i].page_size_mask != mr[i+1].page_size_mask)   continue;  /* move it */  old_start = mr[i].start;  memmove(&mr[i], &mr[i+1],   (nr_range - 1 - i) * sizeof(struct map_range));  mr[i--].start = old_start;  nr_range--; } for (i = 0; i < nr_range; i++)  pr_debug(" [mem %#010lx-%#010lx] page %s\n",    mr[i].start, mr[i].end - 1,    page_size_string(&mr[i])); return nr_range;}
```

PMD_SIZE 用于计算由页中间目录的一个单独表项所映射的区域大小，也就是一个页表的大小。

split_mem_range()根据传入的内存 start 和 end 做四舍五入的对齐操作（即 round_up 和 round_down）

```
#define __round_mask(x, y) ((__typeof__(x))((y)-1))#define round_up(x, y) ((((x)-1) | __round_mask(x, y))+1)//可以理解为：#define round_up(x, y) (((x)+(y) - 1)/(y))*(y))#define round_down(x, y) ((x) & ~__round_mask(x, y))//可以理解为：#define round_down(x, y) ((x/y) * y)
```

round_up 宏依靠整数除法来完成这项工作，仅当两个参数均为整数时，它才有效，x 是需要四舍五入的数字，y 是应该四舍五入的间隔，也就是说，round_up(12,5)应返回 15，因为 15 是大于 12 的 5 的第一个间隔，而 round_down(12,5)应返回 10。

split_mem_range()会**根据对齐的情况，把开始、末尾的不对齐部分及中间部分分成了三段**，使用 save_mr()将其存放在 init_mem_mapping()的局部变量数组 mr 中。**划分开来主要是为了让各部分可以映射到不同大小的页面，最后如果相邻两部分映射页面的大小是一致的，则将其合并。**

可以通过 dmesg 得到划分的情况（以下是我私服的划分情况，不过是 64 位的……）

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171714388654695.png)

初始化完内存段信息 mr 之后，就行了进入到**kernel_physical_mapping_init**，这个函数是建立内核页表的最为关键的一步，负责处理**物理内存的映射**。

在 2.6.11 后，Linux 采用**四级分页模型**，这四级页目录分别为：

- 页全局目录(Page Global Directory)
- 页上级目录(Page Upper Directory)
- 页中间目录(Page Middle Directory)
- 页表(Page Table)

对于没有启动 PAE（物理地址扩展）的 32 位系统，Linux 虽然也采用四级分页模型，但本质上只用到了两级分页，Linux 通过将”页上级目录”位域和“页中间目录”位域全为 0 来达到使用两级分页的目的，但为了保证程序能 32 位和 64 系统上都能运行，内核保留了页上级目录和页中间目录在指针序列中的位置，它们的页目录数都被内核置为 1，并把这 2 个页目录项映射到适合的全局目录项。

开启 PAE 后，32 位系统寻址方式将大大改变，这时候使用的是三级页表，即页上级目录其实没有真正用到。

这里不考虑 PAE

**PAGE_OFFSET 代表的是内核空间和用户空间对虚拟地址空间的划分**，对不同的体系结构不同。比如在 32 位系统中 3G-4G 属于内核使用的内存空间，所以 PAGE_OFFSET = 0xC0000000

```
unsigned long __initkernel_physical_mapping_init(unsigned long start,        unsigned long end,        unsigned long page_size_mask,        pgprot_t prot){ int use_pse = page_size_mask == (1<<PG_LEVEL_2M); unsigned long last_map_addr = end; unsigned long start_pfn, end_pfn; pgd_t *pgd_base = swapper_pg_dir; int pgd_idx, pmd_idx, pte_ofs; unsigned long pfn; pgd_t *pgd; pmd_t *pmd; pte_t *pte; unsigned pages_2m, pages_4k; int mapping_iter; start_pfn = start >> PAGE_SHIFT; end_pfn = end >> PAGE_SHIFT; mapping_iter = 1; if (!boot_cpu_has(X86_FEATURE_PSE))  use_pse = 0;repeat: pages_2m = pages_4k = 0; pfn = start_pfn; //pfn保存起始页框号 pgd_idx = pgd_index((pfn<<PAGE_SHIFT) + PAGE_OFFSET); //低端内存的起始地址对应的pgd的偏移 pgd = pgd_base + pgd_idx; //得到起始页框对应的pgd //由pgd开始遍历    for (; pgd_idx < PTRS_PER_PGD; pgd++, pgd_idx++) {  pmd = one_md_table_init(pgd);//申请得到一个pmd表  if (pfn >= end_pfn)   continue;#ifdef CONFIG_X86_PAE  pmd_idx = pmd_index((pfn<<PAGE_SHIFT) + PAGE_OFFSET);  pmd += pmd_idx;#else  pmd_idx = 0;#endif        //遍历pmd表,对于未激活PAE的32位系统，PTRS_PER_PMD为1，激活PAE则为512  for (; pmd_idx < PTRS_PER_PMD && pfn < end_pfn;       pmd++, pmd_idx++) {   unsigned int addr = pfn * PAGE_SIZE + PAGE_OFFSET;   /*    * Map with big pages if possible, otherwise    * create normal page tables:    */   if (use_pse) {    unsigned int addr2;    pgprot_t prot = PAGE_KERNEL_LARGE;    /*     * first pass will use the same initial     * identity mapping attribute + _PAGE_PSE.     */    pgprot_t init_prot =     __pgprot(PTE_IDENT_ATTR |       _PAGE_PSE);    pfn &= PMD_MASK >> PAGE_SHIFT;    addr2 = (pfn + PTRS_PER_PTE-1) * PAGE_SIZE +     PAGE_OFFSET + PAGE_SIZE-1;    if (__is_kernel_text(addr) ||        __is_kernel_text(addr2))     prot = PAGE_KERNEL_LARGE_EXEC;    pages_2m++;    if (mapping_iter == 1)     set_pmd(pmd, pfn_pmd(pfn, init_prot));    else     set_pmd(pmd, pfn_pmd(pfn, prot));    pfn += PTRS_PER_PTE;    continue;   }   pte = one_page_table_init(pmd); //创建一个页表   //得到pfn在page table中的偏移并定位到具体的pte   pte_ofs = pte_index((pfn<<PAGE_SHIFT) + PAGE_OFFSET);   pte += pte_ofs;            //由pte开始遍历page table   for (; pte_ofs < PTRS_PER_PTE && pfn < end_pfn;        pte++, pfn++, pte_ofs++, addr += PAGE_SIZE) {    pgprot_t prot = PAGE_KERNEL;    /*     * first pass will use the same initial     * identity mapping attribute.     */    pgprot_t init_prot = __pgprot(PTE_IDENT_ATTR);    if (__is_kernel_text(addr)) //如果处于内核代码段，权限设为可执行     prot = PAGE_KERNEL_EXEC;    pages_4k++;                //设置pte与pfn关联    if (mapping_iter == 1) {     set_pte(pte, pfn_pte(pfn, init_prot)); //第一次执行将权限位设为init_prot     last_map_addr = (pfn << PAGE_SHIFT) + PAGE_SIZE;    } else     set_pte(pte, pfn_pte(pfn, prot)); //之后的执行将权限位置为prot   }  } } if (mapping_iter == 1) {  /*   * update direct mapping page count only in the first   * iteration.   */  update_page_count(PG_LEVEL_2M, pages_2m);  update_page_count(PG_LEVEL_4K, pages_4k);  /*   * local global flush tlb, which will flush the previous   * mappings present in both small and large page TLB's.   */  __flush_tlb_all(); //TLB全部刷新  /*   * Second iteration will set the actual desired PTE attributes.   */  mapping_iter = 2;  goto repeat; } return last_map_addr;}
```

**内核的内核页全局目录的基地址保存在 swapper_pg_dir 全局变量中**，但需要使用主内核页表时系统会把这个变量的值放入 cr3 寄存器，详细可参考/arch/x86/kernel/head_32.s。

**Linux 分别采用 pgd_t、pud_t、pmd_t 和 pte_t 四种数据结构来表示页全局目录项、页上级目录项、页中间目录项和页表项**。这四种数据结构本质上都是无符号长整型，Linux 为了更严格数据类型检查，将无符号长整型分别封装成四种不同的页表项。如果不采用这种方法，那么一个无符号长整型数据可以传入任何一个与四种页表相关的函数或宏中，这将大大降低程序的健壮性。下面仅列出 pgd_t 类型的内核源码实现，其他类型与此类似

```
typedef unsigned long   pgdval_t;typedef struct { pgdval_t pgd; } pgd_t;#define pgd_val(x)      native_pgd_val(x)static inline pgdval_t native_pgd_val(pgd_t pgd){      return pgd.pgd;}
```

这里需要区别指向页表项的指针和页表项所代表的数据，如果已知一个 pgd_t 类型的指针 pgd，那么通过 pgd_val(*pgd)即可获得该页表项(也就是一个无符号长整型数据)

**PAGE_SHIFT，PMD_SHIFT，PUD_SHIFT，PGDIR_SHIFT，对应相应的页目录所能映射的区域大小的位数**，如 PAGE_SHIFT 为 12，即页面大小为 4k。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171714575379377.png)

**PTRS_PER_PTE, PTRS_PER_PMD, PTRS_PER_PUD, PTRS_PER_PGD，对应相应页目录中的表项数**。32 位系统下，当 PAE 被禁止时，他们的值分别为 1024,，1，1 和 1024，也就是说只使用两级分页

pgd_index(addr)，pud_index,(addr)，pmd_index(addr)，pte_index(addr)，取 addr 在该目录中的索引

pud_offset(pgd,addr), pmd_offset(pud,addr), pte_offset(pmd,addr)，以 pmd_offset 为例，线性地址 addr 对应的 pmd 索引在在 pud 指定的 pmd 表的偏移地址。在两级或三级分页系统中，pmd_offset 和 pud_offset 都返回页全局目录的地址

至此，低端内存的物理地址和虚拟地址之间的映射关系已全部建立起来了

回到前面的 init_memory_mapping()函数，它的最后一个函数调用为 add_pfn_range_mapped()

```
struct range pfn_mapped[E820_MAX_ENTRIES];int nr_pfn_mapped;static void add_pfn_range_mapped(unsigned long start_pfn, unsigned long end_pfn){ nr_pfn_mapped = add_range_with_merge(pfn_mapped, E820_MAX_ENTRIES,          nr_pfn_mapped, start_pfn, end_pfn); nr_pfn_mapped = clean_sort_range(pfn_mapped, E820_MAX_ENTRIES); max_pfn_mapped = max(max_pfn_mapped, end_pfn); if (start_pfn < (1UL<<(32-PAGE_SHIFT)))  max_low_pfn_mapped = max(max_low_pfn_mapped,      min(end_pfn, 1UL<<(32-PAGE_SHIFT)));}
```

该函数主要是将前面完成内存映射的物理页框范围加入到全局数组**pfn_mapped**中，其中 nr_pfn_mapped 用于表示数组中的有效项数量，之后可以通过内核函数 pfn_range_is_mapped 来判断指定的物理内存是否被映射，避免重复映射的情况

### 2.4 **固定映射区初始化**

再回到更前面的 init_mem_mapping()函数，**early_ioremap_page_table_range_init**()用来**建立高端内存的固定映射区页表，与低端内存的页表初始化不同的是，固定映射区的页表只是被分配，相应的 PTE 项并未初始化**，这个工作交由后面的**set_fixmap**()函数将相关的固定映射区页表与物理内存进行关联

```
# define PMD_MASK (~(PMD_SIZE - 1))void __init early_ioremap_page_table_range_init(void){ pgd_t *pgd_base = swapper_pg_dir; unsigned long vaddr, end; /*  * Fixed mappings, only the page table structure has to be  * created - mappings will be set by set_fixmap():  */ vaddr = __fix_to_virt(__end_of_fixed_addresses - 1) & PMD_MASK; end = (FIXADDR_TOP + PMD_SIZE - 1) & PMD_MASK; page_table_range_init(vaddr, end, pgd_base); early_ioremap_reset();}
```

PUD_MASK、PMD_MASK、PGDIR_MASK，这些 MASK 的作用是：从给定地址中提取某些分量，用给定地址与对应的 MASK 位与操作之后即可获得各个分量，上面的操作为屏蔽低位

这里可以先具体看一下固定映射区的组成

每个固定映射区索引都以枚举类型的形式定义在 enum fixed_addresses 中

```
enum fixed_addresses {#ifdef CONFIG_X86_32 FIX_HOLE,#else#ifdef CONFIG_X86_VSYSCALL_EMULATION VSYSCALL_PAGE = (FIXADDR_TOP - VSYSCALL_ADDR) >> PAGE_SHIFT,#endif#endif FIX_DBGP_BASE, FIX_EARLYCON_MEM_BASE,#ifdef CONFIG_PROVIDE_OHCI1394_DMA_INIT FIX_OHCI1394_BASE,#endif#ifdef CONFIG_X86_LOCAL_APIC FIX_APIC_BASE, /* local (CPU) APIC) -- required for SMP or not */#endif#ifdef CONFIG_X86_IO_APIC FIX_IO_APIC_BASE_0, FIX_IO_APIC_BASE_END = FIX_IO_APIC_BASE_0 + MAX_IO_APICS - 1,#endif#ifdef CONFIG_X86_32    //这里即为固定映射区 FIX_KMAP_BEGIN, /* reserved pte's for temporary kernel mappings */ FIX_KMAP_END = FIX_KMAP_BEGIN+(KM_TYPE_NR*NR_CPUS)-1,#ifdef CONFIG_PCI_MMCONFIG FIX_PCIE_MCFG,#endif#endif#ifdef CONFIG_PARAVIRT_XXL FIX_PARAVIRT_BOOTMAP,#endif#ifdef CONFIG_X86_INTEL_MID FIX_LNW_VRTC,#endif#ifdef CONFIG_ACPI_APEI_GHES /* Used for GHES mapping from assorted contexts */ FIX_APEI_GHES_IRQ, FIX_APEI_GHES_NMI,#endif __end_of_permanent_fixed_addresses, /*  * 512 temporary boot-time mappings, used by early_ioremap(),  * before ioremap() is functional.  *  * If necessary we round it up to the next 512 pages boundary so  * that we can have a single pmd entry and a single pte table:  */#define NR_FIX_BTMAPS  64#define FIX_BTMAPS_SLOTS 8#define TOTAL_FIX_BTMAPS (NR_FIX_BTMAPS * FIX_BTMAPS_SLOTS) FIX_BTMAP_END =  (__end_of_permanent_fixed_addresses ^   (__end_of_permanent_fixed_addresses + TOTAL_FIX_BTMAPS - 1)) &  -PTRS_PER_PTE  ? __end_of_permanent_fixed_addresses + TOTAL_FIX_BTMAPS -    (__end_of_permanent_fixed_addresses & (TOTAL_FIX_BTMAPS - 1))  : __end_of_permanent_fixed_addresses, FIX_BTMAP_BEGIN = FIX_BTMAP_END + TOTAL_FIX_BTMAPS - 1,#ifdef CONFIG_X86_32 FIX_WP_TEST,#endif#ifdef CONFIG_INTEL_TXT FIX_TBOOT_BASE,#endif __end_of_fixed_addresses};#define __fix_to_virt(x) (FIXADDR_TOP - ((x) << PAGE_SHIFT))
```

一个索引对应一个 4KB 的页框，**固定映射区的结束地址为 FIXADDR_TOP**，即 0xfffff000(4G-4K)，固定映射区是反向生长的，也就是说第一个索引对应的地址离 FIXADDR_TOP 最近。另外宏*_fix*to_virt(idx)可以通过索引来计算相应的固定映射区域的线性地址

```
//初始化pgd_base指向的页全局目录中start到end这个范围的线性地址，整个函数结束后只是初始化好了页中间目录项对应的页表，但是页表中的页表项并没有初始化static void __initpage_table_range_init(unsigned long start, unsigned long end, pgd_t *pgd_base){ int pgd_idx, pmd_idx; unsigned long vaddr; pgd_t *pgd; pmd_t *pmd; pte_t *pte = NULL; unsigned long count = page_table_range_init_count(start, end); void *adr = NULL; if (count)  adr = alloc_low_pages(count); vaddr = start; pgd_idx = pgd_index(vaddr); //得到vaddr对应的pgd索引 pmd_idx = pmd_index(vaddr); //得到vaddr对应的pmd索引 pgd = pgd_base + pgd_idx;   //得到pgd项 for ( ; (pgd_idx < PTRS_PER_PGD) && (vaddr != end); pgd++, pgd_idx++) {  pmd = one_md_table_init(pgd); //得到pmd起始项  pmd = pmd + pmd_index(vaddr); //得到偏移后的pmd  for (; (pmd_idx < PTRS_PER_PMD) && (vaddr != end);       pmd++, pmd_idx++) {            //建立pte表并检查vaddr是否对应内核临时映射区,若是则重新申请一个页表来保存pte表   pte = page_table_kmap_check(one_page_table_init(pmd),          pmd, vaddr, pte, &adr);   vaddr += PMD_SIZE;  }  pmd_idx = 0; }}
```

先看 page_table_range_init_count 函数

```
static unsigned long __initpage_table_range_init_count(unsigned long start, unsigned long end){ unsigned long count = 0;#ifdef CONFIG_HIGHMEM int pmd_idx_kmap_begin = fix_to_virt(FIX_KMAP_END) >> PMD_SHIFT; int pmd_idx_kmap_end = fix_to_virt(FIX_KMAP_BEGIN) >> PMD_SHIFT; int pgd_idx, pmd_idx; unsigned long vaddr; if (pmd_idx_kmap_begin == pmd_idx_kmap_end)  return 0; vaddr = start; pgd_idx = pgd_index(vaddr); pmd_idx = pmd_index(vaddr); for ( ; (pgd_idx < PTRS_PER_PGD) && (vaddr != end); pgd_idx++) {  for (; (pmd_idx < PTRS_PER_PMD) && (vaddr != end);       pmd_idx++) {   if ((vaddr >> PMD_SHIFT) >= pmd_idx_kmap_begin &&       (vaddr >> PMD_SHIFT) <= pmd_idx_kmap_end)    count++;   vaddr += PMD_SIZE;  }  pmd_idx = 0; }#endif return count;}
```

**page_table_range_init_count**()用来计算临时映射区间的页表数量。FIXADDR_START 到 FIXADDR_TOP 即整个固定映射区，就如上面所提到的里面有多个索引标识的不同功能的映射区间，而其中的一个区间**FIX_KMAP_BEGIN 到 FIX_KMAP_END 是临时映射区间**。这里再看一下两者的定义：

```
FIX_KMAP_BEGIN, /* reserved pte's for temporary kernel mappings */FIX_KMAP_END = FIX_KMAP_BEGIN+(KM_TYPE_NR*NR_CPUS)-1,
```

固定映射是在编译时就确定的地址空间，但是它对应的物理页可以是任意的，每一个固定地址中的项都是一页范围,这些地址的用途是固定的，这是该区间被设计的最主要的用途。临时映射区间也叫原子映射，它可以用在不能睡眠的地方，如中断处理程序中，因为临时映射获取映射时是绝对不会阻塞的，上面的 KM_TYPE_NR 可以理解为临时内核映射的最大数量，NR_CPUS 则表示 CPU 数量。

如果返回的页表数量不为 0，则 page_table_range_init()函数还会调用 alloc_low_pages()：

```
__ref void *alloc_low_pages(unsigned int num){ unsigned long pfn; int i; if (after_bootmem) {  unsigned int order;  order = get_order((unsigned long)num << PAGE_SHIFT);  return (void *)__get_free_pages(GFP_ATOMIC | __GFP_ZERO, order); } if ((pgt_buf_end + num) > pgt_buf_top || !can_use_brk_pgt) {  unsigned long ret = 0;  if (min_pfn_mapped < max_pfn_mapped) {   ret = memblock_find_in_range(     min_pfn_mapped << PAGE_SHIFT,     max_pfn_mapped << PAGE_SHIFT,     PAGE_SIZE * num , PAGE_SIZE);  }  if (ret)   memblock_reserve(ret, PAGE_SIZE * num);  else if (can_use_brk_pgt)   ret = __pa(extend_brk(PAGE_SIZE * num, PAGE_SIZE));  if (!ret)   panic("alloc_low_pages: can not alloc memory");  pfn = ret >> PAGE_SHIFT; } else {  pfn = pgt_buf_end;  pgt_buf_end += num; } for (i = 0; i < num; i++) {  void *adr;  adr = __va((pfn + i) << PAGE_SHIFT);  clear_page(adr); } return __va(pfn << PAGE_SHIFT);}
```

**alloc_low_pages**函数根据前面 early_alloc_pgt_buf()申请保留的页表缓冲空间使用情况来判断是从页表缓冲空间中申请内存还是通过 memblock 算法申请页表内存，页表缓冲空间空间足够的话就在页表缓冲空间中分配。

回到 page_table_range_init()，其中**one_md_table_init**()是用于申请新物理页作为页中间目录的，不过这里分析的是 x86 非 PAE 环境，不存在页中间目录，因此实际上返回的仍是入参，代码如下：

```
static pmd_t * __init one_md_table_init(pgd_t *pgd){ p4d_t *p4d; pud_t *pud; pmd_t *pmd_table;#ifdef CONFIG_X86_PAE if (!(pgd_val(*pgd) & _PAGE_PRESENT)) {  pmd_table = (pmd_t *)alloc_low_page();  paravirt_alloc_pmd(&init_mm, __pa(pmd_table) >> PAGE_SHIFT);  set_pgd(pgd, __pgd(__pa(pmd_table) | _PAGE_PRESENT));  p4d = p4d_offset(pgd, 0);  pud = pud_offset(p4d, 0);  BUG_ON(pmd_table != pmd_offset(pud, 0));  return pmd_table; }#endif p4d = p4d_offset(pgd, 0); pud = pud_offset(p4d, 0); pmd_table = pmd_offset(pud, 0); return pmd_table;}
```

接着的是**page_table_kmap_check**()，其入参调用的**one_page_table_init**()是用于当入参 pmd 没有页表指向时，创建页表并使其指向被创建的页表。

```
static pte_t * __init one_page_table_init(pmd_t *pmd){ if (!(pmd_val(*pmd) & _PAGE_PRESENT)) {  pte_t *page_table = (pte_t *)alloc_low_page();  paravirt_alloc_pte(&init_mm, __pa(page_table) >> PAGE_SHIFT);  set_pmd(pmd, __pmd(__pa(page_table) | _PAGE_TABLE));  BUG_ON(page_table != pte_offset_kernel(pmd, 0)); } return pte_offset_kernel(pmd, 0);}
```

page_table_kmap_check()的实现如下：

```
#define PTRS_PER_PTE 512static pte_t *__init page_table_kmap_check(pte_t *pte, pmd_t *pmd,        unsigned long vaddr, pte_t *lastpte,        void **adr){#ifdef CONFIG_HIGHMEM /*  * Something (early fixmap) may already have put a pte  * page here, which causes the page table allocation  * to become nonlinear. Attempt to fix it, and if it  * is still nonlinear then we have to bug.  */    //得到内核固定映射区的临时映射区的起始和结束虚拟页框号 int pmd_idx_kmap_begin = fix_to_virt(FIX_KMAP_END) >> PMD_SHIFT; int pmd_idx_kmap_end = fix_to_virt(FIX_KMAP_BEGIN) >> PMD_SHIFT; if (pmd_idx_kmap_begin != pmd_idx_kmap_end     && (vaddr >> PMD_SHIFT) >= pmd_idx_kmap_begin     && (vaddr >> PMD_SHIFT) <= pmd_idx_kmap_end) {  pte_t *newpte;  int i;  BUG_ON(after_bootmem);  newpte = *adr;        //拷贝操作  for (i = 0; i < PTRS_PER_PTE; i++)   set_pte(newpte + i, pte[i]);  *adr = (void *)(((unsigned long)(*adr)) + PAGE_SIZE);  paravirt_alloc_pte(&init_mm, __pa(newpte) >> PAGE_SHIFT);  set_pmd(pmd, __pmd(__pa(newpte)|_PAGE_TABLE)); //pmd与newpte表进行关联  BUG_ON(newpte != pte_offset_kernel(pmd, 0));  __flush_tlb_all();  paravirt_release_pte(__pa(pte) >> PAGE_SHIFT);  pte = newpte; } BUG_ON(vaddr < fix_to_virt(FIX_KMAP_BEGIN - 1)        && vaddr > fix_to_virt(FIX_KMAP_END)        && lastpte && lastpte + PTRS_PER_PTE != pte);#endif return pte;}
```

检查当前页表初始化的地址是否处于该区间范围，如果是，则把其 pte 页表的内容拷贝到 page_table_range_init()的 alloc_low_pages 申请的页表空间中（这里的拷贝主要是为了保证连续性），并将 newpte 新页表的地址设置到 pmd 中（32bit 系统实际上就是页全局目录），然后调用*_flush*tlb_all()刷新 TLB 缓存；如果不是该区间，则仅是由入参中调用的 one_page_table_init()被分配到了页表空间。

至此高端内存的固定映射区的页表分配完成，后面的 paging_init()会负责完成剩下的页表建立工作。

## 3.Linux 内存管理框架

传统的多核运算是使用 SMP(Symmetric Multi-Processor )模式：将多个处理器与一个集中的存储器和 I/O 总线相连，所有处理器访问同一个物理存储器，因此 SMP 系统有时也被称为一致存储器访问（**UMA**）结构体系，即无论在什么时候，处理器只能为内存的每个数据保持或共享唯一一个数值。

而**NUMA**模式是一种分布式存储器访问方式，处理器可以同时访问不同的存储器地址，大幅度提高并行性。NUMA 模式下系统的每个 CPU 都有本地内存，可支持快速访问，各个处理器之间通过总线连接起来，以支持对其它 CPU 本地内存的访问，但是这些访问要比处理器本地内存的慢.

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171715275451885.png)

Linux 内核通过插入一些兼容层，使两个不同体系结构的差异被隐藏，两种模式都使用了同一个数据结构。

在 NUMA 模式下，处理器和内存块被划分成多个”节点”(node)，比如机器上有 2 个处理器、4 个内存块，我们可以把 1 个处理器和 2 个内存块合起来（GNU/Linux 根据物理 CPU 的数量分配 node，一个物理 CPU 对应一个 node），共同组成一个 NUMA 的节点。每个节点被分配有本地存储器空间，所有节点中的处理器都可以访问系统全部的物理存储器，但是访问本节点内的存储器所需要的时间比访问某些远程节点内的存储器所花的时间要少得多。

与 CPU 类似，内存被分割成多个区域 BANK，也叫”簇”，依据簇与 CPU 的”距离”的不同，访问不同簇的方式也会有所不同，CPU 被划分为多个节点，每个 CPU 对应一个本地物理内存, 一般一个 CPU-node 对应一个内存簇，也就是说每个内存簇被认为是一个节点。

而在 UMA 系统中, 内存就相当于一个只使用一个 NUMA 节点来管理整个系统的内存，这样在内存管理的其它地方可以认为他们就是在处理一个(伪)NUMA 系统。

### **3.1 内存管理框架初始化**

在 linux 中，每个物理内存节点 node 都被划分为多个内存管理区域用于表示不同范围的内存，比如上面提到的 NORMAL 内存、高端内存，内核可以使用不同的映射方式映射物理内存。zone 只是内核为了管理方便而做的一种逻辑上的划分，并不存在这种物理硬件单元。

综上，**linux 的物理内存管理机制将物理内存划分为三个层次来管理**，依次是：Node（存储节点）、Zone（管理区）和 Page（页面），它们之间的关系如下：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171715407686398.png)

其中，zone 的类型如下：

```
include/linux/mmzone.henum zone_type {#ifdef CONFIG_ZONE_DMA ZONE_DMA, //通常为内存首部16MB,某些工业标准体系结构设备需要用到ZONE_DMA(较旧的外设，只能寻址24位内存)#endif#ifdef CONFIG_ZONE_DMA32 ZONE_DMA32, //在64位Linux操作系统上,DMA32为16M~4G（可访问32位），高于4G的内存为Normal ZONE#endif ZONE_NORMAL, //通常为16MB~896MB，该部分的内存由内核直接映射到物理地址空间的较高部分#ifdef CONFIG_HIGHMEM ZONE_HIGHMEM, //通常为896MB~末尾，将保留给系统使用，是系统中预留的可用内存空间，动态映射#endif ZONE_MOVABLE, //用于减少内存的碎片化，这个区域的页都是可迁移的#ifdef CONFIG_ZONE_DEVICE ZONE_DEVICE, //为支持热插拔设备而分配的非易失性内存#endif __MAX_NR_ZONES};
```

回到之前的 setup_arch()函数，接着往下走，来到内存管理框架初始化的地方

```
void __init setup_arch(char **cmdline_p){    ...... max_pfn = e820__end_of_ram_pfn(); //max_pfn初始化 ...... find_low_pfn_range(); //max_low_pfn、高端内存初始化 ...... ...... early_alloc_pgt_buf(); //页表缓冲区分配 reserve_brk(); //缓冲区加入memblock.reserve    ......    e820__memblock_setup(); //memblock.memory空间初始化 启动 ......    init_mem_mapping(); //低端内存内核页表初始化 高端内存固定映射区中临时映射区页表初始化  ......    initmem_init(); // <----------------------------------------    ......}
```

**initmem_init**的实现如下：

```
#define PHYS_ADDR_MAX (~(phys_addr_t)0)#ifndef CONFIG_NEED_MULTIPLE_NODESvoid __init initmem_init(void){#ifdef CONFIG_HIGHMEM highstart_pfn = highend_pfn = max_pfn; if (max_pfn > max_low_pfn)  highstart_pfn = max_low_pfn; printk(KERN_NOTICE "%ldMB HIGHMEM available.\n",  pages_to_mb(highend_pfn - highstart_pfn)); high_memory = (void *) __va(highstart_pfn * PAGE_SIZE - 1) + 1;#else high_memory = (void *) __va(max_low_pfn * PAGE_SIZE - 1) + 1;#endif memblock_set_node(0, PHYS_ADDR_MAX, &memblock.memory, 0);#ifdef CONFIG_FLATMEM max_mapnr = IS_ENABLED(CONFIG_HIGHMEM) ? highend_pfn : max_low_pfn;#endif __vmalloc_start_set = true; printk(KERN_NOTICE "%ldMB LOWMEM available.\n",   pages_to_mb(max_low_pfn)); setup_bootmem_allocator();}#endif /* !CONFIG_NEED_MULTIPLE_NODES */
```

这个函数将**high_memory**初始化为低端内存最大页框**max_low_pfn**对应的地址大小，接着调用 memblock_set_node，通过 memblock 内存管理器**设置 node 节点信息**。

```
int __init_memblock memblock_set_node(phys_addr_t base, phys_addr_t size,          struct memblock_type *type, int nid){#ifdef CONFIG_NEED_MULTIPLE_NODES int start_rgn, end_rgn; int i, ret; ret = memblock_isolate_range(type, base, size, &start_rgn, &end_rgn); if (ret)  return ret; for (i = start_rgn; i < end_rgn; i++)  memblock_set_region_node(&type->regions[i], nid); memblock_merge_regions(type);#endif return 0;}
```

memblock_set_node 主要调用了三个函数：memblock_isolate_range、memblock_set_region_node 和 memblock_merge_regions，首先看**memblock_isolate_range**()函数：

```
/* adjust *@size so that (@base + *@size) doesn't overflow, return new size */static inline phys_addr_t memblock_cap_size(phys_addr_t base, phys_addr_t *size){ return *size = min(*size, PHYS_ADDR_MAX - base);}static int __init_memblock memblock_isolate_range(struct memblock_type *type,     phys_addr_t base, phys_addr_t size,     int *start_rgn, int *end_rgn){ phys_addr_t end = base + memblock_cap_size(base, &size); int idx; struct memblock_region *rgn; *start_rgn = *end_rgn = 0; if (!size)  return 0; /* we'll create at most two more regions */ while (type->cnt + 2 > type->max)  if (memblock_double_array(type, base, size) < 0)   return -ENOMEM; for_each_memblock_type(idx, type, rgn) {  phys_addr_t rbase = rgn->base;  phys_addr_t rend = rbase + rgn->size;  if (rbase >= end)   break;  if (rend <= base)   continue;  if (rbase < base) {   /*    * @rgn intersects from below.  Split and continue    * to process the next region - the new top half.    */   rgn->base = base;   rgn->size -= base - rbase;   type->total_size -= base - rbase;   memblock_insert_region(type, idx, rbase, base - rbase,            memblock_get_region_node(rgn),            rgn->flags);  } else if (rend > end) {   /*    * @rgn intersects from above.  Split and redo the    * current region - the new bottom half.    */   rgn->base = end;   rgn->size -= end - rbase;   type->total_size -= end - rbase;   memblock_insert_region(type, idx--, rbase, end - rbase,            memblock_get_region_node(rgn),            rgn->flags);  } else {   /* @rgn is fully contained, record it */   if (!*end_rgn)    *start_rgn = idx;   *end_rgn = idx + 1;  } } return 0;}
```

在*_memblock*remove()中有提到，**memblock_isolate_range()主要作用是将要移除的物理内存区从 reserved 内存区中分离出来**，将 start_rgn 和 end_rgn（该内存区块的起始、结束索引号）返回回去，而这里，由于我们传入的**type 是 memblock.memory**，该函数会根据入参 base 和 size 标记节点内存范围，将**该内存从 memory 中划分开来，同时返回对应的 start_rgn 和 end_rgn。**

1）如果 memblock 中的 region 恰好以在该节点内存范围内的话，那么再未赋值 end_rgn 时将当前 region 的索引记录至 start_rgn，end_rgn 在此基础上加 1；

2）如果 memblock 中的 region 跨越了该节点内存末尾分界，那么将会把当前的 region 边界调整为 node 节点内存范围边界，然后通过 memblock_insert_region()函数将剩下的部分（即越出内存范围的那一块内存）重新插入 memblock 管理 regions 当中，实现拆分；

```
static inline void memblock_set_region_node(struct memblock_region *r, int nid){ r->nid = nid;}static inline int memblock_get_region_node(const struct memblock_region *r){ return r->nid;}static void __init_memblock memblock_insert_region(struct memblock_type *type,         int idx, phys_addr_t base,         phys_addr_t size,         int nid, unsigned long flags){ struct memblock_region *rgn = &type->regions[idx]; BUG_ON(type->cnt >= type->max); memmove(rgn + 1, rgn, (type->cnt - idx) * sizeof(*rgn)); rgn->base = base; rgn->size = size; rgn->flags = flags; memblock_set_region_node(rgn, nid); type->cnt++; type->total_size += size;}
```

上面的 memmove()将后面的 region 信息往后移，另外调用 memblock_set_region_node()将原 region 的 node 节点号保留在被拆分出来的 region 当中。

回到前面的 memblock_set_node()函数，紧接着 memblock_isolate_range()被调用的是**memblock_set_region_node**()，通过这个函数**把划分出来的 region 进行 node 节点号设置**，而后面的**memblock_merge_regions**()前面 MEMBLOCK 内存分配器初始化时已经分析过了，**是用于将相邻的 region 进行合并的**（节点号、flag 等一致才会合并）。

最后回到 initmem_init()函数中，memblock_set_node()返回后，接着调用的函数为 setup_bootmem_allocator()。

```
void __init setup_bootmem_allocator(void){ printk(KERN_INFO "  mapped low ram: 0 - %08lx\n",   max_pfn_mapped<<PAGE_SHIFT); printk(KERN_INFO "  low ram: 0 - %08lx\n", max_low_pfn<<PAGE_SHIFT);}
```

原来该函数是用来初始化 bootmem 管理算法的，但现在 x86 的环境已经使用了 memblock 管理算法，所以这里仅作保留，打印部分信息。

**bootmem 分配器使用一个 bitmap 来标记物理页是否被占用**，分配的时候按照第一适应的原则，从 bitmap 中进行查找，如果这位为 1，表示已经被占用，否则表示未被占用。bootmem 分配器每次分配内存都会在 bitmap 中进行线性搜索，效率非常低，而且容易在内存中留下许多小的空闲碎片，在需要非常大的内存块的时候，检查位图这一过程就显得代价很高。bootmem 分配器是用于在启动阶段分配内存的，对该分配器的需求集中于简单性方面，而不是性能和通用性（和 memblock 管理器一致）。

至此，已完成对内存的节点 node 设置。

回到 setup_arch()函数：

```
void __init setup_arch(char **cmdline_p){    ...... max_pfn = e820__end_of_raCm_pfn(); //max_pfn初始化 ...... find_low_pfn_range(); //max_low_pfn、高端内存初始化 ...... ...... early_alloc_pgt_buf(); //页表缓冲区分配 reserve_brk(); //缓冲区加入memblock.reserve    ......    e820__memblock_setup(); //memblock.memory空间初始化 启动 ......    init_mem_mapping(); //低端内存内核页表初始化 高端内存固定映射区中临时映射区页表初始化  ......    initmem_init(); //high_memory（高端内存起始pfn）初始化 通过memblock内存管理器设置node节点信息    ......    x86_init.paging.pagetable_init(); // <-----------------------------    ......}
```

x86_init 结构体内**pagetable_init**实际上挂接的是**native_pagetable_init**()函数：

```
struct x86_init_ops x86_init __initdata = { ...... .paging = {  .pagetable_init  = native_pagetable_init, }, ......}
```

native_pagetable_init()函数内容如下：

```
void __init native_pagetable_init(void){ unsigned long pfn, va; pgd_t *pgd, *base = swapper_pg_dir; p4d_t *p4d; pud_t *pud; pmd_t *pmd; pte_t *pte;    //循环 低端内存最大物理页号~最大物理页号 for (pfn = max_low_pfn; pfn < 1<<(32-PAGE_SHIFT); pfn++) {  va = PAGE_OFFSET + (pfn<<PAGE_SHIFT);  pgd = base + pgd_index(va);  if (!pgd_present(*pgd))   break;  p4d = p4d_offset(pgd, va);  pud = pud_offset(p4d, va);  pmd = pmd_offset(pud, va);  if (!pmd_present(*pmd))   break;  /* should not be large page here */  if (pmd_large(*pmd)) {   pr_warn("try to clear pte for ram above max_low_pfn: pfn: %lx pmd: %p pmd phys: %lx, but pmd is big page and is not using pte !\n",    pfn, pmd, __pa(pmd));   BUG_ON(1);  }  pte = pte_offset_kernel(pmd, va);  if (!pte_present(*pte))   break;  printk(KERN_DEBUG "clearing pte for ram above max_low_pfn: pfn: %lx pmd: %p pmd phys: %lx pte: %p pte phys: %lx\n",    pfn, pmd, __pa(pmd), pte, __pa(pte));  pte_clear(NULL, va, pte); } paravirt_alloc_pmd(&init_mm, __pa(base) >> PAGE_SHIFT); paging_init();}
```

**PAGE_OFFSET 代表的是内核空间和用户空间对虚拟地址空间的划分，对不同的体系结构不同。比如在 32 位系统中 3G-4G 属于内核使用的内存空间，所以 PAGE_OFFSET = 0xC0000000**。

该函数的 for 循环主要是用于检测 max_low_pfn 后面的内核空间内存直接映射空间后面的物理内存**是否存在系统启动引导时创建的页表**（pfn 通过直接映射的方法得到虚拟地址，然后通过内核页表得到 pgd、pmd、pte），如果存在，则使用 pte_clear()将其清除。

接着看 native_pagetable_init()调用的最后一个函数：**paging_init**()。

```
void __init paging_init(void){ pagetable_init(); __flush_tlb_all(); kmap_init(); /*  * NOTE: at this point the bootmem allocator is fully available.  */ olpc_dt_build_devicetree(); sparse_init(); zone_sizes_init();}
```

可以对着这个图看：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171716098339686.png)

前面已经分析过低端内存、固定映射区中临时映射区的内核页表的建立，这里 paging_init 将会完成剩下的工作，首先看**pagetable_init**()

```
static void __init pagetable_init(void){ pgd_t *pgd_base = swapper_pg_dir; permanent_kmaps_init(pgd_base);}static void __init permanent_kmaps_init(pgd_t *pgd_base){ unsigned long vaddr = PKMAP_BASE; page_table_range_init(vaddr, vaddr + PAGE_SIZE*LAST_PKMAP, pgd_base); pkmap_page_table = virt_to_kpte(vaddr);}
```

**该函数为建立持久映射区（KMAP 区）的页表**，page_table_range_init 函数前面固定映射区页表初始化时已经分析过了（初始化 pgd_base 指向的页全局目录中 start 到 end 这个范围的线性地址，整个函数结束后只是初始化好了页中间目录项对应的页表，但是页表中的页表项并没有初始化），这里建立页表范围为 PKMAP_BASE 到 PKMAP_BASE + PAGE_SIZE*LAST_PKMAP，**建好页表后将页表地址赋值给给持久映射区页表变量 pkmap_page_table**。

*_flush*tlb_all()为刷新全部 TLB，这里不做介绍，接着看 paging_init()调用的下一个函数 kmap_init()。

```
static void __init kmap_init(void){ unsigned long kmap_vstart; /*  * Cache the first kmap pte:  */ kmap_vstart = __fix_to_virt(FIX_KMAP_BEGIN); kmap_pte = virt_to_kpte(kmap_vstart);}
```

可以很容易看到 kmap_init()主要是**获取到临时映射区间的起始页表并往临时映射页表变量 kmap_pte 置值**。

回到 paging_init()，olpc_dt_build_devicetree 这里就不做介绍了，而**sparse_init**()则涉及到了 Linux 的内存模型，这里介绍一下 Linux 的三种内存模型，注意，以下都是从 CPU 的角度来看的。

从系统中任意一个 CPU 的角度来看，当它访问物理内存的时候，物理地址空间是一个**连续的、没有空洞**的地址空间，那么这种计算机系统的内存模型就是**Flat memory**。在这种内存模型下，物理内存的管理比较简单，每一个物理页帧都会有一个 page 数据结构来抽象，因此系统中存在一个 struct page 的数组（位于直接映射区，mem_map，在节点 node 里，后面就会介绍到），每一个数组条目指向一个实际的物理页帧（page frame）。在 flat memory 的情况下，PFN 和 mem_map 数组 index 的关系是线性的（即位于直接映射区，有一个固定偏移），因此从 PFN 到对应的 page 数据结构是非常容易的，反之亦然。

pfn_to_page/page_to_pfn 的作用是 struct page* 和 pfn 页帧号之间的转换，flat memory 内存模型的相关代码如下：

```
#if defined(CONFIG_FLATMEM)#define __pfn_to_page(pfn) (mem_map + ((pfn) - ARCH_PFN_OFFSET))#define __page_to_pfn(page) ((unsigned long)((page) - mem_map) + \     ARCH_PFN_OFFSET)
```

**PFN 和 struct page 数组（mem_map）的 index 是线性关系，有一个固定的偏移就是 ARCH_PFN_OFFSET（跟架构相关的物理起始地址的 PFN）**。

如果 cpu 在访问物理内存的时候，**其地址空间有一些空洞，是不连续的**，那么这种计算机系统的内存模型就是**Discontiguous memory**。一般而言，NUMA 架构的计算机系统的 memory model 都是选择 Discontiguous Memory，不过 NUMA 强调的是 memory 和 processor 的位置关系，和内存模型其实是没有关系的（NUMA 并没有规定其内存的连续性，而 Discontiguous memory 系统也并非一定是 NUMA 系统，但是这两种都是多节点的），只不过，由于同一 node 上的 memory 和 processor 有更紧密的耦合关系（访问更快），因此需要多个 node 来管理。**Discontiguous memory 本质上是 flat memory 内存模型的扩展，整个物理内存的内存空间大部分是成片的大块内存，中间会有一些空洞，每一个成片的内存地址空间属于一个 node（如果局限在一个 node 内部，其内存模型是 flat memory）**。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171716257917012.png)

在这种模型下，从 PFN 转换到具体的 struct page 会稍微复杂一点，首先要从 PFN 得到 node ID，然后根据这个 ID 找到对于的节点 node 数据结构，也就找到了对应的 page 数组，之后的方法就类似 flat memory 了。

```
#elif defined(CONFIG_DISCONTIGMEM)#define __pfn_to_page(pfn)   \({ unsigned long __pfn = (pfn);  \ unsigned long __nid = arch_pfn_to_nid(__pfn);  \ NODE_DATA(__nid)->node_mem_map + arch_local_page_offset(__pfn, __nid);\})#define __page_to_pfn(pg)      \({ const struct page *__pg = (pg);     \ struct pglist_data *__pgdat = NODE_DATA(page_to_nid(__pg)); \ (unsigned long)(__pg - __pgdat->node_mem_map) +   \  __pgdat->node_start_pfn;     \})
```

Discontiguous memory 模型需要获取 node id，只要找到 node id，一切都好办了，后面类比 flat memory model 进行就 OK 了。对于**pfn_to_page 的定义，首先通过 arch_pfn_to_nid 将 PFN 转换成 node id，通过 NODE_DATA 宏定义可以找到该 node 对应的 pglist_data 数据结构，该数据结构的 node_start_pfn 记录了该 node 的第一个 pfn，因此，也就可以得到其对应 struct page 在 node_mem_map 的偏移，**page_to_pfn 类似与上述基本类似(pglist_data 数据结构后面再进行介绍)。

内存模型也是一个演进过程，刚开始的时候，使用 flat memory 去抽象一个连续的内存地址空间，出现 NUMA 之后，整个不连续的内存空间被分成若干个 node，每个 node 上是连续的内存地址空间，也就是从原来单一的一个 mem_maps[]变成了若干个 mem_maps[]。而现在 memory hotplug 的出现让原来完美的设计变得不完美了（热插拔，即带电插拔，热插拔功能就是允许用户在不关闭系统，不切断电源的情况下取出和更换损坏的硬盘、电源或板卡等部件。Linux 内核支持热插拔的部件有 USB 设备、PCI 设备甚至 CPU），因为即便是一个 node 中的 mem_maps[]也有可能是不连续了，因此目前 Discontiguous memory 也逐渐在被 sparse memory 替代。

在**sparse memory**内存模型下，连续的地址空间按照 SECTION（例如 1G）被分成了一段一段的，其中每一 section 都是 hotplug 的，**因此 sparse memory 下，内存地址空间可以被切分的更细**。整个连续的物理地址空间是按照一个个**section**来切断的，在每一个 section 内部，其 memory 是连续的（即符合 flat memory 的特点），因此，**mem_map 的 page 数组依附于 section 结构（struct mem_section），而不是 node 结构（struct pglist_data）。在这个模型下，PFN 转 struct page 变为了转换变成了 PFN<—>Section<—>page。**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171716353283542.png)

linux 内核中静态定义了一个 mem_section 的指针数组，一个 section 中往往包括多个 page，因此需要通过 PFN 得到 section 号，用 section 号做为 index 在 mem_section 指针数组可以找到该 PFN 对应的 section 数据结构。实际上**PFN 分成两个部分：一部分是 section index，另外一个部分是 page 在该 section 的偏移**，找到 section 之后，沿着其 mem_map 就可以找到对应的 page 数据结构。

**对于 page 到 section index 的转换，sparse memory 有 2 种方案**，先看看经典的方案，也就是把 section 号保存在 page->flags 中（page 的结构同样在后面再进行介绍），这种方法的最大的问题是 page->flags 中的 bit 数不一定够用，因为这个 flag 中承载了太多的信息，各种 page flag、node id、zone id，现在又增加一个 section id，在不同的处理器架构中无法实现一致性的算法（上面的图即为采用经典算法的 sparse memory）。

```
#elif defined(CONFIG_SPARSEMEM)/* * Note: section's mem_map is encoded to reflect its start_pfn. * section[i].section_mem_map == mem_map's address - start_pfn; */#define __page_to_pfn(pg)     \({ const struct page *__pg = (pg);    \ int __sec = page_to_section(__pg);   \ (unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec))); \})#define __pfn_to_page(pfn)    \({ unsigned long __pfn = (pfn);   \ struct mem_section *__sec = __pfn_to_section(__pfn); \ __section_mem_map_addr(__sec) + __pfn;  \})#endif /* CONFIG_FLATMEM/DISCONTIGMEM/SPARSEMEM */
```

对于经典的 sparse memory 模型，一个 section 的 struct page 数组所占用的内存来自直接映射区，页表在初始化的时候就建立好了。但是，对于 SPARSEMEM_VMEMMAP 而言，虚拟地址一开始就分配好了，是 vmemmap 开始的一段连续的虚拟地址空间，但是没有物理地址。因此，当一个 section 被发现后，可以立刻找到对应的 struct page 的虚拟地址，而这时候还需要分配一个物理的 page frame，对于这种 sparse memory，开销会稍微大一些，需要建立页表，关联 page 跟物理地址。

```
#elif defined(CONFIG_SPARSEMEM_VMEMMAP)/* memmap is virtually contiguous.  */#define __pfn_to_page(pfn) (vmemmap + (pfn))#define __page_to_pfn(page) (unsigned long)((page) - vmemmap)
```

### 3.2 **内存管理区 Zone 初始化**

回到 paging_init()的最后一个函数调用：**zone_sizes_init**()。

```
void __init zone_sizes_init(void){ unsigned long max_zone_pfns[MAX_NR_ZONES]; memset(max_zone_pfns, 0, sizeof(max_zone_pfns));#ifdef CONFIG_ZONE_DMA max_zone_pfns[ZONE_DMA]  = min(MAX_DMA_PFN, max_low_pfn);#endif#ifdef CONFIG_ZONE_DMA32 max_zone_pfns[ZONE_DMA32] = min(MAX_DMA32_PFN, max_low_pfn);#endif max_zone_pfns[ZONE_NORMAL] = max_low_pfn;#ifdef CONFIG_HIGHMEM max_zone_pfns[ZONE_HIGHMEM] = max_pfn;#endif free_area_init(max_zone_pfns);}
```

这个函数**为 zone 的各个内存管理区域最大物理页号进行初始化**，并作为参数调用 free_area_init_nodes()，其中**free_area_init_nodes**()函数实现如下：

```
void __init free_area_init(unsigned long *max_zone_pfn){ unsigned long start_pfn, end_pfn; int i, nid, zone; bool descending; /* Record where the zone boundaries are */    //全局数组arch_zone_lowest_possible_pfn用来存储各个内存域可使用的最低内存页帧编号    //全局数组arch_zone_highest_possible_pfn用来存储各个内存域可使用的最高内存页帧编号 memset(arch_zone_lowest_possible_pfn, 0,    sizeof(arch_zone_lowest_possible_pfn)); memset(arch_zone_highest_possible_pfn, 0,    sizeof(arch_zone_highest_possible_pfn));    //用于最低内存区域中可用的编号最小的页帧，即memblock.memory.regions[0].base start_pfn = find_min_pfn_with_active_regions();    //return false descending = arch_has_descending_max_zone_pfns();    //根据max_zone_pfn和start_pfn初始化arch_zone_lowest_possible_pfn和arch_zone_highest_possible_pfn for (i = 0; i < MAX_NR_ZONES; i++) {  if (descending)   zone = MAX_NR_ZONES - i - 1;  else   zone = i;  //由于ZONE_MOVABLE是一个虚拟内存域，不与真正的硬件内存域关联，该内存域的边界总是设置为0  if (zone == ZONE_MOVABLE)   continue;  end_pfn = max(max_zone_pfn[zone], start_pfn);  arch_zone_lowest_possible_pfn[zone] = start_pfn;  arch_zone_highest_possible_pfn[zone] = end_pfn;  start_pfn = end_pfn; } /* Find the PFNs that ZONE_MOVABLE begins at in each node */ memset(zone_movable_pfn, 0, sizeof(zone_movable_pfn));    //用于计算ZONE_MOVABLE的内存数量    //在内存较多的32位系统上, 这通常会是ZONE_HIGHMEM, 但是对于64位系统，将使用ZONE_NORMAL或ZONE_DMA32，这里计算也比较复杂，感兴趣的话可以去看一下源码，这里便不做介绍了 find_zone_movable_pfns_for_nodes(); ......    //各种打印    ...... /* Initialise every node */    //为系统中的每个节点调用free_area_init_node() mminit_verify_pageflags_layout(); setup_nr_node_ids(); init_unavailable_mem(); for_each_online_node(nid) {  pg_data_t *pgdat = NODE_DATA(nid);  free_area_init_node(nid);  /* Any memory on that node */  if (pgdat->node_present_pages)   node_set_state(nid, N_MEMORY);  check_for_memory(pgdat, nid); }}
```

**free_area_init_node()函数会计算每个节点中每个区域的大小及其空洞的大小**，如果两个相邻区域之间的最大 PFN 匹配，则可以假定后面的区域为空。例如 arch_max_dma_pfn == arch_max_dma32_pfn，则假定 arch_max_dma32_pfn 该区域为空。

```
pg_data_t node_data[MAX_NUMNODES];EXPORT_SYMBOL(node_data);#ifdef CONFIG_NUMAextern struct pglist_data *node_data[];#define NODE_DATA(nid) (node_data[nid])#endif /* CONFIG_NUMA */static void __init free_area_init_node(int nid){    //从全局节点数组中获取一个节点 pg_data_t *pgdat = NODE_DATA(nid); unsigned long start_pfn = 0; unsigned long end_pfn = 0; /* pg_data_t should be reset to zero when it's allocated */ WARN_ON(pgdat->nr_zones || pgdat->kswapd_highest_zoneidx); //根据节点id获取起始pfn和结束pfn，前面node初始化时，memblock处已经设置好节点ID了 get_pfn_range_for_nid(nid, &start_pfn, &end_pfn); //设置节点ID以及起始pfn pgdat->node_id = nid; pgdat->node_start_pfn = start_pfn; pgdat->per_cpu_nodestats = NULL; pr_info("Initmem setup node %d [mem %#018Lx-%#018Lx]\n", nid,  (u64)start_pfn << PAGE_SHIFT,  end_pfn ? ((u64)end_pfn << PAGE_SHIFT) - 1 : 0); //初始化节点中每个zone的大小和空洞，同时计算节点的spanned_pages和present_pages    calculate_node_totalpages(pgdat, start_pfn, end_pfn); alloc_node_mem_map(pgdat); pgdat_set_deferred_range(pgdat); free_area_init_core(pgdat);}
```

在进入 calculate_node_totalpages 之前，这里还是先简单介绍一下**node**的数据结构。

```
struct zoneref { struct zone *zone; /* Pointer to actual zone */ int zone_idx;  /* zone_idx(zoneref->zone) */};struct zonelist { struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];};typedef struct pglist_data { //本节点的所有zone内存管理区 struct zone node_zones[MAX_NR_ZONES];    //包含了对所有node的zone的引用，通常第一个node_zonelists为本节点自己的zones struct zonelist node_zonelists[MAX_ZONELISTS]; //本节点zone管理区数目 int nr_zones; /* number of populated zones in this node */#ifdef CONFIG_FLAT_NODE_MEM_MAP /* means !SPARSEMEM */    //Discontiguous Memory内存模型，可以找到节点下的所有page struct page *node_mem_map;#ifdef CONFIG_PAGE_EXTENSION struct page_ext *node_page_ext;#endif#endif    ......    //节点第一个页的页号 unsigned long node_start_pfn;    //节点总共的物理页数，不含空洞 unsigned long node_present_pages; /* total number of physical pages */    //节点物理页的范围大小，含空洞 unsigned long node_spanned_pages; /* total size of physical page          range, including holes */ int node_id;    ......    //内存回收相关的数据结构 ...... ZONE_PADDING(_pad1_)    ...... unsigned long  flags; ZONE_PADDING(_pad2_)    ......} pg_data_t;
```

ZONE_PADDING 的作用，通过添加大量的填充把经常被访问的“热门”数据分到了单独的 cache line 上，可以理解为空间换时间。

**calculate_node_totalpages**()实现：

```
static void __init calculate_node_totalpages(struct pglist_data *pgdat,      unsigned long node_start_pfn,      unsigned long node_end_pfn){ unsigned long realtotalpages = 0, totalpages = 0; enum zone_type i; for (i = 0; i < MAX_NR_ZONES; i++) {  struct zone *zone = pgdat->node_zones + i;  unsigned long zone_start_pfn, zone_end_pfn;  unsigned long spanned, absent;  unsigned long size, real_size;  spanned = zone_spanned_pages_in_node(pgdat->node_id, i,           node_start_pfn,           node_end_pfn,           &zone_start_pfn,           &zone_end_pfn);  absent = zone_absent_pages_in_node(pgdat->node_id, i,         node_start_pfn,         node_end_pfn);  size = spanned;  real_size = size - absent;  if (size)   zone->zone_start_pfn = zone_start_pfn;  else   zone->zone_start_pfn = 0;  zone->spanned_pages = size;  zone->present_pages = real_size;  totalpages += size;  realtotalpages += real_size; } pgdat->node_spanned_pages = totalpages; pgdat->node_present_pages = realtotalpages; printk(KERN_DEBUG "On node %d totalpages: %lu\n", pgdat->node_id,       realtotalpages);}
```

其中**zone_spanned_pages_in_node**()：

```
static unsigned long __init zone_spanned_pages_in_node(int nid,     unsigned long zone_type,     unsigned long node_start_pfn,     unsigned long node_end_pfn,     unsigned long *zone_start_pfn,     unsigned long *zone_end_pfn){ unsigned long zone_low = arch_zone_lowest_possible_pfn[zone_type]; unsigned long zone_high = arch_zone_highest_possible_pfn[zone_type]; /* When hotadd a new node from cpu_up(), the node should be empty */ if (!node_start_pfn && !node_end_pfn)  return 0; /* Get the start and end of the zone */    //限制zone_start_pfn和zone_end_pfn所在区间 *zone_start_pfn = clamp(node_start_pfn, zone_low, zone_high); *zone_end_pfn = clamp(node_end_pfn, zone_low, zone_high); adjust_zone_range_for_zone_movable(nid, zone_type,    node_start_pfn, node_end_pfn,    zone_start_pfn, zone_end_pfn); /* Check that this node has pages within the zone's required range */ if (*zone_end_pfn < node_start_pfn || *zone_start_pfn > node_end_pfn)  return 0; /* Move the zone boundaries inside the node if necessary */ *zone_end_pfn = min(*zone_end_pfn, node_end_pfn); *zone_start_pfn = max(*zone_start_pfn, node_start_pfn); /* Return the spanned pages */ return *zone_end_pfn - *zone_start_pfn;}
```

该函数**主要是统计 node 管理节点的内存跨度**，该跨度不包括 movable 管理区的（因为 movable 就是在其它内存管理区里分配出来的），里面调用的 adjust_zone_range_for_zone_movable()则是用于剔除 movable 管理区的部分。

另外的**zone_absent_pages_in_node**()函数：

```
unsigned long __init __absent_pages_in_range(int nid,    unsigned long range_start_pfn,    unsigned long range_end_pfn){ unsigned long nr_absent = range_end_pfn - range_start_pfn; unsigned long start_pfn, end_pfn; int i; for_each_mem_pfn_range(i, nid, &start_pfn, &end_pfn, NULL) {  start_pfn = clamp(start_pfn, range_start_pfn, range_end_pfn);  end_pfn = clamp(end_pfn, range_start_pfn, range_end_pfn);  nr_absent -= end_pfn - start_pfn; } return nr_absent;}static unsigned long __init zone_absent_pages_in_node(int nid,     unsigned long zone_type,     unsigned long node_start_pfn,     unsigned long node_end_pfn){ unsigned long zone_low = arch_zone_lowest_possible_pfn[zone_type]; unsigned long zone_high = arch_zone_highest_possible_pfn[zone_type]; unsigned long zone_start_pfn, zone_end_pfn; unsigned long nr_absent; /* When hotadd a new node from cpu_up(), the node should be empty */ if (!node_start_pfn && !node_end_pfn)  return 0; zone_start_pfn = clamp(node_start_pfn, zone_low, zone_high); zone_end_pfn = clamp(node_end_pfn, zone_low, zone_high); adjust_zone_range_for_zone_movable(nid, zone_type,   node_start_pfn, node_end_pfn,   &zone_start_pfn, &zone_end_pfn); nr_absent = __absent_pages_in_range(nid, zone_start_pfn, zone_end_pfn); /*  * ZONE_MOVABLE handling.  * Treat pages to be ZONE_MOVABLE in ZONE_NORMAL as absent pages  * and vice versa.  */ if (mirrored_kernelcore && zone_movable_pfn[nid]) {  unsigned long start_pfn, end_pfn;  struct memblock_region *r;  for_each_mem_region(r) {   start_pfn = clamp(memblock_region_memory_base_pfn(r),       zone_start_pfn, zone_end_pfn);   end_pfn = clamp(memblock_region_memory_end_pfn(r),     zone_start_pfn, zone_end_pfn);   if (zone_type == ZONE_MOVABLE &&       memblock_is_mirror(r))    nr_absent += end_pfn - start_pfn;   if (zone_type == ZONE_NORMAL &&       !memblock_is_mirror(r))    nr_absent += end_pfn - start_pfn;  } } return nr_absent;}
```

**该函数主要用于计算内存空洞页面数的**，计算方法大致为在 zone 区域范围内，遍历所有 memblock 的内存块，将这些内存块的大小累加，之后两者做差，zone_absent_pages_in_node 后面是对 ZONE_MOVABLE 的特殊处理了，方法是类似的，这里也不做介绍了。

calculate_node_totalpages()后面就是各种简单的赋值操作了，这里也简单介绍一下**zone**的结构：

其中 MAX_NR_ZONES 是一个节点中所能包容纳的 Zones 的最大数。

```
struct zone { ......    //保留页框池，记录每个管理区中必须保留的物理页面数，以用于紧急状况下的内存分配 long lowmem_reserve[MAX_NR_ZONES];    //保持对UMA的兼容(当做一个节点)，NUMA模式下的节点数#ifdef CONFIG_NUMA int node;#endif //该zone的父节点    struct pglist_data *zone_pgdat; ...... //该zone的第一个页的页号 unsigned long  zone_start_pfn;    //伙伴系统管理的page数，这是除去了在初始化阶段被申请的页面（比如memblock） atomic_long_t  managed_pages;    //zone大小，含空洞，即zone_end_pfn - zone_start_pfn unsigned long  spanned_pages;    //zone实际大小，不含空洞 unsigned long  present_pages; //zone的名称，如“DMA”“Normal”“Highmem”，这些名称定义于page_alloc.c的zone_names[MAX_NR_ZONES] const char  *name; ZONE_PADDING(_pad1_) //包含所有空闲页面，伙伴系统使用，里面有数量为MIGRATE_TYPES个的free_list链表，分别用于管理不同迁移类型的内存页面 struct free_area free_area[MAX_ORDER];    //描述zone的当前状态 unsigned long  flags;    /* Primarily protects free_area */    //与伙伴算法的碎片迁移算法有关 spinlock_t  lock; ZONE_PADDING(_pad2_) ...... ZONE_PADDING(_pad3_) ......} ____cacheline_internodealigned_in_smp;
```

回到 free_area_init_node()函数，紧接在 calculate_node_totalpages()后的函数调用的为**alloc_node_mem_map()，这个函数是用于申请 node 节点的 node_mem_map 相应的内存空间**，如果是 sparse memory 内存模型，则该函数实现为空，这里便不做过多介绍了，直接看最后的初始化工作：**free_area_init_core**()。

```
static void __init free_area_init_core(struct pglist_data *pgdat){ enum zone_type j; int nid = pgdat->node_id; //对节点的一些锁和队列进行初始化 pgdat_init_internals(pgdat); pgdat->per_cpu_nodestats = &boot_nodestats; for (j = 0; j < MAX_NR_ZONES; j++) {  struct zone *zone = pgdat->node_zones + j;  unsigned long size, freesize, memmap_pages;  unsigned long zone_start_pfn = zone->zone_start_pfn;  size = zone->spanned_pages;  freesize = zone->present_pages;  //memmap_pages,每一个4k物理页都对应一个mem_map_t来管理  memmap_pages = calc_memmap_size(size, freesize);  if (!is_highmem_idx(j)) {   if (freesize >= memmap_pages) {    freesize -= memmap_pages;    if (memmap_pages)     printk(KERN_DEBUG            "  %s zone: %lu pages used for memmap\n",            zone_names[j], memmap_pages);   } else    pr_warn("  %s zone: %lu pages exceeds freesize %lu\n",     zone_names[j], memmap_pages, freesize);  }  //dma保留页  if (j == 0 && freesize > dma_reserve) {   freesize -= dma_reserve;   printk(KERN_DEBUG "  %s zone: %lu pages reserved\n",     zone_names[0], dma_reserve);  }  //计算nr_kernel_pages（低端内存的页数）和nr_all_pages的数量  if (!is_highmem_idx(j))   nr_kernel_pages += freesize;  /* Charge for highmem memmap if there are enough kernel pages */        //如果有足够的页，则也为高端内存提供memmap_pages  else if (nr_kernel_pages > memmap_pages * 2)   nr_kernel_pages -= memmap_pages;  nr_all_pages += freesize;  /*   * Set an approximate value for lowmem here, it will be adjusted   * when the bootmem allocator frees pages into the buddy system.   * And all highmem pages will be managed by the buddy system.   */        //初始化zone使用的各类锁  zone_init_internals(zone, j, nid, freesize);  if (!size)   continue;  set_pageblock_order();  setup_usemap(pgdat, zone, zone_start_pfn, size);  init_currently_empty_zone(zone, zone_start_pfn, size);  memmap_init(size, nid, j, zone_start_pfn); }}
```

该函数主要用于向节点下的每个 zone 填充相关信息，在 for 循环内，循环遍历统计各个管理区最大跨度间相差的页面数 size 以及除去内存“空洞”后的实际页面数 freesize，然后通过 calc_memmap_size()计算出该管理区所需的页面管理结构占用的页面数 memmap_pages，最后可以计算得出高端内存外的系统内存共有的内存页面数（freesize-memmap_pages）。

**nr_kernel_pages 用于统计低端内存的页数**，此外循环体内的操作则是初始化内存管理区的管理结构，例如各类锁的初始化、队列初始化。其中 set_pageblock_order()用于在 CONFIG_HUGETLB_PAGE_SIZE_VARIABLE 下设置 pageblock_order 的值；setup_usemap()函数则是为了给 zone 管理结构体中的 pageblock_flags 申请内存空间的，pageblock_flags 与伙伴系统的碎片迁移算法有关。init_currently_empty_zone()则主要是初始化管理区的等待队列哈希表和等待队列，同时还初始化了与伙伴系统相关的 free_area 列表

nr_kernel_pages、nr_all_pages 和页面的关系可以参考下图：
![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171717111820985.png)

这里以我自己的私服为例，看一下我私服 node 和 zone 的情况

首先是 node 的个数，GNU/Linux 根据物理 CPU 的数量分配 node，因此可以直接查物理 CPU 的数量：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171717186670264.png)
当然，用 numactl 会更加直观：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171717242420066.png)

我的机器上只有一个 node，接下来可以用 cat /proc/zoneinfo 查看这个 node 下各个 zone 的情况：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171717346972692.png)
回到 free_area_init_core()函数的最后，memmap_init()->memmap_init_zone()，该函数主要是根据 PFN，然后通过 pfn_to_page 找到对应的 struct page 结构，并将该结构进行初始化处理，并设置 MIGRATE_MOVABLE 标志，表明可移动。

```
//遍历memblock，找到节点的内存地址范围，zone的范围不能大于这个，使用memmap_init_zone对该zone进行处理void __meminit __weak memmap_init(unsigned long size, int nid,      unsigned long zone,      unsigned long range_start_pfn){ unsigned long start_pfn, end_pfn; unsigned long range_end_pfn = range_start_pfn + size; int i; for_each_mem_pfn_range(i, nid, &start_pfn, &end_pfn, NULL) {  start_pfn = clamp(start_pfn, range_start_pfn, range_end_pfn);  end_pfn = clamp(end_pfn, range_start_pfn, range_end_pfn);  if (end_pfn > start_pfn) {   size = end_pfn - start_pfn;   memmap_init_zone(size, nid, zone, start_pfn,      MEMINIT_EARLY, NULL, MIGRATE_MOVABLE);  } }}void __meminit memmap_init_zone(unsigned long size, int nid, unsigned long zone,  unsigned long start_pfn,  enum meminit_context context,  struct vmem_altmap *altmap, int migratetype){ unsigned long pfn, end_pfn = start_pfn + size; struct page *page; if (highest_memmap_pfn < end_pfn - 1)  highest_memmap_pfn = end_pfn - 1;#ifdef CONFIG_ZONE_DEVICE /*  * Honor reservation requested by the driver for this ZONE_DEVICE  * memory. We limit the total number of pages to initialize to just  * those that might contain the memory mapping. We will defer the  * ZONE_DEVICE page initialization until after we have released  * the hotplug lock.  */ if (zone == ZONE_DEVICE) {  if (!altmap)   return;  if (start_pfn == altmap->base_pfn)   start_pfn += altmap->reserve;  end_pfn = altmap->base_pfn + vmem_altmap_offset(altmap); }#endif for (pfn = start_pfn; pfn < end_pfn; ) {  /*   * There can be holes in boot-time mem_map[]s handed to this   * function.  They do not exist on hotplugged memory.   */  if (context == MEMINIT_EARLY) {   if (overlap_memmap_init(zone, &pfn))    continue;   if (defer_init(nid, pfn, end_pfn))    break;  }  page = pfn_to_page(pfn);  __init_single_page(page, pfn, zone, nid);  if (context == MEMINIT_HOTPLUG)   __SetPageReserved(page);  /*   * Usually, we want to mark the pageblock MIGRATE_MOVABLE,   * such that unmovable allocations won't be scattered all   * over the place during system boot.   */  if (IS_ALIGNED(pfn, pageblock_nr_pages)) {   set_pageblock_migratetype(page, migratetype);   cond_resched();  }  pfn++; }}
```

### 3.3 **struct page**

最后这里简单介绍一下 struct page，内核会为每一个物理页帧创建一个 struct page 的结构体，因此要保证 page 结构体足够的小，否则仅 struct page 就要占用大量的内存，该结构有很多 union 结构，主要是用于各种算法不同数据的空间复用。

struct page 这个结构相当复杂，这里我放上网上找到的一个全局参考，可以比源码更清晰地了解整个结构体，这里我也只简单介绍里面的几个字段。

```
    struct page (include/linux/mm_types.h)                         page               +--------------------------------------------------------------+               |flags                                                         |               |   (unsigned long)                                            |   --+--       +==============================================================+     |         |..............................................................|     |         |page cache and anonymous pages                                |     |         |    +---------------------------------------------------------+     |         |    |lru                                                      |     |         |    |    (struct list_head)                                   |               |    |mapping                                                  |               |    |    (struct address_space*)                              |               |    |index                                                    |               |    |    (pgoff_t)                                            |  5 words      |    |private                                                  |   union       |    |    (unsigned long)                                      |               |    +---------------------------------------------------------+               |..............................................................|    has        |slab, slob, slub                                              |               |    +---------------------------------------------------------+  7 usage      |    |.........................................................|               |    |                            .+---------------------------|               |    |slab_list                   .|next                       |               |    |    (struct list_head)      .|   (struct page*)          |               |    |                            .|pages                      |               |    |                            .|pobjects                   |               |    |                            .|   (int)                   |               |    |                            .+---------------------------|               |    |.........................................................|               |    +---------------------------------------------------------+               |    |slab_cache                                               |               |    |    (struct kmem_cache*)                                 |               |    |freelist                                                 |               |    |    (void*)                                              |               |    +---------------------------------------------------------+               |    |.........................................................|               |    |s_mem    .counters          .+---------------------------|               |    | (void*) . (unsigned long)  .|inuse                      |               |    |         .                  .|objects                    |               |    |         .                  .|frozen                     |               |    |         .                  .|    (unsigned)             |               |    |         .                  .+---------------------------|               |    |.........................................................|               |    +---------------------------------------------------------+               |..............................................................|               |Tail pages of compound page                                   |               |    +---------------------------------------------------------+               |    |compound_head                                            |               |    |    (unsigned long)                                      |               |    |compound_dtor                                            |               |    |compound_order                                           |               |    |    (unsigned char)                                      |               |    |compound_mapcount                                        |               |    |    (atomic_t)                                           |               |    +---------------------------------------------------------+               |..............................................................|               |Second tail page of compound page                             |               |    +---------------------------------------------------------+               |    |_compound_pad_1                                          |               |    |_compound_pad_2                                          |               |    |    (unsigned long)                                      |               |    |deferred_list                                            |               |    |    (struct list_head)                                   |               |    +---------------------------------------------------------+               |..............................................................|               |Page table pages                                              |               |    +---------------------------------------------------------+               |    |_pt_pad_1                                                |               |    |    (unsigned long)                                      |               |    |pmd_huge_pte                                             |               |    |    (pgtable_t)                                          |               |    |_pt_pad_2                                                |               |    |    (unsigned long)                                      |               |    |.........................................................|               |    |pt_mm                         .pt_frag_refcount          |               |    |    (struct mm_struct*)       .    (atomic_t)            |               |    |.........................................................|               |    |ptl                                                      |               |    |    (spinlock_t/spinlock_t *)                            |               |    +---------------------------------------------------------+               |..............................................................|               |ZONE_DEVICE pages                                             |               |    +---------------------------------------------------------+               |    |pgmap                                                    |               |    |    (struct dev_pagemap*)                                |               |    |hmm_data                                                 |               |    |_zd_pad_1                                                |     |         |    |    (unsigned long)                                      |     |         |    +---------------------------------------------------------+     |         |..............................................................|     |         |rcu_head                                                      |     |         |    (struct rcu_head)                                         |     |         |..............................................................|   --+--       +==============================================================+     |         |..............................................................|               |            .                 .                .              |   4 bytes     |_mapcount   .page_type        .active          .units         |    union      |  (atomic_t).   (unsigned int).  (unsigned int).   (int)      |               |            .                 .                .              |     |         |..............................................................|   --+--       +==============================================================+               |_refcount                                                     |               |     (atomic_t)                                               |               |mem_cgroup                                                    |               |     (struct mem_cgroup)                                      |               |virtual                                                       |               |     (void *)                                                 |               |_last_cpupid                                                  |               |     (int)                                                    |               +--------------------------------------------------------------+
```

首先介绍一下**flags，它描述 page 的状态和其它的一些信息**，如下图。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171717523047065.png)

主要分为 4 部分，其中标志位 flag 向高位增长，其余位字段向低位增长，中间存在空闲位。

- section：主要用于 sparse memory 内存模型，即 section 号。
- node：NUMA 节点号，标识该 page 属于哪一个节点。
- zone：内存域标志，标识该 page 属于哪一个 zone。
- flag：page 的状态标识，常用的有：

```
page-flags.henum pageflags { PG_locked,  /* 表示页面已上锁，不要访问 */ PG_error,     /* 表示页面发生IO错误 */ PG_referenced, /* 用于RCU算法 */ PG_uptodate,   /* 表示页面内容有效，当该页面上读操作完成后，设置该标志位 */ PG_dirty,      /* 表示页面是脏页，内容被修改过 */ PG_lru,        /* 表示该页面在lru链表中 */ PG_active,     /* 表示该页面在活跃lru链表中 */ PG_slab,       /* 表示该页面是属于slab分配器创建的slab */ PG_owner_priv_1, /* 页面的所有者使用，如果是pagecache页面，文件系统可能使用*/ PG_arch_1,       /* 与体系架构相关的页面状态位 */ PG_reserved,     /* 表示该页面不可被换出，防止该page被交换到swap */ PG_private,  /* 如果page中的private成员非空，则需要设置该标志，如果是pagecache, 包含fs-private data */ PG_writeback,  /* 页面正在回写 */ PG_head,  /* A head page */ PG_swapcache,  /* 表示该page处于swap cache中 */ PG_reclaim,  /* 表示该page要被回收，决定要回收某个page后，需要设置该标志 */ PG_swapbacked,  /* 该page的后备存储器是swap/ram，一般匿名页才可以回写swap分区 */ PG_unevictable,  /* 该page被锁住，不能回收，并会出现在LRU_UNEVICTABLE链表中 */ ......}
```

内核定义了一些**标准宏，用于检查页面是否设置了某个特定的标志位或者用于操作某些特定的标志位**，比如

```
PageXXX()（检查是否设置）SetPageXXX()ClearPageXXX()
```

**伙伴系统**，内存被分成含有很多页面的大块，每一块都是 2 个页面大小的方幂。如果找不到想要的块, 一个大块会被分成两部分，这两部分彼此就成为伙伴。其中一半被用来分配，而另一半则空闲。这些块在以后分配的过程中会继续被二分直至产生一个所需大小的块。当一个块被最终释放时, 其伙伴将被检测出来, 如果伙伴也空闲则合并两者

**slab**，Slab 对小对象进行分配，不用为每个小对象去分配页，节省了空间。内核中一些小对象在创建析构时很频繁，Slab 对这些小对象做缓存，可以重复利用一些相同的对象，减少内存分配次数

接着看**struct list_head lru**，链表头，具体作用得看 page 处于什么用途中，如果是伙伴系统则用于连接相同的伙伴，通过第一个 page 可以找到伙伴中所有的 page；如果是 slab，page->lru.next 指向 page 驻留的的缓存管理结构，page->lru.prec 指向保存该 page 的 slab 管理结构；而当 page 被用户态使用或被当做页缓存使用时，lru 则用于将该 page 连入 zone 中相应的 lru 链表，供内存回收时使用

**struct address_space \*mapping**，当 mapping 为 NULL 时，该 page 为交换缓存（swap）；当 mapping 不为 NULL 且第 0 位为 0，该 page 为页缓存或文件映射，mapping 指向文件的地址空间；当 mapping 不为 NULL 且第 0 位为 1，该 page 为匿名页（匿名映射），mapping 指向 struct anon_vma 对象

**pgoff_t index**，映射虚拟内存空间里的地址偏移，一个文件可能只映射其中的一部分，假设映射了 1M 的空间，index 指的是在 1M 空间内的偏移，而不是在整个文件内的偏移

**unsigned long private**，私有数据指针

**atomic_t _mapcount**，该 page 被页表映射的次数，即这个 page 被多少个进程共享，初始值为-1（非伙伴系统，如果是伙伴系统则为 PAGE_BUDDY_MAPCOUNT_VALUE），例如只被一个进程的页表映射的话，值为 0

**atomic_t _refcount**，页表引用计数，内核要操作该 page 时，引用计数会+1，操作完成后则-1，当引用计数为 0 时，表示该 page 没有被引用到，这时候就可以解除该 page 的映射（虚拟页-物理页，该物理页是占用内存的）（用于内存回收）

更详细的内容可以参考源代码~

ok，回到 memmap_init_zone()，直接看关键函数***_init\*single_page**()

```
static void __meminit __init_single_page(struct page *page, unsigned long pfn,    unsigned long zone, int nid){ mm_zero_struct_page(page); //page初始化，根据page大小还有一些特殊操作 set_page_links(page, zone, nid, pfn); //flags初始化，将页面映射到zone和node init_page_count(page); //page的_refcount设置为1 page_mapcount_reset(page); //page的_mapcount设置为-1 INIT_LIST_HEAD(&page->lru); //初始化lru，指向自身 ......}
```

至此，free_area_init_node()的初始化操作执行完毕，据前面分析可以知道该函数**主要是将整个 linux 物理内存管理框架进行初始化，包括内存管理节点 node、管理区 zone 以及页面管理 page 等数据的初始化**。

回到前面的 free_area_init()函数的循环体内的最后两个函数 node_set_state()和 check_for_memory()，node_set_state()主要是对 node 节点进行状态设置，而 check_for_memory()则是做内存检查。

到这里，内存管理框架的构建基本完毕。

```
void __init setup_arch(char **cmdline_p){    ...... max_pfn = e820__end_of_raCm_pfn(); //max_pfn初始化 ...... find_low_pfn_range(); //max_low_pfn、高端内存初始化 ...... ...... early_alloc_pgt_buf(); //页表缓冲区分配 reserve_brk(); //缓冲区加入memblock.reserve    ......    e820__memblock_setup(); //memblock.memory空间初始化 启动 ......    init_mem_mapping(); //低端内存内核页表初始化 高端内存固定映射区中临时映射区页表初始化  ......    initmem_init(); //high_memory初始化 通过memblock内存管理器设置node节点信息    ......    x86_init.paging.pagetable_init(); // 节点node、内存管理区zone、page初始化    ......}
```

最后补充一下 pfn 和物理地址以及 pfn 和虚拟地址的转换。

```
//物理地址->物理页#define PAGE_SHIFT _PAGE_SHIFT#define _PAGE_SHIFT 12#define phys_to_pfn(phys) ((phys) >> PAGE_SHIFT)#define pfn_to_phys(pfn) ((pfn) << PAGE_SHIFT)#define phys_to_page(phys) pfn_to_page(phys_to_pfn(phys))#define page_to_phys(page) pfn_to_phys(page_to_pfn(page))
```

pfn_to_page、page_to_pfn 可以参考上面的内存模型，不同的模型实现的细节不一样。

这里可以看出物理页的大小是 4096，即 4kb，虽然内核在虚拟地址中是在高地址的，但是在物理地址中是从 0 开始的。

在 linux 内核直接映射区里内核逻辑地址与物理页的转换关系如下：

```
#define pfn_to_virt(pfn) __va(pfn_to_phys(pfn))#define virt_to_pfn(kaddr) (phys_to_pfn(__pa(kaddr)))#define virt_to_page(kaddr) pfn_to_page(virt_to_pfn(kaddr))#define page_to_virt(page) pfn_to_virt(page_to_pfn(page))#define __pa(x)         ((unsigned long) (x) - PAGE_OFFSET)#define __va(x)         ((void *)((unsigned long) (x) + PAGE_OFFSET))
```

PAGE_OFFSET 与具体的架构有关，在 x86_32 中，PAGE_OFFSET 是 0xC0000000，即 32 位系统中，内核的逻辑地址只有高位的 1GB

## 4. 总结

Linux 内存管理是一个很复杂的“工程”，Linux 会通过中断调用获取被 BIOS 保留的内存地址范围以及系统可以使用的内存地址范围。在内核初始化阶段，通过 memblock 内存分配器，实现页分配器初始化之前的内存管理和分配请求，memblock 内存区管理算法将可用可分配的内存用 memblock.memory 进行管理，已分配的内存用 memblock.reserved 进行管理，只要内存块加入到 memblock.reserved 里面就表示该内存被申请占用了，申请和释放的操作都集中在 memblock.reserved，这个算法效率不高，但是却是合理的，因为在内核初始化阶段并没有太多复杂的内存操作场景，而且很多地方都是申请的内存都是永久使用的。为了合理地利用 4G 的内存空间，Linux 采用了 3：1 的策略，即内核占用 1G 的线性地址空间，用户占用 3G 的线性地址空间，且 Linux 采用了一种折中方案是只对 1G 内核空间的前 896 MB 按线性映射, 剩下的 128 MB 采用动态映射，即走多级页表翻译，这样，内核态能访问空间就更多了。

传统的多核运算是使用 SMP(Symmetric Multi-Processor )模式，将多个处理器与一个集中的存储器和 I/O 总线相连，所有处理器访问同一个物理存储器，因此 SMP 系统有时也被称为一致存储器访问（UMA）结构体系。而 NUMA 模式是一种分布式存储器访问方式，处理器可以同时访问不同的存储器地址，大幅度提高并行性。NUMA 模式下系统的每个 CPU 都有本地内存，可支持快速访问，各个处理器之间通过总线连接起来，以支持对其它 CPU 本地内存的访问，但是这些访问要比处理器本地内存的慢。Linux 内核通过插入一些兼容层，使两个不同体系结构的差异被隐藏，两种模式都使用了同一个数据结构，另外 linux 的物理内存管理机制将物理内存划分为三个层次来管理，依次是：Node（存储节点）、Zone（管理区）和 Page（页面）。

Linux 内存管理的内容十分多且复杂，上面介绍到的也只是其中的一部分，如果感兴趣的话可以下载一份源代码，然后细细品味。

## 5.参考文献

Linux 内存管理-Zone：https://blog.csdn.net/wyy4045/article/details/81776277

Linux 内存管理-Node：https://blog.csdn.net/wyy4045/article/details/81708895

内存管理框架：https://www.jeanleo.com/

memblock：https://biscuitos.github.io/blog/MMU-ARM32-MEMBLOCK-index/

Zone_sizes_init：http://www.soolco.com/post/19152_1_1.html

内核页表：https://www.daimajiaoliu.com/daima/47db35735100402

linux 内存模型：http://www.wowotech.net/memory_management/memory_model.html

linux 中的分页机制：http://edsionte.com/techblog/archives/3435

linux 内核介绍：https://richardweiyang-2.gitbook.io/kernel-exploring

struct page：https://blog.csdn.net/gatieme/article/details/52384636

页表初始化：https://www.cnblogs.com/tolimit/p/4585803.html

原文作者：dengxuanshi，腾讯 IEG 后台开发工程师

原文链接：https://mp.weixin.qq.com/s/fPPlzx8yfNvC7U6_y8RKPQ