# go-fuse2-example

## Loopback FUSE 请求时序图

```
用户态进程(cat)        内核vfs层       FUSE内核模块     go-fuse Server(Loopback)    底层真实文件系统
|                        |              |                   |                       |
|open("/mnt/a.txt")      |              |                   |                       |
|----------------------->|              |                   |                       |
|                        |查挂载类型    |                   |                       |
|                        |------------->|                   |                       |
|                        |              |发FUSE_LOOKUP      |                       |
|                        |              |------------------>|LoopbackRoot.Lookup    |
|                        |              |                   |---------------------->|syscall.Lstat()
|                        |              |                   |                       |(获取inode信息)
|                        |              |      返回inode信息|<----------------------|
|                        |              |<------------------|                       |
|                        |              |发送FUSE_OPEN      |                       |
|                        |              |------------------>|LoopbackNode.Open      |
|                        |              |                   |---------------------->|os.Open()
|                        |              |                   |                       |
|                        |              |                   |<----------------------|
|                        |              |         返回fh句柄|                       |
|                        |              |<------------------|                       |
|read()                  |              |                   |                       |
|----------------------->|              |                   |                       |
|                        |              |发FUSE_READ        |                       |
|                        |              |------------------>|LoopbackHandle.Read    |
|                        |              |                   |---------------------->|os.Read()
|                        |              |                   |                       |
|                        |              |           返回数据|<----------------------|
|                        |              |<------------------|                       |
|              读取到数据|              |                   |                       |
|<-----------------------|              |                   |                       |
|close()                 |              |                   |                       |
|----------------------->|              |                   |                       |
|                        |              |发FUSE_RELEASE     |                       |
|                        |              |------------------>|                       |
|                        |              |                   |LoopbackHandle.Release |
|                        |              |                   |---------------------->|file.Close()
|                        |              |                   |<----------------------|
```

## go-fuse2机制

1. 挂载阶段
    1. 调用syscall.Mount把挂载点注册到内核vfs，并指定fuse设备
    2. 创建server结构(封装/dev/fuse文件描述符、事件循环)
    3. 启动goroutine读/dev/fuse，并调度到你节点方法
2. 请求分发
    1. 内核通过/dev/fuse把请求(如:LOOKUP, READ, WRITE)发给用户态
    2. server在一个大循环中：1. 从fd读取fuse.InHeader和请求体；2. 根据节点id找到对应的inode或handle；3. 调用实现的接口方法；4. 把方法返回值打包成fuse响应，写回到/dev/fuse中。
3. Node/Handle
    - Node：文件系统结构节点(目录、文件、设备等)，响应结构性操作(Lookup、Getattr、Mkdir...)
    - Handle：打开的文件/目录句柄，响应数据流操作(Read、Write、Readdir)
    - 一个Node可以产生多个Handle，比如同一个文件被多个进程同时打开。

## 项目关键文件

|文件|说明|
|:---|:---|
|fs/bridge.go|连接用户态fuse服务器和内核模块的桥梁，读写/dev/fuse、解析内核fuse请求、封装fuse请求|

## 数据类型

### fs.InodeEmbedder

```go
type InodeEmbedder interface {
	// inode is used internally to link Inode to a Node.
	//
	// See Inode() for the public API to retrieve an inode from Node.
	embed() *Inode

	// EmbeddedInode returns a pointer to the embedded inode.
	EmbeddedInode() *Inode
}
```

go-fuse2中`fs.Inode`用来表示文件系统的每个节点（文件、目录等），用户在写自定义文件系统时候，通常在自己结构体中匿名嵌入（类似继承）：
```go
type MyNode struct {
    fs.Inode
    // 其他字段
}
```
为了让所有自定义类型暴露出嵌入的Inode，于是引入`fs.InodeEmbedder`，embed()为包内私有方法，避免外部随意访问这个接口，保证类型安全
