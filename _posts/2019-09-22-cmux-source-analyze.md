---
layout: post
title: "cmux 源码解析"
date: 2019-09-22
---

一直以来，接触的很多服务都在同一个端口上提供了grpc和http的服务。一直都比较纳闷这个是怎么做到的。 今天有时间看了一下cmux的源码，原来核心逻辑还是比较简单的。

我们知道在服务器端程序中，我们往往会首先监听一个固定的服务器端端口，然后各个客户端（浏览器）访问这个固定端口来和服务器建立连接。 服务器接受到客户端的请求后，会使用一个动态端口和客户端建立tcp连接。这是tcp编程里面常见套路。grpc(http2)和http1作为tcp之上的协议， 完全可以走在同一个tcp连接上。在tcp协议看来，无论是grpc(http2)还是http1都是字节流而已。问题在于我们怎么才能发现某一段字节流是grpc(http2)还是http1. 这就涉及到grpc(http2)和http1协议的具体细节了，实际上，我们只要从字节流中抓取到符合grpc（http2）或者http1的协议的特色特征就可以区分了。cmux正是利用了这一点来区分grpc(http2), http1以及其他基于tcp的上层协议的。

我们以官方的example为例， 来说明cmux内部的工作原理。和以往的grpc或者http服务写法有些不同，我们需要使用cmux提供的Match方法来说明到底哪些特征是属于哪种协议的，我们也可以自定义这些特征。先看Match函数的输入与输出。从中可一旦到， Match函数的输入是一个或者多个Matcher函数， Matcher函数的输入是io.Reader输出是bool。从中我们就可以推测Match函数的逻辑了。 首先cmux会根据Matcher函数判断某一段字节流(io.Reader)是否满足Matcher函数，如果满足则返回一个listener对象。 这个listener对象实际上是已经建立的tcp连接之上的字节流的符合matcher特征的数据，在cmux中使用muxListener来标识。muxListener 实现了net.Listener接口，结合官方example来看， 变量grpcL, httpL和trpcL均属于muxListener类型的实例。当我们通过grpcS.Serve(grpcL), httpS.Serve(httpL)或者trpS.Accept时，我们都会调用muxListener上的Accept函数来”建立连接“。之所以在建立连接上加上引号，是因为这些并非真正的建立连接，而是使用前面提到的已经建立的连接(使用MuxConn标识)而已。
```
type Matcher func(io.Reader) bool
// Match returns a net.Listener that sees (i.e., accepts) only
// the connections matched by at least one of the matcher.
//
// The order used to call Match determines the priority of matchers.
Match(...Matcher) net.Listener

type muxListener struct {
   net.Listener
   connc chan net.Conn
}

func (l muxListener) Accept() (net.Conn, error) {
   c, ok := <-l.connc
   if !ok {
      return nil, ErrListenerClosed
   }
   return c, nil
}

// MuxConn wraps a net.Conn and provides transparent sniffing of connection data.
type MuxConn struct {
   net.Conn
   buf bufferedReader
}



// Create the main listener.
l, err := net.Listen("tcp", ":23456")
if err != nil {
	log.Fatal(err)
}

// Create a cmux.
m := cmux.New(l)

// Match connections in order:
// First grpc, then HTTP, and otherwise Go RPC/TCP.
grpcL := m.Match(cmux.HTTP2HeaderField("content-type", "application/grpc"))
httpL := m.Match(cmux.HTTP1Fast())
trpcL := m.Match(cmux.Any()) // Any means anything that is not yet matched.

// Create your protocol servers.
grpcS := grpc.NewServer()
grpchello.RegisterGreeterServer(grpcS, &server{})

httpS := &http.Server{
	Handler: &helloHTTP1Handler{},
}

trpcS := rpc.NewServer()
trpcS.Register(&ExampleRPCRcvr{})

// Use the muxed listeners for your servers.
go grpcS.Serve(grpcL)
go httpS.Serve(httpL)
go trpcS.Accept(trpcL)

// Start serving!
m.Serve()


func (m *cMux) serve(c net.Conn, donec <-chan struct{}, wg *sync.WaitGroup) {
   defer wg.Done()

   muc := newMuxConn(c)
   if m.readTimeout > noTimeout {
      _ = c.SetReadDeadline(time.Now().Add(m.readTimeout))
   }
   for _, sl := range m.sls {
      for _, s := range sl.ss {
         matched := s(muc.Conn, muc.startSniffing())
         if matched {
            muc.doneSniffing()
            if m.readTimeout > noTimeout {
               _ = c.SetReadDeadline(time.Time{})
            }
            select {
            case sl.l.connc <- muc:
            case <-donec:
               _ = c.Close()
            }
            return
         }
      }
   }

   _ = c.Close()
   err := ErrNotMatched{c: c}
   if !m.handleErr(err) {
      _ = m.root.Close()
   }
}
```

根据刚才的描述结合cmux源码，我们可以总结出以下的cmux数据流图。一旦有新的连接建立， cmux.Serve方法会首先判断这个连接传递数据的类型，然后将连接扔到sl.l.connc里面， 左侧的Grpc.Serve和Http.Serve方法中调用Accept的时候，实际上是从connc里面取出连接而已。

<div class="mermaid">
graph LR
A[Client or brower] -->|tcp 字节流| B[cmux.Serve]
B --> |http2 字节流|C[Grpc.Serve]
B --> |http1 字节流| D[Http.Serve]
</div>

那么cmux是如何做match的呢？
我们以cmux example中的HTTP1Fast Matcher为例，从代码可以看出，这个matcher比较直接暴力。采用了前缀匹配的方式。我们知道在http1中，request开头都是以methods开头的，形如：
```
GET /hello.txt HTTP/1.1
User-Agent: curl/7.16.3 libcurl/7.16.3 OpenSSL/0.9.7l zlib/1.2.3
Host: www.example.com
Accept-Language: en, mi
```
正如Http1Fast文档所说，这个方法并不能100%有效。只能说reqeust是对的上的。在Prefixmatcher实现中，为了加速匹配的速度， 使用了patricia tree算法。在此不再赘述算法细节。
```
// HTTP1Fast only matches the methods in the HTTP request.
//
// This matcher is very optimistic: if it returns true, it does not mean that
// the request is a valid HTTP response. If you want a correct but slower HTTP1
// matcher, use HTTP1 instead.
func HTTP1Fast(extMethods ...string) Matcher {
	return PrefixMatcher(append(defaultHTTPMethods, extMethods...)...)
}
```

可见， cmux设计的十分灵活， 我们不仅仅可以使用cmux默认提供的matcher，还是我们自定义matcher。甚至我们也可以使用cmux来对我们自己的私有协议进行端口复用。

