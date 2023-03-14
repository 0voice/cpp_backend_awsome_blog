# 【NO.233】编译原理初学者入门指南

### **1.引子**

最近的工作需要用表达式做一些参数的配置，然后发现大脑一片空白，在 Google 里试了几个关键词（起初搜了下“符号引擎”，发现根本不是我想要的）之后，明白过来自己应该是需要补一些编译原理的知识了。在掉了两晚上头发之后，决定整理一下自己的知识网络。

要解析的表达式大概长这个样子：

```
avg(teams[*].players.attributes[skill])*rules[latency].maxLatency
```

正则表达式是个办法，但不是最优解，除了很难通过一句正则解析整条语句外，以后扩展更多语法时，正则表达式的修改也分外麻烦。

作为非计算机科班出身的工程师，还是知道一点，任何领域发展出的课题都需要产、学、研三者的配合才能完成，“产” 指企业，“学” 指高等院校，“研” 指科研机构。

其实自打 C 语言从贝尔实验室走出来之后，“研” 究这一步就已经完成大半了，于是被下放到 “学” 校，有了《编译原理》这门课程，最后流入 “产” 业中大规模应用。

所以这篇文章主要从两方面给初学者（尤其是跟我一样非科班出身的 coder）一个指南：

- 在科学原理上，通俗的解释一些专有名词，厘清基本概念——编译原理这块的术语简直太多了，多到糊脸的那种；
- 在工程实践上，给到一个可行的实现方法，主要是面向 golang 的 `goyacc`，如果本文有幸被你搜索到，你肯定最想看这一部分（现网关于 goyacc 的中文资料太少了）。

### **2.理论原理**

以下内容均为个人理解，欢迎探讨，如有不精确之处，以教科书为准～

#### 2.1 计算机语言是怎么回事儿

编译器由词法分析、语法分析、语义检查再到中间表示输出和最后二进制生成的流程，这些已经可以作为前置知识，就不提了。

随手打开一个工程，我们就能发现形形色色的语言文件，比如 yaml 格式的服务配置文件、json 格式的工程配置文件、js 和 go 等源代码文件等。忽略掉他们繁杂的用途，按其表达能力，可以分为两种：

- DSL（Domain Specific Language）：特定领域语言，比如用来描述数据的 json、用来查询数据的 sql、标记型的 xml 和 html，都属于面向特定领域的专用语言，用在正确的领域上就是利器，用错地方就是自找麻烦（比如用 sql 来一段冒泡排序）；
- GPL（General Purpose Language）：通用用途语言，比如 C、JavaScript、Golang，这类语言是 **图灵完备** 的，你可以用一门 GPL 语言去设计和实现一种 DSL 语言。

歪个楼，这里有一个吊诡的事实，yaml 竟然是图灵完备的！甚至很多语言都需要特别使用 safe_load 来加载 yaml 文件，比如用 java 直接 load 这段 yaml，会执行一次 HTTP 请求。

```
!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL [\"http://localhost\"]]]]
```

不管是为特定领域而发明的各类 DSL，还是图灵完备的 GPL 语言，他们基本都符合 BNF（**巴科斯范式**）。

BNF 是一种 **上下文无关文法**，举个例子就是，人类的语言就是一种 上下文**有关**文法，我随时都可以讲一句 “以上说的都是废话”，戏弄一下读者阅读本文所花的时间（每当回忆起来，我都会坐在轮椅上大呼过瘾）。

关于 BNF 具体定义，这里摘抄一下维基百科，后面做详细解说：

