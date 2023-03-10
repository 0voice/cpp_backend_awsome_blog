# 【NO.70】用红黑树封装map和set

## 1.红黑树模拟实现完整代码

如下是红黑树kv模型的模拟实现完整代码，现在我们需要基于此代码封装出map和set基本的接口实现。

```text
#pragma once
#include<iostream>
using namespace std;

enum Color
{
	RED,
	BLACK,
};

template<class K, class V>
struct RBTreeNode
{
	RBTreeNode<K, V>* _left;
	RBTreeNode<K, V>* _right;
	RBTreeNode<K, V>* _parent;

	// 存储数据的键值对
	pair<K, V> _kv;
	// 存储节点颜色
	Color _col;

	RBTreeNode(const pair<K, V>& kv)
		:_left(nullptr)
		, _right(nullptr)
		, _parent(nullptr)
		, _kv(kv)
		, _col(RED)
	{}
};

template<class K, class V>
class RBTree
{
	typedef RBTreeNode<K, V> Node;

	void RotateL(Node* parent)
	{
		Node* subR = parent->_right;
		Node* subRL = subR->_left;
		Node* parentParent = parent->_parent;

		// 让parent的右指针指向subRL,判断一下subRL是否为空
		parent->_right = subRL;
		if (subRL)
			subRL->_parent = parent;

		// subR的左指针链接parent
		subR->_left = parent;
		parent->_parent = subR;

		// parent为根的情况，更新根节点，让根节点指向空
		if (_root == parent)
		{
			_root = subR;
			_root->_parent = nullptr;
		}
		else //若parent为一棵子树，则链接与parentParent的关系
		{
			if (parentParent->_right == parent)
				parentParent->_right = subR;
			else
				parentParent->_left = subR;

			subR->_parent = parentParent;
		}
	}

	void RotateR(Node* parent)
	{
		Node* subL = parent->_left;
		Node* subLR = subL->_right;
		Node* parentParent = parent->_parent;

		// 将subLR链接到parent的左边，这里注意subLR可能为空的情况，需要判断一下
		parent->_left = subLR;
		if (subLR)
		{
			subLR->_parent = parent;
		}

		// 将parent这棵子树链接到subL的右指针
		subL->_right = parent;
		parent->_parent = subL;

		// 若parent为根，则更新新的根节点
		if (parent == _root)
		{
			_root = subL;
			_root->_parent = nullptr;
		}
		else //若parent为一棵子树，则链接与parentParent的关系
		{
			if (parentParent->_right == parent)
				parentParent->_right = subL;
			else
				parentParent->_left = subL;

			subL->_parent = parentParent;
		}
	}

	void _InOrder(Node* root)
	{
		if (root == nullptr)
			return;

		_InOrder(root->_left);
		cout << root->_kv.first << " -> " << root->_kv.second << endl;
		_InOrder(root->_right);
	}

	void _Destroy(Node* root)
	{
		if (root == nullptr)
			return;

		_Destroy(root->_left);
		_Destroy(root->_right);
		delete root;
	}
public:
	void InOrder()
	{
		_InOrder(_root);
	}

	RBTree()
		:_root(nullptr)
	{}

	~RBTree()
	{
		_Destroy(_root);
		_root = nullptr;
	}

	Node* find(const K& key)
	{
		Node* cur = _root;
		while (cur)
		{
			if (cur->_kv.first < key)
				cur = cur->_right;
			else if (cur->_kv.first > key)
				cur = cur->_left;
			else
				return cur;
		}
		return nullptr;
	}

	pair<Node*, bool> insert(const pair<K, V>& kv)
	{
		// 一开始插入新节点时树为空，则直接让新节点作为根节点
		if (_root == nullptr)
		{
			_root = new Node(kv);
			_root->_col = BLACK;
			return make_pair(_root, true);
		}

		Node* cur = _root;
		Node* parent = _root;
		while (cur)
		{
			if (cur->_kv.first < kv.first)
			{
				parent = cur;
				cur = cur->_right;
			}
			else if (cur->_kv.first > kv.first)
			{
				parent = cur;
				cur = cur->_left;
			}
			else
				// 待插入的节点已经存在，则返回该节点的指针，插入失败
				return make_pair(cur, false);
		}

		// 走到这里就已经找到了将要插入的位置
		Node* newnode = new Node(kv);
		newnode->_col = RED;
		// 判断待插入节点与parent指向值的大小，将待插入节点插入到正确的位置
		if (parent->_kv.first < kv.first)
		{
			parent->_right = newnode;
			newnode->_parent = parent;
		}
		else
		{
			parent->_left = newnode;
			newnode->_parent = parent;
		}
		cur = newnode;

		// 若父亲存在且为红色就需要进行处理
		while (parent && parent->_col == RED)
		{
			// 若父节点为红色，则祖父节点一定存在，不需要进行判断
			Node* grandfather = parent->_parent;
			if (parent == grandfather->_left)
			{
				Node* uncle = grandfather->_right;
				//情况1：uncle存在且为红
				if (uncle && uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;

					// 继续向上进行处理
					cur = grandfather;
					parent = cur->_parent;
				}
				else // 情况2+3：uncle不存在或者uncle存在且为黑
				{
					// 情况2：右单旋
					if (cur == parent->_left)
					{
						RotateR(grandfather);
						grandfather->_col = RED;
						parent->_col = BLACK;
					}
					else // 左右双旋
					{
						RotateL(parent);
						RotateR(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;
				}
			}
			else // grandfather->_right == parent;
			{
				Node* uncle = grandfather->_left;
				// 情况1
				if (uncle && uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;

					// 继续向上进行处理
					cur = grandfather;
					parent = cur->_parent;
				}
				else // 情况2+3
				{
					if (cur == parent->_right)
					{
						RotateL(grandfather);
						parent->_col = BLACK;
						grandfather->_col = RED;
					}
					else // cur == parent->_left
					{
						RotateR(parent);
						RotateL(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;
				}
			}
		}

		// 将根节点的颜色处理为黑色
		_root->_col = BLACK;
		return make_pair(newnode, true);
	}

	bool erase(const K& key)
	{
		Node* parent = nullptr;
		Node* cur = _root;
		//用于标记实际的待删除结点及其父结点
		Node* deleteParent = nullptr;
		Node* deleteNode = nullptr;
		while (cur)
		{
			if (key < cur->_kv.first)
			{
				parent = cur;
				cur = cur->_left;
			}
			else if (key > cur->_kv.first)
			{
				parent = cur;
				cur = cur->_right;
			}
			else
			{
				if (cur->_left == nullptr) //待删除结点的左子树为空
				{
					if (cur == _root) //待删除结点是根结点
					{
						_root = _root->_right; //让根结点的右子树作为新的根结点
						if (_root)
						{
							_root->_parent = nullptr;
							_root->_col = BLACK; //根结点为黑色
						}
						delete cur; //删除原根结点
						return true;
					}
					else
					{
						deleteParent = parent; //标记实际删除结点的父结点
						deleteNode = cur; //标记实际删除的结点
					}
					break;
				}
				else if (cur->_right == nullptr) //待删除结点的右子树为空
				{
					if (cur == _root) //待删除结点是根结点
					{
						_root = _root->_left; //让根结点的左子树作为新的根结点
						if (_root)
						{
							_root->_parent = nullptr;
							_root->_col = BLACK; //根结点为黑色
						}
						delete cur; //删除原根结点
						return true;
					}
					else
					{
						deleteParent = parent; //标记实际删除结点的父结点
						deleteNode = cur; //标记实际删除的结点
					}
					break;
				}
				else //待删除结点的左右子树均不为空
				{
					//替换法删除
					//寻找待删除结点右子树当中key值最小的结点作为实际删除结点
					Node* minParent = cur;
					Node* minRight = cur->_right;
					while (minRight->_left)
					{
						minParent = minRight;
						minRight = minRight->_left;
					}
					cur->_kv.first = minRight->_kv.first;
					cur->_kv.second = minRight->_kv.second;
					deleteParent = minParent;
					deleteNode = minRight;
					break;
				}
			}
		}
		if (deleteNode == nullptr)
		{
			return false;
		}

		//记录待删除结点及其父结点
		Node* del = deleteNode;
		Node* delP = deleteParent;

		//调整红黑树
		if (deleteNode->_col == BLACK) //删除的是黑色结点
		{
			if (deleteNode->_left)
				deleteNode->_left->_col = BLACK;

			else if (deleteNode->_right) //待删除结点有一个红色的右孩子（不可能是黑色）
			{
				deleteNode->_right->_col = BLACK;
			}
			else //待删除结点的左右均为空
			{
				while (deleteNode != _root)
				{
					if (deleteNode == deleteParent->_left)
					{
						Node* brother = deleteParent->_right;
						//情况一：brother为红色
						if (brother->_col == RED)
						{
							deleteParent->_col = RED;
							brother->_col = BLACK;
							RotateL(deleteParent);

							brother = deleteParent->_right; //更新brother
						}
						//情况二：brother为黑色，且其左右孩子都是黑色结点或为空
						if (((brother->_left == nullptr) || (brother->_left->_col == BLACK))
							&& ((brother->_right == nullptr) || (brother->_right->_col == BLACK)))
						{
							brother->_col = RED;
							if (deleteParent->_col == RED)
							{
								deleteParent->_col = BLACK;
								break;
							}

							deleteNode = deleteParent;
							deleteParent = deleteNode->_parent;
						}
						else
						{
							//情况三：brother为黑色，且其左孩子是红色结点，右孩子是黑色结点或为空
							if ((brother->_right == nullptr) || (brother->_right->_col == BLACK))
							{
								brother->_left->_col = BLACK;
								brother->_col = RED;
								RotateR(brother);

								brother = deleteParent->_right; //更新brother
							}
							//情况四：brother为黑色，且其右孩子是红色结点
							brother->_col = deleteParent->_col;
							deleteParent->_col = BLACK;
							brother->_right->_col = BLACK;
							RotateL(deleteParent);
							break;
						}
					}
					else  //待删除结点是其父结点的左孩子
					{
						Node* brother = deleteParent->_left;
						//情况一：brother为红色
						if (brother->_col == RED)
						{
							deleteParent->_col = RED;
							brother->_col = BLACK;
							RotateR(deleteParent);
							//需要继续处理
							brother = deleteParent->_left;
						}
						//情况二：brother为黑色，且其左右孩子都是黑色结点或为空
						if (((brother->_left == nullptr) || (brother->_left->_col == BLACK))
							&& ((brother->_right == nullptr) || (brother->_right->_col == BLACK)))
						{
							brother->_col = RED;
							if (deleteParent->_col == RED)
							{
								deleteParent->_col = BLACK;
								break;
							}

							deleteNode = deleteParent;
							deleteParent = deleteNode->_parent;
						}
						else
						{
							//情况三：brother为黑色，且其右孩子是红色结点，左孩子是黑色结点或为空
							if ((brother->_left == nullptr) || (brother->_left->_col == BLACK))
							{
								brother->_right->_col = BLACK;
								brother->_col = RED;
								RotateL(brother);

								brother = deleteParent->_left;
							}
							//情况四：brother为黑色，且其左孩子是红色结点
							brother->_col = deleteParent->_col;
							deleteParent->_col = BLACK;
							brother->_left->_col = BLACK;
							RotateR(deleteParent);
							break;
						}
					}
				}
			}
		}
		//进行实际删除
		if (del->_left == nullptr)
		{
			if (del == delP->_left)
			{
				delP->_left = del->_right;
				if (del->_right)
					del->_right->_parent = delP;
			}
			else
			{
				delP->_right = del->_right;
				if (del->_right)
					del->_right->_parent = delP;
			}
		}
		else
		{
			if (del == delP->_left)
			{
				delP->_left = del->_left;
				if (del->_left)
					del->_left->_parent = delP;
			}
			else
			{
				delP->_right = del->_left;
				if (del->_left)
					del->_left->_parent = delP;
			}
		}
		delete del;
		return true;
	}
private:
	Node* _root;
};
```

