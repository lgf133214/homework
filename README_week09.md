## 第九周，网络编程

### 1、总结几种 socket 粘包的解包方式: fix length/delimiter based/length field based frame decoder。尝试举例其应用

- fix length
每次收发的数据包的长度固定，适合用在数据量少或者数据长度分布均匀的情况，否者会造成流量的浪费（为了解决粘包，每次收发的信息都是独立的，需要立马发送，不能等下一个，所以不够长度要加空格）。加填充字符又导致了填充字符不能和数据里的字符冲突。

- delimiter based
采用分隔符分割，比如生活中的标点符号、http 或者 redis 网络协议（😄）里的 『 \r\n 』、http post 上传文件用的 boundary，excel 的 xls 格式里的逗号等等。显而易见的，分隔符需要独立于数据，不能出现在数据中，所以适用于数据简单，易区分的场景，数据过于复杂，应用程序肯定要做过滤，过滤的话复杂度就上来了（要多扫描几次）。

- length field based frame decoder
显式指出数据包的长度，应用最广泛，很多 api 使用的这种方式，http 中的 content-length，各种三方 api（大小端）等等。

### 2、实现一个从 socket connection 中解码出 goim 协议的解码器

对 goim 协议并无深入了解，代码自我感觉有误

```golang
type Message struct {
	Version   int16
	Operation int32
	Sequence  int32
	Body      []byte
}

func decode(c net.Conn) (message Message, err error) {
	buf := make([]byte, 4)

	// PacketLen
	n, err := c.Read(buf)
	if err != nil {
		return
	} else if n != 4 {
		err = errors.New("PacketLen read error")
		return
	}
	var packetLen int32
	parseByBigEndian(buf, &packetLen)

	// HeaderLen
	// 暂时丢弃，不明白为何会需要这个，目前来看明明头部长度是固定的
	n, err = c.Read(buf[:2])
	//if err != nil {
	//	return
	//}else if n!=2{
	//	err = errors.New("HeaderLen read error")
	//	return
	//}
	//var headerLen int16
	//parseByBigEndian(buf[:2], &headerLen)

	// Version
	n, err = c.Read(buf[:2])
	if err != nil {
		return
	} else if n != 2 {
		err = errors.New("Version read error")
		return
	}
	var version int16
	parseByBigEndian(buf[:2], &version)

	// Operation
	n, err = c.Read(buf)
	if err != nil {
		return
	} else if n != 4 {
		err = errors.New("Operation read error")
		return
	}
	var operation int32
	parseByBigEndian(buf, &operation)

	// Sequence
	n, err = c.Read(buf)
	if err != nil {
		return
	} else if n != 4 {
		err = errors.New("Sequence read error")
		return
	}
	var sequence int32
	parseByBigEndian(buf, &sequence)

	// Body
	body := make([]byte, packetLen-16)
	n, err = c.Read(body)
	if err != nil {
		return
	} else if n != len(body) {
		err = errors.New("Body read error")
		return
	}

	message.Version = version
	message.Operation = operation
	message.Sequence = sequence
	message.Body = body
	return
}

func parseByBigEndian(buf []byte, ret interface{}) {
	buffer := bytes.NewBuffer(buf)
	_ = binary.Read(buffer, binary.BigEndian, ret)
	return
}

```