> **BNF** 规定是[推导规则](https://zh.wikipedia.org/w/index.php?title=推导规则&action=edit&redlink=1)(产生式)的集合，写为：
>
> <符号> ::= <使用符号的表达式>
>
> 这里的 <符号> 是[非终结符](https://zh.wikipedia.org/wiki/非终结符)，而[表达式](https://zh.wikipedia.org/wiki/表达式)由一个符号序列，或用指示[选择](https://zh.wikipedia.org/wiki/选择)的[竖杠](https://zh.wikipedia.org/w/index.php?title=竖杠&action=edit&redlink=1) '|' 分隔的多个符号序列构成，每个符号序列整体都是左端的符号的一种可能的[替代](https://zh.wikipedia.org/w/index.php?title=替代&action=edit&redlink=1)。从未在左端出现的符号叫做[终结符](https://zh.wikipedia.org/wiki/终结符)。

暂且不用理解里面提到的 “终结符” 和 “非终结符”，在明白来龙去脉之前去查这些，说不定大脑会 stackoverflow。但是也别慌，所有术语和英文缩写都是纸老虎，其实他们都是很简单的概念，但是你需要一个合适的场景来理解它们起到的作用。

#### 2.2 学科交叉：自然语言理解

上节我们说到，计算机语言多数是符合 BNF 的上下文无关语言，从表达能力上分为 DSL 和 GPL 两类；而人类语言属于上下文有关语言，其实正是由于这一点，才给在 NLP（自然语言理解）领域带来了不少困难。

好，知道了这些英文缩写，再去读那些专业文章会简单得多。

这些其实都是在 **静态层面** 上对语言的描述，为了实际执行这些语言，就需要对其进行解析，还原出语言本身所描述的信息结构。这件事，在计算机领域的课程叫《编译原理》，在智能科学与技术的课程叫《自然语言理解》。

- 编译原理（一张图）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvat76PRwqySU55JKQXm8SCibEMlFTeuJ3QnaYteoVBdsknaZHriaCffRPuic96G82z9It9pzLZRfIFZ9A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)编译原理

- 自然语言理解（两张图）：

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvat76PRwqySU55JKQXm8SCibEEAnMCOFDeiceOKhib7CI7LZ19CZMAl6gvm2WOScgFsusb6thOhKibqZicw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvat76PRwqySU55JKQXm8SCibELQhLnkJaocT1mHh2ic9523I7ziaHOzOzH52Fpuuk8uJvOu6VzUU6mnBQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)NLP

不难看出，两者的流程惊人的相似：

- 都需要先进行 tokenize 处理，编译器做的是**词法分析**（常用工具搜 lexer），NLP 做的是 分词（最常见的是 jieba 分词）
- 词法分析的产物是有含义的 token，下面都需要进行**语法分析**（即 parser），NLP 里通常会做 向量化（最常见的是 word2vec 方法）
- 这两步完成后，编译器前端得到的产物是 AST（Abstract Syntax Tree，抽象语法树），NLP 得到的产物是一段话的向量化表示

两者的共同点止步于此，鉴于 NLP 技术仍在高速发展（而编译原理早就是老生常谈了），向量化得到的产物难以处理同义词，所以后面的步骤也局限于分析一句话的意图、和提取有效信息（利用这些可以做一个简陋版的 Siri）。最新（其实是两年前了）的进展是 BERT 模型和衍生出来的许多研究上下文关系的方法，现在的 NLP 技术已经可以做阅读理解问题了。

此外，DSL 和 GPL 的共同点也止步于此。要记得，DSL 是面向特定用途的语言，以 JSON 为例，得到 AST 就已经有完整的信息结构了，在面向对象语言里无非再多一步：利用反射将其映射到一个 class 的所有字段里；以 HTML 为例，得到 AST 就已经有完整的 DOM 树了，浏览器已经具备渲染出整个网页所需的大部分信息。

最后，对 GPL 语言来说，编译型语言目的是生成机器可执行的代码，解释型语言的目的是生成虚拟机认识的中间代码。这部分职责由编译器后端承担，现代编译器领域的最佳拍档就是 Clang + LLVM。

#### 2.3 别慌：英文缩写都是纸老虎

现在我们知道了事情的来龙去脉，也就明白了开头的需求属于哪种问题。对工程师来说，解决问题的第一步就是先知道你面对的是什么问题：使用编译原理的知识来解析开头的表达式，相当于定义一个简陋的 DSL 语言，并编写词法解析器和语法解析器（lexer & parser）来将其转换成 AST（抽象语法树），进而对其进行处理。

