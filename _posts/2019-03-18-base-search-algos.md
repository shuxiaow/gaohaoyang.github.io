---
layout: post
title:  "基础搜索算法"
categories: algo
tags: search-algo
author: ShuxiaoW
---

* content
{:toc}

## 1 二叉搜索树

基础API：

```cpp
bool get(Key key, Value& val); // 查找key对应的value
bool put(Key key, Value val); // 插入或更新key对应的value
bool remove(Key key); // 删除指定key
```

### 查找

二叉搜索树本身是维护了有序性：一个节点的值总是大于左子树所有节点的值，小于右子树所有节点的值。

此外，二叉搜索树本身包含了递归的思想：任何一棵树，都是由 根节点 + 左子树 + 右子树 构成。

```cpp
Node* get(Node* node, Key key)
{
    if(node == NULL)
        return NULL;
    if(node->key == key)
        return node;
    if(node->key < key)
        return get(node->right, key);
    else
        return get(node->left, key);
}
```

### 删除

二叉树的删除操作：

- 如果删除的节点是叶子节点，直接删除即可；
- 如果删除的节点只有一个子树（左或者右），直接将子树提上来即可；
- 如果删除的节点有两个子树，那么将它的后继节点，也就是右子树的最小节点提上来，然后将其从右子树删除即可。

注：将一个节点A提到节点B的位置，只需要将A的value赋值到B即可，然后将节点A删除即可。如果是要将节点B的父节点指向节点A来替换的话，那必须修改节点B的父节点的信息，比较麻烦。

如何删除一个树最小的节点：

```cpp
// 删除node为根节点的树中最小的节点，并返回新的根节点
Node* deleteMin(Node* node)
{
    if(!node)
        return NULL;

    if(node->left) {
        node->left = deleteMin(node->left);
    } else {
        Node* right = node->right;
        delete node;
        node = node->right;
    }
    return node;
}
```

## 2 红黑树

红黑树是2-3-4树的等价。一颗2-3-4树有如下性质：

- 每个节点最多可以存储3个key，并按顺序排列；
- 所有的叶子节点（没有儿子节点的节点）都在同一个level（到树根的距离）；
- 所有的非叶子节点可能是：
    - 2-节点：存储1个key，有两个（非空的）儿子节点；
    - 3-节点：存储2个key，有3个（非空的）儿子节点；
    - 4-节点：存储3个key，有4个（非空的）儿子节点。
- 所有的叶子节点可能有1，2或者3个key，没有儿子节点。

可以看到，2-3-4树的性质保证了，查找任何一个key，其最大比较次数为O(logN)。但是2-3-4树实现起来比较麻烦，需要定义3种类型的node。

红黑树通过将二叉树中的节点着色，就可以等价表示任何一颗2-3-4树。只有在删除或插入时需要维护颜色信息以保证2-3-4树的平衡，在查找时它就是一颗普通的二叉树。所以在工程上，一般是使用红黑树来替换2-3-4树。

### 红黑树与2-3-4树的等价表示

在红黑树中，一个红色的节点，表示它与它的父节点在同一个2-3-4树的节点中。所以，2-节点，3-节点，4-节点在红黑树中分别表示如下：

![234tree_to_rbtree](https://raw.githubusercontent.com/shuxiaow/pictures/master/base_algos/234tree_to_rbtree.png)

所以，根据2-3-4树的性质，我们可以得出红黑树需要具备如下性质：

- 首先，它是一颗二叉搜索树，每个节点最多存储1个key，最多可以有1个或2个子节点；
- 每个节点要么是黑色的，要么是红色的；
- 根节点是黑色的，NIL节点按黑色算；
- 如果一个节点是红色的，那么它的儿子节点必须是黑色的；
- 每个*NIL*节点到根节点的路径上，黑色节点的个数必须相等；

### 插入

详见[红黑树中的插入操作](https://shuxiaow.github.io/2019/03/18/rbtree-insert)

### 删除

详见[红黑树中的删除操作](https://shuxiaow.github.io/2019/03/18/rbtree-delete)
