## golang内置rpc的简单使用

### RPC版"Hello,World"

Go 语言的RPC包的路径为net/rpc，也就是放在了net包下面。因此可以猜测该RPC包是建立在net包基础之上的。我们尝试基于RPC实现一个简单的打印例子。

先构造一个HelloService 类型，其中的Hello()方法用于实现打印功能：

```golang
type HelloService struct{}

func (h *HelloService) Hello(request string, reply *string) error {
	*reply = "hello: " + request
	return nil
}
```

其中hello方法必须满足Go语言的RPC规则：方法只能有两个可序列化的参数，其中第二个参数是指针类型，并且返回一个error类型，同时必须是公开的方法。

官方定义规则标准如下：

只有满足这些标准的方法才可用于远程访问；

其他方法将被忽略：

- 该方法的类型是导出的。
- 方法是导出的
- 该方法有两个参数，都是导出（或内置）类型
- 该方法的第二个参数是指针。
- 该方法具有返回类型错误。

该方法必须与下面示例方法一致：

```golang
func (t *T) MethodName(argType T1, replyType *T2) erro
```

然后就可以将HelloService类型的对象注册为一个RPC服务：

```golang
func main() {
	rpc.RegisterName("HelloService", new(HelloService))
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatalf("listen error, err: %v", err)
	}
	con, err := listener.Accept()
	if err != nil {
		log.Fatal("accept error:", err)
	}
	rpc.ServeConn(con)
}
```
其中 rpc.RegisterName()函数调用会将对象类型中所有满足RPC规则的对象方法注册为RPC函数，所有注册的方法会放在HelloService服务的空间之下。然后建立一个唯一的TCP链接，并且通过rpc。ServeConn()函数在该TCP链接上为对方提供RPC服务。

下面是客户端请求HelloService服务的代码：
```golang
func main() {
	client, err := rpc.Dial("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("dialing:", err)
	}
	var reply string
	err = client.Call("HelloService.Hello", "hello", &reply)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(reply)
}
```

首先通过rpc.Dial 拨号RPC服务，然后client.Call()调用具体的RPC方法。在调用client.Call()时，第一个参数是用点号链接的RPC服务名字和方法名字，第二个参数和第三个参数分别是定义RPC方法的两个参数。

### 更安全的RPC接口

在设计RPC的底层应用中，作为开发人员一般包含至少有3种角色：首先是服务器端实现RPC方法的开发人员，其次是客户端调用RPC方法的人员，最后也是最重要的是制定服务器和客户端RPC接口规范的设计人员。在前面的例子中，为了简化我们把几种角色的工作放到了一起，虽然看似实现简单，但不利于后期的维护和工作的切割。

如果要重构HelloService服务，第一步需要明确的名字和接口：

```golang
const HelloServiceName = "path/to/v1/pkg.HelloService"

type HelloServiceInterface interface {
	Hello(request string, reply *string) error
}

func RegisterHelloService(svc HelloServiceInterface) error {
	return rpc.RegisterName(HelloServiceName, svc)
}
```

我们将RPC服务的接口规范分为3部分：首先是服务的名字，然后是服务要实现的详细方法列表，最后是注册该类型服务的函数，为了避免名字冲突和方法的版本控制，我们在RPC服务的名字中增加了包路径前缀（这个是rpc服务抽象的包路径，并非完全等价于go语言的包路径）和版本。RegisterHelloService注册服务时，编译器会要求传入的对象满足HelloServiceInterface接口。

在定义了rpc服务规范后，客户端就可以根据规范编写rpc调用的代码了：
```golang
func main() {
	client, err := rpc.Dial("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("dialing:", err)
	}
	var reply string
	err = client.Call(HelloServiceName + ".Hello", "hello", &reply)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(reply)
}
```
其中唯一的变化是client.Call()的第一个参数用HelloServiceName + ".Hello"代替了"HelloService.Hello"。然而通过client.Call()函数调用了RPC方法依然比较繁琐，同时参数的类型依然无法得到编译器提供的安全保障。

为例简化客户端调用的RPC函数，我们可以在接口规范部分增加对客户端的封装：
```golang
var _ HelloServiceInterface = (*HelloServiceClient)(nil)

type HelloServiceClient struct {
	*rpc.Client
}

func DiaHelloService(network, address string) (*HelloServiceClient, error) {
	c, err := rpc.Dial(network, address)
	if err != nil {
		return nil, err
	}
	return &HelloServiceClient{Client: c}, nil
}

func (h HelloServiceClient) Hello(request string, reply *string) error {
	return h.Client.Call(HelloServiceName + ".Hello", request, reply)
}
```

我们在接口规范中增加真对客户端新增了HelloServiceClient类型，该类型也必须满足HelloServiceInterface接口，这样客户端及可以直接通过接口对应的方法调用RPC函数，同时提供了一个DialHelloService()函数，直接拨号HelloService服务

基于新的客户端接口，我们可以简化客户端用户的用户代码：

```golang
func main() {
	client, err := DiaHelloService("tcp", ":1234")
	if err != nil {
		log.Fatal("dialing:", err)
	}
	var reply string
	err = client.Hello("hello", &reply)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(reply)
}
```

现在客户端用户不需要担心RPC方法名字或参数类型不匹配等低级错误的发生了。

最后基于RPC接口规范编写真实的服务端代码如下：
```golang
func main() {
	RegisterHelloService(new(HelloService))
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatalf("listen error, err: %v", err)
	}
	for {
		con, err := listener.Accept()
		if err != nil {
			log.Fatal("accept error:", err)
		}
		go rpc.ServeConn(con)
	}
}
```

在新的RPC服务器端实现中，我们用RegisterHelloService()函数来注册函数，这样不仅可以避免命名服务名称的工作，同时也保证了传入的服务对象满足RPC接口的定义，最后新的服务改为支持多个TCP链接，然后为每个TCP链接提供RPC服务。