在进行工程实践之前，还有些术语不得不先行了解。

首先是前面提到的终结符和非终结符，重复一下上面解释 BNF 时举的抽象表达式：`<符号> ::= <使用符号的表达式>`。可以这样来理解：

- 由词法解析器生成的符号，也叫 token，是终结符。终结符是最小表义单位，无法继续进行拆解和解析
- 规则左侧定义的符号，是非终结符。非终结符需要进行语法解析，最终由终结符构成其表示形式

其次是 NFA 和 DFA，FA 表示 Finite Automata（有穷状态机），即根据不同的输入来转换内部状态，其内部状态是有限个数的。而 NFA 和 DFA 分别代表 有穷不确定状态机 和 有穷确定状态机。运用子集构造法可以将 NFA 转换为 DFA，让构造得到的 DFA 的每个状态对应于 NFA 的一个状态的集合。

词法分析器（lexer）生成终结符，而语法分析器（parser）则利用自顶向下或自底向上的方法，利用文法中定义的终结符和非终结符，将输入信息转换为 AST（抽象语法树）。也就是我们在此次需求中需要获得的东西。

### **3.工程实践**

我们的案例是使用 golang 来编写 lexer 和 parser。

在工程上，不同语言的实践方式是不一样的。你可以选择自己编写 lexer 和 parser，也可以选择通过定义 yacc 文件的方式让工具自动生成。在参考文献中会给出自己编写它们的相关文章，在 golang 的案例里，lexer 需要自己编写，而 parser 则由工具生成。

如果使用 Antlr 的话，可以将 lexer 和 parser 一同搞定，用得好的话，可以实现诸如像 JS 和 Swift 语言互相转换的特技。不在本文实践范围内。

#### 3.1 goyacc 的安装

Golang 1.8 之前自带 goyacc 工具，由于使用量太少，之后版本就需要手动安装了。

```
go get -u github.com/golang/tools/tree/master/cmd/goyacc
```

