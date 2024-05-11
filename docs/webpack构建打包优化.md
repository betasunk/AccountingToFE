# webpack构建打包优化

webpack学习的三大部分：

- 构建的整体流程
- Loaders的作用
- Plugins的架构与用法

### webpack 的代码分割策略

---

在 Webapck4.x 版本之前，我们都是使用 `CommonsChunkPlugin` 去做分离；webpack 4引入了 `optimization.splitChunks` 如果你的 `mode`是 `production`，那么 webpack4 就会自动开启 `CodeSplitting`。

它内置的代码分割策略是这样的：

- 新的 chunk 是否被共享或者是来自 `node_modules` 的模块
- 新的 chunk 体积在压缩之前是否大于 30kb
- 按需加载 chunk 的并发请求数量小于等于 5 个
- 页面初始加载时的并发请求数量小于等于 3 个

### webpack构建打包优化：

---

首先分析优化的几个方向，再讨论每个方向下的具体步骤和方法。需要注意的是，

1、测量构建打包的快慢大小：

- `speed-measure-webpack-plugin`：测量插件和loader的花费时间。exclude的优先级要高于include，尽量避免exclude，更倾向于用include

2、构建过程的优化—减少构建文件的数量：

- `IgnorePlugin`：作用是忽略第三方包指定目录。举个例子：
    
    ```jsx
    //moment会将所有本地化内容和核心功能一起打包，
    //可以用IgnorePlugin在打包师忽略本地化的内容
    //在使用时如果需要指定语言，可以手动引入语言包
    import 'moment/local/zh-cn'
    ```
    
- `module.noParse` 配置项可以让 Webpack 忽略对部分没采用模块化的文件的递归解析处理，这样做的好处是能提高构建性能。 原因是一些库，例如 jQuery 、ChartJS， 它们庞大又没有采用模块化标准，让 Webpack 去解析这些文件耗时又没有意义。
- `External和cdn`：利用二者配合减少打包的文件数量
    
    ```jsx
    module.exports = {
      ...,
      externals: {
        // key是我们 import 的包名，value 是CDN为我们提供的全局变量名
        // 所以最后 webpack 会把一个静态资源编译成：module.export.react = window.React
        "react": "React",
        "react-dom": "ReactDOM",
        "redux": "Redux",
        "react-router-dom": "ReactRouterDOM"
      }
    }
    ```
    
- 

3、构建过程的优化—多进程构建，加快构建的速度：

- `thread-loader`（webpack4内置官方推荐）：把这个Loader放在其他loader之前，放在这个Loader之后的Loader会在一个worker pool池里运行，每个单独进程处理时间上限为600ms。仅在耗时的Loader上使用。
    - 代码实例
        
        ```jsx
        //官方为了防止启动worker的高延迟，提供了对worker池的优化，预热功能。
        const threadLoader=require('thread-loader')
        const jsWorkerPool={
        	//产生的worker的数量，默认是CPU核心数-1，当require('os').cpus()是undefined时，数量则为1
        	workers:2,
        	//闲置时定时删除worker进程，默认500ms，
        	//可以设置无穷大，这样在监视模式下可以保持worker持续存在
        	poolTimeout:2000
        }
        
        const cssWorkerPool = {
        	//一个worker中并行执行工作的数量，默认是20
        	workerParallelJobs:2,
        	poolTimeout:2000
        }
        threadLoader.warmup(jsWorkerPool,['babel-loader'])
        threadLoader.warmup(cssWorkerPool ,['css-loader'])
        ```
        
- `happypack`：不推荐使用，已经内置在wp4中，开启线程也是耗时，包括线程间通信等，还有几个问题，MiniCssExtractPlugin 无法与 happypack 共存；MiniCssExtractPlugin 必须置于 `cache-loader` 执行之后，否则无法生效

4、从构建后文件的角度优化—添加缓存：

- `cache-loader`：在一些性能开销大的loader前添加`cache-loader`，缓存的文件默认保存在`node_modules/.cache/cache-loader`。在babel-loader中可以通过设置 [cacheDirectory](https://link.juejin.cn/?target=https%3A%2F%2Fwebpack.docschina.org%2Floaders%2Fbabel-loader%2F%23%25E9%2580%2589%25E9%25A1%25B9) 来开启缓存，这样，`babel-loader` 就会将每次的编译结果写进硬盘文件（默认是在项目根目录下的`node_modules/.cache/babel-loader`目录内，当然你也可以自定义）
- `bable配置的优化`：在不配置`@babel/plugin-transform-runtime`时，babel会使用很小的辅助函数来实现公共方法，他会被注入到每个需要他的文件中，这会导致js文件体积变大。我们可以使用这个配置减少代码重复注入。

5、从构建文件后的角度优化—减小文件体积，分割代码块：

- `terser-webpack-plugin`：代码压缩
- `splitChunks`：webpack4之前，我们处理公共模块的方式都是使用`CommonsChunkPlugin`，之后使用了内置的`splitChunks`。
    - 代码实例
        
        ```jsx
        splitChunks: {
              chunks: 'all',
              minSize: 10000, // 提高缓存利用率，这需要在http2/spdy
              maxSize: 0,//没有限制
              minChunks: 3,// 共享最少的chunk数，使用次数超过这个值才会被提取
              maxAsyncRequests: 5,//最多的异步chunk数
              maxInitialRequests: 5,// 最多的同步chunks数
              automaticNameDelimiter: '~',// 多页面共用chunk命名分隔符
              name: true,
              cacheGroups: {// 声明的公共chunk
                vendor: {
                 // 过滤需要打入的模块
                  test: module => {
                    if (module.resource) {
                      const include = [/[\\/]node_modules[\\/]/].every(reg => {
                        return reg.test(module.resource);
                      });
                      const exclude = [/[\\/]node_modules[\\/](react|redux|antd)/].some(reg => {
                        return reg.test(module.resource);
                      });
                      return include && !exclude;
                    }
                    return false;
                  },
                  name: 'vendor',
                  priority: 50,// 确定模块打入的优先级
                  reuseExistingChunk: true,// 使用复用已经存在的模块
                },
                react: {
                  test({ resource }) {
                    return /[\\/]node_modules[\\/](react|redux)/.test(resource);
                  },
                  name: 'react',
                  priority: 20,
                  reuseExistingChunk: true,
                },
                antd: {
                  test: /[\\/]node_modules[\\/]antd/,
                  name: 'antd',
                  priority: 15,
                  reuseExistingChunk: true,
                },
              },
            },
        ```