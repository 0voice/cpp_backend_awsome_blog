# 【NO.545】磁盘存储链式的 B 树与 B+树

## 1.B树的介绍

**在讲B树之前我们先讨论一下内存与磁盘有什么区别?**
对于这个问题很多朋友或多或少可以说点出来
可能很多朋友答的第一点就是
1.内存快磁盘慢
2.断电以后数据消失,磁盘持久存储

![在这里插入图片描述](https://img-blog.csdnimg.cn/7dbb7a85fc114a48951ce80de5fd0311.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


在这个简单的过程当中就构建了三级存储,寄存器,内存,磁盘
寄存器是少量的,速度很快,然后就是内存再是磁盘;



这个概念捋清楚的话我们再聊下:我们在访问内存的时候接下来有几条指令要解释一下为什么讲B树需要从这个地方开始讲,不然肯定不清楚为什么B树与B+树只适合使用在磁盘上面,不时候用在内存里面.
mov eax [0008h] 这条指令就是指内存中的一个值放到寄存器里面去,cpu可以通过指令访问内存中的所有值,我们可以通过cpu指令访问内存中的任意位置;如果当内存中间没有数据的时候,没有命中它就会产生一个缺页异常,比如我们访问磁盘,访问一个文件这个文件很大,这时候访问数据,它首先会把文件加入到内存里面,如果在访问的过程中间发现内存里头这块没有,这时候就会引起一个缺页的动作,这个缺页的动作会发送什么呢?会去磁盘 当中进行寻址,寻址的过程当中请注意,我们在这里面看到的大家可能就是3个框,给大家的直观感受就是三个框,很难感受到磁盘比内存为什么会慢?慢在哪里?没有一个直观的感受,内存是我们可以通过cpu的指令汇编代码是可以随意访问的,访问中间任何一个快,任何一个地址都是可以的,都是磁盘是访问不了的.
**磁盘是通过什么访问的?**

每一次访问磁盘的时候,磁盘就相当于那个风扇,它不断的再旋转,转到固定的位置,磁盘是有很多个面组成然后这个过程当中我们怎么从这个面上面拿到数据呢?
它就通过这个磁头顶住这个磁盘,然后寻址找到这个固定的磁道再拿到对应的数据,可能一看有点绕口,可以这样理解,这里就相当于是一个碗橱柜,你家里那个放碗筷的橱柜,一大堆的盘子,在这个盘子中间存储了一些固定数据,磁盘它慢,慢在哪呢?就慢在这个寻址的过程中间,寻址的时候很慢,它转到那个固定的位置,拿到确定的数据它很慢,顶住磁盘磁头,找到磁盘对应的扇区拿到数据.
做操作系统的时候,你会指定扇区,这个地方就是存储了那个boot,操作系统启动的时候也是从这个扇区里面刚刚开始启动拿到数据,然后在开始加载内存中间执行指令,这不是今天这篇文章重点,重点是磁盘慢,慢在寻址.
**这个慢和这个内存访问有什么区别呢?**
接着就可以参考我上篇红黑树的文章,红黑树是怎样的呢?可以理解为二叉树,对二叉树举一个例子,1024个节点大家构想一下1024个节点什么意思,层高有10层,10层是通过什么方式找到的?
先拿到根节点,通过一次寻址?寻址什么意思?**就是这个指针指向下一个位置,就是一次寻址**;1024节点最差的情况需要寻址10次,才能拿到对应我们想要的数据,请注意这种方式是没有问题的.同样的红黑树我们把它拿到磁盘中去运行的时候就会发现出现一个问题,它不是说做不了,大家就结合这个情况,红黑树来做我们把它放到磁盘里面,同样也是需要寻址,同样是1024个节点我们需要进行10次磁盘的寻址,拿这个10次的磁盘寻址什么意思呢?就相当于这个磁盘在磁道上面转十次,就是我们需要找十次,请注意这十次还是在查找的过程中,读一个数据我们需要寻址10次写一个数据也要寻址10次,你会发现访问磁盘的速度会大大的降低
**有没有一种方法减少寻址次数**,**也就是说降低树层高,寻址次数越少越好,就一俩次越快越好**
在这个前景下面就有了我们的多叉树,多叉树是为了在同样的节点下面,比如6叉8叉可能两次寻址就可以搞定,这个多叉树就是降低层高减少寻址速度,那么在多叉树中间就有了一种树叫B树,那么在多叉树中间也有很多种的树,
什么3叉树4 叉树,5,btree也是多叉树,在定义的时候呢没有定义叉树?
叉是根据你有多少个子节点多少分叉是由应用程序,是由我们自己写代码自己去定义的,它就比什么三叉树,4叉树,乃至N叉树多了更多的变动性,**我们在讲这个多叉树的时候很多时候就等同与在讲这个B树**,B树的定义上面就没有定义多少分叉,
**首先解释B树为什么常用解释完B树后再来解释B+树,**
按照这样一个情况比如说每个节点4K一个页为单位寻址操作内存的时候最小单位是4k,
4Gd的存储我们需要多少次寻址
B树开1024叉,只要寻址两次,
多叉树B树是怎么定义的?
**所有的叶子节点都在同一层决定了平衡**

我相信很多朋友都在解释这样一个问题?这个定义怎么来的
**我想问一下是先有了B树还是先有了定义?**
先有了树的定义慢慢去版本迭代约束我们所做的东西

![在这里插入图片描述](https://img-blog.csdnimg.cn/ca9295cce3934ddc9a4a2756f74129d5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



是先有了结果?
**逻辑矛盾点:思考这个定义再去思考这个树,**
**先把树理解了后再去思考定义,一个约定俗成是OK的**
26个字母B树

![在这里插入图片描述](https://img-blog.csdnimg.cn/723b56d4f538402fa877871216fbac7d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_18,color_FFFFFF,t_70,g_se,x_16)



根节点最少有两个子树这个是成立的,第二个所有叶子节点同一层,
还有这个概念里面跟大家解释一下
**这个阶最大的一个树,也就是这个节点里面最大能存储多少个字母**呢?
5个,最大的这个5,跟我们定义的这个M阶5+1等于M阶,**这个5形容的是有多少key**

**key最多是5,那么它的子树就有6个,M阶代表的就是这个子树里面最大有多少个节点,5个人排队中间有6个空隙**
能够结合代码看出B树;

同样的字母B树的组织方式是N多样的

B树用代码怎么定义?
先定义一个节点出来,这个节点里面包含多少K多少子树然后再包含其它信息比如M阶怎么定义?有多少K值?是否是叶子节点?



![在这里插入图片描述](https://img-blog.csdnimg.cn/aaa463a21e8e46af8c99fdbfee5cb25a.png)



![在这里插入图片描述](https://img-blog.csdnimg.cn/61d7be27c76b4316a22b5207f6f6938c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



一个扇区就是节点,几次寻址找到扇区;
**柔性数组是用在内存里面的,内存已经分配好了我们不需要知道它的大小的时候,就可以直接通过一个标签来访问,**
那么关于B树我们怎么从一个节点变成有n多个节点呢?

总共有多少个单元?
一百万快怎么组织

26个字母是不是只有确定的方式呢?

先分裂再添加

![在这里插入图片描述](https://img-blog.csdnimg.cn/f99852eb6c3840899b4ea1366b24c573.png)


M选偶数k奇数
先讲分裂再创建

![在这里插入图片描述](https://img-blog.csdnimg.cn/0dc99611cdab4d349ff3082c2262cf26.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_17,color_FFFFFF,t_70,g_se,x_16)



如果只有一个节点怎么分裂第i个子节点分裂不出来,它有父节点第几个孩子,

分裂完怎么插入对应位置插入节点分俩步骤:
找到对应位置,把对应节点放进去,
二叉排序树一样,对比节点k找到合适的位置

怎么实现?插的这个节点
随便看一颗B树,这个节点新插的k值最终插入叶子节点上面,不可能插入内节点,
所有节点插都是插到叶子节点上面,那么我们代码怎么实现呢?
比如插入aa比u大,然后分裂
还要引入函数

这个x就是我们要插的未满的节点,不是一个意思,
不是我们我们插的这个函数他就是插的一个为满的节点
x就是相当于这个过程是递归的方法就是a不是叶子节点找到一个然后找插进去,
if(是叶子节点插,不是叶子节点就往下递归
对比的是k值其实找到是它子树位置找到第一个比他小的
它的子树是他最贴近的位置

如果发现子树是满的先进行分裂
找到合适位置然后判断子树是不是满的,如果是满的就先进行分裂
是满的然后进行分裂

和到一起三步骤
分裂完再继续找对应子树是不是满,分裂完再插入进去,分裂完就不要判断了不是满的了

插入一个自己好吧好吧满,会满是再下一次插入节点时再分裂,刚满不分裂,

删除节点,比较复杂
找规律

删除先找的时候正好等于最小的值,遍历到正好等于下限,这时候出现借位动作,

发生了几个步骤:
1.什么时候需要借位,删除的时候
这么几种情况4
1.对比俩颗子树,idx-1,inx+1,如果idx等于最小数量a.借位借位又分两种情况,什么时候向前借位什么时候向后借位,下判断前面前面没有再判断后面
B.合并如果idx
2.如果子树大于,就顺其自然,借不来就合并,
为什么合并要把C合并,

删除的时候同样第一步递归找到对应的子树
迭部进行刚刚的动作找子树的过程就是递归的过程,

删除的这个节点删除的这个值也会是在叶子节点上面

## 2.B树的组成



![在这里插入图片描述](https://img-blog.csdnimg.cn/de1ed1f7e45a482fb545917469690904.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



## 3.B树的实现

```cpp
#include <stdio.h>



#include <stdlib.h>



#include <string.h>



 



#include <assert.h>



//M/2的值方便后面直接判断合并分裂



#define DEGREE        3



typedef int KEY_VALUE;



 



typedef struct _btree_node {



    KEY_VALUE *keys;//数组M-1



    struct _btree_node **childrens;



    int num;//没有占满



    int leaf;//隐其实它是个状态为什么不是bool,情况一样的



} btree_node;



 



typedef struct _btree {



    btree_node *root;



    int t;



} btree;



 



btree_node *btree_create_node(int t, int leaf) {



 



    btree_node *node = (btree_node*)calloc(1, sizeof(btree_node));



    if (node == NULL) assert(0);



 



    node->leaf = leaf;



    node->keys = (KEY_VALUE*)calloc(1, (2*t-1)*sizeof(KEY_VALUE));



    node->childrens = (btree_node**)calloc(1, (2*t) * sizeof(btree_node));



    node->num = 0;



 



    return node;



}



 



void btree_destroy_node(btree_node *node) {



 



    assert(node);



 



    free(node->childrens);



    free(node->keys);



    free(node);



    



}



 



 



void btree_create(btree *T, int t) {



    T->t = t;



    



    btree_node *x = btree_create_node(t, 1);



    T->root = x;



    



}



 



//传那个节点好?1还是2没有父节点要上去,第i个分裂



void btree_split_child(btree *T, btree_node *x, int i) {



    int t = T->t;



 



    btree_node *y = x->childrens[i];



    btree_node *z = btree_create_node(t, y->leaf);



 



    z->num = t - 1;



 



    int j = 0;



    for (j = 0;j < t-1;j ++) {



        z->keys[j] = y->keys[j+t];



    }



    if (y->leaf == 0) {



        for (j = 0;j < t;j ++) {



            z->childrens[j] = y->childrens[j+t];



        }



    }



 



    y->num = t - 1;



    for (j = x->num;j >= i+1;j --) {



        x->childrens[j+1] = x->childrens[j];



    }



 



    x->childrens[i+1] = z;



 



    for (j = x->num-1;j >= i;j --) {



        x->keys[j+1] = x->keys[j];



    }



    x->keys[i] = y->keys[t-1];



    x->num += 1;



    



}



 



void btree_insert_nonfull(btree *T, btree_node *x, KEY_VALUE k) {



 



    int i = x->num - 1;



 



    if (x->leaf == 1) {



        



        while (i >= 0 && x->keys[i] > k) {



            x->keys[i+1] = x->keys[i];



            i --;



        }



        x->keys[i+1] = k;



        x->num += 1;



        



    } else {



        while (i >= 0 && x->keys[i] > k) i --;



 



        if (x->childrens[i+1]->num == (2*(T->t))-1) {



            btree_split_child(T, x, i+1);



            if (k > x->keys[i+1]) i++;



        }



 



        btree_insert_nonfull(T, x->childrens[i+1], k);



    }



}



 



void btree_insert(btree *T, KEY_VALUE key) {



    //int t = T->t;



 



    btree_node *r = T->root;



    if (r->num == 2 * T->t - 1) {



        



        btree_node *node = btree_create_node(T->t, 0);



        T->root = node;



 



        node->childrens[0] = r;



 



        btree_split_child(T, node, 0);



 



        int i = 0;



        if (node->keys[0] < key) i++;



        btree_insert_nonfull(T, node->childrens[i], key);



        



    } else {



        btree_insert_nonfull(T, r, key);



    }



}



 



void btree_traverse(btree_node *x) {



    int i = 0;



 



    for (i = 0;i < x->num;i ++) {



        if (x->leaf == 0) 



            btree_traverse(x->childrens[i]);



        printf("%C ", x->keys[i]);



    }



 



    if (x->leaf == 0) btree_traverse(x->childrens[i]);



}



 



void btree_print(btree *T, btree_node *node, int layer)



{



    btree_node* p = node;



    int i;



    if(p){



        printf("\nlayer = %d keynum = %d is_leaf = %d\n", layer, p->num, p->leaf);



        for(i = 0; i < node->num; i++)



            printf("%c ", p->keys[i]);



        printf("\n");



#if 0



        printf("%p\n", p);



        for(i = 0; i <= 2 * T->t; i++)



            printf("%p ", p->childrens[i]);



        printf("\n");



#endif



        layer++;



        for(i = 0; i <= p->num; i++)



            if(p->childrens[i])



                btree_print(T, p->childrens[i], layer);



    }



    else printf("the tree is empty\n");



}



 



 



int btree_bin_search(btree_node *node, int low, int high, KEY_VALUE key) {



    int mid;



    if (low > high || low < 0 || high < 0) {



        return -1;



    }



 



    while (low <= high) {



        mid = (low + high) / 2;



        if (key > node->keys[mid]) {



            low = mid + 1;



        } else {



            high = mid - 1;



        }



    }



 



    return low;



}



 



 



//{child[idx], key[idx], child[idx+1]} 



void btree_merge(btree *T, btree_node *node, int idx) {



 



    btree_node *left = node->childrens[idx];



    btree_node *right = node->childrens[idx+1];



 



    int i = 0;



 



    /////data merge



    left->keys[T->t-1] = node->keys[idx];



    for (i = 0;i < T->t-1;i ++) {



        left->keys[T->t+i] = right->keys[i];



    }



    if (!left->leaf) {



        for (i = 0;i < T->t;i ++) {



            left->childrens[T->t+i] = right->childrens[i];



        }



    }



    left->num += T->t;



 



    //destroy right



    btree_destroy_node(right);



 



    //node 



    for (i = idx+1;i < node->num;i ++) {



        node->keys[i-1] = node->keys[i];



        node->childrens[i] = node->childrens[i+1];



    }



    node->childrens[i+1] = NULL;



    node->num -= 1;



 



    if (node->num == 0) {



        T->root = left;



        btree_destroy_node(node);



    }



}



 



void btree_delete_key(btree *T, btree_node *node, KEY_VALUE key) {



 



    if (node == NULL) return ;



 



    int idx = 0, i;



 



    while (idx < node->num && key > node->keys[idx]) {



        idx ++;



    }



 



    if (idx < node->num && key == node->keys[idx]) {



 



        if (node->leaf) {



            



            for (i = idx;i < node->num-1;i ++) {



                node->keys[i] = node->keys[i+1];



            }



 



            node->keys[node->num - 1] = 0;



            node->num--;



            



            if (node->num == 0) { //root



                free(node);



                T->root = NULL;



            }



 



            return ;



        } else if (node->childrens[idx]->num >= T->t) {



 



            btree_node *left = node->childrens[idx];



            node->keys[idx] = left->keys[left->num - 1];



 



            btree_delete_key(T, left, left->keys[left->num - 1]);



            



        } else if (node->childrens[idx+1]->num >= T->t) {



 



            btree_node *right = node->childrens[idx+1];



            node->keys[idx] = right->keys[0];



 



            btree_delete_key(T, right, right->keys[0]);



            



        } else {



 



            btree_merge(T, node, idx);



            btree_delete_key(T, node->childrens[idx], key);



            



        }



        



    } else {



 



        btree_node *child = node->childrens[idx];



        if (child == NULL) {



            printf("Cannot del key = %d\n", key);



            return ;



        }



 



        if (child->num == T->t - 1) {



 



            btree_node *left = NULL;



            btree_node *right = NULL;



            if (idx - 1 >= 0)



                left = node->childrens[idx-1];



            if (idx + 1 <= node->num) 



                right = node->childrens[idx+1];



 



            if ((left && left->num >= T->t) ||



                (right && right->num >= T->t)) {



 



                int richR = 0;



                if (right) richR = 1;



                if (left && right) richR = (right->num > left->num) ? 1 : 0;



 



                if (right && right->num >= T->t && richR) { //borrow from next



                    child->keys[child->num] = node->keys[idx];



                    child->childrens[child->num+1] = right->childrens[0];



                    child->num ++;



 



                    node->keys[idx] = right->keys[0];



                    for (i = 0;i < right->num - 1;i ++) {



                        right->keys[i] = right->keys[i+1];



                        right->childrens[i] = right->childrens[i+1];



                    }



 



                    right->keys[right->num-1] = 0;



                    right->childrens[right->num-1] = right->childrens[right->num];



                    right->childrens[right->num] = NULL;



                    right->num --;



                    



                } else { //borrow from prev



 



                    for (i = child->num;i > 0;i --) {



                        child->keys[i] = child->keys[i-1];



                        child->childrens[i+1] = child->childrens[i];



                    }



 



                    child->childrens[1] = child->childrens[0];



                    child->childrens[0] = left->childrens[left->num];



                    child->keys[0] = node->keys[idx-1];



                    



                    child->num ++;



 



                    node->keys[idx-1] = left->keys[left->num-1];



                    left->keys[left->num-1] = 0;



                    left->childrens[left->num] = NULL;



                    left->num --;



                }



 



            } else if ((!left || (left->num == T->t - 1))



                && (!right || (right->num == T->t - 1))) {



 



                if (left && left->num == T->t - 1) {



                    btree_merge(T, node, idx-1);                    



                    child = left;



                } else if (right && right->num == T->t - 1) {



                    btree_merge(T, node, idx);



                }



            }



        }



 



        btree_delete_key(T, child, key);



    }



    



}



 



 



int btree_delete(btree *T, KEY_VALUE key) {



    if (!T->root) return -1;



 



    btree_delete_key(T, T->root, key);



    return 0;



}



 



 



int main() {



    btree T = {0};



 



    btree_create(&T, 3);



    srand(48);



 



    int i = 0;



    char key[26] = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";



    for (i = 0;i < 26;i ++) {



        //key[i] = rand() % 1000;



        printf("%c ", key[i]);



        btree_insert(&T, key[i]);



    }



 



    btree_print(&T, T.root, 0);



 



    for (i = 0;i < 26;i ++) {



        printf("\n---------------------------------\n");



        btree_delete(&T, key[25-i]);



        //btree_traverse(T.root);



        btree_print(&T, T.root, 0);



    }



    



}



 



 
```



![在这里插入图片描述](https://img-blog.csdnimg.cn/59f9926c949f4768ad2df850430d869c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)

原文作者：[我也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/604954819