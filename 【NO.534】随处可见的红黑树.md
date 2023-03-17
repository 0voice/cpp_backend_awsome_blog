# 【NO.534】随处可见的红黑树

## 1.红黑树为什么常用

1.当做查找以key-value
通过key去查找value,查找性能比较快
比如通过socket去查找客户端id,还有内核内存怎么使用?每用malloc分配一个内存就加入一颗红黑树,free(ptr)释放的时候key-value去查找对应的块
2.通过中序遍历是顺序的
进程的调度与红黑树什么关系呢?有些进程由于需要满足某些条件而处于等待状态等待某一个时间在未来某个时刻会再次运行,就把这些等待的再未来某个时刻会再次运行的进程加入红黑树去管理,

![在这里插入图片描述](https://img-blog.csdnimg.cn/1acc0ebae6ca48a9959e66b304d79470.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)



## 2.那么红黑树怎么实现?

那就不得不先聊二叉排序树了,名义上是二叉树,但是二叉树有一种最坏的情况是链表,违背了快速查找,为了更好的解决这个问题就引入了红黑树



![在这里插入图片描述](https://img-blog.csdnimg.cn/d1d3387a17b0418fba0f0d2d5c35312d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



**二叉树的定义与实现**

```cpp
#include <stdio.h>



#include <stdlib.h>



#include <string.h>



 



#include <assert.h>



 



#if 0



 



typedef int KEY_VALUE;



 



struct bstree_node {



    KEY_VALUE data;



    struct bstree_node *left;



    struct bstree_node *right;



};



 



struct bstree {



    struct bstree_node *root;



};



 



struct bstree_node *bstree_create_node(KEY_VALUE key) {



    struct bstree_node *node = (struct bstree_node*)malloc(sizeof(struct bstree_node));



    if (node == NULL) {



        assert(0);



    }



    node->data = key;



    node->left = node->right = NULL;



 



    return node;



}



 



int bstree_insert(struct bstree *T, int key) {



 



    assert(T != NULL);



 



    if (T->root == NULL) {



        T->root = bstree_create_node(key);



        return 0;



    }



 



    struct bstree_node *node = T->root;



    struct bstree_node *tmp = T->root;



 



    while (node != NULL) {



        tmp = node;



        if (key < node->data) {



            node = node->left;



        } else {



            node = node->right;



        }



    }



 



    if (key < tmp->data) {



        tmp->left = bstree_create_node(key);



    } else {



        tmp->right = bstree_create_node(key);



    }



    



    return 0;



}



 



int bstree_traversal(struct bstree_node *node) {



    if (node == NULL) return 0;



    



    bstree_traversal(node->left);



    printf("%4d ", node->data);



    bstree_traversal(node->right);



}



 



 



#define ARRAY_LENGTH        20



int main() {



 



    int keyArray[ARRAY_LENGTH] = {24,25,13,35,23, 26,67,47,38,98, 20,13,17,49,12, 21,9,18,14,15};



 



    struct bstree T = {0};



    int i = 0;



    for (i = 0;i < ARRAY_LENGTH;i ++) {



        bstree_insert(&T, keyArray[i]);



    }



 



    bstree_traversal(T.root);



 



    printf("\n");



}



 



#else



 



typedef int KEY_VALUE;



 



 



#define BSTREE_ENTRY(name, type)     \



    struct name {                    \



        struct type *left;            \



        struct type *right;            \



    }



 



struct bstree_node {



    KEY_VALUE data;



    BSTREE_ENTRY(, bstree_node) bst;



};



 



struct bstree {



    struct bstree_node *root;



};



 



struct bstree_node *bstree_create_node(KEY_VALUE key) {



    struct bstree_node *node = (struct bstree_node*)malloc(sizeof(struct bstree_node));



    if (node == NULL) {



        assert(0);



    }



    node->data = key;



    node->bst.left = node->bst.right = NULL;



 



    return node;



}



 



int bstree_insert(struct bstree *T, int key) {



 



    assert(T != NULL);



 



    if (T->root == NULL) {



        T->root = bstree_create_node(key);



        return 0;



    }



 



    struct bstree_node *node = T->root;



    struct bstree_node *tmp = T->root;



 



    while (node != NULL) {



        tmp = node;



        if (key < node->data) {



            node = node->bst.left;



        } else {



            node = node->bst.right;



        }



    }



 



    if (key < tmp->data) {



        tmp->bst.left = bstree_create_node(key);



    } else {



        tmp->bst.right = bstree_create_node(key);



    }



    



    return 0;



}



 



int bstree_traversal(struct bstree_node *node) {



    if (node == NULL) return 0;



    



    bstree_traversal(node->bst.left);



    printf("%4d ", node->data);



    bstree_traversal(node->bst.right);



}



 



#define ARRAY_LENGTH        20



int main() {



 



    int keyArray[ARRAY_LENGTH] = {24,25,13,35,23, 26,67,47,38,98, 20,13,17,49,12, 21,9,18,14,15};



 



    struct bstree T = {0};



    int i = 0;



    for (i = 0;i < ARRAY_LENGTH;i ++) {



        bstree_insert(&T, keyArray[i]);



    }



 



    bstree_traversal(T.root);



 



    printf("\n");



}



 



 



#endif



 
```

业务与数据结构混合在一起,同样的这套树能不能在不同的业务,比如进程的各种状态,就是这棵树的左右子节点不一样
1.如何创建
void* value ps:没有问题,还是需要再次寻址找内存,往往是这个value不知道是什么的时候写它,不管你定义什么把值直接往里面放,考虑到进程可有不同状态树就按这个技巧可以落在不同的树上结果,宏有美观作用;
malloc 红黑树内存分配如何组织起来,看到这个malloc应该就能想到在虚拟内存堆上面分配一块内存把它加入操作系统的内核,红黑树就是在这个基础上的一个变种;

## 3.红黑树的定义



![在这里插入图片描述](https://img-blog.csdnimg.cn/df25b6b2795f4fee884a71e8df24e10c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



定义怎么来的?可以有数学归纳法去证明
与二叉树这里还有不同之处是定义多一个nil,所有的叶子节点都指向这个nil



![在这里插入图片描述](https://img-blog.csdnimg.cn/7757e58053bc4a189f535de7316ed07c.png)



```cpp
#include <stdio.h>



#include <stdlib.h>



#include <string.h>



 



#define RED                1



#define BLACK             2



 



typedef int KEY_TYPE;



 



typedef struct _rbtree_node {



    unsigned char color;



    struct _rbtree_node *right;



    struct _rbtree_node *left;



    struct _rbtree_node *parent;



    KEY_TYPE key;



    void *value;



} rbtree_node;



 



typedef struct _rbtree {



    rbtree_node *root;



    rbtree_node *nil;



} rbtree;



 



rbtree_node *rbtree_mini(rbtree *T, rbtree_node *x) {



    while (x->left != T->nil) {



        x = x->left;



    }



    return x;



}



 



rbtree_node *rbtree_maxi(rbtree *T, rbtree_node *x) {



    while (x->right != T->nil) {



        x = x->right;



    }



    return x;



}



 



rbtree_node *rbtree_successor(rbtree *T, rbtree_node *x) {



    rbtree_node *y = x->parent;



 



    if (x->right != T->nil) {



        return rbtree_mini(T, x->right);



    }



 



    while ((y != T->nil) && (x == y->right)) {



        x = y;



        y = y->parent;



    }



    return y;



}



 



 



void rbtree_left_rotate(rbtree *T, rbtree_node *x) {



 



    rbtree_node *y = x->right;  // x  --> y  ,  y --> x,   right --> left,  left --> right



 



    x->right = y->left; //1 1



    if (y->left != T->nil) { //1 2



        y->left->parent = x;



    }



 



    y->parent = x->parent; //1 3



    if (x->parent == T->nil) { //1 4



        T->root = y;



    } else if (x == x->parent->left) {



        x->parent->left = y;



    } else {



        x->parent->right = y;



    }



 



    y->left = x; //1 5



    x->parent = y; //1 6



}



 



 



void rbtree_right_rotate(rbtree *T, rbtree_node *y) {



 



    rbtree_node *x = y->left;



 



    y->left = x->right;



    if (x->right != T->nil) {



        x->right->parent = y;



    }



 



    x->parent = y->parent;



    if (y->parent == T->nil) {



        T->root = x;



    } else if (y == y->parent->right) {



        y->parent->right = x;



    } else {



        y->parent->left = x;



    }



 



    x->right = y;



    y->parent = x;



}



 



void rbtree_insert_fixup(rbtree *T, rbtree_node *z) {



 



    while (z->parent->color == RED) { //z ---> RED



        if (z->parent == z->parent->parent->left) {



            rbtree_node *y = z->parent->parent->right;



            if (y->color == RED) {



                z->parent->color = BLACK;



                y->color = BLACK;



                z->parent->parent->color = RED;



 



                z = z->parent->parent; //z --> RED



            } else {



 



                if (z == z->parent->right) {



                    z = z->parent;



                    rbtree_left_rotate(T, z);



                }



 



                z->parent->color = BLACK;



                z->parent->parent->color = RED;



                rbtree_right_rotate(T, z->parent->parent);



            }



        }else {



            rbtree_node *y = z->parent->parent->left;



            if (y->color == RED) {



                z->parent->color = BLACK;



                y->color = BLACK;



                z->parent->parent->color = RED;



 



                z = z->parent->parent; //z --> RED



            } else {



                if (z == z->parent->left) {



                    z = z->parent;



                    rbtree_right_rotate(T, z);



                }



 



                z->parent->color = BLACK;



                z->parent->parent->color = RED;



                rbtree_left_rotate(T, z->parent->parent);



            }



        }



        



    }



 



    T->root->color = BLACK;



}



 



 



void rbtree_insert(rbtree *T, rbtree_node *z) {



 



    rbtree_node *y = T->nil;



    rbtree_node *x = T->root;



 



    while (x != T->nil) {



        y = x;



        if (z->key < x->key) {



            x = x->left;



        } else if (z->key > x->key) {



            x = x->right;



        } else { //Exist



            return ;



        }



    }



 



    z->parent = y;



    if (y == T->nil) {



        T->root = z;



    } else if (z->key < y->key) {



        y->left = z;



    } else {



        y->right = z;



    }



 



    z->left = T->nil;



    z->right = T->nil;



    z->color = RED;



 



    rbtree_insert_fixup(T, z);



}



 



void rbtree_delete_fixup(rbtree *T, rbtree_node *x) {



 



    while ((x != T->root) && (x->color == BLACK)) {



        if (x == x->parent->left) {



 



            rbtree_node *w= x->parent->right;



            if (w->color == RED) {



                w->color = BLACK;



                x->parent->color = RED;



 



                rbtree_left_rotate(T, x->parent);



                w = x->parent->right;



            }



 



            if ((w->left->color == BLACK) && (w->right->color == BLACK)) {



                w->color = RED;



                x = x->parent;



            } else {



 



                if (w->right->color == BLACK) {



                    w->left->color = BLACK;



                    w->color = RED;



                    rbtree_right_rotate(T, w);



                    w = x->parent->right;



                }



 



                w->color = x->parent->color;



                x->parent->color = BLACK;



                w->right->color = BLACK;



                rbtree_left_rotate(T, x->parent);



 



                x = T->root;



            }



 



        } else {



 



            rbtree_node *w = x->parent->left;



            if (w->color == RED) {



                w->color = BLACK;



                x->parent->color = RED;



                rbtree_right_rotate(T, x->parent);



                w = x->parent->left;



            }



 



            if ((w->left->color == BLACK) && (w->right->color == BLACK)) {



                w->color = RED;



                x = x->parent;



            } else {



 



                if (w->left->color == BLACK) {



                    w->right->color = BLACK;



                    w->color = RED;



                    rbtree_left_rotate(T, w);



                    w = x->parent->left;



                }



 



                w->color = x->parent->color;



                x->parent->color = BLACK;



                w->left->color = BLACK;



                rbtree_right_rotate(T, x->parent);



 



                x = T->root;



            }



 



        }



    }



 



    x->color = BLACK;



}



 



rbtree_node *rbtree_delete(rbtree *T, rbtree_node *z) {



 



    rbtree_node *y = T->nil;



    rbtree_node *x = T->nil;



 



    if ((z->left == T->nil) || (z->right == T->nil)) {



        y = z;



    } else {



        y = rbtree_successor(T, z);



    }



 



    if (y->left != T->nil) {



        x = y->left;



    } else if (y->right != T->nil) {



        x = y->right;



    }



 



    x->parent = y->parent;



    if (y->parent == T->nil) {



        T->root = x;



    } else if (y == y->parent->left) {



        y->parent->left = x;



    } else {



        y->parent->right = x;



    }



 



    if (y != z) {



        z->key = y->key;



        z->value = y->value;



    }



 



    if (y->color == BLACK) {



        rbtree_delete_fixup(T, x);



    }



 



    return y;



}



 



rbtree_node *rbtree_search(rbtree *T, KEY_TYPE key) {



 



    rbtree_node *node = T->root;



    while (node != T->nil) {



        if (key < node->key) {



            node = node->left;



        } else if (key > node->key) {



            node = node->right;



        } else {



            return node;



        }    



    }



    return T->nil;



}



 



 



void rbtree_traversal(rbtree *T, rbtree_node *node) {



    if (node != T->nil) {



        rbtree_traversal(T, node->left);



        printf("key:%d, color:%d\n", node->key, node->color);



        rbtree_traversal(T, node->right);



    }



}



 



int main() {



 



    int keyArray[20] = {24,25,13,35,23, 26,67,47,38,98, 20,19,17,49,12, 21,9,18,14,15};



 



    rbtree *T = (rbtree *)malloc(sizeof(rbtree));



    if (T == NULL) {



        printf("malloc failed\n");



        return -1;



    }



    



    T->nil = (rbtree_node*)malloc(sizeof(rbtree_node));



    T->nil->color = BLACK;



    T->root = T->nil;



 



    rbtree_node *node = T->nil;



    int i = 0;



    for (i = 0;i < 20;i ++) {



        node = (rbtree_node*)malloc(sizeof(rbtree_node));



        node->key = keyArray[i];



        node->value = NULL;



 



        rbtree_insert(T, node);



        



    }



 



    rbtree_traversal(T, T->root);



    printf("----------------------------------------\n");



 



    for (i = 0;i < 20;i ++) {



 



        rbtree_node *node = rbtree_search(T, keyArray[i]);



        rbtree_node *cur = rbtree_delete(T, node);



        free(cur);



 



        rbtree_traversal(T, T->root);



        printf("----------------------------------------\n");



    }



    



 



    



}



 



 



 



 



 
```

## 4.红黑树节点旋转



![在这里插入图片描述](https://img-blog.csdnimg.cn/38d57bddc0954b7997f885de278ba3b3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


左旋与右旋,一个操作6根指针,成双成对的先x后y,从右下开始直接下层X



## 5.红黑树的添加问题

1.添加之前满足红黑树的性质
它就是一个合法的树,
**插入的这个节点是红色的好还是黑色的好?**
红色好,是因为红色更容易满足性质,
**什么时候需要调整?**
只有性质四**如果一个结点是红的，则它的两个儿子都是黑的**违背
违背当前节点是红色的,同时父节点是红色的,从而出现多种情况?

![在这里插入图片描述](https://img-blog.csdnimg.cn/b8757bce93cb4edfb9131144735273a9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


父结点是祖父结点的左子树的情况



1. 叔结点是红色的

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/da5771d41fd84080b8e4d41be9b67362.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)

2. 叔结点是黑色的，而且当前结点是右孩子

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/ede2f74bf1954ca780f6d6d3d487b51a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)

3. 叔结点是黑色的，而且当前结点是左孩子

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/fdaa6963d4ea4e54b7db0b1b98512b31.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)

```cpp
void rbtree_insert(rbtree *T, rbtree_node *z) {



 



    rbtree_node *y = T->nil;



    rbtree_node *x = T->root;



 



    while (x != T->nil) {



        y = x;



        if (z->key < x->key) {



            x = x->left;



        } else if (z->key > x->key) {



            x = x->right;



        } else { //Exist



            return ;



        }



    }



 



    z->parent = y;



    if (y == T->nil) {



        T->root = z;



    } else if (z->key < y->key) {



        y->left = z;



    } else {



        y->right = z;



    }



 



    z->left = T->nil;



    z->right = T->nil;



    z->color = RED;



 



    rbtree_insert_fixup(T, z);



}



 



void rbtree_delete_fixup(rbtree *T, rbtree_node *x) {



 



    while ((x != T->root) && (x->color == BLACK)) {



        if (x == x->parent->left) {



 



            rbtree_node *w= x->parent->right;



            if (w->color == RED) {



                w->color = BLACK;



                x->parent->color = RED;



 



                rbtree_left_rotate(T, x->parent);



                w = x->parent->right;



            }



 



            if ((w->left->color == BLACK) && (w->right->color == BLACK)) {



                w->color = RED;



                x = x->parent;



            } else {



 



                if (w->right->color == BLACK) {



                    w->left->color = BLACK;



                    w->color = RED;



                    rbtree_right_rotate(T, w);



                    w = x->parent->right;



                }



 



                w->color = x->parent->color;



                x->parent->color = BLACK;



                w->right->color = BLACK;



                rbtree_left_rotate(T, x->parent);



 



                x = T->root;



            }



 



        } else {



 



            rbtree_node *w = x->parent->left;



            if (w->color == RED) {



                w->color = BLACK;



                x->parent->color = RED;



                rbtree_right_rotate(T, x->parent);



                w = x->parent->left;



            }



 



            if ((w->left->color == BLACK) && (w->right->color == BLACK)) {



                w->color = RED;



                x = x->parent;



            } else {



 



                if (w->left->color == BLACK) {



                    w->right->color = BLACK;



                    w->color = RED;



                    rbtree_left_rotate(T, w);



                    w = x->parent->left;



                }



 



                w->color = x->parent->color;



                x->parent->color = BLACK;



                w->left->color = BLACK;



                rbtree_right_rotate(T, x->parent);



 



                x = T->root;



            }



 



        }



    }



 



    x->color = BLACK;



}
```

if 可以?还是while 肯定得是while直到root的根节点始终扣红色,层层迭代
z=z->parent->parent是关键,z一直是红色

原文作者：[我也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/604954753Linux