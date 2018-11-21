# go learning way

仅以此文档记录一些工作使用go的一些stuffs





### 优秀的go扩展包  
##### 1.networking     
- [valyala/fasthttp](https://github.com/valyala/fasthttp)  - 具有goroutine pool的http扩展

##### 2.goroutine pool
- [panjf2000/ants](https://github.com/panjf2000/ants/)  - 实现一个控制goroutine数量的goroutine pool  

### 优秀的go学习社区
- [Go 夜读](https://github.com/developer-learning/night-reading-go/)  - Go source reading and offline technical discussion every Thursday night.  

### 不错的学习文章
- [实现一个不会由于tcp过期时间导致连接断开的http2client](#httpclient)

### 其他学习资源
- [Markdown 编辑器推荐](https://github.com/wizardforcel/markdown-simple-world/blob/master/1.md)  
- [Markdown 基本语法](https://github.com/younghz/Markdown)  


###文章列表

##### <span id="httpclient">实现一个不会由于tcp过期时间导致连接断开的http2 client</span> 
由于在一本情况下，底层Tcp会在连接空闲2小时之后主动断开连接，而上层的http无法感知，所以就会造成请求失败的情况，这里普遍有三种解决方法：

- 失败重试
- 空间连接加过期时间
- 自建连接池并ping保活
- 修改源代码。在获取连接方法[getClientConn](https://github.com/golang/net/blob/master/http2/client_conn_pool.go#L16) for循环时ping/pong检测之后再返回，如果不行就抛弃此连接。这种方法需要对方服务器支持ping/pong功能

第一种情况很简单 所以这里就不讨论。   
第二种情况：
  在[golang.org/x/net/http2](https://godoc.org/golang.org/x/net/http2)包中，有一个[ConfigureTransport()](https://github.com/golang/net/blob/master/http2/transport.go#L125)方法。这个方法的作用是将http transport 转化成http2 transport。由于在http2包里的transport没有办法直接设置过期时间，所以很多人放弃了这个方法。但是在http transport里可以设置，使用此方法可以将一些http transport的参数赋给http2 transport。查看http2的源码不难发现，http2的transport也会用这些参数，只是开发人员采用了一共更优雅的过度方法。
  例子：

	tlsConfig := &tls.Config{
			Certificates: []tls.Certificate{cert},
		}
		var DialTLS = func(network, addr string) (net.Conn, error) {
			dialer := &net.Dialer{
				Timeout:   5 * time.Second,
				KeepAlive: 5 * time.Second,
			}
			return tls.DialWithDialer(dialer, network, addr, tlsConfig)
		}
		transportApnsConfInit := &http.Transport{
			DialTLS:               DialTLS,
			TLSHandshakeTimeout:   5 * time.Second,
			ResponseHeaderTimeout: 5 * time.Second,
			MaxIdleConns:          100,
			MaxIdleConnsPerHost:   100,
			IdleConnTimeout:       6600 * time.Second,
			TLSClientConfig:       tlsConfig,
		}
		http2.ConfigureTransport(transportApnsConfInit)
		clientApnsInit = &http.Client{Transport: transportApnsConfInit, Timeout: 4 * time.Second}
  这里将空闲连接的过期时间设置为6600秒<2小时。这里的参数配置应设置得更加适合项目而不是照搬   
第三种情况：   
 [blackbeans/apns](https://github.com/blackbeans/apns)     
 由于此问题是在开发苹果apns遇到的问题，所以上面这个包是一个apns推送的功能包    
 里面实现了一个自建的维护http2 ClientConn连接的连接池，同时并采用ping的方式进行保活。不赞同的是代码里用h2c来表示http2 ClientConn，但是h2c另有含义。

