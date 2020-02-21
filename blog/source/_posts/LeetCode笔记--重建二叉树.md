---
title: LeetCode笔记--重建二叉树
tags: 数据结构与算法
date: 2020-2-20 11:14:24
---

二叉树的遍历根据根节点与左右子节点的遍历顺序的不同分为三种：

* 前序遍历

  根左右：先遍历根节点，再左子树，再右子树（先从根节点开始，记录左节点直到没有）

  第一个为根节点

* 中序遍历

  左根右：先左子树，再根子树，再右子树（从树的最左边的节点开始遍历）

* 后序遍历

  左右根：先左子树，后右子树，再根节点

  最后一个为根节点

在遍历的时候，当父节点只有一个子节点时，依然要遵循以上三种遍历的先后顺序（没有该子节点则不写内容），以保证某一侧的子树（“左边的子树”或“右边的子树”）所有节点都被完全遍历，之后才可以根据遍历的规则切换到下一子树。

如如下子树：

```
      G
   /     \
  D       M
 / \     / \
A   F   H   Z
   /
  E
```
前序遍历：GDAFEMHZ

中序遍历：ADEFGHMZ

后续遍历：AEFDHZMG

# 常见应用

一般都是给定中序排序，再加上一个前序排序、后续排序来逆向生成二叉树。

根据之前的知识，此类题的解答思路一般为：

先根据前序排序、后续排序的特点，找到根节点，之后再根据找到的根节点将中序排序分为左、右子树两个部分。这样循环直到整个树的每个节点都被遍历完毕，完整的二叉树也会被建立起来。

我们以下面这个二叉树为例：

```
    1
   / \
  2   3
     / \
    4   5
```

使用代码表示如下：

```kotlin
fun main() {

    val preorder = intArrayOf(1,2,3,4,5)//前序遍历
    val inorder  = intArrayOf(2,1,4,3,5)//中序遍历
    val tree = buildTree(preorder, inorder)
    print(tree)

}

fun buildTree(preorder: IntArray, inorder: IntArray): TreeNode? {
    var tree : TreeNode? = null
    if (preorder.isNotEmpty()) {
        val root = preorder[0]//获取根节点
        val indexOfRoot = inorder.indexOf(root)//获取中序排序中根节点的坐标
        tree = TreeNode(root)
        //根据根节点坐标，将二叉树分为左、右两个子树
        val leftTree = inorder.copyOfRange(0, indexOfRoot)
        val rightTree = inorder.copyOfRange(indexOfRoot + 1, inorder.size)
        //将前序排序也分为左右两个子树的前序排序
        val leftPreOrder = preorder.copyOfRange(1, preorder.size).filter { leftTree.contains(it) }.toIntArray()
        val rightPreOrder = preorder.copyOfRange(1, preorder.size).filter { rightTree.contains(it) }.toIntArray()
        //再次分别循环分析左右两个子树的结构
        tree.left = buildTree(leftPreOrder, leftTree)
        tree.right = buildTree(rightPreOrder, rightTree)
    }

    return tree
}

class TreeNode(var `val`: Int) {
    var left: TreeNode? = null
    var right: TreeNode? = null
}
```

# 参考资料

https://www.jianshu.com/p/9e8922486154

[【直观算法】二叉树遍历算法总结](https://charlesliuyx.github.io/2018/10/22/[直观算法]树的基本操作/)

[知道中序和后序遍历，画二叉树和写出前序遍历 ](https://jingyan.baidu.com/album/cdddd41cb8d79753ca00e144.html?picindex=1)

[leetcode-重建二叉树](https://leetcode-cn.com/problems/zhong-jian-er-cha-shu-lcof/)
