# 【NO.183】STL 红黑树源码分析

最近我们项目上因为STL的使用出了很多问题，尤其是对map的使用。map的底层与set一样同为STL红黑树，所以抽空将STL红黑树的源代码再学习学习。

## 1. STL红黑树的节点

STL的红黑树也属于红黑树（红黑树是一种自平衡的二叉查找树），所以具备普通红黑树的5个特性：

1. **每个节点或是红色的或是黑色的**
2. **根节点是黑色的**
3. **每个叶节点（NULL）是黑色的**
4. **如果一个节点是红色的，则它的两个孩子节点都是黑色的（从每个叶子到根的所有路径上不能有两个连续的红色节点）**
5. **对每个节点，从该节点到其所有后代叶节点的简单路径上，均包含相同数目的黑色节点**

STL的红黑树和普通的红黑树相比较，多了以下两点特性：

1. **header不仅指向root，也指向红黑树的最左节点，以便用常数时间实现begin()，并且也指向红黑树的最右边节点，以便 set相关泛型算法（如set_union等等）可以有线性时间表现**
2. **当要删除的节点有两个子节点时，其后继节点连接到其位置，而不是被复制，因此，唯一使无效的迭代器是引用已删除节点的迭代器**

可以看出，**相比于普通的红黑树多了一个header节点，并且为红色**。如下图所示，普通的红黑树是以100节点开始的，而STL的红黑树以header节点开始。迭代器的begin指向红黑树根节点，也就是header节点的父亲，而end指向header节点。

