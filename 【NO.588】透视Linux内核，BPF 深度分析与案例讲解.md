# 【NO.588】透视Linux内核，BPF 深度分析与案例讲解

本次主要对BPF的部分原理、应用案例上进行一次分析记录。

## 1.BPF介绍

当内核触发事件时，BPF虚拟机能够运行相应的BPF程序指令，但是并不是意味着BPF程序能访问内核触发的所有事件。将BPF目标文件加载到BPF虚拟机时，需要确定特定的程序类型，内核根据BPF程序类型决定BPF程序的上下文参数、在什么事件发生的时候触发，如BPF_PROG_TYPE_SOCKET_FILTER型程序(套接字过滤器程序)的上下文参数是struct __sk_buff(注意并不是struct sk_buff,其只是对struct sk_buff结构体的关键成员变量的封装)，套接字过滤器程序在嗅探到网络收发包事件上触发；BPF_PROG_TYPE_KPROBE(kprobe类型BPF程序)的上下文参数是struct pt regs *ctx，可以从中访问寄存器，BPF附件都kprobe后，启动并命中时触发。

上面的描述中涉及到的几个重要的点：

什么是BPF虚拟机？什么是BPF程序指令？BPF文件目标格式是什么？BPF程序类型有哪些？用户编写的程序是如何加载的？下面进行分析：

## 2.BPF指令与BPF虚拟机

BPF指令解析：

核心的结构体如下，每一条eBPF指令都以一个bpf_insn来表示，在cBPF中是其他的一个结构体(struct sock_filter	)，不过最终都会转换成统一的格式，这里我们只研究eBPF。

```text
struct bpf_insn {
 __u8 code;  /* opcode */
 __u8 dst_reg:4; /* dest register */
 __u8 src_reg:4; /* source register */
 __s16 off;  /* signed offset */
 __s32 imm;  /* signed immediate constant */
};
```

由结构体中`__u8	code;`可以知道，一条bpf指令是8个字节长。这8位的0，1，2位表示的是该操作指令的类别，共8种：

BPF_LD(0X00) 、BPF_LDX(0x01) 、BPF_ST(0x02)、BPF_STX(0x03) 、BPF_ALU(0x04)、BPF_JMP(0x05)、BPF_RET(0x06) 、BPF_MISC(0x07)

BPF(默认指eBPF非cBPF) 程序指令都是64位的，使用了 11 个 64位寄存器，32位称为半寄存器（subregister）和一个程序计数器（program counter），一个大小为 512 字节的 BPF 栈。所有的 BPF 指令都有着相同的编码方式。eBPF虚拟指令系统属于RISC，拥有11个虚拟寄存器，r0-r10，在实际运行时，虚拟机会把这11个寄存器一 一对应于硬件CPU的物理寄存器。如下是x64为例，虚拟寄存器与真实的寄存器的对应关系以及BPF指令的编码方式：

```text
   //R0 - 保存返回值
    //R1-R5 参数传递
    //R6-R9 保存临时变量
    //R10 只读，用做栈指针
    R0 – rax
    R1 - rdi
    R2 - rsi
    R3 - rdx
    R4 - rcx
    R5 - r8
    R6 - rbx
    R7 - r13
    R8 - r14
    R9 - r15
    R10 – rbp（帧指针，frame pointer）
```



```text
msb                                                        lsb
+------------------------+----------------+----+----+--------+
|immediate               |offset          |src |dst |opcode  |
+------------------------+----------------+----+----+--------+
```

从最低位到最高位分别是：

- 8 位的 opcode，有 BPF_X 类型的基于寄存器的指令，也有 BPF_K 类型的基于立即数的指令
- 4 位的目标寄存器 (dst)
- 4 位的原始寄存器 (src)
- 16 位的偏移（有符号），是相对于栈、映射值（map values）、数据包（packet data）等的相对偏移量
- 32 位的立即数 (imm)（有符号）

大多数指令都不会使用所有的域，未被使用的部分就会被置为0。编程时只需要将我们想要程序放到一个`bpf_insn`数组中,也就代表了整个BPF程序的所有指令，如下所示:

```text
struct bpf_insn insn[] = {
   BPF_MOV64_IMM(BPF_REG_1, 0xa21), 
      ......
    BPF_EXIT_INSN(),
};
```

如上面的 BPF_MOV64_IMM与 BPF_EXIT_INSN在内核include\linux\filter.h中的宏定义：

