[TOC]

# rpc

server端是服务提供方，client端是服务调用方。既然server端可以提供服务，那么它要先实现**服务的注册**，只有注册过的服务才能被client端调用。在client端和server端，数据一般是以对象的形式存在，而对象是无法进行网络传输的，在网络传输之前，需要先把对象序列化成字节流，然后传输这些字节流，server端在接收到这些字节流之后，再反序列化得到原始对象.

## 手写rpc

 **server.go**

```go
package main

import (
	"log"
	"net"
	"testrpc"
)

type Args struct {
	A, B int
}

type Arith int

func (t *Arith) Multiply(args Args, reply *int) error {
	*reply = args.A * args.B
	return nil
}

func main() {
	// 创建一个rpc server对象
	newServer := testrpc.NewServer()

	// 向rpc server对象注册一个Arith对象，注册后，client就可以调用Arith的Multiply方法
	arith := new(Arith)
	newServer.Register(arith)

	// 监听本机的1234端口
	l, e := net.Listen("tcp", "127.0.0.1:1234")
	if e != nil {
		log.Fatalf("net.Listen tcp :0: %v", e)
	}

	for {
		// 阻塞直到从1234端口收到一个网络连接
		conn, e := l.Accept()
		if e != nil {
			log.Fatalf("l.Accept: %v", e)
		}

		//开始工作
		go newServer.ServeConn(conn)
	}
}
```

**client.go**

```go
package main

import (
	"log"
	"net"
	"os"
	"testrpc"
)

type Args struct {
	A, B int
}

func main() {

	// 连接本机的1234端口，返回一个net.Conn对象
	conn, err := net.Dial("tcp", "127.0.0.1:1234")
	if err != nil {
		log.Println(err.Error())
		os.Exit(-1)
	}

	// main函数退出时关闭该网络连接
	defer conn.Close()

	// 创建一个rpc client对象
	client := testrpc.NewClient(conn)
	// main函数退出时关闭该client
	defer client.Close()

	// 调用远端Arith.Multiply函数
	args := Args{7, 8}
	var reply int
	err = client.Call("Arith.Multiply", args, &reply)
	if err != nil {
		log.Fatal("arith error:", err)
	}
	log.Println(reply)
}
```

### 实现原理

server端要解决的问题有**服务的注册**，也就是Register方法，那么server端必须要能够存储这些服务，所以server的定义可以如下：

```go
type Service struct {
	Method    reflect.Method
	ArgType   reflect.Type
	ReplyType reflect.Type
}

type Server struct {
	ServiceMap  map[string]map[string]*Service
	serviceLock sync.Mutex
}
```

一个Service对象就对应一个服务，一个服务包括方法、参数类型和返回值类型。Server有两个属性：ServiceMap和serviceLock，ServiceMap是一系列service的集合，之所以要以Map的形式是为了方便查找，serviceLock是为了保护ServiceMap，确保同一时刻只有一个goroutine能够写ServiceMap。

#### 服务的注册

```go
func (server *Server) Register(obj interface{}) error {
	server.serviceLock.Lock()
	defer server.serviceLock.Unlock()

	//通过obj得到其各个方法，存储在servicesMap中
	tp := reflect.TypeOf(obj)
	val := reflect.ValueOf(obj)
	serviceName := reflect.Indirect(val).Type().Name()
	if _, ok := server.ServiceMap[serviceName]; ok {
		return errors.New(serviceName + " already registed.")
	}

	s := make(map[string]*Service)
	numMethod := tp.NumMethod()
	for m := 0; m < numMethod; m++ {
		service := new(Service)
		method := tp.Method(m)
		mtype := method.Type
		mname := method.Name

		service.ArgType = mtype.In(1)
		service.ReplyType = mtype.In(2)
		service.Method = method
		s[mname] = service
	}
	server.ServiceMap[serviceName] = s
	server.ServerType = reflect.TypeOf(obj)
	return nil
}
```

Register的大概逻辑就是**拿到obj（Register的参数）的各个暴露出来的方法（这里只有一个Multiply），然后存到server的ServiceMap中**。

注册之后，ServiceMap大概是这个样子

```json
{
	"Arith": {"Multiply":&{Method:Multiply, ArgType:main.Args, ReplyType:*int}}	
}
```

#### 网络传输

testrpc的网络传输是基于golang提供的net.Conn，这个net.Conn提供了两个方法：Read和Write。Read表示从网络连接中读取数据，Write表示向网络连接中写数据。

