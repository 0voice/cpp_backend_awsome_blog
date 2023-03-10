# 【NO.7】什么是B+树？（详细图解）

## 1.简介

### 1.1一个m阶的B+树具有如下几个特征：

1. 有k个子树的中间节点包含有k个元素（B树中是k-1个元素），每个元素不保存数据，只用来索引，所有数据都保存在叶子节点。
2. 所有的叶子结点中包含了全部元素的信息，及指向含这些元素记录的指针，且叶子结点本身依关键字的大小自小而大顺序链接。
3. 所有的中间节点元素都同时存在于子节点，在子节点元素中是最大（或最小）元素。

**首先来看看B+树的结构：**

![image-20221206223121478](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206223121478.png?lastModify=1670851010)

上面的这颗树中，得出结论：

- 根节点元素8是子节点2,5,8 的最大元素，也是叶子节点6,8 的最大元素；
- 根节点元素15是子节点11,15 的最大元素，也是叶子节点13,15 的最大元素；
- 根节点的最大元素也就是整个B+树的最大元素，以后无论插入删除多少元素，始终要保持最大的元素在根节点当中。
- 由于父节点的元素都出现在子节点中，因此所有的叶子节点包含了全部元素信息，并且每一个叶子节点都带有指向下一个节点的指针，形成了一个有序链表。



## 2.卫星数据

指的的是索引元素所指向的数据记录，比如数据库中的某一行，在B-树中，无论中间节点还是叶子节点都带有卫星数据。

**B-树中的卫星数据（Satellite Information）**：

![image-20221206223220726](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206223220726.png?lastModify=1670851010)

**B+树中的卫星数据（Satellite Information）：**

只有叶子节点带有卫星数据，其余中间节点仅仅是索引，没有任何的数据关联。

![image-20221206223307007](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206223307007.png?lastModify=1670851010)

需要补充的是，在数据库的聚集索引（Clustered Index）中，叶子节点直接包含卫星数据。在非聚集索引（NonClustered Index）中，叶子节点带有指向卫星数据的指针。

## 3.B+ 树的查询

下面分别通过单行查询和范围查询来做分析：

### 3.1 单行查询：

比如我们要查找的元素是3：

B+树会自顶向下逐层查找节点，最终找到匹配的叶子节点。

第一次磁盘IO：

![image-20221206223403066](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206223403066.png?lastModify=1670851010)

第二次磁盘IO：

![image-20221206223410321](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206223410321.png?lastModify=1670851010)

第三次磁盘IO：

![image-20221206223418343](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206223418343.png?lastModify=1670851010)

### 3.2 B+树查询与B-数据对比：

- B+树的中间节点没有卫星数据，所以同样大小的磁盘可以容纳更多的节点元素；
- 在数据量相同的情况下，B+树的结构比B-树更加“矮胖”，因此查询的IO次数也更少；
- B+树的查询必须最终查找到叶子节点，而B-树只要查找匹配的元素即可，无论匹配的元素处于中间节点还是叶子节点。
- B-树的查找性能并不稳定（最好情况只查询根节点，最坏情况是查找叶子节点）。而B+树的每一次查找都是稳定的。

**范围查询：** 比如我们要查询3到11的范围数据：

**B-树的范围查询过程：** 自顶向下，查找到范围的下限（3）：

![image-20221206224008082](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206224008082.png?lastModify=1670851010)

中序遍历到元素6：

![image-20221206224018546](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206224018546.png?lastModify=1670851010)

中序遍历到元素8：

![image-20221206224028847](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206224028847.png?lastModify=1670851010)

中序遍历到元素9：

![image-20221206224037348](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206224037348.png?lastModify=1670851010)

中序遍历到元素11，遍历结束：

![image-20221206224045156](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206224045156.png?lastModify=1670851010)

**B+树的范围查找过程：** 自顶向下，查找到范围的下限（3）：

![image-20221206224057364](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206224057364.png?lastModify=1670851010)

通过链表指针，遍历到元素6, 8：

![image-20221206224111959](file://C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20221206224111959.png?lastModify=1670851010)



## 4.总结

B+树比B-树的优势三个：

单一节点存储更多的元素，使得查询的IO次数更少。 所有查询都要查找到叶子节点，查询性能稳定。 所有叶子节点形成有序链表，便于范围查询。 ———————————————— 版权声明：本文为CSDN博主「初念初恋」的原创文章，遵循CC 4.0 BY-SA版权协议，