```text
#define BPF_MOV64_IMM(DST, IMM)     \
 ((struct bpf_insn) {     \
  .code  = BPF_ALU64 | BPF_MOV | BPF_K,  \
  .dst_reg = DST,     \
  .src_reg = 0,     \
  .off   = 0,     \
  .imm   = IMM })
  
#define BPF_EXIT_INSN()      \
 ((struct bpf_insn) {     \
  .code  = BPF_JMP | BPF_EXIT,   \
  .dst_reg = 0,     \
  .src_reg = 0,     \
  .off   = 0,     \
  .imm   = 0 })  
```

再例如，编写程序kprobe类型的BPF程序，当触发相应事件时输出helloword,。

将相应的BPF内核代码，写入结构体bpf_insn_prog中如下：

```text
struct bpf_insn prog[] = {
  BPF_MOV64_IMM(BPF_REG_1, 0xa21),        /* '!\n' */
        BPF_STX_MEM(BPF_H, BPF_REG_10, BPF_REG_1, -4),
        BPF_MOV64_IMM(BPF_REG_1, 0x646c726f),   /* 'orld' */
        BPF_STX_MEM(BPF_W, BPF_REG_10, BPF_REG_1, -8),
        BPF_MOV64_IMM(BPF_REG_1, 0x57202c6f),   /* 'o, W' */
        BPF_STX_MEM(BPF_W, BPF_REG_10, BPF_REG_1, -12),
        BPF_MOV64_IMM(BPF_REG_1, 0x6c6c6548),   /* 'Hell' */
        BPF_STX_MEM(BPF_W, BPF_REG_10, BPF_REG_1, -16),
        BPF_MOV64_IMM(BPF_REG_1, 0),   
        BPF_STX_MEM(BPF_B, BPF_REG_10, BPF_REG_1, -2),
        BPF_MOV64_REG(BPF_REG_1, BPF_REG_10),
        BPF_ALU64_IMM(BPF_ADD, BPF_REG_1, -16),
        BPF_MOV64_IMM(BPF_REG_2, 15),   
        BPF_RAW_INSN(BPF_JMP|BPF_CALL, 0,0,0, BPF_FUNC_trace_printk),
        BPF_MOV64_IMM(BPF_REG_0, 0),
        BPF_EXIT_INSN(),
 };
```

但是采用这种方式进行BPF编程需要熟悉BPF汇编，不建议使用，开发者往往使用C语言、python等高级语言(如BCC)编写BPF程序，然后通过llvm、clang等编译器将其编译成BPF字节码。

例如下面的BPF Hello word程序：

hello_kern.c:

```text
#include <uapi/linux/bpf.h>
#include <linux/version.h>
#include "bpf_helpers.h"
SEC("tracepoint/sched/sched_switch")
int bpf_prog(void *ctx){
   char msg[] = "hello BPF!\n";
   bpf_trace_printk(msg, sizeof(msg));
   return 0;
}

char _license[] SEC("license") = "GPL";
```

hello_user.c:

```text
#include <stdio.h>
#include "bpf_load.h"

int main(int argc, char **argv)
{
 if( load_bpf_file("hello_kern.o") != 0)
 {
  printf("The kernel didn't load BPF program\n");
  return -1;
 }

 read_trace_pipe();
 return 0;
}
```