## 2.红黑树参数适配改造

这里我们需要用同一棵红黑树来实现map和set，但是map是kv模型，而set是key模型。我们实现的红黑树是kv模型，为了同时适配k的模型，我们需要对红黑树的模板参数做一些处理。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1007' height='531'></svg>)

1. 这里需要控制 map 和 set 传入红黑树底层结构的模板参数，为了与之前的 kv 模型参数进行区分，将红黑树的第二个模板参数改为T。则T参数可能存储键值 key，也有可能存储键值对 key-value。
2. map 传递给红黑树的T模板参数是 pair<K,V> , set 传递给红黑树的T模板参数是 K 。

```text
template<class K>
class set
{
private:
	RBTree<K, K> _t;
};

template<class K,class V>
class map
{
private:
	RBTree<K, pair<const K,V>> _t;
};
```

❓为什么不去掉红黑树的第一个模板参数，只保留第二个模板参数呢？

不能！红黑树中的第二个模板参数是一个键值对，对于 set 容器来说，省略红黑树第一个模板参数是没有任何问题的，但是对于 map 容器而言，若去掉红黑树的第二个模板参数，那么就无法得到 key 的类型，像 find、erase 这样的接口中是需要键值 key 的，所以第一个模板参数不能够省略。



## 3.仿函数

