# 事件循环-EventLoop-宏任务微任务

### 注意点

---

UI rendering的节点，因为这个是由浏览器自行判断决定的，但是只要执行UI rendering，它的节点是在执行完所有的microtask之后，下一个macrotask之前，紧跟着执行UI render。

### 执行一个JavaScript代码的具体流程

---

1. 执行全局Script同步代码，这些同步代码有一些是同步语句，有一些是异步语句（比如setTimeout等）；
2. 全局Script代码执行完毕后，调用栈Stack会清空；
3. 从微队列microtask queue中取出位于队首的回调任务，放入调用栈Stack中执行，执行完后microtask queue长度减1；
4. 继续取出位于队首的任务，放入调用栈Stack中执行，以此类推，直到直到把microtask queue中的所有任务都执行完毕。**注意，如果在执行microtask的过程中，又产生了microtask，那么会加入到队列的末尾，也会在这个周期被调用执行**；
5. microtask queue中的所有任务都执行完毕，此时microtask queue为空队列，调用栈Stack也为空；
6. 取出宏队列macrotask queue中位于队首的任务，放入Stack中执行；
7. 执行完毕后，调用栈Stack为空；
8. 重复第3-7个步骤；

### 分类

---

**宏队列，macrotask，也叫tasks。**一些异步任务的回调会依次进入macro task queue，等待后续被调用，这些异步任务包括：

- setTimeout
- setInterval
- I/O
- UI Rendering
- requestAnimationFrame（浏览器独有）
- setImmediate（node独有）

**微队列，microtask，也叫jobs。**另一些异步任务的回调会依次进入micro task queue，等待后续被调用，这些异步任务包括：

- Promise
- Object.observe
- MutationObserver
    - 通过异步+微任务的策略
- progress.nextTick（node独有）