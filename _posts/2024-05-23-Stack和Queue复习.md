---
title: 2024-05-23-Stack和Queue复习.md

author: Vi_error

description: Stack和Queue复习

categories:

  - Java

tags:

  - [ algorithm, data structure, Java ]

---

复习一下栈和队列的操作。

栈就是后进先出，非常适合顺序反转的操作，树、链表的算法操作都经常需要使用。

队列就是先进先出，顺序暂存非常的方便，树中比较经常使用。

# Stack

最典型特征：

- 后进先出，最后进入的元素最先被处理

常用场景：

- 表达式求值与语法解析，例如中缀表达式转后缀表达式
- 深度优先搜索（DFS）算法中，用于存储路径
- 实现函数调用的递归（函数调用栈）
- 支持撤销操作的实现（例如文本编辑器的撤销功能）

可复习的算法题：

- 有效的括号匹配（Valid Parentheses）
- 最小栈（Min Stack）
- 每日温度（Daily Temperatures）
- 接雨水（Trapping Rain Water）

常用操作：

- push：将元素压入栈顶。时间复杂度O(1)。
- pop：将栈顶元素弹出。时间复杂度O(1)。
- peek（或top）：查看栈顶元素但不弹出。时间复杂度O(1)。
- isEmpty：检查栈是否为空。时间复杂度O(1)。

Stack的Java实现：
在Java中，Stack类是java.util包的一部分，继承自Vector类。尽管Stack类提供了所有基础操作，但它不是最推荐的实现方式，建议使用Deque接口的实现，例如ArrayDeque。

常用方法：

- push(E item)：将元素压入栈。
- pop()：移除并返回栈顶元素。
- peek()：返回栈顶元素但不移除。
- isEmpty()：检查栈是否为空。

Stack示例

```
Stack<Integer> stack = new Stack<>();
stack.push(1);
stack.push(2);
int top = stack.peek(); // 返回2
int removed = stack.pop(); // 返回2，并将其从栈中移除
boolean empty = stack.isEmpty(); // 检查栈是否为空
```

基于Deque的实现

```
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);
stack.push(2);
int top = stack.peek();
int removed = stack.pop();
boolean empty = stack.isEmpty();
```

# Queue

最典型特征

- 先进先出，最先进入的元素最先被处理

常用场景

- 宽度优先搜索（BFS）算法中，用于存储路径
- 操作系统中的任务调度
- 网络中的消息队列
- 实现缓存机制（例如LRU缓存）

可复习的算法题目

- 滑动窗口最大值（Sliding Window Maximum）
- 二叉树的层序遍历（Binary Tree Level Order Traversal）
- 设计循环队列（Design Circular Queue）

常用操作：

- enqueue：将元素加入队尾。时间复杂度O(1)。
- dequeue：将队头元素移除。时间复杂度O(1)。
- front：查看队头元素但不移除。时间复杂度O(1)。
- isEmpty：检查队列是否为空。时间复杂度O(1)。

在Java中，Queue是一个接口，常见的实现有LinkedList和ArrayDeque。PriorityQueue也是Queue的实现，但它的行为与普通的FIFO队列不同。

ps，PriorityQueue在很多算法题里面也是相当好用。

常用方法：

add(E e)/offer(E e)：将元素加入队尾。
remove()/poll()：移除并返回队头元素。
element()/peek()：返回队头元素但不移除。
isEmpty()：检查队列是否为空。

基于LinkedList的实现

```
Queue<Integer> queue = new LinkedList<>();
queue.offer(1);
queue.offer(2);
int front = queue.peek(); // 返回1
int removed = queue.poll(); // 返回1，并将其从队列中移除
boolean empty = queue.isEmpty(); // 检查队列是否为空
```

基于ArrayDeque的实现

```
Queue<Integer> queue = new ArrayDeque<>();
queue.offer(1);
queue.offer(2);
int front = queue.peek();
int removed = queue.poll();
boolean empty = queue.isEmpty();
```