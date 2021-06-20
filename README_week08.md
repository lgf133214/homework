# 问题
## 1、使用 redis benchmark 工具, 测试 10 20 50 100 200 1k 5k 字节 value 大小，redis get set 性能。

> 测试环境: Centos 7 1C 2G ( Docker 1.13 )

- 运行命令

```shell
redis-benchmark -d 10 -t get,set
...
```

- 结果统计

| SET 大小 | 10w 请求总耗时 | 每秒处理请求数 |
|---|---|---|
| 10 | 3.44s | 29044.44  |
| 20 | 3.46s | 28918.45 |
| 50 | 3.45s | 29002.32 |
| 100 | 3.40s | 29385.84 |
| 200 | 3.40s | 29403.12 |
| 1k | 3.55s | 28153.15 |
| 5k | 3.76s | 26609.90 |


| GET 大小 | 10w 请求总耗时 | 每秒处理请求数 |
|---|---|---|
| 10 | 3.46s | 28868.36  |
| 20 | 3.45s | 28960.32 |
| 50 | 3.46s | 28918.45 |
| 100 | 3.45s | 29002.32 |
| 200 | 3.46s | 28860.03 |
| 1k | 3.57s | 27979.86 |
| 5k | 3.80s | 26288.12 |

## 2、写入一定量的 kv 数据, 根据数据大小 1w-50w 自己评估, 结合写入前后的 info memory 信息  , 分析上述不同 value 大小下，平均每个 key 的占用内存空间。

- 测试代码

```golang
package main

import (
	"context"
	"fmt"
	"github.com/go-redis/redis/v8"
	"github.com/hhxsv5/go-redis-memory-analysis"
	"strconv"
)

var client = redis.NewClient(&redis.Options{Addr: "127.0.0.1:6379"})
var ctx = context.Background()

func main() {
	write(10000, "10b_1w", getVal(10))
	write(20000, "10b_2w", getVal(10))
	write(100000, "10b_10w", getVal(10))

	write(10000, "1kb_1w", getVal(1<<10))
	write(20000, "1kb_2w", getVal(1<<10))
	write(100000, "1kb_10w", getVal(1<<10))

	write(10000, "5kb_1w", getVal(5<<10))
	write(20000, "5kb_2w", getVal(5<<10))
	write(100000, "5kb_10w", getVal(5<<10))

	analysis()
}

func write(count int, key, value string) {
	for i := 0; i < count; i++ {
		k := key + ":" + strconv.Itoa(i)
		cmd := client.Set(ctx, k, value, -1)
		err := cmd.Err()
		if err != nil {
			fmt.Println(cmd.String())
		}
	}
}

func analysis() {
	analysis, err := gorma.NewAnalysisConnection("127.0.0.1", 6379, "")
	if err != nil {
		fmt.Println("something wrong:", err)
		return
	}
	defer analysis.Close()

	analysis.Start([]string{":"})

	err = analysis.SaveReports("./reports")
	if err == nil {
		fmt.Println("done")
	} else {
		fmt.Println("error:", err)
	}
}

func getVal(length int) string {
	data := make([]byte, length)
	for i := 0; i < length; i++ {
		data[i] = '1'
	}
	return string(data)
}
```
| Key | Count | Size | NeverExpire | AvgTtl(excluded never expire) |
|---|---|---|---|---|
| 5kb_10w:* | 100000| 6.771 MB| 100000 | 0 |
| 1kb_10w:* | 100000| 2.098 MB| 100000 | 0 |
| 5kb_2w:* | 20000  | 1.354 MB | 20000 | 0 |
| 5kb_1w:* | 10000 | 693.359 KB | 10000 | 0 |
| 10b_10w:* | 100000 | 488.281 KB| 100000 | 0 |
| 1kb_2w:* | 20000 | 429.688 KB | 20000 | 0 |
| 1kb_1w:* | 10000 | 214.844 KB | 10000 | 0 |
| 10b_2w:* | 20000 | 97.656 KB | 20000 | 0 |
| 10b_1w:* | 10000 | 48.828 KB | 10000 | 0 |
![image](https://user-images.githubusercontent.com/50295062/122672581-d166b680-d1fe-11eb-831e-1d9876e476f2.png)