红黑树中的模板参数T可能是 key ，也可能是 <key,value>键值对。在红黑树的实现中需要多次用 key 值进行比较，那我们应该怎样拿到节点的 key 呢？为了解决这个问题，上层容器 map 和 set 需要向底层结构红黑树提供一个仿函数去获取T模板的键值 key 。

![img](https://pic4.zhimg.com/80/v2-7a5c60b0c4a4622715a753a8c3a8d7db_720w.webp)

当上层容器是 set 时，红黑树中的 T 就是键值 key ，因此 set 的仿函数直接返回 K 值就可以了。

```text
template<class K>
class set
{
	struct SetKeyOfT
	{
		const K& operator()(const K& key)
		{
			return key;
		}
	};

private:
	RBTree<K, K, SetKeyOfT> _t;
};
```

当上层容器是 map 时，红黑树中的 T 储存的就是键值对 pair<key,value> ，因此 map 的仿函数需要取出键值对中的第一个参数 key 。

```text
template<class K,class V>
class map
{
	struct MapKeyOfT
	{
		const K& operator()(const pair<K, V>& kv)
		{
			return kv.first;
		}
	};

private:
	RBTree<K, pair<const K, V>, MapKeyOfT> _t;
};
```

✅仿函数应该如何使用呢？以下我们以红黑树的查找为例来看看仿函数的使用方法：

```text
iterator find(const K& key)
{
	KeyOfT kot; // 定义一个仿函数对象
	Node* cur = _root;
	while (cur)
	{
		if (kot(cur->_data) < key) // 利用仿函数对象去取_data中用来进行比较的参数
			cur = cur->_right;
		else if (kot(cur->_data) > key)
			cur = cur->_left;
		else
			return iterator(cur);
	}
	return end();
}
```

## 3.正向迭代器

✔️红黑树的迭代器即对红黑树中节点指针进行封装，便于使用。

```text
// 正向迭代器
template<class T,class Ref,class Ptr>
struct __TreeIterator
{
	typedef Ref reference;
	typedef Ptr pointer;

	typedef RBTreeNode<T> Node; // 重定义节点的类型
	typedef __TreeIterator<T, Ref, Ptr> Self; // 迭代器类型

	Node* _node; // 迭代器封装节点指针

	__TreeIterator(Node* node) // 迭代器的构造函数
		:_node(node)
	{}
}
```

✔️对正向迭代器进行解引用操作，只需要返回对应节点的数据即可。使用 -> 操作时，只需返回对应节点数据指针即可。

```text
Ref operator*()
{
	return _node->_data; // 返回节点数据的引用
}

Ptr operator->()
{
	return &(_node->_data); // 返回节点数据的指针
}
```

✔️在迭代器中我们还要使用 != 这个操作符，让迭代器指针在不越界的情况下依次向后遍历。

```text
bool operator!=(const Self& s)const
{
	return _node != s._node;
}

bool operator==(const Self& s)const
{
	return _node == s._node;
}
```

✔️红黑树正向迭代器找到下一个节点需要进行++操作，应该根据红黑树中序遍历的序列找到当前节点的下一个节点。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='819' height='351'></svg>)

