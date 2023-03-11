# 【NO.87】浅谈 Protobuf 编码

> 近日简单学习了 Protobuf 中的编码实现，总结并整理成文。本文结构总体与 Protobuf 官方文档相似，不少内容也来自官方文档，并在官方文档的基础上添加作者理解的内容，如有出入请以官方文档为准。作者水平有限，难免有疏漏之处，欢迎指正并分享您的意见。

## 1.**0x00 Before you start**

简单来说，Protobuf 的编码是基于变种的 Base128。在学习 Protobuf 编码或是 Base128 之前，先来了解下 Base64 编码。

## **2.0x01 Base 64**

当我们在计算机之间传输数据时，数据本质上是一串字节流。TCP 协议可以保证被发送的字节流正确地达到目的地（至少在出错时有一定的纠错机制），所以本文不讨论因网络因素造成的数据损坏。但数据到达目标机器之后，由于不同机器采用的字符集不同等原因，我们并不能保证目标机器能够正确地“理解”字节流。

Base 64 最初被设计是用于在邮件中嵌入文件（作为 MIME 的一部分）。它可以将任何形式的字节流编码为“安全”的字节流。何为“安全“的字节？先来看看 Base 64 是如何工作的。

假设这里有四个字节,代表你要传输的二进制数据。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151501523865652.png)

首先将这个字节流按每 6 个 bit 为一组进行分组,剩下少于 6 bits 的低位补 0。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151502001648745.png)

然后在每一组 6 bits 的高位补两个 0。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151502115959855.png)

对照 Base 64 table，字节流可以用 `ognC0w` 来表示。另外，Base64 编码是按照 6 bits 为一组进行编码，每 3 个字节的原始数据要用 4 个字节来储存，编码后的长度要为 4 的整数倍，不足 4 字节的部分要使用 pad 补齐，所以最终的编码结果为`ognC0w==`。

任意的字节流均可以使用 Base 64 进行编码，编码之后所有字节均可以用数字、字母和 + / = 号进行表示，这些都是可以被正常显示的 ascii 字符，即“安全”的字节。绝大部分的计算机和操作系统都对 ascii 有着良好的支持，保证了编码之后的字节流能被正确地复制、传播、解析。

注：下文关于字节顺序内容均基于机器采用小端模式的前提进行讨论。

## 3.**0x02 Base 128?**

Base 64 存在的问题就是，编码后的每一个字节的最高两位总是 0，在不考虑 pad 的情况下，有效 bit 只占 bit 总数的 75%，造成大量的空间浪费。是否可以进一步提高信息密度呢？

意识到这一点，你就很自然能想象出 Base 128 的大致实现思路了，将字节流按 7 bits 进行分组，然后低位补 0。

但问题来了，Base 64 实际上用了 64+1 个 ascii 字符，按照这个思路 Base 128 需要使用 128+1 个 ascii 个字符，但是 ascii 字符一共只有 128 个。另外，即使不考虑 pad，ascii 中包含了一些不可以正常打印的控制字符，编码之后的字符还可能包含会被不同操作系统转换的换行符号（10 和 13）。因此，Base 64 至今依然没有被 Base 128 替代。

Base 64 的规则因为上述限制不能完美地扩展到 Base 128，所以现有基于 Base 64 扩展而来的编码方式大部分都属于变种。如 LEB128（Little-Endian Base 128）， Base 85 （Ascii 85），以及本文的主角：Base 128 Varints。

注：下文关于字节顺序内容均基于机器采用小端模式的前提进行讨论。

## **4.0x03 Base 128 Varints**

Base 128 Varints 是 Google 开发的序列化库 Protocol Buffers 所用的编码方式。以下为 Protobuf 官方文档中对于 Varints 的解释：

> Varints are a method of serializing integers using one or more bytes. Smaller numbers take a smaller number of bytes.

使用一个或多个字节对整数进行序列化。小的数字占用更少的字节。简单来说，就是尽量只储存整数的有效位，高位的 0 尽可能抛弃。

有两个需要注意的细节：