```go
const (
	EachReadBytes = 500
)

type Transfer struct {
	conn net.Conn
}

func NewTransfer(conn net.Conn) *Transfer {
	return &Transfer{conn: conn}
}

func (trans *Transfer) ReadData() ([]byte, error) {
	finalData := make([]byte, 0)
	for {
		data := make([]byte, EachReadBytes)
		i, err := trans.conn.Read(data)
		if err != nil {
			return nil, err
		}
		finalData = append(finalData, data[:i]...)
		if i < EachReadBytes {
			break
		}
	}
	return finalData, nil
}

func (trans *Transfer) WriteData(data []byte) (int, error) {
	num, err := trans.conn.Write(data)
	return num, err
}
复制代码
```

ReadData是**从网络连接中读取数据**，每次读500字节（由EachReadBytes指定），直到读完为止，然后把读到的数据返回；WriteData是**向网络连接中写数据**。

#### 序列化与反序列化

网络传输传输的对象是**字节流**。序列化就是负责把对象变成字节流的，相反的，反序列化就是负责将字节流变成程序中的对象的。在网络传输之前我们要先进行序列化，testrpc采用了json做为序列化方式:

```go
type EdCode int

func (edcode EdCode) encode(v interface{}) ([]byte, error) {
	return json.Marshal(v)
}

func (edcode EdCode) decode(data []byte, v interface{}) error {
	return json.Unmarshal(data, v)
}
```

### Server端处理网络连接

1. 构造一个Transfer对象，代表一个网络传输
2. 调用Transfer的ReadData方法从网络连接中**读数据**
3. 调用EdCode的decode方法将数据**反序列化**成普通对象
4. 获取反序列化后数据，如方法名、参数
5. 根据方法名查找ServiceMap，**拿到对应的service**
6. **调用**对应的service，得到结果
7. 将结果**序列化**
8. 使用Transfer的WriteData方法的**写回**到网络连接中

以上就是Server端在拿到Client端请求后的处理过程，我们把它封装在了ServeConn方法里。代码比较长，就不在这里贴了，有兴趣的可以去github里面看。

### Client端发起网络连接

1. 构造一个Transfer对象，代表一个网络传输
2. 将要调用的服务名和参数**序列化**
3. 使用Transfer的WriteData方法将序列化后的数据的**写入**到网络连接中
4. 阻塞直到Server端计算完
5. 调用Transfer的ReadData方法从网络连接中**读取**计算后的结果
6. 将结果**反序列化**后返回给client 代码如下：

```go
func (client *Client) Call(methodName string, req interface{}, reply interface{}) error {

	// 构造一个Request
	request := NewRequest(methodName, req)

	// encode
	var edcode EdCode
	data, err := edcode.encode(request)
	if err != nil {
		return err
	}

	// write
	// 构造一个Transfer
	trans := NewTransfer(client.conn)
	_, err = trans.WriteData(data)
	if err != nil {
		log.Println(err.Error())
		return err
	}

	// read
	data2, err := trans.ReadData()
	if err != nil {
		log.Println(err.Error())
		return err
	}

	// decode and assin to reply
	edcode.decode(data2, reply)

	// return
	return nil
}
复制代码
```

Client的定义很简单，就一个代表网络连接的conn，代码如下：

```go
type Client struct {
	conn net.Conn
}
```

## 原理

Register：

```go
// Register publishes the receiver's methods in the DefaultServer.
func Register(rcvr interface{}) error { return DefaultServer.Register(rcvr) }

// RegisterName is like Register but uses the provided name for the type
// instead of the receiver's concrete type.
func RegisterName(name string, rcvr interface{}) error {
    return DefaultServer.RegisterName(name, rcvr)
}
```

#### 反射处理过程

```go
type methodType struct {
    sync.Mutex // protects counters
    method     reflect.Method    //反射后的函数
    ArgType    reflect.Type      //请求参数的反射值
    ReplyType  reflect.Type      //返回参数的反射值
    numCalls   uint              //调用次数
}

type service struct {
    name   string                 // 服务名,这里通常为register时的对象名或自定义对象名
    rcvr   reflect.Value          // 服务的接收者的反射值
    typ    reflect.Type           // 接收者的类型
    method map[string]*methodType // 对象的所有方法的反射结果.
}
```

反射处理过程,其实就是将对象以及对象的方法,通过反射生成上面的结构,如注册Arith.Multiply(xx,xx) error 这样的对象时,生成的结构为 map["Arith"]*service, service 中method为 map["Multiply"]*methodType.

生成service对象

