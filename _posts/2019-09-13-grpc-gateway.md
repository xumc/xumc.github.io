---
layout: post
title: "grpc gateway设计"
date: 2019-09-13
---

现在很多微服务内部都会使用grpc作为微服务内部调用的方式。 然而在客户端（web，移动应用）很多还是停留在使用restfull接口上， 当然移动应用还是比较容易使用grpc来call后端服务的。 但是web就显得有些不太容易了。当然现在也有了grpc-web(https://github.com/grpc/grpc-web)这种方式，不过这种全局使用grpc的方式在设计的时候用的不是普遍。尤其对于以后系统的改造来说前端改造成call后端的grpc来还是比较困难的。 本文假设我们仅仅进行后端服务的改造，前端仍然使用restfull接口。
比较容易设计以下的后端grcp调用方案。

<div class="mermaid">
graph LR
B[浏览器]-->|HTTP|G[API Gateway]
G -->|HTTP| S1[Service 1内含有grpc-gateway lib]
G -->|HTTP| S2[Service 2 内含有grpc-gateway lib]
</div>
这种方式比较简单，后端的每一个service都使用grpc-gateway进行消息转换，将grpc message变成json或者json转变为grpc message。同时service之间使用grpc方式进行通信。 但是我们忽略了一些问题：
1. 我们的后端service同时提供两种接口，从设计上将不是一个很好的设计。尤其我们还有L7 Gateway的时候。 格式之间的转换应该放到gateway或者类似于gateway角色的地方更合适一些。
2. 有些service提供的grpc  API，我们是希望其他的内部Service来调用的，如何保证不被外部调用到？

所以提出了新的架构：

<div class="mermaid">
graph LR
B[浏览器]-->|HTTP|G[API Gateway]
G -->|GRPC| S1[Service 1]
G -->|GRPC| S2[Service 2]
</div>
但是如果把json和grpc message转换放到L7 gateway上会遇到另外的问题。 

1. 问题1， 假设我们后端服务以及gateway使用Golang来开发的话，每一次service上新加或者更新对外（相对于service之间相互调用来说）service grpc API的时候，我们都需要重新编译部署gateway，会比较奇怪。
2. 在第一种方式中，restfull接口定义和grpc 是在proto文件中进行mapping的，如果我们把格式转换放到gateway，如何做restful接口和grpc接口的对应？

市面上现在有解决此类问题的现有产品吗？

Kong: https://docs.konghq.com/1.3.x/getting-started/configuring-a-grpc-service/
gRPC (basic)
Kong 1.0 now supports primitive, passive, proxy-pass of gRPC traffic in addition to REST. For this initial iteration, no Kong plugins can be applied to manipulate gRPC traffic. gRPC is built on top of HTTP/2, and this initial support provides another option for Kong users looking to connect east-west traffic with low overhead and latency. This is particularly helpful in enabling Kong users to open more mesh deployments in hybrid environments.

ambassador
https://blog.getambassador.io/grpc-and-the-open-source-ambassador-api-gateway-510eaaf9a0e0?gi=73218108e37c

这些产品均没有找到进行json和grpc转换的功能。所以我们只能靠自己了。

针对问题1， 经过查看grpc的代码，找到了grpc的一个功能点. grpc 提供了一个接口来获取grpc proto相关的信息。 https://github.com/grpc/grpc/blob/master/src/proto/grpc/reflection/v1alpha/reflection.proto。 然后就找到了https://github.com/fullstorydev/grpcurl这个工具。 通过对grpcurl的远吗进行研究，实际上我们可以使用这种方式实现后端service主动暴露service列表给gateway。 实际上， 我们也可以基于此做grpc服务的服务发现（Service discovery).

架构也就可以变成这样子：
<div class="mermaid">
graph LR
B[浏览器]-->|HTTP|G[API Gateway]
SD[服务发现]--> G
SD[服务发现]--> S1
SD[服务发现]--> S2
G -->|GRPC| S1[Service 1]
G -->|GRPC| S2[Service 2]
</div>
下面就是编码的问题了。

