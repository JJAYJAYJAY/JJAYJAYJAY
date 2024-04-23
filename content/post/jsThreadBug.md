+++
title = '多人聊天室的线程安全问题'
date = 2024-01-08T21:41:46+08:00
draft = false
categories = [
    "后端",
]

tags = [
    "javaScript",
    "KOA"
]
image = "/cover/cover3.webp"
+++

> 大一发现的一个线程bug，文章本来写在飞书云文档上，现在迁移至博客。

## 问题的发现

由于我们使用的是动态输入风格的聊天室(后端发送的消息会通过传输流一个字一个字的打印在前端界面上)，所以后端在向前端发送请求的时候，会长时间的占用某一用户的输入流。那么就会引发出一些问题，当服务端处理多条消息时是否会存在资源冲突问题？因为每一个线程不是瞬时占用某一资源，而是长久占用。

对此我们做出一下尝试：

1. 先发送一条较长的消息
2. 当发送的数据打印到一半时再次发送一条数据
3. 观察前端，返回结果如下

![Alt text](image/image.png)

果然我们两条消息的内容被混杂在了一起，导致了消息杂糅。
## 问题产生的原因分析
我们查看源码

```js
broadcastNewMessage = async (ctx) => {
    let request = ctx.req
    let response = ctx.res

    request.setEncoding("utf8");

    let body = "";
    for await (let chunk of request)
        body += chunk;

    response.writeHead(200).end();

    new Typer(body, this.write);
}
```

```js
class Typer {
    #message
    #write
    #index = 0
    
    constructor(msg, write) {
        this.#message = msg + '☺'
        this.#write = write
        this.interval = setInterval(this.#type, 100)
    }

    #type = () => {
        const str = this.#message[this.#index];
        if (!str) {
            clearInterval(this.interval)
            return
        }
        this.#write(str)
        this.#index++
    }
}

```
这是两个用于处理消息推送的关键函数，我们可以清晰的看到当后端收到数据的时候，后端会产生一个typer类的对象，专门用于处理该条数据。其中类中设置了一个回调函数，每0.1s向输入流写入一次。这就是问题产生的来源。typer在向输入流写入的时候并没有检验这个输入流是否被占用，导致了消息杂糅。

## 问题的解决

> 问题产生的本质还是在于**线程安全**。

### js的运行机制

js语言实际上是一种单线程语言，也就是说js其实每次只能执行一件任务，js利用promise类来实现伪并发操作。