- Base 128 Varints 只能对一部分数据结构进行编码，不适用于所有字节流（当然你可以把任意字节流转换为 string，但不是所有语言都支持这个 trick）。否则无法识别哪部分是无效的 bits。
- Base 128 Varints 编码后的字节可以不存在于 Ascii 表中，因为和 Base 64 使用场景不同，不用考虑是否能正常打印。

下面以例子进行说明 Base 128 Varints 的编码实现。

对于编码后的每个字节，低 7 位用于储存数据，最高位用来标识当前字节是否是当前整数的最后一个字节，称为最高有效位（most significant bit, msb）。msb 为 1 时，代表着后面还有数据；msb 为 0 时代表着当前字节是当前整数的最后一个字节。

举个例子，下面是编码后的整数`1`。1 只需要用一个字节就能表示完全，所以 msb 为 0。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151502239941675.png)

对于需要多个字节来储存的数据，如 300 (0b100101100)，有效位数为 9，编码后需要两个字节储存。下面是编码后的整数`300`。第一个字节的 msb 为 1，最后一个字节的 msb 为 0。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151502356867932.png)

要将这两个字节解码成整数,需要三个步骤：

- 去除 msb
- 第二步，将字节流逆序（msb 为 0 的字节储存原始数据的高位部分，小端模式）
- 最后拼接所有的 bits。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151502469788623.png)

下面一个例子展示如何将使用 Base 128 Varints 对整数进行编码。

- 将数据按每 7 bits 一组拆分
- 逆序每一个组
- 添加 msb

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151502589907756.png)

需要注意的是，无论是编码还是解码，逆序字节流这一步在机器处理中实际是不存在的，机器采用小端模式处理数据，此处逆序仅是为了符合人的阅读习惯而写出。下面展示 Go 版本的 protobuf 中关于 Base 128 Varints 的实现：

```
// google.golang.org/protobuf@v1.25.0/encoding/protowire/wire.go// AppendVarint appends v to b as a varint-encoded uint64.func AppendVarint(b []byte, v uint64) []byte { switch { case v < 1<<7:  b = append(b, byte(v)) case v < 1<<14:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte(v>>7)) case v < 1<<21:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte((v>>7)&0x7f|0x80),   byte(v>>14)) case v < 1<<28:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte((v>>7)&0x7f|0x80),   byte((v>>14)&0x7f|0x80),   byte(v>>21)) case v < 1<<35:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte((v>>7)&0x7f|0x80),   byte((v>>14)&0x7f|0x80),   byte((v>>21)&0x7f|0x80),   byte(v>>28)) case v < 1<<42:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte((v>>7)&0x7f|0x80),   byte((v>>14)&0x7f|0x80),   byte((v>>21)&0x7f|0x80),   byte((v>>28)&0x7f|0x80),   byte(v>>35)) case v < 1<<49:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte((v>>7)&0x7f|0x80),   byte((v>>14)&0x7f|0x80),   byte((v>>21)&0x7f|0x80),   byte((v>>28)&0x7f|0x80),   byte((v>>35)&0x7f|0x80),   byte(v>>42)) case v < 1<<56:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte((v>>7)&0x7f|0x80),   byte((v>>14)&0x7f|0x80),   byte((v>>21)&0x7f|0x80),   byte((v>>28)&0x7f|0x80),   byte((v>>35)&0x7f|0x80),   byte((v>>42)&0x7f|0x80),   byte(v>>49)) case v < 1<<63:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte((v>>7)&0x7f|0x80),   byte((v>>14)&0x7f|0x80),   byte((v>>21)&0x7f|0x80),   byte((v>>28)&0x7f|0x80),   byte((v>>35)&0x7f|0x80),   byte((v>>42)&0x7f|0x80),   byte((v>>49)&0x7f|0x80),   byte(v>>56)) default:  b = append(b,   byte((v>>0)&0x7f|0x80),   byte((v>>7)&0x7f|0x80),   byte((v>>14)&0x7f|0x80),   byte((v>>21)&0x7f|0x80),   byte((v>>28)&0x7f|0x80),   byte((v>>35)&0x7f|0x80),   byte((v>>42)&0x7f|0x80),   byte((v>>49)&0x7f|0x80),   byte((v>>56)&0x7f|0x80),   1) } return b}
```

