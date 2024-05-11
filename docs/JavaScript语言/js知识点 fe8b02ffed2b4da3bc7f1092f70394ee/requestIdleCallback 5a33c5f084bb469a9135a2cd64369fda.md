# requestIdleCallback

**介绍：**

常用于做一些低优先级的任务。当一帧的末尾有空闲时间，才会执行回调函数。当前帧的运行时间小于刷新频率（16.6ms）,函数fn才会执行。否则，就推迟到下一帧，以此类推。如果timeout时间内都没有时间，fn则会强制执行。

etTimeout 中的代码会在指定的延迟时间后执行，而 requestIdleCallback 中的代码则会在浏览器处于空闲状态时执行。这意味着，requestIdleCallback 中的代码只会在当前任务完成后执行。

**应用场景：**

- 大量数据的处理、图像的压缩等
- 延迟执行一些较为耗时的任务，例如软件更新检查、日志上传等

**代码示例：**

```jsx
// 通过 requestIdleCallback 处理图片
function processImages(images) {
	// 如果浏览器不支持 requestIdleCallback，则使用 setTimeout 实现类似的效果
  const idleCallback = window.requestIdleCallback || (fn) => setTimeout(fn, 5); 

  images.forEach((img) => {
    idleCallback(() => {
      // 在浏览器处于空闲状态时处理图片
      // 如果图片数量太多并且阻塞了浏览器，则不处理在下一个空闲时状态进行处理
      if (img.inViewport && img.loaded && !img.processed) {
        const canvas = document.createElement('canvas');
        // 在 canvas 中处理图片
        // ...
        img.processed = true;
      }
    });
  });
}
```

**什么操作不适合放到 requestIdleCallback 的callback中：**

更新DOM，以及Promise的回调（会使帧超时）