#### 方案1
粘包根本问题在于TCP是面向字节流的，而字节流是没有边界的。因此要解决粘包问题就要发送端和接收端约定某个字节作为数据包的边界或者规定固定大小的数据作为一个包。

大端和小端：数据存取和读取的顺序。大端，高位在左边；小端 ，高位在右边。

```go
// socket_stick/proto/proto.go
package proto

import (
　　"bufio"
　　"bytes"
　　"encoding/binary"
)

// Encode 消息编码
func Encode(message string) ([]byte, error) {
　　// 读取消息的长度，转换成int32类型（占4个字节）
　　var length = int32(len(message))
　　var pkg = new(bytes.Buffer)
　　err := binary.Write(pkg, binary.LittleEndian, length)   //  写入msg长度。小端的方式存放
　　if err != nil {
　　　　return nil, err
　　}
　　err = binary.Write(pkg, binary.LittleEndian, []byte(message))   // 写入消息实体。
　　if err != nil {
　　　　return nil, err
　　}
　　return pkg.Bytes(), nil
}

// Decode 解码消息
func Decode(reader *bufio.Reader) (string, error) {
　　// 读取消息的长度
　　lengthByte, _ := reader.Peek(4)                 // 读取前4个字节byte，存放的是数据长度
　　lengthBuff := bytes.NewBuffer(lengthByte)
　　var length int32
　　err := binary.Read(lengthBuff, binary.LittleEndian, &length)    //  将4个字节的byte，转为int32，放入length，length是数据长度
　　if err != nil {
　　　　return "", err
　　}
　　if int32(reader.Buffered()) < length+4 {   // 如果reader总长度<4报错
　　　　return "", err
　　}

　　// 读取真正的消息数据
　　pack := make([]byte, int(4+length))    // 在reader中读取length+4长度的字节（length是数据的长度）
　　_, err = reader.Read(pack)
　　if err != nil {
　　　　return "", err
　　}
　　return string(pack[4:]), nil
}

```

- 服务端：
```go
// socket_stick/server/main.go

func process(conn net.Conn) {
　　defer conn.Close()
　　reader := bufio.NewReader(conn)
　　for {
　　　　msg, err := proto.Decode(reader)
　　　　if err == io.EOF {
　　　　　　return
　　　　}
　　　　if err != nil {
　　　　　　fmt.Println("decode msg failed, err:", err)
　　　　　　return
　　　　}
　　　　fmt.Println("收到client发来的数据：", msg)
　　}
}

func main() {

　　listen, err := net.Listen("tcp", "127.0.0.1:30000")
　　if err != nil {
　　　　fmt.Println("listen failed, err:", err)
　　　　return
　　}
　　defer listen.Close()
　　for {
　　　　conn, err := listen.Accept()
　　　　if err != nil {
　　　　　　fmt.Println("accept failed, err:", err)
　　　　　　continue
　　　　}
　　　　go process(conn)
　　}
}
```

- 客户端：
```go
// socket_stick/client/main.go
func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:30000")
	if err != nil {
		fmt.Println("dial failed, err", err)
		return
	}
	defer conn.Close()
	for i := 0; i < 20; i++ {
		msg := `Hello, Hello. How are you?`
		data, err := proto.Encode(msg)
		if err != nil {
			fmt.Println("encode msg failed, err:", err)
			return
		}
		conn.Write(data)
	}
}
```

#### 方案二
- 1、数据打包接口
```go
/*
定义一个解决TCP粘包问题的封包和拆包的模块
——针对Message进行TLV格式的封装
  ——Message的长度，ID和内容
——这对Message进行TLV格式的拆包
  ——先读取固定长度的head-->消息内容长度和消息的类型
  ——再根据消息的长度，进行一次读写，从conn中读取消息的内容
 */

//封包，拆包模块，直接面向TCP连接中的数据流，用于处理TCP粘包的问题

type IDataPack interface {
	// 获取包的长度
	GetHeadLen() uint32
	//封包方法
	Pack(msg IMessage) ([]byte, error)
	//拆包方法
	Unpack([]byte) (IMessage, error)
}
```

- 2、消息封装
```go
//消息包含消息ID，消息长度，消息内容三部分
type Message struct {
	Id      uint32 //消息的ID
	DataLen uint32    // 消息长度
	Data    []byte //消息内容
}

//创建一个Message消息包
func NewMsgPackage(id uint32, data []byte) *Message{
	return &Message{
		Id: id,
		DataLen: uint32(len(data)),
		Data: data,
	}
}

//获取消息ID
func (m *Message) GetMessageID() uint32{
	return m.Id
}

//获取消息内容
func (m *Message) GetMessageData() []byte{
	return m.Data
}

//获取消息长度
func (m *Message) GetMessageLen() uint32{
	return m.DataLen
}

//设置消息相关
func (m *Message) SetMessageID(id uint32){
	m.Id = id
}

//设置消息相关
func (m *Message) SetMessageData(data []byte){
	m.Data = data
}

//设置消息长度
func (m *Message) SetMessageLen(length uint32){
	m.DataLen = length
}
```