```go
func (server *Server) register(rcvr interface{}, name string, useName bool) error {
    //生成service
    s := new(service)
    s.typ = reflect.TypeOf(rcvr)
    s.rcvr = reflect.ValueOf(rcvr)
    sname := reflect.Indirect(s.rcvr).Type().Name()
 
    ....
    s.name = sname

    // 通过suitableMethods将对象的方法转换成map[string]*methodType结构
    s.method = suitableMethods(s.typ, true)
    
    ....

    //service存储为键值对
    if _, dup := server.serviceMap.LoadOrStore(sname, s); dup {
        return errors.New("rpc: service already defined: " + sname)
    }
    return nil
}
```

生成 map[string] *methodType:

```go
func suitableMethods(typ reflect.Type, reportErr bool) map[string]*methodType {
    methods := make(map[string]*methodType)

    //通过反射,遍历所有的方法
    for m := 0; m < typ.NumMethod(); m++ {
        method := typ.Method(m)
        mtype := method.Type
        mname := method.Name
        // Method must be exported.
        if method.PkgPath != "" {
            continue
        }
        // Method needs three ins: receiver, *args, *reply.
        if mtype.NumIn() != 3 {
            if reportErr {
                log.Println("method", mname, "has wrong number of ins:", mtype.NumIn())
            }
            continue
        }
        //取出请求参数类型
        argType := mtype.In(1)
        ...

        // 取出响应参数类型,响应参数必须为指针
        replyType := mtype.In(2)
        if replyType.Kind() != reflect.Ptr {
            if reportErr {
                log.Println("method", mname, "reply type not a pointer:", replyType)
            }
            continue
        }
        ...


        // 去除函数的返回值,函数的返回值必须为error.
        if returnType := mtype.Out(0); returnType != typeOfError {
            if reportErr {
                log.Println("method", mname, "returns", returnType.String(), "not error")
            }
            continue
        }
        
        //将方法存储成key-value
        methods[mname] = &methodType{method: method, ArgType: argType, ReplyType: replyType}
    }
    return methods
}
```

#### 网络调用

```go
// Request 每次rpc调用的请求的头部分
type Request struct {
    ServiceMethod string   // 格式为: "Service.Method"
    Seq           uint64   // 客户端生成的序列号
    next          *Request // server端保持的链表
}

// Response 每次rpc调用的响应的头部分
type Response struct {
    ServiceMethod string    // 对应请求部分的 ServiceMethod
    Seq           uint64    // 对应请求部分的 Seq
    Error         string    // 错误
    next          *Response // server端保持的链表
}
```

取出请求,并得到相应函数的调用参数:

```go
func (server *Server) readRequestHeader(codec ServerCodec) (svc *service, mtype *methodType, req *Request, keepReading bool, err error) {
    // Grab the request header.
    req = server.getRequest()
    //编码器读取生成请求
    err = codec.ReadRequestHeader(req)
    if err != nil {
        //错误处理
        ...
        return
    }

    keepReading = true

    //取出服务名以及方法名
    dot := strings.LastIndex(req.ServiceMethod, ".")
    if dot < 0 {
        err = errors.New("rpc: service/method request ill-formed: " + req.ServiceMethod)
        return
    }
    serviceName := req.ServiceMethod[:dot]
    methodName := req.ServiceMethod[dot+1:]

    //从注册时生成的map中查询出相应的方法的结构
    svci, ok := server.serviceMap.Load(serviceName)
    if !ok {
        err = errors.New("rpc: can't find service " + req.ServiceMethod)
        return
    }
    svc = svci.(*service)

    //获取出方法的类型
    mtype = svc.method[methodName]
    if mtype == nil {
        err = errors.New("rpc: can't find method " + req.ServiceMethod)
    }
```

循环处理,不断读取链接上的字节流,解密出请求,调用方法,编码响应,回写到客户端.

```go
func (server *Server) ServeCodec(codec ServerCodec) {
    sending := new(sync.Mutex)
    for {
        //读取请求
        service, mtype, req, argv, replyv, keepReading, err := server.readRequest(codec)
        if err != nil {
            ...
        }

        //调用
        go service.call(server, sending, mtype, req, argv, replyv, codec)
    }
    codec.Close()
}
```

通过参数进行函数调用

