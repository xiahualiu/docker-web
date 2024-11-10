+++
title = "LeetCode Binary Tree Question Summary"
date = 2024-11-09
draft = false
[taxonomies]
  tags = ["LeetCode", "C++", "Algorithm", "Binary Tree"]
[extra]
  toc = true
	keywords = "LeetCode, C++, Algorithm, Binary Tree"
+++

## Type Definition

It is usually defined as:

```cpp
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode() : val(0), left(nullptr), right(nullptr) {}
    TreeNode(int _val) : val(_val), left(nullptr), right(nullptr) {}
    TreeNode(int _val, TreeNode* _left, TreeNode* _right)
        : val(_val), left(_left), right(_right) {}
};
```

## Binary Tree Traversal

There are 3 traversal orders:

* Pre-order traversal.
* In-order traversal.
* Post-order traversal.

### Pre-Order Traversal

Pre-order traversal visits root first, then left node, then right node.

#### Iterative style

```cpp
void pre_order(TreeNode* root) {
    auto next = queue<TreeNode*>({root});
    while (!next.empty()) {
        auto cur = next.front();
        next.pop();
        // Do something to cur
        cout << cur->val << endl;
        if (cur->left != nullptr) {
            next.push(cur->left);
        }
        if (cur->right != nullptr) {
            next.push(cur->right);
        }
    }
}
```

#### Recursive style

```cpp
void pre_order(ListNode* cur) {
    if (cur == nullptr) {
        return;
    } else {
        // Do something to cur
        pre_order(cur->left);
        pre_order(cur->right);
    }
}
```

### In-Order Traversal

This type of traversal is only viable through either a `stack` or recursive functions.

We push left child to stack if left child is not `nullptr`, if left child is `nullptr`, push right child to the stack, and do something to current node and pop current node from stack.

#### Iterative style

In order to interatively do the post-order traversal, a `stack` is needed.

* if current node is not null, we push current node to stack, and go to left.
* if current node is null, we backtrack to top of the stack.
    * Do something to current node.
    * Go to right node.

```cpp
void in_order(TreeNode* root) {
    auto next = stack<TreeNode*>();
    auto cur = root;
    while (!next.empty() || cur != nullptr) {
        if (cur != nullptr) {
            next.push(cur);
            cur = cur->left;
        } else {
            cur = next.top();
            next.pop();
            // Do something to cur
            cur = cur->right;
        }
    }
}
```

#### Recursive style

```cpp
void in_order(TreeNode* cur) {
    if (cur == nullptr) {
        return;
    } else {
        in_order(cur->left);
        // Do something to cur
        in_order(cur->right);
    }
}
```

### Post-Order Traversal

#### Iterative style

TBD

#### Recursive style

```cpp
void post_order(TreeNode* cur) {
    if (cur == nullptr) {
        return;
    } else {
        post_order(cur->left);
        post_order(cur->right);
        // Do something to cur
    }
}
```

## Examples

### Pre-order Traversal Question

[LeetCode Question 112: Path Sum](https://leetcode.com/problems/path-sum/)

In order to get the path sum, you need to visit from parent to child nodes, this requires us to use pre order traversal.

Also we need to add leaf node check in the pre-order traversal algorithm when we return.

#### Iterative style

```cpp
bool hasPathSum(TreeNode* root, int targetSum) {
    auto next = stack<pair<TreeNode*, int>>({pair(root, targetSum)});
    while (!next.empty()) {
        auto cur = next.top();
        next.pop();
        if (cur.first == nullptr) {
            continue;
        }
        auto curSum = cur.second - cur.first->val;
        if (cur.first->left == nullptr && cur.first->right == nullptr) {
            if (curSum == 0) {
                return true;
            }
        }
        next.push(pair(cur.first->left, curSum));
        next.push(pair(cur.first->right, curSum));
    }
    return false;
}
```

#### Recursive style

```cpp
bool hasPathSum(TreeNode* root, int targetSum) {
        bool result = false;
        hasPathSumCore(root, targetSum, result);
        return result;
}

void hasPathSumCore(TreeNode* cur, int targetSum, bool& result) {
    if (result) {
        return;
    }
    if (cur == nullptr) {
        return;
    }
    targetSum = targetSum - cur->val;
    if (cur->left == nullptr && cur->right == nullptr) {
        if (targetSum == 0) {
            result = true;
            return;
        }
    }
    hasPathSum(cur->left, targetSum, result);
    hasPathSum(cur->right, targetSum, result);
}
```

### In-order Traversal Question

[530. Minimum Absolute Difference in BST](https://leetcode.com/problems/minimum-absolute-difference-in-bst/)

This problem requires us to do in-order traversal, and there is a key point of solving this question:

In-order traversal of a BST equals to visiting all elements on BST in sorted order.

#### Iterative style

```cpp
int getMinimumDifference(TreeNode* root) {
    auto next = stack<TreeNode*>();
    auto cur = root;
    auto last = -100000;
    auto result = INT_MAX;
    while(!next.empty() || cur != nullptr) {
        if (cur != nullptr) {
            next.push(cur);
            cur = cur->left;
        } else {
            cur = next.top();
            next.pop();
            result = min(result, cur->val - last);
            last = cur->val;
            cur = cur->right;
        }
    }
    return result;
}
```

#### Recursive style

```cpp
int getMinimumDifference(TreeNode* root) {
    int mind = INT_MAX;
    int last = -100000;
    inorderTraversal(root, last, mind);
    return mind;
}

void inorderTraversal(TreeNode* root, int& last, int& mind) {
    if (root == nullptr) {
        return;
    } else {
        inorderTraversal(root->left, last, mind);
        mind = min(mind, root->val - last);
        last = root->val;
        inorderTraversal(root->right, last, mind);
    }
}
```

### Post-order Traversal Question

[LeetCode Question 236: Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/)

This is a typical post-order traversal question. You need to work from the leaf nodes up to the parents nodes.

```cpp
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (root == nullptr) {
        return nullptr;
    }
    if (root == p || root == q) {
        return root;
    }

    auto left = lowestCommonAncestor(root->left, p, q);
    auto right = lowestCommonAncestor(root->right, p, q);

    // pass result upwards
    if (left != nullptr && right != nullptr) {
        return root;
    } else if (left != nullptr) {
        return left;
    } else if (right != nullptr) {
        return right;
    } else {
        return nullptr;
    }
}
```
