# 题目

定义栈的数据结构，实现一个能够得到栈中最小元素的min函数（时间复杂度应为O（1））。

# 思路

用一个辅助栈stack_min，栈顶元素记录当前栈中所有元素的最小值。入栈时，如果入栈元素num小于stack_min的栈顶元素，则同时将num加入stack_min中；出栈时，原栈和stack_min同时执行出栈。

```java
class MyStack {
    Stack<Integer> stack_data = new Stack<>();
    Stack<Integer> stack_min = new Stack<>();

    // 入栈
    public void push(int num) {
        stack_data.push(num);
        if (stack_min.isEmpty() || num<stack_min.peek()) {
            stack_min.push(node);
        } else {
            stack_min.push(stack_min.peek());
        }
    }

    // 出栈
    public void pop() {
        if (!stack_data.isEmpty() && !stack_min.isEmpty()) {
            stack_data.pop();
            stack_min.pop();
        }
    }

    // 获取栈顶元素
    public int top() {
        return stack_data.peek();
    }

    // 获取当前栈中的最小元素
    public int min() {
        return stack_min.peek();
    }
}
```