实现红黑树正向迭代器时，当前节点进行 ++ 操作后，应该根据红黑树中序遍历的序列找到当前节点的下一个节点。

++ 操作的具体步骤：

- 若当前节点的右子树不为空，则 ++ 之后应是其右子树中的最左节点。
- 若当前节点的右子树为空，则 ++ 之后应是该节点的祖先节点中，孩子不在父节点右的祖先。

实现代码：

```text
Self operator++()
{
	if (_node->_right) // 当前节点的右子树不为空
	{
		// 找到右子树中的最左节点 - 即右子树中中序的第一个节点
		Node* left = _node->_right;
		while (left->_left)
		{
			left = left->_left;
		}
		_node = left;
	}
	else // 当前节点右子树为空
	{
		Node* cur = _node;
		Node* parent = cur->_parent;
		while (parent && parent->_right == cur)
		{
			cur = cur->_parent;
			parent = parent->_parent;
		}
		_node = parent;
	}
	return *this;
}
```

✔️红黑树迭代器中 - - 的实现恰好与 ++ 相反，根据红黑树中序遍历序列找到当前节点的前一个节点。

\- - 操作的具体步骤：

1. 若当前节点的左子树不为空，则 - - 之后应是其左子树中的最右节点。
2. 若当前节点的左子树为空，则 - - 之后应是该节点的祖先节点中，孩子不在父节点左的祖先。

实现代码：

```text
Self& operator--()
{
	if (_node->_left)
	{
		// 左子树的最右节点
		Node* right = _node->_left;
		while (right->_right)
		{
			right = right->_right;
		}
		_node = right;
	}
	else
	{
		Node* cur = _node;
		Node* parent = cur->_parent;
		while (parent && cur == parent->_left)
		{
			cur = parent;
			parent = parent->_parent;
		}
		_node = parent;
	}
	return *this;
}
```

**注意：** 此处的 ++ 操作和 - - 操作和STL中的实现是不一样的，相对库里面的实现是有差距的，这里只是一个简单的模拟。想要看STL库里面的实现可以去查一下。

**正向迭代器完整代码：**

