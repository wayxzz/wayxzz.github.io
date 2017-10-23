
## 背景

一个项目里遇到一个问题：nodejs负责该项目的api请求，其中一部分请求是经过一些验证和修饰之后转发给JAVA端的，另一部分是nodejs直接处理的请求。突然从什么时候开始，“似乎”毫无征兆地，nodejs服务开启一段时间之后，整站的负责转发的接口就完全“瘫痪”了，具体表现就是接口一直处于pending状态，直到超时。重启之后，刚开始时接口是ok的，但过了一段时间，就又突然pending了。现象描述到这儿，如果让你猜，你猜测会是什么原因呢？哈哈。

## 背景具体细节

关于转发，具体使用http.request来实现的，在转发的情景下，nodejs相当于就实现了一个代理服务器。代码大体如下

```js
var proxy = http.request(options, function (res) {
    res.pipe(client_res, {
      end: true
    });
});

client_req.pipe(proxy, {
    end: true
});
```

其中client_req, client_res分别表示nodejs接收到的http.IncomingMessage请求对象和用来返回数据给客户端的http.ServerResponse对象。proxy是利用http.request方法发起请求后得到的请求对象，回调中的res是该请求对应的响应对象。

我在想会不会是因为这代码太“原始”、太“底层”，所以才这么“脆弱”。于是后来我加足了错误处理程序。

```js
var proxy = http.request(options, function (res) {
    res.pipe(client_res, {
      end: true
    }).on('error', function (e) {
        // ...
    });
});

proxy.on('error', function (e) {
    // ...
});

client_req.pipe(proxy, {
    end: true
}).on('error', function (e) {
    // ...
});
```
但是事实证明没有解决问题，发生pending的现象时，上面没有任何一个错误回调被调用到。纠结了一段事件之后，我开始怀疑，是不是nodejs本身的局限性导致了这个问题的发生，因为你看，重启之后问题并不是立即发生，而是慢慢的，问题被“积累”出来了，往往这种现象，是遇到了非常棘手的问题。

## 转机

被这个问题纠结了一两天，一直找不到问题症结所在，直到发现了引文1里提到的一个关键点：http.request的options中有一个agent选项。研究了引文2，发现原来node从0.5.3版本引入了一个http.Agent类，是用来管理socket池的，它使得在高负载的情况下多个不同的请求能重用一个TCP连接，避免低效率的重复断开和连接。正常情况下，当没有了处于pending状态下的请求，也即所有请求都处理完毕之后，该http.Agent实例就会断开TCP连接。而我是后来才直到，这个正是导致问题的一个关键原因。

回到http.request(options, [callback])方法里，其中第一参数options下有个选项agent

```js
agent: Controls Agent behavior. When an Agent is used request will default to Connection: keep-alive. Possible values:
    - undefined (default): use global Agent for this host and port.
    - Agent object: explicitly use the passed in Agent.
    - false: opts out of connection pooling with an Agent, defaults request to Connection: close.
    
```

默认情况下，http.request是用一个全局的agent，即http.globalAgent来管理所有的连接和请求的。而在http.Agent类的说明下有这样一个属性

```js
agent.maxSockets
By default set to 5. Determines how many concurrent sockets the agent can have open per origin. Origin is either a 'host:port' or 'host:port:localAddress' combination.
```

“By default set to 5”，就是说，对于一个域名+端口的组合，一个Agent实例只能同时处理5个拥有这样组合的请求。假设当有5个请求因为种种原因没有及时得到响应，那么第6个请求就不会被http.request处理，这也是个关键点，后面会提到它引发了一种奇怪的现象。

根据引文1提到的尝试将agent设置成false就解决了问题这一情况，我也尝试将agent设置成false。然后，问题奇迹般地消失了！困扰我多日的问题终于解决了！！

## 问题继续

事实证明，我高兴得太早了。将agent改成false之后的第二天，又有用户反馈说出现了请求pending的情况。但是，我打开页面之后发现并没有再出现pending了，那为什么那个用户还是出现最初的问题？我尝试以那个账号登陆（其实是个项目是个供内部人使用的网站），果然出现了转发那部分的接口一直pending的问题。到这个时候我才意识到，之前可能是这个用户的请求导致了所有用户的转发部分的接口出现了问题，而在将agent设置成false之后，问题就限制在了这个单一的用户了。

那么，这个用户为什么会遇到这样的问题？这个用户很特殊么？

## 解决问题

经过debug之后发现，出现问题的用户，http.request请求其实已经发起了，但是后端的JAVA日志未显示有任何访问记录，理论上所有请求JAVA端都会有记录。那么难道是我nodejs这边没有成功发起请求？

说到这，我要提一个细节，用户请求nodejs的API，在转发给后端JAVA之前，会“添油加醋”，加上一些认证的参数，同时会将一部分nodejs能接触到的数据库所请求到的数据传给后端。

问题就出在这儿，出现问题的接口用的是GET请求，刚好那个用户所要“附带”的数据比较多，导致最终总的GET的url长度超过了一定的限度，于是，在接下来的某个环节上（我并不确定是本服务器、还是转发的目标服务器），这个“超限”的GET请求被阻止了，后端的JAVA服务因此就接收不到任何请求。但是响应又没有正常返回，于是nodejs那个发起请求的agent一直还在苦苦等待目标服务器的响应，殊不知流量已半途被截。

最终，在nodejs转发的环节上，将GET改为POST之后，一切才真正归于正常。

值得一提的是，用nodejs在改写客户端发来的POST请求的时候，我又碰到了一段“小插曲”。在发POST请求的时候，必须正确指定Content-Length头字段，因为发送的时候就是按照它指定的长度，指明多少就发多少。实际要发送的数据长度大于Content-Length的话，数据就会因截断而丢失！
