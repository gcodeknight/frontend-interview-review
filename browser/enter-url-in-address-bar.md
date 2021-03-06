# 过程详解

- 用户在地址栏输入一个`URL`, 点击一个链接, 提交表单或者是其他行为

- `DNS`查找

  - 原因: 用户在地址栏输入的是页面的`URL`, 需要去`DNS`服务器那里找到对应的`IP`

  - 需要用到的情况

    - 如果以前没有访问过该网址
    - 之前的`DNS`缓存失效了

  - 优化考虑的方面

    - 减少`DNS`请求数量
    - 缩短`DNS`请求时间

  - 优化点

    - 浏览器通过服务器名称请求`DNS`进行查找，最终返回一个`IP`地址，第一次初始化请求之后，这个`IP`地址可能会被缓存一段时间

    - 通过主机名加载一个页面通常仅需要`DNS`查找一次.。但是, `DNS`需要对不同的页面指向的主机名进行查找。如果`fonts`, `images`, `scripts`, `ads`, and `metrics` 都不同的主机名，`DNS`会对每一个进行查找 -> 减少`DNS`请求数量 -> 减少并行下载数量导致响应时间变慢

    - `dns-prefetch`

      ```html
      <link rel="dns-prefetch" href="//cdn.com" />	// 推荐放在<meta charset="UTF-8" />
      ```

      放在页面的顶部, 能够尽快在`DNS`服务器上面查询到对应的`IP`地址

      但是使用`DNS prefetch`会导致资源浪费, 因此可以在页面中禁止隐式的`DNS prefetch`

      ```html
      <meta http-equiv="x-dns-prefect-control" content="off" />
      ```

    - 使用`CDN`服务器进行优化

- `TCP Handshake`

  `TCP`三次握手建立链接

  这个机制的是用来让两端尝试进行通信—浏览器和服务器在发送数据之前，通过上层协议Https可以协商网络TCP套接字连接的一些参数

  ”三次握手“即”SYN-SYN-ACK“ - (SYN, SYN-ACK, ACK)发送了三个消息进行协商，开始一个TCP会话在两台电脑之间

  

# 部分疑问点
1. 为什么`JS`的解析会阻塞`DOM Tree`的解析

   因为`JS`的解析运行被默认为有可能使用`document.write`方法, `document.write`方法会向文档流里面写入字节, 这样的话, 会强制刷新剩下的文档

   所以为了避免重复操作, 就将剩下的`dom`解析给阻塞了

   而且在这时的`js`脚本能够操作前面已经解析完成的`dom`元素, 而不能操作后面还未解析的`dom`元素, 因为还未解析出来, `dom`元素还不存在

   `document.write`使用流程:

   1. `document.open`
   2. `document.write`
   3. `document.close`

   从另外一个角度理解, 当浏览器在解析DOM树的时候, 已经调用了`document.open`了, 那么在这种情况下, 使用`document.write`可以直接在文档中间的位置写入内容, 当文档解析完了触发`documentloaded`的时候, 浏览器会调用`document.close`。这个时候相当于写入文件完成并且管道关闭了，如果还要再进行`document.write`这个操作的时候, 需要重新调用`document.open`。因为是系统自动调用的， 系统不知道哪个`document`，所以就会重新生成一个空白的`document` , 并且把`document.write`里面要写入的内容写进去, 并且覆盖掉原来的`document`, 这样就会造成`documentloaded`之后调用`document.write`会将整个页面都覆盖掉。

2. JS script不同标签的行为(async, defer)

   async - 异步加载, 同步执行

   defer - 异步加载, onload之后执行

3. CSSOM解析会阻塞DOM Tree的解析或者JS的运行吗

这里共有两种情况：

1. 如果CSS外链标签在文件的最上方, 异步下载解析, 但是文件中间有一个`script`标签, 当整个文件解析到了script标签的时候, 假定CSS标签还未下载完成, 如果script标签里面有get ComptuedStyle的操作, CSS的解析就会阻塞JS的执行, 直到CSS解析完成了以后, 获取了正确的样式, 这样计算出来的结果才是正确的。
```html
<html>
  <head>
    <link src="./index.css" />
    <title>test app</title>
  </head>
  <body>
    <script src="./test.js"></script>
  </body>
</html>
```
2. 前面的情况同上, 唯一不同的点就是, script标签里面没有做get computed style的操作。这时，就和一般的操作是一样的。

4. 浏览器并行下载的最大数量

   各个浏览器的并行下载数量不同, 之前的最大是6个

   ```markdown
   Firefox 2:  2
   Firefox 3+: 6
   Opera 9.26: 4
   Opera 12:   6
   Safari 3:   4
   Safari 5:   6
   IE 7:       2
   IE 8:       6
   IE 10:      8
   Edge:       6
   Chrome:     6
   ```

5. 为什么需要document.write方法
以下内容摘自Stack Overflow
When document.write() is executed after page load, it replaces the entire header and body tag with the given parameter value in string. 

   - document.write (henceforth DW) does not work in XHTML
   - ~~DW does not directly modify the DOM, preventing further manipulation~~ *(trying to find evidence of this, but it's at best situational)*
   - DW executed after the page has finished loading will overwrite the page, or write a new page, or not work
   - DW executes where encountered: it cannot inject at a given node point
   - DW is effectively writing serialised text which is not the way the DOM works conceptually, and is an easy way to create bugs (.innerHTML has the same problem)

   The only seem appropriate usage for document.write() is when working third parties like Google Analytics and such for including their scripts. This is because document.write() is mostly available in any browser. Since third party companies have no control over the user’s browser dependencies (ex. jQuery), document.write() can be used as a fallback or a default method.(兼容性问题, 当需要加入google analytics等的脚本, 这是最容易加入依赖的方法)


6. render tree的行为, 如果CSS发生了改变以后

   repaint(重绘)和reflow(重排)

7. 阻塞渲染和阻塞解析

   阻塞渲染: CSS的解析会阻塞渲染 -> 基于不要让用户看到没有CSS, 即没有样式的页面, 但是也会有例外的情况, 有可能很久没有返回CSS, 这时, CSS一直无法解析, 那就有可能出现没有样式的页面, 但是最后接收到了页面CSS就会让重新刷新

   阻塞解析: JS的运行会阻塞DOM树解析

   并行下载: 最大数量下载量 -> 下载的时间和时机看代码的位置和配置