```text
template<class T,class Ref,class Ptr>
struct __TreeIterator
{
	typedef Ref reference;
	typedef Ptr pointer;

	typedef RBTreeNode<T> Node;
	typedef __TreeIterator<T, Ref, Ptr> Self;

	Node* _node;

	__TreeIterator(Node* node)
		:_node(node)
	{}

	Ref operator*()
	{
		return _node->_data;
	}

	Ptr operator->()
	{
		return &(_node->_data);
	}

	bool operator!=(const Self& s)const
	{
		return _node != s._node;
	}

	bool operator==(const Self& s)const
	{
		return _node == s._node;
	}

	Self operator++()
	{
		if (_node->_right) // 当前节点的右子树不为空
		{
			// 找到右子树中的最左节点 - 即右子树中中序的第一个节点
			Node* left = _node->_right;
			while (left->_left)
			{
				left = left->_left;
			}
			_node = left;
		}
		else // 当前节点右子树为空
		{
			Node* cur = _node;
			Node* parent = cur->_parent;
			while (parent && parent->_right == cur)
			{
				cur = cur->_parent;
				parent = parent->_parent;
			}
			_node = parent;
		}
		return *this;
	}

	Self& operator--()
	{
		if (_node->_left)
		{
			// 左子树的最右节点
			Node* right = _node->_left;
			while (right->_right)
			{
				right = right->_right;
			}
			_node = right;
		}
		else
		{
			Node* cur = _node;
			Node* parent = cur->_parent;
			while (parent && cur == parent->_left)
			{
				cur = parent;
				parent = parent->_parent;
			}
			_node = parent;
		}
		return *this;
	}
};
```

## 4.反向迭代器

红黑树的反向迭代器是由正向迭代器封装的，因此红黑树的反向迭代器也是一个迭代器适配器。和之前学习的 priority_queue、stack、queue 都是属于适配器。

```text
// 反向迭代器 - 迭代器适配器
template<class Iterator>
struct ReverseIterator
{
	typedef typename Iterator::reference Ref; // 节点指针的引用
	typedef typename Iterator::pointer Ptr;  // 节点指针
	typedef typename ReverseIterator<Iterator> Self; // 反向迭代器类型

	ReverseIterator(Iterator it)
		:_it(it) // 用正向迭代器封装一个反向迭代器
	{}

	Ref operator*()
	{
		return *_it; // 调用正向迭代器的operator* 返回节点数据的引用
	}

	Ptr operator->()
	{
		return _it.operator->(); // 调用正向迭代器的operator-> 返回节点数据指针
	}

	Self& operator++()
	{
		--_it; // 调用正向迭代器的 --
		return *this;
	}

	Self& operator--()
	{
		++_it; // 调用正向迭代器的 ++
		return *this;
	}

	bool operator==(const Self& s)const
	{
		return _it == s._it;
	}

	bool operator!=(const Self& s)const
	{
		return _it != s._it;
	}

	Iterator _it; 
};
```

☃️反向迭代器只有一个模板参数，即正向迭代器的类型。因此反向迭代器是不知道节点的引用类型和节点的指针类型的，这时我们需要对正向迭代器中这两个类型进行 typedef ，则反向迭代器可通过正向迭代器获取节点引用类型和节点指针类型。

```text
template<class T,class Ref,class Ptr>
struct __TreeIterator
{
	typedef Ref reference; // 节点指针引用
	typedef Ptr pointer; // 节点指针
}
```

## 5.红黑树封装后的代码

