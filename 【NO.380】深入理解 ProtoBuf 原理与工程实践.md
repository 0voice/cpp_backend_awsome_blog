# 【NO.380】深入理解 ProtoBuf 原理与工程实践

ProtoBuf 作为一种跨平台、语言无关、可扩展的序列化结构数据的方法，已广泛应用于网络数据交换及存储。随着互联网的发展，系统的异构性会愈发突出，跨语言的需求会愈加明显，同时 gRPC 也大有取代Restful之势，而 ProtoBuf 作为g RPC 跨语言、高性能的法宝，我们技术人有必要

深入理解 ProtoBuf 原理，为以后的技术更新和选型打下基础。

我将过去的学习过程以及实践经验，总结成系列文章，与大家一起探讨学习，希望大家能有所收获，当然其中有不正确的地方也欢迎大家批评指正。

本系列文章主要包含：

1. 深入理解 ProtoBuf 原理与工程实践（概述）
2. 深入理解 ProtoBuf 原理与工程实践（编码）
3. 深入理解 ProtoBuf 原理与工程实践（序列化）
4. 深入理解 ProtoBuf 原理与工程实践（工程实践）

## 1.什么是ProtoBuf

ProtoBuf(Protocol Buffers)是一种跨平台、语言无关、可扩展的序列化结构数据的方法，可用于网络数据交换及存储。

在序列化结构化数据的机制中，ProtoBuf是灵活、高效、自动化的，相对常见的XML、JSON，描述同样的信息，ProtoBuf序列化后数据量更小、序列化/反序列化速度更快、更简单。

