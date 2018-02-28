# 二叉树

## 前序遍历 

## 中序遍历  

## 后序遍历  

# 满二叉树

# 完全二叉树

# 平衡二叉树

# 二叉搜索树(Binary Search Tree) 
二叉搜索树(BST)也称为有序二叉树、排序二叉树。 
是指一棵空树或者具有下列性质的二叉树： 
1. 若任意节点的左子树不空，则左子树上所有结点的值均小于它的根结点的值 
2. 任意节点的右子树不空，则右子树上所有结点的值均大于它的根结点的值
3. 任意节点的左、右子树也分别为二叉查找树。
4. 没有键值相等的节点。

二叉查找树相比于其他数据结构的优势在于查找、插入的时间复杂度较低。为O(log n)。二叉查找树是基础性数据结构，用于构建更为抽象的数据结构，如集合、multiset、关联数组等。

这种特殊结构的二叉树，采用中序遍历即可获得一个有序数组。

## BST查找某个元素  
顾名思义，二叉搜索树很多时候用来进行数据查找。这个过程从树的根结点开始，沿着一条简单路径一直向下，直到找到数据或者得到NULL值。

![查找BST中某个元素](http://ww3.sinaimg.cn/large/7cc829d3gw1f62xl45leug20ci0aitbs.gif)

在二叉搜索树T中查找某个元素x的过程为：
1. 若T是空树，则搜索失败，否则2
2. 若x等于T的根节点的数据域之值，则查找成功；否则3：
3. 若x小于T的根节点的数据域之值，则搜索左子树；否则：
4. 查找右子树。

因此，整个查找过程就是从根节点开始一直向下的一条路径，若假设树的高度是h，那么查找过程的时间复杂度就是O(h)。
```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
 // 递归实现
 public TreeNode findNode(TreeNode root, int value) {
    if(root == null || root.val == value) {
        return root;
    }
    if(value < root.val) {
        return findNode(root.left, value);
    } else {
        return findNode(root.right, value);
    }
}

// 非递归
public TreeNode findNode(TreeNode root, int value) {
    while(root != null && root.val != value) {
        if(root.val > value) {
            root = root.left;
        } else {
            root = root.right;
        }
    }
    return root;
}
```
一般来说，迭代方式的效率比递归方式高很多。 


## 查找前驱/后继结点  
如果给定的一棵数所有的结点值都不相同那么：
前驱结点：结点值value小于查找结点值search_value的集合中最大值的结点   
后继结点：结点值value大于查找结点值search_value的集合中最小值的结点  

再具体一点应该是这样的： 

##### 前驱结点  
1. 若一个节点有左子树，那么该节点的前驱结点是其左子树中valu值**最大**的结点 
2. 若一个节点没有左子树，那么判断该结点和其父节点的关系  
 2.1 若该结点是其父节点的**右边**孩子，那么该结点的前驱结点即为其父节点  
 2.2  若该节点是其父节点的左边孩子，那么需要沿着其父亲节点一直向树的顶端寻找，直到找到一个节点P，P节点是其父节点Q的右边孩子

##### 后继结点   
1. 若一个节点有右子树，那么该节点的后继结点是其右子树中valu值**最小**的结点 
2. 若一个节点没有右子树，那么判断该节点和其父节点的关系  
 2.1 若该结点是其父节点的**左边**孩子，那么该结点的后继结点即为其父节点  
 2.2  若该节点是其父节点的右边孩子，那么需要沿着其父亲节点一直向树的顶端寻找，直到找到一个节点P，P节点是其父节点Q的左边孩子


```java
// 查找后继结点
public TreeNode findNextNode(TreeNode root, int value) {
	TreeNode pNode = null;
	// 1. 先找到对应value值的结点，假设一定能找到的
	TreeNode treeNode = getNode(root, value, pNode);
	// 2. 先判断有没有右子树
	if(treeNode.right != null) {
		return minNode(treeNode.right);
	}
	// 3. 找到父节点
	while(pNode != null && pNode.left != treeNode) {
		treeNode = pNode;
		treeNode = getNode(root, treeNode.val, pNode);
	}
	return pNode;
}
/**
 * 找到指定的结点和父结点
 * @param root	根结点
 * @param value 指定结点的value值
 * @param pNode 父结点
 * @return
 */
public TreeNode getNode(TreeNode root, int value, TreeNode pNode) {
	while(root != null && root.val != value) {
		pNode = root;
		if(root.val > value) {
			root = root.left;
		} else {
			root = root.right;
		}
	}
	return root;
}

public TreeNode minNode(TreeNode node) {
	while(node.left != null) {
		node = node.left;
	}
	return node;
}
```

# 相关资料 
[4 张 GIF 图帮助你理解二叉查找树](http://blog.jobbole.com/101366/)

[深入理解二叉搜索树（BST）](http://blog.csdn.net/u013405574/article/details/51058133)

# 红黑树