```text
#pragma once
#include<iostream>
using namespace std;

enum Color
{
	RED,
	BLACK,
};

template<class T>
struct RBTreeNode
{
	RBTreeNode<T>* _left;
	RBTreeNode<T>* _right;
	RBTreeNode<T>* _parent;

	// 存储数据的键值对
	T _data;
	// 存储节点颜色
	Color _col;

	RBTreeNode(const T& data)
		:_left(nullptr)
		,_right(nullptr)
		,_parent(nullptr)
		,_data(data)
		,_col(RED)
	{}
};

template<class T,class Ref,class Ptr>
struct __TreeIterator
{
	typedef Ref reference;
	typedef Ptr pointer;

	typedef RBTreeNode<T> Node;
	typedef __TreeIterator<T, Ref, Ptr> Self;

	Node* _node;

	__TreeIterator(Node* node)
		:_node(node)
	{}

	Ref operator*()
	{
		return _node->_data;
	}

	Ptr operator->()
	{
		return &(_node->_data);
	}

	bool operator!=(const Self& s)const
	{
		return _node != s._node;
	}

	bool operator==(const Self& s)const
	{
		return _node == s._node;
	}

	Self operator++()
	{
		if (_node->_right) // 当前节点的右子树不为空
		{
			// 找到右子树中的最左节点 - 即右子树中中序的第一个节点
			Node* left = _node->_right;
			while (left->_left)
			{
				left = left->_left;
			}
			_node = left;
		}
		else // 当前节点右子树为空
		{
			Node* cur = _node;
			Node* parent = cur->_parent;
			while (parent && parent->_right == cur)
			{
				cur = cur->_parent;
				parent = parent->_parent;
			}
			_node = parent;
		}
		return *this;
	}

	Self& operator--()
	{
		if (_node->_left)
		{
			// 左子树的最右节点
			Node* right = _node->_left;
			while (right->_right)
			{
				right = right->_right;
			}
			_node = right;
		}
		else
		{
			Node* cur = _node;
			Node* parent = cur->_parent;
			while (parent && cur == parent->_left)
			{
				cur = parent;
				parent = parent->_parent;
			}
			_node = parent;
		}
		return *this;
	}
};

template<class K,class T,class KeyOfT>
class RBTree
{
	typedef RBTreeNode<T> Node;
public:
	typedef __TreeIterator<T, T&, T*> iterator;
	typedef __TreeIterator<T, const T&, const T*> const_iterator;
	typedef ReverseIterator<iterator> reverse_iterator;

	iterator begin()
	{
		// 返回红黑树的最左节点
		Node* left = _root;
		while (left && left->_left)
		{
			left = left->_left;
		}
		return iterator(left);
	}

	iterator end()
	{
		return iterator(nullptr); // 用nullptr去标志红黑树的结束位置
	}

	reverse_iterator rbegin()
	{
		// 返回红黑树的最右节点
		Node* right = _root;
		while (right && right->_right)
		{
			right = right->_right;
		}
		return reverse_iterator(iterator(right));
	}

	reverse_iterator rend()
	{
		return reverse_iterator(iterator(nullptr));
	}

private:
	void RotateL(Node* parent)
	{
		Node* subR = parent->_right;
		Node* subRL = subR->_left;
		Node* parentParent = parent->_parent;

		// 让parent的右指针指向subRL,判断一下subRL是否为空
		parent->_right = subRL;
		if (subRL)
			subRL->_parent = parent;

		// subR的左指针链接parent
		subR->_left = parent;
		parent->_parent = subR;

		// parent为根的情况，更新根节点，让根节点指向空
		if (_root == parent)
		{
			_root = subR;
			_root->_parent = nullptr;
		}
		else //若parent为一棵子树，则链接与parentParent的关系
		{
			if (parentParent->_right == parent)
				parentParent->_right = subR;
			else
				parentParent->_left = subR;

			subR->_parent = parentParent;
		}
	}


	void RotateR(Node* parent)
	{
		Node* subL = parent->_left;
		Node* subLR = subL->_right;
		Node* parentParent = parent->_parent;

		// 将subLR链接到parent的左边，这里注意subLR可能为空的情况，需要判断一下
		parent->_left = subLR;
		if (subLR)
		{
			subLR->_parent = parent;
		}

		// 将parent这棵子树链接到subL的右指针
		subL->_right = parent;
		parent->_parent = subL;

		// 若parent为根，则更新新的根节点
		if (parent == _root)
		{
			_root = subL;
			_root->_parent = nullptr;
		}
		else //若parent为一棵子树，则链接与parentParent的关系
		{
			if (parentParent->_right == parent)
				parentParent->_right = subL;
			else
				parentParent->_left = subL;

			subL->_parent = parentParent;
		}
	}

	void _Destroy(Node* root)
	{
		if (root == nullptr)
			return;

		_Destroy(root->_left);
		_Destroy(root->_right);
		delete root;
	}
public:
	RBTree()
		:_root(nullptr)
	{}

	~RBTree()
	{
		_Destroy(_root);
		_root = nullptr;
	}

	iterator find(const K& key)
	{
		KeyOfT kot;
		Node* cur = _root;
		while (cur)
		{
			if (kot(cur->_data) < key)
				cur = cur->_right;
			else if (kot(cur->_data) > key)
				cur = cur->_left;
			else
				return iterator(cur);
		}
		return end();
	}

	pair<iterator, bool> insert(const T& data)
	{
		// 一开始插入新节点时树为空，则直接让新节点作为根节点
		if (_root == nullptr)
		{
			_root = new Node(data);
			_root->_col = BLACK;
			return make_pair(iterator(_root), true);
		}

		KeyOfT kot;
		Node* cur = _root;
		Node* parent = _root;
		while (cur)
		{
			if (kot(cur->_data) < kot(data))
			{
				parent = cur;
				cur = cur->_right;
			}
			else if (kot(cur->_data) > kot(data))
			{
				parent = cur;
				cur = cur->_left;
			}
			else
				// 待插入的节点已经存在，则返回该节点的指针，插入失败
				return make_pair(iterator(cur), false);
		}

		// 走到这里就已经找到了将要插入的位置
		Node* newnode = new Node(data);
		newnode->_col = RED;
		// 判断待插入节点与parent指向值的大小，将待插入节点插入到正确的位置
		if (kot(parent->_data) < kot(data))
		{c
			parent->_right = newnode;
			newnode->_parent = parent;
		}
		else
		{
			parent->_left = newnode;
			newnode->_parent = parent;
		}
		cur = newnode;

		// 若父亲存在且为红色就需要进行处理
		while (parent&& parent->_col == RED)
		{
			// 若父节点为红色，则祖父节点一定存在，不需要进行判断
			Node* grandfather = parent->_parent;
			if (parent == grandfather->_left)
			{
				Node* uncle = grandfather->_right;
				//情况1：uncle存在且为红
				if (uncle && uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;

					// 继续向上进行处理
					cur = grandfather;
					parent = cur->_parent;
				}
				else // 情况2+3：uncle不存在或者uncle存在且为黑
				{
					// 情况2：右单旋
					if (cur == parent->_left)
					{
						RotateR(grandfather);
						grandfather->_col = RED;
						parent->_col = BLACK;
					}
					else // 左右双旋
					{
						RotateL(parent);
						RotateR(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;
				}
			}
			else // grandfather->_right == parent;
			{
				Node* uncle = grandfather->_left;
				// 情况1
				if (uncle&& uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;

					// 继续向上进行处理
					cur = grandfather;
					parent = cur->_parent;
				}
				else // 情况2+3
				{
					if (cur == parent->_right)
					{
						RotateL(grandfather);
						parent->_col = BLACK;
						grandfather->_col = RED;
					}
					else // cur == parent->_left
					{
						RotateR(parent);
						RotateL(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;
				}
			}
		}

		// 将根节点的颜色处理为黑色
		_root->_col = BLACK;
		return make_pair(iterator(newnode), true);
	}

	bool erase(const K& key)
	{
		Node* parent = nullptr;
		Node* cur = _root;
		//用于标记实际的待删除结点及其父结点
		Node* deleteParent = nullptr;
		Node* deleteNode = nullptr;
		while (cur)
		{
			if (key < cur->_kv.first)
			{
				parent = cur;
				cur = cur->_left;
			}
			else if (key > cur->_kv.first)
			{
				parent = cur;
				cur = cur->_right;
			}
			else
			{
				if (cur->_left == nullptr) //待删除结点的左子树为空
				{
					if (cur == _root) //待删除结点是根结点
					{
						_root = _root->_right; //让根结点的右子树作为新的根结点
						if (_root)
						{
							_root->_parent = nullptr;
							_root->_col = BLACK; //根结点为黑色
						}
						delete cur; //删除原根结点
						return true;
					}
					else
					{
						deleteParent = parent; //标记实际删除结点的父结点
						deleteNode = cur; //标记实际删除的结点
					}
					break;
				}
				else if (cur->_right == nullptr) //待删除结点的右子树为空
				{
					if (cur == _root) //待删除结点是根结点
					{
						_root = _root->_left; //让根结点的左子树作为新的根结点
						if (_root)
						{
							_root->_parent = nullptr;
							_root->_col = BLACK; //根结点为黑色
						}
						delete cur; //删除原根结点
						return true;
					}
					else
					{
						deleteParent = parent; //标记实际删除结点的父结点
						deleteNode = cur; //标记实际删除的结点
					}
					break;
				}
				else //待删除结点的左右子树均不为空
				{
					//替换法删除
					//寻找待删除结点右子树当中key值最小的结点作为实际删除结点
					Node* minParent = cur;
					Node* minRight = cur->_right;
					while (minRight->_left)
					{
						minParent = minRight;
						minRight = minRight->_left;
					}
					cur->_kv.first = minRight->_kv.first;
					cur->_kv.second = minRight->_kv.second;
					deleteParent = minParent;
					deleteNode = minRight;
					break;
				}
			}
		}
		if (deleteNode == nullptr)
		{
			return false;
		}

		//记录待删除结点及其父结点
		Node* del = deleteNode;
		Node* delP = deleteParent;

		//调整红黑树
		if (deleteNode->_col == BLACK) //删除的是黑色结点
		{
			if (deleteNode->_left)
				deleteNode->_left->_col = BLACK;

			else if (deleteNode->_right) //待删除结点有一个红色的右孩子（不可能是黑色）
			{
				deleteNode->_right->_col = BLACK;
			}
			else //待删除结点的左右均为空
			{
				while (deleteNode != _root)
				{
					if (deleteNode == deleteParent->_left)
					{
						Node* brother = deleteParent->_right;
						//情况一：brother为红色
						if (brother->_col == RED)
						{
							deleteParent->_col = RED;
							brother->_col = BLACK;
							RotateL(deleteParent);

							brother = deleteParent->_right; //更新brother
						}
						//情况二：brother为黑色，且其左右孩子都是黑色结点或为空
						if (((brother->_left == nullptr) || (brother->_left->_col == BLACK))
							&& ((brother->_right == nullptr) || (brother->_right->_col == BLACK)))
						{
							brother->_col = RED;
							if (deleteParent->_col == RED)
							{
								deleteParent->_col = BLACK;
								break;
							}

							deleteNode = deleteParent;
							deleteParent = deleteNode->_parent;
						}
						else
						{
							//情况三：brother为黑色，且其左孩子是红色结点，右孩子是黑色结点或为空
							if ((brother->_right == nullptr) || (brother->_right->_col == BLACK))
							{
								brother->_left->_col = BLACK;
								brother->_col = RED;
								RotateR(brother);

								brother = deleteParent->_right; //更新brother
							}
							//情况四：brother为黑色，且其右孩子是红色结点
							brother->_col = deleteParent->_col;
							deleteParent->_col = BLACK;
							brother->_right->_col = BLACK;
							RotateL(deleteParent);
							break;
						}
					}
					else  //待删除结点是其父结点的左孩子
					{
						Node* brother = deleteParent->_left;
						//情况一：brother为红色
						if (brother->_col == RED)
						{
							deleteParent->_col = RED;
							brother->_col = BLACK;
							RotateR(deleteParent);
							//需要继续处理
							brother = deleteParent->_left;
						}
						//情况二：brother为黑色，且其左右孩子都是黑色结点或为空
						if (((brother->_left == nullptr) || (brother->_left->_col == BLACK))
							&& ((brother->_right == nullptr) || (brother->_right->_col == BLACK)))
						{
							brother->_col = RED;
							if (deleteParent->_col == RED)
							{
								deleteParent->_col = BLACK;
								break;
							}

							deleteNode = deleteParent;
							deleteParent = deleteNode->_parent;
						}
						else
						{
							//情况三：brother为黑色，且其右孩子是红色结点，左孩子是黑色结点或为空
							if ((brother->_left == nullptr) || (brother->_left->_col == BLACK))
							{
								brother->_right->_col = BLACK;
								brother->_col = RED;
								RotateL(brother);

								brother = deleteParent->_left;
							}
							//情况四：brother为黑色，且其左孩子是红色结点
							brother->_col = deleteParent->_col;
							deleteParent->_col = BLACK;
							brother->_left->_col = BLACK;
							RotateR(deleteParent);
							break;
						}
					}
				}
			}
		}
		//进行实际删除
		if (del->_left == nullptr)
		{
			if (del == delP->_left)
			{
				delP->_left = del->_right;
				if (del->_right)
					del->_right->_parent = delP;
			}
			else
			{
				delP->_right = del->_right;
				if (del->_right)
					del->_right->_parent = delP;
			}
		}
		else
		{
			if (del == delP->_left)
			{
				delP->_left = del->_left;
				if (del->_left)
					del->_left->_parent = delP;
			}
			else
			{
				delP->_right = del->_left;
				if (del->_left)
					del->_left->_parent = delP;
			}
		}
		delete del;
		return true;
	}

private:
	Node* _root;
};
```

