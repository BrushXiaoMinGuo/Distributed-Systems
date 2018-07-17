所有的分布式系统都在使用RPC：
RPC goals:
1: 容易编写网络编程
2: 隐藏客户端/服务端实现的细节
3: 客户端就像普通的方法调用
4: 服务端像普通的方法处理
 
一个典型的demo，将网络通讯变得像方法调用:
  Client:
    z = fn(x, y)
  Server:
    fn(x, y) {
      compute
      return z
    }
  

RPC message diagram:
  Client             Server
    request--->
       <---response

Software structure
  client app         handlers
    stubs           dispatcher
   RPC lib           RPC lib
     net  ------------ net
 
A few details:
  Which server function (handler) to call?
  Marshalling: format data into packets
    Tricky for arrays, pointers, objects, &c
    Go's RPC library is pretty powerful!
    some things you cannot pass: e.g., channels, functions
  Binding: how does client know who to talk to?
    Maybe client supplies server host name
    Maybe a name service maps service names to best server host
  Threads:
    Clients may have many threads, so > 1 call outstanding, match up replies
    Handlers may be slow, so server often runs each in a thread

RPC 问题: 怎么处理失败
  丢包, 网络故障, 服务器处理慢, 服务器崩溃

错误对rpc客户端意味着什么？
  客户端看不到回应；不知道server是否接到请求

处理失败最简单模式: 
  1: 超时重发
  2: 得到设定值报错


Q: what can go wrong with this client program?
  Put("k", 10) -- an RPC to set key's value in a DB server
  Put("k", 20) -- client then does a 2nd Put to same key
  [diagram, timeout, re-send, original arrives very late]

Q: is at-least-once ever OK?
  yes: if it's OK to repeat operations, e.g. read-only op
  yes: if application has its own plan for coping w/ duplicates
    which you will need for Lab 1

更好的关于“至多一次”：
  idea: 服务器返回之前的响应，而不是重新运行
  Q: 如何发现相同请求
  客户端包含一个XID，重发的时候使用相同的XID
  server:
    if seen[xid]:
      r = old[xid]
    else
      r = handler()
      old[xid] = r
      seen[xid] = true

至多一次的复杂性
  如何确保XID唯一?
    很大的随机数
    包含客户端的id
  服务端必须最后丢弃老的rpc请求
    什么时候丢弃
    idea:
      unique client IDs
      per-client RPC sequence numbers
      client includes "seen all replies <= X" with every RPC
      much like TCP sequence #s and acks
    or only allow client one outstanding RPC at a time
      arrival of seq+1 allows server to discard all <= seq
    or client agrees to keep retrying for < 5 minutes
      server discards after 5+ minutes
  how to handle dup req while original is still executing?
    server doesn't know reply yet; don't want to run twice
    idea: "pending" flag per executing RPC; wait or ignore

What if an at-most-once server crashes and re-starts?
  if at-most-once duplicate info in memory, server will forget
    and accept duplicate requests after re-start
  maybe it should write the duplicate info to disk?
  maybe replica server should also replicate duplicate info?

What about "exactly once"?
  at-most-once plus unbounded retries plus fault-tolerant service
  Lab 3

Go RPC is "at-most-once"
  open TCP connection
  write request to TCP connection
  TCP may retransmit, but server's TCP will filter out duplicates
  no retry in Go code (i.e. will NOT create 2nd TCP connection)
  Go RPC code returns an error if it doesn't get a reply
    perhaps after a timeout (from TCP)
    perhaps server didn't see request
    perhaps server processed request but server/net failed before reply came back

Go RPC's at-most-once isn't enough for Lab 1
  it only applies to a single RPC call
  if worker doesn't respond, the master re-send to it to another worker
    but original worker may have not failed, and is working on it too
  Go RPC can't detect this kind of duplicate
    No problem in lab 1, which handles at application level
    Lab 2 will explicitly detect duplicates

Threads
  threads are a fundamental server structuring tool
  you'll use them a lot in the labs
  they can be tricky
  useful with RPC 
  Go calls them goroutines; everyone else calls them threads

Thread = "thread of control"
  threads allow one program to (logically) do many things at once
  the threads share memory
  each thread includes some per-thread state:
    program counter, registers, stack

Threading challenges:
  sharing data 
     two threads modify the same variable at same time?
     one thread reads data that another thread is changing?
     these problems are often called races
     need to protect invariants on shared data
     use Go sync.Mutex
  coordination between threads
    e.g. wait for all Map threads to finish
    use Go channels
  deadlock 
     thread 1 is waiting for thread 2
     thread 2 is waiting for thread 1
     easy detectable (unlike races)
  lock granularity
     coarse-grained -> simple, but little concurrency/parallelism
     fine-grained -> more concurrency, more races and deadlocks
  let's look at labrpc RPC package to illustrate these problems

look at today's handout -- labrpc.go
  it similar to Go's RPC system, but with a simulated network
    the network delays requests and replies
    the network loses requests and replies
    the network re-orders requests and replies
    useful for testing labs 2 etc.
  illustrates threads, mutexes, channels
  complete RPC package is written in Go itself

structure
  


struct Network
  description of network
    servers
    client endpoints
  mutex per network

RPC overview
  many examples in test_test.go
    e.g., TestBasic()
  application calls Call()
     reply := end.Call("Raft.AppendEntries", args, &reply) --   send an RPC, wait for reply
  servers side:
     srv := MakeServer()
     srv.AddService(svc) -- a server can have multiple services, e.g. Raft and k/v
       pass srv to net.AddServer()
     svc := MakeService(receiverObject) -- obj's methods will handle RPCs
       much like Go's rpcs.Register()
       pass svc to srv.AddService()

struct Server
  a server support many services

AddService
  add a service name
  Q: why a lock?
  Q: what is defer()?
  
Dispatch
  dispatch a request to the right service
  Q: why hold a lock?
  Q: why not hold lock to end of function?

Call():
  Use reflect to find type of argument
  Use gob marshall argument
  e.ch is the channel to the network to send request
  Make a channel to receive reply from network ( <- req.replyCh)

MakeEnd():
  has a thread/goroutine that simulates the network
    reads from e.ch and process requests
    each requests is processed in a separate goroutine
      Q: can an end point have many outstanding requests?
    Q: why rn.mu.Lock()?
    Q: what does lock protect?

ProcessReq():
  finds server endpoint
  if network unreliable, may delay and drop requests,
  dispatch request to server in a new thread
  waits on reply by reading ech or until 100 msec has passed
    100 msec just to see if server is dead
  then return reply
    Q: who will read the reply?
  Q: is ok that ProcessReq doesn't hold rn lock?

Service.dispatch():
 find method for a request
 unmarshall arguments
 call method
 marshall reply
 return reply

Go's "memory model" requires explicit synchronization to communicate!
  This code is not correct:
    var x int
    done := false
    go func() { x = f(...); done = true }
    while done == false { }
  it's very tempting to write, but the Go spec says it's undefined
  use a channel or sync.WaitGroup or instead

Study the Go tutorials on goroutines and channels
  Use Go's race detector:
    https://golang.org/doc/articles/race_detector.html
    go test --race mypkg