编译运行该程序，可参考该链接([https://cloudnative.to/blog/compile-bpf-examples/](https://link.zhihu.com/?target=https%3A//cloudnative.to/blog/compile-bpf-examples/)) 主要是修改内核案例中写好的Makefile文件。

## 3.BPF目标文件

将内核给出的BPF案例进行编译后(BPF编程环境、编译参考[BPF编程环境搭建](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzg5MTU1ODgyMA%3D%3D%26mid%3D2247483953%26idx%3D1%26sn%3Df204e441e066302b9999cdcff690981d%26chksm%3Dcfcaccfaf8bd45ec2912ab7fa388a0286659fb6c315bf8244ba1bda5b981fe7f7ab5fbf6dc1d%26scene%3D21%23wechat_redirect))，编译后生成的ELF文件格式的目标文件，随便选择其中一个(sockex1_kern.o)进行分析：

**查看文件头信息**

```text
#如下Machine字段，显示：Linux BPF
dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ readelf -h sockex1_kern.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Linux BPF
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          528 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         10
  Section header string table index: 1
```

**查看文件的节头信息**

如下所示，13行就是指明此BPF程序的类型，“socket”就是BPF_PROG_TYPE_SOCKET_FILTER类型，下面章节会进行介绍，其他的节头如"Map"、"license"是用户编程通过SEC()宏进行自定义的节头。

> SEC宏的定义：#define SEC(NAME) **attribute**((section(NAME), used))

```text
dx@ubuntu:/usr/src/linux-4.15.0/samples/bpf$ readelf -S sockex1_kern.o
There are 10 section headers, starting at offset 0x210:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .strtab           STRTAB           0000000000000000  000001b8
       0000000000000056  0000000000000000           0     0     1
  [ 2] .text             PROGBITS         0000000000000000  00000040
       0000000000000000  0000000000000000  AX       0     0     4
  [ 3] socket            PROGBITS         0000000000000000  00000040
       0000000000000078  0000000000000000  AX       0     0     8
  [ 4] .relsocket        REL              0000000000000000  00000198
       0000000000000010  0000000000000010           9     3     8
  [ 5] maps              PROGBITS         0000000000000000  000000b8
       000000000000001c  0000000000000000  WA       0     0     4
  [ 6] license           PROGBITS         0000000000000000  000000d4
       0000000000000004  0000000000000000  WA       0     0     1
  [ 7] .eh_frame         PROGBITS         0000000000000000  000000d8
       0000000000000030  0000000000000000   A       0     0     8
  [ 8] .rel.eh_frame     REL              0000000000000000  000001a8
       0000000000000010  0000000000000010           9     7     8
  [ 9] .symtab           SYMTAB           0000000000000000  00000108
       0000000000000090  0000000000000018           1     3     8
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

如程序sockex1_kern.c:

```text
#include <uapi/linux/bpf.h>
#include <uapi/linux/if_ether.h>
#include <uapi/linux/if_packet.h>
#include <uapi/linux/ip.h>
#include "bpf_helpers.h"
//定义了一个MAP头，声名要创建一个MAP，内核识别到"map"后会创建相应的内存
struct bpf_map_def SEC("maps") my_map = {
 .type = BPF_MAP_TYPE_ARRAY,
 .key_size = sizeof(u32),
 .value_size = sizeof(long),
 .max_entries = 256,
};
//定义了一个socket头，声明了该BPF的程序类型，而程序类型决定了什么时候触发BPF内核程序
SEC("socket1")
int bpf_prog1(struct __sk_buff *skb)
{
 int index = load_byte(skb, ETH_HLEN + offsetof(struct iphdr, protocol));
 long *value;

 if (skb->pkt_type != PACKET_OUTGOING)
  return 0;

 value = bpf_map_lookup_elem(&my_map, &index);
 if (value)
  __sync_fetch_and_add(value, skb->len);

 return 0;
}
//定义了一个license头，声名遵循的GPL协议
char _license[] SEC("license") = "GPL";
```

而声名的这些section，内核是如何识别、处理这些section的(也就是说BPF虚拟机是如何根据SEC属性进行执行相应程序的)将在后面章节进行分析。

下面将分析一下，BPF虚拟机是如何根据程序定义的section进行处理的：

上面的sockex1_kern.c通过编译成sockex1_kern.o的目标文件，并最终用户在用户态通过BPF辅助函数将sockex1_kern.o插入到内核，在BPF虚拟机上处理。

其中最重要的函数就是：`load_bpf_file` (samples/bpf/bpf_load.c)

> 用户态：main->`load_bpf_file` ->do_load_bpf_file

在`do_load_bpf_file`：

```text
static int do_load_bpf_file(const char *path, fixup_map_cb fixup_map)
{
    ......
        
// 循环读取elf中所有的sections,获取license和map相关信息
 for (i = 1; i < ehdr.e_shnum; i++) {
  if (get_sec(elf, i, &ehdr, &shname, &shdr, &data))//shname为“section”的名字
   continue;

  if (0) /* helpful for llvm debugging */
   printf("section %d:%s data %p size %zd link %d flags %d\n",
          i, shname, data->d_buf, data->d_size,
          shdr.sh_link, (int) shdr.sh_flags);
        //如果license字段
  if (strcmp(shname, "license") == 0) {
   processed_sec[i] = true;
   memcpy(license, data->d_buf, data->d_size);//把data->d_buf拷贝到license数组中
  } else if (strcmp(shname, "version") == 0) {   //如果是"version"字段
   processed_sec[i] = true;
   if (data->d_size != sizeof(int)) {
    printf("invalid size of version section %zd\n",
           data->d_size);
    return 1;
   }
   memcpy(&kern_version, data->d_buf, sizeof(int));//把data-d_buf拷贝到kern_version变量中
  } else if (strcmp(shname, "maps") == 0) { //如果是maps字段
   int j;

   maps_shndx = i;
   data_maps = data;
   for (j = 0; j < MAX_MAPS; j++)
    map_data[j].fd = -1;
  } else if (shdr.sh_type == SHT_SYMTAB) {
   strtabidx = shdr.sh_link;
   symbols = data;
  }
 }

   ......
       
 //解析map section,并创建bpf系统调用创建map
 if (data_maps) {
        //使用elf中的map section填充map_data
  nr_maps = load_elf_maps_section(map_data, maps_shndx,
      elf, symbols, strtabidx);
  if (nr_maps < 0) {
   printf("Error: Failed loading ELF maps (errno:%d):%s\n",
          nr_maps, strerror(-nr_maps));
   ret = 1;
   goto done;
  }
        /*
         使用系统调用syscall(__NR_bpf,0,attr,size);创建各个map,此时用户空间可以
         使用两个全局变量:map_fd,map_data来定位创建的map,全局变量map_data_count记录着
         该程序创建的map数量
        */
  if (load_maps(map_data, nr_maps, fixup_map))
   goto done;
  map_data_count = nr_maps;

  processed_sec[maps_shndx] = true;
 }

 /* process all relo sections, and rewrite bpf insns for maps */
 for (i = 1; i < ehdr.e_shnum; i++) {
  if (processed_sec[i])
   continue;

  if (get_sec(elf, i, &ehdr, &shname, &shdr, &data))
   continue;

  if (shdr.sh_type == SHT_REL) {
   struct bpf_insn *insns;

   /* locate prog sec that need map fixup (relocations) */
   if (get_sec(elf, shdr.sh_info, &ehdr, &shname_prog,
        &shdr_prog, &data_prog))
    continue;

   if (shdr_prog.sh_type != SHT_PROGBITS ||
       !(shdr_prog.sh_flags & SHF_EXECINSTR))
    continue;

   insns = (struct bpf_insn *) data_prog->d_buf;
   processed_sec[i] = true; /* relo section */

   if (parse_relo_and_apply(data, symbols, &shdr, insns,
       map_data, nr_maps))
    continue;
  }
 }

 /* load programs */
 for (i = 1; i < ehdr.e_shnum; i++) {

  if (processed_sec[i])
   continue;

  if (get_sec(elf, i, &ehdr, &shname, &shdr, &data))
   continue;
        /*判断定义的程序类型并加载，可以看到判断的时候，给出了判断的字符串的大小，也就是编程时，
        我们可以定义Section("xdp1")、Section("xdp2")、Section("socket1")...去区别相同程序类型的不同Section
        */
  if (memcmp(shname, "kprobe/", 7) == 0 ||
      memcmp(shname, "kretprobe/", 10) == 0 ||
      memcmp(shname, "tracepoint/", 11) == 0 ||
      memcmp(shname, "xdp", 3) == 0 ||
      memcmp(shname, "perf_event", 10) == 0 ||
      memcmp(shname, "socket", 6) == 0 ||
      memcmp(shname, "cgroup/", 7) == 0 ||
      memcmp(shname, "sockops", 7) == 0 ||
      memcmp(shname, "sk_skb", 6) == 0) {
   ret = load_and_attach(shname, data->d_buf,
           data->d_size);
   if (ret != 0)
    goto done;
  }
 }
......
```

上面的do_load_bpf_file函数判断完，程序类型后调用了load_and_attach函数：

```text
static int load_and_attach(const char *event, struct bpf_insn *prog, int size)
{   /*
     根据判断的程序类型，给以下变量赋值0、1
    */
 bool is_socket = strncmp(event, "socket", 6) == 0;
 bool is_kprobe = strncmp(event, "kprobe/", 7) == 0;
 bool is_kretprobe = strncmp(event, "kretprobe/", 10) == 0;
 bool is_tracepoint = strncmp(event, "tracepoint/", 11) == 0;
 bool is_xdp = strncmp(event, "xdp", 3) == 0;
 bool is_perf_event = strncmp(event, "perf_event", 10) == 0;
 bool is_cgroup_skb = strncmp(event, "cgroup/skb", 10) == 0;
 bool is_cgroup_sk = strncmp(event, "cgroup/sock", 11) == 0;
 bool is_sockops = strncmp(event, "sockops", 7) == 0;
 bool is_sk_skb = strncmp(event, "sk_skb", 6) == 0;
 size_t insns_cnt = size / sizeof(struct bpf_insn);
 enum bpf_prog_type prog_type;
 char buf[256];
 int fd, efd, err, id;
 struct perf_event_attr attr = {};

 attr.type = PERF_TYPE_TRACEPOINT;
 attr.sample_type = PERF_SAMPLE_RAW;
 attr.sample_period = 1;
 attr.wakeup_events = 1;
    /*
    开始创建SEC头和实际prog_type之间的关联
    */
     if (is_socket) {
  prog_type = BPF_PROG_TYPE_SOCKET_FILTER;
 } else if (is_kprobe || is_kretprobe) {
  prog_type = BPF_PROG_TYPE_KPROBE;
 } else if (is_tracepoint) {
  prog_type = BPF_PROG_TYPE_TRACEPOINT;
 } else if (is_xdp) {
  prog_type = BPF_PROG_TYPE_XDP;
 } else if (is_perf_event) {
  prog_type = BPF_PROG_TYPE_PERF_EVENT;
 } else if (is_cgroup_skb) {
  prog_type = BPF_PROG_TYPE_CGROUP_SKB;
 } else if (is_cgroup_sk) {
  prog_type = BPF_PROG_TYPE_CGROUP_SOCK;
 } else if (is_sockops) {
  prog_type = BPF_PROG_TYPE_SOCK_OPS;
 } else if (is_sk_skb) {
  prog_type = BPF_PROG_TYPE_SK_SKB;
 } else {
  printf("Unknown event '%s'\n", event);
  return -1;
 }
    fd = bpf_load_program(prog_type, prog, insns_cnt, license, kern_version,
         bpf_log_buf, BPF_LOG_BUF_SIZE);
    ......
}
```

通过分析上面的do_load_bpf_file函数，大致就明白了在写BPF程序时定义的Section如何进行判断然后进一步加载的了。

## 4.BPF程序类型

BPF 相关的程序，首先需要设置为相对应的的程序类型，截止Linux 内核 5.8 程序类型定义有 29 个，而且还是持续增加中，BPF 程序类型（prog_type）决定了程序可以调用的内核辅助函数的子集。BPF 程序类型也决定了程序输入上下文 -- bpf_context 结构的格式，其作为 BPF 程序中的第一个输入参数（数据blog），每类程序都有自己独特的上下文参数。例如，跟踪程序与套接字过滤器程序的辅助函数子集并不完全相同（尽管它们可能具有一些共同的帮助函数）。同样，跟踪程序的输入上下文（context）是一组寄存器值，如套接字过滤器的输入是一个网络数据包。

特定类型的eBPF程序可用的功能集将来可能会增加。

include/uapi/linux/bpf.h

```text
 161 enum bpf_prog_type {
 162         BPF_PROG_TYPE_UNSPEC,
 163         BPF_PROG_TYPE_SOCKET_FILTER,
 164         BPF_PROG_TYPE_KPROBE,
 165         BPF_PROG_TYPE_SCHED_CLS,
 166         BPF_PROG_TYPE_SCHED_ACT,
 167         BPF_PROG_TYPE_TRACEPOINT,
 168         BPF_PROG_TYPE_XDP,
 169         BPF_PROG_TYPE_PERF_EVENT,
 170         BPF_PROG_TYPE_CGROUP_SKB,
 171         BPF_PROG_TYPE_CGROUP_SOCK,
 172         BPF_PROG_TYPE_LWT_IN,
 173         BPF_PROG_TYPE_LWT_OUT,
 174         BPF_PROG_TYPE_LWT_XMIT,
 175         BPF_PROG_TYPE_SOCK_OPS,
 176         BPF_PROG_TYPE_SK_SKB,
 177         BPF_PROG_TYPE_CGROUP_DEVICE,
 178         BPF_PROG_TYPE_SK_MSG,
 179         BPF_PROG_TYPE_RAW_TRACEPOINT,
 180         BPF_PROG_TYPE_CGROUP_SOCK_ADDR,
 181         BPF_PROG_TYPE_LWT_SEG6LOCAL,
 182         BPF_PROG_TYPE_LIRC_MODE2,
 183         BPF_PROG_TYPE_SK_REUSEPORT,
 184         BPF_PROG_TYPE_FLOW_DISSECTOR,
 185         BPF_PROG_TYPE_CGROUP_SYSCTL,
 186         BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE,
 187         BPF_PROG_TYPE_CGROUP_SOCKOPT,
 188         BPF_PROG_TYPE_TRACING,
 189         BPF_PROG_TYPE_STRUCT_OPS,
 190         BPF_PROG_TYPE_EXT,
 191         BPF_PROG_TYPE_LSM,
 192 };
```

为了直观看出来这些程序类型，在bpftool源码中摘取了这些名字和类型对应的关系

```text
const char * const prog_type_name[] = {
        [BPF_PROG_TYPE_UNSPEC]                  = "unspec",
        [BPF_PROG_TYPE_SOCKET_FILTER]           = "socket_filter",
        [BPF_PROG_TYPE_KPROBE]                  = "kprobe",
        [BPF_PROG_TYPE_SCHED_CLS]               = "sched_cls",
        [BPF_PROG_TYPE_SCHED_ACT]               = "sched_act",
        [BPF_PROG_TYPE_TRACEPOINT]              = "tracepoint",
        [BPF_PROG_TYPE_XDP]                     = "xdp",
        [BPF_PROG_TYPE_PERF_EVENT]              = "perf_event",
        [BPF_PROG_TYPE_CGROUP_SKB]              = "cgroup_skb",
        [BPF_PROG_TYPE_CGROUP_SOCK]             = "cgroup_sock",
        [BPF_PROG_TYPE_LWT_IN]                  = "lwt_in",
        [BPF_PROG_TYPE_LWT_OUT]                 = "lwt_out",
        [BPF_PROG_TYPE_LWT_XMIT]                = "lwt_xmit",
        [BPF_PROG_TYPE_SOCK_OPS]                = "sock_ops",
        [BPF_PROG_TYPE_SK_SKB]                  = "sk_skb",
        [BPF_PROG_TYPE_CGROUP_DEVICE]           = "cgroup_device",
        [BPF_PROG_TYPE_SK_MSG]                  = "sk_msg",
        [BPF_PROG_TYPE_RAW_TRACEPOINT]          = "raw_tracepoint",
        [BPF_PROG_TYPE_CGROUP_SOCK_ADDR]        = "cgroup_sock_addr",
        [BPF_PROG_TYPE_LWT_SEG6LOCAL]           = "lwt_seg6local",
        [BPF_PROG_TYPE_LIRC_MODE2]              = "lirc_mode2",
        [BPF_PROG_TYPE_SK_REUSEPORT]            = "sk_reuseport",
        [BPF_PROG_TYPE_FLOW_DISSECTOR]          = "flow_dissector",
        [BPF_PROG_TYPE_CGROUP_SYSCTL]           = "cgroup_sysctl",
        [BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE] = "raw_tracepoint_writable",
        [BPF_PROG_TYPE_CGROUP_SOCKOPT]          = "cgroup_sockopt",
        [BPF_PROG_TYPE_TRACING]                 = "tracing",
        [BPF_PROG_TYPE_STRUCT_OPS]              = "struct_ops",
        [BPF_PROG_TYPE_EXT]                     = "ext",
        [BPF_PROG_TYPE_LSM]                     = "lsm",
        [BPF_PROG_TYPE_SK_LOOKUP]               = "sk_lookup",
}
```

这些程序类型，我们可以认为是有两大类，其一是用来跟踪，编写程序来更好地了解系统正在发生什么，这类程序提供了系统行为以及系统硬件的直接信息，同时可以访问特定程序的内存区域，从运行进程中提取执行跟踪信息。其二是网络，这类程序可以监测和控制系统的网络流量，注意：“监测”、“控制”，“监测”也就是说我们可以观测，如套接字过滤器程序、“控制”可以去修改网络一些信息，如XDP程序做过滤、拒绝数据包、重定向数据包。

上面提到了每类BPF程序都有自己独特的上下文参数和自己触发的事件，下面将分析其中一种：BPF_PROG_TYPE_SOCKET_FILTER类型的上下文参数和触发的事件，其他的BPF程序类型将在今后的文章进行分析和讲解。

## 5.**套接字过滤器程序**

**BPF_PROG_TYPE_SOCKET_FILTER**

这类程序可以说是刚才提到的两种大类中：网络类，刚才提到这类程序可以监测和控制系统的网络流量，套接字过滤器程序就仅仅可以进行监测，不允许修改数据包内容或更改其目的地等。**每类程序都有自己的独特的上下文参数**，该类程序的上下文入口参数是`struct __sk_buff`，struct __sk_buff的定义在include/uapi/linux/bpf.h>，如下所示：

```text
struct __sk_buff {
 __u32 len;
 __u32 pkt_type;
 __u32 mark;
 __u32 queue_mapping;
 __u32 protocol;
 __u32 vlan_present;
 __u32 vlan_tci;
 __u32 vlan_proto;
 __u32 priority;
 __u32 ingress_ifindex;
 __u32 ifindex;
 __u32 tc_index;
 __u32 cb[5];
 __u32 hash;
 __u32 tc_classid;
 __u32 data;
 __u32 data_end;
 __u32 napi_id;

 /* Accessed by BPF_PROG_TYPE_sk_skb types from here to ... */
 __u32 family;
 __u32 remote_ip4; /* Stored in network byte order */
 __u32 local_ip4; /* Stored in network byte order */
 __u32 remote_ip6[4]; /* Stored in network byte order */
 __u32 local_ip6[4]; /* Stored in network byte order */
 __u32 remote_port; /* Stored in network byte order */
 __u32 local_port; /* stored in host byte order */
 /* ... here. */

 __u32 data_meta;
};
```

从上面的struct __sk_buff 可以看到，这个结构体来自于struct sk_buff中的一些关键字段。那么会有一个疑问，该结构体只有struct sk_buff的关键字段而且与struct sk_buff中的字段并没有做到对齐(位置对应)，是怎样做到访问到这些字段的？

> 参考：[https://lwn.net/Articles/636647/](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/636647/)
> They were few other ideas considered. At the end the best seems to be to introduce a user accessible mirror of in-kernel sk_buff structure:
> struct __sk_buff { __u32 len; __u32 pkt_type; __u32 mark; __u32 ifindex; __u32 queue_mapping;
> .......
> };
> bpf programs will do:
> int bpf_prog1(struct __sk_buff *skb) { __u32 var = skb->pkt_type;
> which will be compiled to bpf assembler as:
> dst_reg = *(u32 *)(src_reg + 4) // 4 == offsetof(struct __sk_buff, pkt_type)
> bpf verifier will check validity of access and will convert it to:
> dst_reg = *(u8 *)(src_reg + offsetof(struct sk_buff, __pkt_type_offset)) dst_reg &= 7
> since 'pkt_type' is a bitfield.

上面这段参考，大概就是在说struct __sk_buff包含了这个一些sk_buff结构体的关键字段，内核的BPF程序将对这些关键字段的访问转换成“真实”sk_buff的偏移量，这一点很关键！！

该程序类型用于过滤进出口网络报文，功能上和 cBPF 类似，也就是和在tcpdump中所运用的cBPF相似,可以在tcpdump系统文章中进行查看相关的实现。

关于套接字过滤器程序的案例，还是选择sockex1:

sockex1_kern.c

```text
#include <uapi/linux/bpf.h>
#include <uapi/linux/if_ether.h>
#include <uapi/linux/if_packet.h>
#include <uapi/linux/ip.h>
#include "bpf_helpers.h"
/*
预先定义好的MAP对象，然而MAP的真正创建时机，是用户态程序调用load_bpf_file时
解析到section后进行创建；这里SEC宏会在obj文件中新增一个段(section)
这里定义的是一个数组型的MAP,索引是数组下标，值是数组对应的值
*/
struct bpf_map_def SEC("maps") my_map = {
 .type = BPF_MAP_TYPE_ARRAY,
 .key_size = sizeof(u32),
 .value_size = sizeof(long),
 .max_entries = 256,
};

SEC("socket1")
    //struct __sk_buff就是该程序类型的上下文参数，封装了struct sk_buff中的一些关键的成员变量
int bpf_prog1(struct __sk_buff *skb)
{   
    //load_byte实际指向了llvm的buit-in函数asm(llvm.bpf.load.byte)，该语句是读取输入报文的包头中的协议头
 int index = load_byte(skb, ETH_HLEN + offsetof(struct iphdr, protocol));
 long *value;
     //pkt_type表明了数据包的方向，PACKET_OUTGOING表示发往其他主机的报文和本地主机的环回报文
 if (skb->pkt_type != PACKET_OUTGOING)
  return 0;

 value = bpf_map_lookup_elem(&my_map, &index);
 if (value)
  __sync_fetch_and_add(value, skb->len);//统计流量包的大小

 return 0;
}
char _license[] SEC("license") = "GPL";
```

sockex1_user.c

```text
#include <stdio.h>
#include <assert.h>
#include <linux/bpf.h>
#include "libbpf.h"
#include "bpf_load.h"
#include "sock_example.h"
#include <unistd.h>
#include <arpa/inet.h>

int main(int ac, char **argv)
{
 char filename[256];
 FILE *f;
 int i, sock;

 snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);
    //加载sockex1_kern.c编译后的sockex1_kern.o，解析出定义的section,完成创建MAP，向内核注册插入BPF代码。
 if (load_bpf_file(filename)) {
  printf("%s", bpf_log_buf);
  return 1;
 }
     //主要是完成：sock = socket(PF_PACKET, SOCK_RAW | SOCK_NONBLOCK | SOCK_CLOEXEC, htons(ETH_P_ALL));
 sock = open_raw_sock("lo");
    /*因为 sockex1_kern.o 中 bpf 程序的类型为 BPF_PROG_TYPE_SOCKET_FILTER,所以这里需要用用 SO_ATTACH_BPF 来指明程序的 sk_filter 要挂载到哪一个套接字上，其中prof_fd为注入到内核的BPF程序的描述符*/
 assert(setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, prog_fd,
     sizeof(prog_fd[0])) == 0);

 f = popen("ping -c5 localhost", "r");
 (void) f;
    //循环5次查询相应协议对应的数据包大小
 for (i = 0; i < 5; i++) {
  long long tcp_cnt, udp_cnt, icmp_cnt;
  int key;
  key = IPPROTO_TCP;
  assert(bpf_map_lookup_elem(map_fd[0], &key, &tcp_cnt) == 0);

  key = IPPROTO_UDP;
  assert(bpf_map_lookup_elem(map_fd[0], &key, &udp_cnt) == 0);

  key = IPPROTO_ICMP;
  assert(bpf_map_lookup_elem(map_fd[0], &key, &icmp_cnt) == 0);

  printf("TCP %lld UDP %lld ICMP %lld bytes\n",
         tcp_cnt, udp_cnt, icmp_cnt);
  sleep(1);
 }

 return 0;
}
```

sockex1_user.c中的：

```text
 if (load_bpf_file(filename)) {
  printf("%s", bpf_log_buf);
  return 1;
 }
```

用户空间通过辅助函数load_bpf_file加载BPF的elf文件后，将返回使用MAP两个全局变量:map_fd数组,map_data,同时也会创建 struct bpf_prog存储指令，struct bpf_prog内容如下所示,每一个load到内核的BPF程序都有一个fd(prog_fd)返回给用户态，它对应一个bpf_prog。将指令Load内核，创建struct bpf_prog存储指令外，只是第一步，成功运行还需要：

- 将bpf_prog与内核中的特定Hook点关联起来，也就是将程序挂到钩子上
- 在Hook点被访问到，取出bpf_prog，执行这些指令。

**那socket类型的BPF程序，也就是BPF_PROG_TYPE_SOCKET_FILTER类型的程序什么时候会执行注入到内核的指令？**

其实正如tcpdump原理一样，可以参考tcpdump1和tcpdump2,当绑定的指定的网卡在发包或者收包时会进行嗅探并执行注入的指令。

```text
    struct bpf_prog {
 u16   pages;  // Number of allocated pages 
 u16   jited:1, // Is our filter JIT'ed? 
    locked:1, // Program image locked? 
    gpl_compatible:1, // Is filter GPL compatible? 
    cb_access:1, // Is control block accessed? 
    dst_needed:1; // Do we need dst entry? 
 enum bpf_prog_type type;  // Type of BPF program 
 u32   len;  // Number of filter blocks 
 u32   jited_len; // Size of jited insns in bytes 
 u8   tag[BPF_TAG_SIZE];
 struct bpf_prog_aux *aux;  // Auxiliary fields 
 struct sock_fprog_kern *orig_prog; // Original BPF program 
 unsigned int  (*bpf_func)(const void *ctx,
         const struct bpf_insn *insn);
 // Instructions for interpreter 
 union {
  struct sock_filter insns[0];   //对应cBPF的指令集
  struct bpf_insn  insnsi[0];  //对应eBPF的指令集
 };
};
```

BPF_PROG_TYPE_SOCKET_FILTER用来干啥？

> filter操作包括丢弃数据包（如果程序返回0）或修改数据包（如果程序返回的长度小于原始长度）。请参见sk_filter_trim_cap（）及其对bpf_prog_run_save_cb（）的调用。请注意，我们不是修改或者丢弃原始的数据包，这些数据包都会到达原始的套接字。我们处理的是原始数据包元数据的copy，这些copy的数据包可以被raw socket访问到。除此之外，我们还可以做一些其他的事情。例如进行流量统计在BPF 的Maps中。

如何attach我们的BPF program?

> 它可以通过SO_ATTACH_BPF setsockopt()进行attach。

该类型上下文参数?

> struct __sk_buff 包含这个packet的metadata/data。他的结构在include/linux/ bpf.h中定义，并包含来自实际sk_buff的关键字段。bpf验证程序将对有效的__sk_buff字段的访问转换为“真实”sk_buff的偏移量。

它什么时候被执行？

> 绑定到指定的网卡后，在发包或者收包时会进行嗅探并执行注入的程序，该类型程序被各种协议（TCP，UDP，ICMP，原始套接字等）调用，可用于过滤inbound的流量。

原文地址：https://zhuanlan.zhihu.com/p/502555210

作者：linux