- 3、封包拆包实现
```go
//拆包封包的具体模块
type DataPack struct {
	dataHeadLen uint32
}

//拆包封包实例的初始化方法
func NewDataPack() *DataPack {
	return &DataPack{}
}

// 获取包的长度
func (dp *DataPack) GetHeadLen() uint32{
	//DataHeadLen：uint32（4个字节）+ID：uint32（4个字节）=8个字节
	return 8
}

//封包方法, Message结构体变成二进制序列化的格式数据
func (dp *DataPack) Pack(msg *IMessage) ([]byte, error){
	//创建一个存放bytes字节的缓冲
	dataBuff := bytes.NewBuffer([]byte{})

	//注意写入的顺序
	//将dataLen写入databuff中
	//这里涉及到一个网络传输的大小端问题，大端序，小端序
	if err := binary.Write(dataBuff, binary.LittleEndian, msg.GetMessageLen()); err !=nil{
		return nil, err
	}

	//将MessageID写入databuff中
	if err := binary.Write(dataBuff, binary.LittleEndian, msg.GetMessageID()); err !=nil{
		return nil, err
	}

	//将data写入databuff中
	if err := binary.Write(dataBuff, binary.LittleEndian, msg.GetMessageData()); err !=nil{
		return nil, err
	}

	//二进制的序列化返回
	return dataBuff.Bytes(), nil
}

//拆包方法（）
func (dp *DataPack) Unpack(binaryData []byte) (*Message, error){
	//创建一个输入二进制数据的ioReader
	dataBuff := bytes.NewBuffer(binaryData)

	//接受消息,直解压head，获得datalen和id
	msg := &Message{}

	//读取dataLen
	//这里的&msg.DataLen是为了写入地址
	if err := binary.Read(dataBuff, binary.LittleEndian, &msg.DataLen); err!=nil{
		return nil, err
	}

	//这里的从dataBuff读取数据，应该是连续读，先读len，然后读id，不会重复
	//读取dataID
	if err := binary.Read(dataBuff, binary.LittleEndian, &msg.Id); err != nil{
		return nil, err
	}

	//这里还可以加一个判断datalen是否超出定义的长度的逻辑

    //读取data
    if err := binary.Read(dataBuff, binary.LittleEndian, &msg.Data); err != nil{
		return nil, err
	}

	//只需拆包获取msg的head，然后通过head的长度，从conn中读取一次数据
	return msg, nil
}

```

#### 方案三
使用TLV类型的数据协议，分别是Type、Len、Value。Type和Len为数据头，可以将这个两个字段都固定为四个字节。读取数据时，先将Type和Len读取出来，然后再根据Len来读取剩余的数据。

- client：
```go
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
	"net"
)

// 对数据进行编码
func Encode(id uint32, msg []byte) []byte {
	var dataLen uint32 = uint32(len(msg))
	// *Buffer实现了Writer
	buffer := bytes.NewBuffer([]byte{})
    // 将id写入字节切片
	if err := binary.Write(buffer, binary.LittleEndian, &id); err != nil {
		fmt.Println("Write to buffer error:", err)
	}
	// 将数据长度写入字节切片
	if err := binary.Write(buffer, binary.LittleEndian, &dataLen); err != nil {
		fmt.Println("Write to buffer error:", err)
	}
    // 将数据添加到后面
	msg = append(buffer.Bytes(), msg...)
	return msg
}

func main() {
	dial, err := net.Dial("tcp4", "127.0.0.1:6666")
	if err != nil {
		fmt.Println("Dial tcp error:", err)
	}
    // 向服务端发送hello,world!
	msg := []byte("hello,world!")
	var id uint32 = 1
	data := Encode(id, msg)
	dial.Write(data)
	dial.Close()
}
// 运行结果：
// Receive Data, Type:1, Len:12, Message:hello,world!
// Connection has been closed by client
```

- server：
```go
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
	"io"
	"net"
)

// 解码，从字节切片中获取id和len
func Decode(encoded []byte) (id uint32, l uint32) {
	buffer := bytes.NewBuffer(encoded)
	if err := binary.Read(buffer, binary.LittleEndian, &id); err != nil {
		fmt.Println("Read from buffer error:", err)
	}
	if err := binary.Read(buffer, binary.LittleEndian, &l); err != nil {
		fmt.Println("Read from buffer error:", err)
	}
	return id, l
}

const MAX_PACKAGE = 4096

func HandleConn(conn net.Conn) {
	defer conn.Close()
	head := make([]byte, 8)
	for {
        // 先读取8个字节的头部，也就是id和dataLen
		_, err := io.ReadFull(conn, head)
		if err != nil {
			if err == io.EOF {
				fmt.Println("Connection has been closed by client")
			} else {
				fmt.Println("Read error:", err)
			}
			return
		}
		id, l := Decode(head)
		if l > MAX_PACKAGE {
			fmt.Println("Received data grater than MAX_PACKAGE")
			return
		}
        // 然后读取剩余数据
		data := make([]byte, l)
		_, err = io.ReadFull(conn, data)
		if err != nil {
			if err == io.EOF {
				fmt.Println("Connection has been closed by client")
			} else {
				fmt.Println("Read error:", err)
			}
			return
		}
        // 打印收到的数据
		fmt.Printf("Receive Data, Type:%d, Len:%d, Message:%s\n",id, l, string(data))
	}
}

func main() {
	listener, err := net.Listen("tcp", "127.0.0.1:6666")
	if err != nil {
		fmt.Println("Listen tcp error:", err)
		return
	}
	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("Accept error:", err)
            break
		}
		// 启动一个协程处理客户端
		go HandleConn(conn)
	}
}
```