从源码中可以看出，protobuf 的 varints 最多可以编码 8 字节的数据，这是因为绝大部分的现代计算机最高支持处理 64 位的整型。

## **5.0x04 其他类型**

Protobuf 不仅支持整数类型，下图列出 protobuf 支持的数据类型（wire type）。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151503114423539.png)

在上一小节中展示的编码与解码的例子中的“整数”并不是我们一般理解的整数（编程语言中的 int32，uint32 等），而是对应着上图中的 Varint。

当实际使用编程语言使用 protobuf 进行编码时经过了两步处理：

- 将编程语言的数据结构转化为 wire type。
- 根据不同的 wire type 使用对应的方法编码。前文所提到的 Base 128 Varints 用来编码 varint 类型的数据，其他 wire type 则使用其他编码方式。

```
    {obj}  -> {wire type} -> {encoded byte stream}    uint32 -> wire type 0 -> varint    int32  -> wire type 0 -> varint    bool   -> wire type 0 -> varint    string -> wire type 2 -> length-delimited    ...
```

不同语言中 wire type 实际上也可能采用了语言中的某种类型来储存 wire type 的数据。例如，Go 中使用了 uint64 来储存 wire type 0。一般来说，大多数语言中的无符号整型被转换为 varints 之后，有效位上的内容并没有改变。

下面说明部分其他数据类型到 wire type 的转换规则：

- ### **有符号整型**

采用 ZigZag 编码来将 sint32 和 sint64 转换为 wire type 0。下面是 ZigZag 编码的规则（注意是算术位移）：

```
    (n << 1) ^ (n >> 31)  // for 32-bit signed integer    (n << 1) ^ (n >> 63)  // for 64-bit signed integer
```

或者从数学意义来理解：

```
    n * 2       // when n >= 0    -n * 2 - 1  // when n < 0
```

下图展示了部分 ZigZag 编码的例子：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151503251275844.png)

如果不先采用 ZigZag 编码成 wire type，负值 sint64 直接使用 Base 128 Varints 编码之后的长度始终为`ceil(64/7)=10bytes`，浪费大量空间。使用 ZigZag 编码后，绝对值较小的负数的长度能够被显著压缩。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151503364873212.png)
对于 -234(sint32) 这个例子，编码成 varints 之前采用 ZigZag 编码，比直接编码成 varints 少用了 60%的空间。当然，ZigZag 编码也不是完美的方法。当你尝试把 sint32 或 sint64 范围内所有的整数都编码成 varints 字节流,使用 ZigZag 已经不能压缩字节数量了。ZigZag 虽然能压缩部分负数的空间，但同时正数变得需要更多的空间来储存。因此，建议在业务场景允许的场景下尽量用无符号整型，有助于进一步压缩编码后的空间。

- ### **定长数据（64-bit）**

直接采用小端模式储存，不作转换。

- ### **字符串**

以字符串`"testing"`为例

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151503531303965.png)
编码后的 value 分为两部分：

- 蓝色，表示字符串采用 UTF-8 编码后字节流的长度（bytes），采用 Base 128 Varints 进行编码。
- 白色，字符串用 UTF-8 编码后的字节流。

## **6.0x05 消息结构**

Protobuf 采用 proto3 作为 DSL 来描述其支持的消息结构。

```
syntax = "proto3";message SearchRequest {  string query = 1;  int32 page_number = 2;  int32 result_per_page = 3;}
```

设想一下这样一个场景：数据的发送方在业务迭代之后需要在消息内携带更多的字段，而有的接收方并没有更新自己的 proto 文件。要保持较好的兼容性，接收方分辨出哪些字段是自己可以识别的，哪些是不能识别的新增字段。要做到这一点，发送方在编码消息时还必须附带每个字段的 key，客户端读取到未知的 key 时，可以直接跳过对应的 value。

proto3 中每一个字段后面都有一个 `= x`，比如：

```
  string query = 1;
```

