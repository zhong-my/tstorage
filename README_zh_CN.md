# tstorage [![Go Reference](https://pkg.go.dev/badge/mod/github.com/nakabonne/tstorage.svg)](https://pkg.go.dev/mod/github.com/nakabonne/tstorage)

`tstorage` 是一个轻量级的本地磁盘存储引擎，用于处理时间序列数据，具有简单的API。特别是对写入和读取 TSDB 的 goroutine 安全功能做了大量优化，并支持按时间划分数据点。

## 动机

我正在开发几个工具，用于处理大量的时间序列数据，例如 [Ali](https://github.com/nakabonne/ali) 和 [Gosivy](https://github.com/nakabonne/gosivy)。特别是 Ali，我一直面临一个问题，随着时间的推移，堆的消耗越来也大，因为它是一个进行实时分析的负载测试工具。我在一个提供 API 的快速 TSDB 库中小试牛刀，但最终没有任何东西能如我所愿，这就是为什么我决定自己编写这个包。

要了解 `tstorage` 对 Ali 的性能提升有多大帮助，请看发行说明 [here](https://github.com/nakabonne/ali/releases/tag/v0.7.0)。

## 运用
当前，`tstorage` 支持 Go 1.16 或以上版本。

默认情况下，`tstorage.Storage` 作为内存数据库。以下例子说明了如何添加一行到内存中并搜索它。

```go
package main

import (
	"fmt"

	"github.com/nakabonne/tstorage"
)

func main() {
	storage, _ := tstorage.NewStorage(
		tstorage.WithTimestampPrecision(tstorage.Seconds),
	)
	defer storage.Close()

	_ = storage.InsertRows([]tstorage.Row{
		{
			Metric: "metric1",
			DataPoint: tstorage.DataPoint{Timestamp: 1600000000, Value: 0.1},
		},
	})
	points, _ := storage.Select("metric1", nil, 1600000000, 1600000001)
	for _, p := range points {
		fmt.Printf("timestamp: %v, value: %v\n", p.Timestamp, p.Value)
		// => timestamp: 1600000000, value: 0.1
	}
}
```

### 使用磁盘
要使时间序列数据在磁盘上持久存在，需要设定存储目录路径 [WithDataPath](https://pkg.go.dev/github.com/nakabonne/tstorage#WithDataPath) 的选项。

```go
storage, _ := tstorage.NewStorage(
	tstorage.WithDataPath("./data"),
)
defer storage.Close()
```

### 标记
在 tstorage，你可以给 metric 的组合定义名字和可选的标签。下面是一个向磁盘插入 metric 标签的例子

```go
metric := "mem_alloc_bytes"
labels := []tstorage.Label{
	{Name: "host", Value: "host-1"},
}

_ = storage.InsertRows([]tstorage.Row{
	{
		Metric:    metric,
		Labels:    labels,
		DataPoint: tstorage.DataPoint{Timestamp: 1600000000, Value: 0.1},
	},
})
points, _ := storage.Select(metric, labels, 1600000000, 1600000001)
```

更多例子看这里 [文档](https://pkg.go.dev/github.com/nakabonne/tstorage#pkg-examples).

## 基准测试
基础测试将使用计算机 Intel(R) Core(TM) i7-8559U CPU @ 2.70GHz with 16GB of RAM on macOS 10.15.7

```
$ go version
go version go1.16.2 darwin/amd64

$ go test -benchtime=4s -benchmem -bench=. .
goos: darwin
goarch: amd64
pkg: github.com/nakabonne/tstorage
cpu: Intel(R) Core(TM) i7-8559U CPU @ 2.70GHz
BenchmarkStorage_InsertRows-8                  	14135685	       305.9 ns/op	     174 B/op	       2 allocs/op
BenchmarkStorage_SelectAmongThousandPoints-8   	20548806	       222.4 ns/op	      56 B/op	       2 allocs/op
BenchmarkStorage_SelectAmongMillionPoints-8    	16185709	       292.2 ns/op	      56 B/op	       1 allocs/op
PASS
ok  	github.com/nakabonne/tstorage	16.501s
```

## 内部细节
Time-series database has specific characteristics in its workload.
In terms of write operations, a time-series database has to ingest a tremendous amount of data points ordered by time.
Time-series data is immutable, mostly an append-only workload with delete operations performed in batches on less recent data.
In terms of read operations, in most cases, we want to retrieve multiple data points by specifying its time range, also, most recent first: query the recent data in real-time.
Besides, time-series data is already indexed in time order.

Based on these characteristics, `tstorage` adopts a linear data model structure that partitions data points by time, totally different from the B-trees or LSM trees based storage engines.
Each partition acts as a fully independent database containing all data points for its time range.


```
  │                 │
Read              Write
  │                 │
  │                 V
  │      ┌───────────────────┐ max: 1600010800
  ├─────>   Memory Partition
  │      └───────────────────┘ min: 1600007201
  │
  │      ┌───────────────────┐ max: 1600007200
  ├─────>   Memory Partition
  │      └───────────────────┘ min: 1600003601
  │
  │      ┌───────────────────┐ max: 1600003600
  └─────>   Disk Partition
         └───────────────────┘ min: 1600000000
```

Key benefits:
- We can easily ignore all data outside of the partition time range when querying data points.
- Most read operations work fast because recent data get cached in heap.
- When a partition gets full, we can persist the data from our in-memory database by sequentially writing just a handful of larger files. We avoid any write-amplification and serve SSDs and HDDs equally well.

### Memory partition
The memory partition is writable and stores data points in heap. The head partition is always memory partition. Its next one is also memory partition to accept out-of-order data points.
It stores data points in an ordered Slice, which offers excellent cache hit ratio compared to linked lists unless it gets updated way too often (like delete, add elements at random locations).

All incoming data is written to a write-ahead log (WAL) right before inserting into a memory partition to prevent data loss.

### Disk partition
The old memory partitions get compacted and persisted to the directory prefixed with `p-`, under the directory specified with the [WithDataPath](https://pkg.go.dev/github.com/nakabonne/tstorage#WithDataPath) option.
Here is the macro layout of disk partitions:

```
$ tree ./data
./data
├── p-1600000001-1600003600
│   ├── data
│   └── meta.json
├── p-1600003601-1600007200
│   ├── data
│   └── meta.json
└── p-1600007201-1600010800
    ├── data
    └── meta.json
```

As you can see each partition holds two files: `meta.json` and `data`.
The `data` is compressed, read-only and is memory-mapped with [mmap(2)](https://en.wikipedia.org/wiki/Mmap) that maps a kernel address space to a user address space.
Therefore, what it has to store in heap is only partition's metadata. Just looking at `meta.json` gives us a good picture of what it stores:

```json
$ cat ./data/p-1600000001-1600003600/meta.json
{
  "minTimestamp": 1600000001,
  "maxTimestamp": 1600003600,
  "numDataPoints": 7200,
  "metrics": {
    "metric-1": {
      "name": "metric-1",
      "offset": 0,
      "minTimestamp": 1600000001,
      "maxTimestamp": 1600003600,
      "numDataPoints": 3600
    },
    "metric-2": {
      "name": "metric-2",
      "offset": 36014,
      "minTimestamp": 1600000001,
      "maxTimestamp": 1600003600,
      "numDataPoints": 3600
    }
  }
}
```

Each metric has its own file offset of the beginning.
Data point slice for each metric is compressed separately, so all we have to do when reading is to seek, and read the points off.

### Out-of-order data points
What data points get out-of-order in real-world applications is not uncommon because of network latency or clock synchronization issues; `tstorage` basically doesn't discard them.
If out-of-order data points are within the range of the head memory partition, they get temporarily buffered and merged at flush time.
Sometimes we should handle data points that cross a partition boundary. That is the reason why `tstorage` keeps more than one partition writable.

## More
Want to know more details on tstorage internal? If so see the blog post: [Write a time-series database engine from scratch](https://nakabonne.dev/posts/write-tsdb-from-scratch).

## Acknowledgements
This package is implemented based on tons of existing ideas. What I especially got inspired by are:
- https://misfra.me/state-of-the-state-part-iii
- https://fabxc.org/tsdb
- https://questdb.io/blog/2020/11/26/why-timeseries-data
- https://akumuli.org/akumuli/2017/04/29/nbplustree
- https://github.com/VictoriaMetrics/VictoriaMetrics

A big "thank you!" goes out to all of them.
