```java
//输入两棵二叉树A和B，判断B是不是A的子结构。(约定空树不是任意一个树的子结构) 
//
// B是A的子结构， 即 A中有出现和B相同的结构和节点值。 
//
// 例如: 给定的树 A: 
//
// 3 / \ 4 5 / \ 1 2 给定的树 B： 
//
// 4 / 1 返回 true，因为 B 与 A 的一个子树拥有相同的结构和节点值。 
//
// 示例 1： 
//
// 输入：A = [1,2,3], B = [3,1]
//输出：false
// 
//
// 示例 2： 
//
// 输入：A = [3,4,5,1,2], B = [4,1]
//输出：true 
//
// 限制： 
//
// 0 <= 节点个数 <= 10000 
//
// Related Topics 树 深度优先搜索 二叉树 👍 751 👎 0


//leetcode submit region begin(Prohibit modification and deletion)
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */

//leetcode submit region end(Prohibit modification and deletion)

```

```java
class Solution {
    public boolean isSubStructure(TreeNode A, TreeNode B) {
        if (A == null || B == null) {
            return false;
        }

        LinkedList<TreeNode> queue = new LinkedList<TreeNode>() {{
            add(A);
        }};
        // 目的是找到和 B 相等的节点
        while (!queue.isEmpty()) {
            TreeNode node = queue.removeFirst();
            // 如果找到和 B 相等的节点，开启判断流程
            if (node.val == B.val) {
                if (checkSubStructure(node, B)) {
                    return true;
                }
            }
            if (node.left != null) {
                queue.addLast(node.left);
            }
            if (node.right != null) {
                queue.addLast(node.right);
            }
        }

        return false;
    }

    public boolean checkSubStructure(TreeNode A, TreeNode B) {
        LinkedList<TreeNode> aQueue = new LinkedList<>();
        aQueue.addLast(A);
        LinkedList<TreeNode> bQueue = new LinkedList<>();
        bQueue.addLast(B);
        while (!bQueue.isEmpty()) {
            TreeNode aNode = aQueue.removeFirst();
            TreeNode bNode = bQueue.removeFirst();
            if ((aNode.val != bNode.val) ||
                    (bNode.left != null && aNode.left == null) ||
                    (bNode.right != null && aNode.right == null)) {
                return false;
            }
            if (bNode.left != null) {
                bQueue.add(bNode.left);
                aQueue.add(aNode.left);
            }
            if (bNode.right != null) {
                bQueue.add(bNode.right);
                aQueue.add(aNode.right);
            }
        }
        return true;
    }
}
```