这里的等号并不是用于赋值，而是给每一个字段指定一个 ID，称为 field number。消息内同一层次字段的 field number 必须各不相同。

上面所说的 key，在 protobuf 源码中被称为 tag。tag 由 field number 和 type 两部分组成：

- field number 左移 3 bits
- 在最低 3 bits 写入 wire type

下面展示一个生成 tag 例子：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151504038447723.png)

Go 版本 Protobuf 中生成 tag 的源码：

```
// google.golang.org/protobuf@v1.25.0/encoding/protowire/wire.go// EncodeTag encodes the field Number and wire Type into its unified form.func EncodeTag(num Number, typ Type) uint64 {    return uint64(num)<<3 | uint64(typ&7)}
```

源码中生成的 tag 是 uint64，代表着 field number 可以使用 61 个 bit 吗？并非如此。事实上，tag 的长度不能超过 32 bits，意味着 field number 的最大取值为 2^29-1 (536870911)。而且在这个范围内，有一些数是不能被使用的：

- 0 ，protobuf 规定 field number 必须为正整数。
- 19000 到 19999， protobuf 仅供内部使用的保留位。

理解了生成 tag 的规则之后，不难得出以下结论：

- field number 不必从 1 开始，可以从合法范围内的任意数字开始。
- 不同字段间的 field number 不必连续，只要合法且不同即可。

但是实际上，大多数人分配 field number 还是会从 1 开始，因为 tag 最终要经过 Base 128 Varints 编码，较小的 field number 有助于压缩空间，field number 为 1 到 15 的 tag 最终仅需占用一个字节。当你的 message 有超过 15 个字段时，Google 也不建议你将 1 到 15 立马用完。如果你的业务日后有新增字段的可能，并且新增的字段使用比较频繁，你应该在 1 到 15 内预留一部分供新增的字段使用。

当你修改的 proto 文件需要注意：

- field number 一旦被分配了就不应该被更改，除非你能保证所有的接收方都能更新到最新的 proto 文件。
- 由于 tag 中不携带 field name 信息，更改 field name 并不会改变消息的结构。发送方认为的 apple 到接受方可能会被识别成 pear。双方把字段读取成哪个名字完全由双方自己的 proto 文件决定，只要字段的 wire type 和 field number 相同即可。

由于 tag 中携带的类型是 wire type，不是语言中具体的某个数据结构，而同一个 wire type 可以被解码成多种数据结构，具体解码成哪一种是根据接收方自己的 proto 文件定义的。修改 proto 文件中的类型，有可能导致错误。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151504159043275.png)

最后用一个比前面复杂一点的例子来结束本节内容：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151504234942704.png)

## **7.0x06 嵌套消息**

嵌套消息的实现并不复杂。在上一节展示的 protobuf 的 wire type 中，wire type2 （length-delimited）不仅支持 string，也支持 embedded messages。

对于嵌套消息，首先你要将被嵌套的消息进行编码成字节流，然后你就可以像处理 UTF-8 编码的字符串一样处理这些字节流：在字节流前面加入使用 Base 128 Varints 编码的长度即可。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151504322045900.png)

## **8.0x07 重复消息**

假设接收方的 proto3 中定义了某个字段（假设 field number=1），当接收方从字节流中读取到多个 field number=1 的字段时，会执行 merge 操作。merge 的规则如下：

- 如果字段为不可分割的类型，则直接覆盖
- 如果字段为 repeated，则 append 到已有字段
- 如果字段为嵌套消息，则递归执行 merge

如果字段的 field number 相同但是结构不同，则出现 error。以下为 Go 版本 Protobuf 中 merge 的部分源码：

