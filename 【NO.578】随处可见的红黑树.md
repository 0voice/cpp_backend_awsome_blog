# 【NO.578】随处可见的红黑树



## 1.背景

我们知道二叉搜索树（BST），在极端情况下，会退化为单支树，查找效率从 O(log2n) 退化为 O(n)。主要原因就是BST不够平衡(左右子树高度差太大)。既然如此，那么我们就需要通过一定的算法，将不平衡树改变成平衡树。因此，AVL树就诞生了。AVL要求左右子树的高度差不能超过1，是严格平衡的二叉搜索树，为了维持这种严格平衡，每次插入和删除的时候都需要旋转操作。在频繁插入、删除的场景下，AVL的性能会大打折扣。红黑树通过牺牲严格的平衡性质，换取减少每次插入和删除的旋转操作。

## 2.概述

红黑树是一种自平衡的二叉搜索树。不论哪种搜索树，通过中序遍历，结果就是有序的。

红黑树五大性质：

每个节点是红色或者黑色
根节点是黑色的
叶子节点是黑的
如果一个节点是红的，那么该节点的两个儿子都是黑的
每个节点，到叶子所有路径上的黑色节点个数相同。

![在这里插入图片描述](https://img-blog.csdnimg.cn/233fb546a38948699560dbbbcf245dbc.png)

## 3.红黑树代码实现

旋转：红黑树在插入或删除节点时，上面的性质可能会被破坏， 旋转操作就是为了使变化后的红黑树继续满足上面的性质。旋转操作分为左旋和右旋，旋转的本质，实际上是修改若干指针的指向。

![在这里插入图片描述](https://img-blog.csdnimg.cn/cf3098c57d1741ebaaf9557fab0e1bef.png)

左旋代码实现：

```
void rbtree_left_rotate(rbtree *T, rbtree_node *x) {

	rbtree_node *y = x->right;  // 获取x的右子树，保存到y
	x->right = y->left; // x的右子树指向y的左子树；
	
	//如果y的左子树不是叶子结点，需要改变y的左子树的父节点到x
	if (y->left != T->nil) { 
		y->left->parent = x;
	}
	
	y->parent = x->parent; // y的父节点指向x的父节点；
	// 此时需要判断x，x如果是根节点，则x的父节点为NULL，则需要让根节点指向此时的y；
	// 如果不是，则需要判断x是它父节点的左子树还是右子树,
	// 如果是左子树，就让父节点的左子树指向y，如果是右子树就让父节点的左=右子树指向y。
	if (x->parent == T->nil) { 
		T->root = y;
	} else if (x == x->parent->left) {
		x->parent->left = y;
	} else {
		x->parent->right = y;
	}
	
	y->left = x; // y的左子树指向x
	x->parent = y; // x的父节点指向y

}
```


右旋代码实现：只需要将左旋代码中的 x和y互换位置，left换成right，right换成left。

```
void rbtree_right_rotate(rbtree *T, rbtree_node *y) {
	rbtree_node *x = y->left;
	y->left = x->right;
	if (x->right != T->nil)
	{
		x->right->parent = y;
	}
	x->parent = y->parent;
	if (y->parent == T->nil) 
	{
		T->root = x;
	}
	else if (y == y->parent->right) 
	{
		y->parent->right = x;
	}
	else 
	{
		y->parent->left = x;
	}
	x->right = y;
	y->parent = x;
}



```


插入：每次插入都会插入到红黑树的最底层。并且上色为红色，因为红色不会影响第五条性质。插入完再继续进行调整（这里主要看自己与父节点是否为红色，如果是红色，迭代向上进行调整，如果是黑色就不用调整）。循环的中止条件就是遍历到叶子结点。如果插入的节点已经存在，也就是相等。取决于业务场景。比如定时任务，定时时间相同，可以稍稍修改一丢丢值。

始终需要扣住的点：红黑树不论是在什么时候，都是一棵红黑树，也就是说满足红黑树的性质。听起来像是废话，需要慢慢体会！

### 3.1 插入节点实现

红黑树插入节点z代码实现（看代码的时候结合着图看，会比较好理解一点）：

思考：假设插入节点z为红色，其父节点也为红色，那么z的祖父节点一定是黑色的！（红黑树性质决定），z的叔父节点的颜色就不确定了，需要分情况讨论：

要插入节点的父节点是祖父节点的左子树
a) 叔父节点是红色的情况：

![在这里插入图片描述](https://img-blog.csdnimg.cn/53c711e086cd4f1990f486154c6d27b8.png)

b) 叔父节点是黑色的情况：
这种情况下，需要考虑要插入的节点是其父节点的左孩子还是右孩子。
b.1) 要插入的节点是其父节点的右孩子

