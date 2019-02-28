# 题目

给定一棵二叉搜索树，请找出其中第k大的节点，k从1开始计数。例如给定如下二叉搜索树：

```
     5
    / \
   3   7
  / \ / \
 2  4 6  8
```

按节点数值大小排序，第三个的节点应该是4。

# 思路1

由于二叉搜索树的中序遍历序列是递增的，因此模拟一遍二叉搜索树的中序遍历，每次记录当前节点的编号，当前节点的编号等于k时即找到了第k大的节点。

```java
private TreeNode result;	// 第k大的节点
private int index;    // 指示当前中序遍历到的节点的编号，从1开始

public TreeNode kthNode(TreeNode root, int k) {
    if (root == null || k < 1) return null;
    result = null;
    index = 1;
    kthNodeCore(root, k);
    return result;
}

private void kthNodeCore(TreeNode root, int k) {
    if (root == null) return;
    kthNodeCore(root.left, k);
    if (k == index++) result = root;
    kthNodeCore(root.right, k);
}
```

# 思路2

《Algorithm》中的方法。要求的是树中第k小的节点，假设我们把树中的节点按节点值从小到大排列并从0开始编号，首先求出根节点左子树的节点数t，如果t == k，说明当前节点就是第k小的节点；如果t > k，说明第k小的节点在根节点的左子树中，因此需要继续在左子树中寻找第k小的节点；如果t < k，说明第k小的节点在根节点的右子树中，且是根节点的第k - t - 1小的节点（-1是因为要减去根节点），因此继续在右子树中寻找第k - t - 1小的节点。

```java
public TreeNode kthNode(TreeNode root,int k) {
    if (root == null || k < 1) return null;
    return kthNodeCore(root, k - 1);   // 调用时传入的是k-1
}

private TreeNode kthNodeCore(TreeNode root, int k) {
    if (root == null) return null;
    int t = size(root.left);	// root左子树中的节点总数
    if (k < t) {
        return kthNodeCore(root.left, k);
    } else if (k > t) {
        return kthNodeCore(root.right, k-t-1);
    } else {
        return root;
    }
}

// 返回root中的节点总数
private int size(TreeNode root){
    if(root==null) return 0;
    return size(root.left)+size(root.right)+1;
}
```

如果以kthNodeCore(root, k)这种方式调用，则kthNodeCore()应相应修改为：

```java
private TreeNode kthNodeCore(TreeNode root, int k) {
    if (root == null) return null;
    int t = size(root.left);
    if (k < t + 1) {
        return kthNodeCore(root.left, k);
    } else if (k > t + 1) {
        return kthNodeCore(root.right, k-t-1);
    } else {
        return root;
    }
}
```

