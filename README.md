# tstorage [![Go Reference](https://pkg.go.dev/badge/mod/github.com/nakabonne/tstorage.svg)](https://pkg.go.dev/mod/github.com/nakabonne/tstorage)

`tstorage` is a fast local in-memory/on-disk storage package for time-series data with a straightforward API.
It massively optimises ingestion as it allows the database to slice data extremely efficiently in small chunks and process it all in parallel.

## Usage
By default, `tstorage.Storage` works as an in-memory database.
The below example illustrates how to insert a row into the process memory and immediately select it.

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

### Examples
To make time-series data persistent on disk, specify the path to directory that stores time-series data through `WithDataPath` option.
Also, in tstorage, the combination of metric name and labels preserves uniqueness.

Here is an example of insertion a labeled metric to disk.

```go
package main

import (
	"fmt"

	"github.com/nakabonne/tstorage"
)

func main() {
	storage, _ := tstorage.NewStorage(
		tstorage.WithDataPath("./data"),
		tstorage.WithTimestampPrecision(tstorage.Seconds),
	)
	defer storage.Close()

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
	for _, p := range points {
		fmt.Printf("timestamp: %v, value: %v\n", p.Timestamp, p.Value)
		// => timestamp: 1600000000, value: 0.1
	}
}
```

<!-- TODO: Add an example for partition duration with description about partition-->


For more examples see [the documentation](https://pkg.go.dev/github.com/nakabonne/tstorage#pkg-examples).

## Benchmarks
Benchmark tests were made using Intel(R) Core(TM) i7-8559U CPU @ 2.70GHz with 16GB of RAM on macOS 10.15.7

```
$ go version
go version go1.16.2 darwin/amd64

$ go test -benchtime=4s -benchmem -bench=. .
goos: darwin
goarch: amd64
pkg: github.com/nakabonne/tstorage
cpu: Intel(R) Core(TM) i7-8559U CPU @ 2.70GHz
BenchmarkStorage_InsertRows-8                            	15981140	       303.4 ns/op	     169 B/op	       2 allocs/op
BenchmarkStorage_SelectAmongThousandPoints-8   	21974707	       217.5 ns/op	      56 B/op	       2 allocs/op
BenchmarkStorage_SelectAmongMillionPoints-8    	16951826	       282.4 ns/op	      55 B/op	       1 allocs/op
PASS
ok  	github.com/nakabonne/tstorage	17.166s
```

## Internal
Time-series database has specific characteristics in its workload.
In terms of write operations, a time-series database has to ingest a tremendous amount of data points.
Time-series data is mostly an append-only workload with delete operations performed in batches on less recent data.
In terms of read operations, in most cases, we want to retrieve multiple data points by specifying its time range, also, most recent first: query the recent data in real-time.

Based on these characteristics, `tstorage` adopts a linear data model structure that partitions data points by time, totally different from the B-trees or LSM trees based storage engines.
Each partition acts as a fully independent database containing all data points for its time range.

![Screenshot](architecture.jpg)

Key benefits:
- We can insert data without reading (blind writes) and rewriting that do not change.
- We can easily ignore all data outside of the partition time range when querying data points.
- When a partition gets full, we can persist the data from our in-memory database by sequentially writing just a handful of larger files. We avoid any write-amplification and serve SSDs and HDDs equally well.

### Memory partition
The memory partition is writable and stores data points in heap. The head partition is always memory partition. Its next one is also memory partition to accept out-of-order data points.
It stores data points in an ordered Slice, which offers excellent cache hit ratio compared to linked lists unless it gets updated way too often (like delete, add elements at random locations).

### Disk partition
The old memory partitions get compacted and persisted to the directory prefixed with `p-`, under the directory specified with the `WithDataPath` option.
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
The `data` is compressed, read-only and is memory-mapped with [mmap](https://en.wikipedia.org/wiki/Mmap) that maps a kernel address space to a user address space.
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


## Used by
- [ali](https://github.com/nakabonne/ali) - A load testing tool capable of performing real-time analysis

## References
- https://misfra.me/state-of-the-state-part-iii
- https://fabxc.org/tsdb
- https://questdb.io/blog/2020/11/26/why-timeseries-data
- https://www.xaprb.com/blog/2014/06/08/time-series-database-requirements
- https://github.com/VictoriaMetrics/VictoriaMetrics
