# 【NO.43】红黑树的原理以及实现

**红黑树**

**红黑树基于二叉查找树的附加特性**

1. 节点是红色或黑色。
2. 根节点是黑色。
3. 每个叶子节点都是黑色的空节点（叶子结点指为空的叶子结点）。
4. 每个红色节点的两个子节点都是黑色的（从每个叶子到根的所有路径上不能有两个连续的红色节点）。
5. 从任意节点到其每个叶子的所有路径都包含相同数目的黑色节点。

## **1. 数据结构**

```text
class TreeNode{
    private Boolean color;
    private int val;
    private TreeNode left;
    private TreeNode right;
    private TreeNode parent;
    get,set...
}
class RBTree{
    public boolean add(int val){...}
    public boolean delete(int val){...}
    public void display(){...}
}
```

## **2. 左旋以及右旋**

### **2.1 左旋**

![img](https://pic2.zhimg.com/80/v2-ee7b35428f94dbbb1a0b77be93b9116d_720w.webp)

### **2.2 右旋**

![img](https://pic2.zhimg.com/80/v2-4822d2fb2cf7e522a4d19c8a72799511_720w.webp)

## **3. 插入**

- 新插入的节点（newNode）为红色。
- 按照二分查找树插入规则插入。
- **分情况讨论**（**以下情况基本都是为了保持上文所讲的的红黑树特性4和特5**）
  1、若newNode为根节点，则变为黑色，插入完毕，返回 true。
  2、若newNode父节点为黑色，则插入完毕，返回 true。
  3、如下图所示，若newNode父节点为红色，且叔叔节点存在且为红色，则父节点与叔叔节点变为黑色，祖父节点变为红色，newNode = 祖父节点。

![img](https://pic2.zhimg.com/80/v2-dbcbfb84ec49083414f92625b1d90bbd_720w.webp)


4、如下图所示，若newNode父节点为红色，叔叔节点不存在或为黑色，且newNode为父节点右孩子，父节点为祖父节点左孩子，则以父节点为轴左旋，进入情况6.

![img](https://pic1.zhimg.com/80/v2-3cba7f4e8bd4ca3a568f5450c961be54_720w.webp)

5、如下图所示，若newNode父节点为红色，叔叔节点不存在或为黑色，且newNode为父节点左孩子，父节点为祖父节点右孩子，则以父节点为轴右旋，进入情况7

![img](https://pic1.zhimg.com/80/v2-c111482e5099c1568ecef70e3de34db8_720w.webp)

6、如下图，此时以祖父节点为轴进行右旋，将祖父节点变为红色，newNode变为黑色。

![img](https://pic3.zhimg.com/80/v2-59fe561a9ce96ce46e04d5894216dc76_720w.webp)

7、如下图，此时以祖父节点为轴进行左旋，将祖父节点变为红色，newNode变为黑色。

![img](https://pic1.zhimg.com/80/v2-1030a853056be151c322ad68b31bc0f0_720w.webp)

## **4. 删除**

**分情况讨论**（和插入一样，以下情况基本都是为了保持上文所讲的的红黑树特性4和特5）

1. 如下图，如果待删除节点**B**有两个非空的孩子节点，转化成待删除节点只有右孩子（或没有孩子）的情况，习惯性选取待删除节点右子树最小节点**E**替换待删除节点（只是值替换，颜色不变），并将待删除节点变为E。

![img](https://pic3.zhimg.com/80/v2-ec68d96519294a04a60240be1bd32546_720w.webp)

1. 根据待删除节点和唯一子节点颜色，**分情况处理：**

2. 1. 自身O是红色，子节点N是黑色，直接删除。
   2. 自身O是黑色，子节点N是红色，直接删除**并**将子节点N变为黑色。
   3. 自身O是黑色，子节点N不存在(不存在即子节点为空黑色节点，也可以用来判断)**或**也是黑色，较为复杂，**先删除，再分情况讨论：**
      1、N是根节点，则不需要调整。
      2、如下图，N的父亲、兄弟、侄子都是黑色，则将兄弟变为红色，父亲视作N，进行递归处理。

![img](https://pic1.zhimg.com/80/v2-5a159ab516c5f352cdc39bd9e04fa238_720w.webp)

3、（**存在镜像**）N的兄弟节点是红色，且N为父亲节点左儿子，则以父亲节点为轴左旋（否则右旋），并将旋转后N的祖父节点变为黑色，N的父节点变为红色，进入情况4,5或6.

![img](https://pic3.zhimg.com/80/v2-8b2cff6a44151e7a702935f3729daf56_720w.webp)

4、N的父亲节点是红色，兄弟和侄子节点是黑色，父亲节点变为黑色，兄弟节点变为红色。

![img](https://pic3.zhimg.com/80/v2-8180d41932ed875f0e4ce984d316e43a_720w.webp)

 5、（**存在镜像**）N的父节点颜色随意，兄弟节点为父节点黑色右孩子，左侄子节点为红色，右侄子节点为黑色，以兄弟节点为轴进行右旋，将旋转后N的兄弟节点变为黑色，N的右侄子节点变为红色，进入情况6

![img](https://pic3.zhimg.com/80/v2-5a30f3fd2aac1134fa2e50031c31223a_720w.webp)

 6、（**存在镜像**）N的父节点随意，兄弟节点为父节点的黑色右儿子，右侄子节点为红色，以N的父节点为轴进行左旋，左旋后的N的祖父节点变为父节点颜色，父节点变黑，叔叔节点变黑。

![img](https://pic4.zhimg.com/80/v2-1cc5c141b027e59b1cd78362bef97c23_720w.webp)

## 5.**测试**

**原树（上右下左）**

![img](https://pic3.zhimg.com/80/v2-77246a8a5f27c41930382fc8c29a64c2_720w.webp)

**删除53**

![img](https://pic2.zhimg.com/80/v2-1f4bcf75365b8df10a0b5893c047b85d_720w.webp)

**删除23**

![img](https://pic1.zhimg.com/80/v2-79ad4c9fc33586bfe468113ea11affa0_720w.webp)

**删除54**

![img](https://pic2.zhimg.com/80/v2-270cd680a5c46cde236714a42f71e085_720w.webp)

**添加67**

![img](https://pic3.zhimg.com/80/v2-e906fa056465876568af351ce3f55c72_720w.webp)



原文链接：https://zhuanlan.zhihu.com/p/588861677

原文作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)