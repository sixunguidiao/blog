# 题目

输入一个链表，输出该链表中倒数第 k 个结点。

# 思路1

先遍历一遍链表，将每一个节点都压入一个 stack 中，将 stack pop k 次即可得到倒数第 k 个节点。注意，这种方法要求 k 不能大于链表的长度，为此需要先求出链表的长度。

```java
public ListNode FindKthToTail(ListNode head, int k) {
    if (head == null || k < 1 || k > length(head)) return null;
    Stack<ListNode> stack = new Stack<>();
    ListNode curr = head;
    while (curr != null) {
        stack.push(curr);
        curr = curr.next;
    }
    for (int i = 0; i < k - 1; i++) {
        stack.pop();
    }
    return stack.peek();
}

private int length(ListNode head) {
    if (head == null) return 0;
    int len = 0;
    ListNode curr = head;
    while (curr != null) {
        len++;
        curr = curr.next;
    }
    return len;
}
```

# 思路2

用一个 slow 指针和一个 fast 指针，fast 从表头开始先向后移动 k - 1 个节点，之后两个指针再一起移动，当 fast移动到链表的最后一个节点时，slow 指示的刚好是链表中倒数第 k 个节点

```java
public ListNode FindKthToTail(ListNode head, int k) {
    if (head == null || k < 1) return null;
    ListNode fast = head;
    for (int i = 0; i < k; i++) {
        fast = fast.next;
    }
    ListNode slow = head;
    while (fast != null) {
        slow = slow.next;
        fast = fast.next;
    }
    return slow;
}
```

