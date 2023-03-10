# 【NO.4】红黑树(三)之 Linux内核中红黑树的经典实现

## **1.Linux内核中红黑树(完整源码)**

### 1.1红黑树的实现文件(rbtree.h)

![img](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  1 /*
  2   Red Black Trees
  3   (C) 1999  Andrea Arcangeli <andrea@suse.de>
  4   
  5   This program is free software; you can redistribute it and/or modify
  6   it under the terms of the GNU General Public License as published by
  7   the Free Software Foundation; either version 2 of the License, or
  8   (at your option) any later version.
  9 
 10   This program is distributed in the hope that it will be useful,
 11   but WITHOUT ANY WARRANTY; without even the implied warranty of
 12   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 13   GNU General Public License for more details.
 14 
 15   You should have received a copy of the GNU General Public License
 16   along with this program; if not, write to the Free Software
 17   Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
 18 
 19   linux/include/linux/rbtree.h
 20 
 21   To use rbtrees you'll have to implement your own insert and search cores.
 22   This will avoid us to use callbacks and to drop drammatically performances.
 23   I know it's not the cleaner way,  but in C (not in C++) to get
 24   performances and genericity...
 25 
 26   Some example of insert and search follows here. The search is a plain
 27   normal search over an ordered tree. The insert instead must be implemented
 28   in two steps: First, the code must insert the element in order as a red leaf
 29   in the tree, and then the support library function rb_insert_color() must
 30   be called. Such function will do the not trivial work to rebalance the
 31   rbtree, if necessary.
 32 
 33 -----------------------------------------------------------------------
 34 static inline struct page * rb_search_page_cache(struct inode * inode,
 35                          unsigned long offset)
 36 {
 37     struct rb_node * n = inode->i_rb_page_cache.rb_node;
 38     struct page * page;
 39 
 40     while (n)
 41     {
 42         page = rb_entry(n, struct page, rb_page_cache);
 43 
 44         if (offset < page->offset)
 45             n = n->rb_left;
 46         else if (offset > page->offset)
 47             n = n->rb_right;
 48         else
 49             return page;
 50     }
 51     return NULL;
 52 }
 53 
 54 static inline struct page * __rb_insert_page_cache(struct inode * inode,
 55                            unsigned long offset,
 56                            struct rb_node * node)
 57 {
 58     struct rb_node ** p = &inode->i_rb_page_cache.rb_node;
 59     struct rb_node * parent = NULL;
 60     struct page * page;
 61 
 62     while (*p)
 63     {
 64         parent = *p;
 65         page = rb_entry(parent, struct page, rb_page_cache);
 66 
 67         if (offset < page->offset)
 68             p = &(*p)->rb_left;
 69         else if (offset > page->offset)
 70             p = &(*p)->rb_right;
 71         else
 72             return page;
 73     }
 74 
 75     rb_link_node(node, parent, p);
 76 
 77     return NULL;
 78 }
 79 
 80 static inline struct page * rb_insert_page_cache(struct inode * inode,
 81                          unsigned long offset,
 82                          struct rb_node * node)
 83 {
 84     struct page * ret;
 85     if ((ret = __rb_insert_page_cache(inode, offset, node)))
 86         goto out;
 87     rb_insert_color(node, &inode->i_rb_page_cache);
 88  out:
 89     return ret;
 90 }
 91 -----------------------------------------------------------------------
 92 */
 93 
 94 #ifndef    _SLINUX_RBTREE_H
 95 #define    _SLINUX_RBTREE_H
 96 
 97 #include <stdio.h>
 98 //#include <linux/kernel.h>
 99 //#include <linux/stddef.h>
100 
101 struct rb_node
102 {
103     unsigned long  rb_parent_color;
104 #define    RB_RED        0
105 #define    RB_BLACK    1
106     struct rb_node *rb_right;
107     struct rb_node *rb_left;
108 } /*  __attribute__((aligned(sizeof(long))))*/;
109     /* The alignment might seem pointless, but allegedly CRIS needs it */
110 
111 struct rb_root
112 {
113     struct rb_node *rb_node;
114 };
115 
116 
117 #define rb_parent(r)   ((struct rb_node *)((r)->rb_parent_color & ~3))
118 #define rb_color(r)   ((r)->rb_parent_color & 1)
119 #define rb_is_red(r)   (!rb_color(r))
120 #define rb_is_black(r) rb_color(r)
121 #define rb_set_red(r)  do { (r)->rb_parent_color &= ~1; } while (0)
122 #define rb_set_black(r)  do { (r)->rb_parent_color |= 1; } while (0)
123 
124 static inline void rb_set_parent(struct rb_node *rb, struct rb_node *p)
125 {
126     rb->rb_parent_color = (rb->rb_parent_color & 3) | (unsigned long)p;
127 }
128 static inline void rb_set_color(struct rb_node *rb, int color)
129 {
130     rb->rb_parent_color = (rb->rb_parent_color & ~1) | color;
131 }
132 
133 #define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
134 
135 #define container_of(ptr, type, member) ({          \
136     const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
137     (type *)( (char *)__mptr - offsetof(type,member) );})
138 
139 #define RB_ROOT    (struct rb_root) { NULL, }
140 #define    rb_entry(ptr, type, member) container_of(ptr, type, member)
141 
142 #define RB_EMPTY_ROOT(root)    ((root)->rb_node == NULL)
143 #define RB_EMPTY_NODE(node)    (rb_parent(node) == node)
144 #define RB_CLEAR_NODE(node)    (rb_set_parent(node, node))
145 
146 static inline void rb_init_node(struct rb_node *rb)
147 {
148     rb->rb_parent_color = 0;
149     rb->rb_right = NULL;
150     rb->rb_left = NULL;
151     RB_CLEAR_NODE(rb);
152 }
153 
154 extern void rb_insert_color(struct rb_node *, struct rb_root *);
155 extern void rb_erase(struct rb_node *, struct rb_root *);
156 
157 typedef void (*rb_augment_f)(struct rb_node *node, void *data);
158 
159 extern void rb_augment_insert(struct rb_node *node,
160                   rb_augment_f func, void *data);
161 extern struct rb_node *rb_augment_erase_begin(struct rb_node *node);
162 extern void rb_augment_erase_end(struct rb_node *node,
163                  rb_augment_f func, void *data);
164 
165 /* Find logical next and previous nodes in a tree */
166 extern struct rb_node *rb_next(const struct rb_node *);
167 extern struct rb_node *rb_prev(const struct rb_node *);
168 extern struct rb_node *rb_first(const struct rb_root *);
169 extern struct rb_node *rb_last(const struct rb_root *);
170 
171 /* Fast replacement of a single node without remove/rebalance/add/rebalance */
172 extern void rb_replace_node(struct rb_node *victim, struct rb_node *new, 
173                 struct rb_root *root);
174 
175 static inline void rb_link_node(struct rb_node * node, struct rb_node * parent,
176                 struct rb_node ** rb_link)
177 {
178     node->rb_parent_color = (unsigned long )parent;
179     node->rb_left = node->rb_right = NULL;
180 
181     *rb_link = node;
182 }
183 
184 #endif    /* _LINUX_RBTREE_H */
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