![img](https://pic1.zhimg.com/80/v2-147ee77160c2d36dc9006c5faff7313c_720w.webp)

**注：此处的begin()和end()并不是指内置函数，而是指迭代器。begin()指向最左节点，图中标注的有错**

STL红黑树的数据存储类型为链式存储，因此存储单元为节点。因此，我们先看一下节点的定义。

如下所示为红黑树节点基类_Rb_tree_node_base的源代码，可以看出定义了颜色标记_Rb_tree_color。基类中分别声明了_M_left、_M_right、_M_parent三个指针成员同时声明了一个颜色标记常量_M_color成员。

```text
// 颜色标记
enum _Rb_tree_color { _S_red = false, _S_black = true };

// 基类，用来定义一个节点的属性
struct _Rb_tree_node_base
{
    // typedef重命名
    typedef _Rb_tree_node_base* _Base_ptr;
    typedef const _Rb_tree_node_base* _Const_Base_ptr;

    // 节点的属性，颜色、指向父节点的指针、指向左右孩子节点的指针
    _Rb_tree_color  _M_color;  // 颜色
    _Base_ptr       _M_parent; // 指向父亲
    _Base_ptr       _M_left;   // 指向左孩子
    _Base_ptr       _M_right;  // 指向右孩子
    
    // 求节点__x的最小节点
    static _Base_ptr
    _S_minimum(_Base_ptr __x) _GLIBCXX_NOEXCEPT
    {
        while (__x->_M_left != 0) __x = __x->_M_left;
        return __x;
    }

    // 求节点__x的最大节点
    static _Base_ptr
    _S_maximum(_Base_ptr __x) _GLIBCXX_NOEXCEPT
    {
        while (__x->_M_right != 0) __x = __x->_M_right;
        return __x;
    }
};
```

同时，节点基类里面还定义了两个重要的函数，分别是获取红黑树中__x节点的最小节点的函数_S_minimum与最大节点的函数_S_maximum。

由于STL红黑树是有序的，所以需要一个比较函数对象，该对象类型定义如下

```text
template<typename _Key_compare>
  struct _Rb_tree_key_compare
  {
      // 成员变量就是我们提供给红黑树的仿函数对象
      _Key_compare    _M_key_compare;
  }
```

当我们初始化一个空的红黑树对象的时候，也会初始化一个节点出来，该节点就是header节点，header节点的定义如下：

```text
// Helper type to manage
// default initialization of node count and header.
struct _Rb_tree_header
{
    _Rb_tree_node_base  _M_header; // 不存储数据
    size_t    _M_node_count; // 用来统计红黑树中节点的数量
    _Rb_tree_header()
    {
        _M_header._M_color = _S_red; // header节点一定是红色的
        _M_reset();
    }
    void _M_reset()
    {
        _M_header._M_parent = 0;
        _M_header._M_left = &_M_header;
        _M_header._M_right = &_M_header;
        _M_node_count = 0;
    }
}
```

红黑树节点_Rb_tree_node继承自红黑树基类_Rb_tree_node_base。可以计算一下一个红黑树节点的大小为：**4个指针+一个枚举+_Val**。

```text
template<typename _Val>
struct _Rb_tree_node : public _Rb_tree_node_base
{
    // 定义了一个节点类型的指针
    typedef _Rb_tree_node<_Value>* _Link_type;
    // 节点数据域，即节点中存储的值
    _Val _M_value_field;
};
```

## **2. STL红黑树**

STL红黑树的源代码如下所示，是一个模板类，_Key是key的类型，_Val是value的类型，_KeyOfValue是<key, value>对的类型，_Campare是比较函数对象类型，_Alloc是空间配置器的类型，默认为标准的allocator分配器。

包含了一个_Rb_tree_impl类型的成员变量_M_impl，对红黑树进行初始化操作与内存管理操作。_Rb_tree_impl继承了header节点_Rb_tree_header。

STL红黑树包含两种迭代器分别为_Rb_tree_iterator<value_type>和std::reverse_iterator<iterator>，说明STL红黑树支持rbegin和rend操作。

下面我们对rb-tree的源代码逐步进行解析。

```text
template<typename _Key, typename _Val, typename _KeyOfValue,
           typename _Compare, typename _Alloc = allocator<_Val> >
class _Rb_tree
{
public:
    typedef _Key                  key_type;
    typedef _Val                  value_type;
    typedef value_type*           pointer;
    typedef const value_type*     const_pointer;
    typedef value_type&           reference;
    typedef const value_type&     const_reference;
    typedef size_t                size_type;
    typedef ptrdiff_t             difference_type;
    typedef _Alloc                allocator_type;
    // 将节点类型_Rb_tree_node<_Val>与空间配置器_Alloc绑定
    // 空间配置器具体为配置节点空间的_Node_allocator
    typedef typename __gnu_cxx::__alloc_traits<_Alloc>::template
    rebind<_Rb_tree_node<_Val> >::other _Node_allocator;
    // 空间配置器萃取机为具体配置器_Node_allocator的萃取机
    typedef __gnu_cxx::__alloc_traits<_Node_allocator> _Alloc_traits;
protected:
    typedef _Rb_tree_node_base*     _Base_ptr;  // 节点基类指针
    typedef _Rb_tree_node<_Val>*    _Link_type; // 节点指针
   
protected:
    // 包含一个_Rb_tree_impl对象，因此一个红黑树对象至少包含
    // 一个header节点对象
    // 一个具体配置器_Node_allocator的对象
    _Rb_tree_impl<_Compare> _M_impl;

    // _Rb_tree_impl的定义如下，包含一个header节点
    template<typename _Key_compare,
       bool _Is_pod_comparator = __is_pod(_Key_compare)>
    struct _Rb_tree_impl : public _Node_allocator
                         , public _Rb_tree_key_compare<_Key_compare>
                         , public _Rb_tree_header
    {
        typedef _Rb_tree_key_compare<_Key_compare> _Base_key_compare;
        // 构造函数如下
        _Rb_tree_impl() : _Node_allocator()
        {}
        
        _Rb_tree_impl(const _Rb_tree_impl& __x)
        : _Node_allocator(_Alloc_traits::_S_select_on_copy(__x))
        , _Base_key_compare(__x._M_key_compare)
        {}
    };
public:
    typedef _Rb_tree_iterator<value_type>       iterator;
    typedef std::reverse_iterator<iterator>     reverse_iterator;
    
    // 重要的函数
    // _M_get_Node_allocator用来获取红黑树节点对象的空间配置器对象
    _Node_allocator& _M_get_Node_allocator()
    { return this->_M_impl; }
    // 为节点对象分配空间，返回值为内存地址，每次分配一个空间
    _Link_type _M_get_node()
    { return _Alloc_traits::allocate(_M_get_Node_allocator(), 1); }
    // 空间释放函数，将节点指针传进去
    void _M_put_node(_Link_type __p)
    { _Alloc_traits::deallocate(_M_get_Node_allocator(), __p, 1); }
    // 构造节点的底层函数
    template<typename... _Args>
    void _M_construct_node(_Link_type __node, _Args&&... __args)
    {
        ::new(__node) _Rb_tree_node<_Val>;
        _Alloc_traits::construct(_M_get_Node_allocator(),
                     __node->_M_valptr(),
                     std::forward<_Args>(__args)...);
    }
    // 构造节点对象，调用_M_construct_node
    template<typename... _Args>
    _Link_type _M_create_node(_Args&&... __args)
    {
        _Link_type __tmp = _M_get_node();
        _M_construct_node(__tmp, std::forward<_Args>(__args)...);
        return __tmp;
    }
    // 节点对象的析构函数
    void _M_destroy_node(_Link_type __p)
    {
        // 释放内容
        _Alloc_traits::destroy(_M_get_Node_allocator(), __p->_M_valptr());
        // 释放内存
        __p->~_Rb_tree_node<_Val>();
    }
    // 红黑树的构造函数
    _Rb_tree() = default;
    _Rb_tree(const _Compare& __comp
            , const allocator_type& __a = allocator_type())
    : _M_impl(__comp, _Node_allocator(__a)) { }
    // 红黑树的析构函数
    ~_Rb_tree()
    { _M_erase(_M_begin()); }
};
```

还有如下操作函数：

```text
// 图中100 节点
 _Base_ptr&
_M_root() _GLIBCXX_NOEXCEPT
{ return this->_M_impl._M_header._M_parent; }

// 图中most left标记
_Base_ptr&
_M_leftmost() _GLIBCXX_NOEXCEPT
{ return this->_M_impl._M_header._M_left; }

// 图中most right标记
_Base_ptr&
_M_rightmost() _GLIBCXX_NOEXCEPT
{ return this->_M_impl._M_header._M_right; }

// 图中begin()标记
_Link_type
_M_begin() _GLIBCXX_NOEXCEPT
{ return static_cast<_Link_type>(this->_M_impl._M_header._M_parent); }

// 图中end()标记
_Link_type
_M_end() _GLIBCXX_NOEXCEPT
{ return reinterpret_cast<_Link_type>(&this->_M_impl._M_header);
```

![img](https://pic1.zhimg.com/80/v2-d4cb199bf2556854440e963148e981f8_720w.webp)

## **3. STL红黑树迭代器**

STL红黑树也自定义了一个迭代器_Rb_tree_iterator。

```text
template<typename _Tp>
struct _Rb_tree_iterator
{
    typedef _Tp  value_type;
    typedef _Tp& reference;
    typedef _Tp* pointer;
    typedef bidirectional_iterator_tag iterator_category;
    typedef ptrdiff_t                  difference_type;

    typedef _Rb_tree_iterator<_Tp>        _Self;
    typedef _Rb_tree_node_base::_Base_ptr _Base_ptr;
    typedef _Rb_tree_node<_Tp>*           _Link_type;
    // 迭代器是一个指向红黑树节点的指针
    _Base_ptr _M_node;
};
```

重载了*操作符和->操作符来获取节点中存储的数据

```text
reference
operator*() const _GLIBCXX_NOEXCEPT
{ return *static_cast<_Link_type>(_M_node)->_M_valptr(); }

pointer
operator->() const _GLIBCXX_NOEXCEPT
{ return static_cast<_Link_type> (_M_node)->_M_valptr(); }
```

迭代器类重载的++操作符，自增（自底而上）的时候调用。本质是调用函数_Rb_tree_increment

```text
_Self&
operator++() _GLIBCXX_NOEXCEPT
{
    _M_node = _Rb_tree_increment(_M_node);
    return *this;
}
```

而_Rb_tree_increment函数的底层是local_Rb_tree_increment，源代码如下，可以看出采用的是前序遍历。

```text
static _Rb_tree_node_base *
local_Rb_tree_increment( _Rb_tree_node_base* __x ) throw ()
{
  /* 存在右子树,那么下一个节点为右子树的最小节点 */
  if ( __x->_M_right != 0 )
  {
    __x = __x->_M_right;
    while ( __x->_M_left != 0 )
      __x = __x->_M_left;
  }else  {
/* 不存在右子树,那么分为两种情况：自底往上搜索,当前节点为父节点的左孩子的时候,父节点就是后继节点；*/
/* 第二种情况:_x为header节点了,那么_x就是最后的后继节点. 简言之_x为最小节点且往上回溯,一直为父节点的右孩子,直到_x变为父节点,_y为其右孩子 */
    _Rb_tree_node_base *__y = __x->_M_parent;
    while ( __x == __y->_M_right )
    {
      __x  = __y;
      __y  = __y->_M_parent;
    }
    if ( __x->_M_right != __y )
      __x = __y;
  }
  return (__x);
}
```

同理，也重载了--操作符，调用_Rb_tree_decrement函数

```text
_Self&
operator--() _GLIBCXX_NOEXCEPT
{
  _M_node = _Rb_tree_decrement(_M_node);
    return *this;
}
```

同理，local_Rb_tree_decrement的源代码如下

```text
static _Rb_tree_node_base *
local_Rb_tree_decrement( _Rb_tree_node_base * __x )
throw ()
{
/* header节点 */
  if ( __x->_M_color ==
       _S_red
       && __x
       ->_M_parent->_M_parent == __x )
    __x = __x->_M_right;
  /* 左节点不为空,返回左子树中最大的节点 */
  else if ( __x->_M_left != 0 )
  {
    _Rb_tree_node_base *__y = __x->_M_left;
    while ( __y->_M_right != 0 )
      __y = __y->_M_right;
    __x = __y;
  }else  {
/* 自底向上找到当前节点为其父节点的右孩子,那么父节点就是前驱节点 */
    _Rb_tree_node_base *__y = __x->_M_parent;
    while ( __x == __y->_M_left )
    {
      __x  = __y;
      __y  = __y->_M_parent;
    }
    __x = __y;
  }
  return
    (__x);
}
```

也重载了==与!=操作符，用来判断节点指针是否相等。

```text
bool
operator==(const _Self& __x) const _GLIBCXX_NOEXCEPT
{ return _M_node == __x._M_node; }

bool
operator!=(const _Self& __x) const _GLIBCXX_NOEXCEPT
{ return _M_node != __x._M_node; }
```

黑节点的统计函数：

```text
unsigned int
_Rb_tree_black_count(const _Rb_tree_node_base *__node,
                     const _Rb_tree_node_base *__root) throw() {
    if (__node == 0)
        return 0;
    unsigned int __sum = 0;
    do {
        if (__node->_M_color == _S_black)
            ++__sum;
        if (__node == __root)
            break;
        __node = __node->_M_parent;
    } while (1);
    return __sum;
}
```

## **4.STL红黑树的自平衡**

我们在一开始就说明了STL红黑树也要满足普通红黑树的规则，这些规则才保证了红黑树的自平衡。红黑树从根到叶子的最长路径不会超过最短路径的2倍。

当插入节点或者删除节点的时候，红黑树的规则可能会被打破，这时候就需要做出一些调整，调整的方法有**变色**和**旋转**两种。旋转又包含左旋转和右旋转两种形式。

**变色**

为了重新符合红黑树的规则，尝试把红色节点变为黑色，或者把黑色节点变为红色。

如下所示为变色的场景：

![img](https://pic4.zhimg.com/80/v2-b896a65ec8035fe978432c7d8ca72fc7_720w.webp)

![img](https://pic1.zhimg.com/80/v2-6d0216a179b17163b6672022fecd1198_720w.webp)

![img](https://pic3.zhimg.com/80/v2-724d12ea47fbee58584b21c72445218a_720w.webp)

STL的源代码如下所示：

```text
_Rb_tree_node_base *const __y = __xpp->_M_right;    // 得到叔叔节点
if (__y && __y->_M_color == _S_red)     // case1: 叔叔节点存在，且为红色
{
    /**
     * 解决办法是：颜色翻转，父亲与叔叔的颜色都变为黑色,祖父节点变为红色,然后当前节点设为祖父，依次网上来判断是否破坏了红黑树性质
    */
    __x->_M_parent->_M_color = _S_black;    // 将其父节点改为黑色
    __y->_M_color = _S_black;               // 将其叔叔节点改为黑色
    __xpp->_M_color = _S_red;               // 将其祖父节点改为红色
    __x = __xpp;                            // 修改_x,往上回溯
} else {        // 无叔叔或者叔叔为黑色
    if (__x == __x->_M_parent->_M_right) {          // 当前节点为父亲节点的右孩子
        __x = __x->_M_parent;
        local_Rb_tree_rotate_left(__x, __root);     // 以父节点进行左旋转
    }
    // 旋转之后,节点x变成其父节点的左孩子
    __x->_M_parent->_M_color = _S_black;            // 将其父亲节点改为黑色
    __xpp->_M_color = _S_red;                       // 将其祖父节点改为红色
    local_Rb_tree_rotate_right(__xpp, __root);      // 以祖父节点右旋转
}
```

![img](https://pic2.zhimg.com/80/v2-a9e6d721c41a15941ef47283027292e9_720w.webp)

![img](https://pic3.zhimg.com/80/v2-7ba4aaad6b38b6858028f309a3c846d6_720w.webp)

![img](https://pic4.zhimg.com/80/v2-1812c199e724145d244ce484bdbaf26b_720w.webp)

代码如下：

```text
_Rb_tree_node_base *const __y = __xpp->_M_left; // 保存叔叔节点
if (__y && __y->_M_color == _S_red) {       // 叔叔节点存在且为红色
    __x->_M_parent->_M_color = _S_black;    // 父亲节点改为黑色
    __y->_M_color = _S_black;               // 祖父节点改为红色
    __xpp->_M_color = _S_red;
    __x = __xpp;
} else {        // 若无叔叔节点或者其叔叔节点为黑色
    if (__x == __x->_M_parent->_M_left) {   // 当前节点为父亲节点的左孩子
        __x = __x->_M_parent;
        local_Rb_tree_rotate_right(__x, __root);    // 以父节点右旋转
    }
    __x->_M_parent->_M_color = _S_black;        // 父节点置为黑色
    __xpp->_M_color = _S_red;                   // 祖父节点置为红色
    local_Rb_tree_rotate_left(__xpp, __root);   // 左旋转
}
```

**左旋转**

逆时针旋转红黑树的两个节点，使得父节点被自己的右孩子取代，而自己成为自己的左孩子。

```text
/**
* 当前节点的左旋转过程
* 将该节点的右节点设置为它的父节点，该节点将变成刚才右节点的左孩子
* 该节点的右节点的左孩子变成该节点的右孩子
* @param _x
*/
//    _x                      _y
//  /   \     左旋转         /  \
// T1   _y   --------->   _x    T3
//     / \              /   \
//    T2 T3            T1   T2
void leftRotate(Node *_x) {
    // step1 处理_x的右孩子
    // 右节点变为_x节点的父亲节点,先保存一下右节点
    Node *_y = _x->right;
    // T2变为node的右节点
    _x->right = _y->left;
    if (NULL != _y->left)
        _y->left->parent = _x;

    // step2 处理_y与父亲节点关系
    _y->parent = _x->parent;      // 原来_x的父亲变为_y的父亲
    // 说明原来_x为root节点,此时需要将_y设为新root节点
    // 或者判断NULL == _y->parent
    if (_x == root)
        root = _y;
    else if (_x == _x->parent->left)    // 原_x的父节点的左孩子连接新节点_y
        _x->parent->left = _y;
    else // 原_x的父节点的右孩子连接新节点_y
        _x->parent->right = _y;

    // step3 处理_x与_y关系
    _y->left = _x;      // _y的左孩子为_x
    _x->parent = _y;    // _x的父亲是_y
}
```

**右旋转**

顺时针旋转红黑树的两个节点，使得父节点被自己的左孩子取代，而自己成为自己的右孩子。

```text
//        _x                      _y
//      /   \     右旋转         /  \
//     _y    T2 ------------->  T0  _x
//    /  \                         /  \
//   T0  T1                       T1  T2
void rightRotate(Node *_x) {
    // step1 处理_x的左孩子
    // 左节点变为_x节点的父亲节点,先保存一下左节点
    Node *_y = _x->left;
    // T1变为_x的左孩子
    _x->left = _y->right;
    if (NULL != _y->right)
        _y->right->parent = _x;

    // step2 处理_y与父节点之间的关系
    // 或者判断_x->parent==NULL
    if (_x == root)
        root = _y;
    else if (_x == _x->parent->right)
        _x->parent->right = _y;
    else
        _x->parent->left = _y;

    // step3 处理_x与_y关系
    _y->right = _x;     // _y的右孩子为_x
    _x->parent = _y;    // _x的父亲是_y
}
```

STL红黑树会在插入和删除节点的时候进行自平衡操作，底层调用的是_Rb_tree_insert_and_rebalance函数，该函数的源码分析如下：

```text
void
_Rb_tree_insert_and_rebalance(const bool __insert_left,
                              _Rb_tree_node_base *__x,
                              _Rb_tree_node_base *__p,
                              _Rb_tree_node_base &__header) throw() {
    _Rb_tree_node_base * &__root = __header._M_parent;

    // Initialize fields in new node to insert.
    __x->_M_parent = __p;
    __x->_M_left = 0;
    __x->_M_right = 0;
    __x->_M_color = _S_red;

    // 处理__header部分
    // Insert.
    // Make new node child of parent and maintain root, leftmost and
    // rightmost nodes.
    // N.B. First node is always inserted left.
    if (__insert_left) {
        __p->_M_left = __x; // also makes leftmost = __x when __p == &__header

        if (__p == &__header) {
            __header._M_parent = __x;
            __header._M_right = __x;
        } else if (__p == __header._M_left)
            __header._M_left = __x; // maintain leftmost pointing to min node
    } else {
        __p->_M_right = __x;

        if (__p == __header._M_right)
            __header._M_right = __x; // maintain rightmost pointing to max node
    }

 // Rebalance.
    while (__x != __root
           && __x->_M_parent->_M_color == _S_red)   // 若新插入节点不是为RB-Tree的根节点，且其父节点color属性也是红色,即违反了性质4.
    {
        _Rb_tree_node_base *const __xpp = __x->_M_parent->_M_parent;        // 祖父节点

        if (__x->_M_parent == __xpp->_M_left)   // 父亲是祖父节点的左孩子
        {
            _Rb_tree_node_base *const __y = __xpp->_M_right;    // 得到叔叔节点
            if (__y && __y->_M_color == _S_red)     // case1: 叔叔节点存在，且为红色
            {
                /**
                 * 解决办法是：颜色翻转，父亲与叔叔的颜色都变为黑色,祖父节点变为红色,然后当前节点设为祖父，依次网上来判断是否破坏了红黑树性质
                 */
                __x->_M_parent->_M_color = _S_black;    // 将其父节点改为黑色
                __y->_M_color = _S_black;               // 将其叔叔节点改为黑色
                __xpp->_M_color = _S_red;               // 将其祖父节点改为红色
                __x = __xpp;                            // 修改_x,往上回溯
            } else {        // 无叔叔或者叔叔为黑色
                if (__x == __x->_M_parent->_M_right) {          // 当前节点为父亲节点的右孩子
                    __x = __x->_M_parent;
                    local_Rb_tree_rotate_left(__x, __root);     // 以父节点进行左旋转
                }
                // 旋转之后,节点x变成其父节点的左孩子
                __x->_M_parent->_M_color = _S_black;            // 将其父亲节点改为黑色
                __xpp->_M_color = _S_red;                       // 将其祖父节点改为红色
                local_Rb_tree_rotate_right(__xpp, __root);      // 以祖父节点右旋转
            }
        } else {        // 父亲是祖父节点的右孩子
            _Rb_tree_node_base *const __y = __xpp->_M_left; // 保存叔叔节点
            if (__y && __y->_M_color == _S_red) {       // 叔叔节点存在且为红色
                __x->_M_parent->_M_color = _S_black;    // 父亲节点改为黑色
                __y->_M_color = _S_black;               // 祖父节点改为红色
                __xpp->_M_color = _S_red;
                __x = __xpp;
            } else {        // 若无叔叔节点或者其叔叔节点为黑色
                if (__x == __x->_M_parent->_M_left) {   // 当前节点为父亲节点的左孩子
                    __x = __x->_M_parent;
                    local_Rb_tree_rotate_right(__x, __root);    // 以父节点右旋转
                }
                __x->_M_parent->_M_color = _S_black;        // 父节点置为黑色
                __xpp->_M_color = _S_red;                   // 祖父节点置为红色
                local_Rb_tree_rotate_left(__xpp, __root);   // 左旋转
            }
        }
    }
    //若新插入节点为根节点,则违反性质2
    //只需将其重新赋值为黑色即可
    __root->_M_color = _S_black;
}
```

map和set其实都是对rb-tree的包装，操作函数最终都是调用rb-tree提供的操作。map和set的增删改查，其实就是rb-tree的增删改查，所以掌握rb-tree的底层原理即可。这篇文章的内容已经足够多了，下一篇我会仔细分析一下rb-tree的增删改查，如果不理解这其中的细节，会导致我们在使用map和set时踩到很多陷阱。

原文地址：https://zhuanlan.zhihu.com/p/557734821

作者：linux