这里挂上一篇与js运行机制有关的[文章](https://blog.csdn.net/weixin_51433264/article/details/131589264)。

根据这篇文章，我们可以更加好的解析问题产生的原因。由于setInterval函数，每0.1s会向前端发送一条数据。但是我们所提到，js中不存在所谓的并发操作，都是通过单线程来实现模拟。

所以我们可以得出结论，两条消息杂糅的原因是两个typer对象交替调用回调函数产生的！

这时候我们回去查看刚才的杂糅消息，我们发现，刚好我们的两条消息被交替地插入成了一条消息，我们是能够剥离出来的。这更加验证了交替输入这个结论。

![Alt text](image/image-1.png)

### 加锁保证线程安全

在一些高并发的编程语言中，例如java语言自己会提供锁来保证线程安全，但很可惜js做为单线程语言，其本身并不能提供锁来保证我们的线程安全。所以此时我们需要手动为其添加锁这个功能。

先给出以下有关加锁的代码实现

```js
class Server {
    #clients = []
    #clientsLock = false
    constructor() {
        new MyKoa(__dirname, __dirname)
            .get([
                ['/', this.index],
                ['/chat', this.acceptNewClient]
            ])
            .post(['/chat', this.broadcastNewMessage])
            .start(3080)
    }
    async index(ctx) {
        await ctx.render(`chatClient`)
    }
      write = (str) => {
      //解锁操作
        if(!str){
            this.#clientsLock = false;
            return
        }
        const event = SSE.mkEvent(str)
        this.#clients.forEach(client => client.write(event));
    }
    acceptNewClient = (ctx) => {
        new SSE(ctx, this.#clients)
    }
    broadcastNewMessage = async (ctx) => {
        let request = ctx.req
        let response = ctx.res

        request.setEncoding("utf8");

        let body = "";
        for await (let chunk of request)
            body += chunk;

        response.writeHead(200).end();
        // 每0.1秒检测一次锁有没有被释放掉
        while (this.#clientsLock) {
            await new Promise(resolve => setTimeout(resolve, 100))
        }
        this.#clientsLock = true;
        new Typer(body, this.write);
    }
}
```

```js
class Typer {
    #message
    #write
    #index = 0

    constructor(msg, write) {
        this.#message = msg + '☺'
        this.#write = write
        this.interval = setInterval(this.#type, 100)
    }

    #type = () => {
        const str = this.#message[this.#index];
        //这里先进行写入，在write函数里执行锁的解除
        //添加这一块
        this.#write(str)
        if (!str) {
            clearInterval(this.interval)
            return
        }

        this.#index++
    }
}
```

#### promise对象与resolve方法

promise对象有三种状态：Pending(进行中)、Resolve(已完成)、Reject（已失败）。

resolve方法：将该方法的所指向的promise对象的状态改为resolve。

await关键字: 此关键字只能在async函数内使用，等待一个promise对象执行完毕，在此期间这只会阻塞该异步函数的进行，异步函数之外的代码仍然会执行。

- 会不会发生两个函数同时检测这个锁有没有被释放的问题？

    当然不会，如果是java当然会存在这样的问题，但js是单线程语言，它的执行顺序永远是有先后的，所以这种情况不会发生。

- 这里的promise对象究竟是如何执行的？

    这里挂上一篇与异步处理有关的[文章](https://blog.csdn.net/qq_54075517/article/details/131706299)。

    有了以上这些，我们来理解一下这个promise对象是如何执行
```js
while (this.#clientsLock) {
    await new Promise(resolve => setTimeout(resolve, 100))
}
```

首先await——等待，等待一个promise对象执行完毕，在此之前此异步函数的代码会卡在这里不继续运行。

promise对象的构造函数接受一个执行器函数做为参数，其中这个执行器函数接受两个参数，它们都是函数，其中第一个是resolve函数（执行成功时调用），第二个是reject函数（执行失败时调用）。
这里我们只使用第一个函数。然后我们设置了一个setTimeout函数，这个函数接受两个参数，第一个是需要回调的函数，第二个是时间（意思是在多少秒之后执行我们的回调函数）。所以我们将resolve函数传给这个函数，并设置0.1s的回调。

那么在0.1s后我们才会执行resolve函数，将promise对象的状态改为resolve，使得await接受到promise的完成，继续线程的执行，进入下一次检测。

这样我们就实现了对锁的检测，每0.1s检测锁是否被释放。

- 为什么直接使用setTimeout函数不会有这样的效果？

    因为setTimeout不会阻塞线程的运行，他只是简单地让一个函数在多少秒后回调。

### 问题解决

按照上述修改之后我们再去进行同样的操作，我们会发现两条消息不再会杂糅，而是一条一条的逐个展示出来。通过对共享资源的加锁，完美解决问题。

## 问题引发的思考

### 加锁引发的服务器消息处理过慢问题

由于使用了锁，服务器每次只能处理一条消息，这便使得服务器失去了处理多条消息的能力，我们有没有办法来提高处理效率而消息达到前端时又不杂糅？

更改处理思路。我们可以思考是否可以给每个消息设计独立的输入流，在输入完毕后进行关闭即可？这样子也可以解决问题，前端只需要为每个消息设置独立的div标签展示即可。

另外一种处理方法是利用js的单线程机制，前面讲过是我们的消息是交替的插入成为了杂糅消息，但是这是有规律的！所以我们可以尝试通过前端程序，来对收到的消息进行剥离，来实现相同的效果。（但是实现难度还是过高，因为网络、消息数量、消息长度等等干扰因素没有被考虑进来，这个处理方法并不优秀。）