### 1.2红黑树的实现文件(rbtree.c)

![img](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  1 /*
  2   Red Black Trees
  3   (C) 1999  Andrea Arcangeli <andrea@suse.de>
  4   (C) 2002  David Woodhouse <dwmw2@infradead.org>
  5   
  6   This program is free software; you can redistribute it and/or modify
  7   it under the terms of the GNU General Public License as published by
  8   the Free Software Foundation; either version 2 of the License, or
  9   (at your option) any later version.
 10 
 11   This program is distributed in the hope that it will be useful,
 12   but WITHOUT ANY WARRANTY; without even the implied warranty of
 13   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 14   GNU General Public License for more details.
 15 
 16   You should have received a copy of the GNU General Public License
 17   along with this program; if not, write to the Free Software
 18   Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
 19 
 20   linux/lib/rbtree.c
 21 */
 22 
 23 #include "rbtree.h"
 24 
 25 static void __rb_rotate_left(struct rb_node *node, struct rb_root *root)
 26 {
 27     struct rb_node *right = node->rb_right;
 28     struct rb_node *parent = rb_parent(node);
 29 
 30     if ((node->rb_right = right->rb_left))
 31         rb_set_parent(right->rb_left, node);
 32     right->rb_left = node;
 33 
 34     rb_set_parent(right, parent);
 35 
 36     if (parent)
 37     {
 38         if (node == parent->rb_left)
 39             parent->rb_left = right;
 40         else
 41             parent->rb_right = right;
 42     }
 43     else
 44         root->rb_node = right;
 45     rb_set_parent(node, right);
 46 }
 47 
 48 static void __rb_rotate_right(struct rb_node *node, struct rb_root *root)
 49 {
 50     struct rb_node *left = node->rb_left;
 51     struct rb_node *parent = rb_parent(node);
 52 
 53     if ((node->rb_left = left->rb_right))
 54         rb_set_parent(left->rb_right, node);
 55     left->rb_right = node;
 56 
 57     rb_set_parent(left, parent);
 58 
 59     if (parent)
 60     {
 61         if (node == parent->rb_right)
 62             parent->rb_right = left;
 63         else
 64             parent->rb_left = left;
 65     }
 66     else
 67         root->rb_node = left;
 68     rb_set_parent(node, left);
 69 }
 70 
 71 void rb_insert_color(struct rb_node *node, struct rb_root *root)
 72 {
 73     struct rb_node *parent, *gparent;
 74 
 75     while ((parent = rb_parent(node)) && rb_is_red(parent))
 76     {
 77         gparent = rb_parent(parent);
 78 
 79         if (parent == gparent->rb_left)
 80         {
 81             {
 82                 register struct rb_node *uncle = gparent->rb_right;
 83                 if (uncle && rb_is_red(uncle))
 84                 {
 85                     rb_set_black(uncle);
 86                     rb_set_black(parent);
 87                     rb_set_red(gparent);
 88                     node = gparent;
 89                     continue;
 90                 }
 91             }
 92 
 93             if (parent->rb_right == node)
 94             {
 95                 register struct rb_node *tmp;
 96                 __rb_rotate_left(parent, root);
 97                 tmp = parent;
 98                 parent = node;
 99                 node = tmp;
100             }
101 
102             rb_set_black(parent);
103             rb_set_red(gparent);
104             __rb_rotate_right(gparent, root);
105         } else {
106             {
107                 register struct rb_node *uncle = gparent->rb_left;
108                 if (uncle && rb_is_red(uncle))
109                 {
110                     rb_set_black(uncle);
111                     rb_set_black(parent);
112                     rb_set_red(gparent);
113                     node = gparent;
114                     continue;
115                 }
116             }
117 
118             if (parent->rb_left == node)
119             {
120                 register struct rb_node *tmp;
121                 __rb_rotate_right(parent, root);
122                 tmp = parent;
123                 parent = node;
124                 node = tmp;
125             }
126 
127             rb_set_black(parent);
128             rb_set_red(gparent);
129             __rb_rotate_left(gparent, root);
130         }
131     }
132 
133     rb_set_black(root->rb_node);
134 }
135 
136 static void __rb_erase_color(struct rb_node *node, struct rb_node *parent,
137                  struct rb_root *root)
138 {
139     struct rb_node *other;
140 
141     while ((!node || rb_is_black(node)) && node != root->rb_node)
142     {
143         if (parent->rb_left == node)
144         {
145             other = parent->rb_right;
146             if (rb_is_red(other))
147             {
148                 rb_set_black(other);
149                 rb_set_red(parent);
150                 __rb_rotate_left(parent, root);
151                 other = parent->rb_right;
152             }
153             if ((!other->rb_left || rb_is_black(other->rb_left)) &&
154                 (!other->rb_right || rb_is_black(other->rb_right)))
155             {
156                 rb_set_red(other);
157                 node = parent;
158                 parent = rb_parent(node);
159             }
160             else
161             {
162                 if (!other->rb_right || rb_is_black(other->rb_right))
163                 {
164                     rb_set_black(other->rb_left);
165                     rb_set_red(other);
166                     __rb_rotate_right(other, root);
167                     other = parent->rb_right;
168                 }
169                 rb_set_color(other, rb_color(parent));
170                 rb_set_black(parent);
171                 rb_set_black(other->rb_right);
172                 __rb_rotate_left(parent, root);
173                 node = root->rb_node;
174                 break;
175             }
176         }
177         else
178         {
179             other = parent->rb_left;
180             if (rb_is_red(other))
181             {
182                 rb_set_black(other);
183                 rb_set_red(parent);
184                 __rb_rotate_right(parent, root);
185                 other = parent->rb_left;
186             }
187             if ((!other->rb_left || rb_is_black(other->rb_left)) &&
188                 (!other->rb_right || rb_is_black(other->rb_right)))
189             {
190                 rb_set_red(other);
191                 node = parent;
192                 parent = rb_parent(node);
193             }
194             else
195             {
196                 if (!other->rb_left || rb_is_black(other->rb_left))
197                 {
198                     rb_set_black(other->rb_right);
199                     rb_set_red(other);
200                     __rb_rotate_left(other, root);
201                     other = parent->rb_left;
202                 }
203                 rb_set_color(other, rb_color(parent));
204                 rb_set_black(parent);
205                 rb_set_black(other->rb_left);
206                 __rb_rotate_right(parent, root);
207                 node = root->rb_node;
208                 break;
209             }
210         }
211     }
212     if (node)
213         rb_set_black(node);
214 }
215 
216 void rb_erase(struct rb_node *node, struct rb_root *root)
217 {
218     struct rb_node *child, *parent;
219     int color;
220 
221     if (!node->rb_left)
222         child = node->rb_right;
223     else if (!node->rb_right)
224         child = node->rb_left;
225     else
226     {
227         struct rb_node *old = node, *left;
228 
229         node = node->rb_right;
230         while ((left = node->rb_left) != NULL)
231             node = left;
232 
233         if (rb_parent(old)) {
234             if (rb_parent(old)->rb_left == old)
235                 rb_parent(old)->rb_left = node;
236             else
237                 rb_parent(old)->rb_right = node;
238         } else
239             root->rb_node = node;
240 
241         child = node->rb_right;
242         parent = rb_parent(node);
243         color = rb_color(node);
244 
245         if (parent == old) {
246             parent = node;
247         } else {
248             if (child)
249                 rb_set_parent(child, parent);
250             parent->rb_left = child;
251 
252             node->rb_right = old->rb_right;
253             rb_set_parent(old->rb_right, node);
254         }
255 
256         node->rb_parent_color = old->rb_parent_color;
257         node->rb_left = old->rb_left;
258         rb_set_parent(old->rb_left, node);
259 
260         goto color;
261     }
262 
263     parent = rb_parent(node);
264     color = rb_color(node);
265 
266     if (child)
267         rb_set_parent(child, parent);
268     if (parent)
269     {
270         if (parent->rb_left == node)
271             parent->rb_left = child;
272         else
273             parent->rb_right = child;
274     }
275     else
276         root->rb_node = child;
277 
278  color:
279     if (color == RB_BLACK)
280         __rb_erase_color(child, parent, root);
281 }
282 
283 static void rb_augment_path(struct rb_node *node, rb_augment_f func, void *data)
284 {
285     struct rb_node *parent;
286 
287 up:
288     func(node, data);
289     parent = rb_parent(node);
290     if (!parent)
291         return;
292 
293     if (node == parent->rb_left && parent->rb_right)
294         func(parent->rb_right, data);
295     else if (parent->rb_left)
296         func(parent->rb_left, data);
297 
298     node = parent;
299     goto up;
300 }
301 
302 /*
303  * after inserting @node into the tree, update the tree to account for
304  * both the new entry and any damage done by rebalance
305  */
306 void rb_augment_insert(struct rb_node *node, rb_augment_f func, void *data)
307 {
308     if (node->rb_left)
309         node = node->rb_left;
310     else if (node->rb_right)
311         node = node->rb_right;
312 
313     rb_augment_path(node, func, data);
314 }
315 
316 /*
317  * before removing the node, find the deepest node on the rebalance path
318  * that will still be there after @node gets removed
319  */
320 struct rb_node *rb_augment_erase_begin(struct rb_node *node)
321 {
322     struct rb_node *deepest;
323 
324     if (!node->rb_right && !node->rb_left)
325         deepest = rb_parent(node);
326     else if (!node->rb_right)
327         deepest = node->rb_left;
328     else if (!node->rb_left)
329         deepest = node->rb_right;
330     else {
331         deepest = rb_next(node);
332         if (deepest->rb_right)
333             deepest = deepest->rb_right;
334         else if (rb_parent(deepest) != node)
335             deepest = rb_parent(deepest);
336     }
337 
338     return deepest;
339 }
340 
341 /*
342  * after removal, update the tree to account for the removed entry
343  * and any rebalance damage.
344  */
345 void rb_augment_erase_end(struct rb_node *node, rb_augment_f func, void *data)
346 {
347     if (node)
348         rb_augment_path(node, func, data);
349 }
350 
351 /*
352  * This function returns the first node (in sort order) of the tree.
353  */
354 struct rb_node *rb_first(const struct rb_root *root)
355 {
356     struct rb_node    *n;
357 
358     n = root->rb_node;
359     if (!n)
360         return NULL;
361     while (n->rb_left)
362         n = n->rb_left;
363     return n;
364 }
365 
366 struct rb_node *rb_last(const struct rb_root *root)
367 {
368     struct rb_node    *n;
369 
370     n = root->rb_node;
371     if (!n)
372         return NULL;
373     while (n->rb_right)
374         n = n->rb_right;
375     return n;
376 }
377 
378 struct rb_node *rb_next(const struct rb_node *node)
379 {
380     struct rb_node *parent;
381 
382     if (rb_parent(node) == node)
383         return NULL;
384 
385     /* If we have a right-hand child, go down and then left as far
386        as we can. */
387     if (node->rb_right) {
388         node = node->rb_right; 
389         while (node->rb_left)
390             node=node->rb_left;
391         return (struct rb_node *)node;
392     }
393 
394     /* No right-hand children.  Everything down and left is
395        smaller than us, so any 'next' node must be in the general
396        direction of our parent. Go up the tree; any time the
397        ancestor is a right-hand child of its parent, keep going
398        up. First time it's a left-hand child of its parent, said
399        parent is our 'next' node. */
400     while ((parent = rb_parent(node)) && node == parent->rb_right)
401         node = parent;
402 
403     return parent;
404 }
405 
406 struct rb_node *rb_prev(const struct rb_node *node)
407 {
408     struct rb_node *parent;
409 
410     if (rb_parent(node) == node)
411         return NULL;
412 
413     /* If we have a left-hand child, go down and then right as far
414        as we can. */
415     if (node->rb_left) {
416         node = node->rb_left; 
417         while (node->rb_right)
418             node=node->rb_right;
419         return (struct rb_node *)node;
420     }
421 
422     /* No left-hand children. Go up till we find an ancestor which
423        is a right-hand child of its parent */
424     while ((parent = rb_parent(node)) && node == parent->rb_left)
425         node = parent;
426 
427     return parent;
428 }
429 
430 void rb_replace_node(struct rb_node *victim, struct rb_node *new,
431              struct rb_root *root)
432 {
433     struct rb_node *parent = rb_parent(victim);
434 
435     /* Set the surrounding nodes to point to the replacement */
436     if (parent) {
437         if (victim == parent->rb_left)
438             parent->rb_left = new;
439         else
440             parent->rb_right = new;
441     } else {
442         root->rb_node = new;
443     }
444     if (victim->rb_left)
445         rb_set_parent(victim->rb_left, new);
446     if (victim->rb_right)
447         rb_set_parent(victim->rb_right, new);
448 
449     /* Copy the pointers/colour from the victim to the replacement */
450     *new = *victim;
451 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

### 1.3红黑树的测试文件(test.c)

![img](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  1 /**
  2  * 根据Linux Kernel定义的红黑树(Red Black Tree)
  3  *
  4  * @author skywang
  5  * @date 2013/11/18
  6  */
  7 
  8 #include <stdio.h>
  9 #include <stdlib.h>
 10 #include "rbtree.h"
 11 
 12 #define CHECK_INSERT 0    // "插入"动作的检测开关(0，关闭；1，打开)
 13 #define CHECK_DELETE 0    // "删除"动作的检测开关(0，关闭；1，打开)
 14 #define LENGTH(a) ( (sizeof(a)) / (sizeof(a[0])) )
 15 
 16 typedef int Type;
 17 
 18 struct my_node {
 19     struct rb_node rb_node;    // 红黑树节点
 20     Type key;                // 键值
 21     // ... 用户自定义的数据
 22 };
 23 
 24 /*
 25  * 查找"红黑树"中键值为key的节点。没找到的话，返回NULL。
 26  */
 27 struct my_node *my_search(struct rb_root *root, Type key)
 28 {
 29     struct rb_node *rbnode = root->rb_node;
 30 
 31     while (rbnode!=NULL)
 32     {
 33         struct my_node *mynode = container_of(rbnode, struct my_node, rb_node);
 34 
 35         if (key < mynode->key)
 36             rbnode = rbnode->rb_left;
 37         else if (key > mynode->key)
 38             rbnode = rbnode->rb_right;
 39         else
 40             return mynode;
 41     }
 42     
 43     return NULL;
 44 }
 45 
 46 /*
 47  * 将key插入到红黑树中。插入成功，返回0；失败返回-1。
 48  */
 49 int my_insert(struct rb_root *root, Type key)
 50 {
 51     struct my_node *mynode; // 新建结点
 52     struct rb_node **tmp = &(root->rb_node), *parent = NULL;
 53 
 54     /* Figure out where to put new node */
 55     while (*tmp)
 56     {
 57         struct my_node *my = container_of(*tmp, struct my_node, rb_node);
 58 
 59         parent = *tmp;
 60         if (key < my->key)
 61             tmp = &((*tmp)->rb_left);
 62         else if (key > my->key)
 63             tmp = &((*tmp)->rb_right);
 64         else
 65             return -1;
 66     }
 67 
 68     // 如果新建结点失败，则返回。
 69     if ((mynode=malloc(sizeof(struct my_node))) == NULL)
 70         return -1; 
 71     mynode->key = key;
 72 
 73     /* Add new node and rebalance tree. */
 74     rb_link_node(&mynode->rb_node, parent, tmp);
 75     rb_insert_color(&mynode->rb_node, root);
 76 
 77     return 0;
 78 }
 79 
 80 /* 
 81  * 删除键值为key的结点
 82  */
 83 void my_delete(struct rb_root *root, Type key)
 84 {
 85     struct my_node *mynode;
 86 
 87     // 在红黑树中查找key对应的节点mynode
 88     if ((mynode = my_search(root, key)) == NULL)
 89         return ;
 90 
 91     // 从红黑树中删除节点mynode
 92     rb_erase(&mynode->rb_node, root);
 93     free(mynode);
 94 }
 95 
 96 /*
 97  * 打印"红黑树"
 98  */
 99 static void print_rbtree(struct rb_node *tree, Type key, int direction)
100 {
101     if(tree != NULL)
102     {   
103         if(direction==0)    // tree是根节点
104             printf("%2d(B) is root\n", key);
105         else                // tree是分支节点
106             printf("%2d(%s) is %2d's %6s child\n", key, rb_is_black(tree)?"B":"R", key, direction==1?"right" : "left");
107 
108         if (tree->rb_left)
109             print_rbtree(tree->rb_left, rb_entry(tree->rb_left, struct my_node, rb_node)->key, -1);
110         if (tree->rb_right)
111             print_rbtree(tree->rb_right,rb_entry(tree->rb_right, struct my_node, rb_node)->key,  1); 
112     }   
113 }
114 
115 void my_print(struct rb_root *root)
116 {
117     if (root!=NULL && root->rb_node!=NULL)
118         print_rbtree(root->rb_node, rb_entry(root->rb_node, struct my_node, rb_node)->key,  0); 
119 }
120 
121 
122 void main()
123 {
124     int a[] = {10, 40, 30, 60, 90, 70, 20, 50, 80};
125     int i, ilen=LENGTH(a);
126     struct rb_root mytree = RB_ROOT;
127 
128     printf("== 原始数据: ");
129     for(i=0; i<ilen; i++)
130         printf("%d ", a[i]);
131     printf("\n");
132 
133     for (i=0; i < ilen; i++) 
134     {
135         my_insert(&mytree, a[i]);
136 #if CHECK_INSERT
137         printf("== 添加节点: %d\n", a[i]);
138         printf("== 树的详细信息: \n");
139         my_print(&mytree);
140         printf("\n");
141 #endif
142 
143     }
144 
145 #if CHECK_DELETE
146     for (i=0; i<ilen; i++) {
147         my_delete(&mytree, a[i]);
148 
149         printf("== 删除节点: %d\n", a[i]);
150         printf("== 树的详细信息: \n");
151         my_print(&mytree);
152         printf("\n");
153     }
154 #endif
155 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

rbtree.h和rbtree.c基本上是从Linux 3.0的Kernel中移植出来的。仅仅只添加了offestof和container_of两个宏，这两个宏在文章"[Linux内核中双向链表的经典实现](http://www.cnblogs.com/skywang12345/p/3562146.html)"中已经介绍过了，这里就不再重复说明了。 test.c中包含了两部分内容：一是，基于内核红黑树接口，自定义的一个结构体，并提供了相应的接口(添加、删除、搜索、打印)。二是，包含了相应的测试程序。