# vmselect

## 命令行参数
### search.maxWorkersPerQuery
单次查询请求处理可用的 cpu 核心数，默认是 gomaxprocs（系统可用的 cpu 核心数），最大值为 32

### dedup.minScrapeInterval
查询处理vmstroage返回的数据，去重间隔


## 源码亮点

### RegisterAndWriteBlock
这段代码使用 addrsPool 优化内存分配值得借鉴
```Go
func (tbfw *tmpBlocksFileWrapper) RegisterAndWriteBlock(mb *storage.MetricBlock, workerID uint) error {
	tbfwLocal := &tbfw.shards[workerID]
	tbfwLocal.initIfNeeded()

	bb := tmpBufPool.Get()
	bb.B = storage.MarshalBlock(bb.B[:0], &mb.Block)

	addr, err := tbfwLocal.tbf.WriteBlockData(bb.B, workerID)
	tmpBufPool.Put(bb)
	if err != nil {
		return err
	}

	m := tbfwLocal.m
	metricName := mb.MetricName
	addrsIdx := tbfwLocal.prevAddrsIdx
	if tbfwLocal.prevMetricName == nil || string(metricName) != string(tbfwLocal.prevMetricName) {
		idx, ok := m[string(metricName)]
		if !ok {
			idx = tbfwLocal.newBlockAddrs()
		}
		addrsIdx = idx
		tbfwLocal.prevMetricName = append(tbfwLocal.prevMetricName[:0], metricName...)
		tbfwLocal.prevAddrsIdx = addrsIdx
	}
	addrs := &tbfwLocal.addrssPool[addrsIdx]

	addrsPool := tbfwLocal.addrsPool
	if uintptr(cap(addrsPool)) >= maxFastAllocBlockSize/unsafe.Sizeof(tmpBlockAddr{}) && len(addrsPool) == cap(addrsPool) {
		// Allocate a new addrsPool in order to avoid slow allocation of an object
		// bigger than maxFastAllocBlockSize bytes at append() below.
		addrsPool = make([]tmpBlockAddr, 0, maxFastAllocBlockSize/unsafe.Sizeof(tmpBlockAddr{}))
		tbfwLocal.addrsPool = addrsPool
	}
	if canAppendToBlockAddrPool(addrsPool, addrs.addrs) {
		// It is safe appending addr to addrsPool, since there are no other items added there yet.
		addrsPool = append(addrsPool, addr)
		tbfwLocal.addrsPool = addrsPool
		addrs.addrs = addrsPool[len(addrsPool)-len(addrs.addrs)-1 : len(addrsPool) : len(addrsPool)]
	} else {
		// It is unsafe appending addr to addrsPool, since there are other items added there.
		// So just append it to addrs.addrs.
		addrs.addrs = append(addrs.addrs, addr)
	}

	if len(addrs.addrs) == 1 {
		metricNamesBuf := tbfwLocal.metricNamesBuf
		if cap(metricNamesBuf) >= maxFastAllocBlockSize && len(metricNamesBuf)+len(metricName) > cap(metricNamesBuf) {
			// Allocate a new metricNamesBuf in order to avoid slow allocation of byte slice
			// bigger than maxFastAllocBlockSize bytes at append() below.
			metricNamesBuf = make([]byte, 0, maxFastAllocBlockSize)
		}
		metricNamesBufLen := len(metricNamesBuf)
		metricNamesBuf = append(metricNamesBuf, metricName...)
		metricNameStr := bytesutil.ToUnsafeString(metricNamesBuf[metricNamesBufLen:])

		orderedMetricNames := tbfwLocal.orderedMetricNames
		orderedMetricNames = append(orderedMetricNames, metricNameStr)
		m[metricNameStr] = addrsIdx

		tbfwLocal.orderedMetricNames = orderedMetricNames
		tbfwLocal.metricNamesBuf = metricNamesBuf
	}

	return nil
}
```

## 核心流程点
### ExportCSVHandler
```Go
app/vmselect/prometheus/prometheus.go:158
```