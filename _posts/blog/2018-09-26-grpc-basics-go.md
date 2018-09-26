---
layout: post
title: Go使用gRPC的入门指南
categories: 分布式
description: Go版本单例模式
keywords: 分布式
---

本篇文章是基于[gRPC官网](https://grpc.io/docs/tutorials/basic/go.html)翻译。代码较多，建议在电脑上浏览。

本教程提供了 Go 程序员如何使用 gRPC 的指南。请在阅读本篇文章之前保证已经正确安装最新的Go版本。

通过学习教程中例子，你可以学会如何：

* 在一个 .proto 文件内定义服务。
* 用 protocol buffer 编译器生成服务器和客户端代码。
* 使用 gRPC 的 Go API 为你的服务实现一个简单的客户端和服务器。
假设你已经阅读了概览 并且熟悉protocol buffers。 注意，教程中的例子使用的是 protocol buffers 语言的 proto3 版本。

## 为什么使用 gRPC?
我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互路由信息，如服务器和其他客户端的流量更新。

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑—— gRPC 帮你解决了不同语言及环境间通信的复杂性.使用 protocol buffers 还能获得其他好处，包括高效的序列号，简单的 IDL 以及容易进行接口更新。

## 自带例子说明

这篇文章只关注 Go 版本 gRPC，通过运行下面的命令去克隆grpc-go代码库：

```go
$ go get google.golang.org/grpc
```
⚠️注意：由于在国内访问google有限制，我们直接可以到[github](https://github.com/grpc/grpc-go)直接clone项目，然后将grpc-go的所有文件拷贝到`$GOPATH/src/google.golang.org/` 目录下即可。

然后改变当前的目录到 route_guide:

```go
$ cd $GOPATH/src/google.golang.org/grpc/examples/route_guide
```


### 定义服务
我们的第一步是使用 protocol buffers去定义 gRPC service 和方法 request 以及 response 的类型。你可以在`examples/route_guide/routeguide/route_guide.proto`看到完整的 .proto 文件。

要定义一个服务，你必须在你的 .proto 文件中指定 service：

```go
service RouteGuide {
   ...
}
```
然后在你的服务中定义 rpc 方法，指定请求的和响应类型。gRPC 允许你定义4种类型的 service 方法，这些都在 RouteGuide 服务中使用：

#### 1.简单 RPC

客户端使用存根发送请求到服务器并等待响应返回，就像平常的函数调用一样。

```go
   // Obtains the feature at a given position.
   rpc GetFeature(Point) returns (Feature) {}
```
#### 2.服务器端流式 RPC 

客户端发送请求到服务器，拿到一个流去读取返回的消息序列。 客户端读取返回的流，直到里面没有任何消息。从例子中可以看出，通过在 响应 类型前插入 stream 关键字，可以指定一个服务器端的流方法。

```go
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
  
```
 
#### 3.客户端流式 RPC 

客户端写入一个消息序列并将其发送到服务器，同样也是使用流。一旦客户端完成写入消息，它等待服务器完成读取返回它的响应。通过在 请求 类型前指定 stream 关键字来指定一个客户端的流方法。

```go
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
  
```
  
#### 4.双向流式 RPC
双方使用读写流去发送一个消息序列。两个流独立操作，因此客户端和服务器可以以任意喜欢的顺序读写：比如， 服务器可以在写入响应前等待接收所有的客户端消息，或者可以交替的读取和写入消息，或者其他读写的组合。 每个流中的消息顺序被预留。你可以通过在请求和响应前加 stream 关键字去制定方法的类型。

```go
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

我们的 .proto 文件也包含了所有请求的 protocol buffer 消息类型定义以及在服务方法中使用的响应类型——比如，下面的Point消息类型：

```go
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

### 生成客户端和服务器端代码

接下来我们需要从 .proto 的服务定义中生成 gRPC 客户端和服务器端的接口。我们通过 protocol buffer 的编译器 protoc 以及一个特殊的 gRPC Go 插件来完成。

简单起见，我们提供一个 bash 脚本 帮你用合适的插件，输入，输出去运行 protoc(如果你想自己去运行，确保你已经安装了 protoc，并且请遵循下面的 gRPC-Go 安装指南)来操作：

```go
$ codegen.sh route_guide.proto
```

实际上运行的是：

```go
$ protoc --go_out=plugins=grpc:. route_guide.proto
```

运行这个命令可以在当前目录中生成下面的文件：

`route_guide.pb.go`

这些包括：

* 所有用于填充，序列化和获取我们请求和响应消息类型的 protocol buffer 代码
* 一个为客户端调用定义在RouteGuide服务的方法的接口类型（或者 存根 ）
* 一个为服务器使用定义在RouteGuide服务的方法去实现的接口类型（或者 存根 ）

### 创建服务器

首先来看看我们如何创建一个 RouteGuide 服务器。如果你只对创建 gRPC 客户端感兴趣，你可以跳过这个部分，直接到创建客户端 (当然你也可能发现它也很有意思)。

让 RouteGuide 服务工作有两个部分：

实现我们服务定义的生成的服务接口：做我们的服务的实际的“工作”。
运行一个 gRPC 服务器，监听来自客户端的请求并返回服务的响应。
你可以从`grpc/examples/route_guide/server/server.go`看到我们的 RouteGuide 服务器的实现代码。现在让我们近距离研究它是如何工作的。

#### 实现RouteGuide

我们可以看出，服务器有一个实现了生成的 RouteGuideServer 接口的 routeGuideServer 结构类型：

```go
type routeGuideServer struct {
        ...
}
...

func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
        ...
}
...

func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
        ...
}
...

func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
        ...
}
...

func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
        ...
}
...
```
以下是对应在.proto中定义的4种service方法的服务端实现：

#### 1.简单 RPC
routeGuideServer 实现了我们所有的服务方法。首先让我们看看最简单的类型 GetFeature，它从客户端拿到一个 Point 对象，然后从返回包含从数据库拿到的feature信息的 Feature.

```go
func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
    for _, feature := range s.savedFeatures {
        if proto.Equal(feature.Location, point) {
            return feature, nil
        }
    }
    // No feature was found, return an unnamed feature
    return &pb.Feature{"", point}, nil
}
```
该方法传入了 RPC 的上下文对象，以及客户端的 Point protocol buffer请求。它返回了一个包含响应信息和error 的 Feature protocol buffer对象。在方法中我们用适当的信息填充 Feature，然后将其和一个nil错误一起返回，告诉 gRPC 我们完成了对 RPC 的处理，并且 Feature 可以返回给客户端。

#### 2.服务器端流式 RPC
现在让我们来看看我们的一种流式 RPC。 ListFeatures 是一个服务器端的流式 RPC，所以我们需要将多个 Feature 发回给客户端。

```go
func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
    for _, feature := range s.savedFeatures {
        if inRange(feature.Location, rect) {
            if err := stream.Send(feature); err != nil {
                return err
            }
        }
    }
    return nil
}
```
如你所见，这里的请求对象是一个 Rectangle，客户端期望从中找到 Feature，这次我们得到了一个请求对象和一个特殊的RouteGuide_ListFeaturesServer来写入我们的响应，而不是得到方法参数中的简单请求和响应对象。

在这个方法中，我们填充了尽可能多的 Feature 对象去返回，用它们的 Send() 方法把它们写入 RouteGuide_ListFeaturesServer。最后，在我们的简单 RPC中，我们返回了一个 nil 错误告诉 gRPC 响应的写入已经完成。如果在调用过程中发生任何错误，我们会返回一个非 nil 的错误；gRPC 层会将其转化为合适的 RPC 状态通过线路发送。

#### 3.客户端流式 RPC

现在让我们看看稍微复杂点的东西：客户端流方法 RecordRoute，我们通过它可以从客户端拿到一个 Point 的流，其中包括它们路径的信息。如你所见，这次这个方法没有请求参数。相反的，它拿到了一个 RouteGuide_RecordRouteServer 流，服务器可以用它来同时读和写消息——它可以用自己的 Recv() 方法接收客户端消息并且用 SendAndClose() 方法返回它的单个响应。

```go
func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
    var pointCount, featureCount, distance int32
    var lastPoint *pb.Point
    startTime := time.Now()
    for {
        point, err := stream.Recv()
        if err == io.EOF {
            endTime := time.Now()
            return stream.SendAndClose(&pb.RouteSummary{
                PointCount:   pointCount,
                FeatureCount: featureCount,
                Distance:     distance,
                ElapsedTime:  int32(endTime.Sub(startTime).Seconds()),
            })
        }
        if err != nil {
            return err
        }
        pointCount++
        for _, feature := range s.savedFeatures {
            if proto.Equal(feature.Location, point) {
                featureCount++
            }
        }
        if lastPoint != nil {
            distance += calcDistance(lastPoint, point)
        }
        lastPoint = point
    }
}
```

在方法体中，我们使用 RouteGuide_RecordRouteServer 的 Recv() 方法去反复读取客户端的请求到一个请求对象（在这个场景下是 Point），直到没有更多的消息：服务器需要在每次调用后检查 Read() 返回的错误。如果返回值为 nil，流依然完好，可以继续读取；如果返回值为 io.EOF，消息流结束，服务器可以返回它的 RouteSummary。如果它还有其它值，我们原样返回错误，gRPC 层会把它转换为 RPC 状态。

#### 4.双向流式 RPC

最后，让我们看看双向流式 RPC RouteChat()。

```go
func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
    for {
        in, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        key := serialize(in.Location)
                ... // look for notes to be sent to client
        for _, note := range s.routeNotes[key] {
            if err := stream.Send(note); err != nil {
                return err
            }
        }
    }
}
```

这次我们得到了一个 RouteGuide_RouteChatServer 流，和我们的客户端流的例子一样，它可以用来读写消息。但是，这次当客户端还在往 它们 的消息流中写入消息时，我们通过方法的流返回值。

这里读写的语法和客户端流方法相似，除了服务器会使用流的 Send() 方法而不是 SendAndClose()，因为它需要写多个响应。虽然客户端和服务器端总是会拿到对方写入时顺序的消息，它们可以以任意顺序读写——流的操作是完全独立的。

#### 启动服务器

一旦我们实现了所有的方法，我们还需要启动一个gRPC服务器，这样客户端才可以使用服务。下面这段代码展示了在我们RouteGuide服务中实现的过程：

```go
flag.Parse()
lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
if err != nil {
        log.Fatalf("failed to listen: %v", err)
}
grpcServer := grpc.NewServer()
pb.RegisterRouteGuideServer(grpcServer, &routeGuideServer{})
... // determine whether to use TLS
grpcServer.Serve(lis)
```

为了构建和启动服务器，我们需要：

```go
lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
```
表示指定我们期望客户端请求的监听端口。

```go
grpc.NewServer()
```

表示创建 gRPC 服务器的一个实例。

```go
pb.RegisterRouteGuideServer(grpcServer, &routeGuideServer{})
```

表示在 gRPC 服务器注册我们的服务实现。

```go
grpcServer.Serve(lis)
```

最后使用服务器 Serve() 方法以及我们的端口信息区实现阻塞等待，直到进程被杀死或者 Stop() 被调用。

### 创建客户端

在这部分，我们将尝试为 RouteGuide 服务创建一个 Go 的客户端。你可以从`grpc/examples/route_guide/client/client.go`看到我们完整的客户端例子代码.

#### 创建存根(stub)
为了调用服务方法，我们首先创建一个 gRPC channel 和服务器交互。我们通过给 grpc.Dial() 传入服务器地址和端口号做到这点，如下：

```go
conn, err := grpc.Dial(*serverAddr)
if err != nil {
    ...
}
defer conn.Close()
```

你可以使用 DialOptions 在 grpc.Dial 中设置授权认证（如， TLS，GCE认证，JWT认证），如果服务有这样的要求的话 —— 但是对于 RouteGuide 服务，我们不用这么做。

一旦 gRPC channel 建立起来，我们需要一个客户端 存根 去执行 RPC。我们通过 .proto 生成的 pb 包提供的 NewRouteGuideClient 方法来完成。

```go
client := pb.NewRouteGuideClient(conn)
```

#### 调用服务方法
现在让我们看看如何调用服务方法。注意，在 gRPC-Go 中，RPC以阻塞/同步模式操作，这意味着 RPC 调用等待服务器响应，同时要么返回响应，要么返回错误。

#### 1.简单 RPC

调用简单 RPC GetFeature 几乎是和调用一个本地方法一样直观。

```go
feature, err := client.GetFeature(context.Background(), &pb.Point{409146138, -746188906})
if err != nil {
        ...
}
```

如你所见，我们调用了前面创建的存根上的方法。在我们的方法参数中，我们创建并且填充了一个请求的 protocol buffer 对象（例子中为 Point）。我们同时传入了一个 context.Context ，在有需要时可以让我们改变 RPC 的行为，比如超时/取消一个正在运行的 RPC。 如果调用没有返回错误，那么我们就可以从服务器返回的第一个返回值中读到响应信息。

```go
log.Println(feature)
```

#### 2.服务器端流式 RPC

ListFeatures 就是我们说的服务器端流方法，它会返回地理的Feature 流。 如果你已经读过创建服务器，本节的一些内容也许看上去会很熟悉——流式 RPC 是在客户端和服务器两端以一种类似的方式实现的。

```go
rect := &pb.Rectangle{ ... }  // initialize a pb.Rectangle
stream, err := client.ListFeatures(context.Background(), rect)
if err != nil {
    ...
}
for {
    feature, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatalf("%v.ListFeatures(_) = _, %v", client, err)
    }
    log.Println(feature)
}
```

在简单 RPC 的例子中，我们给方法传入一个上下文和请求。然而，我们得到返回的是一个 RouteGuide_ListFeaturesClient 实例，而不是一个应答对象。客户端可以使用 RouteGuide_ListFeaturesClient 流去读取服务器的响应。

我们使用 RouteGuide_ListFeaturesClient 的 Recv() 方法去反复读取服务器的响应到一个响应 protocol buffer 对象（在这个场景下是Feature）直到消息读取完毕：每次调用完成时，客户端都要检查从 Recv() 返回的错误 err。如果返回为 nil，流依然完好并且可以继续读取；如果返回为 io.EOF，则说明消息流已经结束；否则就一定是一个通过 err 传过来的 RPC 错误。

#### 3.客户端流式 RPC

除了我们需要给方法传入一个上下文而后返回 RouteGuide_RecordRouteClient 流以外，客户端流方法 RecordRoute 和服务器端方法类似，它可以用来读 和 写消息。

```go
// Create a random number of random points
r := rand.New(rand.NewSource(time.Now().UnixNano()))
pointCount := int(r.Int31n(100)) + 2 // Traverse at least two points
var points []*pb.Point
for i := 0; i < pointCount; i++ {
    points = append(points, randomPoint(r))
}
log.Printf("Traversing %d points.", len(points))
stream, err := client.RecordRoute(context.Background())
if err != nil {
    log.Fatalf("%v.RecordRoute(_) = _, %v", client, err)
}
for _, point := range points {
    if err := stream.Send(point); err != nil {
        log.Fatalf("%v.Send(%v) = %v", stream, point, err)
    }
}
reply, err := stream.CloseAndRecv()
if err != nil {
    log.Fatalf("%v.CloseAndRecv() got error %v, want %v", stream, err, nil)
}
log.Printf("Route summary: %v", reply)
```

RouteGuide_RecordRouteClient 有一个 Send() 方法，我们可以用它来给服务器发送请求。一旦我们完成使用 Send() 方法将客户端请求写入流，就需要调用流的 CloseAndRecv()方法，让 gRPC 知道我们已经完成了写入同时期待返回应答。我们从 CloseAndRecv() 返回的 err 中获得 RPC 的状态。如果状态为nil，那么CloseAndRecv()的第一个返回值将会是合法的服务器应答。

#### 4.双向流式 RPC

最后，让我们看看双向流式 RPC RouteChat()。 和 RecordRoute 的场景类似，我们只给函数传
入一个上下文对象，拿到可以用来读写的流。但是，当服务器依然在往 他们 的消息流写入消息时，我们
通过方法流返回值。

```go
stream, err := client.RouteChat(context.Background())
waitc := make(chan struct{})
go func() {
    for {
        in, err := stream.Recv()
        if err == io.EOF {
            // read done.
            close(waitc)
            return
        }
        if err != nil {
            log.Fatalf("Failed to receive a note : %v", err)
        }
        log.Printf("Got message %s at point(%d, %d)", in.Message, in.Location.Latitude, in.Location.Longitude)
    }
}()
for _, note := range notes {
    if err := stream.Send(note); err != nil {
        log.Fatalf("Failed to send a note: %v", err)
    }
}
stream.CloseSend()
<-waitc
```

这里读写的语法和我们的客户端流方法很像，除了在完成调用时，我们会使用流的 CloseSend() 方法。
虽然每一端获取对方信息的顺序和信息被写入的顺序一致，客户端和服务器都可以以任意顺序读写——流的操作是完全独立的。

来试试吧！
假设你在 `$GOPATH/src/google.golang.org/grpc/examples/route_guide` 目录，要编译和运行服务器，只需要运行：

```go
$ go run server/server.go
```

同样的，运行客户端:

```go
$ go run client/client.go
```