```go
func (s *service) call(server *Server, sending *sync.Mutex, mtype *methodType, req *Request, argv, replyv reflect.Value, codec ServerCodec) {
    mtype.Lock()
    mtype.numCalls++
    mtype.Unlock()
    function := mtype.method.Func
    // 通过反射进行函数调用
    returnValues := function.Call([]reflect.Value{s.rcvr, argv, replyv})
    // 返回值是不为空时,则取出错误的string
    errInter := returnValues[0].Interface()
    errmsg := ""
    if errInter != nil {
        errmsg = errInter.(error).Error()
    }
    
    //发送相应,并释放请求结构
    server.sendResponse(sending, req, replyv.Interface(), codec, errmsg)
    server.freeRequest(req)
}
```

### client端

```go
// 异步调用
func (client *Client) Go(serviceMethod string, args interface{}, reply interface{}, done chan *Call) *Call {
}

// 同步调用
func (client *Client) Call(serviceMethod string, args interface{}, reply interface{}) error {
}
```

```go
// Call represents an active RPC.
type Call struct {
    ServiceMethod string      // 服务名及方法名 格式:服务.方法
    Args          interface{} // 函数的请求参数 (*struct).
    Reply         interface{} // 函数的响应参数 (*struct).
    Error         error       // 方法完成后 error的状态.
    Done          chan *Call  // 方法调用结束后的channel.
}
```

发送请求部分代码,每次send一次请求,均生成一个call对象,并使用seq作为key保存在map中,服务端返回时从map取出call,进行相应处理.

```go
func (client *Client) send(call *Call) {
    //请求级别的锁
    client.reqMutex.Lock()
    defer client.reqMutex.Unlock()

    // Register this call.
    client.mutex.Lock()
    if client.shutdown || client.closing {
        call.Error = ErrShutdown
        client.mutex.Unlock()
        call.done()
        return
    }

    //生成seq,每次调用均生成唯一的seq,在服务端相应后会通过该值进行匹配
    seq := client.seq
    client.seq++
    client.pending[seq] = call
    client.mutex.Unlock()

    // 请求并发送请求
    client.request.Seq = seq
    client.request.ServiceMethod = call.ServiceMethod
    err := client.codec.WriteRequest(&client.request, call.Args)
    if err != nil {
        //发送请求错误时,将map中call对象删除.
        client.mutex.Lock()
        call = client.pending[seq]
        delete(client.pending, seq)
        client.mutex.Unlock()
        if call != nil {
            call.Error = err
            call.done()
        }
    }
}
```

接收响应部分的代码,这里是一个for循环,不断读取tcp上的流,并解码成Response对象以及方法的Reply对象.

```go
func (client *Client) input() {
    var err error
    var response Response
    for err == nil {
        response = Response{}
        err = client.codec.ReadResponseHeader(&response)
        if err != nil {
            break
        }

        //通过response中的 Seq获取call对象
        seq := response.Seq
        client.mutex.Lock()
        call := client.pending[seq]
        delete(client.pending, seq)
        client.mutex.Unlock()

        switch {
        case call == nil:
            err = client.codec.ReadResponseBody(nil)
            if err != nil {
                err = errors.New("reading error body: " + err.Error())
            }
        case response.Error != "":
            //服务端返回错误,直接将错误返回
            call.Error = ServerError(response.Error)
            err = client.codec.ReadResponseBody(nil)
            if err != nil {
                err = errors.New("reading error body: " + err.Error())
            }
            call.done()
        default:
            //通过编码器,将Resonse的body部分解码成reply对象.
            err = client.codec.ReadResponseBody(call.Reply)
            if err != nil {
                call.Error = errors.New("reading body " + err.Error())
            }
            call.done()
        }
    }

        // 客户端退出处理
    client.reqMutex.Lock()
    client.mutex.Lock()
    client.shutdown = true
    closing := client.closing
    if err == io.EOF {
        if closing {
            err = ErrShutdown
        } else {
            err = io.ErrUnexpectedEOF
        }
    }
    for _, call := range client.pending {
        call.Error = err
        call.done()
    }
    client.mutex.Unlock()
    client.reqMutex.Unlock()
    if debugLog && err != io.EOF && !closing {
        log.Println("rpc: client protocol error:", err)
    }
}
```

## 一些坑

- 同步调用无法超时

由于原生rpc只提供两个方法,同步的Call以及异步的Go,同步的Call服务端不返回则会一直阻塞,这里如果存在大量的不返回,会导致协程一直无法释放.

- 异步调用超时后会内存泄漏

基于异步调用加channel实现超时功能也会存在泄漏问题,原因是client的请求会存在map结构中,Go函数退出并不会清理map的内容,因此如果server端不返回的话,map中的请求会一直存储,从而导致内存泄漏.