一旦定义了要处理的数据的数据结构之后，就可以利用ProtoBuf的代码生成工具生成相关的代码。只需使用 Protobuf 对数据结构进行一次描述，即可利用各种不同语言(proto3支持C++, Java, Python, Go, Ruby, Objective-C, C#)或从各种不同流中对你的结构化数据轻松读写。

## 2.为什么是 ProtoBuf

大家可能会觉得 Google 发明 ProtoBuf 是为了解决序列化速度的，其实真实的原因并不是这样的。

ProtoBuf最先开始是 Google用来解决索引服务器 request/response 协议的。没有ProtoBuf之前，Google 已经存在了一种 request/response 格式，用于手动处理 request/response 的编解码。它也能支持多版本协议，不过代码不够优雅：

```
if (protocolVersion=1) {
    doSomething();
} else if (protocolVersion=2) {
    doOtherThing();
} ...
```

如果是非常明确的格式化协议，会使新协议变得非常复杂。因为开发人员必须确保请求发起者与处理请求的实际服务器之间的所有服务器都能理解新协议，然后才能切换开关以开始使用新协议。

这也就是每个服务器开发人员都遇到过的低版本兼容、新旧协议兼容相关的问题。

为了解决这些问题，于是ProtoBuf就诞生了。

ProtoBuf 最初被寄予以下 2 个特点：

- **更容易引入新的字段**，并且不需要检查数据的中间服务器可以简单地解析并传递数据，而无需了解所有字段。
- **数据格式更加具有自我描述性**，可以用各种语言来处理(C++, Java 等各种语言)。

这个版本的 ProtoBuf 仍需要自己手写解析的代码。

不过随着系统慢慢发展，演进，ProtoBuf具有了更多的特性：

- 自动生成的序列化和反序列化代码避免了手动解析的需要。（官方提供自动生成代码工具，各个语言平台的基本都有）。
- 除了用于数据交换之外，ProtoBuf被用作持久化数据的便捷自描述格式。

ProtoBuf 现在是 Google 用于数据交换和存储的通用语言。谷歌代码树中定义了 48162 种不同的消息类型，包括 12183 个 .proto 文件。它们既用于 RPC 系统，也用于在各种存储系统中持久存储数据。

ProtoBuf 诞生之初是为了解决服务器端新旧协议(高低版本)兼容性问题，名字也很体贴，“协议缓冲区”。只不过后期慢慢发展成用于传输数据。

Protocol Buffers 命名由来：

> Why the name “Protocol Buffers”?
> The name originates from the early days of the format, before we had the protocol buffer compiler to generate classes for us. At the time, there was a class called ProtocolBuffer which actually acted as a buffer for an individual method. Users would add tag/value pairs to this buffer individually by calling methods like AddValue(tag, value). The raw bytes were stored in a buffer which could then be written out once the message had been constructed.
> Since that time, the “buffers” part of the name has lost its meaning, but it is still the name we use. Today, people usually use the term “protocol message” to refer to a message in an abstract sense, “protocol buffer” to refer to a serialized copy of a message, and “protocol message object” to refer to an in-memory object representing the parsed message.

## 3.如何使用 ProtoBuf

### 3.1. ProtoBuf 协议的工作流程

![img](https://pic4.zhimg.com/80/v2-6bd90593f1d04ef0c50be6f503ae4f0b_720w.webp)

可以看到，对于序列化协议来说，使用方只需要关注业务对象本身，即 idl 定义，序列化和反序列化的代码只需要通过工具生成即可。

### 3.2 .ProtoBuf 消息定义

ProtoBuf 的消息是在idl文件(.proto)中描述的。下面是本次样例中使用到的消息描述符customer.proto：

```
syntax = "proto3";
package domain;
option java_package = "com.protobuf.generated.domain";
option java_outer_classname = "CustomerProtos";
message Customers {
    repeated Customer customer = 1;
}
message Customer {
    int32 id = 1;
    string firstName = 2;
    string lastName = 3;
    enum EmailType {
        PRIVATE = 0;
        PROFESSIONAL = 1;
    }
    message EmailAddress {
        string email = 1;
        EmailType type = 2;
    }
    repeated EmailAddress email = 5;
}
```

上面的消息比较简单，Customers包含多个Customer，Customer包含一个id字段，一个firstName字段，一个lastName字段以及一个email的集合。

除了这些定义外，文件顶部还有三行可帮助代码生成器：

1. 首先，syntax = “proto3”用于idl语法版本，目前有两个版本proto2和proto3，两个版本语法不兼容，如果不指定，默认语法是proto2。由于proto3比proto2支持的语言更多，语法更简洁，本文使用的是proto3。
2. 其次有一个package domain;定义。此配置用于嵌套生成的类/对象。
3. 有一个option java_package定义。生成器还使用此配置来嵌套生成的源。此处的区别在于这仅适用于Java。在使用Java创建代码和使用JavaScript创建代码时，使用了两种配置来使生成器的行为有所不同。也就是说，Java类是在包com.protobuf.generated.domain下创建的，而JavaScript对象是在包domain下创建的。

ProtoBuf 提供了更多选项和数据类型，本文不做详细介绍，感兴趣可以参考这里。

### 3.3 .代码生成

首先安装 ProtoBuf 编译器 protoc，这里有详细的安装教程，安装完成后，可以使用以下命令生成 Java 源代码：

```
protoc --java_out=./src/main/java ./src/main/idl/customer.proto
```

从项目的根路径执行该命令，并添加了两个参数：java_out，定义./src/main/java/为Java代码的输出目录；而./src/main/idl/customer.proto是.proto文件所在目录。

生成的代码非常复杂，但是幸运的是它的用法却非常简单。

```
CustomerProtos.Customer.EmailAddress email = CustomerProtos.Customer.EmailAddress.newBuilder()
                .setType(CustomerProtos.Customer.EmailType.PROFESSIONAL)
                .setEmail("crichardson@email.com").build();
        CustomerProtos.Customer customer = CustomerProtos.Customer.newBuilder()
                .setId(1)
                .setFirstName("Lee")
                .setLastName("Richardson")
                .addEmail(email)
                .build();
        // 序列化
        byte[] binaryInfo = customer.toByteArray();
        System.out.println(bytes_String16(binaryInfo));
        System.out.println(customer.toByteArray().length);
        // 反序列化
        CustomerProtos.Customer anotherCustomer = CustomerProtos.Customer.parseFrom(binaryInfo);
        System.out.println(anotherCustomer.toString());
```

### 3.4. 性能数据

我们简单地以Customers为模型，分别构造、选取小对象、普通对象、大对象进行性能对比。

序列化耗时以及序列化后数据大小对比

![img](https://pic2.zhimg.com/80/v2-6b3bca2954b3717861c50ada727cd9c9_720w.webp)

反序列化耗时

![img](https://pic2.zhimg.com/80/v2-97d226a1fce4f599183f115aab7db929_720w.webp)

## 4.总结

上面介绍了 ProtoBuf 是什么、产生的背景、基本用法，我们再总结下。

**优点：**

> \1. 效率高
> 从序列化后的数据体积角度，与XML、JSON这类文本协议相比，ProtoBuf通过T-(L)-V（TAG-LENGTH-VALUE）方式编码，不需要”, {, }, :等分隔符来结构化信息，同时在编码层面使用varint压缩，所以描述同样的信息，ProtoBuf序列化后的体积要小很多，在网络中传输消耗的网络流量更少，进而对于网络资源紧张、性能要求非常高的场景，ProtoBuf协议是不错的选择。

```
// 我们简单做个对比// 要描述如下JSON数据{"id":1,"firstName":"Chris","lastName":"Richardson","email":[{"type":"PROFESSIONAL","email":"crichardson@email.com"}]}# 使用JSON序列化后的数据大小为118byte7b226964223a312c2266697273744e616d65223a224368726973222c226c6173744e616d65223a2252696368617264736f6e222c22656d61696c223a5b7b22747970<br>65223a2250524f46455353494f4e414c222c22656d61696c223a226372696368617264736f6e40656d61696c2e636f6d227d5d7d# 而使用ProtoBuf序列化后的数据大小为48byte0801120543687269731a0a52696368617264736f6e2a190a156372696368617264736f6e40656d61696c2e636f6d1001
```

> 从序列化/反序列化速度角度，与XML、JSON相比，ProtoBuf序列化/反序列化的速度更快，比XML要快20-100倍。
> \2. 支持跨平台、多语言
> ProtoBuf是平台无关的，无论是Android与PC，还是C#与Java都可以利用ProtoBuf进行无障碍通讯。
> proto3支持C++, Java, Python, Go, Ruby, Objective-C, C#。
> \3. 扩展性、兼容性好
> 具有向后兼容的特性，更新数据结构以后，老版本依旧可以兼容，这也是ProtoBuf诞生之初被寄予解决的问题。因为编译器对不识别的新增字段会跳过不处理。
> \4. 使用简单
> ProtoBuf 提供了一套编译工具，可以自动生成序列化、反序列化的样板代码，这样开发者只要关注业务数据idl，简化了编码解码工作以及多语言交互的复杂度。

**缺点：**

> 可读性差，缺乏自描述
> XML，JSON是自描述的，而ProtoBuf则不是。
> ProtoBuf是二进制协议，编码后的数据可读性差，如果没有idl文件，就无法理解二进制数据流，对调试不友好。

不过Charles已经支持ProtoBuf协议，导入数据的描述文件即可。

此外，由于没有idl文件无法解析二进制数据流，ProtoBuf在一定程度上可以保护数据，提升核心数据被破解的门槛，降低核心数据被盗爬的风险。

原文链接：https://zhuanlan.zhihu.com/p/407874981

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)