```
// google.golang.org/protobuf@v1.25.0/proto/merge.go// Merge merges src into dst, which must be a message with the same descriptor.//// Populated scalar fields in src are copied to dst, while populated// singular messages in src are merged into dst by recursively calling Merge.// The elements of every list field in src is appended to the corresponded// list fields in dst. The entries of every map field in src is copied into// the corresponding map field in dst, possibly replacing existing entries.// The unknown fields of src are appended to the unknown fields of dst.//// It is semantically equivalent to unmarshaling the encoded form of src// into dst with the UnmarshalOptions.Merge option specified.func Merge(dst, src Message) { // TODO: Should nil src be treated as semantically equivalent to a // untyped, read-only, empty message? What about a nil dst? dstMsg, srcMsg := dst.ProtoReflect(), src.ProtoReflect() if dstMsg.Descriptor() != srcMsg.Descriptor() {  if got, want := dstMsg.Descriptor().FullName(), srcMsg.Descriptor().FullName(); got != want {   panic(fmt.Sprintf("descriptor mismatch: %v != %v", got, want))  }  panic("descriptor mismatch") } mergeOptions{}.mergeMessage(dstMsg, srcMsg)}func (o mergeOptions) mergeMessage(dst, src protoreflect.Message) { methods := protoMethods(dst) if methods != nil && methods.Merge != nil {  in := protoiface.MergeInput{   Destination: dst,   Source:      src,  }  out := methods.Merge(in)  if out.Flags&protoiface.MergeComplete != 0 {   return  } } if !dst.IsValid() {  panic(fmt.Sprintf("cannot merge into invalid %v message", dst.Descriptor().FullName())) } src.Range(func(fd protoreflect.FieldDescriptor, v protoreflect.Value) bool {  switch {  case fd.IsList():   o.mergeList(dst.Mutable(fd).List(), v.List(), fd)  case fd.IsMap():   o.mergeMap(dst.Mutable(fd).Map(), v.Map(), fd.MapValue())  case fd.Message() != nil:   o.mergeMessage(dst.Mutable(fd).Message(), v.Message())  case fd.Kind() == protoreflect.BytesKind:   dst.Set(fd, o.cloneBytes(v))  default:   dst.Set(fd, v)  }  return true }) if len(src.GetUnknown()) > 0 {  dst.SetUnknown(append(dst.GetUnknown(), src.GetUnknown()...)) }}
```

## **9.0x08 字段顺序**

Proto 文件中定义字段的顺序与最终编码结果的字段顺序无关，两者有可能相同也可能不同。当消息被编码时，Protobuf 无法保证消息的顺序，消息的顺序可能随着版本或者不同的实现而变化。任何 Protobuf 的实现都应该保证字段以任意顺序编码的结果都能被读取。

- 序列化后的消息字段顺序是不稳定的。
- 对同一段字节流进行解码，不同实现或版本的 Protobuf 解码得到的结果不一定完全相同（bytes 层面）。只能保证相同版本相同实现的 Protobuf 对同一段字节流多次解码得到的结果相同。
- 假设有一条消息`foo`，以下关系可能不成立：

```
foo.SerializeAsString() == foo.SerializeAsString()Hash(foo.SerializeAsString()) == Hash(foo.SerializeAsString())CRC(foo.SerializeAsString()) == CRC(foo.SerializeAsString())FingerPrint(foo.SerializeAsString()) == FingerPrint(foo.SerializeAsString())
```

- 假设有两条逻辑上相等消息，但是序列化之后的内容（bytes 层面）不相同，部分可能的原因有：
- - 其中一条消息可能使用了较老版本的 protobuf，不能处理某些类型的字段，设为 unknwon。+ 使用了不同语言实现的 Protobuf，并且以不同的顺序编码字段。+ 消息中的字段使用了不稳定的算法进行序列化。+ 某条消息中有 bytes 类型的字段，用于储存另一条消息使用 Protobuf 序列化的结果，而这个 bytes 使用了不同的 Protobuf 进行序列化。+ 使用了新版本的 Protobuf，序列化实现不同。+ 消息字段顺序不同。

**References**

https://datatracker.ietf.org/doc/html/rfc4648
https://developers.google.com/protocol-buffers/docs/encodinghttps://developers.google.com/protocol-buffers/docs/proto3https://stackoverflow.com/questions/3538021/why-do-we-use-base64

原文作者：SG4YK，腾讯 PCG 后台开发工程师

原文链接：https://mp.weixin.qq.com/s/enDUynhZ1Pnzg_4xEjR21A