![在这里插入图片描述](https://img-blog.csdnimg.cn/83873e0e79784f1bba4955120cecddf0.png)

b.2) 要插入的节点是其父节点的左孩子

![在这里插入图片描述](https://img-blog.csdnimg.cn/f3a9747c91e24e48b98dbd36b87bffd9.png)

要插入节点的父节点是祖父节点的右子树（对照左子树情况理解就好）

```
void rbtree_insert_fixup(rbtree *T, rbtree_node *z) 
{
	 while (z->parent->color == RED) // 只要当前z与其父节点的颜色都是红的就进行调整
	 { 
		if (z->parent == z->parent->parent->left)  // 要插入节点的父节点是祖父节点的左子树
		{  
			rbtree_node *y = z->parent->parent->right;  // 拿到祖父节点的右节点，也就是叔父节点
			if (y->color == RED)  // 叔父节点的颜色是红色
			{ 
				z->parent->color = BLACK;  // 将插入节点z的父节点染成黑色
				y->color = BLACK;          // 将插入节点z的叔父节点染成黑色
				z->parent->parent->color = RED;   // 将插入节点z的祖父节点染成红色
				z = z->parent->parent;     // 更新z节点为它的祖父节点，再次进行判断
			} 
			else // 叔父节点的颜色是黑色
			{       
				if (z == z->parent->right)  // 当前插入节点z是其父节点的右孩子
				{
					z = z->parent;
					rbtree_left_rotate(T, z);  
				}
				z->parent->color = BLACK;
				z->parent->parent->color = RED;
				rbtree_right_rotate(T, z->parent->parent);
			}
		}
		else  // 要插入节点的父节点是祖父节点的右子树
		{
			rbtree_node *y = z->parent->parent->left; // 拿到叔父节点
			if (y->color == RED) 
			{
				z->parent->color = BLACK;
				y->color = BLACK;
				z->parent->parent->color = RED;
				z = z->parent->parent; //z --> RED
			}
			 else 
			 {
				if (z == z->parent->left) 
				{
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
	rbtree_node *x = T->root;  // 从根节点开始遍历
	
	while (x != T->nil)  // 遍历找插入位置，循环到叶子节点终止
	{ 
		y = x;   // 保存x的上一级节点
		if (z->key < x->key) {
			x = x->left;
		} else if (z->key > x->key) {
			x = x->right;
		} else { //Exist
			return ;
		}
	}
	// 循环退出，找到插入位置，让待插入节点的父指针指向y（注意通过上述循环，x就是待插入的位置）
	z->parent = y;
	
	if (y == T->nil) {  // 当前红黑树为空
		T->root = z;
	} else if (z->key < y->key) {   // 当前红黑树不为空，需要判断是到y的左孩子还是右孩子
		y->left = z;
	} else {
		y->right = z;
	}
	
	// 因为每次插入，一定是插入到最后一行，所以需要让z节点的左右孩子都指向空
	z->left = T->nil;
	z->right = T->nil;
	z->color = RED; // 染成红色
	
	rbtree_insert_fixup(T, z);  // 迭代调整

}
```

### 3.2 删除及查找节点实现

```
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
```



## 4.完整代码

```


#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define RED				1
#define BLACK 			2

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



————————————————
版权声明：本文为CSDN博主「基层搬砖的Panda」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_46935110/article/details/125666526