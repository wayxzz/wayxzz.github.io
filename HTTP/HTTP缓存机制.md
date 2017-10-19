[TOC]

Web缓存大致可分为:数据库缓存,服务器端缓存(代理服务器缓存,CDN缓存),浏览器缓存.

浏览器缓存也包含很多内容: HTTP缓存, indexDB, cookie, localStorage等.

**术语**

- 缓存命中率: 从缓存中得到数据的请求数与所有请求数的比率.理想状态是越高越好;
- 过期内容: 超过设置的有效时间,被标记为"陈旧"的内容. 通常过期内容不能用于回复客户端的请求,必须重新向源服务器请求新的内容或者验证缓存的内容是否依然有效;
- 验证: 验证缓存中的内容是否仍然有效,验证通过的话刷新过期时间;
- 失效: 把内容从缓存中移除.当内容发生改变时必须移除失效的内容.

浏览器缓存主要是HTTP协议定义的缓存机制.HTML meta标签,例如

```html
<meta http-equiv="Pragma" content="no-cache">    
```

含义是让浏览器不缓存当前页面.但是代理服务器不解析HTML内容,一般应用广泛的是HTTP头信息控制缓存.

## HTTP头信息控制缓存

大致分为两种: 强缓存和协议缓存.强缓存如果命中缓存不需要和服务器端发生交互,而协商缓存不管是否命中都要和服务器端发生交互,强缓存优先级高于协商缓存.

匹配流程