## 6.map完整代码

```text
template<class K,class V>
class map
{
	struct MapKeyOfT
	{
		const K& operator()(const pair<const K, V>& kv)
		{
			return kv.first;
		}
	};

public:
	typedef typename RBTree<K, pair<const K, V>, MapKeyOfT>::iterator iterator;
	typedef typename RBTree<K, pair<const K, V>, MapKeyOfT>::reverse_iterator reverse_iterator;

	reverse_iterator rbegin()
	{
		return _t.rbegin();
	}

	reverse_iterator rend()
	{
		return _t.rend();
	}

	iterator begin()
	{
		return _t.begin();
	}

	iterator end()
	{
		return _t.end();
	}

	pair<iterator, bool> insert(const pair<const K, V>& kv)
	{
		return _t.insert(kv);
	}

	// []的运算符重载
	V& operator[](const K& key)
	{
		pair<iterator, bool> ret = insert(make_pair(key, V()));
		return ret.first->second;
	}

	void erase(const K& k)
	{
		return _t.erase();
	}

	iterator find(const K& k)
	{
		return _t.find();
	}
private:
	RBTree<K, pair<const K, V>, MapKeyOfT> _t;
};
```

## 7.set完整代码

```text
template<class K>
class set
{
	struct SetKeyOfT
	{
		const K& operator()(const K& key)
		{
			return key;
		}
	};
public:
	typedef typename RBTree<K, K, SetKeyOfT>::iterator iterator;
	typedef typename RBTree<K, K, SetKeyOfT>::reverse_iterator reverse_iterator;

	iterator begin()
	{
		return _t.begin();
	}

	iterator end()
	{
		return _t.end();
	}

	reverse_iterator rbegin()
	{
		return _t.rbegin();
	}

	reverse_iterator rend()
	{
		return _t.rend();
	}

	pair<iterator, bool> insert(const K& k)
	{
		return _t.insert(k);
	}

	void erase(const K& k)
	{
		return _t.erase(k);
	}

	iterator find(const K& k)
	{
		return _t.find(k);
	}

private:
	RBTree<K, K, SetKeyOfT> _t;
};
```

**说明：**我们对 map 和 set 进行模拟实现只是为了让我们更加了解它们的框架和红黑树的底层实现原理。

原文地址：https://zhuanlan.zhihu.com/p/579730274

作者：linux