使用起来参数如下：![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvat76PRwqySU55JKQXm8SCibEYXWTuhPsIS4rKDYnwa5eG5bkS9vrKcMOrhjUtOAOThulPTuDJkuCpw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

然后我们需要搞定词法分析器和语法分析器。

#### 3.2 使用 goyacc 的思路

yacc 类工具的共同特点就是，通过编写 .y 格式的说明文件定义语法，然后使用 yacc 命令行工具生成对应语言的源代码。

所以尝起来就比较像 protobuf，proto 文件就像 .y 文件一样本身不可执行，需要用一些 protogen 工具来生成对应每种语言的源代码文件。

在 goyacc 中，lexer 本身相对简单，自己编写 go 代码实现就够了，parser 部分所需的文法约定，需要我们编写 .y 文件，也就需要了解 yacc 的文法约定。goyacc 会在生成好的 go 源代码中提供 `yyParse` 、`yyText` 、`yyLex` 等接口，最后我们自己编写调用 parser 的文件，就能把流程跑起来了。

我们的目的，就是给定如下示例输入，然后输出能代表 AST 的数据结构：

```
# 示例输入
avg(teams[*].maxPlayers) *flatten(rules[red].players.playerAttributes[exp])

# 示例输出
parsed obj: [map[avg:[map[teams:*] map[maxPlayers:]]] map[flatten:[map[rules:red] map[players:] map[playerAttributes:exp]] last_operator:*]]

[
    {
        "avg": [
            {
                "teams": "*"
            },
            {
                "maxPlayers": ""
            }
        ]
    },
    {
        "flatten": [
            {
                "rules": "red"
            },
            {
                "players": ""
            },
            {
                "playerAttributes": "exp"
            }
        ],
        "last_operator": "*"
    }
]
```

#### 3.3 词法分析器

lexer 我们选择自己用 golang 编写。lexer 的基本功能是通过一个 buffer reader 不断读取文本，然后告诉 goyacc 遇到的是什么符号。

Lex 函数的返回值类型（即词法分析器的实际产物）需要在后面的 yacc 文件的 token 部分定义。

为了与 goyacc 结合，我们需要定义和实现以下接口：

```
type Scanner struct {
 buf   *bufio.Reader
 data  interface{}
 err   error
 debug bool
}

func NewScanner(r io.Reader, debug bool) *Scanner {
 return &Scanner{
  buf:   bufio.NewReader(r),
  debug: debug,
 }
}

func (sc *Scanner) Error(s string) {
 fmt.Printf("syntax error: %s\n", s)
}

func (sc *Scanner) Reduced(rule, state int, lval *yySymType) bool {
 if sc.debug {
  fmt.Printf("rule: %v; state %v; lval: %v\n", rule, state, lval)
 }
 return false
}

func (s *Scanner) Lex(lval *yySymType) int {
 return s.lex(lval)
}
```

我们可以定义私有函数完成 lex 的实际工作。

#### 3.4 语法分析器

上节我们有说，yacc 文件最终会生成 go 源代码文件，里面包含了 `yyParse` 、`yyText` 、`yyLex` 等接口的具体实现。

而 yacc 只包含定义文法的语法，不含各类编程语言的语法，所以聪明的你肯定能猜到，yacc 文件中免不了会出现类似宏定义的东西，会直接嵌入各类编程语言的代码片段。

有了这个心理预期，我们看一下 yacc 文件的结构：

```
{%
嵌入代码
%}
文法定义
%%
文法规则
%%
嵌入代码 （golang代码，通常忽略此部分直接在写在代码头中）
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvat76PRwqySU55JKQXm8SCibEOicwmjnLfX5Qf9c8T1xZW3RJ5Jtogbez1qV5ZF1OicezSrGufEZHmicqg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其文法定义如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvat76PRwqySU55JKQXm8SCibEUdgAia8TTWjzFCA7s1HBfwL2aCKdodVDZWpJGcRaFbKw5ls5TwbkScQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们自己编写 yacc 实现 parser，最少需要知道的就是前面四种描述符。一开始我们只实现最简单的语法规则，后面自己就会逐渐了解更高级的文法规则了。

#### 3.5 参考工程

goyacc 的示例工程不多，不推荐用 yacc 实现计算器的例子，参考性比较差。

如下工程实现了用 golang 解析 JSON 数据，只需要补一个 go.mod 和 main 函数就能拿来边调试边参考着实现自己的需求了，十分推荐：https://github.com/sjjian/yacc-examples

```
module example.com/m

go 1.14

require github.com/pkg/errors v0.9.1
package main

import (
 "encoding/json"
 "fmt"
 "io/ioutil"

 "example.com/m/yacc_parseJson"
)

func check(e error) {
 if e != nil {
  panic(e)
 }
}

func main() {
 dat, err := ioutil.ReadFile("json.txt")
 check(err)
 fmt.Printf("raw str: %s\n", string(dat))
 jsonobj, err := yacc_parseJson.ParseJson(string(dat), true)
 fmt.Printf("parsed obj: %+v\n", jsonobj)
 jsonStr, _ := json.Marshal(jsonobj)
 fmt.Printf(string(jsonStr))
}
```

### **4.参考文献**

- [编译原理（基础篇）](https://www.cnblogs.com/antispam/p/4015116.html)
- [golang 实现自定义语言的基础](https://www.1thx.com/golang/189.html)
- [什么是 NFA 和 DFA](https://www.cnblogs.com/AndyEvans/p/10240790.html)
- [从 antlr 扯淡到一点点编译原理](https://awhisper.github.io/2016/11/18/从antlr到语法解析/)
- [How to Write a Parser in Go](https://about.sourcegraph.com/go/gophercon-2018-how-to-write-a-parser-in-go/)

原文作者：pixelcao，腾讯 IEG 后台开发工程师

原文链接：https://mp.weixin.qq.com/s/ZTxVG6KG-4vzbvclC_Q1LQ