![](https://user-gold-cdn.xitu.io/2017/10/12/6897e7bb30d063f5e7a0a36101568c7a?imageView2/0/w/1280/h/960/ignore-error/1)

## 强缓存

可以理解为无需验证的缓存策略.对强缓存来说,响应头中有两个字段Expires/Cache-Control来表明规则.

### Expires

Expires指缓存过期时间,超过了这个时间点就代表资源过期.由于使用具体时间,如果时间表示出错或者没有转换到正确的时区都可能造成缓存生命周期出错.并且Expires是HTTP/1.0的标准,现在更倾向于用HTTP/1.1中定义的Cache-Control.两个同时存在时也是Cache-Control的优先级更高.

### Cache-Control

Cache-control由多个字段组合而成,主要有以下几个取值:

1. **max-age** 指定一个时间长度,在这个时间段内缓存是有效的,单位是秒(s).例如设置 `Cache-Control:max-age=31536000` ,也就是缓存有效期为(31536000/24/360)天,第一次访问这个资源的时候,服务器端也返回了`Expires`字段,并且过期时间是一年后:
![](https://user-gold-cdn.xitu.io/2017/10/12/5f4dd5fb278b21b76da14ec25377db5e?imageView2/0/w/1280/h/960/ignore-error/1)

  在没有禁用缓存并且没有超过有效时间的情况下,再次访问这个资源就命中了缓存,不会向服务器端请求资源而是直接从浏览器缓存中取.
![](https://user-gold-cdn.xitu.io/2017/10/12/5f4dd5fb278b21b76da14ec25377db5e?imageView2/0/w/1280/h/960/ignore-error/1)

2. **s-maxage** 同`max-age`,覆盖`max-age`和`Expires`,但仅适用用共享缓存,在私有缓存中被忽略.
3. **public** 表明响应可以被任何对象(发送请求的客户端,代理服务器等)缓存.
4. **private** 表明响应只能被单个用户(可能是操作系统用户,浏览器用户)缓存,是非共享的,不能被代理服务器缓存.
5. **no-cache** 强制所有缓存了该响应的用户,在使用已缓存的数据前,发送带验证的请求到服务器.不是字面上的不缓存.
6. **no-store** 禁止缓存,每次请求都要向服务器重新获取数据.

## 协商缓存

缓存的资源到期了,并不意味着资源内容发生了改变,如果和服务器上的资源没有差异,实际上没有必要再次请求.客户端和服务端通过某种验证机制验证当前请求资源是否可以使用缓存.

浏览器第一次请求数据之后会将数据和响应头部的缓存标识存储起来。再次请求时会带上存储的头部字段，服务器端验证是否可用。如果返回 `304 Not Modified`，代表资源没有发生改变可以使用缓存的数据，获取新的过期时间。反之返回 `200` 就相当于重新请求了一遍资源并替换旧资源。

### Last-Modified/If-Modified-Since

`Last-Modified`: 服务器端资源最后修改的时间,响应头会带上这个标识.第一次请求后,浏览器会记录这个时间,再次请求时,请求头部带上`If-Modified-Since`即为之前记录下的时间.服务器端收到带`If-Modified-Since`的请求后会和资源的最后修改时间对比.若修改过返回最新资源,状态码`200`, 若没有修改过则返回`304`.

![](https://user-gold-cdn.xitu.io/2017/10/12/c785aa638c10f7adfe27492c82aa1e60?imageView2/0/w/1280/h/960/ignore-error/1)

**注意**:如果响应头中有 `Last-modified` 而没有 `Expire` 或 `Cache-Control` 时，浏览器会有自己的算法来推算出一个时间缓存该文件多久，不同浏览器得出的时间不一样，所以 `Last-modified` 要记得配合 `Expires/Cache-Control` 使用。


### Etag/If-None-Match

由服务器上生成一段hash字符串,第一次请求时响应头带上`ETag:abcd`,之后的请求带上`If-None-match:abcd`, 服务器检查ETag,返回304或200.

![](https://user-gold-cdn.xitu.io/2017/10/12/94f6230ea5be20e83eb69ada69ca1ee8?imageView2/0/w/1280/h/960/ignore-error/1)

### last-modified 和 Etag 区别

- 某些服务器不能精确得到资源的最后修改时间，这样就无法通过最后修改时间判断资源是否更新。
- Last-modified 只能精确到秒。
- 一些资源的最后修改时间改变了，但是内容没改变，使用 Last-modified 看不出内容没有改变。
- Etag 的精度比 Last-modified 高，属于强验证，要求资源字节级别的一致，优先级高。如果服务器端有提供 ETag 的话，必须先对 ETag 进行 Conditional Request。

**注意**：实际使用 `ETag/Last-modified` 要注意保持一致性，做负载均衡和反向代理的话可能会出现不一致的情况。计算 ETag 也是需要占用资源的，如果修改不是过于频繁，看自己的需求用 Cache-Control 是否可以满足。

## 选择 Cache-Control 的策略（摘自 Google Developers）

![](https://user-gold-cdn.xitu.io/2017/10/12/1b413b65743cdef241a426d75bf81555?imageView2/0/w/1280/h/960/ignore-error/1)

## 实际应用

回到实际应用上来，首先要明确哪些内容适合被缓存哪些不适合。

### 考虑缓存的内容

- css样式文件
- js文件
- logo和图标
- html文件
- 可以被下载的内容

### 一些不应该被缓存的内容

- 业务敏感的 GET 请求

可缓存的内容又分为几种不同的情况：

### 不经常改变的文件

> 给 max-age 设置一个较大的值，一般设置 max-age=31536000

比如引入的一些第三方文件、打包出来的带有 hash 后缀 css、js 文件。一般来说文件内容改变了，会更新版本号、hash 值，相当于请求另一个文件。

标准中规定 max-age 的值最大不超过一年，所以设成 max-age=31536000。至于过期内容，缓存区会将一段时间没有使用的文件删除掉。

有看到用对话的形式来描述这个过程，便仿照着试图更清晰地解释：

![](https://user-gold-cdn.xitu.io/2017/10/12/454acb384c3a5a8f81485a07fa63c983?imageView2/0/w/1280/h/960/ignore-error/1)

![](https://user-gold-cdn.xitu.io/2017/10/12/21e40026b5950ef6a0b2794f1cf91543?imageView2/0/w/1280/h/960/ignore-error/1)

### 可能经常需要变动的文件

> Cache-Control: no-cache / max-age=0

比如入口 index.html 文件、文件内容改变但名称不变的资源。选择 ETag 或 Last-Modified 来做验证，在使用缓存资源之前一定会去服务器端做验证，命中缓存时会比第一种情况慢一点点，毕竟还要发请求进行通信。

![](https://user-gold-cdn.xitu.io/2017/10/12/454acb384c3a5a8f81485a07fa63c983?imageView2/0/w/1280/h/960/ignore-error/1)

![](https://user-gold-cdn.xitu.io/2017/10/12/946aa2998724789889decf59c33f702b?imageView2/0/w/1280/h/960/ignore-error/1)
