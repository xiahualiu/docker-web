+++
title = "LeetCode Linked List Question Summary"
date = 2024-11-09
draft = false
[taxonomies]
  tags = ["LeetCode", "C++", "Algorithm"]
[extra]
  toc = true
	keywords = "LeetCode, C++, Algorithm"
+++

After solved around 20 questions on leetcode about linked list, I think it is time to summarize everything I learned so far:

## Type Definition

It is usually defined as:

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode() : val(0), next(nullptr) {}
    ListNode(int _val) : val(_val), next(nullptr) {}
    ListNode(int _val, ListNode* _next) : val(_val), next(_next) {}
}
```

## Linked List Traversal

It is important to know whether you want to traverse a linked list in pre order or post order.

### Pre-Order Traversal

#### Iterative style

The logic is very simple:

```cpp
for (auto p = head; p != nullptr; p = p->next) {
  // Do something to p.
}
```

If we need to visit 2 nodes at once (like manipulating the link), use `prev` pointer to store the last visited node.

```cpp
ListNode* prev = nullptr;
for (auto p = head; p != nullptr; p = p->next) {
  // Do something to prev, p.
  prev = p;
}
```

#### Recursive style

```cpp
void do_something(ListNode* cur) {
    if (cur == nullptr) {
        return;
    } else {
        // Do something to cur
        do_something(cur->next);
    }
}
```

If 2 nodes are needed inside a loop:

```cpp
void do_something(ListNode* prev, ListNode* cur) {
    if (cur == nullptr) {
        // Do something at the end
        return;
    } else {
        // Do something to prev, cur
        do_something(cur, cur->next);
    }
}

do_something(nullptr, head);
```

### Post-Order Traversal

This type of traversal is only viable through either a `stack` or recursive functions.

We process the end node first, and go back to the head node one by one.

#### Iterative style

In order to interatively do the post-order traversal, a `stack` is needed at the beginning.

```cpp
auto next = stack<ListNode*>();
// First push all nodes to a stack
for (auto p = head; p != nullptr; p = p->next) {
    next.push(p);
}
// Pop one by one
while (!next.empty()) {
    auto p = next.top();
    next.pop();
    // Do something to p
}
```

If 2 node is needed, you can push a `nullptr` at the bottom, and check if `nullptr` is at the top during the loop:

```cpp
auto next = stack<ListNode*>({nullptr});
// First push all nodes to a stack
for (auto p = head; p != nullptr; p = p->next) {
    next.push(p);
}
// Pop one by one
while (next.top() != nullptr) {
    auto p = next.top();
    next.pop();
    // Do something to p and next.top()
}
```

#### Recursive style

You can also do the post order traversal in the recursive style:

```cpp
void do_something(ListNode* cur) {
    if (cur == nullptr) {
        return;
    } else {
        do_something(cur->next);
        // Do something to cur
    }
}
```

Notice the difference between the pre and post order traversal because they are very similar here.

For post-order traversal, we first call the recursive function, then do something to the current node.

2-node post-order traversal:

```cpp
void do_something(ListNode* prev, ListNode* cur) {
    if (cur == nullptr) {
        return;
    } else {
        do_something(cur, cur->next);
        // Do something to cur, cur->next
    }
}

do_something(nullptr, head);
```

## Example

A good example to use all of above things we dicussed is [LeetCode question 206: Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/).

### Pre-Order Traversal Solution

Because we need to visit 2 nodes at once:

#### Iterative Style

```cpp
ListNode* reverse_linked_list_pre_iter(ListNode* head) {
    ListNode* prev = nullptr;
    for (auto p = head; p != nullptr;) {
        auto next_tmp = p->next;
        p->next = prev;
        prev = p;
        p = next_tmp;
    }
    return prev;
}

ListNode* reverse_linked_list(ListNode* head) {
    return reverse_linked_list_pre_iter(head);
}
```

#### Recursive Style

```cpp
ListNode* reverse_linked_list_pre_recur(ListNode* prev, ListNode* cur) {
    if (cur == nullptr) {
        return prev;
    } else {
        auto next_tmp = cur->next;
        cur->next = prev;
        return reverse_linked_list_pre_recur(cur, next_tmp);
    }
}

ListNode* reverse_linked_list(ListNode* head) {
    return reverse_linked_list_pre_recur(nullptr, head);
}
```

### Post-Order Traversal Solution

#### Iterative Style

```cpp
ListNode* reverse_linked_list_post_iter(ListNode* head) {
    auto next = stack<ListNode*>({nullptr});
    for (auto p = head; p != nullptr; p = p->next) {
        next.push(p);
    }
    auto new_head = next.top();
    while (next.top() != nullptr) {
        auto cur = next.top();
        next.pop();
        auto prev = next.top();
        cur->next = prev;
    }
    return new_head;
}

ListNode* reverse_linked_list(ListNode* head) {
    return reverse_linked_list_post_iter(head);
}
```

#### Recursive Style

```cpp
void reverse_linked_list_post_recur(ListNode* prev, ListNode* cur, ListNode*& result) {
    if (cur == nullptr) {
        result = prev;
    } else {
        reverse_linked_list_post_recur(cur, cur->next, result);
        cur->next = prev;
    }
}

ListNode* reverse_linked_list(ListNode* head) {
    ListNode* result = nullptr;
    reverse_linked_list_post_recur(nullptr, head, result);
    return result;
}
``` 

## Create a Linked List

Always add a dummy node at the beginning for null case.